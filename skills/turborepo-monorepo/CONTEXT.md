# Turborepo Monorepo Context

Comprehensive guide to Turborepo configuration and monorepo patterns.

## Quick Start

```bash
# Create new monorepo
npx create-turbo@latest

# Or add to existing repo
npm install turbo -D
```

## Project Structure

```
my-monorepo/
├── apps/
│   ├── web/                    # Next.js app
│   │   ├── package.json
│   │   └── next.config.js
│   ├── api/                    # API server
│   │   └── package.json
│   └── docs/                   # Documentation
│       └── package.json
├── packages/
│   ├── ui/                     # Shared UI components
│   │   ├── package.json
│   │   └── src/
│   ├── config/                 # Shared configs
│   │   ├── eslint/
│   │   ├── typescript/
│   │   └── tailwind/
│   └── utils/                  # Shared utilities
│       └── package.json
├── package.json                # Root package.json
├── pnpm-workspace.yaml
├── turbo.json
└── .npmrc
```

## Root Configuration

### package.json
```json
{
  "name": "my-monorepo",
  "private": true,
  "workspaces": ["apps/*", "packages/*"],
  "scripts": {
    "build": "turbo build",
    "dev": "turbo dev",
    "lint": "turbo lint",
    "test": "turbo test",
    "typecheck": "turbo typecheck",
    "clean": "turbo clean && rm -rf node_modules",
    "format": "prettier --write \"**/*.{ts,tsx,md}\""
  },
  "devDependencies": {
    "prettier": "^3.2.0",
    "turbo": "^2.0.0"
  },
  "packageManager": "pnpm@9.0.0",
  "engines": {
    "node": ">=20.0.0"
  }
}
```

### pnpm-workspace.yaml
```yaml
packages:
  - "apps/*"
  - "packages/*"
```

### turbo.json
```json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": ["**/.env.*local"],
  "globalEnv": ["NODE_ENV", "CI"],
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": [
        ".next/**",
        "!.next/cache/**",
        "dist/**"
      ]
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "lint": {
      "dependsOn": ["^build"]
    },
    "typecheck": {
      "dependsOn": ["^build"]
    },
    "test": {
      "dependsOn": ["^build"]
    },
    "clean": {
      "cache": false
    }
  }
}
```

### .npmrc
```ini
# Hoist for compatibility
shamefully-hoist=true

# Use exact versions
save-exact=true

# Strict peer deps
strict-peer-dependencies=false
```

## Package Configuration

### Shared UI Package
```json
// packages/ui/package.json
{
  "name": "@repo/ui",
  "version": "0.0.0",
  "private": true,
  "main": "./src/index.ts",
  "types": "./src/index.ts",
  "exports": {
    ".": "./src/index.ts",
    "./button": "./src/button.tsx",
    "./card": "./src/card.tsx"
  },
  "scripts": {
    "lint": "eslint . --ext .ts,.tsx",
    "typecheck": "tsc --noEmit"
  },
  "peerDependencies": {
    "react": "^18.0.0",
    "react-dom": "^18.0.0"
  },
  "devDependencies": {
    "@repo/eslint-config": "workspace:*",
    "@repo/typescript-config": "workspace:*",
    "typescript": "^5.4.0"
  }
}
```

### App Package
```json
// apps/web/package.json
{
  "name": "@repo/web",
  "version": "0.0.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "@repo/ui": "workspace:*",
    "@repo/utils": "workspace:*",
    "next": "^14.2.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
  },
  "devDependencies": {
    "@repo/eslint-config": "workspace:*",
    "@repo/typescript-config": "workspace:*",
    "typescript": "^5.4.0"
  }
}
```

## Shared Configurations

### TypeScript Config
```json
// packages/config/typescript/base.json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "compilerOptions": {
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "declaration": true,
    "declarationMap": true
  }
}

// packages/config/typescript/nextjs.json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "extends": "./base.json",
  "compilerOptions": {
    "lib": ["dom", "dom.iterable", "esnext"],
    "jsx": "preserve",
    "module": "esnext",
    "plugins": [{ "name": "next" }]
  }
}

// packages/config/typescript/package.json
{
  "name": "@repo/typescript-config",
  "version": "0.0.0",
  "private": true,
  "files": ["*.json"]
}
```

