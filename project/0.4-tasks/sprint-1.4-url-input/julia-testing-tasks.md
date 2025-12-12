# Julia Santos - Testing & QA Tasks: Sprint 1.4 (URL Input with Security)

**Sprint**: 1.4 - URL Input with Security
**Role**: Testing & Quality Assurance Specialist
**Document Version**: 1.0.0
**Created**: 2025-12-12
**References**:
- Implementation Plan v1.2.0 Section 4.4
- Detailed Specification v1.2.0 Sections 10.1.1, 10.2

---

## Sprint 1.4 Testing Objectives

Implement incremental unit tests for the URL input system with special focus on security testing (SSRF prevention). Ensure comprehensive coverage of URL validation, document store state management, and accessibility compliance.

---

## Tasks

### JUL-1.4-001: Write Unit Tests for SSRF Validation

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.4-001 |
| **Title** | Write Comprehensive SSRF Prevention Tests |
| **Priority** | P0 (Critical - Security) |
| **Effort** | 1.25 hours |
| **Dependencies** | NEO-1.4-002 (SSRF prevention implemented) |

#### Description

Write exhaustive security tests for SSRF prevention covering all blocked IP patterns, DNS rebinding protection, and internal domain blocking. This is a security-critical testing task.

#### Acceptance Criteria

- [ ] Test blocks localhost (127.0.0.1, localhost)
- [ ] Test blocks IPv6 localhost (::1)
- [ ] Test blocks 10.0.0.0/8 private range
- [ ] Test blocks 172.16.0.0/12 private range
- [ ] Test blocks 192.168.0.0/16 private range
- [ ] Test blocks 169.254.0.0/16 link-local
- [ ] Test blocks *.hx.dev.local internal domains
- [ ] Test blocks AWS metadata endpoint (169.254.169.254)
- [ ] Test returns error code E104 for blocked URLs
- [ ] Test allows valid public URLs
- [ ] Test handles URL encoding attempts (bypass prevention)
- [ ] Test handles decimal IP notation
- [ ] Test handles octal IP notation
- [ ] Coverage 100% for SSRF validation logic

#### Technical Notes

```typescript
// src/lib/validation/__tests__/url.test.ts
import { describe, it, expect } from 'vitest';
import { urlSchema, isURLBlocked, validateUrl } from '../url';

describe('SSRF Prevention', () => {
  const BLOCKED_URLS = [
    // Localhost variations
    'http://localhost/secret',
    'http://127.0.0.1/admin',
    'http://[::1]/ipv6-localhost',
    'http://0.0.0.0/admin',

    // Private IPv4 ranges (RFC 1918)
    'http://10.0.0.1/internal',
    'http://10.255.255.255/internal',
    'http://172.16.0.1/internal',
    'http://172.31.255.255/internal',
    'http://192.168.0.1/internal',
    'http://192.168.255.255/internal',

    // Link-local
    'http://169.254.1.1/link-local',
    'http://169.254.169.254/latest/meta-data/', // AWS metadata

    // Internal HX domains
    'http://internal.hx.dev.local/api',
    'http://db.hx.dev.local/admin',
    'http://redis.hx.dev.local/',

    // Bypass attempts
    'http://127.0.0.1:8080/admin',
    'http://127.0.0.1.nip.io/',
    'http://0x7f.0x00.0x00.0x01/', // Hex encoding
    'http://2130706433/', // Decimal encoding (127.0.0.1)
    'http://017700000001/', // Octal encoding
  ];

  const ALLOWED_URLS = [
    'https://example.com/document.pdf',
    'https://www.google.com/',
    'http://external-api.com/v1/doc',
    'https://cdn.example.org/file.docx',
  ];

  describe('isURLBlocked', () => {
    BLOCKED_URLS.forEach((url) => {
      it(`blocks SSRF attempt to ${url}`, () => {
        expect(isURLBlocked(url)).toBe(true);
      });
    });

    ALLOWED_URLS.forEach((url) => {
      it(`allows public URL ${url}`, () => {
        expect(isURLBlocked(url)).toBe(false);
      });
    });
  });

  describe('validateUrl returns E104 for blocked URLs', () => {
    it('returns E104 error code for blocked URL', () => {
      const result = validateUrl('http://192.168.1.1/admin');
      expect(result.success).toBe(false);
      expect(result.error?.code).toBe('E104');
    });
  });

  describe('bypass attempt prevention', () => {
    it('blocks URL-encoded bypass attempts', () => {
      expect(isURLBlocked('http://127%2E0%2E0%2E1/')).toBe(true);
    });

    it('blocks double-encoded bypass attempts', () => {
      expect(isURLBlocked('http://127%252E0%252E0%252E1/')).toBe(true);
    });

    it('blocks alternate localhost representations', () => {
      expect(isURLBlocked('http://0177.0.0.1/')).toBe(true); // Octal
      expect(isURLBlocked('http://0x7f.0.0.1/')).toBe(true); // Hex
    });
  });
});
```

