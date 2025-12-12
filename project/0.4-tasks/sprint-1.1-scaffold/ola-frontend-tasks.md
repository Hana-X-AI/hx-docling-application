# Sprint 1.1: Frontend UI Tasks

**Agent**: Ola Mae Johnson (`@ola`)
**Role**: Frontend UI Developer & Accessibility Specialist
**Sprint**: 1.1 - Project Scaffold & Infrastructure
**Dependencies**: Sprint 0 Prerequisites Complete
**Effort**: 1.0h (of 3.25h total sprint)

---

## Task Overview

| Task ID | Title | Effort | Status |
|---------|-------|--------|--------|
| OLA-1.1-001 | Configure Dark Mode Theme Support | 30m | Pending |
| OLA-1.1-002 | Customize Tailwind Theme with Brand Colors | 25m | Pending |
| OLA-1.1-003 | Verify shadcn/ui Component Installation | 15m | Pending |
| OLA-1.1-004 | Create Base Layout with Theme Toggle | 30m | Pending |

**Total Effort**: 1.0h (60 minutes)

---

## OLA-1.1-001: Configure Dark Mode Theme Support

### Description

Set up dark mode support using `next-themes` package, configured with class-based theme switching. This provides the foundation for the application's theme system.

### Acceptance Criteria

- [ ] `next-themes` package installed and configured in `package.json`
- [ ] `ThemeProvider` added to root layout (`src/app/layout.tsx`)
- [ ] Provider configured with `attribute="class"` for Tailwind integration
- [ ] Default theme set to `system` (respects user OS preference)
- [ ] Storage enabled for theme persistence across sessions
- [ ] `suppressHydrationWarning` added to `<html>` tag to prevent hydration mismatch

### Dependencies

- Sprint 1.1 Task 2: Package.json configuration complete
- Sprint 1.1 Task 3: shadcn/ui initialized

### Deliverables

**File**: `src/app/layout.tsx`

**Configuration**:
```typescript
import { ThemeProvider } from 'next-themes'

export default function RootLayout({ children }) {
  return (
    <html lang="en" suppressHydrationWarning>
      <body>
        <ThemeProvider
          attribute="class"
          defaultTheme="system"
          enableSystem
          disableTransitionOnChange
        >
          {children}
        </ThemeProvider>
      </body>
    </html>
  )
}
```

### Technical Notes

- **Reference**: Implementation Plan Sprint 1.1 Task 13
- **Package**: `next-themes@^0.4.x`
- **Integration**: Works with Tailwind `darkMode: 'class'` configuration
- **Persistence**: Theme choice saved to localStorage automatically
- **SSR Handling**: `suppressHydrationWarning` prevents mismatch between server and client
- **Accessibility**: Respects `prefers-color-scheme` when defaultTheme is "system"

### Testing

```bash
# Verify ThemeProvider renders without errors
npm run dev
# Open browser console - no hydration warnings
# Check localStorage for theme-mode key
```

---

## OLA-1.1-002: Customize Tailwind Theme with Brand Colors

### Description

Extend the Tailwind configuration with HX-Infrastructure brand colors, custom spacing, and typography settings to ensure consistent design system throughout the application.

### Acceptance Criteria

- [ ] `tailwind.config.ts` extended with brand color palette
- [ ] Dark mode configured with `darkMode: 'class'`
- [ ] Custom spacing scale added for consistent layout
- [ ] Typography settings configured (font families, sizes, weights)
- [ ] CSS variables defined in `globals.css` for theme compatibility
- [ ] shadcn/ui theme variables integrated (primary, secondary, accent, destructive)
- [ ] Color contrast verified for WCAG 2.1 Level AA compliance

### Dependencies

- OLA-1.1-001: Dark mode provider configured
- Sprint 1.1 Task 3: shadcn/ui initialized

### Deliverables

**File**: `tailwind.config.ts`

**Configuration Example**:
```typescript
import type { Config } from 'tailwindcss'

const config: Config = {
  darkMode: 'class',
  content: [
    './src/pages/**/*.{js,ts,jsx,tsx,mdx}',
    './src/components/**/*.{js,ts,jsx,tsx,mdx}',
    './src/app/**/*.{js,ts,jsx,tsx,mdx}',
  ],
  theme: {
    extend: {
      colors: {
        // HX-Infrastructure Brand Colors
        hx: {
          primary: '#0066CC',
          secondary: '#4A5568',
          accent: '#38B2AC',
          success: '#48BB78',
          warning: '#ECC94B',
          error: '#F56565',
        },
        // shadcn/ui theme variables (CSS var based)
        border: 'hsl(var(--border))',
        background: 'hsl(var(--background))',
        foreground: 'hsl(var(--foreground))',
        // ... additional shadcn theme colors
      },
      spacing: {
        '18': '4.5rem',
        '88': '22rem',
        '128': '32rem',
      },
      fontFamily: {
        sans: ['Inter', 'sans-serif'],
        mono: ['JetBrains Mono', 'monospace'],
      },
    },
  },
  plugins: [require('tailwindcss-animate')],
}
```

