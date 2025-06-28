# GitHub デプロイ設定ガイド

## デプロイ戦略

### 推奨構成: GitHub + Vercel
1. **GitHub Repository**: ソースコード管理
2. **Vercel**: 自動デプロイ・ホスティング
3. **GitHub Actions**: テスト自動化

---

## 1. GitHub Repository 設定

### Repository 構造
```
1year-flexible-calendar/
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── deploy.yml
├── docs/
│   ├── 要件定義書.md
│   ├── 使い方マニュアル.txt
│   ├── アプリ構成図.md
│   └── UI_UX設計書.md
├── public/
├── src/
├── package.json
├── vite.config.ts
├── tailwind.config.js
├── tsconfig.json
├── .env.example
├── .gitignore
├── README.md
└── vercel.json
```

### .gitignore
```gitignore
# Dependencies
node_modules/
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# Production builds
dist/
build/

# Environment variables
.env
.env.local
.env.development.local
.env.test.local
.env.production.local

# IDE
.vscode/
.idea/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db

# Logs
logs
*.log

# Runtime data
pids
*.pid
*.seed
*.pid.lock

# Coverage directory used by tools like istanbul
coverage/

# nyc test coverage
.nyc_output

# Dependency directories
jspm_packages/

# Optional npm cache directory
.npm

# Optional REPL history
.node_repl_history

# Output of 'npm pack'
*.tgz

# Yarn Integrity file
.yarn-integrity

# parcel-bundler cache (https://parceljs.org/)
.cache
.parcel-cache

# next.js build output
.next

# nuxt.js build output
.nuxt

# vuepress build output
.vuepress/dist

# Serverless directories
.serverless

# FuseBox cache
.fusebox/

# DynamoDB Local files
.dynamodb/

# TernJS port file
.tern-port
```

---

## 2. Vercel デプロイ設定

### vercel.json
```json
{
  "version": 2,
  "name": "1year-flexible-calendar",
  "builds": [
    {
      "src": "package.json",
      "use": "@vercel/static-build",
      "config": {
        "distDir": "dist"
      }
    }
  ],
  "routes": [
    {
      "src": "/(.*)",
      "dest": "/index.html"
    }
  ],
  "env": {
    "VITE_APP_NAME": "1年単位変形労働時間制カレンダー",
    "VITE_HOLIDAY_API_URL": "https://holidays-jp.github.io/api/v1/"
  },
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        {
          "key": "X-Content-Type-Options",
          "value": "nosniff"
        },
        {
          "key": "X-Frame-Options",
          "value": "DENY"
        },
        {
          "key": "X-XSS-Protection",
          "value": "1; mode=block"
        }
      ]
    }
  ]
}
```

### package.json (build設定)
```json
{
  "name": "1year-flexible-calendar",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview",
    "lint": "eslint . --ext ts,tsx --report-unused-disable-directives --max-warnings 0",
    "lint:fix": "eslint . --ext ts,tsx --fix",
    "test": "vitest",
    "test:coverage": "vitest --coverage",
    "type-check": "tsc --noEmit"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-router-dom": "^6.8.0",
    "date-fns": "^2.29.3",
    "zustand": "^4.3.6",
    "react-hook-form": "^7.43.1",
    "zod": "^3.20.6",
    "@hookform/resolvers": "^2.9.11",
    "clsx": "^1.2.1",
    "tailwind-merge": "^1.10.0",
    "framer-motion": "^10.0.0",
    "jspdf": "^2.5.1",
    "xlsx": "^0.18.5",
    "lucide-react": "^0.321.0"
  },
  "devDependencies": {
    "@types/react": "^18.0.28",
    "@types/react-dom": "^18.0.11",
    "@typescript-eslint/eslint-plugin": "^5.54.0",
    "@typescript-eslint/parser": "^5.54.0",
    "@vitejs/plugin-react-swc": "^3.0.0",
    "autoprefixer": "^10.4.13",
    "eslint": "^8.35.0",
    "eslint-plugin-react-hooks": "^4.6.0",
    "eslint-plugin-react-refresh": "^0.3.4",
    "postcss": "^8.4.21",
    "tailwindcss": "^3.2.7",
    "typescript": "^4.9.3",
    "vite": "^4.1.0",
    "vitest": "^0.28.5",
    "@testing-library/react": "^14.0.0",
    "@testing-library/jest-dom": "^5.16.5",
    "jsdom": "^21.1.0"
  }
}
```

---

## 3. GitHub Actions 設定

### .github/workflows/ci.yml
```yaml
name: CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        node-version: [18.x, 20.x]

    steps:
    - name: Checkout
      uses: actions/checkout@v4

    - name: Use Node.js ${{ matrix.node-version }}
      uses: actions/setup-node@v4
      with:
        node-version: ${{ matrix.node-version }}
        cache: 'npm'

    - name: Install dependencies
      run: npm ci

    - name: Type check
      run: npm run type-check

    - name: Lint
      run: npm run lint

    - name: Test
      run: npm run test:coverage

    - name: Build
      run: npm run build

    - name: Upload coverage reports
      uses: codecov/codecov-action@v3
      if: matrix.node-version == '20.x'
      with:
        file: ./coverage/lcov.info
        flags: unittests
        name: codecov-umbrella
```

