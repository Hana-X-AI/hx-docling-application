# Tasks: shadcn/ui Setup & Configuration (Sprint 1.1)

**Agent**: Gordon Zain (`@gordon`) - shadcn/ui Specialist
**Sprint**: 1.1 - Project Scaffold & Infrastructure
**Duration**: ~1.5h
**Prerequisites**: Next.js 16 project initialized, package.json configured

## MCP Integration

**hx-shadcn MCP Server**: The HX-Infrastructure ecosystem provides a dedicated shadcn/ui MCP server for programmatic component access during AI-assisted development workflows.

**Service Details**:
- **Host**: hx-shadcn.hx.dev.local (192.168.10.229)
- **Port**: 7423
- **Protocol**: SSE + MCP
- **Status**: OPERATIONAL
- **Service Descriptor**: `project/0.8-references/hx-shadcn-service-descriptor.md`

**Available MCP Tools**:
| Tool | Description | Usage Example |
|------|-------------|---------------|
| `list_components` | List all available shadcn/ui components | Discover available components before installation |
| `get_component` | Retrieve component source code with dependencies | Preview component implementation before adding |
| `get_component_demo` | Get usage examples and demo code | View component usage patterns |
| `get_component_metadata` | Get props, installation instructions, documentation | Review component API before integration |
| `list_blocks` | List UI blocks by category (React, Svelte, Vue only) | Discover pre-built UI patterns |
| `get_block` | Retrieve block source with all components | Get complete UI block implementations |

**Development Workflow**:
1. **Standard CLI Approach** (Recommended for manual installation):
   ```bash
   npx shadcn@latest add button
   ```

2. **MCP Server Approach** (For AI-assisted workflows):
   ```json
   {
     "method": "tools/call",
     "params": {
       "name": "get_component",
       "arguments": {"componentName": "button"}
     }
   }
   ```

3. **MCP Server Benefits**:
   - Programmatic component discovery during development
   - AI assistants can retrieve component source for code generation
   - Metadata access for prop type inference
   - Demo code for usage examples

**Note**: While `npx shadcn@latest add` is the standard CLI approach for component installation, the MCP server provides programmatic access for AI-assisted development workflows. Both approaches result in the same copied component files in `src/components/ui/`.

## Task List

### GOR-1.1-001: Initialize shadcn/ui with CLI
**Status**: Pending
**Effort**: 15 minutes
**Dependencies**: Sprint 1.1 Task 2 (package.json configuration)
**Priority**: P0 - Critical Path

#### Description
Initialize shadcn/ui in the Next.js 16 project using the official CLI. This is a **copy-paste component library**, not an npm package. Components will be copied directly into `src/components/ui/` and can be customized by editing the source code.

#### Acceptance Criteria
- [ ] Run `npx shadcn@latest init` successfully
- [ ] Configuration file `components.json` created in project root
- [ ] CSS variables defined in `src/app/globals.css` for theming
- [ ] `cn()` utility function created in `src/lib/utils.ts` for class merging
- [ ] TailwindCSS configuration updated with shadcn/ui settings
- [ ] TypeScript paths configured for `@/components/*` imports
- [ ] No TypeScript errors after initialization

#### Technical Implementation
```bash
# Execute in project root
npx shadcn@latest init

# Configuration prompts:
# - TypeScript: Yes
# - Style: Default
# - Base color: Slate
# - CSS variables: Yes
# - Tailwind config: tailwind.config.ts
# - Import alias: @/components
# - React Server Components: Yes
```

**Critical Files Created**:
- `components.json` - shadcn/ui configuration
- `src/lib/utils.ts` - `cn()` utility with clsx + tailwind-merge
- `src/app/globals.css` - CSS variables for theming

#### Validation
```bash
# Verify TypeScript compilation
npm run typecheck

# Check that utils exist
cat src/lib/utils.ts | grep "cn("
```

