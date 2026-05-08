---
name: openhands-design
description: Build or restyle frontend UI to match the OpenHands design system: dark-first surfaces, semantic color tokens, shared Tailwind primitives, and disciplined spacing, typography, and motion. Use when the user wants screens, components, or styling to feel like OpenHands-Design.
triggers:
- openhands design
- openhands design system
- openhands ui design
- match openhands styling
- semantic token ui
---

# OpenHands Design

Use this skill when implementing or refining UI to match the `OpenHands-Design` system distilled from the OpenHands prototype.

This is a **first-pass reference skill**. Apply it pragmatically inside the host project instead of forcing a full rewrite.

## First principles

- Discover the host project's existing design assets before adding new ones.
- Keep the product shell **dark-first**. Do not invent or ship a full light theme unless the user explicitly asks for that work.
- Use **semantic CSS-variable tokens** for colors and radii instead of raw Tailwind palette classes or arbitrary values.
- Reuse shared UI primitives such as `Button`, `Input`, `SearchInput`, and `NativeSelect` when they exist.
- Treat `text-sm` as the default body size, `rounded-md` as the default element radius, and `rounded-lg` as the default container radius.
- Use `gap-2` as the standard internal spacing and `p-4` as the standard card padding.
- Use `transition-colors duration-200` for most interactive feedback.
- Use `hover:bg-muted/60` on dark interactive surfaces and `hover:bg-primary/85` on white or primary buttons.

## Workflow

### 1. Discover the local design foundation

Before changing UI, look for the project's existing equivalents of:

- global token CSS such as `globals.css` or `index.css`
- `tailwind.config.js` or `tailwind.config.ts`
- shared primitives in `src/components/ui/`
- helper utilities such as `cn()` / `tailwind-merge`
- any local design guide or copied `OpenHands-Design/` folder

If the project already has an OpenHands-style token and component foundation, extend it instead of duplicating or replacing it.

### 2. Establish the minimum token set

When the host project lacks the needed theme foundation, add only the smallest set of tokens required for the requested UI. Prioritize these semantic tokens:

- surfaces: `background`, `card`, `secondary`, `muted`, `border`
- text: `foreground`, `muted-foreground`, `primary`, `primary-foreground`
- status: `success`, `warning`, `info`, `destructive`
- interaction: `ring`, `muted-hover`

Prefer Tailwind mappings like `hsl(var(--token))` so the UI stays themeable.

### 3. Build with canonical primitives and recipes

- **Buttons:** prefer the shared `Button` component and its variants instead of handwritten button class strings.
- **Inputs:** prefer `Input`, `SearchInput`, and `NativeSelect` so focus, hover, and sizing stay consistent.
- **Cards:** compose with `bg-card border border-border rounded-lg p-4` unless the design clearly needs a more elevated treatment.
- **Text:** keep primary text on `text-foreground` and secondary copy on `text-muted-foreground`.
- **Icons:** default to `text-muted-foreground` and brighten on hover with `group-hover:`.
- **Status states:** use semantic tokens like `text-success-foreground`, `text-warning`, `text-info`, and `text-destructive`.

### 4. Apply layout, interaction, and motion conventions

- Default control height is `h-10`; compact controls can use `h-9`.
- Use `gap-2` for rows and small groups, `gap-4` for larger sections.
- Prefer `rounded-md` for controls and `rounded-lg` for cards and panels.
- Use `focus-visible:ring-1 focus-visible:ring-ring focus-visible:ring-offset-2` for keyboard-visible focus.
- Prefer `transition-colors duration-200` and avoid `transition-all` unless multiple property types genuinely change.
- Avoid press-scale behavior on shared buttons unless the existing project already uses it intentionally.

### 5. Audit before finishing

Before you stop, scan for avoidable drift:

- raw palette classes such as `stone-*`, `gray-*`, `slate-*`, `neutral-*`
- arbitrary radii or font sizes where the standard scale already works
- white buttons using dark-surface hover styles
- `focus:` styles where `focus-visible:` should be used
- inconsistent hover opacity variants on dark surfaces

## Guardrails

- Do **not** use raw palette classes for standard UI when semantic tokens exist.
- Do **not** style white or primary buttons with `hover:bg-muted/60`.
- Do **not** use `focus:ring-2` or click-triggered focus styles as the default pattern.
- Do **not** rebuild shared primitives from scratch if the project already has them.
- Do **not** add a full light theme just because Tailwind `dark:` exists.
- Do **not** over-normalize legacy code if the user asked for a narrow UI change.

## Companion files

- See [REFERENCE.md](./REFERENCE.md) for tokens, recipes, and an audit checklist.
- See [EXAMPLES.md](./EXAMPLES.md) for component snippets and prompt patterns.
