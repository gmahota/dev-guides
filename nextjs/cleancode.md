# CleanCode — Next.js (React + TypeScript)

Sem floreados. Pega, cola, corre. Foco: **clean code rápido**, com guardrails e prompts para o Copilot.

---

## 1) Objetivo
- Garantir **padrão mínimo de qualidade** (lint, format, types, testes) sem discussão.
- Facilitar **refactors pelo Copilot** com prompts curtos.
- Reduzir tempo gasto em “clean code” manual.

---

## 2) Setup (1x por projeto)

```bash
# deps base
pnpm add next react react-dom zod
pnpm add -D typescript @types/node @types/react @types/react-dom

# lint/format/test + qualidade
pnpm add -D eslint prettier eslint-config-prettier \
  eslint-plugin-import eslint-plugin-react eslint-plugin-react-hooks \
  eslint-plugin-jsx-a11y eslint-plugin-unicorn eslint-plugin-sonarjs \
  vitest @testing-library/react @testing-library/jest-dom @testing-library/user-event \
  jsdom @vitest/coverage-v8 \
  husky lint-staged rimraf

# iniciar hooks git
pnpm dlx husky init
```

Cria estes ficheiros no **raiz** do repo:

**.editorconfig**
```
root = true
[*]
charset = utf-8
end_of_line = lf
indent_style = space
indent_size = 2
insert_final_newline = true
trim_trailing_whitespace = true
```

**.prettierrc**
```json
{ "singleQuote": true, "semi": true, "printWidth": 100, "trailingComma": "all" }
```

**.eslintrc.json**
```json
{
  "env": { "browser": true, "es2022": true, "jest": true },
  "extends": [
    "next/core-web-vitals",
    "plugin:react-hooks/recommended",
    "plugin:jsx-a11y/recommended",
    "plugin:import/recommended",
    "plugin:unicorn/recommended",
    "plugin:sonarjs/recommended",
    "prettier"
  ],
  "settings": { "react": { "version": "detect" } },
  "plugins": ["import", "jsx-a11y", "unicorn", "sonarjs"],
  "rules": {
    "no-console": ["warn", { "allow": ["warn", "error", "info"] }],
    "import/order": ["warn", {
      "groups": [["builtin","external"],["internal"],["parent","sibling","index"]],
      "newlines-between": "always",
      "alphabetize": { "order": "asc", "caseInsensitive": true }
    }],
    "unicorn/filename-case": ["error", { "case": "kebabCase", "ignore": ["^\\\\w+\\\\.tsx?$"] }],
    "unicorn/prevent-abbreviations": "off",
    "react/jsx-no-useless-fragment": "warn"
  }
}
```

**tsconfig.json**
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "jsx": "preserve",
    "allowJs": false,
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "exactOptionalPropertyTypes": true,
    "baseUrl": ".",
    "paths": { "@/*": ["src/*"] },
    "types": ["vitest/globals", "jest-dom"]
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx"],
  "exclude": ["node_modules"]
}
```

**next.config.ts**
```ts
import type { NextConfig } from 'next';
const nextConfig: NextConfig = {
  reactStrictMode: true,
  experimental: { typedRoutes: true }
};
export default nextConfig;
```

**package.json (scripts + lint-staged)** *(mantém o que já tens, garante estes scripts)*
```json
{
  "scripts": {
    "dev": "next dev",
    "build": "rimraf .next && next build",
    "start": "next start",
    "lint": "eslint . --ext .ts,.tsx",
    "format": "prettier -w .",
    "type-check": "tsc --noEmit",
    "test": "vitest run",
    "test:ui": "vitest --ui",
    "check-all": "pnpm lint && pnpm type-check && pnpm test && pnpm build"
  },
  "lint-staged": {
    "*.{ts,tsx,js,jsx}": ["eslint --fix", "prettier -w"],
    "*.{json,md,css,scss}": ["prettier -w"]
  }
}
```

**.husky/pre-commit**
```sh
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"
pnpm lint-staged
```

**.env.example**
```
NEXT_PUBLIC_API_BASE_URL=http://localhost:3001
```

---

## 3) Estrutura de pastas
```
src/
  app/
    layout.tsx
    page.tsx
    error.tsx            # boundary por rota
    globals.css
    health/route.ts      # endpoint simples de saúde
  components/            # UI pura (sem side-effects)
    button.tsx
  lib/
    fetcher.ts
    env.ts
    schemas/
      user.ts
  tests/
    setup.ts
```

**src/app/error.tsx**
```tsx
'use client';
import React from 'react';

export default function Error({ error, reset }: { error: Error; reset: () => void }) {
  return (
    <div>
      <h2>Ocorreu um erro.</h2>
      <pre>{error.message}</pre>
      <button onClick={() => reset()}>Tentar novamente</button>
    </div>
  );
}
```

**src/lib/env.ts**
```ts
export function getEnv(key: string): string {
  const value = process.env[key] ?? '';
  if (!value) throw new Error(`Missing env: ${key}`);
  return value;
}
```

**src/lib/fetcher.ts**
```ts
import { ZodSchema } from 'zod';

