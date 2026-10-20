# GitHub Actions — on-demand tour video

A `workflow_dispatch` workflow that boots the stack, runs the tour headless, and uploads **one** video plus the summary and screenshots as a downloadable artefact.

```yaml
name: E2E store tour (video)

on:
  workflow_dispatch:
    inputs:
      headed:
        description: Run with a visible browser (self-hosted only)
        type: boolean
        default: false

jobs:
  tour:
    runs-on: ubuntu-latest
    timeout-minutes: 60

    services:
      sqlserver:
        image: mcr.microsoft.com/mssql/server:2022-latest
        env:
          ACCEPT_EULA: "Y"
          MSSQL_SA_PASSWORD: ${{ secrets.E2E_SA_PASSWORD }}
        ports: ["1433:1433"]
        options: >-
          --health-cmd "/opt/mssql-tools18/bin/sqlcmd -C -S localhost -U sa -P ${{ secrets.E2E_SA_PASSWORD }} -Q 'SELECT 1'"
          --health-interval 10s --health-timeout 5s --health-retries 20

    env:
      E2E_ADMIN_EMAIL: admin@localhost
      E2E_ADMIN_PASSWORD: ${{ secrets.E2E_ADMIN_PASSWORD }}
      E2E_MANAGER_PASSWORD: ${{ secrets.E2E_MANAGER_PASSWORD }}
      E2E_USER_PASSWORD: ${{ secrets.E2E_USER_PASSWORD }}
      ConnectionStrings__DefaultConnection: >-
        Server=localhost,1433;Database=StoreDb;User Id=sa;Password=${{ secrets.E2E_SA_PASSWORD }};TrustServerCertificate=True
      E2E_BASE_URL: http://localhost:5174

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-dotnet@v4
        with: { dotnet-version: '10.0.x' }

      - uses: actions/setup-node@v4
        with: { node-version: '22', cache: npm, cache-dependency-path: web/client-side/package-lock.json }

      # 1. API (background) — seeds the DB on first run
      - name: Start API
        working-directory: web/StoreManagementSystem
        run: dotnet run --project StoreManagementSystem.API --urls http://localhost:60806 &
      - name: Wait for API
        run: |
          for i in $(seq 1 60); do
            curl -sf http://localhost:60806/health && break || sleep 5
          done

      # 2. Web (background)
      - name: Install web deps
        working-directory: web/client-side
        run: npm ci
      - name: Install Playwright browsers
        working-directory: web/client-side
        run: npx playwright install --with-deps chromium
      - name: Start web
        working-directory: web/client-side
        run: npm run dev -- --port 5174 --strictPort &
      - name: Wait for web
        run: |
          for i in $(seq 1 60); do
            curl -sf http://localhost:5174 && break || sleep 5
          done

      # 3. The tour
      - name: Run tour
        working-directory: web/client-side
        run: npx playwright test --project=tour

      # 4. Artefacts — always upload, even when a step failed
      - name: Upload tour video + report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: store-tour
          path: |
            web/test-results/store-tour.webm
            web/test-results/tour-summary.md
            web/test-results/downloads/**
            web/client-side/e2e/screenshots/**
          retention-days: 14
```

## Notes

- **Always upload with `if: always()`** — the video and summary are written before the final assertion, so they exist even when the run is red.
- Store passwords in **repository secrets**. The workflow only wires them into env vars; nothing is committed.
- Prefer starting the API and web as background steps and polling a health/URL endpoint over `docker-compose` sleeping.
- If the SPA's default port is taken on CI runners, pin a free one with `--strictPort` and set `E2E_BASE_URL` to match.
- Keep `retries: 0` for the tour: a retry would record a second video and defeat the "one continuous video" goal.
