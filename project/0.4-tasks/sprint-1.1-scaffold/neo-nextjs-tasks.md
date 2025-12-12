# Neo Next.js Tasks: Sprint 1.1 - Project Scaffold & Infrastructure

**Sprint**: 1.1 - Project Scaffold & Infrastructure
**Duration**: ~3.25 hours
**Role**: Lead Developer
**Agent**: Neo (Next.js Senior Developer)
**Support**: William (@william), Trinity (@trinity), Gordon (@gordon)
**Review**: Alex (@alex)

---

## Development Tools

### shadcn/ui MCP Server Reference

For component discovery and retrieval during development, use the **hx-shadcn MCP Server**:

| Property | Value |
|----------|-------|
| **Service Descriptor** | `project/0.8-references/hx-shadcn-service-descriptor.md` |
| **Host** | hx-shadcn.hx.dev.local (192.168.10.229) |
| **Port** | 7423 |
| **SSE Endpoint** | `http://hx-shadcn.hx.dev.local:7423/sse` |
| **Protocol** | SSE + MCP (Model Context Protocol) |
| **Status** | OPERATIONAL |

**Available MCP Tools:**
- `list_components` - List all available shadcn/ui components for React framework
- `get_component` - Retrieve component source code with dependencies
- `get_component_demo` - Get usage examples and demo code
- `get_component_metadata` - Get props, installation instructions, and documentation

**Example MCP Component Retrieval:**
```json
{
  "method": "tools/call",
  "params": {
    "name": "get_component",
    "arguments": {"componentName": "button"}
  }
}
```

**Note:** While the standard CLI (`npx shadcn@latest add <component>`) remains valid for installation, the MCP server provides programmatic access for component discovery, source code retrieval, and usage examples. Coordinate with Gordon (@gordon) for MCP-assisted component work.

---

## Overview

Initialize the Next.js 16 project with App Router, configure shadcn/ui components, set up Prisma schema, configure development environment, establish test infrastructure, and implement dark mode support.

---

## Tasks

### NEO-1.1-001: Create Next.js 16 Project with TypeScript

**Priority**: P0 (Critical)
**Effort**: 20 minutes
**Dependencies**: Sprint 0 Prerequisites Complete

**Description**:
Initialize a new Next.js 16 project with TypeScript, App Router, and strict mode enabled. Configure for the HX-Infrastructure ecosystem.

**Acceptance Criteria**:
- [ ] Project created at `/home/agent0/hx-docling-ui/`
- [ ] Next.js 16 with App Router (not Pages Router)
- [ ] TypeScript strict mode enabled in `tsconfig.json`
- [ ] `npm run dev` starts development server on port 3000
- [ ] Base directory structure created: `src/app/`, `src/components/`, `src/lib/`

**Deliverables**:
- `/home/agent0/hx-docling-ui/package.json`
- `/home/agent0/hx-docling-ui/tsconfig.json`
- `/home/agent0/hx-docling-ui/next.config.mjs`

**Technical Notes**:
```bash
npx create-next-app@latest hx-docling-ui \
  --typescript \
  --eslint \
  --tailwind \
  --app \
  --src-dir \
  --import-alias "@/*"
```

---

### NEO-1.1-002: Configure Package Dependencies

**Priority**: P0 (Critical)
**Effort**: 20 minutes
**Dependencies**: NEO-1.1-001

**Description**:
Install all required dependencies per the comprehensive dependency inventory in the implementation plan (Section 13). Ensure all 25+ packages are installed with compatible versions.

**Acceptance Criteria**:
- [ ] All production dependencies installed without errors
- [ ] All development dependencies installed without errors
- [ ] `npm ci` completes successfully
- [ ] No security vulnerabilities in `npm audit`
- [ ] Package versions match specification

**Deliverables**:
- `/home/agent0/hx-docling-ui/package.json` (updated with all dependencies)
- `/home/agent0/hx-docling-ui/package-lock.json`

