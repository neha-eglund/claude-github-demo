# [Dependency Audit] Nightly Scan — 2026-04-26

## All Dependencies Checked (pom.xml)

| Library | Version | HIGH/CRITICAL CVEs | Status |
|---|---|---|---|
| rest-assured | 5.4.0 | None known | ✅ OK |
| rest-assured json-path | 5.4.0 | None known | ✅ OK |
| jackson-databind | 2.17.0 | None known in 2.17.x | ✅ OK |
| okhttp | 4.12.0 | None known | ✅ OK |
| junit-jupiter | 5.10.2 | None known | ✅ OK |
| junit-platform-launcher | 1.10.2 | None known | ✅ OK |
| assertj-core | 3.25.3 | None known | ✅ OK |
| nimbus-jose-jwt | 9.37.3 | CVE-2023-52428 fixed in 9.37.2 ✓; transitive json-smart risk ⚠️ | ⚠️ REVIEW |
| datafaker | 2.1.0 | None known | ✅ OK |
| allure-junit5 | 2.27.0 | None known | ✅ OK |
| swagger-parser | 2.1.21 | None known | ✅ OK |
| slf4j-api | 2.0.12 | None known | ✅ OK |
| logback-classic | 1.4.14 | CVE-2023-6481 fixed in 1.4.14 ✓ | ✅ OK |

## CVE Details

### ⚠️ nimbus-jose-jwt 9.37.3 — Transitive Risk via json-smart

**CVE-2023-52428** (HIGH, CVSS 7.5) — DoS via large JWE `p2c` header iteration count. Fixed in 9.37.2. Version 9.37.3 is **patched**.

**CVE-2023-1370** (HIGH, CVSS 8.6) — json-smart ≤ 2.4.10: StackOverflowError when parsing deeply nested JSON arrays/objects.
- Nimbus JOSE+JWT depends on net.minidev:json-smart transitively.
- If nimbus-jose-jwt 9.37.3 ships with json-smart ≤ 2.4.10, this CVE applies.
- **Action required**: Run `mvn dependency:tree | grep json-smart` to verify the resolved version.
- If json-smart ≤ 2.4.10 is resolved, upgrade nimbus-jose-jwt to latest (≥ 9.39.x ships json-smart 2.4.11+).

### ✅ logback-classic 1.4.14 — Patched

**CVE-2023-6481** (HIGH) — serialization DoS in logback receiver component affecting 1.4.x ≤ 1.4.11. Version 1.4.14 is the **patched** release.

## Build Environment Warning

`pom.xml` declares `<maven.compiler.source>24</maven.compiler.source>` (Java 24), but `JAVA_HOME` on the CI runner points to `java-21-openjdk-amd64` (Java 21). Maven will fail with "source release 24 requires target release 24" unless the workflow explicitly installs Java 24. Verify that the `nightly-pentest.yml` workflow sets up the correct JDK version.

## Recommended Actions

1. **[MEDIUM]** Run `mvn dependency:tree | grep json-smart` and confirm json-smart ≥ 2.4.11 is resolved; if not, bump nimbus-jose-jwt to ≥ 9.39.x.
2. **[LOW]** Confirm CI runner has Java 24 installed or downgrade pom.xml compiler source/target to match the runner (Java 21).
3. **[INFO]** All other dependencies are clean. No further action required.