#### Deliverables
- Configuration file: `components.json`
- Utility function: `src/lib/utils.ts`
- Updated globals: `src/app/globals.css`
- Updated config: `tailwind.config.ts`

---

### GOR-1.1-002: Pre-install Core shadcn/ui Components
**Status**: Pending
**Effort**: 30 minutes
**Dependencies**: GOR-1.1-001
**Priority**: P0 - Critical Path

#### Description
Pre-install all required shadcn/ui components using the CLI. This copies component source code into the project, allowing full customization. Install components in order of dependencies (primitives first).

#### Acceptance Criteria
- [ ] Button component installed (`src/components/ui/button.tsx`)
- [ ] Input component installed (`src/components/ui/input.tsx`)
- [ ] Card component installed (`src/components/ui/card.tsx`)
- [ ] Form component installed (`src/components/ui/form.tsx`)
- [ ] Tabs component installed (`src/components/ui/tabs.tsx`)
- [ ] Table component installed (`src/components/ui/table.tsx`)
- [ ] Dialog component installed (`src/components/ui/dialog.tsx`)
- [ ] Select component installed (`src/components/ui/select.tsx`)
- [ ] Toast component installed (`src/components/ui/toast.tsx`, `src/components/ui/toaster.tsx`, `src/components/ui/use-toast.ts`)
- [ ] Skeleton component installed (`src/components/ui/skeleton.tsx`)
- [ ] Pagination component installed (`src/components/ui/pagination.tsx`)
- [ ] Label component installed (dependency for Form)
- [ ] All Radix UI dependencies installed in package.json
- [ ] No TypeScript errors in any component file

#### Technical Implementation
```bash
# Install components in dependency order
# Each command copies source code to src/components/ui/

npx shadcn@latest add button
npx shadcn@latest add input
npx shadcn@latest add label
npx shadcn@latest add card
npx shadcn@latest add form
npx shadcn@latest add tabs
npx shadcn@latest add table
npx shadcn@latest add dialog
npx shadcn@latest add select
npx shadcn@latest add toast
npx shadcn@latest add skeleton
npx shadcn@latest add pagination
```

**Component Inventory** (11 components total):
| Component | File Path | Radix Primitive | Usage Sprint |
|-----------|-----------|-----------------|--------------|
| Button | `ui/button.tsx` | - | 1.3, 1.4, 1.6, 1.7 |
| Input | `ui/input.tsx` | - | 1.3, 1.4 |
| Label | `ui/label.tsx` | `@radix-ui/react-label` | 1.3, 1.4 |
| Card | `ui/card.tsx` | - | 1.3, 1.5b, 1.6 |
| Form | `ui/form.tsx` | `@radix-ui/react-label` | 1.3, 1.4 |
| Tabs | `ui/tabs.tsx` | `@radix-ui/react-tabs` | 1.6 |
| Table | `ui/table.tsx` | - | 1.7 |
| Dialog | `ui/dialog.tsx` | `@radix-ui/react-dialog` | 1.7 |
| Select | `ui/select.tsx` | `@radix-ui/react-select` | Future |
| Toast | `ui/toast.tsx`, `ui/toaster.tsx`, `ui/use-toast.ts` | `@radix-ui/react-toast` | 1.3, 1.4, 1.5b |
| Skeleton | `ui/skeleton.tsx` | - | 1.5b, 1.6 |
| Pagination | `ui/pagination.tsx` | - | 1.7 |

#### Validation
```bash
# Verify all components exist
ls -1 src/components/ui/*.tsx | wc -l  # Should be >= 11

# Verify TypeScript compilation
npm run typecheck

# Test component imports
node -e "console.log(require('./src/components/ui/button.tsx'))"
```

#### Deliverables
- 11+ component files in `src/components/ui/`
- Updated `package.json` with Radix UI dependencies
- Component documentation in code comments

---

### GOR-1.1-003: Configure Tailwind Theme Customization
**Status**: Pending
**Effort**: 15 minutes
**Dependencies**: GOR-1.1-001
**Priority**: P1 - High

