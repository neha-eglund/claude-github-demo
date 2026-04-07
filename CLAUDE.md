# Project: Payment API Pentest Suite

A Java 17 Maven project for automated penetration testing of payment APIs, with nightly execution via GitHub Actions. Currently configured to run against the **Stripe sandbox API**.

---

## Project Structure

```
payment-api-pentest/
├── pom.xml                                          Maven build (Java 17)
├── config/
│   ├── pentest.properties.example                   Config template — commit-safe, no real values
│   └── pentest.properties                           Local config — git-ignored, fill in real values
├── src/test/java/com/example/pentest/
│   ├── base/
│   │   ├── ConfigLoader.java                        Loads config from env vars or properties file
│   │   ├── ApiClient.java                           HTTP client (RestAssured + OkHttp, Basic auth)
│   │   └── PentestBase.java                         Shared @BeforeAll setup for all test classes
│   ├── auth/
│   │   └── AuthStripeApiKeyTest.java                API key auth tests (invalid, empty, Bearer)
│   ├── accesscontrol/
│   │   ├── BolaIdorPaymentTest.java                 IDOR/BOLA, path traversal, cross-account access
│   │   └── MassAssignmentPaymentTest.java           Mass assignment, negative amounts, status bypass
│   ├── injection/
│   │   ├── InjectionSqlPaymentTest.java             SQLi, command injection, oversized metadata
│   │   └── SsrfPaymentCallbackTest.java             SSRF via webhook URL registration
│   ├── ratelimit/
│   │   └── RateLimitPaymentSubmissionTest.java      Burst requests, oversized payloads, idempotency
│   ├── exposure/
│   │   └── ExposurePanInResponseTest.java           PAN/CVV masking, HSTS, CORS, path traversal
│   └── report/
│       └── PentestReportSummary.java                JUnit listener → target/pentest-reports/summary.txt
├── src/test/resources/
│   ├── logback-test.xml                             Test logging config
│   └── META-INF/services/...TestExecutionListener   Auto-registers PentestReportSummary
└── .github/workflows/
    ├── claude.yml                                   Claude PR assistant (triggered by @claude mentions)
    ├── claude-code-review.yml                       Claude auto code review on PRs
    └── nightly-pentest.yml                          Nightly pentest job (02:00 UTC daily)
```

---

## Configuration

Tests read credentials from **environment variables** (priority) or `config/pentest.properties` (local dev fallback). Never commit real credentials.

### Required config keys

| Key | Description |
|---|---|
| `PENTEST_BASE_URL` | Target API base URL (e.g. `https://api.stripe.com`) |
| `PENTEST_USER_TOKEN` | API key / user credential |
| `PENTEST_ADMIN_TOKEN` | Admin API key / credential |
| `PENTEST_API_KEY` | Primary API key used by `ApiClient` |
| `PENTEST_TEST_ACCOUNT_ID` | Test account/customer ID (e.g. Stripe `cus_xxx`) |
| `TARGET_ENV` | `staging` (default) or `sandbox` — set to `sandbox` to enable destructive tests |

### Local setup

```bash
cp config/pentest.properties.example config/pentest.properties
# Fill in values, then run:
mvn test
```

### GitHub Actions secrets

Add these in **Settings → Secrets and variables → Actions**:
- `PENTEST_BASE_URL`, `PENTEST_USER_TOKEN`, `PENTEST_ADMIN_TOKEN`, `PENTEST_API_KEY`, `PENTEST_TEST_ACCOUNT_ID`
- `ANTHROPIC_API_KEY` — required for `claude.yml` and `claude-code-review.yml`

---

## Running Tests

| Command | What it runs |
|---|---|
| `mvn test` | All non-destructive tests |
| `mvn test -Dgroups=auth` | Auth tests only |
| `mvn test -Dgroups=auth,injection` | Multiple tag groups |
| `mvn test -DexcludedGroups=destructive` | Skip destructive tests |
| `mvn allure:report` | Generate HTML report → `target/site/allure-maven-plugin/index.html` |

### JUnit tags

| Tag | Test class |
|---|---|
| `auth` | `AuthStripeApiKeyTest` |
| `bola` | `BolaIdorPaymentTest`, `MassAssignmentPaymentTest` |
| `injection` | `InjectionSqlPaymentTest`, `SsrfPaymentCallbackTest` |
| `ratelimit` | `RateLimitPaymentSubmissionTest` |
| `exposure` | `ExposurePanInResponseTest` |
| `destructive` | Tests that create/modify real data — require `TARGET_ENV=sandbox` |

---

## Stripe-Specific Notes

The suite is adapted for **Stripe's sandbox API**:
- **Auth**: HTTP Basic auth — API key as username, empty password (not Bearer JWT)
- **POST bodies**: `application/x-www-form-urlencoded` (not JSON)
- **Amounts**: in cents (e.g. `100` = $1.00)
- **Endpoints**: `/v1/payment_intents`, `/v1/charges`, `/v1/customers`, `/v1/webhook_endpoints`
- **Customer ID**: obtain via `curl https://api.stripe.com/v1/customers -u sk_test_...: -d name=Test`

### Rotating the Stripe key

1. Stripe Dashboard → **Developers → API keys → Roll key**
2. Update `config/pentest.properties` locally
3. Update GitHub Actions secrets

---

## Known Security Findings

### SSRF — Stripe Webhook Validation Gap (CRITICAL)

Stripe's sandbox **accepts** RFC 1918 private IPs and cloud metadata endpoints as webhook registration targets:

| URL | Stripe response |
|---|---|
| `http://169.254.169.254/latest/meta-data/` | **200 Accepted** |
| `http://10.0.0.1/admin` | **200 Accepted** |
| `http://192.168.1.1/router` | **200 Accepted** |
| `http://127.0.0.1/internal` | 400 Blocked ✓ |
| `http://localhost:8080/admin` | 400 Blocked ✓ |

These 3 tests **intentionally fail** in CI to surface the finding. Stripe likely blocks actual HTTP delivery at the network layer, but the registration gap is a validated vulnerability. On any custom payment API, all RFC 1918 ranges and cloud metadata endpoints must be rejected at the URL validation layer.

---

## GitHub Workflows

### `nightly-pentest.yml`
- Runs at **02:00 UTC** every night
- Excludes `@Tag("destructive")` tests by default
- Manual trigger available from GitHub Actions UI (with optional tag group + env inputs)
- Uploads artifacts for 30 days: Allure results, findings summary, Surefire XML
- Prints findings summary to the **GitHub Actions job summary tab**
- Requires GitHub Environment `pentest-staging` (**Settings → Environments**)

### `claude-code-review.yml`
- Triggers on PRs that change `.java`, `.yml`, or `.properties` files
- Requires `ANTHROPIC_API_KEY` secret

### `claude.yml`
- Responds to `@claude` mentions in PR/issue comments and reviews

---

## Development Standards

- Java 17+ — use Records, Text Blocks, Switch Expressions where appropriate
- No hardcoded secrets — all credentials via env vars or git-ignored properties file
- Test naming: `given_<precondition>_when_<action>_then_<securityOutcome>`
- Security findings surface as **hard test failures** so they appear in CI reports
- Destructive tests gated behind `TARGET_ENV=sandbox`
