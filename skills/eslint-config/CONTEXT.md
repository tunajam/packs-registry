# ESLint Configuration Context

Comprehensive guide to ESLint flat config for TypeScript and React projects.

## Setup

```bash
npm install -D eslint @eslint/js typescript-eslint eslint-plugin-react eslint-plugin-react-hooks globals
```

## Flat Config (ESLint 9+)

### Basic TypeScript Config
```javascript
// eslint.config.mjs
import eslint from '@eslint/js'
import tseslint from 'typescript-eslint'
import globals from 'globals'

export default tseslint.config(
  eslint.configs.recommended,
  ...tseslint.configs.recommended,
  {
    languageOptions: {
      globals: {
        ...globals.node,
        ...globals.es2022,
      },
      parserOptions: {
        project: true,
        tsconfigRootDir: import.meta.dirname,
      },
    },
    rules: {
      '@typescript-eslint/no-unused-vars': ['error', { 
        argsIgnorePattern: '^_',
        varsIgnorePattern: '^_',
      }],
      '@typescript-eslint/consistent-type-imports': ['error', {
        prefer: 'type-imports',
        fixStyle: 'inline-type-imports',
      }],
      '@typescript-eslint/no-import-type-side-effects': 'error',
    },
  },
  {
    ignores: ['dist/', 'node_modules/', '*.config.js'],
  }
)
```

### React + TypeScript Config
```javascript
// eslint.config.mjs
import eslint from '@eslint/js'
import tseslint from 'typescript-eslint'
import react from 'eslint-plugin-react'
import reactHooks from 'eslint-plugin-react-hooks'
import reactRefresh from 'eslint-plugin-react-refresh'
import globals from 'globals'

export default tseslint.config(
  eslint.configs.recommended,
  ...tseslint.configs.recommendedTypeChecked,
  {
    languageOptions: {
      globals: {
        ...globals.browser,
        ...globals.es2022,
      },
      parserOptions: {
        project: true,
        tsconfigRootDir: import.meta.dirname,
        ecmaFeatures: {
          jsx: true,
        },
      },
    },
    settings: {
      react: {
        version: 'detect',
      },
    },
    plugins: {
      react,
      'react-hooks': reactHooks,
      'react-refresh': reactRefresh,
    },
    rules: {
      // React
      ...react.configs.recommended.rules,
      ...react.configs['jsx-runtime'].rules,
      ...reactHooks.configs.recommended.rules,
      'react-refresh/only-export-components': ['warn', {
        allowConstantExport: true,
      }],
      'react/prop-types': 'off',
      'react/react-in-jsx-scope': 'off',
      
      // TypeScript
      '@typescript-eslint/no-unused-vars': ['error', { 
        argsIgnorePattern: '^_',
        varsIgnorePattern: '^_',
      }],
      '@typescript-eslint/consistent-type-imports': ['error', {
        prefer: 'type-imports',
        fixStyle: 'inline-type-imports',
      }],
      '@typescript-eslint/no-misused-promises': ['error', {
        checksVoidReturn: { attributes: false },
      }],
      
      // General
      'no-console': ['warn', { allow: ['warn', 'error'] }],
    },
  },
  {
    ignores: ['dist/', 'node_modules/', '*.config.*', 'coverage/'],
  }
)
```

### Next.js Config
```javascript
// eslint.config.mjs
import eslint from '@eslint/js'
import tseslint from 'typescript-eslint'
import next from '@next/eslint-plugin-next'
import react from 'eslint-plugin-react'
import reactHooks from 'eslint-plugin-react-hooks'
import globals from 'globals'

export default tseslint.config(
  eslint.configs.recommended,
  ...tseslint.configs.recommended,
  {
    languageOptions: {
      globals: {
        ...globals.browser,
        ...globals.node,
        React: 'readonly',
      },
    },
    settings: {
      react: {
        version: 'detect',
      },
    },
    plugins: {
      react,
      'react-hooks': reactHooks,
      '@next/next': next,
    },
    rules: {
      ...react.configs.recommended.rules,
      ...react.configs['jsx-runtime'].rules,
      ...reactHooks.configs.recommended.rules,
      ...next.configs.recommended.rules,
      ...next.configs['core-web-vitals'].rules,
      
      'react/prop-types': 'off',
      '@typescript-eslint/no-unused-vars': ['error', {
        argsIgnorePattern: '^_',
        varsIgnorePattern: '^_',
      }],
    },
  },
  {
    ignores: ['.next/', 'node_modules/', 'out/'],
  }
)
```

## Custom Rules

