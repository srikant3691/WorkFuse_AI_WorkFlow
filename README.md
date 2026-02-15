# WorkFuse

**WorkFuse** is an open-source, AI-native workflow automation platform designed for developers who need durable, long-running execution without the timeout limits of standard serverless functions.

Think of it as a self-hosted alternative to n8n or Zapier, but built on a modern AWS-powered stack. It decouples the visual definition of a workflow from its execution, allowing you to chain AI agents (OpenAI, Anthropic, Gemini) with API calls, all managed by a durable event-driven engine.

## Why WorkFuse?

I needed a way to orchestrate AI agents and API calls without getting hit by serverless function timeouts or losing state when a service flakes out. WorkFuse solves this by separating the **Editor** (React Flow) from the **Engine** (Inngest).

You define the logic visually, but the execution happens securely in the background, complete with real-time feedback loops so you can see exactly which step your workflow is currently processing.

## Key Features

* **AI Auto-Configuration (Magic Input):** Don't know JSON? Just type in plain English (e.g., *"Send a POST request to Stripe to create a customer named John"*), and our "Intent-to-Config" layer automatically fills out the complex node settings and API headers for you.
* **Visual Node Editor:** Built on **React Flow**. Support for custom node types, connecting triggers to actions, and intuitive edge mapping.
* **Durable Execution Engine:** Powered by **Inngest**. Workflows can run for seconds, minutes, or days. The engine handles retries, sleep/delays, and step persistence automatically—no timeouts.
* **Enterprise-Grade Security:**
    * **AWS Secrets Manager:** Sensitive API keys are **never** stored in the database. They are encrypted and stored in AWS Secrets Manager; only the ARN is saved in Postgres.
    * **Zero-Trust Runtime:** Keys are decrypted only in the ephemeral memory of the worker function during execution.
* **AI-First Architecture:** Native integration with the **Vercel AI SDK**. Switch models (GPT-4o, Claude 3.5, Gemini 1.5) instantly via a dropdown.
* **Real-Time Observability:** The UI updates live via WebSockets. Watch nodes turn green (success) or red (failure) as the backend processes the graph.
* **Dynamic Templating:** Pass data between nodes using Handlebars syntax (e.g., `{{ google_form.responses.email }}`).

---

## Tech Stack

This project uses a "Bleeding Edge" stack optimized for type safety and scalability on AWS.

### **Core & Frontend**
* **Framework:** Next.js
* **Language:** TypeScript
* **Styling:** Tailwind CSS + Shadcn/UI
* **Canvas:** React Flow (xyflow)
* **State:** Jotai (Client) & Nuqs (URL State)

### **Cloud Infrastructure (AWS)**
* **Hosting:** **AWS Amplify** (CI/CD & Hosting for Next.js)
* **Database:** **Amazon RDS** (Managed PostgreSQL)
* **Secrets:** **AWS Secrets Manager** (Secure storage for user credentials)

### **Backend & Execution**
* **API Layer:** tRPC (End-to-end type safety)
* **ORM:** Prisma
* **Workflow Engine:** Inngest (Serverless Queue & Orchestration)
* **AI Layer:** Vercel AI SDK (Generative UI & Schema Mapping)
* **Auth:** Better Auth (GitHub/Google OAuth)

---

## Architecture

WorkFuse separates the **Builder** (Frontend) from the **Runner** (Backend Worker).

1. **Design Phase:** The user builds a graph. If they use "Magic Input," the backend uses an LLM to map their English intent to the node's Zod schema.
2. **Secure Storage:** When a user adds an API Key, the backend sends it to **AWS Secrets Manager**, receives an ARN, and stores *only* the ARN in the database.
3. **Execution Phase:** A trigger (Webhook) pushes an event to **Inngest**. The worker fetches the graph, topologically sorts the nodes, retrieves secrets via the ARN, and executes the logic step-by-step.

