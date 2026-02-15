# WorkFuse - Design Document

## 1. System Architecture

### 1.1 High-Level Architecture

WorkFuse follows a three-tier architecture separating concerns between presentation, business logic, and data persistence:

```
┌─────────────────────────────────────────────────────────────┐
│                     Client Layer (Browser)                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ React Flow   │  │ AI Config    │  │ WebSocket    │      │
│  │ Canvas       │  │ Helper       │  │ Client       │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
                            │
                    tRPC over HTTPS
                            │
┌─────────────────────────────────────────────────────────────┐
│              Application Layer (AWS Amplify)                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ Next.js      │  │ tRPC Router  │  │ Auth         │      │
│  │ API Routes   │  │              │  │ (Better Auth)│      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
                            │
              ┌─────────────┼─────────────┐
              │             │             │
┌─────────────▼──┐  ┌──────▼──────┐  ┌──▼──────────────┐
│ Amazon RDS     │  │ AWS Secrets │  │ Inngest Cloud   │
│ (PostgreSQL)   │  │ Manager     │  │ (Orchestration) │
└────────────────┘  └─────────────┘  └─────────────────┘
```

### 1.2 Component Responsibilities

#### Client Layer
- **React Flow Canvas:** Visual workflow editor with drag-and-drop
- **AI Config Helper:** Natural language to JSON configuration
- **WebSocket Client:** Real-time execution updates
- **State Management:** Jotai for client state, Nuqs for URL state

#### Application Layer
- **Next.js API Routes:** HTTP endpoints and server-side rendering
- **tRPC Router:** Type-safe API with automatic validation
- **Auth Module:** OAuth integration and session management
- **Workflow Service:** CRUD operations for workflows
- **Credential Service:** Secure credential management via AWS

#### Data Layer
- **PostgreSQL:** Workflow definitions, execution logs, user data
- **AWS Secrets Manager:** Encrypted credential storage
- **Inngest:** Durable workflow execution engine

## 2. Data Models

### 2.1 Database Schema (Prisma)

```prisma
model User {
  id            String       @id @default(cuid())
  email         String       @unique
  name          String?
  image         String?
  workflows     Workflow[]
  credentials   Credential[]
  createdAt     DateTime     @default(now())
  updatedAt     DateTime     @updatedAt
}

model Workflow {
  id            String       @id @default(cuid())
  name          String
  description   String?
  nodes         Json         // React Flow nodes
  edges         Json         // React Flow edges
  isActive      Boolean      @default(false)
  userId        String
  user          User         @relation(fields: [userId], references: [id])
  executions    Execution[]
  createdAt     DateTime     @default(now())
  updatedAt     DateTime     @updatedAt
  
  @@index([userId])
}

model Execution {
  id            String       @id @default(cuid())
  workflowId    String
  workflow      Workflow     @relation(fields: [workflowId], references: [id])
  status        ExecutionStatus
  startedAt     DateTime     @default(now())
  completedAt   DateTime?
  error         String?
  steps         ExecutionStep[]
  triggerData   Json?
  
  @@index([workflowId])
  @@index([status])
}

model ExecutionStep {
  id            String       @id @default(cuid())
  executionId   String
  execution     Execution    @relation(fields: [executionId], references: [id])
  nodeId        String
  nodeName      String
  status        StepStatus
  input         Json?
  output        Json?
  error         String?
  startedAt     DateTime     @default(now())
  completedAt   DateTime?
  
  @@index([executionId])
}

model Credential {
  id            String       @id @default(cuid())
  name          String
  type          CredentialType
  secretArn     String       // AWS Secrets Manager ARN
  userId        String
  user          User         @relation(fields: [userId], references: [id])
  createdAt     DateTime     @default(now())
  updatedAt     DateTime     @updatedAt
  
  @@index([userId])
}

enum ExecutionStatus {
  PENDING
  RUNNING
  COMPLETED
  FAILED
  CANCELLED
}

enum StepStatus {
  PENDING
  RUNNING
  SUCCESS
  FAILED
  SKIPPED
}

enum CredentialType {
  API_KEY
  OAUTH2
  BASIC_AUTH
  BEARER_TOKEN
}
```

### 2.2 Node Data Structure

