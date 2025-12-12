# Julia Santos - Testing & QA Tasks: Sprint 1.1 (Project Scaffold)

**Sprint**: 1.1 - Project Scaffold & Infrastructure
**Role**: Testing & Quality Assurance Specialist
**Document Version**: 1.0.0
**Created**: 2025-12-12
**References**:
- Implementation Plan v1.2.0 Section 4.1
- Detailed Specification v1.2.0 Section 10

---

## Sprint 1.1 Testing Objectives

Establish the foundational test infrastructure to support TDD practices and continuous quality validation throughout the project. This includes configuring Vitest for unit/integration testing, Playwright for E2E testing, and MSW for API mocking.

---

## Tasks

### JUL-1.1-001: Configure Vitest Test Framework

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.1-001 |
| **Title** | Configure Vitest with React Testing Library |
| **Priority** | P0 (Critical) |
| **Effort** | 1.5 hours |
| **Dependencies** | NEO-1.1-001 (Next.js project created), NEO-1.1-002 (package.json configured) |

#### Description

Set up Vitest as the primary unit testing framework with React Testing Library integration for component testing. Configure jsdom environment for DOM testing and ensure compatibility with Next.js 16 and React 19.

#### Acceptance Criteria

- [ ] `vitest.config.ts` created with proper configuration
- [ ] jsdom environment configured for component tests
- [ ] React Testing Library integrated and functional
- [ ] Path aliases from `tsconfig.json` properly resolved
- [ ] Coverage provider configured (v8 or istanbul)
- [ ] Test file patterns defined (`**/*.test.ts`, `**/*.test.tsx`)
- [ ] Setup file created for global test utilities
- [ ] `npm run test` command executes tests successfully
- [ ] `npm run test:coverage` generates coverage report
- [ ] Coverage thresholds configured (80% lines, 75% branches)

#### Technical Notes

```typescript
// vitest.config.ts structure
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: ['./src/test/setup.ts'],
    include: ['src/**/*.test.{ts,tsx}', 'test/**/*.test.{ts,tsx}'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'html', 'lcov'],
      exclude: ['node_modules/', 'test/', '**/*.d.ts'],
      thresholds: {
        lines: 80,
        branches: 75,
        functions: 80,
        statements: 80,
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

#### Deliverables

| Deliverable | Path |
|-------------|------|
| Vitest configuration | `vitest.config.ts` |
| Test setup file | `src/test/setup.ts` |
| Test utilities | `src/test/utils.tsx` |

---

### JUL-1.1-002: Configure Playwright for E2E Testing

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.1-002 |
| **Title** | Configure Playwright for E2E Testing |
| **Priority** | P0 (Critical) |
| **Effort** | 1.0 hour |
| **Dependencies** | NEO-1.1-001 (Next.js project created) |

#### Description

Set up Playwright for end-to-end testing with proper browser configuration, test directories, and integration with the Next.js development server.

#### Acceptance Criteria

- [ ] `playwright.config.ts` created with proper configuration
- [ ] Chromium, Firefox, and WebKit browsers configured
- [ ] Base URL configured for development server
- [ ] Test directories organized (`test/e2e/`)
- [ ] Screenshot and trace configuration for debugging
- [ ] Timeout configuration per specification (60s for processing tests)
- [ ] `npm run test:e2e` command configured
- [ ] Playwright can start and interact with development server
- [ ] axe-core integration configured for accessibility testing

#### Technical Notes

```typescript
// playwright.config.ts structure
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
    { name: 'firefox', use: { ...devices['Desktop Firefox'] } },
    { name: 'webkit', use: { ...devices['Desktop Safari'] } },
  ],
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
    timeout: 120000,
  },
});
```

#### Deliverables

| Deliverable | Path |
|-------------|------|
| Playwright configuration | `playwright.config.ts` |
| E2E test directory | `test/e2e/` |
| Example E2E test | `test/e2e/example.spec.ts` |

---

### JUL-1.1-003: Configure MSW for API Mocking

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.1-003 |
| **Title** | Configure MSW (Mock Service Worker) for API Mocking |
| **Priority** | P0 (Critical) |
| **Effort** | 1.0 hour |
| **Dependencies** | JUL-1.1-001 (Vitest configured) |

#### Description

Set up MSW for mocking API requests during unit and integration tests. Create initial handlers for MCP server responses and establish patterns for test mocking.

#### Acceptance Criteria

- [ ] MSW installed and configured for Node.js test environment
- [ ] Mock handlers directory structure created
- [ ] Initial MCP mock handlers defined (health, tools/list)
- [ ] Session mock handlers defined
- [ ] Server setup integrated with Vitest setup file
- [ ] Handler reset between tests ensured
- [ ] Response factories for common data shapes created
- [ ] Documentation for adding new handlers provided

#### Technical Notes

```typescript
// src/mocks/handlers.ts structure
import { http, HttpResponse } from 'msw';

