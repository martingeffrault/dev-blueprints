# Storybook 8 (2025-2026)

> **Last updated**: January 2026
> **Versions covered**: 8.x (8.5+), CSF 3, CSF Factories (preview)
> **Integration**: React, Vue, Angular, Svelte, Web Components, Next.js

---

## Philosophy (2025-2026)

Storybook in 2025-2026 has evolved into a **complete component development and testing platform**. It's no longer just a component library viewer but a first-class testing framework integrated with Vitest.

**Key philosophical shifts:**
- **Component-driven development** — Build UIs from the bottom up, in isolation
- **Stories as tests** — Every story is a testable unit (visual, interaction, a11y)
- **Type-safe by design** — CSF Factories provide full TypeScript inference
- **Visual testing first** — Catch UI regressions before they ship
- **Documentation as code** — Autodocs generates living documentation from stories
- **Portable stories** — Reuse stories in Vitest, Playwright, and Jest

---

## TL;DR

- Use CSF 3 format (object-based stories with `args`)
- Enable `autodocs` tag for automatic documentation generation
- Write interaction tests with `play` functions using `@storybook/test`
- Use decorators for providers, themes, and layout wrappers
- Run visual tests with the built-in Visual Tests addon (Chromatic)
- Configure in `.storybook/main.ts` and `.storybook/preview.ts`
- Use Vitest addon for Vite-based projects instead of test-runner
- Leverage CSF Factories (preview) for full type safety

---

## Best Practices

### Project Configuration

#### main.ts

```typescript
// .storybook/main.ts
import type { StorybookConfig } from "@storybook/react-vite";

const config: StorybookConfig = {
  // Required
  framework: "@storybook/react-vite",
  stories: ["../src/**/*.mdx", "../src/**/*.stories.@(js|jsx|ts|tsx)"],

  // Recommended addons
  addons: [
    "@storybook/addon-essentials",    // Controls, actions, docs, viewport, etc.
    "@storybook/addon-a11y",          // Accessibility checks
    "@storybook/addon-interactions",  // Interaction testing panel
    "@chromatic-com/storybook",       // Visual testing
  ],

  // Static assets
  staticDirs: ["../public"],

  // Documentation settings
  docs: {
    autodocs: "tag",  // Generate docs for stories with 'autodocs' tag
  },

  // TypeScript configuration
  typescript: {
    reactDocgen: "react-docgen",  // Faster than react-docgen-typescript
  },
};

export default config;
```

#### preview.ts

```typescript
// .storybook/preview.ts
import type { Preview } from "@storybook/react";
import { themes } from "@storybook/theming";
import "../src/styles/globals.css";  // Global styles

const preview: Preview = {
  parameters: {
    // Control type matchers
    controls: {
      matchers: {
        color: /(background|color)$/i,
        date: /Date$/i,
      },
    },
    // Actions for event handlers
    actions: { argTypesRegex: "^on[A-Z].*" },
    // Layout options
    layout: "centered",  // 'centered' | 'fullscreen' | 'padded'
    // Docs theme
    docs: {
      theme: themes.light,
    },
  },

  // Global decorators
  decorators: [
    // Theme provider decorator
    (Story, context) => (
      <ThemeProvider theme={context.globals.theme}>
        <Story />
      </ThemeProvider>
    ),
  ],

  // Global toolbar controls
  globalTypes: {
    theme: {
      name: "Theme",
      description: "Global theme for components",
      defaultValue: "light",
      toolbar: {
        icon: "circlehollow",
        items: ["light", "dark"],
        showName: true,
      },
    },
  },

  // Default tags for all stories
  tags: ["autodocs"],
};

export default preview;
```

### Story Writing (CSF 3)

#### Basic Story Structure

```typescript
// Button.stories.tsx
import type { Meta, StoryObj } from "@storybook/react";
import { Button } from "./Button";

// Meta configuration (default export)
const meta = {
  title: "Components/Button",
  component: Button,
  tags: ["autodocs"],

  // Default args for all stories
  args: {
    children: "Button",
  },

  // Arg types for controls
  argTypes: {
    variant: {
      control: "select",
      options: ["primary", "secondary", "ghost"],
      description: "Visual style variant",
    },
    size: {
      control: "radio",
      options: ["sm", "md", "lg"],
    },
    onClick: { action: "clicked" },
  },
} satisfies Meta<typeof Button>;

export default meta;
type Story = StoryObj<typeof meta>;

// Named exports = stories
export const Primary: Story = {
  args: {
    variant: "primary",
  },
};

export const Secondary: Story = {
  args: {
    variant: "secondary",
  },
};

export const Large: Story = {
  args: {
    size: "lg",
  },
};

export const WithIcon: Story = {
  args: {
    children: (
      <>
        <PlusIcon /> Add Item
      </>
    ),
  },
};
```