### Import Organization
```javascript
import importPlugin from 'eslint-plugin-import'

// In your config
{
  plugins: {
    import: importPlugin,
  },
  rules: {
    'import/order': ['error', {
      groups: [
        'builtin',
        'external',
        'internal',
        ['parent', 'sibling'],
        'index',
        'type',
      ],
      'newlines-between': 'always',
      alphabetize: {
        order: 'asc',
        caseInsensitive: true,
      },
    }],
    'import/no-duplicates': 'error',
    'import/no-cycle': 'error',
  },
}
```

### Accessibility
```javascript
import jsxA11y from 'eslint-plugin-jsx-a11y'

{
  plugins: {
    'jsx-a11y': jsxA11y,
  },
  rules: {
    ...jsxA11y.configs.recommended.rules,
    'jsx-a11y/anchor-is-valid': ['error', {
      components: ['Link'],
      specialLink: ['hrefLeft', 'hrefRight'],
      aspects: ['invalidHref', 'preferButton'],
    }],
  },
}
```

### Tailwind CSS
```javascript
import tailwind from 'eslint-plugin-tailwindcss'

{
  plugins: {
    tailwindcss: tailwind,
  },
  rules: {
    ...tailwind.configs.recommended.rules,
    'tailwindcss/classnames-order': 'warn',
    'tailwindcss/no-custom-classname': 'warn',
    'tailwindcss/no-contradicting-classname': 'error',
  },
  settings: {
    tailwindcss: {
      callees: ['cn', 'clsx', 'cva'],
      config: 'tailwind.config.js',
    },
  },
}
```

## Common Rule Configurations

### Strict TypeScript Rules
```javascript
{
  rules: {
    // Require explicit return types
    '@typescript-eslint/explicit-function-return-type': ['error', {
      allowExpressions: true,
      allowTypedFunctionExpressions: true,
    }],
    
    // Prefer nullish coalescing
    '@typescript-eslint/prefer-nullish-coalescing': 'error',
    
    // Prefer optional chaining
    '@typescript-eslint/prefer-optional-chain': 'error',
    
    // No floating promises
    '@typescript-eslint/no-floating-promises': 'error',
    
    // Require await in async functions
    '@typescript-eslint/require-await': 'error',
    
    // Strict boolean expressions
    '@typescript-eslint/strict-boolean-expressions': 'error',
    
    // Exhaustive switch
    '@typescript-eslint/switch-exhaustiveness-check': 'error',
  },
}
```

### React Best Practices
```javascript
{
  rules: {
    // No array index as key
    'react/no-array-index-key': 'warn',
    
    // Self-closing components
    'react/self-closing-comp': 'error',
    
    // Consistent component definition
    'react/function-component-definition': ['error', {
      namedComponents: 'arrow-function',
      unnamedComponents: 'arrow-function',
    }],
    
    // No unstable components in JSX
    'react/no-unstable-nested-components': 'error',
    
    // Hook dependencies
    'react-hooks/exhaustive-deps': 'warn',
    
    // Rules of hooks
    'react-hooks/rules-of-hooks': 'error',
  },
}
```

## VSCode Integration

```json
// .vscode/settings.json
{
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit"
  },
  "eslint.validate": [
    "javascript",
    "javascriptreact",
    "typescript",
    "typescriptreact"
  ],
  "eslint.useFlatConfig": true
}
```

## CLI Usage

```bash
# Lint files
npx eslint .

# Lint specific patterns
npx eslint "src/**/*.{ts,tsx}"

# Fix auto-fixable issues
npx eslint . --fix

# Show only errors
npx eslint . --quiet

# Output as JSON
npx eslint . --format json

# Check specific rule
npx eslint . --rule '@typescript-eslint/no-unused-vars: error'

# Debug config
npx eslint --print-config src/index.ts
```

## Package.json Scripts

```json
{
  "scripts": {
    "lint": "eslint .",
    "lint:fix": "eslint . --fix",
    "lint:strict": "eslint . --max-warnings 0"
  }
}
```

## Migrating from Legacy Config

```bash
# Install migration tool
npm install -D @eslint/migrate-config

# Run migration
npx @eslint/migrate-config .eslintrc.json
```

## Best Practices

1. **Use flat config** - Modern, composable, type-safe
2. **Enable type-aware rules** - More powerful with TypeScript
3. **Configure ignores** - Don't lint generated code
4. **Use presets** - Start with recommended configs
5. **Fix on save** - Configure editor integration
6. **CI enforcement** - Fail builds on lint errors
7. **Share configs** - Create reusable config packages
8. **Document exceptions** - Comment `eslint-disable` usage
