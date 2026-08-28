---
name: agent-canvas-extension-builder
description: This skill should be used when the user asks to "create an Agent Canvas extension", "build a Canvas Extension", "add a page to Agent Canvas", "scaffold canvas-extension.json", "make an OpenHands Canvas addon", "validate a Canvas Extension", or mentions the Agent Canvas extension host API, registerPage, extension pages, or Canvas extension manifests.
---

# Agent Canvas Extension Builder

Create, extend, and validate OpenHands Agent Canvas Extensions that target the currently implemented manifest schema 1 and host API 1 routed-page ABI.

Treat extensions as trusted, same-realm browser code owned by the active Agent Server. Distinguish them from skills and plugins: extensions change the Agent Canvas application; skills and plugins change agent behavior.

## Establish the target

Start by locating the target directory instead of assuming the current workspace. Inspect repository instructions, existing package management, build tooling, tests, and git status before editing.

Clarify only choices that materially affect implementation:

- extension purpose and page behavior;
- target repository and subdirectory;
- dependency-free JavaScript versus a bundled TypeScript/framework project;
- Agent Server endpoints or host metadata required;
- whether to build only, install locally, or prepare publication instructions.

Default to one extension with one routed page when requirements are otherwise clear. Keep the first implementation small and dependency-free unless the requested UI clearly benefits from a framework or the repository already has a bundler.

When creating multiple extensions in one repository, establish a monorepo layout before implementation. Give every extension an independent package root containing its own `canvas-extension.json`, entrypoint, version, tests, and optional README. Prefer stable subpaths such as `extensions/<extension-name>/`. Share source modules and build tooling only when each extension still emits an independent self-contained ESM entrypoint. Keep extension manifest names globally distinct within the Agent Server installation.

Treat installation as one extension per request. The current Customize -> Extensions flow accepts one `source`, optional `ref`, and optional `repo_path`; it does not recursively discover or bulk-install every manifest in a repository. Add each extension separately using the same source/ref and its own `repo_path`.

Read `references/v1-contract.md` before implementing unfamiliar host API behavior. Read `references/testing-and-installation.md` before installing or testing inside Agent Canvas, especially for a multi-extension repository.

## Inspect current upstream behavior

Treat the v1 contract as young and subject to change. Before substantial work, compare the installed or target OpenHands version with the current upstream sources when access is available:

- `specs/canvas-extensions.md`
- `src/types/canvas-extension.ts`
- `src/components/features/canvas-extensions/canvas-extensions-runtime.tsx`
- `src/routes/canvas-extension-page.tsx`
- `docs/CANVAS_EXTENSIONS_TESTING.md`

Do not invent planned surfaces such as conversation tabs, slots, themes, or visualizer replacement. Implement only contributions supported by the target version. At the initial landing commit, routed pages are the only implemented contribution.

## Create the package

Place `canvas-extension.json` at the extension root. Point `entrypoint` to one self-contained browser ESM file inside that root.

Use this minimal manifest shape:

```json
{
  "schema_version": 1,
  "name": "example-dashboard",
  "display_name": "Example dashboard",
  "version": "0.1.0",
  "description": "A backend-specific project dashboard.",
  "entrypoint": "extension.js",
  "contributes": {
    "pages": [
      {
        "id": "dashboard",
        "title": "Dashboard",
        "path": "/dashboard",
        "nav_label": "Dashboard"
      }
    ]
  }
}
```

Follow these invariants:

- Use lowercase letters, digits, and hyphens for extension names and page IDs.
- Use absolute kebab-case page paths with a leading slash.
- Keep every manifest path within the extension root.
- Give each page ID and path a unique value.
- Register only page IDs declared in the manifest.
- Keep the entrypoint free of unresolved bare imports and external chunks.
- Bundle dependencies, CSS, and small assets into the entrypoint when using build tooling.

Add a concise `README.md` when the extension is intended for reuse or publication. Document purpose, build command if any, output entrypoint, installation coordinate, and verification steps. Avoid adding explanatory change-log documents.

## Implement activation and pages

Export an `activate` function from the ESM entrypoint:

```js
export function activate(host) {
  if (host.apiVersion !== "1") {
    throw new Error(`This extension requires host API 1.`);
  }

  return host.registerPage("dashboard", ({ container, path, navigate }) => {
    const root = document.createElement("section");
    root.textContent = path ? `Nested route: ${path}` : "Dashboard";
    container.append(root);

    return () => root.remove();
  });
}
```

Use the host contract deliberately:

- Read immutable metadata from `host.extension` and `host.backend`.
- Call `host.registerPage(id, mount)` during activation.
- Use the mount callback's `path` as the route remainder below the declared page path.
- Use `navigate(absoluteCanvasPath)` for Canvas-aware routing.
- Use `host.agentServer.request({ path, method, body, headers })` for authenticated calls to the owning Agent Server.
- Pass only root-relative request paths beginning with one `/`; never pass absolute URLs or `//` paths.

Return cleanup from every layer that creates effects:

- Return the unregister function or an activation disposer from `activate`.
- Return a mount disposer for DOM nodes, timers, listeners, observers, subscriptions, and outstanding state.
- Make async work disposal-aware so late responses cannot mutate an unmounted page.
- Remove injected styles when the page unmounts.

Assume activation, mounting, and disposal may happen repeatedly during hot enable/disable, updates, backend switches, reconnects, and route changes.

## Build a production-quality page

Render within the supplied `container`; do not replace unrelated Canvas DOM. Scope CSS under an extension-specific root class. Prefer Canvas CSS variables with sensible fallbacks rather than copying host implementation classes.

Provide:

- semantic structure and an accessible page label;
- keyboard-operable controls and visible focus;
- responsive behavior for narrow layouts;
- loading, empty, stale, and error states;
- reduced-motion handling for animation;
- text-safe rendering through DOM APIs or escaping rather than untrusted `innerHTML`;
- clear handling for malformed or partial Agent Server responses.

Avoid global event handlers, prototype changes, global CSS selectors, and ambient state unless unavoidable. Same-realm execution means these effects have full Canvas authority and cleanup is only best-effort.

## Add tests

Test real extension code with a DOM environment and a small host test double. Test at minimum:

1. `activate` registers every declared page exactly once.
2. The mount renders its initial state.
3. Agent Server requests use the expected root-relative path and method.
4. Nested route remainders render correctly or navigate correctly.
5. The mount disposer removes DOM and stops effects.
6. The activation disposer unregisters contributions.
7. Unsupported host API versions fail clearly when compatibility is checked.
8. Network and malformed-response errors render safely.

Use the repository's existing test infrastructure. Do not introduce a new framework when adequate tests already exist. For dependency-free fixtures, Vitest with a DOM environment matches the upstream example, but it is not part of the extension runtime contract.

## Validate the package

Run the bundled validator before reporting completion:

```sh
node /path/to/agent-canvas-extension-builder/scripts/validate-extension.mjs /path/to/extension
```

Then run the target repository's formatter, linter, tests, and build. Inspect the final entrypoint rather than assuming the bundler configuration worked:

- confirm the file exists at the manifest path;
- confirm it exports `activate`;
- reject bare imports and dynamic external chunks;
- confirm CSS and required small assets are bundled or embedded;
- confirm the output runs as browser ESM without Node globals.

Treat validator warnings as prompts for inspection, not proof of invalidity. The helper uses conservative static checks and cannot replace loading the bundle in Canvas.

## Install and verify safely

Install only when requested. Installation and enablement are separate product actions: installation must leave the extension disabled, and enabling executes trusted same-realm code.

For a backend-local extension, install the path as interpreted on the Agent Server machine. For a Git repository, provide `source`, optional `ref`, and optional `repo_path`. Never assume a frontend-local path exists inside a remote or containerized backend.

For multiple extensions in one repository, produce an install matrix listing extension name, manifest directory, source, ref, and `repo_path`. Submit one Add extension operation per row. Omit `repo_path` only when the selected extension lives at the repository root. Do not claim that selecting the repository root installs nested extensions.

Validate and test every extension package independently, then run shared repository checks once. Report partial failures by extension name rather than treating one passing package as validation of the entire repository.

Verify the lifecycle in Canvas:

1. Install and confirm the inventory entry is disabled.
2. Review source, revision, manifest metadata, and contributions.
3. Enable and accept the trusted-code disclosure.
4. Open the navigation item and exercise root and nested routes.
5. Disable and confirm navigation and mounted UI disappear.
6. Re-enable without restarting Canvas.
7. Switch backend or reconnect when relevant and confirm isolation.
8. Uninstall only when requested.

Do not enable, uninstall, publish, push, or open a pull request without the user's authorization.

## Report completion

Summarize:

- extension location and contributed pages;
- manifest schema and host API targeted;
- build output and validation results;
- tests run and their results;
- installation status, explicitly stating whether it remains disabled;
- remaining limitations tied to the current v1 ABI.

## Resources

- `references/v1-contract.md` — exact manifest, host API, lifecycle, routing, trust, and current limitations.
- `references/testing-and-installation.md` — test strategy, manual Canvas workflow, and installation coordinates.
- `scripts/validate-extension.mjs` — dependency-free static validator for an extension directory.