**Technical Notes**:
Core dependencies to install:
```json
{
  "dependencies": {
    "next": "^15.0.0",
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "@prisma/client": "^5.22.0",
    "ioredis": "^5.4.1",
    "zod": "^3.23.8",
    "zustand": "^5.0.1",
    "react-hook-form": "^7.53.2",
    "@hookform/resolvers": "^3.9.1",
    "react-dropzone": "^14.3.5",
    "react-markdown": "^9.0.1",
    "remark-gfm": "^4.0.0",
    "next-themes": "^0.4.3",
    "lucide-react": "^0.460.0",
    "class-variance-authority": "^0.7.1",
    "clsx": "^2.1.1",
    "tailwind-merge": "^2.5.5"
  },
  "devDependencies": {
    "typescript": "^5.6.3",
    "prisma": "^5.22.0",
    "vitest": "^2.1.5",
    "@vitejs/plugin-react": "^4.3.4",
    "@testing-library/react": "^16.0.1",
    "@testing-library/jest-dom": "^6.6.3",
    "@testing-library/user-event": "^14.5.2",
    "playwright": "^1.49.0",
    "@playwright/test": "^1.49.0",
    "msw": "^2.6.5",
    "tailwindcss": "^3.4.15",
    "postcss": "^8.4.49",
    "autoprefixer": "^10.4.20",
    "@types/node": "^22.10.1",
    "@types/react": "^19.0.1",
    "@types/react-dom": "^19.0.1",
    "eslint": "^9.15.0",
    "eslint-config-next": "^15.0.3"
  }
}
```

---

### NEO-1.1-003: Initialize shadcn/ui

**Priority**: P0 (Critical)
**Effort**: 15 minutes
**Dependencies**: NEO-1.1-002

**Description**:
Initialize shadcn/ui with the default theme configuration. Configure CSS variables for light and dark mode support.

**Acceptance Criteria**:
- [ ] shadcn/ui initialized successfully
- [ ] `components.json` configured correctly
- [ ] CSS variables for theming in `src/app/globals.css`
- [ ] Tailwind config updated with shadcn/ui settings
- [ ] `cn()` utility function available at `src/lib/utils.ts`

**Deliverables**:
- `/home/agent0/hx-docling-ui/components.json`
- `/home/agent0/hx-docling-ui/src/lib/utils.ts`
- `/home/agent0/hx-docling-ui/src/app/globals.css` (updated)

**Technical Notes**:
```bash
npx shadcn@latest init
# Select: New York style, Slate base color, CSS variables: yes
```

---

### NEO-1.1-004: Pre-install shadcn/ui Components

**Priority**: P1 (High)
**Effort**: 30 minutes
**Dependencies**: NEO-1.1-003

**Description**:
Pre-install all 11 required shadcn/ui components to ensure consistency and availability throughout development.

**Acceptance Criteria**:
- [ ] Button component installed and functional
- [ ] Input component installed and functional
- [ ] Card component installed and functional
- [ ] Tabs component installed and functional
- [ ] Form component installed and functional
- [ ] Dialog component installed and functional
- [ ] Select component installed and functional
- [ ] Toast component (Sonner) installed and functional
- [ ] Skeleton component installed and functional
- [ ] Table component installed and functional
- [ ] Pagination component installed and functional
- [ ] All components render without errors

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/components/ui/button.tsx`
- `/home/agent0/hx-docling-ui/src/components/ui/input.tsx`
- `/home/agent0/hx-docling-ui/src/components/ui/card.tsx`
- `/home/agent0/hx-docling-ui/src/components/ui/tabs.tsx`
- `/home/agent0/hx-docling-ui/src/components/ui/form.tsx`
- `/home/agent0/hx-docling-ui/src/components/ui/dialog.tsx`
- `/home/agent0/hx-docling-ui/src/components/ui/select.tsx`
- `/home/agent0/hx-docling-ui/src/components/ui/sonner.tsx`
- `/home/agent0/hx-docling-ui/src/components/ui/skeleton.tsx`
- `/home/agent0/hx-docling-ui/src/components/ui/table.tsx`
- `/home/agent0/hx-docling-ui/src/components/ui/pagination.tsx`

**Technical Notes**:
```bash
npx shadcn@latest add button input card tabs form dialog select sonner skeleton table pagination
```

---

### NEO-1.1-005: Create Prisma Schema

**Priority**: P0 (Critical)
**Effort**: 30 minutes
**Dependencies**: NEO-1.1-002

**Description**:
Create the Prisma schema with Job and Result models, including all enums and indexes per the specification.

**Acceptance Criteria**:
- [ ] Job model with all required fields (id, sessionId, status, inputType, etc.)
- [ ] Result model with format and content fields
- [ ] JobStatus enum with all 10 states (PENDING, UPLOADING, PROCESSING, RETRY_1, RETRY_2, RETRY_3, COMPLETE, PARTIAL_COMPLETE, CANCELLED, ERROR)
- [ ] InputType enum (FILE, URL)
- [ ] ResultFormat enum (MARKDOWN, HTML, JSON, RAW)
- [ ] Database indexes on sessionId, status, createdAt
- [ ] `npx prisma generate` succeeds

**Deliverables**:
- `/home/agent0/hx-docling-ui/prisma/schema.prisma`

**Technical Notes**:
```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider  = "postgresql"
  url       = env("DATABASE_URL")
  directUrl = env("DIRECT_DATABASE_URL")
}