#### Stories with Complex Props

```typescript
// Card.stories.tsx
import type { Meta, StoryObj } from "@storybook/react";
import { Card } from "./Card";

const meta = {
  title: "Components/Card",
  component: Card,
  // Render function for complex composition
  render: (args) => (
    <Card {...args}>
      <Card.Header>
        <Card.Title>{args.title}</Card.Title>
      </Card.Header>
      <Card.Content>{args.content}</Card.Content>
      <Card.Footer>{args.footer}</Card.Footer>
    </Card>
  ),
  args: {
    title: "Card Title",
    content: "Card content goes here",
    footer: "Card footer",
  },
} satisfies Meta<typeof Card>;

export default meta;
type Story = StoryObj<typeof meta>;

export const Default: Story = {};

export const WithImage: Story = {
  render: (args) => (
    <Card {...args}>
      <Card.Image src="/placeholder.jpg" alt="Placeholder" />
      <Card.Header>
        <Card.Title>{args.title}</Card.Title>
      </Card.Header>
      <Card.Content>{args.content}</Card.Content>
    </Card>
  ),
};
```

### Decorators

#### Component-Level Decorator

```typescript
// Modal.stories.tsx
const meta = {
  title: "Components/Modal",
  component: Modal,
  decorators: [
    // Provide necessary context
    (Story) => (
      <ModalProvider>
        <Story />
      </ModalProvider>
    ),
    // Add padding/layout
    (Story) => (
      <div style={{ minHeight: "400px" }}>
        <Story />
      </div>
    ),
  ],
} satisfies Meta<typeof Modal>;
```

#### Story-Level Decorator

```typescript
export const DarkMode: Story = {
  decorators: [
    (Story) => (
      <div className="dark bg-gray-900 p-4">
        <Story />
      </div>
    ),
  ],
};
```

#### Global Decorators (preview.ts)

```typescript
// .storybook/preview.ts
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";

const queryClient = new QueryClient({
  defaultOptions: {
    queries: { retry: false },
  },
});

const preview: Preview = {
  decorators: [
    // React Query provider
    (Story) => (
      <QueryClientProvider client={queryClient}>
        <Story />
      </QueryClientProvider>
    ),
    // Router mock
    (Story) => (
      <MemoryRouter>
        <Story />
      </MemoryRouter>
    ),
    // i18n provider
    (Story) => (
      <I18nextProvider i18n={i18n}>
        <Story />
      </I18nextProvider>
    ),
  ],
};
```

### Interaction Testing

#### Basic Play Function

```typescript
// LoginForm.stories.tsx
import type { Meta, StoryObj } from "@storybook/react";
import { expect, fn, userEvent, within } from "@storybook/test";
import { LoginForm } from "./LoginForm";

const meta = {
  title: "Forms/LoginForm",
  component: LoginForm,
  args: {
    onSubmit: fn(),  // Mock function
  },
} satisfies Meta<typeof LoginForm>;

export default meta;
type Story = StoryObj<typeof meta>;

export const FilledForm: Story = {
  play: async ({ canvasElement, args }) => {
    const canvas = within(canvasElement);

    // Type in email field
    await userEvent.type(
      canvas.getByLabelText(/email/i),
      "user@example.com"
    );

    // Type in password field
    await userEvent.type(
      canvas.getByLabelText(/password/i),
      "password123"
    );

    // Click submit
    await userEvent.click(canvas.getByRole("button", { name: /sign in/i }));

    // Assert the callback was called
    await expect(args.onSubmit).toHaveBeenCalledWith({
      email: "user@example.com",
      password: "password123",
    });
  },
};

export const ValidationError: Story = {
  play: async ({ canvasElement }) => {
    const canvas = within(canvasElement);

    // Submit empty form
    await userEvent.click(canvas.getByRole("button", { name: /sign in/i }));

    // Assert error message appears
    await expect(canvas.getByText(/email is required/i)).toBeVisible();
  },
};
```

#### Testing Async Behavior