**File**: `src/app/globals.css`

**CSS Variables**:
```css
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  :root {
    --background: 0 0% 100%;
    --foreground: 222.2 84% 4.9%;
    --primary: 221.2 83.2% 53.3%;
    /* ... additional light theme variables */
  }

  .dark {
    --background: 222.2 84% 4.9%;
    --foreground: 210 40% 98%;
    --primary: 217.2 91.2% 59.8%;
    /* ... additional dark theme variables */
  }
}
```

### Technical Notes

- **Reference**: Implementation Plan Sprint 1.1 Task 15, Review Issue SHADCN-M2
- **Color Contrast**: All color combinations tested with WebAIM Contrast Checker
- **CSS Variables**: Enables runtime theme switching without rebuilds
- **Brand Colors**: Based on HX-Infrastructure design system
- **Typography**: Inter for UI, JetBrains Mono for code blocks
- **Accessibility**: Minimum contrast ratio 4.5:1 for normal text, 3:1 for large text

### Testing

```bash
# Verify Tailwind compilation
npm run build
# Check for compilation errors

# Visual testing
npm run dev
# Toggle dark/light mode and verify:
# - All colors render correctly
# - No color contrast issues
# - Typography scales properly
```

---

## OLA-1.1-003: Verify shadcn/ui Component Installation

### Description

Validate that all required shadcn/ui components are correctly installed and functional. This task ensures the UI component library is ready for use in subsequent sprints.

### Acceptance Criteria

- [ ] All 11 required components installed via `npx shadcn@latest add [component]`
- [ ] Components verified: Button, Input, Card, Tabs, Form, Dialog, Select, Toast, Skeleton, Table, Pagination
- [ ] Component files exist in `src/components/ui/` directory
- [ ] TypeScript types resolve without errors
- [ ] Dark mode variants functional for all components
- [ ] Example usage documented in component comments

### Dependencies

- Sprint 1.1 Task 3: shadcn/ui initialization complete
- Sprint 1.1 Task 4: Component pre-installation complete
- OLA-1.1-001: Dark mode configured
- OLA-1.1-002: Tailwind theme customized

### Deliverables

**Components Directory**: `src/components/ui/`

**Required Files**:
```
src/components/ui/
├── button.tsx
├── input.tsx
├── card.tsx
├── tabs.tsx
├── form.tsx
├── dialog.tsx
├── select.tsx
├── toast.tsx
├── skeleton.tsx
├── table.tsx
└── pagination.tsx
```

**Verification Checklist**:
| Component | File Exists | TypeScript OK | Dark Mode OK | Notes |
|-----------|-------------|---------------|--------------|-------|
| Button | ☐ | ☐ | ☐ | Primary, secondary, destructive variants |
| Input | ☐ | ☐ | ☐ | With error state support |
| Card | ☐ | ☐ | ☐ | Header, content, footer sections |
| Tabs | ☐ | ☐ | ☐ | Used in results viewer |
| Form | ☐ | ☐ | ☐ | react-hook-form integration |
| Dialog | ☐ | ☐ | ☐ | Modal dialogs |
| Select | ☐ | ☐ | ☐ | Dropdown selections |
| Toast | ☐ | ☐ | ☐ | Notification system |
| Skeleton | ☐ | ☐ | ☐ | Loading states |
| Table | ☐ | ☐ | ☐ | History view |
| Pagination | ☐ | ☐ | ☐ | History pagination |

### Technical Notes

- **Reference**: Implementation Plan Sprint 1.1 Task 4, Review Issue SHADCN-M1
- **Installation Command**: `npx shadcn@latest add button input card tabs form dialog select toast skeleton table pagination`
- **Configuration**: Uses Tailwind CSS variables for theming
- **Accessibility**: All components include ARIA attributes out-of-box
- **Customization**: Can override default styles in component files

### Testing

```bash
# TypeScript compilation check
npm run typecheck

# Verify files exist
ls -la src/components/ui/

# Test component imports in a demo page
# Create src/app/test-components/page.tsx
# Import and render each component
npm run dev
```

---

## OLA-1.1-004: Create Base Layout with Theme Toggle

### Description

Build the application's base layout structure including header with theme toggle, main content area, and footer. This provides the container for all application pages.

### Acceptance Criteria

- [ ] Header component created with navigation structure
- [ ] Theme toggle button implemented using shadcn/ui Button
- [ ] Toggle icon switches between sun (light) and moon (dark)
- [ ] Main content area with proper spacing and responsive width
- [ ] Footer component with application info and links
- [ ] Layout responsive across mobile (320px), tablet (768px), desktop (1024px+)
- [ ] Accessibility: theme toggle has ARIA label, keyboard accessible
- [ ] Smooth transition when switching themes (optional but recommended)