export async function fetchJson<T>(path: string, schema: ZodSchema<T>, init?: RequestInit) {
  const base = process.env.NEXT_PUBLIC_API_BASE_URL ?? '';
  const res = await fetch(`${base}${path}`, init);
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  const data = await res.json();
  return schema.parse(data);
}
```

**src/components/button.tsx**
```tsx
import React from 'react';

type ButtonProps = {
  onClick?: () => void;
  children: React.ReactNode;
  disabled?: boolean;
  type?: 'button' | 'submit' | 'reset';
};

export function Button({ onClick, children, disabled, type = 'button' }: ButtonProps) {
  return (
    <button type={type} onClick={onClick} disabled={disabled} aria-disabled={disabled}>
      {children}
    </button>
  );
}
```

**src/tests/setup.ts**
```ts
import '@testing-library/jest-dom';
```
**vitest.config.ts (raiz)**
```ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    environment: 'jsdom',
    setupFiles: ['src/tests/setup.ts'],
    coverage: { reporter: ['text', 'html'] }
  }
});
```

---

## 4) Guardrails de qualidade (usar sempre)
- **WIP = 1** (um foco por vez).
- Commits passam por **pre-commit** (lint-staged).
- `pnpm check-all` antes de pedir PR/merge.

**Mini-checklist de Clean Code**
- [ ] Tipos explícitos em props e retornos públicos  
- [ ] Zod valida toda resposta de API antes de usar  
- [ ] Sem `any` e sem `as unknown as`  
- [ ] Sem lógica pesada no JSX (extrai para funções)  
- [ ] Hooks no topo; deps completas  
- [ ] Nomes verbais para acções, substantivos para dados  
- [ ] Erros tratados com mensagem útil (não engolir `catch`)  
- [ ] Imports ordenados; sem circulares  
- [ ] Testes mínimos (feliz + inválido)

---

## 5) Workflow rápido (se não tocas no código no trabalho)
- **Manhã (15 min):** “Copilot, aplica *Refactor clean* (abaixo) a 3 ficheiros grandes.” → `pnpm check-all` → guarda o output.
- **Almoço (10 min):** “Copilot, corrige erros do ESLint/SonarJS neste log: <cola o output>.”
- **Fim do dia (15–20 min):** “Copilot, gera testes Vitest mínimos para componentes alterados.” → Commit: `chore(clean): lint+refactor+tests <data>`.

---

## 6) Prompts úteis para o Copilot

**A) Refactor clean (por ficheiro)**
> Revê este ficheiro Next/TS e aplica clean code: nomes explícitos, early-returns, extrai UI pura sem side-effects, respeita Regras de Hooks, remove código morto. Devolve **patch diff**.

**B) Zod + fetch**
> Cria schema Zod a partir deste JSON e função `loadUsers()` que usa `fetchJson` e valida resposta. Inclui teste Vitest (caso feliz + inválido).

**C) Quebra de componente grande**
> Divide em `Container` (dados/efeitos) e `View` (UI pura), mantendo props claras e reduzindo rerenders (keys estáveis e callbacks memorizados só onde precise). Devolve **patch diff**.

**D) Checklist de PR**
> Gera checklist de PR para este diff cobrindo: segurança (XSS), acessibilidade (labels/aria), performance (re-render), compatibilidade (peer deps), e casos de erro.

---

## 7) README curto (cola no repo)
```
# Project Name

## Correr
1) pnpm i
2) cp .env.example .env.local
3) pnpm dev  # http://localhost:3000

## Verificações
- pnpm check-all  # lint + types + tests + build

## Estrutura
- app/: rotas (App Router)
- components/: UI pura
- lib/: fetch/env/schemas
- tests/: setup Vitest
```
---

## 8) Modelo de PR
```
### Objetivo
- <o que muda>

### Checklist
- [ ] Lint OK
- [ ] Type-check OK
- [ ] Testes mínimos (feliz + inválido)
- [ ] Zod valida I/O de API
- [ ] Sem any/ts-ignore
- [ ] Sem lógica pesada no JSX
- [ ] Hooks no topo e deps completas
- [ ] Acessibilidade básica (labels/aria)

### Notas de risco
- <quebras potenciais / migrações>
```
---

## 9) Notas rápidas (evitar bugs comuns)
- **Server vs Client** no App Router: se precisares de estado/efeitos, primeira linha do ficheiro: `use client`.
- **ESM/CJS:** mantém coerência; não misturar. Em Next, usa ESM por omissão.
- **Peer deps:** instala pares compatíveis (`react-dom` vs `react`).
- **Dedup:** `pnpm dedupe` quando surgirem versões duplicadas.
- **AbortController:** cancela fetches em desmontagem para evitar “setState after unmount”.