### .github/workflows/deploy.yml
```yaml
name: Deploy to Vercel

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout
      uses: actions/checkout@v4

    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '20.x'
        cache: 'npm'

    - name: Install dependencies
      run: npm ci

    - name: Build
      run: npm run build
      env:
        VITE_APP_NAME: ${{ vars.VITE_APP_NAME }}
        VITE_HOLIDAY_API_URL: ${{ vars.VITE_HOLIDAY_API_URL }}

    - name: Deploy to Vercel
      uses: vercel/action@v1
      with:
        vercel-token: ${{ secrets.VERCEL_TOKEN }}
        vercel-org-id: ${{ secrets.ORG_ID }}
        vercel-project-id: ${{ secrets.PROJECT_ID }}
        vercel-args: '--prod'
```

---

## 4. プロジェクト設定ファイル

### vite.config.ts
```typescript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react-swc'
import path from 'path'

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
      '@/components': path.resolve(__dirname, './src/components'),
      '@/hooks': path.resolve(__dirname, './src/hooks'),
      '@/utils': path.resolve(__dirname, './src/utils'),
      '@/types': path.resolve(__dirname, './src/types'),
      '@/store': path.resolve(__dirname, './src/store'),
    },
  },
  build: {
    outDir: 'dist',
    sourcemap: true,
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom'],
          router: ['react-router-dom'],
          date: ['date-fns'],
          forms: ['react-hook-form', 'zod', '@hookform/resolvers'],
          ui: ['framer-motion', 'lucide-react'],
          export: ['jspdf', 'xlsx'],
        },
      },
    },
  },
  server: {
    port: 3000,
    open: true,
  },
  preview: {
    port: 3000,
  },
})
```

### tailwind.config.js
```javascript
/** @type {import('tailwindcss').Config} */
export default {
  content: [
    "./index.html",
    "./src/**/*.{js,ts,jsx,tsx}",
  ],
  theme: {
    extend: {
      colors: {
        primary: {
          50: '#eff6ff',
          100: '#dbeafe',
          200: '#bfdbfe',
          300: '#93c5fd',
          400: '#60a5fa',
          500: '#3b82f6',
          600: '#2563eb',
          700: '#1d4ed8',
          800: '#1e40af',
          900: '#1e3a8a',
        },
        calendar: {
          work: '#ffffff',
          rest: '#e0f2fe',
          holiday: '#fce4ec',
          adjusted: '#fff3e0',
          overLimit: '#ffebee',
        },
      },
      fontFamily: {
        sans: ['Inter', 'Hiragino Kaku Gothic ProN', 'Hiragino Sans', 'Meiryo', 'sans-serif'],
      },
      spacing: {
        '18': '4.5rem',
        '88': '22rem',
        '128': '32rem',
      },
    },
  },
  plugins: [],
}
```

### tsconfig.json
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,

    /* Bundler mode */
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx",

    /* Linting */
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,

    /* Path mapping */
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"],
      "@/components/*": ["./src/components/*"],
      "@/hooks/*": ["./src/hooks/*"],
      "@/utils/*": ["./src/utils/*"],
      "@/types/*": ["./src/types/*"],
      "@/store/*": ["./src/store/*"]
    }
  },
  "include": ["src"],
  "references": [{ "path": "./tsconfig.node.json" }]
}
```

---

## 5. デプロイ手順

### 初回セットアップ
1. **GitHub Repository 作成**
   ```bash
   # ローカルで初期化
   git init
   git add .
   git commit -m "Initial commit"
   git branch -M main
   git remote add origin https://github.com/USERNAME/1year-flexible-calendar.git
   git push -u origin main
   ```

2. **Vercel アカウント設定**
   - [vercel.com](https://vercel.com) でアカウント作成
   - GitHub アカウントと連携
   - Repository を選択してインポート

3. **自動デプロイ設定**
   - Vercel が自動的に GitHub と連携
   - `main` ブランチへの push で自動デプロイ
   - PR作成時にプレビューデプロイ

### 環境変数設定
Vercel ダッシュボードで設定:
```
VITE_APP_NAME=1年単位変形労働時間制カレンダー
VITE_HOLIDAY_API_URL=https://holidays-jp.github.io/api/v1/
```

---

## 6. ブランチ戦略

### Git Flow 簡易版
```
main (本番)
├── develop (開発)
│   ├── feature/basic-settings
│   ├── feature/calendar-generation
│   ├── feature/legal-check
│   └── feature/export-functions
└── hotfix/* (緊急修正)
```

### ブランチルール
- `main`: 本番環境 (Vercel 自動デプロイ)
- `develop`: 開発環境
- `feature/*`: 機能開発
- `hotfix/*`: 緊急修正

---

## 7. カスタムドメイン設定 (オプション)

### 独自ドメイン設定
1. ドメイン取得 (例: calendar.example.com)
2. Vercel ダッシュボードで追加
3. DNS設定 (CNAME record)
4. SSL自動設定

---

## 8. モニタリング設定

### Vercel Analytics
```typescript
// src/main.tsx
import { Analytics } from '@vercel/analytics/react';

function App() {
  return (
    <>
      <YourApp />
      <Analytics />
    </>
  );
}
```

### Error Tracking (オプション)
```typescript
// Sentry integration
import * as Sentry from "@sentry/react";

Sentry.init({
  dsn: "YOUR_DSN",
  environment: import.meta.env.MODE,
});
```

---

## 9. パフォーマンス最適化

### Build最適化設定
```typescript
// vite.config.ts に追加
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom'],
          calendar: ['date-fns'],
          export: ['jspdf', 'xlsx'],
        },
      },
    },
    minify: 'terser',
    terserOptions: {
      compress: {
        drop_console: true,
        drop_debugger: true,
      },
    },
  },
});
```

---

**準備完了！**
これでGitHub + Vercelでの自動デプロイ環境が構築できます。

次のステップ：
1. GitHub Repository 作成
2. Vercel アカウント設定
3. 実装開始
4. 自動デプロイ確認