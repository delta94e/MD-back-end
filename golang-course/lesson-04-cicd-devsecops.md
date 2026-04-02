# Lesson 04 — CI/CD, GitHub Actions & DevSecOps

> **Mục tiêu:** Xây dựng pipeline tự động hoàn chỉnh — từ lúc push code
> đến khi app được deploy lên production một cách an toàn, có kiểm soát.

---

## Phần 1: CI/CD Là Gì?

### 1.1 Khái Niệm

```
╔══════════════════════════════════════════════════════════════════════╗
║  CI / CD PIPELINE                                                    ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  CI — Continuous Integration (Tích hợp liên tục):                  ║
║  → Mỗi lần push code, tự động chạy:                                ║
║     - Lint / format check                                           ║
║     - Unit tests + integration tests                                ║
║     - Code coverage                                                  ║
║     - Security scan                                                  ║
║     - Build Docker image                                             ║
║  → Mục tiêu: phát hiện lỗi NGAY KHI code được push                ║
║                                                                      ║
║  CD — Continuous Delivery (Phân phối liên tục):                    ║
║  → Sau khi CI pass, tự động:                                       ║
║     - Push image lên registry                                       ║
║     - Deploy lên staging (tự động)                                  ║
║     - Deploy lên production (cần approval thủ công)                 ║
║                                                                      ║
║  CD — Continuous Deployment (Triển khai liên tục):                 ║
║  → Giống Delivery nhưng production cũng tự động                    ║
║  → Chỉ dùng khi test coverage + monitoring rất tốt                 ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 1.2 Luồng Tổng Quan

```
Developer → git push → GitHub
                           │
                    GitHub Actions
                           │
              ┌────────────┴────────────┐
              │          CI             │
              │  1. Lint & Format       │
              │  2. Unit Tests          │
              │  3. Integration Tests   │
              │  4. Security Scan       │
              │  5. Build Docker Image  │
              └────────────┬────────────┘
                           │ (pass)
              ┌────────────┴────────────┐
              │          CD             │
              │  1. Push to Registry    │
              │  2. Deploy to Staging   │
              │  3. Smoke Tests         │
              │  4. [Approval]          │
              │  5. Deploy to Prod      │
              └─────────────────────────┘
```

---

## Phần 2: Git Flow & Branching Strategy

### 2.1 Git Flow (Mô Hình Đầy Đủ)

```
╔══════════════════════════════════════════════════════════════════════╗
║  GIT FLOW — BRANCHING MODEL                                          ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  main        ─●────────────────────────────●──→  (production)      ║
║               ↑                            ↑                         ║
║  release    ──●──(test)──(fix)─────────────┘                        ║
║               ↑                                                      ║
║  develop   ─●─────●────────●────●──────────────────→   (staging)   ║
║               ↑   ↑        ↑    ↑                                   ║
║  feature      │   └feature │    └feature/payment                    ║
║               └feature/login    └────(merged)──┘                    ║
║                                                                      ║
║  hotfix    ──────────────────────────●────→ main & develop          ║
║                                      ↑ urgent bug fix               ║
║                                                                      ║
║  Branches:                                                           ║
║  main     = production code (luôn stable, tagged)                   ║
║  develop  = integration branch (next release)                        ║
║  feature/ = tính năng mới (từ develop, merge vào develop)           ║
║  release/ = chuẩn bị release (chỉ fix bug, không feature mới)      ║
║  hotfix/  = fix bug khẩn cấp trên production (từ main)             ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 2.2 GitHub Flow (Đơn Giản Hơn — Phù Hợp Startup)

```
main        ─●──────────────────────────────────────→
              ↑
feature/     └─●──(commit)──(commit)──PR──review──merge
```

```bash
# GitHub Flow workflow hàng ngày:

# 1. Tạo branch mới từ main
git checkout main && git pull
git checkout -b feature/user-authentication

# 2. Code và commit theo Conventional Commits
git add .
git commit -m "feat(auth): add JWT authentication middleware"
git commit -m "test(auth): add unit tests for JWT validation"
git commit -m "docs(auth): update README with auth endpoints"

# 3. Push và tạo Pull Request
git push origin feature/user-authentication
# → Tạo PR trên GitHub → CI chạy → Code review → Merge
```