```typescript
interface WorkflowNode {
  id: string;
  type: NodeType;
  position: { x: number; y: number };
  data: NodeData;
}

type NodeType = 
  | 'trigger.webhook'
  | 'trigger.schedule'
  | 'action.http'
  | 'action.ai'
  | 'logic.condition'
  | 'logic.loop'
  | 'utility.delay'
  | 'utility.transform';

interface NodeData {
  label: string;
  config: Record<string, any>;
  credentialId?: string;
}

interface WorkflowEdge {
  id: string;
  source: string;
  target: string;
  sourceHandle?: string;
  targetHandle?: string;
}
```

## 3. Core Modules

### 3.1 Workflow Editor Module

#### 3.1.1 Component Structure
```
features/editor/
├── components/
│   ├── Canvas.tsx              # Main React Flow canvas
│   ├── NodePalette.tsx         # Draggable node types
│   ├── NodeConfig.tsx          # Node configuration panel
│   ├── AIConfigInput.tsx       # Magic input component
│   └── nodes/
│       ├── TriggerNode.tsx
│       ├── ActionNode.tsx
│       ├── LogicNode.tsx
│       └── UtilityNode.tsx
├── hooks/
│   ├── useWorkflowEditor.ts    # Canvas state management
│   ├── useNodeValidation.ts    # Validation logic
│   └── useAIConfig.ts          # AI configuration
└── utils/
    ├── nodeFactory.ts          # Node creation
    ├── edgeValidation.ts       # Connection rules
    └── templateParser.ts       # Handlebars parsing
```

#### 3.1.2 State Management
```typescript
// Jotai atoms for editor state
const nodesAtom = atom<Node[]>([]);
const edgesAtom = atom<Edge[]>([]);
const selectedNodeAtom = atom<string | null>(null);
const isExecutingAtom = atom<boolean>(false);
```

### 3.2 AI Auto-Configuration Module

#### 3.2.1 Architecture
```
features/ai-config/
├── components/
│   ├── MagicInput.tsx          # Natural language input
│   ├── ConfigPreview.tsx       # Generated config display
│   └── FeedbackButton.tsx      # User feedback
├── services/
│   ├── intentParser.ts         # LLM integration
│   ├── schemaMapper.ts         # Zod schema mapping
│   └── configGenerator.ts      # JSON generation
└── prompts/
    ├── httpRequest.ts          # HTTP node prompts
    ├── aiAgent.ts              # AI node prompts
    └── transform.ts            # Transform node prompts
```

#### 3.2.2 AI Configuration Flow
```typescript
async function generateConfig(
  intent: string,
  nodeType: NodeType,
  schema: ZodSchema
): Promise<NodeConfig> {
  // 1. Build prompt with schema context
  const prompt = buildPrompt(intent, nodeType, schema);
  
  // 2. Call LLM (OpenAI/Anthropic/Gemini)
  const response = await ai.generateObject({
    model: 'gpt-4o',
    schema: schema,
    prompt: prompt
  });
  
  // 3. Validate against Zod schema
  const validated = schema.parse(response);
  
  // 4. Return structured config
  return validated;
}
```

### 3.3 Execution Engine Module

#### 3.3.1 Inngest Function Structure
```typescript
// inngest/workflows/execute.ts
export const executeWorkflow = inngest.createFunction(
  { id: 'workflow-execution' },
  { event: 'workflow/trigger' },
  async ({ event, step }) => {
    const { workflowId, triggerData } = event.data;
    
    // 1. Fetch workflow definition
    const workflow = await step.run('fetch-workflow', async () => {
      return db.workflow.findUnique({ where: { id: workflowId } });
    });
    
    // 2. Topological sort
    const sortedNodes = await step.run('sort-nodes', async () => {
      return topologicalSort(workflow.nodes, workflow.edges);
    });
    
    // 3. Execute each node
    for (const node of sortedNodes) {
      await step.run(`execute-${node.id}`, async () => {
        return executeNode(node, triggerData);
      });
    }
    
    return { status: 'completed' };
  }
);
```

#### 3.3.2 Node Executors
```
features/execution/
├── executors/
│   ├── BaseExecutor.ts         # Abstract executor
│   ├── HttpExecutor.ts         # HTTP requests
│   ├── AIExecutor.ts           # AI agent calls
│   ├── TransformExecutor.ts    # Data transformation
│   └── DelayExecutor.ts        # Sleep/delay
├── services/
│   ├── credentialResolver.ts   # AWS Secrets fetch
│   ├── templateEngine.ts       # Handlebars rendering
│   └── errorHandler.ts         # Error recovery
└── utils/
    ├── topologicalSort.ts      # DAG sorting
    └── dataFlow.ts             # Data passing
```