export const mcpHandlers = [
  http.post('http://hx-docling-server:8000/rpc', async ({ request }) => {
    const body = await request.json();

    if (body.method === 'initialize') {
      return HttpResponse.json({
        jsonrpc: '2.0',
        id: body.id,
        result: {
          protocolVersion: '2024-11-05',
          serverInfo: { name: 'hx-docling-mcp-server', version: '1.0.0' },
          capabilities: { tools: {} },
        },
      });
    }

    // Add more handlers...
  }),
];

export const handlers = [...mcpHandlers];
```

#### Deliverables

| Deliverable | Path |
|-------------|------|
| MSW handlers | `src/mocks/handlers.ts` |
| MCP mock handlers | `src/mocks/handlers/mcp.ts` |
| Mock server setup | `src/mocks/server.ts` |
| Response factories | `src/mocks/factories/` |

---

### JUL-1.1-004: Create Test Directory Structure

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.1-004 |
| **Title** | Create Standardized Test Directory Structure |
| **Priority** | P1 (High) |
| **Effort** | 0.5 hours |
| **Dependencies** | JUL-1.1-001, JUL-1.1-002 |

#### Description

Establish a standardized test directory structure following testing best practices. Create directories for different test types and establish naming conventions.

#### Acceptance Criteria

- [ ] Unit test directories created (`src/**/__tests__/`)
- [ ] Integration test directory created (`test/integration/`)
- [ ] E2E test directory created (`test/e2e/`)
- [ ] Security test directory created (`test/security/`)
- [ ] Accessibility test directory created (`test/a11y/`)
- [ ] Test fixtures directory created (`test/fixtures/`)
- [ ] Naming conventions documented
- [ ] README.md with testing guidelines created

#### Technical Notes

```
test/
├── e2e/                    # Playwright E2E tests
│   ├── critical-path.spec.ts
│   └── error-handling.spec.ts
├── integration/            # API route tests
│   └── api/
├── security/               # Security tests
│   ├── ssrf.test.ts
│   └── input-validation.test.ts
├── a11y/                   # Accessibility tests
│   └── accessibility.spec.ts
├── fixtures/               # Test data files
│   ├── sample.pdf
│   └── sample.docx
└── README.md               # Testing guidelines

src/
├── components/
│   └── upload/
│       ├── UploadZone.tsx
│       └── __tests__/
│           └── UploadZone.test.tsx
└── lib/
    └── validation/
        ├── file.ts
        └── __tests__/
            └── file.test.ts
