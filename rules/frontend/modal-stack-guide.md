# Modal Stack Management Guide

## Overview

This guide covers the implementation of modal and dropdown management with proper keyboard navigation support, specifically ESC key handling with a modal stack.

## ⚠️ MANDATORY REQUIREMENTS

1. **All modals MUST support ESC key to close**
2. **All select dropdowns MUST support ESC key to close**
3. **MUST implement a modal stack (LIFO - Last In, First Out)**
4. **ESC key MUST close only the last opened modal/dropdown**

## Why Modal Stack?

When multiple modals or dropdowns are open simultaneously:
- User opens Modal A
- Inside Modal A, user opens Dropdown B
- Inside Modal A, user opens Modal C

Pressing ESC should close Modal C first, then Dropdown B, then Modal A (LIFO order).

## Architecture

```
┌─────────────────────────────────┐
│  Modal Stack Manager             │
│  ┌───────────────────────────┐  │
│  │ Stack: [Modal A, Drop B,  │  │
│  │         Modal C]           │  │
│  └───────────────────────────┘  │
│                                  │
│  ESC Key → Close Modal C (last) │
└─────────────────────────────────┘
```

## Implementation

### 1. Modal Stack Manager

Create a global modal stack manager using React Context:

```typescript
// src/lib/modal-stack/ModalStackContext.tsx
import { createContext, useContext, useCallback, useEffect, useState, ReactNode } from 'react';

interface ModalStackItem {
  id: string;
  onClose: () => void;
  type: 'modal' | 'dropdown' | 'select';
}

interface ModalStackContextValue {
  push: (item: ModalStackItem) => void;
  pop: (id: string) => void;
  closeTop: () => void;
}

const ModalStackContext = createContext<ModalStackContextValue | undefined>(undefined);

export const ModalStackProvider = ({ children }: { children: ReactNode }) => {
  const [stack, setStack] = useState<ModalStackItem[]>([]);

  const push = useCallback((item: ModalStackItem) => {
    setStack((prev) => [...prev, item]);
  }, []);

  const pop = useCallback((id: string) => {
    setStack((prev) => prev.filter((item) => item.id !== id));
  }, []);

  const closeTop = useCallback(() => {
    const topItem = stack[stack.length - 1];
    if (topItem) {
      topItem.onClose();
    }
  }, [stack]);

  // Global ESC key handler
  useEffect(() => {
    const handleEscape = (event: KeyboardEvent) => {
      if (event.key === 'Escape') {
        event.preventDefault();
        event.stopPropagation();
        closeTop();
      }
    };

    if (stack.length > 0) {
      document.addEventListener('keydown', handleEscape, { capture: true });
      return () => {
        document.removeEventListener('keydown', handleEscape, { capture: true });
      };
    }
  }, [stack, closeTop]);

  return (
    <ModalStackContext.Provider value={{ push, pop, closeTop }}>
      {children}
    </ModalStackContext.Provider>
  );
};

export const useModalStack = () => {
  const context = useContext(ModalStackContext);
  if (!context) {
    throw new Error('useModalStack must be used within ModalStackProvider');
  }
  return context;
};
```

### 2. Setup Provider in App

```typescript
// src/main.tsx
import { ModalStackProvider } from './lib/modal-stack/ModalStackContext';

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <ModalStackProvider>
      <App />
    </ModalStackProvider>
  </React.StrictMode>
);
```

### 3. Modal Component with Stack Integration