### 3.4 Credential Management Module

#### 3.4.1 Secure Storage Flow
```typescript
// features/credentials/services/credentialService.ts
export async function storeCredential(
  userId: string,
  name: string,
  type: CredentialType,
  value: string
): Promise<Credential> {
  // 1. Store in AWS Secrets Manager
  const secretArn = await awsSecretsManager.createSecret({
    Name: `workfuse/${userId}/${name}`,
    SecretString: value,
    KmsKeyId: process.env.AWS_KMS_KEY_ID
  });
  
  // 2. Store only ARN in database
  const credential = await db.credential.create({
    data: {
      name,
      type,
      secretArn,
      userId
    }
  });
  
  return credential;
}

export async function retrieveCredential(
  credentialId: string
): Promise<string> {
  // 1. Fetch ARN from database
  const credential = await db.credential.findUnique({
    where: { id: credentialId }
  });
  
  // 2. Retrieve secret value from AWS
  const secret = await awsSecretsManager.getSecretValue({
    SecretId: credential.secretArn
  });
  
  // 3. Return decrypted value (ephemeral)
  return secret.SecretString;
}
```

### 3.5 Real-Time Updates Module

#### 3.5.1 WebSocket Architecture
```typescript
// lib/websocket/server.ts
export class ExecutionUpdateServer {
  private wss: WebSocketServer;
  
  constructor() {
    this.wss = new WebSocketServer({ port: 3001 });
    this.setupHandlers();
  }
  
  broadcastUpdate(executionId: string, update: ExecutionUpdate) {
    const clients = this.getClientsForExecution(executionId);
    clients.forEach(client => {
      client.send(JSON.stringify(update));
    });
  }
}

// Client-side hook
export function useExecutionUpdates(executionId: string) {
  const [status, setStatus] = useState<ExecutionStatus>('PENDING');
  
  useEffect(() => {
    const ws = new WebSocket(`ws://localhost:3001/executions/${executionId}`);
    
    ws.onmessage = (event) => {
      const update = JSON.parse(event.data);
      setStatus(update.status);
    };
    
    return () => ws.close();
  }, [executionId]);
  
  return status;
}
```

## 4. API Design

### 4.1 tRPC Router Structure

```typescript
// server/routers/_app.ts
export const appRouter = router({
  workflows: workflowRouter,
  executions: executionRouter,
  credentials: credentialRouter,
  aiConfig: aiConfigRouter,
});

