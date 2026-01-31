# Node.js Project Configuration Context

Comprehensive guide to Node.js project setup, package.json, and tooling configuration.

## Modern package.json

```json
{
  "name": "@org/package-name",
  "version": "1.0.0",
  "description": "A well-configured Node.js package",
  "type": "module",
  "main": "./dist/index.cjs",
  "module": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "import": {
        "types": "./dist/index.d.ts",
        "default": "./dist/index.js"
      },
      "require": {
        "types": "./dist/index.d.cts",
        "default": "./dist/index.cjs"
      }
    },
    "./utils": {
      "import": "./dist/utils.js",
      "require": "./dist/utils.cjs"
    }
  },
  "files": ["dist"],
  "sideEffects": false,
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview",
    "lint": "eslint . --ext .ts,.tsx",
    "lint:fix": "eslint . --ext .ts,.tsx --fix",
    "format": "prettier --write .",
    "format:check": "prettier --check .",
    "typecheck": "tsc --noEmit",
    "test": "vitest",
    "test:ui": "vitest --ui",
    "test:coverage": "vitest --coverage",
    "prepare": "husky",
    "preinstall": "npx only-allow pnpm"
  },
  "keywords": ["keyword1", "keyword2"],
  "author": "Your Name <email@example.com>",
  "license": "MIT",
  "repository": {
    "type": "git",
    "url": "git+https://github.com/user/repo.git"
  },
  "bugs": {
    "url": "https://github.com/user/repo/issues"
  },
  "homepage": "https://github.com/user/repo#readme",
  "engines": {
    "node": ">=20.0.0",
    "pnpm": ">=9.0.0"
  },
  "packageManager": "pnpm@9.0.0"
}
```

## Script Patterns

### Development Scripts
```json
{
  "scripts": {
    "dev": "next dev",
    "dev:turbo": "next dev --turbo",
    "dev:debug": "NODE_OPTIONS='--inspect' next dev",
    "dev:mock": "MSW_ENABLED=true next dev"
  }
}
```

### Build Scripts
```json
{
  "scripts": {
    "build": "tsc && vite build",
    "build:prod": "NODE_ENV=production npm run build",
    "build:analyze": "ANALYZE=true npm run build",
    "build:clean": "rm -rf dist && npm run build"
  }
}
```

### Test Scripts
```json
{
  "scripts": {
    "test": "vitest",
    "test:watch": "vitest watch",
    "test:ui": "vitest --ui",
    "test:coverage": "vitest --coverage",
    "test:e2e": "playwright test",
    "test:e2e:ui": "playwright test --ui"
  }
}
```

### Quality Scripts
```json
{
  "scripts": {
    "lint": "eslint . --ext .ts,.tsx --report-unused-disable-directives --max-warnings 0",
    "lint:fix": "npm run lint -- --fix",
    "format": "prettier --write \"**/*.{ts,tsx,md,json}\"",
    "format:check": "prettier --check \"**/*.{ts,tsx,md,json}\"",
    "typecheck": "tsc --noEmit",
    "check": "npm run lint && npm run typecheck && npm run format:check"
  }
}
```

### Composite Scripts
```json
{
  "scripts": {
    "validate": "npm-run-all --parallel lint typecheck test:coverage",
    "ci": "npm-run-all --sequential build test:coverage",
    "release": "npm run validate && changeset publish"
  }
}
```

## ESM vs CommonJS

### ESM Project (Recommended)
```json
{
  "type": "module"
}
```

```javascript
// Works
import { something } from './module.js' // Note: .js extension needed
export const value = 42

// CommonJS in ESM context
import { createRequire } from 'module'
const require = createRequire(import.meta.url)
const pkg = require('./package.json')
```

### Dual Package (ESM + CJS)
```json
{
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs"
    }
  }
}
```

## TypeScript Configuration

### tsconfig.json (Node.js)
```json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022"],
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "outDir": "./dist",
    "rootDir": "./src",
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "verbatimModuleSyntax": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### tsconfig.json (React/Next.js)
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["DOM", "DOM.Iterable", "ES2022"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "preserve",
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "isolatedModules": true,
    "incremental": true,
    "plugins": [{ "name": "next" }],
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

## Prettier Configuration

```json
// .prettierrc
{
  "semi": false,
  "singleQuote": true,
  "trailingComma": "es5",
  "tabWidth": 2,
  "printWidth": 100,
  "plugins": ["prettier-plugin-tailwindcss"],
  "tailwindFunctions": ["cn", "clsx"]
}
```

```javascript
// prettier.config.mjs
/** @type {import("prettier").Config} */
export default {
  semi: false,
  singleQuote: true,
  trailingComma: 'es5',
  plugins: ['prettier-plugin-tailwindcss'],
}
```

## Git Hooks with Husky + lint-staged

```bash
# Setup
npm install -D husky lint-staged
npx husky init
```

```json
// package.json
{
  "lint-staged": {
    "*.{ts,tsx}": ["eslint --fix", "prettier --write"],
    "*.{json,md,yml}": ["prettier --write"]
  }
}
```

```bash
# .husky/pre-commit
npx lint-staged

# .husky/commit-msg
npx --no -- commitlint --edit $1
```

## Commitlint Configuration

```javascript
// commitlint.config.js
export default {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [
      2,
      'always',
      ['feat', 'fix', 'docs', 'style', 'refactor', 'perf', 'test', 'chore', 'ci', 'revert'],
    ],
    'scope-case': [2, 'always', 'lower-case'],
    'subject-case': [2, 'never', ['start-case', 'pascal-case', 'upper-case']],
  },
}
```

## NPM/PNPM Configuration

### .npmrc
```ini
# Use exact versions
save-exact=true

# Hoist node_modules (for compatibility)
shamefully-hoist=true

# Registry
registry=https://registry.npmjs.org/

# Scoped packages
@myorg:registry=https://npm.myorg.com/
```

### pnpm-workspace.yaml
```yaml
packages:
  - 'apps/*'
  - 'packages/*'
  - 'tooling/*'
```

## Environment Files

### .nvmrc
```
20
```

### .node-version
```
20.11.0
```

## Project Files Checklist

```
project/
├── .github/
│   └── workflows/
│       └── ci.yml
├── .husky/
│   ├── pre-commit
│   └── commit-msg
├── src/
├── tests/
├── .env.example
├── .eslintrc.cjs
├── .gitignore
├── .npmrc
├── .nvmrc
├── .prettierrc
├── commitlint.config.js
├── package.json
├── pnpm-lock.yaml
├── README.md
├── tsconfig.json
└── vitest.config.ts
```

## Monorepo Scripts (Turborepo)

```json
{
  "scripts": {
    "build": "turbo build",
    "dev": "turbo dev",
    "lint": "turbo lint",
    "test": "turbo test",
    "typecheck": "turbo typecheck",
    "clean": "turbo clean && rm -rf node_modules"
  }
}
```

### turbo.json
```json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": ["**/.env.*local"],
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": [".next/**", "!.next/cache/**", "dist/**"]
    },
    "lint": {},
    "dev": {
      "cache": false,
      "persistent": true
    }
  }
}
```

## Best Practices

1. **Lock Node version** - Use `.nvmrc` or `engines` field
2. **Use strict TypeScript** - Enable all strict checks
3. **Enforce code style** - Husky + lint-staged + Prettier
4. **Conventional commits** - For automated changelogs
5. **ESM by default** - Set `"type": "module"`
6. **Exact versions** - Use `save-exact=true` in .npmrc
7. **Define exports** - For proper subpath imports
8. **Document scripts** - Use clear, consistent naming
