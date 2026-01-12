# Technical Architecture Specification
## CompeteDexter Platform

**Version**: 1.0
**Last Updated**: 2026-01-12
**Owner**: Engineering Team

---

## Architecture Overview

CompeteDexter is built on a microservices-inspired architecture with three main

 subsystems:

```
┌──────────────────────────────────────────────────────────────────────┐
│                         CLIENT LAYER                                 │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐                    │
│  │   Web App  │  │ Mobile App │  │  CLI Tool  │                    │
│  │  (React)   │  │(React Native)│  │(Node.js)  │                    │
│  └──────┬─────┘  └──────┬─────┘  └──────┬─────┘                    │
└─────────┼────────────────┼────────────────┼──────────────────────────┘
          │                │                │
          └────────────────┴────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────────┐
│                         API GATEWAY                                  │
│              (Authentication, Rate Limiting, Routing)                │
└──────────────────────────┬───────────────────────────────────────────┘
                           │
          ┌────────────────┼────────────────┐
          │                │                │
          ▼                ▼                ▼
┌─────────────────┐ ┌─────────────────┐ ┌──────────────────┐
│  Query Service  │ │  Data Service   │ │  Agent Service   │
│                 │ │                 │ │                  │
│ - Chat API      │ │ - Competitors   │ │ - Orchestrator   │
│ - History       │ │ - Data Ingestion│ │ - Task Executor  │
│ - Suggestions   │ │ - Data Access   │ │ - Tool Executor  │
└────────┬────────┘ └────────┬────────┘ └────────┬─────────┘
         │                   │                    │
         └───────────────────┴────────────────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
         ▼                   ▼                   ▼
┌────────────────┐  ┌────────────────┐  ┌────────────────┐
│   PostgreSQL   │  │     Redis      │  │   RabbitMQ     │
│  (Primary DB)  │  │    (Cache)     │  │  (Job Queue)   │
└────────────────┘  └────────────────┘  └────────────────┘
         │
         ▼
┌────────────────────────────────────────────────────────────┐
│                    DATA INGESTION LAYER                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │ Website  │  │   Jobs   │  │Financial │  │  Social  │  │
│  │ Crawler  │  │ Crawler  │  │ Crawler  │  │ Crawler  │  │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘  │
└────────────────────────────────────────────────────────────┘
```

### Key Architectural Principles

1. **Separation of Concerns**: Clear boundaries between query, data, and agent services
2. **Scalability**: Horizontal scaling at every layer
3. **Resilience**: Graceful degradation, retry logic, circuit breakers
4. **Observability**: Comprehensive logging, metrics, and tracing
5. **Security**: Zero-trust architecture, encryption everywhere
6. **Performance**: Aggressive caching, async processing, streaming responses

---

## System Components

### 1. API Gateway

**Technology**: Node.js + Express + Kong or AWS API Gateway

**Responsibilities:**
- Authentication & authorization (JWT tokens)
- Rate limiting (per user, per plan tier)
- Request routing to appropriate services
- Request/response transformation
- API versioning
- CORS handling
- Request logging

**Implementation:**

```typescript
// API Gateway Configuration
const gateway = {
  authentication: {
    type: 'jwt',
    secret: process.env.JWT_SECRET,
    expiry: '7d',
  },
  rateLimiting: {
    free: { requests: 20, window: '1 month' },
    starter: { requests: 1000, window: '1 month' },
    professional: { requests: 10000, window: '1 month' },
    enterprise: { requests: 'unlimited' },
  },
  routes: [
    { path: '/api/v1/query/*', service: 'query-service', auth: true },
    { path: '/api/v1/competitors/*', service: 'data-service', auth: true },
    { path: '/api/v1/data/*', service: 'data-service', auth: true },
  ],
};

// Rate Limiting Middleware
async function rateLimitMiddleware(req, res, next) {
  const user = req.user;
  const plan = user.subscription.plan;
  const limit = gateway.rateLimiting[plan];

  const usage = await redis.get(`ratelimit:${user.id}:${currentMonth}`);

  if (limit.requests !== 'unlimited' && usage >= limit.requests) {
    return res.status(429).json({
      error: 'Rate limit exceeded',
      limit: limit.requests,
      reset: startOfNextMonth(),
    });
  }

  await redis.incr(`ratelimit:${user.id}:${currentMonth}`);
  next();
}
```