// server/routers/workflows.ts
export const workflowRouter = router({
  list: protectedProcedure
    .query(async ({ ctx }) => {
      return ctx.db.workflow.findMany({
        where: { userId: ctx.user.id }
      });
    }),
    
  create: protectedProcedure
    .input(z.object({
      name: z.string(),
      description: z.string().optional(),
      nodes: z.array(nodeSchema),
      edges: z.array(edgeSchema),
    }))
    .mutation(async ({ ctx, input }) => {
      return ctx.db.workflow.create({
        data: { ...input, userId: ctx.user.id }
      });
    }),
    
  update: protectedProcedure
    .input(z.object({
      id: z.string(),
      nodes: z.array(nodeSchema),
      edges: z.array(edgeSchema),
    }))
    .mutation(async ({ ctx, input }) => {
      return ctx.db.workflow.update({
        where: { id: input.id },
        data: { nodes: input.nodes, edges: input.edges }
      });
    }),
    
  delete: protectedProcedure
    .input(z.object({ id: z.string() }))
    .mutation(async ({ ctx, input }) => {
      return ctx.db.workflow.delete({
        where: { id: input.id }
      });
    }),
});
```

## 5. Security Design

### 5.1 Authentication Flow

```
1. User clicks "Login with GitHub"
2. Redirect to GitHub OAuth
3. GitHub redirects back with code
4. Exchange code for access token
5. Fetch user profile from GitHub
6. Create/update user in database
7. Generate session token (JWT)
8. Set secure HTTP-only cookie
9. Redirect to dashboard
```

### 5.2 Authorization Middleware

```typescript
// server/middleware/auth.ts
export const protectedProcedure = publicProcedure.use(async ({ ctx, next }) => {
  if (!ctx.session?.user) {
    throw new TRPCError({ code: 'UNAUTHORIZED' });
  }
  
  return next({
    ctx: {
      ...ctx,
      user: ctx.session.user,
    },
  });
});
```

### 5.3 Credential Security Layers

```
Layer 1: Transport Security (HTTPS/TLS)
Layer 2: Application Security (Session validation)
Layer 3: Storage Security (AWS Secrets Manager encryption)
Layer 4: Runtime Security (Ephemeral memory only)
Layer 5: Access Control (IAM policies)
```

## 6. UI/UX Design

### 6.1 Page Structure

```
/                           # Landing page
/dashboard                  # Workflow list
/workflows/new              # Create workflow
/workflows/[id]             # Edit workflow
/workflows/[id]/executions  # Execution history
/credentials                # Manage credentials
/settings                   # User settings
```

### 6.2 Workflow Editor Layout

```
┌─────────────────────────────────────────────────────────┐
│  WorkFuse                    [Save] [Test] [Deploy]     │
├──────────┬──────────────────────────────────────────────┤
│          │                                              │
│  Node    │                                              │
│  Palette │          React Flow Canvas                   │
│          │                                              │
│  [HTTP]  │                                              │
│  [AI]    │                                              │
│  [Logic] │                                              │
│  [Util]  │                                              │
│          │                                              │
├──────────┴──────────────────────────────────────────────┤
│  Node Configuration Panel                               │
│  ┌────────────────────────────────────────────────┐    │
│  │ Magic Input: "Send POST to Stripe..."         │    │
│  └────────────────────────────────────────────────┘    │
│  [Generated Config Preview]                            │
└─────────────────────────────────────────────────────────┘
```

### 6.3 Component Library (Shadcn/UI)

- Button, Input, Select, Textarea
- Dialog, Sheet, Popover
- Card, Badge, Alert
- Table, Tabs, Accordion
- Toast notifications
- Loading spinners

## 7. Deployment Architecture

### 7.1 AWS Infrastructure

```
┌─────────────────────────────────────────────────────────┐
│                      AWS Account                         │
│                                                          │
│  ┌────────────────┐  ┌────────────────┐                │
│  │  AWS Amplify   │  │  Amazon RDS    │                │
│  │  (Next.js App) │  │  (PostgreSQL)  │                │
│  └────────────────┘  └────────────────┘                │
│                                                          │
│  ┌────────────────┐  ┌────────────────┐                │
│  │ AWS Secrets    │  │  CloudWatch    │                │
│  │ Manager        │  │  (Logs)        │                │
│  └────────────────┘  └────────────────┘                │
│                                                          │
│  ┌────────────────┐                                     │
│  │  IAM Roles     │                                     │
│  │  & Policies    │                                     │
│  └────────────────┘                                     │
└─────────────────────────────────────────────────────────┘
```

### 7.2 CI/CD Pipeline

```
1. Developer pushes to GitHub
2. GitHub webhook triggers Amplify build
3. Amplify runs: npm install && npm run build
4. Amplify deploys to CDN
5. Database migrations run automatically
6. Health checks verify deployment
7. Rollback on failure
```

## 8. Performance Optimization

### 8.1 Frontend Optimization
- Code splitting by route
- React Flow virtualization for large graphs
- Debounced auto-save
- Optimistic UI updates
- Service worker for offline support

### 8.2 Backend Optimization
- Database connection pooling
- Query optimization with indexes
- tRPC batching for multiple requests
- Redis caching for frequently accessed data
- CDN for static assets

### 8.3 Execution Optimization
- Parallel node execution where possible
- Lazy loading of credentials
- Stream processing for large datasets
- Inngest step memoization

## 9. Monitoring & Observability

### 9.1 Metrics to Track
- Workflow execution duration
- Node execution success/failure rates
- API response times
- Database query performance
- WebSocket connection stability
- AI configuration accuracy

### 9.2 Logging Strategy
- Structured JSON logs
- Log levels: DEBUG, INFO, WARN, ERROR
- Correlation IDs for request tracing
- CloudWatch log aggregation
- Error tracking with Sentry

## 10. Testing Strategy

### 10.1 Unit Tests
- Node executors
- Template engine
- Validation logic
- Utility functions

### 10.2 Integration Tests
- tRPC endpoints
- Database operations
- AWS Secrets Manager integration
- Inngest function execution

### 10.3 E2E Tests
- Workflow creation flow
- Execution monitoring
- Credential management
- AI configuration

## 11. Future Considerations

### 11.1 Scalability
- Multi-region deployment
- Database sharding
- Horizontal scaling of workers
- CDN optimization

### 11.2 Features
- Workflow templates
- Custom node SDK
- Team collaboration
- Advanced debugging
- Workflow versioning