// ============================================================================
// Enums
// ============================================================================

enum JobStatus {
  PENDING
  UPLOADING
  PROCESSING
  RETRY_1
  RETRY_2
  RETRY_3
  COMPLETE
  PARTIAL_COMPLETE
  CANCELLED
  ERROR
}

enum InputType {
  FILE
  URL
}

enum ResultFormat {
  MARKDOWN
  HTML
  JSON
  RAW
}

// ============================================================================
// Models
// ============================================================================

model Job {
  id             String    @id @default(uuid()) @db.Uuid
  sessionId      String    @db.VarChar(64)
  status         JobStatus @default(PENDING)
  inputType      InputType
  fileName       String?   @db.VarChar(255)
  fileSize       Int?
  filePath       String?   @db.VarChar(512)
  mimeType       String?   @db.VarChar(127)
  url            String?   @db.VarChar(2048)
  currentStage   String?   @db.VarChar(64)
  currentPercent Int?      @db.SmallInt
  currentMessage String?   @db.Text
  checkpointData Json?     @db.JsonB
  createdAt      DateTime  @default(now()) @db.Timestamptz
  updatedAt      DateTime  @updatedAt @db.Timestamptz
  completedAt    DateTime? @db.Timestamptz
  error          String?   @db.Text
  errorCode      String?   @db.VarChar(64)
  retryCount     Int       @default(0) @db.SmallInt
  results        Result[]

  @@index([sessionId])
  @@index([status, createdAt(sort: Desc)])
  @@index([completedAt])
  @@index([sessionId, status])
  @@index([sessionId, createdAt])
  @@map("jobs")
}

model Result {
  id        String       @id @default(uuid()) @db.Uuid
  jobId     String       @db.Uuid
  format    ResultFormat
  content   Json         @db.JsonB
  mimeType  String?      @db.VarChar(127)
  fileName  String?      @db.VarChar(255)
  fileSize  Int?
  createdAt DateTime     @default(now()) @db.Timestamptz
  updatedAt DateTime     @updatedAt @db.Timestamptz

  // Relations
  job Job @relation(fields: [jobId], references: [id], onDelete: Cascade)

  @@index([jobId])
  @@index([format])
  @@index([jobId, format])
  @@index([createdAt])
  @@map("results")
}
```

---

### NEO-1.1-006: Configure Environment Files

**Priority**: P1 (High)
**Effort**: 15 minutes
**Dependencies**: NEO-1.1-001

**Description**:
Create environment file templates with all required variables for database, Redis, MCP, and file storage configuration.

**Acceptance Criteria**:
- [ ] `.env.example` template with all variables documented
- [ ] `.env.local` created from template (gitignored)
- [ ] Variables include: DATABASE_URL, DIRECT_DATABASE_URL, REDIS_URL, MCP_SERVER_URL, UPLOAD_PATH
- [ ] SSL paths configured for PostgreSQL and Redis
- [ ] Next.js public environment variables prefixed with NEXT_PUBLIC_

**Deliverables**:
- `/home/agent0/hx-docling-ui/.env.example`
- `/home/agent0/hx-docling-ui/.env.local`
- `/home/agent0/hx-docling-ui/.gitignore` (updated)

**Technical Notes**:
```env
# Database (PostgreSQL via PgBouncer)
DATABASE_URL="postgresql://docling_app:password@hx-postgres-server:6432/docling_db?sslmode=require"
DIRECT_DATABASE_URL="postgresql://docling_app:password@hx-postgres-server:5432/docling_db?sslmode=require"