```typescript
// SearchInput.stories.tsx
import { expect, userEvent, within, waitFor } from "@storybook/test";

export const WithSearchResults: Story = {
  play: async ({ canvasElement }) => {
    const canvas = within(canvasElement);

    // Type search query
    await userEvent.type(canvas.getByRole("searchbox"), "react");

    // Wait for debounced results
    await waitFor(
      () => expect(canvas.getByText(/results found/i)).toBeVisible(),
      { timeout: 2000 }
    );

    // Verify results rendered
    const results = canvas.getAllByRole("listitem");
    await expect(results.length).toBeGreaterThan(0);
  },
};
```

### Documentation with MDX

#### Custom Documentation Page

```mdx
{/* Button.mdx */}
import { Canvas, Meta, Story, Controls, ArgTypes } from "@storybook/blocks";
import * as ButtonStories from "./Button.stories";

<Meta of={ButtonStories} />

# Button

Buttons trigger actions when clicked. Use them for form submissions,
dialogs, and other interactive elements.

## Usage

```tsx
import { Button } from "@/components/Button";

<Button variant="primary" onClick={handleClick}>
  Click me
</Button>
```

## Interactive Demo

<Canvas of={ButtonStories.Primary} />

## Controls

<Controls />

## All Variants

### Primary

<Canvas of={ButtonStories.Primary} />

### Secondary

<Canvas of={ButtonStories.Secondary} />

### Ghost

<Canvas of={ButtonStories.Ghost} />

## Props

<ArgTypes of={ButtonStories} />

## Accessibility

- Always include descriptive text or `aria-label`
- Disabled buttons should use `disabled` attribute, not just styling
- Ensure sufficient color contrast (4.5:1 minimum)
```

### Visual Testing

#### Chromatic Configuration

```typescript
// Button.stories.tsx
export const AllVariants: Story = {
  parameters: {
    // Chromatic-specific options
    chromatic: {
      modes: {
        light: { theme: "light" },
        dark: { theme: "dark" },
      },
      viewports: [320, 768, 1200],
    },
  },
  render: () => (
    <div className="flex flex-col gap-4">
      <Button variant="primary">Primary</Button>
      <Button variant="secondary">Secondary</Button>
      <Button variant="ghost">Ghost</Button>
    </div>
  ),
};

// Skip visual testing for specific stories
export const Animated: Story = {
  parameters: {
    chromatic: { disableSnapshot: true },
  },
};
```

#### Running Visual Tests

```bash
# Install Chromatic
npm install --save-dev chromatic

# Run visual tests (CI)
npx chromatic --project-token=<your-token>

# Run from Storybook UI
# Click the "Visual Tests" addon panel
```

### Portable Stories (Vitest Integration)

```typescript
// Button.test.tsx
import { composeStories } from "@storybook/react";
import { render, screen } from "@testing-library/react";
import { describe, it, expect } from "vitest";
import * as stories from "./Button.stories";

// Compose all stories with decorators/args applied
const { Primary, Secondary, WithIcon } = composeStories(stories);

describe("Button", () => {
  it("renders primary variant", () => {
    render(<Primary />);
    expect(screen.getByRole("button")).toHaveClass("btn-primary");
  });

  it("renders secondary variant", () => {
    render(<Secondary />);
    expect(screen.getByRole("button")).toHaveClass("btn-secondary");
  });

  it("renders with icon", () => {
    render(<WithIcon />);
    expect(screen.getByRole("button")).toContainElement(
      screen.getByTestId("plus-icon")
    );
  });
});
```

### CSF Factories (Storybook 10 Preview)

```typescript
// .storybook/preview.ts (using factories)
import { definePreview } from "@storybook/react-vite";
import addonDocs from "@storybook/addon-docs";
import addonA11y from "@storybook/addon-a11y";

export default definePreview({
  addons: [addonDocs(), addonA11y()],
  parameters: {
    layout: "centered",
  },
});

// Button.stories.tsx (using factories)
import preview from "#.storybook/preview";
import { Button } from "./Button";

const meta = preview.meta({
  component: Button,
  args: {
    children: "Button",
  },
});

export const Primary = meta.story({
  args: {
    variant: "primary",
  },
});

export const Secondary = meta.story({
  args: {
    variant: "secondary",
  },
});
```

---

## Anti-Patterns

### Hiding States Under Controls

**Why it's bad**: Each state becomes undocumented, untestable, and unshareable.

```typescript
// DON'T - Single story with knobs for everything
export const Button: Story = {
  args: {
    variant: "primary",  // User must manually toggle to see other variants
    disabled: false,
    loading: false,
  },
};

// DO - Explicit stories for each meaningful state
export const Primary: Story = {
  args: { variant: "primary" },
};

export const Secondary: Story = {
  args: { variant: "secondary" },
};

export const Disabled: Story = {
  args: { variant: "primary", disabled: true },
};

export const Loading: Story = {
  args: { variant: "primary", loading: true },
};
```