```typescript
// src/components/ui/Modal.tsx
import { useEffect, useId } from 'react';
import { useModalStack } from '@/lib/modal-stack/ModalStackContext';
import { X } from 'lucide-react';

interface ModalProps {
  isOpen: boolean;
  onClose: () => void;
  title: string;
  children: React.ReactNode;
}

export const Modal = ({ isOpen, onClose, title, children }: ModalProps) => {
  const modalId = useId();
  const { push, pop } = useModalStack();

  useEffect(() => {
    if (isOpen) {
      // Register modal in stack when opened
      push({
        id: modalId,
        onClose,
        type: 'modal',
      });

      // Unregister when closed
      return () => {
        pop(modalId);
      };
    }
  }, [isOpen, modalId, push, pop, onClose]);

  if (!isOpen) return null;

  return (
    <div
      className="fixed inset-0 z-50 flex items-center justify-center"
      data-testid="modal-overlay"
    >
      {/* Backdrop */}
      <div
        className="fixed inset-0 bg-black/50"
        onClick={onClose}
        data-testid="modal-backdrop"
      />

      {/* Modal Content */}
      <div
        className="relative z-50 bg-white rounded-lg shadow-xl max-w-md w-full mx-4"
        role="dialog"
        aria-modal="true"
        aria-labelledby={`${modalId}-title`}
        data-testid="modal-content"
      >
        {/* Header */}
        <div className="flex items-center justify-between p-4 border-b">
          <h2 id={`${modalId}-title`} className="text-lg font-semibold">
            {title}
          </h2>
          <button
            onClick={onClose}
            className="p-1 hover:bg-gray-100 rounded"
            aria-label="Close modal"
            data-testid="modal-close-button"
          >
            <X className="w-5 h-5" />
          </button>
        </div>

        {/* Body */}
        <div className="p-4">{children}</div>
      </div>
    </div>
  );
};
```

### 4. Select/Dropdown Component with Stack Integration

```typescript
// src/components/ui/Select.tsx
import { useState, useEffect, useId, useRef } from 'react';
import { useModalStack } from '@/lib/modal-stack/ModalStackContext';
import { ChevronDown } from 'lucide-react';

interface SelectOption {
  value: string;
  label: string;
}

interface SelectProps {
  options: SelectOption[];
  value?: string;
  onChange: (value: string) => void;
  placeholder?: string;
}

export const Select = ({ options, value, onChange, placeholder = 'Select...' }: SelectProps) => {
  const [isOpen, setIsOpen] = useState(false);
  const selectId = useId();
  const { push, pop } = useModalStack();
  const ref = useRef<HTMLDivElement>(null);

  const handleClose = () => setIsOpen(false);

  useEffect(() => {
    if (isOpen) {
      // Register dropdown in stack when opened
      push({
        id: selectId,
        onClose: handleClose,
        type: 'dropdown',
      });

      // Unregister when closed
      return () => {
        pop(selectId);
      };
    }
  }, [isOpen, selectId, push, pop]);

  const selectedOption = options.find((opt) => opt.value === value);

  return (
    <div className="relative" ref={ref} data-testid="select-container">
      {/* Trigger Button */}
      <button
        onClick={() => setIsOpen(!isOpen)}
        className="w-full flex items-center justify-between px-3 py-2 border rounded-md bg-white hover:bg-gray-50"
        aria-haspopup="listbox"
        aria-expanded={isOpen}
        data-testid="select-trigger"
      >
        <span>{selectedOption?.label || placeholder}</span>
        <ChevronDown className={`w-4 h-4 transition-transform ${isOpen ? 'rotate-180' : ''}`} />
      </button>

      {/* Dropdown Menu */}
      {isOpen && (
        <>
          {/* Backdrop */}
          <div
            className="fixed inset-0 z-40"
            onClick={handleClose}
            data-testid="select-backdrop"
          />

          {/* Options */}
          <div
            className="absolute z-50 w-full mt-1 bg-white border rounded-md shadow-lg max-h-60 overflow-auto"
            role="listbox"
            data-testid="select-dropdown"
          >
            {options.map((option) => (
              <button
                key={option.value}
                onClick={() => {
                  onChange(option.value);
                  handleClose();
                }}
                className={`w-full text-left px-3 py-2 hover:bg-gray-100 ${
                  option.value === value ? 'bg-blue-50' : ''
                }`}
                role="option"
                aria-selected={option.value === value}
                data-testid={`select-option-${option.value}`}
              >
                {option.label}
              </button>
            ))}
          </div>
        </>
      )}
    </div>
  );
};
```

### 5. shadcn/ui Dialog Integration

If using shadcn/ui Dialog component, wrap it with stack integration:

