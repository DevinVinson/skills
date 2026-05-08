# OpenHands Design Reference

This skill is derived from the `OpenHands-Design` package and its `DESIGN.md` guide.

## Core visual model

- **Theme:** dark-first, near-black monochrome shell
- **Fonts:** `Inter` for UI text, `JetBrains Mono` for code and technical labels
- **Default text:** `text-sm font-normal text-foreground`
- **Secondary text:** `text-sm text-muted-foreground`
- **Heading:** `text-lg` or `text-2xl` with `font-semibold`
- **Default motion:** `transition-colors duration-200`

## Core semantic tokens

| Concern | Token / class | Typical use |
|---|---|---|
| Page background | `bg-background` | app shell, full-page surfaces |
| Card surface | `bg-card` | cards, panels, drawers |
| Secondary surface | `bg-secondary` | secondary panels, toolbar fills |
| Muted surface | `bg-muted` | pills, subtle fills, hover bases |
| Border | `border-border` | borders and dividers |
| Primary text | `text-foreground` | titles and normal text |
| Secondary text | `text-muted-foreground` | helper text, metadata, placeholders |
| Primary action bg | `bg-primary` | white/high-emphasis buttons |
| Primary action text | `text-primary-foreground` | text on `bg-primary` |
| Success | `text-success-foreground` / `bg-success/10` | success states |
| Warning | `text-warning` / `bg-warning/10` | caution or in-progress states |
| Info | `text-info` / `bg-info/10` | links and informational states |
| Destructive | `text-destructive` / `bg-destructive/10` | error and danger states |
| Focus ring | `ring-ring` | keyboard-visible focus |

## Canonical spacing and radius

| Pattern | Default |
|---|---|
| Standard item gap | `gap-2` |
| Section gap | `gap-4` |
| Card padding | `p-4` |
| Dialog / generous panel padding | `p-6` |
| Default control height | `h-10` |
| Compact control height | `h-9` |
| Element radius | `rounded-md` |
| Container radius | `rounded-lg` |
| Modal radius | `rounded-modal` or `rounded-xl` |
| Pill / avatar radius | `rounded-full` |

## Canonical hover and focus behavior

- **Dark interactive surfaces:** `hover:bg-muted/60`
- **Primary / white buttons:** `hover:bg-primary/85`
- **Focus pattern:** `focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring focus-visible:ring-offset-2`
- Prefer `focus-visible:` over `focus:` for shared controls.

## Shared component conventions

### Button

Prefer a shared `Button` primitive with variants matching these roles:

| Variant | Use |
|---|---|
| `default` | primary CTA |
| `destructive` | delete / danger |
| `outline` | common secondary action |
| `light` | high-contrast primary on dark backgrounds |
| `secondary` | tertiary action |
| `muted` | subdued action |
| `ghost` | minimal chrome action |
| `link` | inline action |

Primary button convention:

```tsx
<Button className="bg-primary text-primary-foreground hover:bg-primary/85" />
```

Avoid:

```tsx
<button className="bg-white text-black hover:bg-muted/60" />
```

### Cards and containers

Standard card:

```tsx
<div className="bg-card border border-border rounded-lg p-4" />
```

Elevated card:

```tsx
<div className="bg-card border border-border rounded-xl p-6 shadow-lg" />
```

### Inputs

Canonical input recipe:

```tsx
className="h-10 w-full rounded-md border border-border bg-muted/40 px-3 py-2 text-base md:text-sm placeholder:text-muted-foreground ring-offset-background focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring focus-visible:ring-offset-2 focus-visible:bg-muted/60 hover:bg-muted/60"
```

Use the shared `Input` component when possible.

### Search input

- Build on top of `Input`
- Standard sizes: `sm`, `default`, `lg`
- Search icon stays `text-muted-foreground`
- Clear button should match the same muted hover and focus treatment

### Status badges

Success pill example:

```tsx
<span className="inline-flex items-center rounded-full bg-success/10 px-2 py-0.5 text-xs font-medium text-success-foreground">
  Running
</span>
```

### Menus and interactive rows

Text rows inside dark menus commonly use:

```tsx
className="group flex items-center gap-2 rounded-md px-3 py-1.5 text-sm text-muted-foreground transition-colors hover:bg-muted/60 hover:text-foreground"
```

## Do and do not

### Do

- Use semantic tokens instead of raw palette classes.
- Default to `text-sm` for UI body copy.
- Keep icons muted until hover or active state.
- Normalize to `rounded-md` and `rounded-lg` instead of arbitrary pixel radii.
- Use `transition-colors duration-200` for standard interaction feedback.

### Do not

- Use `text-white`, `bg-black`, or raw `stone-*` classes for standard app UI.
- Mix `hover:bg-muted/40`, `/50`, `/70` randomly when `/60` is the intended default.
- Put dark hover fills on primary white buttons.
- Default to `transition-all` when only colors change.
- Add a full light theme unless the user asked for it.

## Fast audit checklist

Before finishing UI work, quickly check for:

- raw palette drift: `stone-`, `gray-`, `slate-`, `neutral-`
- arbitrary classes worth normalizing: `rounded-[`, `text-[`, `p-[`, `px-[`, `py-[`
- incorrect primary button hover: `bg-white text-black hover:bg-muted/60`
- focus drift: `focus:ring-2`, `focus:bg-`, or other click-triggered focus patterns

Useful searches:

```bash
grep -RInE 'stone-|gray-|slate-|neutral-' src
```

```bash
grep -RInE 'rounded-\[|text-\[|p-\[|px-\[|py-\[' src
```
