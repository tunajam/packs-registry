# Vite Configuration Context

Comprehensive guide to Vite configuration, plugins, and performance optimization.

## Basic Configuration

```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import path from 'path'

export default defineConfig({
  plugins: [react()],
  
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
  
  server: {
    port: 3000,
    host: true, // Expose to network
    open: true, // Open browser on start
  },
  
  build: {
    outDir: 'dist',
    sourcemap: true,
  },
})
```

## Environment-Specific Config

```typescript
import { defineConfig, loadEnv } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig(({ command, mode }) => {
  // Load env file based on `mode`
  const env = loadEnv(mode, process.cwd(), '')
  
  const isDev = command === 'serve'
  const isProd = mode === 'production'
  
  return {
    plugins: [react()],
    
    define: {
      // Make env vars available at build time
      __APP_VERSION__: JSON.stringify(process.env.npm_package_version),
    },
    
    server: isDev ? {
      port: parseInt(env.PORT) || 3000,
      proxy: {
        '/api': {
          target: env.API_URL || 'http://localhost:8000',
          changeOrigin: true,
        },
      },
    } : undefined,
    
    build: isProd ? {
      minify: 'terser',
      terserOptions: {
        compress: {
          drop_console: true,
          drop_debugger: true,
        },
      },
    } : undefined,
  }
})
```

## Path Aliases

```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import tsconfigPaths from 'vite-tsconfig-paths'

export default defineConfig({
  plugins: [tsconfigPaths()],
  
  // Or manual aliases
  resolve: {
    alias: {
      '@': '/src',
      '@components': '/src/components',
      '@hooks': '/src/hooks',
      '@utils': '/src/utils',
      '@assets': '/src/assets',
    },
  },
})
```

```json
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@components/*": ["src/components/*"]
    }
  }
}
```

## Essential Plugins

### React
```typescript
import react from '@vitejs/plugin-react'
// Or SWC for faster builds
import react from '@vitejs/plugin-react-swc'

export default defineConfig({
  plugins: [
    react({
      // Babel plugins
      babel: {
        plugins: [
          ['babel-plugin-styled-components', { displayName: true }],
        ],
      },
    }),
  ],
})
```

### SVG as Components
```typescript
import svgr from 'vite-plugin-svgr'

export default defineConfig({
  plugins: [
    svgr({
      svgrOptions: {
        icon: true,
        dimensions: false,
      },
    }),
  ],
})

// Usage
import { ReactComponent as Logo } from './logo.svg'
// Or
import Logo from './logo.svg?react'
```

### Auto-Import
```typescript
import AutoImport from 'unplugin-auto-import/vite'

export default defineConfig({
  plugins: [
    AutoImport({
      imports: ['react', 'react-router-dom'],
      dts: './src/auto-imports.d.ts',
    }),
  ],
})
```

### Component Auto-Import
```typescript
import Components from 'unplugin-vue-components/vite'
// Or for React
import { TanStackRouterVite } from '@tanstack/router-vite-plugin'
```

### PWA Support
```typescript
import { VitePWA } from 'vite-plugin-pwa'

export default defineConfig({
  plugins: [
    VitePWA({
      registerType: 'autoUpdate',
      includeAssets: ['favicon.ico', 'robots.txt'],
      manifest: {
        name: 'My App',
        short_name: 'App',
        theme_color: '#ffffff',
        icons: [
          { src: '/icon-192.png', sizes: '192x192', type: 'image/png' },
          { src: '/icon-512.png', sizes: '512x512', type: 'image/png' },
        ],
      },
    }),
  ],
})
```

## Build Optimization

### Code Splitting
```typescript
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          // Vendor chunks
          'react-vendor': ['react', 'react-dom', 'react-router-dom'],
          'ui-vendor': ['@radix-ui/react-dialog', '@radix-ui/react-dropdown-menu'],
          
          // Or dynamic splitting
          manualChunks(id) {
            if (id.includes('node_modules')) {
              if (id.includes('@radix-ui')) return 'radix'
              if (id.includes('lodash')) return 'lodash'
              return 'vendor'
            }
          },
        },
      },
    },
  },
})
```