# Redis
REDIS_URL="rediss://hx-redis-server:6379"

# MCP Server
MCP_SERVER_URL="http://hx-docling-server:8000"

# File Storage
UPLOAD_PATH="/data/docling-uploads"

# Session
SESSION_SECRET="your-session-secret-32-chars-min"

# Application
NEXT_PUBLIC_APP_URL="http://hx-cc-server.hx.dev.local:3000"
```

---

### NEO-1.1-007: Set Up Directory Structure

**Priority**: P1 (High)
**Effort**: 20 minutes
**Dependencies**: NEO-1.1-003

**Description**:
Create the complete directory structure per the charter and architecture specifications.

**Acceptance Criteria**:
- [ ] `src/app/` - App Router pages and API routes
- [ ] `src/components/` - UI components (upload, processing, results, history, shared)
- [ ] `src/lib/` - Utilities (db, redis, mcp, sse, validation)
- [ ] `src/stores/` - Zustand stores
- [ ] `src/hooks/` - Custom React hooks
- [ ] `src/types/` - TypeScript type definitions
- [ ] `test/` - Test directories (e2e, unit, fixtures)
- [ ] All directories have appropriate `.gitkeep` or index files

**Deliverables**:
- Complete directory structure with placeholder files
- `/home/agent0/hx-docling-ui/src/types/index.ts` (shared types)

**Technical Notes**:
```
src/
  app/
    api/v1/
      health/
      upload/
      process/
      jobs/
      history/
    history/
    layout.tsx
    page.tsx
  components/
    ui/          # shadcn components
    upload/
    processing/
    results/
    history/
    shared/
  lib/
    db/
    redis/
    mcp/
    sse/
    validation/
    utils/
  stores/
  hooks/
  types/
test/
  e2e/
  fixtures/
```

---

### NEO-1.1-008: Configure ESLint and Prettier

**Priority**: P2 (Medium)
**Effort**: 10 minutes
**Dependencies**: NEO-1.1-002

**Description**:
Configure ESLint with strict TypeScript rules and Prettier for consistent code formatting.

**Acceptance Criteria**:
- [ ] ESLint configured with eslint-config-next
- [ ] TypeScript strict rules enabled
- [ ] Prettier integrated with ESLint
- [ ] `npm run lint` passes on initial codebase
- [ ] Pre-commit hooks considered (optional for Phase 1)

**Deliverables**:
- `/home/agent0/hx-docling-ui/eslint.config.mjs`
- `/home/agent0/hx-docling-ui/.prettierrc`

**Technical Notes**:
```javascript
// eslint.config.mjs
import { dirname } from "path";
import { fileURLToPath } from "url";
import { FlatCompat } from "@eslint/eslintrc";

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);

const compat = new FlatCompat({
  baseDirectory: __dirname,
});

const eslintConfig = [
  ...compat.extends("next/core-web-vitals", "next/typescript"),
  {
    rules: {
      "@typescript-eslint/no-explicit-any": "error",
      "@typescript-eslint/explicit-function-return-type": "warn",
    },
  },
];