### Dependencies

- OLA-1.1-001: Dark mode provider configured
- OLA-1.1-002: Tailwind theme customized
- OLA-1.1-003: shadcn/ui components verified
- Sprint 1.1 Task 9: Base layout components task

### Deliverables

**File**: `src/components/layout/Header.tsx`

```typescript
'use client'

import { useTheme } from 'next-themes'
import { Moon, Sun } from 'lucide-react'
import { Button } from '@/components/ui/button'

export function Header() {
  const { theme, setTheme } = useTheme()

  return (
    <header className="border-b border-border bg-background">
      <div className="container mx-auto flex h-16 items-center justify-between px-4">
        <div className="flex items-center gap-2">
          <h1 className="text-xl font-semibold">HX Docling UI</h1>
        </div>
        <nav className="flex items-center gap-4">
          <Button
            variant="ghost"
            size="icon"
            onClick={() => setTheme(theme === 'dark' ? 'light' : 'dark')}
            aria-label={`Switch to ${theme === 'dark' ? 'light' : 'dark'} mode`}
          >
            {theme === 'dark' ? (
              <Sun className="h-5 w-5" />
            ) : (
              <Moon className="h-5 w-5" />
            )}
          </Button>
        </nav>
      </div>
    </header>
  )
}
```

**File**: `src/components/layout/Footer.tsx`

```typescript
export function Footer() {
  return (
    <footer className="border-t border-border bg-background">
      <div className="container mx-auto px-4 py-6">
        <p className="text-center text-sm text-muted-foreground">
          HX Docling UI - Document Processing Application
        </p>
      </div>
    </footer>
  )
}
```

**File**: `src/app/layout.tsx` (Updated)

```typescript
import { Header } from '@/components/layout/Header'
import { Footer } from '@/components/layout/Footer'
import { ThemeProvider } from 'next-themes'
import './globals.css'

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" suppressHydrationWarning>
      <body>
        <ThemeProvider attribute="class" defaultTheme="system" enableSystem>
          <div className="flex min-h-screen flex-col">
            <Header />
            <main className="flex-1 container mx-auto px-4 py-8">
              {children}
            </main>
            <Footer />
          </div>
        </ThemeProvider>
      </body>
    </html>
  )
}
```

### Technical Notes

- **Reference**: Implementation Plan Sprint 1.1 Task 9
- **Icon Library**: `lucide-react@^0.460.x` for Sun/Moon icons
- **Client Component**: Header must be `'use client'` for `useTheme` hook
- **Responsive**: Uses Tailwind `container` and breakpoint utilities
- **Accessibility**:
  - ARIA label describes current and next state
  - Button keyboard accessible (Enter/Space)
  - Icon size meets minimum 24px touch target
- **Layout Strategy**: Flexbox with `flex-1` for sticky footer

### Testing

```bash
# Run development server
npm run dev

# Manual testing checklist:
# [ ] Theme toggle visible in header
# [ ] Clicking toggle switches between light/dark
# [ ] Icon changes (Sun <-> Moon)
# [ ] Theme persists on page reload
# [ ] Header responsive on mobile/tablet/desktop
# [ ] Footer sticks to bottom on short pages
# [ ] Keyboard navigation works (Tab to toggle, Enter/Space to activate)
# [ ] Screen reader announces theme state (test with NVDA/JAWS)
```

---

## Sprint Completion Checklist

### Acceptance Criteria (Sprint 1.1 Frontend)

- [ ] Dark mode toggle works and persists across sessions
- [ ] Tailwind theme customized with brand colors
- [ ] All 11 shadcn/ui components installed and verified
- [ ] Base layout renders with header, main, footer
- [ ] Layout responsive across all breakpoints
- [ ] Theme switching smooth with no flicker
- [ ] Accessibility: keyboard navigation, screen reader support
- [ ] No TypeScript errors (`npm run typecheck`)
- [ ] No console warnings in browser

### Integration Points

**With Neo (Backend)**:
- Layout structure ready for page routing
- Theme variables available for all components

**With Gordon (shadcn/ui Specialist)**:
- Component customization follows shadcn/ui patterns
- Theme variables align with design system

**With Julia (Testing)**:
- Accessibility testing framework ready
- Visual regression baseline established

### Next Sprint Preview

**Sprint 1.3** will build upon this foundation with:
- UploadZone component using Card and Button
- FilePreview using shadcn/ui styles
- Form components with react-hook-form integration

---

## Notes

- **Performance**: Theme toggle uses CSS variables for instant switching
- **Best Practices**: All components follow React Server Components pattern (except Header which requires client interactivity)
- **Design System**: Colors and spacing align with HX-Infrastructure brand guidelines
- **Scalability**: Layout structure supports future navigation additions
