# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

FastGPT is an AI Agent construction platform that provides out-of-the-box data processing and model calling capabilities. It uses Flow visualization for workflow orchestration to implement complex application scenarios. The project is a full-stack web application built with NextJS, TypeScript, and MongoDB/PostgreSQL.

## Development Commands

### Setup and Installation
```bash
# Install all dependencies (monorepo)
pnpm i

# Give script execution permissions (Linux/Mac)
chmod -R +x ./scripts/

# For Node >= 20, use:
NODE_OPTIONS=--no-node-snapshot pnpm i
```

### Development
```bash
# Development (using Make - recommended)
make dev name=app

# Development (direct approach)
cd projects/app
pnpm dev

# Development for other projects
make dev name=sandbox
make dev name=mcp_server
```

### Building and Testing
```bash
# Run tests
pnpm test
pnpm test:workflow

# Lint code
pnpm lint

# Format code
pnpm format-code

# Build Docker image
make build name=app image=registry.cn-hangzhou.aliyuncs.com/fastgpt/fastgpt:v4.8.1

# Build with proxy (for China)
make build name=app image=my-image:tag proxy=taobao
```

### Code Generation and Utilities
```bash
# Generate theme typings for Chakra UI
pnpm gen:theme-typings

# Generate OpenAPI documentation
pnpm api:gen

# Generate LLM documentation
pnpm gen:llms

# Create i18n files
pnpm create:i18n

# Initialize icons
pnpm initIcon
```

## Architecture Overview

### Monorepo Structure

FastGPT uses a monorepo architecture with the following key structure:

- **`projects/app/`** - Main NextJS web application (primary development target)
- **`projects/sandbox/`** - Code execution sandbox service (NestJS)
- **`projects/mcp_server/`** - MCP (Model Context Protocol) server
- **`packages/global/`** - Shared TypeScript types and utilities
- **`packages/service/`** - Backend services and database operations
- **`packages/web/`** - Frontend components and utilities
- **`packages/templates/`** - Workflow templates

### Core System Components

#### 1. Workflow Engine (`packages/global/core/workflow/`)
The workflow system is the heart of FastGPT, enabling visual workflow construction:
- **Nodes**: Different types of processing units (AI, data, logic, I/O)
- **Runtime**: Execution engine for workflows
- **Templates**: Pre-built workflow patterns
- **Type System**: Comprehensive TypeScript definitions for workflow components

#### 2. AI Integration (`packages/service/core/ai/`)
Handles AI model interactions and configurations:
- **Providers**: Support for multiple LLM providers (OpenAI, Claude, local models)
- **Embedding**: Text embedding services
- **Rerank**: Document reranking capabilities
- **Functions**: AI-powered utilities like query extension

#### 3. Dataset System (`packages/service/core/dataset/`)
Knowledge base management:
- **Collections**: Document collections and chunking
- **Search**: Vector and hybrid search capabilities
- **Training**: Data processing and indexing
- **API Datasets**: External data source integration

#### 4. Permission System (`packages/support/permission/`)
Role-based access control:
- **Teams**: Multi-tenant organization support
- **Collaborators**: Resource sharing and permissions
- **Auth**: Authentication and authorization layers

### Frontend Architecture

The main app uses NextJS with:
- **Chakra UI**: Component library and theming
- **React Flow**: Workflow visual editor
- **i18next**: Internationalization (supports zh-CN, en, zh-Hant)
- **Zustand**: State management

### Backend Architecture

- **MongoDB/PostgreSQL**: Primary databases
- **Vector DB**: Milvus or PGVector for embeddings
- **Redis**: Caching and session management
- **BullMQ**: Job queue system

## Key Development Patterns

### Workspace Dependencies
The project uses pnpm workspaces. Shared packages are referenced as:
```json
"@fastgpt/global": "workspace:*",
"@fastgpt/service": "workspace:*",
"@fastgpt/web": "workspace:*"
```

### TypeScript Path Mapping
Important path aliases defined in `vitest.config.mts`:
```typescript
'@': 'projects/app/src',
'@fastgpt': 'packages',
'@test': 'test'
```

### Internationalization
- Use `useTranslation()` hook for dynamic content
- Static content uses `i18nT()` from `@fastgpt/web/i18n/utils`
- Translation files in `packages/web/i18n/`
- Format: `t('namespace:key')` (e.g., `t('common:close')`)

### API Structure
- **API Routes**: Located in `projects/app/src/pages/api/`
- **Core APIs**: Business logic in `packages/service/core/`
- **Support APIs**: Utility services in `packages/service/support/`

### Workflow Node Development
When creating custom workflow nodes:
1. Define node template in `packages/global/core/workflow/template/system/`
2. Implement execution logic in `packages/service/core/workflow/dispatch/`
3. Add UI components in `projects/app/src/components/core/`

### Testing
- Uses Vitest for unit and integration testing
- Test files: `test/cases/` and `projects/app/test/`
- MongoDB memory server for database tests
- Coverage reporting enabled

## Environment and Configuration

### Required Environment Variables
The app requires various environment variables for:
- Database connections (MongoDB, PostgreSQL, Redis)
- AI provider API keys
- Authentication secrets
- Vector database configuration

### Docker Development
Uses multi-stage Docker builds with support for:
- Development with hot reload
- Production optimization
- Proxy support for China (taobao registry)

## Important Notes

- **Node Version**: Requires Node.js >= 18.16.0
- **Package Manager**: Requires pnpm >= 9.0.0
- **Husky**: Git hooks for linting and formatting
- **ESLint**: Configured for TypeScript and Next.js
- **License**: FastGPT Open Source License (see LICENSE file)

## Common Workflow Development Tasks

1. **Adding New Node Types**: Extend the workflow system by adding templates and dispatch handlers
2. **AI Provider Integration**: Add new providers in `packages/service/core/ai/config/provider/`
3. **Database Schema Changes**: Update schemas in `packages/service/` and run migration scripts
4. **UI Components**: Use Chakra UI patterns and add to `packages/web/components/`
5. **API Endpoints**: Follow RESTful patterns and add proper authentication