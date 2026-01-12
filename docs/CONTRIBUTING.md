# Contributing to CompeteDexter

Thank you for your interest in contributing to CompeteDexter! This document provides guidelines and instructions for developers.

---

## Table of Contents

1. [Getting Started](#getting-started)
2. [Development Workflow](#development-workflow)
3. [Code Standards](#code-standards)
4. [Testing](#testing)
5. [Pull Request Process](#pull-request-process)
6. [Architecture Overview](#architecture-overview)

---

## Getting Started

### Prerequisites

- **Node.js**: v20+ (LTS)
- **Bun**: v1.0+ (recommended) or npm
- **PostgreSQL**: v14+
- **Redis**: v7+
- **Git**

### Environment Setup

1. **Clone the repository:**
```bash
git clone https://github.com/yourorg/competedexter.git
cd competedexter
```

2. **Install dependencies:**
```bash
bun install
# or: npm install
```

3. **Set up environment variables:**
```bash
cp .env.example .env
# Edit .env with your API keys and database credentials
```

4. **Set up the database:**
```bash
# Create database
createdb competedexter_dev

# Run migrations
bun run db:migrate

# Seed with sample data (optional)
bun run db:seed
```

5. **Start development servers:**
```bash
# Terminal 1: API server
bun run dev:api

# Terminal 2: Web app
bun run dev:web

# Terminal 3: Crawler workers
bun run dev:workers
```

6. **Verify setup:**
- API: http://localhost:3000
- Web: http://localhost:5173
- Health check: http://localhost:3000/health

---

## Development Workflow

### Branch Naming

- **Feature**: `feature/add-linkedin-integration`
- **Bug Fix**: `fix/query-streaming-error`
- **Documentation**: `docs/update-api-spec`
- **Refactor**: `refactor/improve-caching`

### Commit Messages

Follow [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<scope>): <description>

[optional body]

[optional footer]
```

**Types:**
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation changes
- `style`: Code style changes (formatting)
- `refactor`: Code refactoring
- `test`: Adding or updating tests
- `chore`: Maintenance tasks

**Examples:**
```bash
feat(api): add competitor comparison endpoint
fix(agent): resolve streaming response timeout
docs(readme): update installation instructions
refactor(crawler): improve error handling
test(query): add integration tests for agent execution
```

---

## Code Standards

### TypeScript

**Use strict TypeScript:**
```typescript
// ✅ Good
interface Competitor {
  id: string;
  name: string;
  domain: string;
}

async function getCompetitor(id: string): Promise<Competitor> {
  // implementation
}

// ❌ Bad
async function getCompetitor(id: any): Promise<any> {
  // implementation
}
```

**Prefer interfaces over types for objects:**
```typescript
// ✅ Good
interface User {
  id: string;
  email: string;
}

// ✅ Also good (for unions/intersections)
type Status = 'active' | 'inactive';

// ❌ Bad (use interface for object shapes)
type User = {
  id: string;
  email: string;
};
```

### Code Formatting

We use **Prettier** and **ESLint**:

```bash
# Format code
bun run format

# Lint code
bun run lint

# Fix linting issues
bun run lint:fix
```

**Prettier Configuration:**
```json
{
  "semi": true,
  "singleQuote": true,
  "trailingComma": "es5",
  "printWidth": 100,
  "tabWidth": 2
}
```

### Naming Conventions

**Variables & Functions:**
```typescript
// camelCase
const userName = 'John';
function getUserById(id: string) {}
```

**Classes & Interfaces:**
```typescript
// PascalCase
class UserService {}
interface QueryResponse {}
```

**Constants:**
```typescript
// UPPER_SNAKE_CASE
const MAX_RETRY_ATTEMPTS = 3;
const API_BASE_URL = 'https://api.example.com';
```

**Files:**
```
// kebab-case for files
user-service.ts
competitor-profile.tsx

// PascalCase for components
CompetitorCard.tsx
QueryInterface.tsx
```

---

## Testing

### Running Tests

```bash
# Run all tests
bun test

# Run tests in watch mode
bun test --watch

# Run specific test file
bun test src/agent/orchestrator.test.ts

# Run tests with coverage
bun test --coverage
```

### Writing Tests

**Unit Tests:**
```typescript
import { describe, it, expect, beforeEach } from '@jest/globals';
import { CompetitorService } from './competitor-service.js';

describe('CompetitorService', () => {
  let service: CompetitorService;

  beforeEach(() => {
    service = new CompetitorService();
  });

  it('should create a new competitor', async () => {
    const competitor = await service.create({
      name: 'Acme Corp',
      domain: 'acme.com',
    });

    expect(competitor.id).toBeDefined();
    expect(competitor.name).toBe('Acme Corp');
  });

  it('should throw error if domain is invalid', async () => {
    await expect(
      service.create({ name: 'Test', domain: 'invalid' })
    ).rejects.toThrow('Invalid domain');
  });
});
```

**Integration Tests:**
```typescript
import { describe, it, expect } from '@jest/globals';
import request from 'supertest';
import { app } from './app.js';

describe('POST /api/v1/competitors', () => {
  it('should create a new competitor', async () => {
    const response = await request(app)
      .post('/api/v1/competitors')
      .set('Authorization', `Bearer ${testToken}`)
      .send({
        name: 'Acme Corp',
        domain: 'acme.com',
      });

    expect(response.status).toBe(201);
    expect(response.body.name).toBe('Acme Corp');
  });
});
```

### Test Coverage

- **Target**: 80% overall coverage
- **Required**: 90% coverage for critical paths (agent, payment)
- **Check coverage**: `bun test --coverage`

---

## Pull Request Process

### Before Submitting

1. **Ensure all tests pass:**
```bash
bun test
bun run typecheck
bun run lint
```

2. **Update documentation:**
- Add/update JSDoc comments
- Update relevant .md files
- Update API spec if API changed

3. **Test manually:**
- Test the feature end-to-end
- Check for edge cases
- Verify error handling

### PR Template

```markdown
## Description
Brief description of changes

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Documentation update

## Testing
- [ ] Unit tests added/updated
- [ ] Integration tests added/updated
- [ ] Manual testing completed

## Screenshots (if applicable)
[Add screenshots]

## Checklist
- [ ] Code follows style guidelines
- [ ] Self-review completed
- [ ] Documentation updated
- [ ] Tests pass
- [ ] No new warnings
```

### Review Process

1. **Create PR**: Push branch and create PR on GitHub
2. **CI/CD**: Automated tests run
3. **Code Review**: At least 1 approval required
4. **Address Feedback**: Make requested changes
5. **Merge**: Squash and merge once approved

---

## Architecture Overview

### Project Structure

```
competedexter/
├── src/
│   ├── agent/          # Agent orchestration (from Dexter)
│   │   ├── orchestrator.ts
│   │   ├── phases/
│   │   └── prompts.ts
│   ├── api/            # REST API
│   │   ├── routes/
│   │   ├── middleware/
│   │   └── server.ts
│   ├── crawler/        # Data ingestion
│   │   ├── crawlers/
│   │   └── scheduler.ts
│   ├── database/       # Database layer
│   │   ├── models/
│   │   ├── migrations/
│   │   └── client.ts
│   ├── tools/          # Agent tools
│   │   └── finance/
│   └── web/            # React web app
│       ├── components/
│       ├── pages/
│       └── App.tsx
├── tests/
├── docs/
└── package.json
```

### Key Concepts

**Agent Architecture:**
- Adapted from Dexter (see `CLAUDE.md`)
- Five phases: Understand → Plan → Execute → Reflect → Answer
- Tools execute data gathering tasks
- Context caching prevents redundant API calls

**Data Flow:**
1. User asks query via web UI
2. API forwards to Agent Service
3. Agent executes research workflow
4. Results streamed back to user
5. Stored in database for future reference

**Data Ingestion:**
1. Crawler scheduler adds jobs to queue
2. Workers pick up jobs and execute
3. Data processed and stored in PostgreSQL
4. Alerts triggered based on rules

---

## Common Tasks

### Adding a New Tool

1. **Create tool file:**
```typescript
// src/tools/competitor/my-tool.ts
import { tool } from '@langchain/core/tools';
import { z } from 'zod';

export const myNewTool = tool(
  async ({ competitorId }) => {
    // Fetch data
    const data = await fetchData(competitorId);

    // Format result
    return formatToolResult(data, sourceUrls);
  },
  {
    name: 'my_new_tool',
    description: 'Clear description of what this tool does',
    schema: z.object({
      competitorId: z.string().describe('Competitor ID'),
    }),
  }
);
```

2. **Register tool:**
```typescript
// src/tools/index.ts
import { myNewTool } from './competitor/my-tool.js';

export const TOOLS = [
  // existing tools...
  myNewTool,
];
```

3. **Add tests:**
```typescript
// src/tools/competitor/my-tool.test.ts
describe('myNewTool', () => {
  it('should return formatted data', async () => {
    const result = await myNewTool.invoke({ competitorId: 'test' });
    expect(result).toContain('expected data');
  });
});
```

### Adding a New API Endpoint

1. **Define route:**
```typescript
// src/api/routes/competitors.ts
router.post('/competitors/:id/action', async (req, res) => {
  const { id } = req.params;
  const result = await competitorService.performAction(id, req.body);
  res.json(result);
});
```

2. **Add validation:**
```typescript
import { z } from 'zod';

const actionSchema = z.object({
  param: z.string(),
});

router.post('/competitors/:id/action', async (req, res) => {
  const validation = actionSchema.safeParse(req.body);
  if (!validation.success) {
    return res.status(400).json({ error: validation.error });
  }
  // handle request
});
```

3. **Update API docs:**
```markdown
// docs/specs/API_SPECIFICATION.md

### POST /competitors/{id}/action
Description of endpoint...
```

---

## Getting Help

- **Documentation**: Check `/docs` folder
- **Slack**: `#engineering` channel
- **GitHub Issues**: Open an issue for bugs
- **Email**: engineering@competedexter.com

---

## License

By contributing, you agree that your contributions will be licensed under the same license as the project.