```typescript
// src/components/ui/dialog-with-stack.tsx
import { useEffect, useId } from 'react';
import { Dialog, DialogContent, DialogHeader, DialogTitle } from '@/components/ui/dialog';
import { useModalStack } from '@/lib/modal-stack/ModalStackContext';

interface DialogWithStackProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  title: string;
  children: React.ReactNode;
}

export const DialogWithStack = ({ open, onOpenChange, title, children }: DialogWithStackProps) => {
  const dialogId = useId();
  const { push, pop } = useModalStack();

  useEffect(() => {
    if (open) {
      push({
        id: dialogId,
        onClose: () => onOpenChange(false),
        type: 'modal',
      });

      return () => {
        pop(dialogId);
      };
    }
  }, [open, dialogId, push, pop, onOpenChange]);

  return (
    <Dialog open={open} onOpenChange={onOpenChange}>
      <DialogContent>
        <DialogHeader>
          <DialogTitle>{title}</DialogTitle>
        </DialogHeader>
        {children}
      </DialogContent>
    </Dialog>
  );
};
```

## Usage Examples

### Example 1: Simple Modal

```typescript
import { useState } from 'react';
import { Modal } from '@/components/ui/Modal';

function UserProfile() {
  const [isOpen, setIsOpen] = useState(false);

  return (
    <>
      <button onClick={() => setIsOpen(true)}>Open Profile</button>

      <Modal isOpen={isOpen} onClose={() => setIsOpen(false)} title="User Profile">
        <p>Profile content here</p>
      </Modal>
    </>
  );
}
```

### Example 2: Nested Modals with Dropdown

```typescript
import { useState } from 'react';
import { Modal } from '@/components/ui/Modal';
import { Select } from '@/components/ui/Select';

function ComplexForm() {
  const [modal1Open, setModal1Open] = useState(false);
  const [modal2Open, setModal2Open] = useState(false);
  const [country, setCountry] = useState('');

  const countries = [
    { value: 'us', label: 'United States' },
    { value: 'uk', label: 'United Kingdom' },
    { value: 'ca', label: 'Canada' },
  ];

  return (
    <>
      <button onClick={() => setModal1Open(true)}>Open Form</button>

      {/* First Modal */}
      <Modal isOpen={modal1Open} onClose={() => setModal1Open(false)} title="User Form">
        <div className="space-y-4">
          <Select
            options={countries}
            value={country}
            onChange={setCountry}
            placeholder="Select country"
          />

          <button onClick={() => setModal2Open(true)}>
            Open Additional Info
          </button>
        </div>
      </Modal>

      {/* Second Modal (nested) */}
      <Modal isOpen={modal2Open} onClose={() => setModal2Open(false)} title="Additional Info">
        <p>More information here</p>
      </Modal>
    </>
  );
}

// ESC key behavior:
// 1. Open Modal 1 → ESC closes Modal 1
// 2. Open Modal 1 → Open Select → ESC closes Select
// 3. Open Modal 1 → Open Modal 2 → ESC closes Modal 2 (not Modal 1)
// 4. Open Modal 1 → Open Select → Open Modal 2 → ESC closes Modal 2, then Select, then Modal 1
```

## Testing with Playwright

### Page Object for Modal

```typescript
// e2e/pages/components/ModalComponent.ts
import { Page, Locator } from '@playwright/test';

export class ModalComponent {
  readonly page: Page;
  readonly overlay: Locator;
  readonly content: Locator;
  readonly closeButton: Locator;

  constructor(page: Page) {
    this.page = page;
    this.overlay = page.getByTestId('modal-overlay');
    this.content = page.getByTestId('modal-content');
    this.closeButton = page.getByTestId('modal-close-button');
  }

  async expectVisible() {
    await expect(this.content).toBeVisible();
  }

  async expectHidden() {
    await expect(this.content).not.toBeVisible();
  }

  async closeWithButton() {
    await this.closeButton.click();
  }

  async closeWithEscape() {
    await this.page.keyboard.press('Escape');
  }

  async closeWithBackdrop() {
    // Click backdrop (outside modal content)
    await this.overlay.click({ position: { x: 10, y: 10 } });
  }
}
```

### Test Modal Stack Behavior

