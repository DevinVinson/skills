# OpenHands Design Examples

## Example prompts

### Match the design system

> Build this UI following the OpenHands design system. Use semantic tokens for colors, `text-sm` as the default body size, `rounded-md` for controls, `rounded-lg` for containers, `hover:bg-muted/60` for dark interactive surfaces, and `hover:bg-primary/85` for primary buttons.

### Extend an existing screen

> Restyle this feature to feel native to OpenHands. Reuse the project's existing `Button`, `Input`, and other `components/ui` primitives before creating new styles.

## Example: settings card

```tsx
import { Button } from '@/components/ui/button';

export function SettingsCard() {
  return (
    <section className="bg-card border border-border rounded-lg p-4">
      <div className="flex items-start justify-between gap-4">
        <div className="space-y-1">
          <h2 className="text-lg font-semibold text-foreground">Workspace</h2>
          <p className="text-sm text-muted-foreground">
            Control how the agent uses your current project.
          </p>
        </div>
        <Button variant="outline">Edit</Button>
      </div>
    </section>
  );
}
```

## Example: search toolbar

```tsx
import { Button } from '@/components/ui/button';
import { SearchInput } from '@/components/ui/search-input';

export function SearchToolbar({ value, onValueChange }: { value: string; onValueChange: (value: string) => void }) {
  return (
    <div className="flex flex-col gap-3 rounded-lg border border-border bg-card p-4 md:flex-row md:items-center md:justify-between">
      <SearchInput
        value={value}
        onValueChange={onValueChange}
        placeholder="Search conversations"
        className="md:max-w-sm"
      />
      <div className="flex items-center gap-2">
        <Button variant="outline">Filter</Button>
        <Button>New conversation</Button>
      </div>
    </div>
  );
}
```

## Example: labeled form field

```tsx
import { Input } from '@/components/ui/input';
import { NativeSelect } from '@/components/ui/native-select';

export function ProfileFields() {
  return (
    <div className="grid gap-4 md:grid-cols-2">
      <label className="space-y-1.5">
        <span className="text-sm font-medium text-foreground">Display name</span>
        <Input placeholder="Ada Lovelace" />
        <span className="text-xs text-muted-foreground">Shown anywhere your work is visible.</span>
      </label>

      <label className="space-y-1.5">
        <span className="text-sm font-medium text-foreground">Default model</span>
        <NativeSelect defaultValue="gpt-4.1">
          <option value="gpt-4.1">GPT-4.1</option>
          <option value="claude-sonnet">Claude Sonnet</option>
        </NativeSelect>
        <span className="text-xs text-muted-foreground">Used when you start a new conversation.</span>
      </label>
    </div>
  );
}
```

## Example: status row

```tsx
export function JobStatus() {
  return (
    <div className="flex items-center justify-between gap-4 rounded-lg border border-border bg-card p-4">
      <div className="space-y-1">
        <p className="text-sm font-medium text-foreground">Background sync</p>
        <p className="text-sm text-muted-foreground">Last run completed 2 minutes ago.</p>
      </div>
      <span className="inline-flex items-center rounded-full bg-success/10 px-2 py-0.5 text-xs font-medium text-success-foreground">
        Healthy
      </span>
    </div>
  );
}
```

## Example: interactive menu row

```tsx
import { ChevronRight } from 'lucide-react';

export function MenuRow() {
  return (
    <button className="group flex w-full items-center justify-between rounded-md px-3 py-1.5 text-sm text-muted-foreground transition-colors hover:bg-muted/60 hover:text-foreground">
      <span className="flex items-center gap-2">
        <ChevronRight className="h-4 w-4 shrink-0 text-muted-foreground transition-colors group-hover:text-foreground" />
        Open logs
      </span>
    </button>
  );
}
```