### ESLint Config
```javascript
// packages/config/eslint/next.js
module.exports = {
  extends: [
    'next/core-web-vitals',
    'turbo',
    'prettier',
  ],
  rules: {
    '@next/next/no-html-link-for-pages': 'off',
  },
}

// packages/config/eslint/package.json
{
  "name": "@repo/eslint-config",
  "version": "0.0.0",
  "private": true,
  "main": "index.js",
  "files": ["*.js"],
  "dependencies": {
    "eslint-config-next": "^14.2.0",
    "eslint-config-prettier": "^9.1.0",
    "eslint-config-turbo": "^2.0.0"
  }
}
```

### Tailwind Config
```typescript
// packages/config/tailwind/tailwind.config.ts
import type { Config } from 'tailwindcss'

const config: Omit<Config, 'content'> = {
  theme: {
    extend: {
      // Shared theme extensions
    },
  },
  plugins: [],
}

export default config

// packages/config/tailwind/package.json
{
  "name": "@repo/tailwind-config",
  "version": "0.0.0",
  "private": true,
  "main": "tailwind.config.ts",
  "dependencies": {
    "tailwindcss": "^3.4.0"
  }
}
```

## Internal Packages

### Transitive Dependencies
```json
// packages/ui/package.json
{
  "name": "@repo/ui",
  "dependencies": {
    "clsx": "^2.0.0"
  },
  "peerDependencies": {
    "react": "^18.0.0"
  }
}
```

Apps get `clsx` transitively through `@repo/ui`.

### TypeScript Project References
```json
// apps/web/tsconfig.json
{
  "extends": "@repo/typescript-config/nextjs.json",
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

## Task Configuration

### Environment Variables
```json
// turbo.json
{
  "tasks": {
    "build": {
      "env": ["DATABASE_URL", "API_KEY"],
      "passThroughEnv": ["CI", "VERCEL_*"]
    }
  }
}
```

### Filtering Tasks
```bash
# Run only in specific packages
turbo build --filter=@repo/web
turbo build --filter=@repo/web...  # Include dependencies
turbo build --filter=...@repo/web  # Include dependents

# Exclude packages
turbo build --filter=!@repo/docs

# By directory
turbo build --filter=./apps/*
turbo build --filter=./packages/ui

# Changed since
turbo build --filter=[origin/main]
```

### Remote Caching
```bash
# Login to Vercel
npx turbo login

# Link to Vercel project
npx turbo link

# Or self-hosted
# turbo.json
{
  "remoteCache": {
    "signature": true
  }
}

# Environment variables
TURBO_TEAM=your-team
TURBO_TOKEN=your-token
TURBO_API=https://your-cache-server.com
```

## CI/CD

### GitHub Actions
```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 2

      - uses: pnpm/action-setup@v3
        with:
          version: 9

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'

      - run: pnpm install

      - run: pnpm turbo build lint typecheck test
        env:
          TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}
          TURBO_TEAM: ${{ vars.TURBO_TEAM }}
```

### Vercel Deployment
```json
// apps/web/vercel.json
{
  "framework": "nextjs",
  "installCommand": "pnpm install",
  "buildCommand": "cd ../.. && pnpm turbo build --filter=@repo/web..."
}
```

## Best Practices

1. **Use workspace protocol** - `"workspace:*"` for internal deps
2. **Share configs** - ESLint, TypeScript, Tailwind in packages/config
3. **Keep packages focused** - Single responsibility
4. **Use internal packages** - Don't publish what's private
5. **Cache aggressively** - Enable remote caching
6. **Filter in CI** - Only build what changed
7. **Document dependencies** - Clear package.json relationships
8. **Version together** - Use same versions for shared deps
