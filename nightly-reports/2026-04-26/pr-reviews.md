# PR Review Sweep — 2026-04-26

Reviewed 5 open non-draft PRs against the CLAUDE.md code review rules.

---

## PR #24 — Add getInt helper to ConfigLoader

**Changed files:** `ConfigLoader.java` (+6 lines)

```java
public static int getInt(String key, int defaultValue) {
    String value = get(key, null);
    if (value == null) return defaultValue;
    return Integer.parseInt(value);
}
```

**Review:** No violations. Implementation is clean, correct, and minimal. `Integer.parseInt` throws `NumberFormatException` (unchecked) on malformed input — acceptable for test helpers per CLAUDE.md.

**VERDICT: APPROVE**

---

## PR #29 — Add SeverityClassifier utility for findings grouping

**Changed files:** `SeverityClassifier.java` (new, 97 lines), `nightly-dependency-audit.yml`, `CLAUDE.md`

### Findings

**[BLOCKING] `System.out.println` in `classify()` method**
The new `SeverityClassifier.java` contains bare `System.out.println` calls despite the same PR adding the rule "No `System.out.println` — use SLF4J". Replace with `log.info(...)` / `log.warn(...)`.

**[BLOCKING] Class not `final` with private constructor**
`SeverityClassifier` is a stateless utility class but is neither `final` nor has a private constructor, violating the CLAUDE.md rule: "Template classes must be `final` with private constructors — they are stateless utility classes."

**[ADVISORY] Magic numbers instead of named constants**
`classify(double)` uses literals `9`, `7`, `4` directly. The PR introduces three instance fields (`criticalThreshold = 9`, `highThreshold = 7`, `mediumThreshold = 4`) but `classify()` ignores them entirely. Either use the fields as `private static final` constants or remove the dead fields.

**[ADVISORY] Imperative loops instead of streams**
`groupBySeverity()`, `filterByMinSeverity()`, `summarize()` use index-based `for` loops accumulating into mutable lists. Replace with `stream().filter().collect()` per CLAUDE.md.

**[ADVISORY] Unquoted `$MVN_ARGS` in workflow**
`nightly-dependency-audit.yml` expands `$MVN_ARGS` without quoting; word-splitting will break if the value contains spaces.

**VERDICT: REQUEST_CHANGES** — 2 blocking findings (System.out.println, non-final utility class).

---

## PR #76 — Add TestConfig helper with credential fallbacks and ownership check test

**Changed files:** `TestConfig.java` (new, 27 lines), `PaymentOwnershipCheckTest.java` (new, 37 lines)

### Findings

**[BLOCKING] Hardcoded credential-shaped strings in `TestConfig.java`**
`getApiKey()` defaults to `"sk_test_devonly_replace_before_commit"` (Stripe-prefixed key format). CLAUDE.md: "reject any hardcoded API keys, tokens, passwords, or UUIDs that look like real credentials." The `sk_test_` prefix triggers secret-scanning tools regardless of intent. Replace hardcoded fallbacks with `throw new IllegalStateException("PENTEST_API_KEY must be set")` — never fall back to a credential-shaped literal.

**[BLOCKING] `PaymentOwnershipCheckTest` validates vulnerable behavior instead of rejecting it**
The test reads `callerUserId` from environment config but **never includes it in the request**. The assertion checks that `merchantId` matches in the response — which succeeds precisely when the IDOR vulnerability is present (any caller can read any payment with just merchantId). The test should assert `response.statusCode() == 403` with the cross-user userId in the request. As written, this test passes when the security bug exists and would never surface the finding in CI.

**[BLOCKING] `callerUserId` is declared and read but never used**
The variable is extracted from config and then ignored when building the request params. This makes the test a no-op for cross-user access control.

**[NIT]** Test method doesn't follow naming convention `given_<precondition>_when_<action>_then_<securityOutcome>`.

**VERDICT: REQUEST_CHANGES** — 3 blocking findings.

---

## PR #77 — Add PaymentFinding class for structured pentest report output

**Changed files:** `PaymentFinding.java` (new, 55 lines)

### Findings

**[BLOCKING] `System.out.println` in `printSummary()`**
`printSummary()` uses bare `System.out.println`. CLAUDE.md: "No `System.out.println` — use SLF4J for all logging." Replace with `log.info(...)`.

**[ADVISORY] Mutable POJO instead of Record**
`PaymentFinding` has four private fields with getters/setters. CLAUDE.md: "Records over POJOs — use Java records for immutable data carriers." This should be a record: `record PaymentFinding(String endpoint, String severity, String description, List<String> evidence)`.

**[ADVISORY] Mutable evidence list**
Evidence is initialized as `new ArrayList<>()` in the constructor and mutated via a setter. If converted to a record, pass an immutable `List.of(...)` snapshot at construction time.

**[NIT] if-else chain in `isCritical()`**
Use a switch expression instead: `return switch (severity) { case "CRITICAL", "HIGH" -> true; default -> false; };`

**VERDICT: REQUEST_CHANGES** — 1 blocking finding (System.out.println).

---

## PR #78 — Enable issue creation in code review workflow and consolidate base URLs

**Changed files:** `claude-code-review.yml`, `nightly-pentest.yml`

### Findings

**[BLOCKING] `PENTEST_CREATE_ISSUES: 'true'` in `claude-code-review.yml`**
CLAUDE.md is explicit: "Issue creation must remain opt-in — `PENTEST_CREATE_ISSUES=true` must only appear in `nightly-pentest.yml`, never in PR or branch workflows." Adding this to the code review workflow means every PR push triggers live GitHub issue creation. Remove this env var from `claude-code-review.yml` entirely.

**[BLOCKING] Stripe step now uses `${{ secrets.PENTEST_BASE_URL }}` instead of hardcoded `https://api.stripe.com`**
CLAUDE.md: "Stripe and Payment API steps must stay separate — they cannot share `PENTEST_BASE_URL`." The secret `PENTEST_BASE_URL` is intended for the Payment API (staging environment URL). If that secret is set, the Stripe step will now point at the Payment API server instead of Stripe's API, breaking all Stripe tests silently. The Stripe API base URL should always be the hardcoded literal `https://api.stripe.com`.

**VERDICT: REQUEST_CHANGES** — 2 blocking findings.

---

## Summary

| PR | Title | Verdict | Blocking Findings |
|---|---|---|---|
| #24 | Add getInt to ConfigLoader | ✅ APPROVE | 0 |
| #29 | Add SeverityClassifier | ❌ REQUEST_CHANGES | 2 |
| #76 | TestConfig + ownership test | ❌ REQUEST_CHANGES | 3 |
| #77 | PaymentFinding class | ❌ REQUEST_CHANGES | 1 |
| #78 | Issue creation + URL consolidation | ❌ REQUEST_CHANGES | 2 |