### 2.3 Conventional Commits — Chuẩn Commit Message

```bash
# Format: <type>(<scope>): <description>
#         [optional body]
#         [optional footer]

# Types:
feat:     tính năng mới
fix:      bug fix
docs:     thay đổi documentation
style:    format, whitespace (không ảnh hưởng logic)
refactor: refactor code (không phải feat, không phải fix)
test:     thêm hoặc sửa tests
chore:    build process, dependency updates
perf:     cải thiện performance
ci:       thay đổi CI configuration
revert:   revert commit trước

# Ví dụ:
git commit -m "feat(user): add profile update endpoint"
git commit -m "fix(auth): correct token expiration handling"
git commit -m "feat!: redesign user API (breaking change)"
#  ↑ dấu ! = breaking change → semver MAJOR bump
git commit -m "chore(deps): upgrade gin to v1.9.1"
git commit -m "ci: add integration test job to pipeline"
```

### 2.4 Semantic Versioning

```
MAJOR.MINOR.PATCH

v1.0.0 → v1.0.1  (patch: bug fix)
v1.0.1 → v1.1.0  (minor: tính năng mới, backward compatible)
v1.1.0 → v2.0.0  (major: breaking change)

Git tags:
git tag -a v1.2.0 -m "Release v1.2.0"
git push origin v1.2.0
```

---

## Phần 3: GitHub Actions

### 3.1 Cấu Trúc Workflow

```
.github/
  workflows/
    ci.yml          # chạy khi push / PR
    cd-staging.yml  # deploy staging khi merge develop
    cd-prod.yml     # deploy production khi tag release
    security.yml    # security scan định kỳ
```

```yaml
# Anatomy của một GitHub Actions workflow
name: CI Pipeline

# Triggers — khi nào chạy
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]
  workflow_dispatch:              # cho phép trigger thủ công

# Environment variables dùng chung
env:
  GO_VERSION: "1.22"
  REGISTRY: "ghcr.io"
  IMAGE_NAME: ${{ github.repository }}

jobs:
  job-name:
    runs-on: ubuntu-latest        # runner: ubuntu, windows, macos
    timeout-minutes: 15           # tránh job chạy mãi mãi

    # Matrix build — chạy với nhiều config
    strategy:
      matrix:
        go-version: ["1.21", "1.22"]

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true             # cache Go modules

      - name: Run specific step
        run: echo "Hello World"

      - name: Use secret
        run: echo ${{ secrets.MY_SECRET }}  # lấy từ GitHub Secrets
```

### 3.2 CI Pipeline — Đầy Đủ

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

env:
  GO_VERSION: "1.22"

jobs:
  # ─── JOB 1: Lint ──────────────────────────────────────────────────
  lint:
    name: Lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true

      - name: golangci-lint
        uses: golangci/golangci-lint-action@v4
        with:
          version: latest
          args: --timeout=5m

  # ─── JOB 2: Test ──────────────────────────────────────────────────
  test:
    name: Test
    runs-on: ubuntu-latest

    # Services chạy song song (containers)
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true

      - name: Run unit tests
        run: go test -race -count=1 ./...

      - name: Run integration tests
        env:
          DB_HOST: localhost
          DB_PORT: 5432
          DB_NAME: testdb
          DB_USER: testuser
          DB_PASSWORD: testpass
          REDIS_ADDR: localhost:6379
        run: go test -v -tags=integration -race -count=1 ./...

      - name: Generate coverage report
        run: |
          go test -coverprofile=coverage.out ./...
          go tool cover -html=coverage.out -o coverage.html

      - name: Check coverage threshold
        run: |
          COVERAGE=$(go tool cover -func=coverage.out | tail -1 | awk '{print $3}' | tr -d '%')
          echo "Coverage: $COVERAGE%"
          if (( $(echo "$COVERAGE < 70" | bc -l) )); then
            echo "Coverage $COVERAGE% thấp hơn 70% threshold!"
            exit 1
          fi

      - name: Upload coverage
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage.html

  # ─── JOB 3: Build Docker Image ────────────────────────────────────
  build:
    name: Build
    runs-on: ubuntu-latest
    needs: [lint, test]           # chỉ build khi lint + test pass

    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}  # tự động có sẵn

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=sha,prefix=sha-

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha   # GitHub Actions cache
          cache-to: type=gha,mode=max