### Mixing Connected and Presentational Logic

**Why it's bad**: Stories become dependent on external services/APIs.

```typescript
// DON'T - Component fetches its own data
function UserCard({ userId }: { userId: string }) {
  const { data: user } = useQuery(["user", userId], fetchUser);
  return <div>{user?.name}</div>;
}

// Story requires mocking the entire fetch chain
export const Default: Story = {
  args: { userId: "123" },
  // Requires MSW, fetch mocks, etc.
};

// DO - Presentational component with data as props
function UserCard({ user }: { user: User }) {
  return <div>{user.name}</div>;
}

// Stories are simple and explicit
export const Default: Story = {
  args: {
    user: { id: "123", name: "Alice", email: "alice@example.com" },
  },
};
```

### Inconsistent Story Naming

**Why it's bad**: Hard to find components, breaks URL sharing.

```typescript
// DON'T - Inconsistent hierarchy
// Components/buttons/MainButton
// forms/Input
// UI-Elements/Card

// DO - Consistent hierarchy
// Components/Buttons/Button
// Components/Forms/Input
// Components/Cards/Card

const meta = {
  title: "Components/Buttons/Button",  // Consistent pattern
  component: Button,
} satisfies Meta<typeof Button>;
```

### Skipping Type Annotations

**Why it's bad**: Loses autodocs generation, no IDE support.

```typescript
// DON'T - No types
const meta = {
  component: Button,
};

export default meta;

export const Primary = {
  args: { variant: "primary" },
};

// DO - Full type safety
import type { Meta, StoryObj } from "@storybook/react";

const meta = {
  component: Button,
} satisfies Meta<typeof Button>;

export default meta;
type Story = StoryObj<typeof meta>;

export const Primary: Story = {
  args: { variant: "primary" },
};
```

### Not Using Play Functions for Interactive States

**Why it's bad**: Cannot test or document interactive behavior.

```typescript
// DON'T - Static modal story (can't see open state behavior)
export const Modal: Story = {
  args: { isOpen: true },
};

// DO - Use play function to demonstrate interaction
export const OpenModal: Story = {
  play: async ({ canvasElement }) => {
    const canvas = within(canvasElement);
    await userEvent.click(canvas.getByRole("button", { name: /open modal/i }));
    await expect(canvas.getByRole("dialog")).toBeVisible();
  },
};
```

### Duplicating Story Data

**Why it's bad**: Maintenance burden, inconsistencies.

```typescript
// DON'T - Duplicate data in every story
export const CardWithUser: Story = {
  args: {
    user: { id: "1", name: "Alice", email: "alice@example.com" },
  },
};

export const AnotherCardWithUser: Story = {
  args: {
    user: { id: "1", name: "Alice", email: "alice@example.com" },  // Duplicated!
  },
};

// DO - Extract mock data
// mocks/users.ts
export const mockUser = { id: "1", name: "Alice", email: "alice@example.com" };

// Stories
import { mockUser } from "@/mocks/users";

export const CardWithUser: Story = {
  args: { user: mockUser },
};
```

### Ignoring Accessibility Addon Results

**Why it's bad**: Ships inaccessible components to production.

```typescript
// DON'T - Disable a11y checks to "fix" failing tests
export const IconButton: Story = {
  parameters: {
    a11y: { disable: true },  // Hiding the problem
  },
};

// DO - Fix the accessibility issue
export const IconButton: Story = {
  args: {
    "aria-label": "Add item",  // Proper accessible name
    icon: <PlusIcon />,
  },
};
```

---

## 2025-2026 Changelog

| Version | Date | Key Changes |
|---------|------|-------------|
| 8.0 | Mar 2024 | Visual Tests addon, Vite 5, 2-4x faster builds, mobile UX, RSC support |
| 8.1 | Apr 2024 | Portable stories for Playwright CT (experimental) |
| 8.2 | Jun 2024 | Vitest addon (experimental), improved test coverage |
| 8.3 | Aug 2024 | Test addon improvements, better error messages |
| 8.4 | Oct 2024 | ESM/CJS compatibility, test panel improvements |
| 8.5 | Jan 2025 | Realtime a11y tests, code coverage, focused tests, RN Web Vite |
| 8.6 | Feb 2025 | Storybook Test enhancements, a11y vision filter lock |
| **9.0** | **Jun 2025** | **Vitest collab, 48% leaner, story globals, React Native Web** |
| 10.0 | Nov 2025 | CSF Factories (preview), ESM-only, 29% smaller, module automocking |