```mermaid
graph TD
    subgraph Client [Frontend - AWS Amplify]
        Browser[User Browser]
        ReactFlow[React Flow Canvas]
        AI_Config[AI Auto-Config Helper]
        tRPC_Client[tRPC Client]
        
        Browser --> ReactFlow
        ReactFlow --> AI_Config
        AI_Config -->|English Intent| tRPC_Client
        tRPC_Client -->|JSON Schema| AI_Config
    end
    
    subgraph AWS_Cloud [AWS Infrastructure]
        Amplify[Amplify Hosting]
        RDS[(Amazon RDS - Postgres)]
        Secrets[AWS Secrets Manager]
    end
    
    subgraph Backend_Logic [Server-Side]
        NextAPI[Next.js API Routes]
        tRPC_Server[tRPC Router]
        InngestSDK[Inngest SDK]
        
        tRPC_Client -->|API Calls| Amplify
        Amplify --> NextAPI
        NextAPI --> tRPC_Server
        tRPC_Server -->|Read/Write Workflows| RDS
        tRPC_Server -->|Store API Keys| Secrets
        tRPC_Server -->|Generate Config| LLM[LLM Service]
    end
    
    subgraph Execution_Engine [The Worker]
        InngestServer[Inngest Cloud]
        Worker[Inngest Function]
        
        NextAPI -->|Webhook Trigger| InngestSDK
        InngestSDK -->|Event| InngestServer
        InngestServer -->|Run Job| Worker
        Worker -->|Fetch Graph| RDS
        Worker -->|Get Secret Value| Secrets
        Worker -->|Execute Node| External_APIs[OpenAI / Stripe / Slack]
        Worker -->|Realtime Update| WebSocket[User UI]
    end
```

---

## Getting Started

### Prerequisites
* Node.js 18+ installed locally
* An AWS Account with permissions to create RDS instances and Secrets
* Docker (Optional, if you prefer running a local Postgres container)

### 1. Clone & Install

```bash
git clone https://github.com/yourusername/workfuse.git
cd workfuse
npm install
```

### 2. Environment Setup

Rename `.env.example` to `.env` and configure your AWS and Database credentials.

```env
# Database (Amazon RDS Connection String)
DATABASE_URL="postgresql://user:password@workfuse-db.xxxx.us-east-1.rds.amazonaws.com:5432/workfuse"

# Auth (Better Auth)
BETTER_AUTH_SECRET="generate_a_random_string"
BETTER_AUTH_URL="http://localhost:3000"

# AWS Credentials (For Secrets Manager)
AWS_ACCESS_KEY_ID="your_access_key"
AWS_SECRET_ACCESS_KEY="your_secret_key"
AWS_REGION="us-east-1"

# Inngest (Flow Engine)
INNGEST_EVENT_KEY="local"
INNGEST_SIGNING_KEY="local"

# AI Configuration (For Magic Input)
OPENAI_API_KEY="sk-..."
```

### 3. AWS Configuration (Brief Guide)

* **RDS:** Create a PostgreSQL database in RDS. Ensure "Public Access" is enabled if running dev locally, or use an SSH tunnel.
* **IAM:** Create an IAM User with `SecretsManagerReadWrite` policy. Use these keys in your `.env`.

### 4. Database Migration

Push the Prisma schema to your RDS instance.

```bash
npx prisma migrate dev --name init
```

### 5. Run Development Server

This command runs both the Next.js app and the Inngest local dashboard.

```bash
npm run dev:all
```

* **App:** http://localhost:3000
* **Inngest Dev Server:** http://localhost:8288

---

## Directory Structure

The project is structured by feature rather than technology type, making it easier to scale.

```
src/
├── app/                        # Next.js App Router
├── components/                 # Shared Shadcn UI Components
├── lib/                        # Global Utilities (AWS, Encryption, DB)
├── server/                     # Backend Logic (tRPC)
│   ├── routers/
│   │   ├── ai-config.ts        # ✨ AI Auto-Configuration Router
│   │   ├── workflows.ts
│   │   └── credentials.ts
├── features/                   # Feature-Specific Logic
│   ├── editor/                 # The React Flow Canvas
│   ├── execution/              # The Execution Engine (Executors)
│   └── credentials/            # AWS Secrets Manager Logic
└── inngest/                    # Background Functions
```

---

## Security Model

### Credential Handling Flow

1. **User Input:** User enters an API Key in the UI.
2. **Transport:** Key is sent via HTTPS/tRPC to the Next.js API.
3. **Encryption:** The backend uses `aws-sdk` to create a new secret in AWS Secrets Manager.
4. **Reference Storage:** AWS returns an ARN (e.g., `arn:aws:secretsmanager:...`). Only this ARN is stored in Postgres.
5. **Execution:** When a workflow runs, the backend worker uses the ARN to fetch the decrypted key directly from AWS into memory. The key is discarded immediately after the API call.

---

## Contributing

We welcome contributions! Please fork the repository and submit a Pull Request. Ensure you run `npm run lint` before submitting.

---

## License

This project is licensed under the MIT License - see the LICENSE file for details.