```

### 3.3 CD — Deploy Staging

```yaml
# .github/workflows/cd-staging.yml
name: Deploy Staging

on:
  push:
    branches: [develop]

jobs:
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    environment: staging          # cần approval trong GitHub Environments

    steps:
      - uses: actions/checkout@v4

      - name: Deploy to staging server
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.STAGING_HOST }}
          username: ${{ secrets.STAGING_USER }}
          key: ${{ secrets.STAGING_SSH_KEY }}
          script: |
            cd /opt/myapp-staging
            export APP_VERSION="sha-${{ github.sha }}"
            docker compose pull app
            docker compose up -d --no-deps --scale app=2 app
            sleep 5
            docker compose ps

      - name: Smoke test
        run: |
          sleep 10
          curl -f https://staging.yourdomain.com/health || exit 1
          echo "Staging deployment successful!"
```

### 3.4 CD — Deploy Production (Với Approval)

```yaml
# .github/workflows/cd-prod.yml
name: Deploy Production

on:
  push:
    tags:
      - "v*"                      # chỉ chạy khi push tag vX.Y.Z

jobs:
  deploy-prod:
    name: Deploy to Production
    runs-on: ubuntu-latest
    environment: production       # cần manual approval trên GitHub

    steps:
      - name: Deploy to production
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.PROD_HOST }}
          username: ${{ secrets.PROD_USER }}
          key: ${{ secrets.PROD_SSH_KEY }}
          script: |
            cd /opt/myapp
            export APP_VERSION="${{ github.ref_name }}"  # v1.2.0
            docker compose pull app
            # Rolling update: scale to 3, then back to 2
            docker compose up -d --no-deps --scale app=3 app
            sleep 10
            docker compose up -d --no-deps --scale app=2 app
            echo "Deployed $APP_VERSION to production"

      - name: Verify production
        run: |
          sleep 15
          curl -f https://yourdomain.com/health || exit 1

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          generate_release_notes: true  # tự tổng hợp changelog từ commits
```

---

## Phần 4: DevOps & DevSecOps

### 4.1 DevOps Là Gì?

```
╔══════════════════════════════════════════════════════════════════════╗
║  DEVOPS = Development + Operations                                   ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  Trước DevOps:                                                       ║
║  Dev team   → "code xong, ném qua tường cho Ops"                   ║
║  Ops team   → "chúng tôi không biết code này làm gì"               ║
║  → Release mất hàng tuần, bug nhiều, blame game                    ║
║                                                                      ║
║  Với DevOps:                                                         ║
║  Dev + Ops cùng chịu trách nhiệm → vận hành và phát triển          ║
║  → CI/CD tự động hóa → release nhanh hơn và ít lỗi hơn            ║
║                                                                      ║
║  DevOps Practices:                                                   ║
║  + Infrastructure as Code (IaC) — Terraform, Ansible               ║
║  + CI/CD Pipelines                                                   ║
║  + Monitoring & Alerting                                             ║
║  + Container Orchestration (Kubernetes)                             ║
║  + GitOps (Git = source of truth cho cả code lẫn infra)            ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 4.2 DevSecOps — Bảo Mật Tích Hợp Vào Pipeline

```
DevSecOps = DevOps + Security "shifted left"
→ Không check bảo mật ở cuối, check NGAY từ đầu pipeline

CI Pipeline với Security:
  Code → SAST (code scan) → Build → DAST (runtime scan) → Deploy
          ↑                           ↑
   Phát hiện sớm               Phát hiện runtime issues
```