---

### 2. Query Service

**Technology**: Node.js + Express + Dexter Agent Core

**Responsibilities:**
- Handle natural language queries from users
- Manage conversation context and history
- Stream responses back to clients
- Provide query suggestions
- Store query history and results

**API Endpoints:**

```typescript
// POST /api/v1/query/ask
interface AskQueryRequest {
  query: string;
  competitorIds?: string[]; // optional filter
  context?: {
    conversationId?: string;
    previousQueries?: string[];
  };
}

interface AskQueryResponse {
  queryId: string;
  answer: string; // streamed
  sources: Source[];
  confidence: number;
  taskResults: TaskResult[];
  metadata: {
    tokensUsed: number;
    executionTime: number;
    dataSourcesUsed: string[];
  };
}

// GET /api/v1/query/history
interface QueryHistoryResponse {
  queries: Array<{
    id: string;
    query: string;
    answer: string;
    timestamp: Date;
    competitorIds: string[];
  }>;
  pagination: {
    total: number;
    page: number;
    pageSize: number;
  };
}

// GET /api/v1/query/suggestions
interface QuerySuggestionsResponse {
  suggestions: Array<{
    query: string;
    category: string;
    description: string;
  }>;
}
```

**Agent Integration:**

```typescript
import { AgentOrchestrator } from '../agent/orchestrator.js';

class QueryService {
  private orchestrator: AgentOrchestrator;

  async handleQuery(req: Request, res: Response) {
    const { query, competitorIds, context } = req.body;
    const userId = req.user.id;

    // Set up streaming response
    res.setHeader('Content-Type', 'text/event-stream');
    res.setHeader('Cache-Control', 'no-cache');
    res.setHeader('Connection', 'keep-alive');

    // Create query record
    const queryRecord = await db.queries.create({
      userId,
      query,
      competitorIds,
      status: 'processing',
    });

    // Execute agent
    try {
      const result = await this.orchestrator.execute({
        query,
        competitorIds,
        messageHistory: context?.previousQueries || [],
        callbacks: {
          onPhaseStart: (phase) => {
            res.write(`data: ${JSON.stringify({ type: 'phase', phase })}\n\n`);
          },
          onTaskStart: (task) => {
            res.write(`data: ${JSON.stringify({ type: 'task_start', task })}\n\n`);
          },
          onTaskComplete: (task, result) => {
            res.write(`data: ${JSON.stringify({ type: 'task_complete', task, result })}\n\n`);
          },
          onAnswerChunk: (chunk) => {
            res.write(`data: ${JSON.stringify({ type: 'answer', chunk })}\n\n`);
          },
        },
      });

      // Update query record
      await db.queries.update(queryRecord.id, {
        status: 'completed',
        answer: result.answer,
        sources: result.sources,
        metadata: result.metadata,
      });

      // Send final event
      res.write(`data: ${JSON.stringify({ type: 'done', queryId: queryRecord.id })}\n\n`);
      res.end();
    } catch (error) {
      await db.queries.update(queryRecord.id, {
        status: 'failed',
        error: error.message,
      });

      res.write(`data: ${JSON.stringify({ type: 'error', error: error.message })}\n\n`);
      res.end();
    }
  }
}
```

---

### 3. Data Service

**Technology**: Node.js + Express + PostgreSQL

**Responsibilities:**
- CRUD operations for competitors
- Data ingestion orchestration
- Data access layer for agent
- Competitor profiles and metadata
- Historical data access

**Database Schema:**