export default eslintConfig;
```

---

### NEO-1.1-009: Add Base Layout Components

**Priority**: P1 (High)
**Effort**: 20 minutes
**Dependencies**: NEO-1.1-004

**Description**:
Create the base application layout with Header and Footer components, navigation structure, and responsive design.

**Acceptance Criteria**:
- [ ] Root layout (`src/app/layout.tsx`) configured with providers
- [ ] Header component with navigation links
- [ ] Footer component with version/copyright
- [ ] Responsive navigation (mobile menu)
- [ ] Navigation links: Home, History
- [ ] Accessibility: semantic HTML, skip-to-content link

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/app/layout.tsx`
- `/home/agent0/hx-docling-ui/src/components/shared/Header.tsx`
- `/home/agent0/hx-docling-ui/src/components/shared/Footer.tsx`
- `/home/agent0/hx-docling-ui/src/components/shared/Navigation.tsx`

**Technical Notes**:
- Use Server Components for layout
- Header is a Client Component (for mobile menu state)
- Include ThemeProvider wrapper (for dark mode)

---

### NEO-1.1-010: Set Up Vitest with React Testing Library

**Priority**: P1 (High)
**Effort**: 20 minutes
**Dependencies**: NEO-1.1-002

**Description**:
Configure Vitest for unit testing with React Testing Library and jsdom environment.

**Acceptance Criteria**:
- [ ] Vitest configured with jsdom environment
- [ ] React Testing Library installed and configured
- [ ] Test setup file with jest-dom matchers
- [ ] `npm run test` executes test runner
- [ ] Coverage reporting configured (target: 80% lines)
- [ ] Test watch mode available (`npm run test:watch`)

**Deliverables**:
- `/home/agent0/hx-docling-ui/vitest.config.ts`
- `/home/agent0/hx-docling-ui/test/setup.ts`

**Technical Notes**:
```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: ['./test/setup.ts'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
      thresholds: {
        lines: 80,
        branches: 75,
      },
    },
  },
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
});
```

---

### NEO-1.1-011: Set Up Playwright for E2E Testing

**Priority**: P1 (High)
**Effort**: 15 minutes
**Dependencies**: NEO-1.1-010

**Description**:
Configure Playwright for end-to-end testing with proper test directories and base configuration.

**Acceptance Criteria**:
- [ ] Playwright configured for Chromium (primary), Firefox, WebKit (secondary)
- [ ] Test directory: `test/e2e/`
- [ ] Base URL configured for local development
- [ ] `npm run test:e2e` starts Playwright tests
- [ ] Screenshots and traces configured for failures

**Deliverables**:
- `/home/agent0/hx-docling-ui/playwright.config.ts`
- `/home/agent0/hx-docling-ui/test/e2e/.gitkeep`

**Technical Notes**:
```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './test/e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: 'html',
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
  ],
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
});
```

---

### NEO-1.1-012: Set Up MSW for API Mocking

**Priority**: P1 (High)
**Effort**: 15 minutes
**Dependencies**: NEO-1.1-010

**Description**:
Configure Mock Service Worker (MSW) for API mocking in tests, with initial handlers for MCP mock responses.

**Acceptance Criteria**:
- [ ] MSW installed and configured
- [ ] Browser and Node handlers set up
- [ ] Initial MCP mock handlers defined
- [ ] Health endpoint mock handler
- [ ] MSW integrates with Vitest setup

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/mocks/handlers.ts`
- `/home/agent0/hx-docling-ui/src/mocks/server.ts`
- `/home/agent0/hx-docling-ui/src/mocks/browser.ts`

**Technical Notes**:
```typescript
// src/mocks/handlers.ts
import { http, HttpResponse } from 'msw';