```yaml
# .github/workflows/security.yml
name: Security Scan

on:
  push:
    branches: [main, develop]
  schedule:
    - cron: "0 0 * * 1"          # Quét mỗi thứ Hai hàng tuần

jobs:
  # ─── SAST — Static Application Security Testing ──────────────────
  sast:
    name: SAST - Code Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Gosec (Go security checker)
        uses: securego/gosec@master
        with:
          args: ./...

      # CodeQL — GitHub's SAST tool (miễn phí cho public repos)
      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: go

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3

  # ─── Dependency Scan ─────────────────────────────────────────────
  dependency-scan:
    name: Dependency Vulnerabilities
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: govulncheck (Go official)
        run: |
          go install golang.org/x/vuln/cmd/govulncheck@latest
          govulncheck ./...

      # Nancy — scan go.sum cho CVEs
      - name: Nancy dependency scanner
        uses: sonatype-nexus-community/nancy-github-action@main

  # ─── Container Image Scan ────────────────────────────────────────
  image-scan:
    name: Docker Image Scan
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Trivy — scan image cho CVEs
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: "ghcr.io/${{ github.repository }}:sha-${{ github.sha }}"
          format: "sarif"
          output: "trivy-results.sarif"
          severity: "CRITICAL,HIGH"
          exit-code: "1"          # fail nếu có CRITICAL/HIGH CVE

      - name: Upload Trivy results to Security tab
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: "trivy-results.sarif"

  # ─── Secret Scanning ─────────────────────────────────────────────
  secret-scan:
    name: Secret Detection
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0           # cần full history để scan

      - name: Gitleaks — phát hiện secrets bị commit
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## Phần 5: Release Flow Hoàn Chỉnh

### 5.1 Chiến Lược Release

```bash
# Quy trình từ feature → production:

# 1. Developer làm việc trên feature branch
git checkout -b feature/payment-integration develop
# ... code, test local ...
git push origin feature/payment-integration

# 2. Tạo Pull Request → develop
# → CI tự động chạy: lint + test + security scan
# → Code review bởi peers
# → Merge vào develop

# 3. develop → staging tự động
# → CD pipeline deploy lên staging
# → QA team test trên staging

# 4. Khi sẵn sàng release → tạo tag
git checkout main
git merge develop
git tag -a v1.3.0 -m "Release v1.3.0: add payment integration"
git push origin main --tags

# 5. Tag push → CD production pipeline kích hoạt
# → Cần approval của team lead trên GitHub
# → Deploy với rolling update
# → Smoke tests tự động
# → GitHub Release được tạo tự động
```

### 5.2 GitHub Environments & Secrets

```yaml
# Cấu hình trong GitHub → Settings → Environments

# Staging environment:
# - No protection rules (auto-deploy)
# - Secrets: STAGING_HOST, STAGING_USER, STAGING_SSH_KEY

# Production environment:
# - Required reviewers: [team-lead, senior-dev]
# - Wait timer: 5 minutes (thời gian đọc changes)
# - Secrets: PROD_HOST, PROD_USER, PROD_SSH_KEY

# Secrets cần setup (Settings → Secrets → Actions):
# STAGING_HOST     = IP address của staging server
# STAGING_USER     = ubuntu (hoặc user khác)
# STAGING_SSH_KEY  = nội dung private SSH key
# PROD_HOST        = IP address production
# PROD_USER        = ubuntu
# PROD_SSH_KEY     = private SSH key cho prod
# REGISTRY_TOKEN   = Docker registry token (nếu không dùng GHCR)
```

### 5.3 Makefile — Tích Hợp Release Flow

```makefile
VERSION = $(shell git describe --tags --always)
BRANCH  = $(shell git rev-parse --abbrev-ref HEAD)

.PHONY: release-patch release-minor release-major

# Tự động bump version và tạo tag
release-patch:
	@echo "Current: $(VERSION)"
	@NEW_VERSION=$$(semver bump patch $(VERSION)); \
	 git tag -a $$NEW_VERSION -m "Release $$NEW_VERSION"; \
	 git push origin $$NEW_VERSION; \
	 echo "Released: $$NEW_VERSION"

release-minor:
	@NEW_VERSION=$$(semver bump minor $(VERSION)); \
	 git tag -a $$NEW_VERSION -m "Release $$NEW_VERSION"; \
	 git push origin $$NEW_VERSION

release-major:
	@NEW_VERSION=$$(semver bump major $(VERSION)); \
	 git tag -a $$NEW_VERSION -m "Release $$NEW_VERSION"; \
	 git push origin $$NEW_VERSION

# Xem trạng thái CI của branch hiện tại
ci-status:
	gh run list --branch $(BRANCH) --limit 5

# Xem logs của run gần nhất
ci-logs:
	gh run view --log