```sql
-- Core Tables

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    name VARCHAR(255),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE subscriptions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    plan VARCHAR(50) NOT NULL, -- free, starter, professional, enterprise
    status VARCHAR(50) NOT NULL, -- active, cancelled, past_due
    current_period_start TIMESTAMP NOT NULL,
    current_period_end TIMESTAMP NOT NULL,
    stripe_subscription_id VARCHAR(255),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE workspaces (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    owner_id UUID REFERENCES users(id) ON DELETE CASCADE,
    settings JSONB DEFAULT '{}',
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE workspace_members (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID REFERENCES workspaces(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    role VARCHAR(50) NOT NULL, -- owner, admin, member, viewer
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(workspace_id, user_id)
);

CREATE TABLE competitors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID REFERENCES workspaces(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    domain VARCHAR(255),
    description TEXT,
    logo_url VARCHAR(500),
    tier VARCHAR(50) DEFAULT 'secondary', -- primary, secondary, emerging
    tags TEXT[] DEFAULT '{}',
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(workspace_id, domain)
);

CREATE INDEX idx_competitors_workspace ON competitors(workspace_id);
CREATE INDEX idx_competitors_tier ON competitors(tier);

-- Data Tables

CREATE TABLE competitor_snapshots (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    competitor_id UUID REFERENCES competitors(id) ON DELETE CASCADE,
    snapshot_type VARCHAR(100) NOT NULL, -- website, pricing, features, team_size, etc.
    data JSONB NOT NULL,
    snapshot_date TIMESTAMP DEFAULT NOW(),
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_snapshots_competitor ON competitor_snapshots(competitor_id);
CREATE INDEX idx_snapshots_type ON competitor_snapshots(snapshot_type);
CREATE INDEX idx_snapshots_date ON competitor_snapshots(snapshot_date);

CREATE TABLE data_sources (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    competitor_id UUID REFERENCES competitors(id) ON DELETE CASCADE,
    source_type VARCHAR(100) NOT NULL, -- website, jobs, funding, news, etc.
    source_url VARCHAR(500),
    last_crawled_at TIMESTAMP,
    last_success_at TIMESTAMP,
    crawl_frequency VARCHAR(50) DEFAULT 'daily', -- hourly, daily, weekly
    status VARCHAR(50) DEFAULT 'active', -- active, paused, failed
    error_count INT DEFAULT 0,
    config JSONB DEFAULT '{}',
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_data_sources_competitor ON data_sources(competitor_id);
CREATE INDEX idx_data_sources_type ON data_sources(source_type);

CREATE TABLE ingested_data (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_source_id UUID REFERENCES data_sources(id) ON DELETE CASCADE,
    competitor_id UUID REFERENCES competitors(id) ON DELETE CASCADE,
    data_type VARCHAR(100) NOT NULL,
    raw_data JSONB NOT NULL,
    processed_data JSONB,
    source_url VARCHAR(500),
    ingested_at TIMESTAMP DEFAULT NOW(),
    processed_at TIMESTAMP
);

CREATE INDEX idx_ingested_competitor ON ingested_data(competitor_id);
CREATE INDEX idx_ingested_type ON ingested_data(data_type);
CREATE INDEX idx_ingested_date ON ingested_data(ingested_at);

-- Tool Context (from Dexter architecture)

CREATE TABLE tool_contexts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    competitor_id UUID REFERENCES competitors(id) ON DELETE CASCADE,
    tool_name VARCHAR(255) NOT NULL,
    tool_args JSONB NOT NULL,
    args_hash VARCHAR(64) NOT NULL, -- SHA-256 hash of args for deduplication
    result JSONB NOT NULL,
    source_urls TEXT[],
    description TEXT,
    timestamp TIMESTAMP DEFAULT NOW(),
    UNIQUE(competitor_id, tool_name, args_hash)
);

CREATE INDEX idx_tool_contexts_competitor ON tool_contexts(competitor_id);
CREATE INDEX idx_tool_contexts_tool ON tool_contexts(tool_name);

-- Query History

CREATE TABLE queries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    workspace_id UUID REFERENCES workspaces(id) ON DELETE CASCADE,
    query_text TEXT NOT NULL,
    competitor_ids UUID[] DEFAULT '{}',
    status VARCHAR(50) NOT NULL, -- processing, completed, failed
    answer TEXT,
    sources JSONB,
    task_results JSONB,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMP DEFAULT NOW(),
    completed_at TIMESTAMP
);

CREATE INDEX idx_queries_user ON queries(user_id);
CREATE INDEX idx_queries_workspace ON queries(workspace_id);
CREATE INDEX idx_queries_created ON queries(created_at);

-- Alerts & Notifications

CREATE TABLE alerts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID REFERENCES workspaces(id) ON DELETE CASCADE,
    competitor_id UUID REFERENCES competitors(id) ON DELETE SET NULL,
    alert_type VARCHAR(100) NOT NULL,
    title VARCHAR(500) NOT NULL,
    description TEXT,
    data JSONB,
    severity VARCHAR(50) DEFAULT 'medium', -- low, medium, high, critical
    read BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_alerts_workspace ON alerts(workspace_id);
CREATE INDEX idx_alerts_competitor ON alerts(competitor_id);
CREATE INDEX idx_alerts_created ON alerts(created_at);
CREATE INDEX idx_alerts_unread ON alerts(workspace_id, read) WHERE read = FALSE;

CREATE TABLE alert_rules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID REFERENCES workspaces(id) ON DELETE CASCADE,
    competitor_id UUID REFERENCES competitors(id) ON DELETE CASCADE,
    rule_type VARCHAR(100) NOT NULL, -- funding, product_launch, pricing_change, etc.
    conditions JSONB NOT NULL,
    channels TEXT[] DEFAULT '{}', -- email, slack, in_app
    enabled BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);
```