export const handlers = [
  // Health check mock
  http.get('/api/v1/health', () => {
    return HttpResponse.json({
      status: 'healthy',
      timestamp: new Date().toISOString(),
      version: '1.0.0',
      checks: {
        mcp: { status: 'ok', latency: 45 },
        postgres: { status: 'ok', latency: 12 },
        redis: { status: 'ok', latency: 5 },
      },
    });
  }),

  // MCP tool mock
  http.post('http://hx-docling-server:8000/jsonrpc', () => {
    return HttpResponse.json({
      jsonrpc: '2.0',
      id: 1,
      result: { success: true },
    });
  }),
];
```

---

### NEO-1.1-013: Configure Dark Mode with next-themes

**Priority**: P1 (High)
**Effort**: 15 minutes
**Dependencies**: NEO-1.1-004

**Description**:
Implement dark mode support using next-themes with system preference detection and manual toggle.

**Acceptance Criteria**:
- [ ] ThemeProvider wraps application in layout
- [ ] System theme preference detected
- [ ] Theme persisted to localStorage
- [ ] Theme toggle component available
- [ ] No hydration mismatch warnings
- [ ] CSS variables switch between light/dark themes

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/components/shared/ThemeProvider.tsx`
- `/home/agent0/hx-docling-ui/src/components/shared/ThemeToggle.tsx`

**Technical Notes**:
```typescript
// src/components/shared/ThemeProvider.tsx
'use client';

import { ThemeProvider as NextThemesProvider } from 'next-themes';

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  return (
    <NextThemesProvider
      attribute="class"
      defaultTheme="system"
      enableSystem
      disableTransitionOnChange
    >
      {children}
    </NextThemesProvider>
  );
}
```

---

### NEO-1.1-014: Configure Tailwind for Dark Mode

**Priority**: P1 (High)
**Effort**: 10 minutes
**Dependencies**: NEO-1.1-013

**Description**:
Update Tailwind configuration to support class-based dark mode switching.

**Acceptance Criteria**:
- [ ] `darkMode: 'class'` configured in Tailwind
- [ ] Dark mode CSS variables defined in globals.css
- [ ] Dark mode utility classes available (e.g., `dark:bg-gray-900`)
- [ ] All shadcn/ui components support dark mode

**Deliverables**:
- `/home/agent0/hx-docling-ui/tailwind.config.ts` (updated)
- `/home/agent0/hx-docling-ui/src/app/globals.css` (updated)

**Technical Notes**:
```typescript
// tailwind.config.ts
import type { Config } from 'tailwindcss';

const config: Config = {
  darkMode: 'class',
  content: [
    './src/**/*.{js,ts,jsx,tsx,mdx}',
  ],
  theme: {
    extend: {
      // Custom theme extensions
    },
  },
  plugins: [require('tailwindcss-animate')],
};

export default config;
```

---

### NEO-1.1-015: Customize Tailwind Theme

**Priority**: P2 (Medium)
**Effort**: 15 minutes
**Dependencies**: NEO-1.1-014

**Description**:
Customize Tailwind theme with brand colors, custom spacing, and typography per design specifications.

**Acceptance Criteria**:
- [ ] Brand primary color defined (HX blue)
- [ ] Custom spacing scale extended
- [ ] Custom font family configured
- [ ] Border radius tokens aligned with shadcn
- [ ] Colors documented in CSS variables

**Deliverables**:
- `/home/agent0/hx-docling-ui/tailwind.config.ts` (extended)
- `/home/agent0/hx-docling-ui/src/app/globals.css` (custom properties)

**Technical Notes**:
```typescript
theme: {
  extend: {
    colors: {
      primary: {
        DEFAULT: 'hsl(var(--primary))',
        foreground: 'hsl(var(--primary-foreground))',
      },
      // HX brand colors
      hx: {
        blue: '#0066CC',
        dark: '#1a1a2e',
      },
    },
    fontFamily: {
      sans: ['Inter', 'system-ui', 'sans-serif'],
      mono: ['JetBrains Mono', 'monospace'],
    },
  },
},
```

---

### NEO-1.1-016: Write Example Unit Test

**Priority**: P2 (Medium)
**Effort**: 10 minutes
**Dependencies**: NEO-1.1-010

**Description**:
Create an example unit test to verify the test infrastructure is working correctly.

**Acceptance Criteria**:
- [ ] Example test file created
- [ ] Test imports work correctly
- [ ] `npm run test` passes
- [ ] Coverage report generates
- [ ] Test demonstrates component testing pattern

**Deliverables**:
- `/home/agent0/hx-docling-ui/src/components/ui/button.test.tsx`

