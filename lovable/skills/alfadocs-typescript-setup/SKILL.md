---
name: alfadocs-typescript-setup
description: Use when setting up, scaffolding, or reviewing the TypeScript configuration (tsconfig.json) of an AlfaDocs/Lovable app — the strict baseline that Engineering Standard §3 requires. Load this when starting a project or when a type-safety review needs the exact config.
---

# TypeScript setup — the strict baseline (§3)

This is the `tsconfig.json` baseline for every AlfaDocs/Lovable app. **Never weaken
`strict` or any `no*` flag** — they are what keep boundary bugs out.

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "noImplicitReturns": true,
    "noUncheckedIndexedAccess": true,
    "skipLibCheck": true,
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "useDefineForClassFields": true,
    "noEmit": true,
    "allowImportingTsExtensions": true,
    "baseUrl": ".",
    "paths": { "@/*": ["./src/*"] }
  },
  "include": ["src"],
  "references": [{ "path": "./tsconfig.node.json" }]
}
```

## Why these flags matter

- **`strict`** — the umbrella for `strictNullChecks`, `noImplicitAny`, etc. Everything below assumes it's on.
- **`noUncheckedIndexedAccess`** — `array[i]` and `record[key]` are typed as possibly `undefined`. Narrow before use; don't assert.
- **`noUnusedLocals` / `noUnusedParameters`** — remove unused symbols rather than prefixing with `_`.
- **`noImplicitReturns` / `noFallthroughCasesInSwitch`** — every branch returns when a return type is declared; switches use `break`/`return` and a `never` default for exhaustiveness.
- **`paths: { "@/*": ["./src/*"] }`** — import internals with the `@/*` alias, never `../../../` chains.

## Companion rules (Engineering Standard §3)

- No `any`, no `as` outside type guards, no `@ts-ignore`. Prefer `unknown` and narrow.
- Every service method has an explicit return type referencing a domain model.
- **Validate every external response** (HTTP, webhook, storage) with a schema (e.g. Zod) **at the service boundary**, then map to the domain model.
- Fail loudly on shape drift — **no `x.id ?? x.Id ?? x.ID` fallback chains**; fix the type at the boundary.

`tsconfig.node.json` (referenced above) holds the Vite/tooling config and may stay
loose; the app config (`src`) is the one that must be strict.