**API Endpoints:**

```typescript
// Competitor Management
POST   /api/v1/competitors          // Create competitor
GET    /api/v1/competitors          // List competitors
GET    /api/v1/competitors/:id      // Get competitor details
PUT    /api/v1/competitors/:id      // Update competitor
DELETE /api/v1/competitors/:id      // Delete competitor

// Data Access
GET    /api/v1/competitors/:id/data              // Get all data for competitor
GET    /api/v1/competitors/:id/data/:type        // Get specific data type
GET    /api/v1/competitors/:id/timeline          // Get activity timeline
GET    /api/v1/competitors/:id/insights          // Get aggregated insights

// Alerts
GET    /api/v1/alerts                             // List alerts
POST   /api/v1/alerts/rules                       // Create alert rule
GET    /api/v1/alerts/rules                       // List alert rules
PUT    /api/v1/alerts/:id/read                    // Mark as read
```

---

### 4. Agent Service

**Technology**: Adapted from Dexter (Node.js + TypeScript + LangChain)

**Components:**
- Orchestrator: Manages agent execution flow
- Phase Executors: Understand, Plan, Execute, Reflect, Answer
- Tool Executor: Executes tools and manages caching
- Task Executor: Parallel task execution with dependencies

**See existing Dexter codebase** (`src/agent/`) **for detailed implementation**

Key adaptations for CompeteDexter:

1. **Modified Understanding Phase**: Extract competitors mentioned, not just financial entities
2. **Competitor-Aware Tool Selection**: Tools filtered by relevance to competitive intelligence
3. **Multi-Competitor Analysis**: Support comparing 3-10 competitors in single query
4. **Competitive Context**: Use tool context from multiple competitors

---

### 5. Data Ingestion Service

**Technology**: Node.js + BullMQ + Puppeteer

**Components:**
- Crawler Scheduler: Manages crawl schedules (cron)
- Crawler Workers: Execute crawls in parallel
- Data Processors: Transform raw data into structured format
- Quality Validators: Check data quality

**Architecture:**

```typescript
// Crawler Scheduler
import cron from 'node-cron';
import { Queue } from 'bullmq';

const crawlQueue = new Queue('crawl-jobs', {
  connection: redis,
});

// Schedule crawlers
cron.schedule('0 * * * *', async () => {
  // Hourly: Critical data sources
  const criticalSources = await db.dataSources.findMany({
    where: { crawl_frequency: 'hourly', status: 'active' },
  });

  for (const source of criticalSources) {
    await crawlQueue.add('crawl', {
      sourceId: source.id,
      competitorId: source.competitor_id,
      sourceType: source.source_type,
    });
  }
});

// Crawler Worker
import { Worker } from 'bullmq';

const worker = new Worker(
  'crawl-jobs',
  async (job) => {
    const { sourceId, competitorId, sourceType } = job.data;

    try {
      // Get appropriate crawler
      const crawler = getCrawlerForType(sourceType);

      // Execute crawl
      const data = await crawler.crawl(competitorId, sourceId);

      // Validate data quality
      const validation = await validateData(data, sourceType);

      if (validation.quality < 0.5) {
        throw new Error(`Data quality too low: ${validation.quality}`);
      }

      // Process and store
      await processAndStore(competitorId, sourceType, data);

      // Update source status
      await db.dataSources.update(sourceId, {
        last_crawled_at: new Date(),
        last_success_at: new Date(),
        error_count: 0,
      });

      // Check for significant changes and create alerts
      await checkForAlerts(competitorId, sourceType, data);

      return { success: true };
    } catch (error) {
      await db.dataSources.update(sourceId, {
        last_crawled_at: new Date(),
        error_count: db.literal('error_count + 1'),
      });

      throw error;
    }
  },
  {
    connection: redis,
    concurrency: 10,
  }
);
```