**Technical Notes**:
```typescript
// src/components/ui/button.test.tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { describe, it, expect, vi } from 'vitest';
import { Button } from './button';

describe('Button', () => {
  it('renders with text', () => {
    render(<Button>Click me</Button>);
    expect(screen.getByRole('button', { name: /click me/i })).toBeInTheDocument();
  });

  it('handles click events', async () => {
    const handleClick = vi.fn();
    render(<Button onClick={handleClick}>Click me</Button>);

    await userEvent.click(screen.getByRole('button'));
    expect(handleClick).toHaveBeenCalledTimes(1);
  });

  it('can be disabled', () => {
    render(<Button disabled>Click me</Button>);
    expect(screen.getByRole('button')).toBeDisabled();
  });
});
```

---

## Summary

| Task ID | Task Title | Effort | Priority |
|---------|-----------|--------|----------|
| NEO-1.1-001 | Create Next.js 16 Project | 20m | P0 |
| NEO-1.1-002 | Configure Package Dependencies | 20m | P0 |
| NEO-1.1-003 | Initialize shadcn/ui | 15m | P0 |
| NEO-1.1-004 | Pre-install shadcn/ui Components | 30m | P1 |
| NEO-1.1-005 | Create Prisma Schema | 30m | P0 |
| NEO-1.1-006 | Configure Environment Files | 15m | P1 |
| NEO-1.1-007 | Set Up Directory Structure | 20m | P1 |
| NEO-1.1-008 | Configure ESLint and Prettier | 10m | P2 |
| NEO-1.1-009 | Add Base Layout Components | 20m | P1 |
| NEO-1.1-010 | Set Up Vitest | 20m | P1 |
| NEO-1.1-011 | Set Up Playwright | 15m | P1 |
| NEO-1.1-012 | Set Up MSW | 15m | P1 |
| NEO-1.1-013 | Configure Dark Mode | 15m | P1 |
| NEO-1.1-014 | Configure Tailwind Dark Mode | 10m | P1 |
| NEO-1.1-015 | Customize Tailwind Theme | 15m | P2 |
| NEO-1.1-016 | Write Example Unit Test | 10m | P2 |

**Total Effort**: ~3.25 hours (195 minutes)
**Total Tasks**: 16

---

## Dependencies Graph

```
NEO-1.1-001 (Next.js Project)
    |
    +-> NEO-1.1-002 (Dependencies)
    |       |
    |       +-> NEO-1.1-003 (shadcn init)
    |       |       |
    |       |       +-> NEO-1.1-004 (shadcn components)
    |       |       |       |
    |       |       |       +-> NEO-1.1-009 (Layout)
    |       |       |       +-> NEO-1.1-013 (Dark Mode)
    |       |       |               |
    |       |       |               +-> NEO-1.1-014 (Tailwind Dark)
    |       |       |                       |
    |       |       |                       +-> NEO-1.1-015 (Theme)
    |       |
    |       +-> NEO-1.1-005 (Prisma Schema)
    |       |
    |       +-> NEO-1.1-008 (ESLint)
    |       |
    |       +-> NEO-1.1-010 (Vitest)
    |               |
    |               +-> NEO-1.1-011 (Playwright)
    |               +-> NEO-1.1-012 (MSW)
    |               +-> NEO-1.1-016 (Example Test)
    |
    +-> NEO-1.1-006 (Environment)
    |
    +-> NEO-1.1-007 (Directory Structure)
```

---

## Parallel Execution Opportunities

The following tasks can be executed in parallel after NEO-1.1-002:

**Parallel Group A** (After NEO-1.1-002):
- [P] NEO-1.1-005 (Prisma Schema)
- [P] NEO-1.1-006 (Environment Files)
- [P] NEO-1.1-008 (ESLint)

**Parallel Group B** (After NEO-1.1-003):
- [P] NEO-1.1-004 (shadcn components)
- [P] NEO-1.1-007 (Directory Structure)

**Parallel Group C** (After NEO-1.1-010):
- [P] NEO-1.1-011 (Playwright)
- [P] NEO-1.1-012 (MSW)
- [P] NEO-1.1-016 (Example Test)