```typescript
// e2e/tests/modal-stack.spec.ts
import { test, expect } from '@playwright/test';
import { HomePage } from '../pages/HomePage';
import { ModalComponent } from '../pages/components/ModalComponent';

test.describe('Modal Stack Management', () => {
  test('should close modals in LIFO order with ESC key', async ({ page }) => {
    const homePage = new HomePage(page);
    const modal1 = new ModalComponent(page);
    const modal2 = new ModalComponent(page);

    await homePage.goto();

    // Open first modal
    await homePage.clickOpenModal1();
    await modal1.expectVisible();

    // Open second modal
    await homePage.clickOpenModal2();
    await modal2.expectVisible();

    // Press ESC - should close second modal only
    await page.keyboard.press('Escape');
    await modal2.expectHidden();
    await modal1.expectVisible();

    // Press ESC again - should close first modal
    await page.keyboard.press('Escape');
    await modal1.expectHidden();
  });

  test('should close dropdown before modal when ESC pressed', async ({ page }) => {
    const homePage = new HomePage(page);
    const modal = new ModalComponent(page);

    await homePage.goto();

    // Open modal
    await homePage.clickOpenModal();
    await modal.expectVisible();

    // Open dropdown inside modal
    await page.getByTestId('select-trigger').click();
    await expect(page.getByTestId('select-dropdown')).toBeVisible();

    // Press ESC - should close dropdown only
    await page.keyboard.press('Escape');
    await expect(page.getByTestId('select-dropdown')).not.toBeVisible();
    await modal.expectVisible();

    // Press ESC again - should close modal
    await page.keyboard.press('Escape');
    await modal.expectHidden();
  });
});
```

## Best Practices

### 1. Always Register in Stack

Every modal, dialog, dropdown, or select MUST register itself in the modal stack when opened.

### 2. Use Unique IDs

Use `useId()` hook to generate unique IDs for each modal instance.

### 3. Clean Up on Unmount

Always unregister from stack in the cleanup function of `useEffect`.

### 4. Prevent Event Bubbling

In the global ESC handler, use `event.stopPropagation()` to prevent multiple handlers from firing.

### 5. Z-Index Management

Ensure proper z-index values:
- Backdrop: z-40
- First modal: z-50
- Dropdown in modal: z-50 (relative to modal)
- Nested modal: z-60

### 6. Focus Management

When a modal opens, trap focus inside it. When it closes, return focus to the trigger element.

```typescript
useEffect(() => {
  if (isOpen) {
    const previousActiveElement = document.activeElement as HTMLElement;

    // Focus modal content
    modalRef.current?.focus();

    return () => {
      // Return focus to previous element
      previousActiveElement?.focus();
    };
  }
}, [isOpen]);
```

## Accessibility Considerations

1. **ARIA Attributes**: Use proper ARIA roles and attributes
   - `role="dialog"`
   - `aria-modal="true"`
   - `aria-labelledby` for title
   - `aria-describedby` for description

2. **Focus Trap**: Implement focus trapping within modal

3. **Screen Reader Announcements**: Announce when modals open/close

4. **Keyboard Navigation**: Support Tab, Shift+Tab, ESC

## Common Pitfalls

### ❌ Mistake 1: Not Using the Stack

```typescript
// ❌ BAD: Handling ESC directly in modal
useEffect(() => {
  const handleEscape = (e: KeyboardEvent) => {
    if (e.key === 'Escape') onClose();
  };
  document.addEventListener('keydown', handleEscape);
  return () => document.removeEventListener('keydown', handleEscape);
}, [onClose]);
```

This will close ALL modals when ESC is pressed, not just the topmost one.

### ❌ Mistake 2: Not Cleaning Up

```typescript
// ❌ BAD: Not removing from stack on unmount
useEffect(() => {
  if (isOpen) {
    push({ id: modalId, onClose, type: 'modal' });
  }
  // Missing cleanup!
}, [isOpen]);
```

### ❌ Mistake 3: Using Same ID for Multiple Instances

```typescript
// ❌ BAD: Hardcoded ID
const modalId = 'my-modal'; // Will cause issues with multiple instances
```

Always use `useId()` hook for unique IDs.

## Related Documentation

- [Frontend Development Guide](./frontend-development-guide.md)
- [Playwright E2E Guide](./playwright-e2e-guide.md)
- [Form Validation Guide](./form-validation-guide.md)