---

## Data Flow

### Query Flow

```
User Query
    │
    ▼
API Gateway (Auth + Rate Limit)
    │
    ▼
Query Service
    │
    ├──> Check Cache (Redis)
    │    └──> Return if fresh
    │
    ├──> Agent Orchestrator
    │         │
    │         ├──> Understand Phase
    │         │    └──> Extract: competitors, intent, entities
    │         │
    │         ├──> Plan Phase
    │         │    └──> Create task list
    │         │
    │         ├──> Execute Phase
    │         │    ├──> Fetch data from Data Service
    │         │    ├──> Execute tools (with caching)
    │         │    └──> Parallel task execution
    │         │
    │         ├──> Reflect Phase
    │         │    └──> Check completeness
    │         │
    │         └──> Answer Phase
    │              └──> Stream response
    │
    ├──> Save to Query History
    │
    └──> Return to User (streaming)
```

### Ingestion Flow

```
Crawler Scheduler
    │
    ├──> Schedule jobs (cron)
    │
    ▼
Crawl Queue (RabbitMQ)
    │
    ▼
Crawler Workers (parallel)
    │
    ├──> Fetch data (API / scrape)
    ├──> Validate quality
    ├──> Process & structure
    │
    ▼
Data Service
    │
    ├──> Store in PostgreSQL
    ├──> Update tool context cache
    ├──> Check alert rules
    │
    └──> Trigger alerts if needed
         └──> Notification Service
```

---

## Infrastructure

### Cloud Provider: AWS

**Compute:**
- ECS Fargate: Container orchestration
- EC2 (optional): For resource-intensive crawling

**Database:**
- RDS PostgreSQL: Primary database (Multi-AZ)
- ElastiCache Redis: Caching and session storage
- Amazon MQ (RabbitMQ): Message queue

**Storage:**
- S3: Screenshots, HTML snapshots, backups
- EFS: Shared file system (if needed)

**Networking:**
- VPC: Isolated network
- Application Load Balancer: Traffic distribution
- CloudFront: CDN for static assets
- Route 53: DNS management

**Security:**
- AWS WAF: Web application firewall
- Secrets Manager: API keys and credentials
- KMS: Encryption keys
- IAM: Access control

**Monitoring:**
- CloudWatch: Logs and metrics
- X-Ray: Distributed tracing
- CloudWatch Alarms: Automated alerts