### Storybook 8.5 Highlights

**Realtime Accessibility Tests** — A11y addon now runs Axe checks as you develop:
```typescript
// Automatically enabled with @storybook/addon-a11y
// Results appear in the Accessibility panel in real-time
```

**Code Coverage** — Built-in coverage reporting with Vitest:
```bash
# Enable coverage in vitest.config.ts
# Results visible in Storybook UI
```

**Focused Tests** — Test single stories or components:
```bash
# Right-click a story in sidebar -> "Run tests"
# Or use the test panel to filter
```

### Storybook 9 Highlights (June 2025)

Storybook 9 is a huge release focused on testing and bundle size.

**Core Improvements:**
- **48% leaner bundle size** through flatter dependency structure
- Interaction tests with Vitest collaboration
- Accessibility tests (built-in)
- Visual tests
- Coverage reports
- Story generation

**Story Globals:**
```typescript
// v9: Assign global parameters per individual story
export const DarkTheme: Story = {
  globals: {
    theme: 'dark',
    viewport: 'mobile',
    locale: 'es-ES',
  },
};
```

**Tag-Based Organization:**
Stories can now use tags for better filtering and organization.

**Framework Support in v9:**
- Next.js + Vite: Zero-config support, smoother than ever
- Svelte 5: Full support including runes, #snippet, new compiler directives
- React Native: Full-featured web interface and device testing support
- Angular 18, Lit 3, Vue 3.4+ with latest internals

**Vitest 3 Integration:**
Addon Test now includes Vitest 3 support. `@vitest/coverage-v8` is added during postinstall if no coverage reporter is installed.

### Storybook 10 Preview Features

**CSF Factories** — Full type safety without boilerplate:
```typescript
// Chain: definePreview -> preview.meta -> meta.story
const meta = preview.meta({ component: Button });
export const Primary = meta.story({ args: { variant: "primary" } });
```

**Module Automocking** — Automatic mocking for imports:
```typescript
// Storybook 10 can auto-mock modules referenced in stories
```

---

## Quick Reference

### Configuration Files

| File | Purpose |
|------|---------|
| `.storybook/main.ts` | Framework, addons, story globs, build config |
| `.storybook/preview.ts` | Decorators, parameters, global types |
| `.storybook/manager.ts` | UI customization, sidebar config |

### Common Addons

| Addon | Purpose |
|-------|---------|
| `@storybook/addon-essentials` | Controls, actions, docs, viewport, backgrounds |
| `@storybook/addon-a11y` | Accessibility checks (axe-core) |
| `@storybook/addon-interactions` | Interaction test debugging panel |
| `@chromatic-com/storybook` | Visual regression testing |
| `@storybook/addon-coverage` | Code coverage reporting |

### Story Tags

| Tag | Effect |
|-----|--------|
| `autodocs` | Generate automatic documentation page |
| `!autodocs` | Exclude from automatic documentation |
| `!dev` | Hide from sidebar in development |
| `!test` | Exclude from test runs |

### Testing Utilities

| Import | Usage |
|--------|-------|
| `expect` | Assertions (`expect(element).toBeVisible()`) |
| `fn` | Mock functions (`fn()`) |
| `userEvent` | User interactions (`userEvent.click()`) |
| `within` | Scoped queries (`within(canvas).getByRole()`) |
| `waitFor` | Async assertions (`waitFor(() => expect(...))`) |

### Commands

| Command | Description |
|---------|-------------|
| `npx storybook dev` | Start development server |
| `npx storybook build` | Build static Storybook |
| `npx chromatic` | Run visual tests |
| `npx storybook test` | Run interaction tests |

---

## Resources

- [Official Storybook Documentation](https://storybook.js.org/docs)
- [Storybook 8 Release Blog](https://storybook.js.org/blog/storybook-8/)
- [CSF 3 Documentation](https://storybook.js.org/docs/api/csf)
- [CSF Factories RFC](https://github.com/storybookjs/storybook/discussions/30112)
- [Interaction Testing Guide](https://storybook.js.org/docs/writing-tests/interaction-testing)
- [Visual Testing with Chromatic](https://www.chromatic.com/docs/)
- [Portable Stories Guide](https://storybook.js.org/docs/api/portable-stories/portable-stories-vitest)
- [Storybook Tutorials](https://storybook.js.org/tutorials/)