#### Deliverables

| Deliverable | Path |
|-------------|------|
| SSRF validation tests | `src/lib/validation/__tests__/url.test.ts` |
| Security test suite | `test/security/ssrf.test.ts` |

---

### JUL-1.4-002: Write Unit Tests for URL Format Validation

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.4-002 |
| **Title** | Write Unit Tests for URL Format Validation |
| **Priority** | P1 (High) |
| **Effort** | 0.5 hours |
| **Dependencies** | NEO-1.4-003 (URL validation implemented) |

#### Description

Write unit tests for URL format validation covering protocol checking, length limits, and malformed URL handling.

#### Acceptance Criteria

- [ ] Test accepts valid HTTP URLs
- [ ] Test accepts valid HTTPS URLs
- [ ] Test rejects non-HTTP/HTTPS protocols (ftp://, file://, etc.)
- [ ] Test rejects URLs over 2048 characters
- [ ] Test rejects malformed URLs (E101)
- [ ] Test handles URLs with query parameters
- [ ] Test handles URLs with fragments
- [ ] Test handles internationalized domain names
- [ ] Coverage >= 95% for URL format validation

#### Technical Notes

```typescript
// Additional tests for url.test.ts
describe('URL Format Validation', () => {
  describe('Protocol validation', () => {
    it('accepts HTTP URL', () => {
      const result = urlSchema.safeParse('http://example.com/doc.pdf');
      expect(result.success).toBe(true);
    });

    it('accepts HTTPS URL', () => {
      const result = urlSchema.safeParse('https://example.com/doc.pdf');
      expect(result.success).toBe(true);
    });

    it('rejects FTP URL', () => {
      const result = urlSchema.safeParse('ftp://example.com/doc.pdf');
      expect(result.success).toBe(false);
    });

    it('rejects file:// URL', () => {
      const result = urlSchema.safeParse('file:///etc/passwd');
      expect(result.success).toBe(false);
    });

    it('rejects javascript: URL', () => {
      const result = urlSchema.safeParse('javascript:alert(1)');
      expect(result.success).toBe(false);
    });
  });

  describe('Length validation', () => {
    it('accepts URL under 2048 characters', () => {
      const url = 'https://example.com/' + 'a'.repeat(2000);
      const result = urlSchema.safeParse(url);
      expect(result.success).toBe(true);
    });

    it('rejects URL over 2048 characters', () => {
      const url = 'https://example.com/' + 'a'.repeat(2050);
      const result = urlSchema.safeParse(url);
      expect(result.success).toBe(false);
    });
  });

  describe('Malformed URL handling', () => {
    it('rejects malformed URL with E101', () => {
      const result = urlSchema.safeParse('not-a-url');
      expect(result.success).toBe(false);
      expect(result.error?.issues[0].code).toBe('E101');
    });

    it('rejects URL without protocol', () => {
      const result = urlSchema.safeParse('example.com/doc.pdf');
      expect(result.success).toBe(false);
    });
  });
});
```

#### Deliverables

| Deliverable | Path |
|-------------|------|
| URL format tests | `src/lib/validation/__tests__/url.test.ts` (extended) |

---

### JUL-1.4-003: Write Unit Tests for Document Store

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.4-003 |
| **Title** | Write Unit Tests for Document Store State Management |
| **Priority** | P1 (High) |
| **Effort** | 0.75 hours |
| **Dependencies** | NEO-1.4-004 (Document store created) |

#### Description

Write comprehensive unit tests for the Zustand document store covering state transitions, mutual exclusion, and processing lock behavior.

#### Acceptance Criteria

- [ ] Test initial state is empty
- [ ] Test setFile sets file and clears URL
- [ ] Test setUrl sets URL and clears file
- [ ] Test activeInput tracks current input type
- [ ] Test clearInputs resets all state
- [ ] Test startProcessing sets isProcessing flag
- [ ] Test input changes blocked during processing
- [ ] Test state reset after processing completes
- [ ] Coverage >= 95% for document store

#### Technical Notes

```typescript
// src/stores/__tests__/documentStore.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import { useDocumentStore } from '../documentStore';

describe('documentStore', () => {
  beforeEach(() => {
    // Reset store state before each test
    useDocumentStore.setState({
      file: null,
      url: null,
      activeInput: null,
      isProcessing: false,
      currentJobId: null,
    });
  });

  describe('Initial state', () => {
    it('has null file', () => {
      expect(useDocumentStore.getState().file).toBeNull();
    });

    it('has null url', () => {
      expect(useDocumentStore.getState().url).toBeNull();
    });

    it('has null activeInput', () => {
      expect(useDocumentStore.getState().activeInput).toBeNull();
    });

    it('is not processing', () => {
      expect(useDocumentStore.getState().isProcessing).toBe(false);
    });
  });

  describe('setFile', () => {
    it('sets file and activeInput', () => {
      const mockFile = new File([''], 'test.pdf');
      useDocumentStore.getState().setFile(mockFile);

      expect(useDocumentStore.getState().file).toBe(mockFile);
      expect(useDocumentStore.getState().activeInput).toBe('file');
    });

    it('clears URL when file is set (mutual exclusion)', () => {
      useDocumentStore.getState().setUrl('https://example.com');
      useDocumentStore.getState().setFile(new File([''], 'test.pdf'));

      expect(useDocumentStore.getState().url).toBeNull();
    });
  });

  describe('setUrl', () => {
    it('sets url and activeInput', () => {
      useDocumentStore.getState().setUrl('https://example.com/doc.pdf');

      expect(useDocumentStore.getState().url).toBe('https://example.com/doc.pdf');
      expect(useDocumentStore.getState().activeInput).toBe('url');
    });

    it('clears file when URL is set (mutual exclusion)', () => {
      useDocumentStore.getState().setFile(new File([''], 'test.pdf'));
      useDocumentStore.getState().setUrl('https://example.com');

      expect(useDocumentStore.getState().file).toBeNull();
    });
  });

  describe('Processing lock', () => {
    it('prevents file changes during processing', () => {
      const originalFile = new File([''], 'original.pdf');
      useDocumentStore.getState().setFile(originalFile);
      useDocumentStore.getState().startProcessing('job-123');

      useDocumentStore.getState().setFile(new File([''], 'new.pdf'));

      expect(useDocumentStore.getState().file).toBe(originalFile);
    });

    it('prevents URL changes during processing', () => {
      useDocumentStore.getState().setUrl('https://original.com');
      useDocumentStore.getState().startProcessing('job-123');

      useDocumentStore.getState().setUrl('https://new.com');

      expect(useDocumentStore.getState().url).toBe('https://original.com');
    });
  });
});
```

#### Deliverables

| Deliverable | Path |
|-------------|------|
| Document store tests | `src/stores/__tests__/documentStore.test.ts` |

---

### JUL-1.4-004: Write Component Tests for UrlInput

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.4-004 |
| **Title** | Write Component Tests for UrlInput Component |
| **Priority** | P1 (High) |
| **Effort** | 0.75 hours |
| **Dependencies** | NEO-1.4-001 (UrlInput component created) |

#### Description

Write component tests for the UrlInput component covering user interactions, validation feedback, and accessibility.

#### Acceptance Criteria

- [ ] Test renders input field and submit button
- [ ] Test accepts valid URL input
- [ ] Test shows validation error for invalid URL
- [ ] Test shows E104 error for blocked URL
- [ ] Test clear button functionality
- [ ] Test paste from clipboard support
- [ ] Test disabled state during processing
- [ ] Test keyboard accessibility
- [ ] Test ARIA labels and error announcements
- [ ] Coverage >= 90% for UrlInput component

#### Technical Notes

```typescript
// src/components/upload/__tests__/UrlInput.test.tsx
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { describe, it, expect, vi } from 'vitest';
import { UrlInput } from '../UrlInput';

describe('UrlInput', () => {
  it('renders input field', () => {
    render(<UrlInput onUrlSubmit={vi.fn()} />);
    expect(screen.getByPlaceholderText(/enter url/i)).toBeInTheDocument();
  });

  it('accepts valid URL', async () => {
    const user = userEvent.setup();
    const onUrlSubmit = vi.fn();
    render(<UrlInput onUrlSubmit={onUrlSubmit} />);

    const input = screen.getByRole('textbox');
    await user.type(input, 'https://example.com/doc.pdf');
    await user.click(screen.getByRole('button', { name: /process/i }));

    expect(onUrlSubmit).toHaveBeenCalledWith('https://example.com/doc.pdf');
  });

  it('shows error for invalid URL', async () => {
    const user = userEvent.setup();
    render(<UrlInput onUrlSubmit={vi.fn()} />);

    const input = screen.getByRole('textbox');
    await user.type(input, 'not-a-url');
    await user.click(screen.getByRole('button', { name: /process/i }));

    expect(screen.getByRole('alert')).toHaveTextContent(/invalid url/i);
  });

  it('shows E104 error for blocked URL', async () => {
    const user = userEvent.setup();
    render(<UrlInput onUrlSubmit={vi.fn()} />);

    const input = screen.getByRole('textbox');
    await user.type(input, 'http://192.168.1.1/admin');
    await user.click(screen.getByRole('button', { name: /process/i }));

    expect(screen.getByRole('alert')).toHaveTextContent(/blocked/i);
  });

  it('clears input on clear button click', async () => {
    const user = userEvent.setup();
    render(<UrlInput onUrlSubmit={vi.fn()} />);

    const input = screen.getByRole('textbox');
    await user.type(input, 'https://example.com');
    await user.click(screen.getByRole('button', { name: /clear/i }));

    expect(input).toHaveValue('');
  });

  it('is keyboard accessible', async () => {
    const user = userEvent.setup();
    const onUrlSubmit = vi.fn();
    render(<UrlInput onUrlSubmit={onUrlSubmit} />);

    const input = screen.getByRole('textbox');
    await user.type(input, 'https://example.com/doc.pdf');
    await user.keyboard('{Enter}');

    expect(onUrlSubmit).toHaveBeenCalled();
  });

  it('is disabled during processing', () => {
    render(<UrlInput onUrlSubmit={vi.fn()} disabled />);

    expect(screen.getByRole('textbox')).toBeDisabled();
    expect(screen.getByRole('button', { name: /process/i })).toBeDisabled();
  });
});
```

#### Deliverables

| Deliverable | Path |
|-------------|------|
| UrlInput component tests | `src/components/upload/__tests__/UrlInput.test.tsx` |

---

### JUL-1.4-005: Verify URL Components Accessibility

| Attribute | Value |
|-----------|-------|
| **Task ID** | JUL-1.4-005 |
| **Title** | Verify URL Components Accessibility Compliance |
| **Priority** | P1 (High) |
| **Effort** | 0.5 hours |
| **Dependencies** | JUL-1.4-004 |

#### Description

Run accessibility audit on URL input components and verify screen reader compatibility for error messages.

#### Acceptance Criteria

- [ ] axe-core audit passes with no violations
- [ ] Error messages use aria-live for screen reader announcements
- [ ] Input has proper label association
- [ ] Focus management correct on error
- [ ] Color contrast meets WCAG AA
- [ ] Tab order logical

#### Technical Notes

```typescript
// src/components/upload/__tests__/url-accessibility.test.tsx
import { render } from '@testing-library/react';
import { axe, toHaveNoViolations } from 'jest-axe';
import { describe, it, expect } from 'vitest';
import { UrlInput } from '../UrlInput';

expect.extend(toHaveNoViolations);

describe('UrlInput Accessibility', () => {
  it('has no accessibility violations', async () => {
    const { container } = render(<UrlInput onUrlSubmit={() => {}} />);
    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });

  it('has proper label association', () => {
    render(<UrlInput onUrlSubmit={() => {}} />);
    const input = screen.getByRole('textbox');
    expect(input).toHaveAccessibleName();
  });

  it('announces errors to screen readers', async () => {
    const user = userEvent.setup();
    render(<UrlInput onUrlSubmit={() => {}} />);

    await user.type(screen.getByRole('textbox'), 'invalid');
    await user.click(screen.getByRole('button'));

    const alert = screen.getByRole('alert');
    expect(alert).toHaveAttribute('aria-live', 'assertive');
  });
});
```

#### Deliverables

| Deliverable | Path |
|-------------|------|
| URL accessibility tests | `src/components/upload/__tests__/url-accessibility.test.tsx` |

---

## Sprint 1.4 Testing Summary

| Metric | Target | Notes |
|--------|--------|-------|
| Total Tasks | 5 | |
| Total Effort | 3.75 hours | |
| Critical Tasks | 1 | SSRF prevention (security-critical) |
| High Priority Tasks | 4 | Validation, store, component, accessibility |

### Coverage Targets

| Component | Target Coverage |
|-----------|-----------------|
| `lib/validation/url.ts` | >= 95% (100% for SSRF) |
| `stores/documentStore.ts` | >= 95% |
| `UrlInput.tsx` | >= 90% |

### Quality Gates for Sprint 1.4

- [ ] All SSRF test patterns pass (security-critical)
- [ ] All unit tests pass
- [ ] Coverage >= 80% for sprint components
- [ ] No accessibility violations
- [ ] Document store state transitions verified

---

## Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0.0 | 2025-12-12 | Julia Santos | Initial creation |