### Deployment Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                         AWS CLOUD                           │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    Public Subnet                     │   │
│  │  ┌──────────────┐         ┌──────────────┐         │   │
│  │  │     ALB      │────────│  CloudFront  │         │   │
│  │  └──────┬───────┘         └──────────────┘         │   │
│  └─────────┼──────────────────────────────────────────┘   │
│            │                                               │
│  ┌─────────┼──────────────────────────────────────────┐   │
│  │         │         Private Subnet                    │   │
│  │         ▼                                           │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐   │   │
│  │  │   Query    │  │    Data    │  │   Agent    │   │   │
│  │  │  Service   │  │  Service   │  │  Service   │   │   │
│  │  │ (ECS x3)   │  │ (ECS x3)   │  │ (ECS x5)   │   │   │
│  │  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘   │   │
│  │        │               │               │          │   │
│  │        └───────────────┼───────────────┘          │   │
│  │                        │                          │   │
│  │  ┌────────────────────┼──────────────────────┐   │   │
│  │  │                    ▼                      │   │   │
│  │  │  ┌──────────┐  ┌──────────┐  ┌─────────┐ │   │   │
│  │  │  │PostgreSQL│  │  Redis   │  │RabbitMQ │ │   │   │
│  │  │  │   RDS    │  │ElastiCache│ │AmazonMQ │ │   │   │
│  │  │  └──────────┘  └──────────┘  └─────────┘ │   │   │
│  │  │                                           │   │   │
│  │  │          Data Layer (Private)            │   │   │
│  │  └───────────────────────────────────────────┘   │   │
│  │                                                   │   │
│  │  ┌─────────────────────────────────────────┐    │   │
│  │  │       Crawler Workers (ECS x10)          │    │   │
│  │  └─────────────────────────────────────────┘    │   │
│  └───────────────────────────────────────────────────┘   │
│                                                           │
│  ┌───────────────────────────────────────────────────┐   │
│  │              S3 (Data Lake)                       │   │
│  │  - HTML snapshots  - Screenshots  - Backups      │   │
│  └───────────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────────┘
```

### Scaling Strategy

**Horizontal Scaling:**
- Query Service: Auto-scale 2-10 instances based on request rate
- Agent Service: Auto-scale 5-20 instances based on queue depth
- Crawler Workers: Fixed 10 instances (can manually scale)

**Vertical Scaling:**
- Database: Start with db.t3.large, scale to db.r6g.xlarge
- Redis: Start with cache.t3.medium, scale to cache.r6g.large

**Auto-Scaling Rules:**

```yaml
query-service:
  min: 2
  max: 10
  metrics:
    - type: CPU
      target: 70%
    - type: RequestCount
      target: 1000 requests/minute

agent-service:
  min: 5
  max: 20
  metrics:
    - type: CPU
      target: 80%
    - type: QueueDepth
      target: 50 messages

crawler-workers:
  min: 10
  max: 20
  schedule:
    - cron: "0 0-6 * * *"  # Night hours (low traffic)
      desired: 20
    - cron: "0 7-23 * * *" # Day hours
      desired: 10
```

---

## Performance Optimization

### Caching Strategy

**Multi-Level Cache:**

```typescript
// L1: In-memory cache (Node.js)
const memoryCache = new Map();

// L2: Redis cache
const redisCache = new Redis(process.env.REDIS_URL);

// L3: Database (tool_contexts table)

async function getCachedData(key: string) {
  // Check L1
  if (memoryCache.has(key)) {
    return { data: memoryCache.get(key), source: 'memory' };
  }

  // Check L2
  const redisData = await redisCache.get(key);
  if (redisData) {
    memoryCache.set(key, JSON.parse(redisData)); // Populate L1
    return { data: JSON.parse(redisData), source: 'redis' };
  }

  // Check L3 (database)
  const dbData = await db.toolContexts.findOne({ where: { key } });
  if (dbData) {
    redisCache.setex(key, 3600, JSON.stringify(dbData)); // Populate L2
    memoryCache.set(key, dbData); // Populate L1
    return { data: dbData, source: 'database' };
  }

  return null;
}
```

**Cache Invalidation:**
- Time-based: Different TTLs for different data types
- Event-based: Invalidate when new data ingested
- Manual: Admin can force refresh

**TTLs by Data Type:**
```typescript
const cacheTTLs = {
  // Short TTL (frequent updates)
  news: 3600, // 1 hour
  social_media: 3600,
  job_postings: 7200, // 2 hours

  // Medium TTL
  website: 86400, // 24 hours
  reviews: 86400,
  funding: 86400,

  // Long TTL (slow-changing)
  patents: 604800, // 7 days
  tech_stack: 604800,
  company_info: 604800,
};
```

### Database Optimization

**Indexing Strategy:**
- B-tree indexes on foreign keys and frequently queried columns
- GIN indexes on JSONB columns for fast lookups
- Partial indexes for common query patterns

**Query Optimization:**
```sql
-- Example: Efficient competitor data fetching
CREATE INDEX idx_ingested_competitor_type_date ON ingested_data(competitor_id, data_type, ingested_at DESC);