#### Description
Customize Tailwind theme with brand colors, spacing, and typography per design specification. This addresses **SHADCN-M2** from the implementation plan (Tailwind Customization).

#### Acceptance Criteria
- [ ] Brand colors defined in `tailwind.config.ts` (primary, secondary, accent)
- [ ] Dark mode configured with `darkMode: 'class'`
- [ ] Custom spacing scale defined if needed
- [ ] Font family configured (system fonts or custom)
- [ ] Border radius customization applied
- [ ] CSS variables in `globals.css` match Tailwind config
- [ ] Theme works in both light and dark mode

#### Technical Implementation
```typescript
// tailwind.config.ts
import type { Config } from 'tailwindcss';

const config: Config = {
  darkMode: 'class', // Enable class-based dark mode
  content: [
    './src/pages/**/*.{js,ts,jsx,tsx,mdx}',
    './src/components/**/*.{js,ts,jsx,tsx,mdx}',
    './src/app/**/*.{js,ts,jsx,tsx,mdx}',
  ],
  theme: {
    extend: {
      colors: {
        border: 'hsl(var(--border))',
        input: 'hsl(var(--input))',
        ring: 'hsl(var(--ring))',
        background: 'hsl(var(--background))',
        foreground: 'hsl(var(--foreground))',
        primary: {
          DEFAULT: 'hsl(var(--primary))',
          foreground: 'hsl(var(--primary-foreground))',
        },
        secondary: {
          DEFAULT: 'hsl(var(--secondary))',
          foreground: 'hsl(var(--secondary-foreground))',
        },
        destructive: {
          DEFAULT: 'hsl(var(--destructive))',
          foreground: 'hsl(var(--destructive-foreground))',
        },
        muted: {
          DEFAULT: 'hsl(var(--muted))',
          foreground: 'hsl(var(--muted-foreground))',
        },
        accent: {
          DEFAULT: 'hsl(var(--accent))',
          foreground: 'hsl(var(--accent-foreground))',
        },
        card: {
          DEFAULT: 'hsl(var(--card))',
          foreground: 'hsl(var(--card-foreground))',
        },
      },
      borderRadius: {
        lg: 'var(--radius)',
        md: 'calc(var(--radius) - 2px)',
        sm: 'calc(var(--radius) - 4px)',
      },
      fontFamily: {
        sans: ['var(--font-sans)', 'system-ui', 'sans-serif'],
      },
    },
  },
  plugins: [require('tailwindcss-animate')],
};

export default config;
```

**CSS Variables** (in `src/app/globals.css`):
```css
@layer base {
  :root {
    --background: 0 0% 100%;
    --foreground: 222.2 84% 4.9%;
    --card: 0 0% 100%;
    --card-foreground: 222.2 84% 4.9%;
    --primary: 221.2 83.2% 53.3%;
    --primary-foreground: 210 40% 98%;
    --radius: 0.5rem;
  }

  .dark {
    --background: 222.2 84% 4.9%;
    --foreground: 210 40% 98%;
    --card: 222.2 84% 4.9%;
    --card-foreground: 210 40% 98%;
    --primary: 217.2 91.2% 59.8%;
    --primary-foreground: 222.2 47.4% 11.2%;
  }
}
```

#### Validation
```bash
# Verify Tailwind compiles
npm run dev
# Check for CSS errors in terminal

# Test dark mode toggle
# Visit localhost:3000 and toggle theme
```

#### Deliverables
- Updated `tailwind.config.ts`
- Customized CSS variables in `src/app/globals.css`
- Theme documentation in comments

---

### GOR-1.1-004: Verify Component Rendering
**Status**: Pending
**Effort**: 20 minutes
**Dependencies**: GOR-1.1-002, GOR-1.1-003
**Priority**: P1 - High

#### Description
Create a test page to verify all shadcn/ui components render correctly in both light and dark modes. This ensures the installation is complete and components work as expected.