```

---

## Phần 6: .golangci.yml — Lint Configuration

```yaml
# .golangci.yml
linters:
  enable:
    - errcheck        # kiểm tra error không bị ignore
    - gosimple        # gợi ý code đơn giản hơn
    - govet           # kiểm tra lỗi chính xác trong Go
    - ineffassign     # phát hiện gán giá trị không bao giờ dùng
    - staticcheck     # static analysis tool
    - unused          # phát hiện code không dùng đến
    - gofmt           # format check
    - goimports       # import ordering
    - gocritic        # opinionated linter
    - gosec           # security issues
    - misspell        # kiểm tra lỗi chính tả trong comments

linters-settings:
  gosec:
    excludes:
      - G401           # MD5 weak hash (OK nếu không dùng cho security)
  errcheck:
    exclude-functions:
      - (net/http.ResponseWriter).Write

issues:
  exclude-rules:
    - path: "_test.go"
      linters:
        - gosec        # test files OK không cần security check

run:
  timeout: 5m
  tests: true
```

---

## Phần 7: Bài Tập

### Bài 1 — Setup CI Pipeline

Tạo `.github/workflows/ci.yml` cho project của mình:
- Job `lint` với golangci-lint
- Job `test` với postgres + redis services
- Job `build` build Docker image và push lên GHCR
- Verify: tạo PR và xem CI chạy tự động

### Bài 2 — Setup CD Pipeline

Tạo CD pipeline cho staging:
- Tạo GitHub Environment `staging`
- Setup secrets (dùng staging server từ Lesson 03)
- Deploy tự động khi push vào `develop`
- Smoke test sau khi deploy

### Bài 3 — Security Scanning

Thêm security job vào pipeline:
- `govulncheck` kiểm tra dependencies
- `gosec` kiểm tra code
- `trivy` quét Docker image
- Verify: push code với một lỗ hổng giả định và xem pipeline fail

### Bài 4 — Release Flow Hoàn Chỉnh

Thực hiện full release flow:
```bash
# 1. Tạo feature branch
git checkout -b feature/my-new-feature develop

# 2. Code + commit theo conventional commits
git commit -m "feat(api): add new endpoint"

# 3. Push + tạo PR → CI runs → review → merge vào develop
# 4. Staging tự động deploy → kiểm tra trên staging

# 5. Release
git checkout main && git merge develop
git tag -a v1.0.0 -m "Initial release"
git push origin main --tags

# 6. Approve production deployment trên GitHub
# 7. Verify app chạy trên production domain
```

---

## Tổng Kết Lesson 04

```
╔══════════════════════════════════════════════════════════════════════╗
║  LESSON 04 — KEY TAKEAWAYS                                          ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  Git Workflow:                                                       ║
║  + Conventional Commits → changelog tự động, semver rõ ràng        ║
║  + GitHub Flow: main + feature branches (đơn giản, hiệu quả)       ║
║  + Tags → trigger production deployment                             ║
║                                                                      ║
║  CI Pipeline:                                                        ║
║  + Lint → Test → Build → Security Scan                             ║
║  + Services (postgres, redis) chạy trong CI dưới dạng containers   ║
║  + Coverage threshold → không merge nếu coverage giảm              ║
║  + Cache Go modules + Docker layers → CI nhanh hơn                 ║
║                                                                      ║
║  CD Pipeline:                                                        ║
║  + Staging: auto-deploy khi merge develop                           ║
║  + Production: manual approval + rolling update + smoke test        ║
║  + GitHub Environments → kiểm soát quyền deploy                    ║
║                                                                      ║
║  DevSecOps:                                                         ║
║  + "Shift left security" — phát hiện sớm, không phải cuối cùng    ║
║  + SAST (gosec, CodeQL) → lỗi trong code                           ║
║  + Dependency scan (govulncheck) → CVE trong thư viện              ║
║  + Image scan (Trivy) → CVE trong Docker image/OS packages         ║
║  + Secret scan (Gitleaks) → không bao giờ để secret trong git!     ║
║                                                                      ║
║  Sau lesson này:                                                    ║
║  → Mỗi lần git push, pipeline tự động test + deploy               ║
║  → Production chỉ deploy khi có approval                           ║
║  → Security được check tự động, không dựa vào con người           ║
╚══════════════════════════════════════════════════════════════════════╝
```