-- Efficient query
SELECT * FROM ingested_data
WHERE competitor_id = $1
  AND data_type = $2
ORDER BY ingested_at DESC
LIMIT 1;
```

**Connection Pooling:**
```typescript
const pool = new Pool({
  host: process.env.DB_HOST,
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  max: 20, // Maximum pool size
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});
```

### LLM Cost Optimization

**Model Selection:**
- GPT-4o-mini for tool selection (fast, cheap)
- GPT-4o or Claude Sonnet for analysis (quality)
- Cache aggressively to avoid redundant API calls

**Prompt Optimization:**
- Keep prompts concise
- Use system prompts effectively
- Cache common prompt patterns

**Cost Monitoring:**
```typescript
async function trackLLMUsage(userId: string, tokens: number, model: string) {
  const cost = calculateCost(tokens, model);

  await db.llmUsage.create({
    user_id: userId,
    tokens,
    model,
    cost,
    timestamp: new Date(),
  });

  // Alert if user approaching limit
  const monthlyUsage = await getMonthlyUsage(userId);
  if (monthlyUsage.cost > userPlan.llmBudget * 0.8) {
    await sendUsageAlert(userId);
  }
}
```

---

## Security

### Authentication & Authorization

**JWT-based Authentication:**

```typescript
interface JWTPayload {
  userId: string;
  email: string;
  workspaceIds: string[];
  plan: string;
  iat: number;
  exp: number;
}

// Generate token
function generateToken(user: User): string {
  return jwt.sign(
    {
      userId: user.id,
      email: user.email,
      workspaceIds: user.workspaces.map(w => w.id),
      plan: user.subscription.plan,
    },
    process.env.JWT_SECRET,
    { expiresIn: '7d' }
  );
}

// Verify token
function verifyToken(token: string): JWTPayload {
  return jwt.verify(token, process.env.JWT_SECRET);
}
```

**Role-Based Access Control (RBAC):**

```typescript
const permissions = {
  viewer: ['read:competitors', 'read:queries'],
  member: ['read:competitors', 'write:queries', 'read:alerts'],
  admin: ['read:competitors', 'write:competitors', 'write:queries', 'read:alerts', 'write:alerts'],
  owner: ['*'], // All permissions
};

function checkPermission(user: User, action: string, workspaceId: string): boolean {
  const membership = user.workspaces.find(w => w.id === workspaceId);
  if (!membership) return false;

  const userPermissions = permissions[membership.role];
  return userPermissions.includes(action) || userPermissions.includes('*');
}
```

### Data Encryption

**At Rest:**
- PostgreSQL: Encryption enabled (AWS RDS)
- S3: Server-side encryption (SSE-S3)
- Secrets: AWS Secrets Manager

**In Transit:**
- TLS 1.3 for all API communication
- Certificate from AWS Certificate Manager

### API Security

**Rate Limiting:**
- Prevent abuse and DDoS
- Per-user limits based on plan
- Global limits for free tier

**Input Validation:**
```typescript
import { z } from 'zod';

const querySchema = z.object({
  query: z.string().min(3).max(1000),
  competitorIds: z.array(z.string().uuid()).optional(),
  context: z.object({
    conversationId: z.string().uuid().optional(),
  }).optional(),
});

app.post('/api/v1/query/ask', async (req, res) => {
  const validation = querySchema.safeParse(req.body);

  if (!validation.success) {
    return res.status(400).json({
      error: 'Invalid request',
      details: validation.error.format(),
    });
  }

  // Process valid request
});
```

**SQL Injection Prevention:**
- Use parameterized queries
- ORM (Prisma) with built-in protection

**XSS Prevention:**
- Sanitize user inputs
- Content Security Policy headers
- Escape output in responses

---

## Monitoring & Observability

### Logging

**Structured Logging:**

```typescript
import winston from 'winston';

const logger = winston.createLogger({
  format: winston.format.json(),
  defaultMeta: { service: 'query-service' },
  transports: [
    new winston.transports.Console(),
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' }),
  ],
});

// Usage
logger.info('Query started', {
  queryId: query.id,
  userId: user.id,
  query: query.text,
});