```

#### Deliverables

| Deliverable | Path |
|-------------|------|
| Test directories | `test/` structure |
| Testing guidelines | `test/README.md` |

---

### JUL-1.1-005: Write Example Unit Test to Verify Infrastructure

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.1-005 |
| **Title** | Write Example Unit Test to Verify Test Infrastructure |
| **Priority** | P1 (High) |
| **Effort** | 0.5 hours |
| **Dependencies** | JUL-1.1-001, JUL-1.1-003 |

#### Description

Create example unit tests that verify the test infrastructure is working correctly. These tests serve as templates for future test development and validate the configuration.

#### Acceptance Criteria

- [ ] Example unit test written and passing
- [ ] Example component test written and passing
- [ ] MSW integration verified in test
- [ ] Test utilities (render, screen, etc.) verified
- [ ] Coverage report generated successfully
- [ ] All test commands documented in package.json

#### Technical Notes

```typescript
// src/test/example.test.ts
import { describe, it, expect } from 'vitest';

describe('Test Infrastructure Verification', () => {
  it('vitest is working correctly', () => {
    expect(1 + 1).toBe(2);
  });

  it('can use test utilities', () => {
    const add = (a: number, b: number) => a + b;
    expect(add(2, 3)).toBe(5);
  });
});

// src/components/__tests__/Example.test.tsx
import { render, screen } from '@testing-library/react';
import { describe, it, expect } from 'vitest';

function ExampleComponent() {
  return <div>Hello, Test!</div>;
}

describe('Example Component Test', () => {
  it('renders correctly', () => {
    render(<ExampleComponent />);
    expect(screen.getByText('Hello, Test!')).toBeInTheDocument();
  });
});
```

#### Deliverables

| Deliverable | Path |
|-------------|------|
| Example unit test | `src/test/example.test.ts` |
| Example component test | `src/components/__tests__/Example.test.tsx` |

---

### JUL-1.1-006: Define Quality Gate Scripts

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.1-006 |
| **Title** | Define Quality Gate Scripts in package.json |
| **Priority** | P1 (High) |
| **Effort** | 0.5 hours |
| **Dependencies** | JUL-1.1-001, JUL-1.1-002 |

#### Description

Configure npm scripts for all quality gates including testing, linting, type checking, and coverage verification. Ensure scripts align with CI/CD requirements.

#### Acceptance Criteria

- [ ] `npm run test` - runs all unit tests
- [ ] `npm run test:watch` - runs tests in watch mode
- [ ] `npm run test:coverage` - runs tests with coverage report
- [ ] `npm run test:e2e` - runs Playwright E2E tests
- [ ] `npm run test:e2e:ui` - runs Playwright with UI mode
- [ ] `npm run quality` - runs all quality gates (lint + typecheck + test)
- [ ] Coverage thresholds fail build if not met
- [ ] Scripts documented in README

#### Technical Notes

```json
// package.json scripts
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage",
    "test:e2e": "playwright test",
    "test:e2e:ui": "playwright test --ui",
    "lint": "eslint . --ext .ts,.tsx",
    "typecheck": "tsc --noEmit",
    "quality": "npm run lint && npm run typecheck && npm run test:coverage"
  }
}
```

#### Deliverables

| Deliverable | Path |
|-------------|------|
| Package.json scripts | `package.json` |
| Quality gate documentation | `docs/quality-gates.md` |

---

## Sprint 1.1 Testing Summary

| Metric | Target | Notes |
|--------|--------|-------|
| Total Tasks | 6 | |
| Total Effort | 5.0 hours | |
| Critical Tasks | 3 | Test framework configurations |
| High Priority Tasks | 3 | Directory structure and verification |

### Dependencies on Other Agents

| Agent | Tasks Required | Blocking |
|-------|----------------|----------|
| Neo | NEO-1.1-001, NEO-1.1-002 | JUL-1.1-001, JUL-1.1-002, JUL-1.1-003 |

### Quality Gates for Sprint 1.1

- [ ] All 6 testing tasks completed
- [ ] `npm run test` executes successfully
- [ ] `npm run test:e2e` starts Playwright
- [ ] Coverage report generates
- [ ] Test directory structure established
- [ ] MSW handlers functional

---

## Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0.0 | 2025-12-12 | Julia Santos | Initial creation |