#### Acceptance Criteria
- [ ] Test page created at `src/app/test-components/page.tsx`
- [ ] All 11 components render without errors
- [ ] Components styled correctly in light mode
- [ ] Components styled correctly in dark mode
- [ ] No console errors in browser
- [ ] TypeScript compilation passes
- [ ] Components respond to interactions (buttons click, tabs switch, etc.)

#### Technical Implementation
```tsx
// src/app/test-components/page.tsx
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card';
import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs';
import { Table, TableBody, TableCell, TableHead, TableHeader, TableRow } from '@/components/ui/table';
import { Skeleton } from '@/components/ui/skeleton';
import { Badge } from '@/components/ui/badge';
import { Dialog, DialogContent, DialogTrigger } from '@/components/ui/dialog';

export default function TestComponentsPage() {
  return (
    <div className="container mx-auto p-8 space-y-8">
      <h1 className="text-3xl font-bold">shadcn/ui Component Test</h1>

      <Card>
        <CardHeader>
          <CardTitle>Buttons</CardTitle>
        </CardHeader>
        <CardContent className="flex gap-2">
          <Button>Default</Button>
          <Button variant="secondary">Secondary</Button>
          <Button variant="outline">Outline</Button>
          <Button variant="destructive">Destructive</Button>
        </CardContent>
      </Card>

      <Card>
        <CardHeader>
          <CardTitle>Input</CardTitle>
        </CardHeader>
        <CardContent>
          <Input placeholder="Enter text..." />
        </CardContent>
      </Card>

      <Tabs defaultValue="tab1">
        <TabsList>
          <TabsTrigger value="tab1">Tab 1</TabsTrigger>
          <TabsTrigger value="tab2">Tab 2</TabsTrigger>
        </TabsList>
        <TabsContent value="tab1">Content 1</TabsContent>
        <TabsContent value="tab2">Content 2</TabsContent>
      </Tabs>

      <Table>
        <TableHeader>
          <TableRow>
            <TableHead>Column 1</TableHead>
            <TableHead>Column 2</TableHead>
          </TableRow>
        </TableHeader>
        <TableBody>
          <TableRow>
            <TableCell>Data 1</TableCell>
            <TableCell>Data 2</TableCell>
          </TableRow>
        </TableBody>
      </Table>

      <Skeleton className="h-4 w-[250px]" />
    </div>
  );
}
```

#### Validation
```bash
# Start dev server
npm run dev

# Visit test page
# http://localhost:3000/test-components

# Toggle dark mode and verify components adapt
```

#### Deliverables
- Test page: `src/app/test-components/page.tsx`
- Screenshot of rendered components in light mode
- Screenshot of rendered components in dark mode

---

## Summary

**Total Tasks**: 4
**Total Effort**: 1.5 hours
**Critical Path**: GOR-1.1-001 → GOR-1.1-002 → GOR-1.1-004

**Success Criteria**:
- [ ] shadcn/ui initialized via CLI
- [ ] All 11 components pre-installed
- [ ] Tailwind theme customized (colors, dark mode)
- [ ] Components render correctly in test page
- [ ] Zero TypeScript errors
- [ ] Dark mode toggle works

**Coordination Points**:
- **William Chen (TailwindCSS)**: Review custom utility classes in Task GOR-1.1-003
- **Neo (Next.js Lead)**: Coordinate base layout integration in Sprint 1.1 Task 9
- **Trinity (Database)**: No direct coordination needed

**Anti-Patterns Prevented**:
- ✅ Using CLI instead of npm install for shadcn/ui
- ✅ Pre-installing all components to avoid mid-sprint delays
- ✅ CSS variables for theming instead of hardcoded colors
- ✅ Proper dark mode configuration with `darkMode: 'class'`

**Notes**:
- Components are **copied into the project**, not installed as dependencies
- Full source code ownership enables customization
- Radix UI primitives provide built-in accessibility
- All components use the `cn()` utility for class merging