logger.error('Query failed', {
  queryId: query.id,
  error: error.message,
  stack: error.stack,
});
```

**Log Aggregation:**
- CloudWatch Logs for centralized logging
- Log retention: 30 days for all, 1 year for errors

### Metrics

**Key Metrics:**

```typescript
// Application Metrics
const metrics = {
  // Performance
  query_latency_ms: Histogram,
  agent_execution_time_ms: Histogram,
  crawler_success_rate: Gauge,

  // Business
  queries_per_day: Counter,
  active_users: Gauge,
  mrr: Gauge,

  // Infrastructure
  cpu_usage: Gauge,
  memory_usage: Gauge,
  db_connections: Gauge,
  queue_depth: Gauge,
};

// Example: Track query latency
const timer = metrics.query_latency_ms.startTimer();
await executeQuery();
timer();
```

**Dashboards:**
- Real-time query metrics
- Crawler health
- System resource utilization
- Business KPIs

### Alerting

**Alert Rules:**

```yaml
alerts:
  - name: High Error Rate
    condition: error_rate > 5%
    window: 5m
    severity: high
    channels: [pagerduty, slack]

  - name: Query Latency High
    condition: p95_latency > 30s
    window: 5m
    severity: medium
    channels: [slack]

  - name: Database Connections High
    condition: db_connections > 80% of max
    window: 2m
    severity: high
    channels: [pagerduty]

  - name: Crawler Failure Rate High
    condition: crawler_failure_rate > 20%
    window: 30m
    severity: medium
    channels: [slack]
```

---

## Disaster Recovery

### Backup Strategy

**Database Backups:**
- Automated daily backups (RDS)
- Retention: 30 days
- Cross-region replication for critical data

**S3 Backups:**
- Versioning enabled
- Lifecycle policies (move to Glacier after 90 days)

**Restore Testing:**
- Monthly restore drills
- Document restore procedures

### High Availability

**Multi-AZ Deployment:**
- RDS in Multi-AZ configuration
- ECS services in multiple AZs
- Load balancer across AZs

**Failover Strategy:**
- Automatic RDS failover (< 2 minutes)
- ECS service auto-restart on failure
- Circuit breakers for external dependencies

---

## Deployment

### CI/CD Pipeline

```yaml
# GitHub Actions Workflow
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: npm install
      - run: npm test
      - run: npm run typecheck

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: docker/build-push-action@v4
        with:
          push: true
          tags: competedexter/api:${{ github.sha }}

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to ECS
        run: |
          aws ecs update-service \
            --cluster production \
            --service api \
            --force-new-deployment
```

### Blue-Green Deployment

1. Deploy new version (green) alongside current (blue)
2. Route 10% of traffic to green
3. Monitor error rates and latency
4. Gradually increase traffic to green (25%, 50%, 100%)
5. Decommission blue after 24 hours

---

## Cost Estimation

### Monthly Costs (1,000 Competitors, 10,000 Queries/month)

**Infrastructure:**
- ECS (Query + Agent + Data): $600
- RDS PostgreSQL (db.r6g.large): $300
- Redis (cache.r6g.large): $200
- RabbitMQ (mq.m5.large): $250
- S3 Storage (5TB): $120
- Data Transfer: $200
- CloudWatch/X-Ray: $100
**Subtotal: $1,770**

**External Services:**
- OpenAI API (GPT-4o): $1,500
- Crunchbase API: $50
- BuiltWith API: $300
- NewsAPI: $150
- PhantomBuster: $200
**Subtotal: $2,200**

**Total Monthly Cost: ~$4,000**

**Cost per Query: $0.40**
**Target ARPU: $1,500/year = $125/month**
**Queries per User: ~50/month (if evenly distributed)**
**Gross Margin: ~85%**

---

## Future Enhancements

### Phase 2 (Months 6-12)
- GraphQL API
- Real-time collaboration (websockets)
- Mobile apps (iOS, Android)
- Advanced analytics dashboard
- Custom data source connectors

### Phase 3 (Year 2)
- ML-powered predictions
- Automated competitive strategy recommendations
- Integration marketplace
- White-label solution
- Multi-language support

---

**Document Version**: 1.0
**Last Updated**: 2026-01-12
**Next Review**: 2026-03-12