### Chunk Size Warnings
```typescript
export default defineConfig({
  build: {
    chunkSizeWarningLimit: 500, // KB
    
    rollupOptions: {
      output: {
        // Ensure consistent chunk names
        chunkFileNames: 'assets/[name]-[hash].js',
        entryFileNames: 'assets/[name]-[hash].js',
        assetFileNames: 'assets/[name]-[hash].[ext]',
      },
    },
  },
})
```

### Tree Shaking
```typescript
export default defineConfig({
  build: {
    // Enable rollup tree-shaking annotations
    rollupOptions: {
      treeshake: {
        moduleSideEffects: false, // Assume no side effects
        propertyReadSideEffects: false,
      },
    },
  },
})
```

### CSS Code Splitting
```typescript
export default defineConfig({
  build: {
    cssCodeSplit: true, // Default: true
    cssMinify: 'lightningcss', // Faster than default
  },
  
  css: {
    devSourcemap: true,
    modules: {
      localsConvention: 'camelCaseOnly',
      generateScopedName: '[name]__[local]___[hash:base64:5]',
    },
  },
})
```

## Development Server

### Proxy Configuration
```typescript
export default defineConfig({
  server: {
    proxy: {
      '/api': {
        target: 'http://localhost:8000',
        changeOrigin: true,
        rewrite: (path) => path.replace(/^\/api/, ''),
      },
      '/socket.io': {
        target: 'ws://localhost:8000',
        ws: true,
      },
    },
  },
})
```

### HTTPS
```typescript
import fs from 'fs'

export default defineConfig({
  server: {
    https: {
      key: fs.readFileSync('./certs/localhost-key.pem'),
      cert: fs.readFileSync('./certs/localhost.pem'),
    },
  },
})

// Or use mkcert plugin
import mkcert from 'vite-plugin-mkcert'

export default defineConfig({
  plugins: [mkcert()],
  server: { https: true },
})
```

### HMR Configuration
```typescript
export default defineConfig({
  server: {
    hmr: {
      overlay: true, // Show errors as overlay
      // For Docker/WSL environments
      host: 'localhost',
      port: 24678,
      clientPort: 24678,
    },
  },
})
```

## Testing Integration

### Vitest
```typescript
/// <reference types="vitest" />
import { defineConfig } from 'vite'

export default defineConfig({
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: './src/test/setup.ts',
    include: ['src/**/*.{test,spec}.{js,ts,jsx,tsx}'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
    },
  },
})
```

## Library Mode

```typescript
import { defineConfig } from 'vite'
import { resolve } from 'path'
import dts from 'vite-plugin-dts'

export default defineConfig({
  plugins: [dts({ rollupTypes: true })],
  
  build: {
    lib: {
      entry: resolve(__dirname, 'src/index.ts'),
      name: 'MyLib',
      fileName: (format) => `my-lib.${format}.js`,
      formats: ['es', 'cjs', 'umd'],
    },
    rollupOptions: {
      external: ['react', 'react-dom'],
      output: {
        globals: {
          react: 'React',
          'react-dom': 'ReactDOM',
        },
      },
    },
  },
})
```

## Performance Tips

1. **Use SWC over Babel** - 20x faster transforms
2. **Optimize deps** - Pre-bundle large dependencies
3. **Lazy imports** - Use `import()` for code splitting
4. **External large libs** - Don't bundle what CDN can serve
5. **Analyze bundle** - Use `rollup-plugin-visualizer`

```typescript
import { visualizer } from 'rollup-plugin-visualizer'

export default defineConfig({
  plugins: [
    visualizer({
      open: true,
      filename: 'stats.html',
    }),
  ],
})
```

## Common Issues

### Dependency Pre-Bundling
```typescript
export default defineConfig({
  optimizeDeps: {
    include: ['linked-dep', 'some-cjs-package'],
    exclude: ['your-local-package'],
    esbuildOptions: {
      target: 'esnext',
    },
  },
})
```

### CommonJS Compatibility
```typescript
import commonjs from '@rollup/plugin-commonjs'

export default defineConfig({
  build: {
    rollupOptions: {
      plugins: [commonjs()],
    },
  },
})
```
