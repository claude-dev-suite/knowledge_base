# Vite Build Optimization Guide

## Code Splitting Strategies

### Manual Chunks

```typescript
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks(id) {
          // Vendor chunk
          if (id.includes('node_modules')) {
            if (id.includes('react') || id.includes('react-dom')) {
              return 'react-vendor';
            }
            if (id.includes('@tanstack')) {
              return 'tanstack-vendor';
            }
            return 'vendor';
          }

          // Feature-based chunks
          if (id.includes('/src/features/admin')) {
            return 'admin';
          }
          if (id.includes('/src/features/dashboard')) {
            return 'dashboard';
          }
        }
      }
    }
  }
});
```

### Dynamic Imports

```typescript
// Lazy load routes
const AdminPanel = lazy(() => import('./pages/AdminPanel'));
const Dashboard = lazy(() => import('./pages/Dashboard'));

// Preload on hover
const preloadAdmin = () => import('./pages/AdminPanel');

<Link to="/admin" onMouseEnter={preloadAdmin}>
  Admin
</Link>
```

## Dependency Pre-bundling

### Include/Exclude Dependencies

```typescript
export default defineConfig({
  optimizeDeps: {
    include: ['react', 'react-dom', '@tanstack/react-query'],
    exclude: ['@vite/client', '@vite/env']
  }
});
```

### Force Re-optimization

```bash
# Clear cache and re-optimize
rm -rf node_modules/.vite
npm run dev
```

## Build Performance

### Parallel Processing

```typescript
export default defineConfig({
  build: {
    // Use esbuild for faster minification
    minify: 'esbuild',

    // Parallel workers
    rollupOptions: {
      maxParallelFileOps: 4
    }
  }
});
```

### Terser Configuration

```typescript
export default defineConfig({
  build: {
    minify: 'terser',
    terserOptions: {
      compress: {
        drop_console: true,
        drop_debugger: true,
        pure_funcs: ['console.log', 'console.debug']
      },
      format: {
        comments: false
      }
    }
  }
});
```

## Asset Optimization

### Image Optimization

```typescript
import { ViteImageOptimizer } from 'vite-plugin-image-optimizer';

export default defineConfig({
  plugins: [
    ViteImageOptimizer({
      png: { quality: 80 },
      jpeg: { quality: 80 },
      webp: { quality: 80 }
    })
  ]
});
```

### Asset Inlining

```typescript
export default defineConfig({
  build: {
    assetsInlineLimit: 4096, // 4kb (default)
    // Assets < 4kb will be base64 inlined
  }
});
```

## CSS Optimization

### Code Splitting

```typescript
export default defineConfig({
  build: {
    cssCodeSplit: true, // Split CSS by entry point
    cssMinify: 'lightningcss'
  }
});
```

### PostCSS Optimization

```typescript
// postcss.config.js
export default {
  plugins: {
    'tailwindcss': {},
    'autoprefixer': {},
    'cssnano': {
      preset: ['default', {
        discardComments: { removeAll: true }
      }]
    }
  }
};
```

## Tree Shaking

### Ensure Proper Imports

```typescript
// ✅ Good - tree-shakeable
import { useState, useEffect } from 'react';

// ❌ Bad - imports everything
import * as React from 'react';

// ✅ Good - named imports
import { Button } from '@/components/ui/button';

// ❌ Bad - namespace import
import * as UI from '@/components/ui';
```

### Sideeffects Configuration

```json
// package.json
{
  "sideEffects": [
    "*.css",
    "*.scss",
    "src/polyfills.ts"
  ]
}
```

## Bundle Analysis

### Rollup Plugin Visualizer

```typescript
import { visualizer } from 'rollup-plugin-visualizer';

export default defineConfig({
  plugins: [
    visualizer({
      filename: './dist/stats.html',
      open: true,
      gzipSize: true,
      brotliSize: true
    })
  ]
});
```

### Size Limits

```typescript
export default defineConfig({
  build: {
    chunkSizeWarningLimit: 500, // KB
    rollupOptions: {
      output: {
        // Warn on large chunks
        manualChunks(id) {
          if (id.includes('node_modules')) {
            const match = id.match(/node_modules\/(@?[^/]+)/);
            if (match) {
              const packageName = match[1];
              // Split large packages
              if (['lodash', 'moment'].includes(packageName)) {
                return `vendor-${packageName}`;
              }
            }
          }
        }
      }
    }
  }
});
```

## Caching Strategies

### Long-term Caching

```typescript
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        entryFileNames: 'assets/[name].[hash].js',
        chunkFileNames: 'assets/[name].[hash].js',
        assetFileNames: 'assets/[name].[hash].[ext]'
      }
    }
  }
});
```

### HTTP/2 Considerations

```typescript
// For HTTP/2, avoid too many small chunks
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          // Combine related modules
          'ui-components': [
            './src/components/Button',
            './src/components/Input',
            './src/components/Select'
          ]
        }
      }
    }
  }
});
```

## Development Performance

### HMR Optimization

```typescript
export default defineConfig({
  server: {
    hmr: {
      overlay: true,
      // Use WebSocket for HMR
      protocol: 'ws'
    },
    watch: {
      // Ignore files for better performance
      ignored: ['**/node_modules/**', '**/.git/**']
    }
  }
});
```

### Faster Reload

```typescript
export default defineConfig({
  server: {
    // Warm up frequently used files
    warmup: {
      clientFiles: [
        './src/main.tsx',
        './src/App.tsx',
        './src/routes/**/*.tsx'
      ]
    }
  }
});
```

## Production Checklist

- [ ] Enable minification
- [ ] Configure code splitting
- [ ] Remove console.log
- [ ] Enable source maps for debugging
- [ ] Optimize images
- [ ] Enable gzip/brotli compression
- [ ] Analyze bundle size
- [ ] Test production build locally
- [ ] Configure CDN for assets
- [ ] Set proper cache headers

## Performance Metrics

### Target Metrics

| Metric | Target |
|--------|--------|
| First Contentful Paint | < 1.5s |
| Time to Interactive | < 3.5s |
| Total Bundle Size | < 250KB (gzipped) |
| Largest Chunk | < 150KB |
| Number of Chunks | 5-10 |

### Monitoring

```typescript
// Add bundle size check to CI
{
  "scripts": {
    "build": "vite build",
    "size": "npm run build && size-limit"
  }
}

// .size-limit.json
[
  {
    "path": "dist/assets/*.js",
    "limit": "150 KB",
    "gzip": true
  }
]
```

## References

- [Vite Performance Guide](https://vitejs.dev/guide/performance.html)
- [Rollup Optimization](https://rollupjs.org/configuration-options/)
- [Web.dev Performance](https://web.dev/fast/)
