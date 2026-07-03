# Private Landing + Deploy Pipeline Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Приватный репо `metastack-dev/landing` (Next.js static export, плейсхолдер) с GitHub Actions деплоем собранного `out/` в публичный `metastack-dev/metastack-dev.github.io`.

**Architecture:** Исходники живут в приватном репо; workflow на push в `main` собирает статический экспорт и force-пушит артефакты в `main` публичного репо через `peaceiris/actions-gh-pages` с deploy key. Публично виден только билд.

**Tech Stack:** Next.js (App Router, TypeScript, `output: 'export'`), GitHub Actions, `peaceiris/actions-gh-pages@v4`, deploy key (ed25519).

## Global Constraints

- Приватный репо: `metastack-dev/landing`; публичный: `metastack-dev/metastack-dev.github.io`.
- Никаких серверных фич Next.js (API routes, SSR); `images: { unoptimized: true }`.
- Деплой только билд-артефактов — исходники не должны попадать в публичный репо.
- Секрет с приватным ключом называется `ACTIONS_DEPLOY_KEY`.
- Локальная папка для приватного репо: `/Users/loonuh/Developer/MetaStack/landing`.

---

### Task 1: Создать приватный репо и скаффолдинг Next.js

**Files:**
- Create: `/Users/loonuh/Developer/MetaStack/landing/` (create-next-app)
- Modify: `/Users/loonuh/Developer/MetaStack/landing/next.config.ts`
- Modify: `/Users/loonuh/Developer/MetaStack/landing/app/page.tsx`
- Create: `/Users/loonuh/Developer/MetaStack/landing/docs/specs/2026-07-03-private-landing-deploy-design.md` (копия спеки из публичного репо)

**Interfaces:**
- Produces: репо `metastack-dev/landing` (private) с рабочим `npm run build`, выдающим `out/index.html`.

- [ ] **Step 1: Скаффолдинг Next.js**

```bash
cd /Users/loonuh/Developer/MetaStack
npx create-next-app@latest landing --typescript --app --eslint --no-tailwind --no-src-dir --import-alias "@/*" --use-npm --no-turbopack
```

- [ ] **Step 2: Включить статический экспорт**

Заменить содержимое `next.config.ts`:

```ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  output: "export",
  images: { unoptimized: true },
};

export default nextConfig;
```

- [ ] **Step 3: Плейсхолдер-страница**

Заменить содержимое `app/page.tsx`:

```tsx
export default function Home() {
  return (
    <main style={{ display: "grid", placeItems: "center", minHeight: "100vh" }}>
      <h1>MetaStack — coming soon</h1>
    </main>
  );
}
```

- [ ] **Step 4: Скопировать спеку в приватный репо**

```bash
mkdir -p /Users/loonuh/Developer/MetaStack/landing/docs/specs
cp /Users/loonuh/Developer/MetaStack/metastack-dev.github.io/docs/superpowers/specs/2026-07-03-private-landing-deploy-design.md /Users/loonuh/Developer/MetaStack/landing/docs/specs/
```

- [ ] **Step 5: Проверить билд**

```bash
cd /Users/loonuh/Developer/MetaStack/landing && npm run build && test -f out/index.html && echo BUILD_OK
```

Expected: `BUILD_OK`.

- [ ] **Step 6: Создать приватный репо и запушить**

```bash
cd /Users/loonuh/Developer/MetaStack/landing
git add -A && git commit -m "feat: scaffold Next.js static-export landing with placeholder"
gh repo create metastack-dev/landing --private --source . --push
```

Expected: `gh repo view metastack-dev/landing --json visibility` → `PRIVATE`.

---

### Task 2: Deploy key

**Files:**
- Create (временно, удалить после): `<scratchpad>/deploy_key`, `<scratchpad>/deploy_key.pub`

**Interfaces:**
- Consumes: репо `metastack-dev/landing` из Task 1.
- Produces: секрет `ACTIONS_DEPLOY_KEY` в `landing`; write deploy key в `metastack-dev.github.io`.

- [ ] **Step 1: Сгенерировать ключ (в scratchpad, не в репо)**

```bash
ssh-keygen -t ed25519 -N "" -C "landing-deploy" -f <scratchpad>/deploy_key
```

- [ ] **Step 2: Публичный ключ → deploy key с записью в публичный репо**

```bash
gh repo deploy-key add <scratchpad>/deploy_key.pub -R metastack-dev/metastack-dev.github.io --allow-write --title "landing CI deploy"
```

- [ ] **Step 3: Приватный ключ → секрет в landing**

```bash
gh secret set ACTIONS_DEPLOY_KEY -R metastack-dev/landing < <scratchpad>/deploy_key
```

- [ ] **Step 4: Удалить локальные копии ключа**

```bash
rm <scratchpad>/deploy_key <scratchpad>/deploy_key.pub
```

- [ ] **Step 5: Проверка**

```bash
gh repo deploy-key list -R metastack-dev/metastack-dev.github.io
gh secret list -R metastack-dev/landing
```

Expected: ключ `landing CI deploy` (read/write) и секрет `ACTIONS_DEPLOY_KEY`.

---

### Task 3: Workflow деплоя и сквозная проверка

**Files:**
- Create: `/Users/loonuh/Developer/MetaStack/landing/.github/workflows/deploy.yml`

**Interfaces:**
- Consumes: секрет `ACTIONS_DEPLOY_KEY` из Task 2; билд из Task 1.
- Produces: живой сайт `https://metastack-dev.github.io`.

- [ ] **Step 1: Написать workflow**

`.github/workflows/deploy.yml`:

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npm run build
      - uses: peaceiris/actions-gh-pages@v4
        with:
          deploy_key: ${{ secrets.ACTIONS_DEPLOY_KEY }}
          external_repository: metastack-dev/metastack-dev.github.io
          publish_branch: main
          publish_dir: ./out
```

Примечание: `peaceiris/actions-gh-pages` по умолчанию добавляет `.nojekyll` — отдельный файл не нужен.

- [ ] **Step 2: Закоммитить и запушить**

```bash
cd /Users/loonuh/Developer/MetaStack/landing
git add .github && git commit -m "ci: deploy static export to metastack-dev.github.io"
git push
```

- [ ] **Step 3: Дождаться зелёного workflow**

```bash
gh run watch -R metastack-dev/landing --exit-status
```

Expected: exit 0. Если упал — `gh run view -R metastack-dev/landing --log-failed`.

- [ ] **Step 4: Проверить Pages и сайт**

Убедиться, что Pages публичного репо билдится из `main` (для user-site это дефолт):

```bash
gh api repos/metastack-dev/metastack-dev.github.io/pages
curl -sSf https://metastack-dev.github.io | grep -i "coming soon" && echo SITE_OK
```

Expected: `SITE_OK` (Pages может деплоиться пару минут после пуша — повторить curl при 404).

- [ ] **Step 5: Проверить, что исходники не утекли**

```bash
gh api repos/metastack-dev/metastack-dev.github.io/contents/ --jq '.[].name'
```

Expected: только билд-артефакты (`index.html`, `_next`, `.nojekyll`, `404.html` и т.п.), никаких `app/`, `package.json`, `next.config.ts`.
