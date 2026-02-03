# Debugging Exercise Bug Proposal

## Repo map (1–2 pages)

- **Module: `hofund-core` (core 20%)**
  - **Connection checks & metrics**  
    - `connection/AbstractHofundBasicHttpConnection.java`: core HTTP health check logic (timeouts, request method/headers, env-driven disable).
    - `connection/HofundConnection.java`: connection identity, tags, env-var naming; shared across metrics/table/graph.
    - `connection/HofundConnectionResult.java`: extracts version from HTTP responses.
    - `connection/HofundConnectionMeter.java`: Micrometer gauges for connection status.
    - `connection/HofundConnectionsTable.java`: ASCII table generation for connection status.
    - `connection/Version.java`: semantic-ish version parsing and comparisons.
  - **Graph visualization**  
    - `graph/node/HofundNodeMeter.java`: node graph metrics, ID collision checks.
    - `graph/edge/HofundEdgeMeter.java`: edge graph metrics, ID collision checks.
  - **Info & metadata meters**  
    - `info/HofundInfoMeter.java` + `HofundInfoProvider.java`: application name/version/type/icon.
    - `git/HofundGitInfoMeter.java` + `HofundGitInfoProvider.java`: git metadata tags.
    - `java/HofundJavaInfo*`, `os/HofundOsInfo*`, `web/HofundWebServerInfo*`: runtime metadata.
  - **Utilities**  
    - `AsciiTable.java`, `StringUtils.java`, `EnvProvider.java`.

- **Module: `hofund-spring` (core integrations)**
  - **DataSource connections**  
    - `connection/spring/datasource/DataSourceConnectionFactory.java`: DB product detection + connection wrapper selection.
    - `connection/spring/datasource/DatasourceConnection.java`: DB connection test query & timeout handling.
    - `connection/spring/datasource/*Connection.java`: DB-specific target/url derivation.
    - `connection/spring/datasource/UnknownDatasourceConnection.java`: fallback connection metadata discovery.
  - **HTTP connection provider**  
    - `connection/spring/http/HofundBasicHttpConnectionProvider.java`: adapts HTTP connections into providers.

- **Module: `hofund-spring-boot-autoconfigure`**
  - Auto-configuration for info/git/connection/graph/java/os/web meters and property mapping.
  - Key files: `HofundInfoAutoConfiguration.java`, `HofundGitInfoAutoConfiguration.java`,
    `HofundConnectionAutoConfiguration.java`, `HofundGraphAutoConfiguration.java`,
    `Hofund*Properties.java`.

- **Other modules**
  - `hofund-spring-boot-e2e`: end-to-end samples/tests.
  - `hofund-test-reports`: test report module.

**Core data flow:** Spring auto-config wires providers → meters → Prometheus scrape.  
Connection checks (HTTP/DB) and version parsing feed both metrics and the ASCII table, plus graph node/edge identities.

---

## Bug candidates

> Ranking scale: **1 (low) – 5 (high)** for each of Exercise Value / Stealth / Scorability.

### B01
- **Location:** `hofund-core/src/main/java/dev/logchange/hofund/connection/AbstractHofundBasicHttpConnection.java`  
  `testConnection()` ~lines 135–166
- **Core relevance:** HTTP connection checks are on the hot path for health status and version extraction.
- **Bug type:** Correctness / API contract drift
- **Proposed change:** Tighten success range to **2xx only** (e.g., `responseCode >= 200 && responseCode < 300`) so 3xx is treated as DOWN.
- **Trigger conditions:** Targets that respond with redirects (302/307), 304 caching, or L7 proxy behavior.
- **Expected symptom:** Connections shown as DOWN even though reachable; version might still be extracted or become UNKNOWN.
- **Why it’s hard:** Only affects redirecting services; behavior differs across environments (proxy config).
- **Static-analysis discoverability:** **Medium** (looks like a reasonable “strictness” tweak).
- **Suggested detection:** Integration test with 302 health endpoint; monitoring alert for mismatch between user traffic and hofund status.
- **Ranks:** Exercise **4**, Stealth **3**, Scorability **5**  
- **Note:** **Likely too easy unless direct test is removed** (there is a redirect response test).

### B02
- **Location:** `hofund-core/src/main/java/dev/logchange/hofund/connection/AbstractHofundBasicHttpConnection.java`  
  `testConnection()` ~lines 150–153
- **Core relevance:** Timeouts define availability checks and are part of core liveness checks.
- **Bug type:** Reliability / Timeout handling
- **Proposed change:** Copy/paste mistake: set read timeout using **connect timeout** (or swap the two).
- **Trigger conditions:** Custom read timeout > connect timeout; slow response bodies (large / cold).
- **Expected symptom:** Sporadic DOWN when response body is slow even though connection succeeds.
- **Why it’s hard:** Only surfaces under specific latency patterns; logs show “timeout” without obvious cause.
- **Static-analysis discoverability:** **Low** (both methods look similar and tests use equal timeouts).
- **Suggested detection:** Integration test with slow response body; synthetic latency/chaos test.
- **Ranks:** Exercise **4**, Stealth **4**, Scorability **4**

### B03
- **Location:** `hofund-core/src/main/java/dev/logchange/hofund/connection/HofundConnectionResult.java`  
  `extractVersionFromResponse()` ~lines 59–86
- **Core relevance:** Version parsing feeds metrics tags and version mismatch alerts.
- **Bug type:** Data integrity
- **Proposed change:** Drop the `"application"` anchor and pick the **first** `"version"` in the response body.
- **Trigger conditions:** Responses containing multiple “version” fields (e.g., git, dependencies, build).
- **Expected symptom:** Wrong detected version; false alarms in `checkVersions` and graph tags.
- **Why it’s hard:** Works for most payloads; only misbehaves when payloads include other version fields.
- **Static-analysis discoverability:** **Low** (string parsing is already ad‑hoc).
- **Suggested detection:** Property-based test generating JSON with multiple `version` fields.
- **Ranks:** Exercise **5**, Stealth **4**, Scorability **4**

### B04
- **Location:** `hofund-core/src/main/java/dev/logchange/hofund/connection/HofundConnection.java`  
  `getEnvVarName()` ~lines 139–143
- **Core relevance:** Env flag enables disabling connection checks; used operationally.
- **Bug type:** Edge-case input validation / Config parsing
- **Proposed change:** Normalize target by **removing** hyphens instead of replacing with underscores (or strip underscores).
- **Trigger conditions:** Target names containing `-` or `_`.
- **Expected symptom:** Env var set but connection still checked (disable doesn’t work).
- **Why it’s hard:** Only affects specific target naming styles; symptoms look like config drift.
- **Static-analysis discoverability:** **Medium** (looks like a harmless normalization tweak).
- **Suggested detection:** Unit test for env var name construction with hyphens/underscores.
- **Ranks:** Exercise **3**, Stealth **3**, Scorability **5**  
- **Note:** **Likely too easy unless direct test is removed** (env var naming is tested).

### B05
- **Location:** `hofund-core/src/main/java/dev/logchange/hofund/graph/node/HofundNodeMeter.java`  
  `checkIdCollision()` ~lines 41–54
- **Core relevance:** Graph IDs are core to node/edge visualization; collisions break graphs.
- **Bug type:** Concurrency / race condition
- **Proposed change:** Use `connections.parallelStream().forEach(...)` in `checkIdCollision()` without synchronizing `ids`.
- **Trigger conditions:** Many connections (large systems); parallelism enabled under load.
- **Expected symptom:** Non-deterministic collisions missed or sporadic `ConcurrentModificationException`.
- **Why it’s hard:** Nondeterministic; only surfaces with enough connections and parallel execution.
- **Static-analysis discoverability:** **Low** (parallel stream looks like a performance improvement).
- **Suggested detection:** Concurrency stress test with many connections and randomized order.
- **Ranks:** Exercise **5**, Stealth **5**, Scorability **3**

### B06
- **Location:** `hofund-core/src/main/java/dev/logchange/hofund/connection/HofundConnectionMeter.java`  
  `bindTo()` ~lines 29–34
- **Core relevance:** Connection status metrics are a primary output for monitoring.
- **Bug type:** Caching / invalidation
- **Proposed change:** Precompute status at registration and register gauge with a **constant** value.
- **Trigger conditions:** Connection status changes after startup.
- **Expected symptom:** Metrics never update (stale status).
- **Why it’s hard:** Looks like a performance optimization; only observable when status changes.
- **Static-analysis discoverability:** **Medium-Low** (Micrometer overloads are easy to misuse).
- **Suggested detection:** Integration test that flips a connection from UP→DOWN and asserts metric changes.
- **Ranks:** Exercise **4**, Stealth **4**, Scorability **4**

### B07
- **Location:** `hofund-spring/src/main/java/dev/logchange/hofund/connection/spring/datasource/DataSourceConnectionFactory.java`  
  `of()` ~lines 20–41
- **Core relevance:** DB connections are the most critical dependency type.
- **Bug type:** Resource leak / cleanup
- **Proposed change:** Remove try‑with‑resources (or hold onto connection) so the metadata connection isn’t closed.
- **Trigger conditions:** Multiple DataSources or repeated context refreshes.
- **Expected symptom:** Gradual exhaustion of DB pool; later connection checks fail.
- **Why it’s hard:** Leak grows slowly and appears as DB flakiness, not obviously tied to hofund.
- **Static-analysis discoverability:** **Medium** (resource lifecycle is subtle in factory code).
- **Suggested detection:** Leak detection on pool (Hikari leak detection) or load test with repeated init.
- **Ranks:** Exercise **5**, Stealth **4**, Scorability **4**

### B08
- **Location:** `hofund-spring/src/main/java/dev/logchange/hofund/connection/spring/datasource/DatasourceConnection.java`  
  `testConnection()` ~lines 49–54
- **Core relevance:** DB health checks are critical and on every scrape.
- **Bug type:** Performance regression / reliability
- **Proposed change:** Treat `QUERY_TIMEOUT` as milliseconds (e.g., multiply by 1000 or set to 0 for “no timeout”).
- **Trigger conditions:** DB latency spikes or stalls.
- **Expected symptom:** Metrics scrape hangs; thread pool pressure; cascading timeouts.
- **Why it’s hard:** Only appears during partial outages, looks like DB latency not hofund.
- **Static-analysis discoverability:** **Low** (JDBC timeout units are non-obvious).
- **Suggested detection:** Integration test with slow query; alert on metrics endpoint latency.
- **Ranks:** Exercise **4**, Stealth **5**, Scorability **4**

### B09
- **Location:** `hofund-spring/src/main/java/dev/logchange/hofund/connection/spring/datasource/UnknownDatasourceConnection.java`  
  constructor / `deriveTarget()` ~lines 26–103
- **Core relevance:** Fallback path used for unsupported or misconfigured DataSources.
- **Bug type:** Security issue / data leak
- **Proposed change:** When user is blank, set **target = full JDBC URL** (instead of sanitized last segment).
- **Trigger conditions:** JDBC URLs containing credentials or sensitive query parameters.
- **Expected symptom:** Credentials leaked via metrics tags, logs, or Grafana labels.
- **Why it’s hard:** Only affects fallback path; leaks surface in observability systems, not app logs.
- **Static-analysis discoverability:** **Medium** (requires understanding that tags are exported).
- **Suggested detection:** Security test scanning metrics output for secrets; SAST rule for JDBC URL tags.
- **Ranks:** Exercise **5**, Stealth **4**, Scorability **4**

### B10
- **Location:** `hofund-spring-boot-autoconfigure/src/main/java/dev/logchange/hofund/git/springboot/autoconfigure/HofundGitInfoAutoConfiguration.java`  
  `getBuildTime()` ~lines 58–61
- **Core relevance:** Git/build metadata is exposed in metrics; used by dashboards.
- **Bug type:** Time/date/timezone handling
- **Proposed change:** “Normalize” build time by parsing to `LocalDateTime` (dropping UTC offset)
  and reformatting without timezone.
- **Trigger conditions:** `git.build.time` includes `Z`/offset (typical in git.properties).
- **Expected symptom:** Build time appears shifted in dashboards; cross‑region comparisons fail.
- **Why it’s hard:** Subtle; value is “plausible” but wrong by hours.
- **Static-analysis discoverability:** **Low** (time parsing looks reasonable).
- **Suggested detection:** Unit test comparing parsed/expected UTC offset; regression test with fixed `git.build.time`.
- **Ranks:** Exercise **4**, Stealth **4**, Scorability **3**

### B11
- **Location:** `hofund-spring/src/main/java/dev/logchange/hofund/connection/spring/datasource/postgresql/PostgreSQLConnection.java`  
  constructor ~lines 23–33
- **Core relevance:** DB target naming drives metrics tags and graph IDs; used everywhere.
- **Bug type:** Internationalization / correctness
- **Proposed change:** Replace `toLowerCase(Locale.ROOT)` with `toLowerCase()` (default locale).
- **Trigger conditions:** Turkish/Azeri locale and target names containing “I/İ” (e.g., INVENTORY).
- **Expected symptom:** Target tag differs between services; env-var disable names and graph edges stop matching.
- **Why it’s hard:** Only manifests in specific locales and names; looks like config drift.
- **Static-analysis discoverability:** **Low** (locale bugs are easy to miss).
- **Suggested detection:** Locale-specific unit test using `Locale("tr","TR")`.
- **Ranks:** Exercise **4**, Stealth **5**, Scorability **4**

### B12
- **Location:** `hofund-core/src/main/java/dev/logchange/hofund/connection/Version.java`  
  `compareTo()` ~lines 52–63
- **Core relevance:** Version comparison drives required-version checks and alerts.
- **Bug type:** Numeric precision / correctness
- **Proposed change:** Use `Math.min(thisParts.length, otherParts.length)` for loop length (ignore extra segments).
- **Trigger conditions:** Version strings with different segment counts (e.g., `1.2` vs `1.2.3`).
- **Expected symptom:** Outdated services treated as up-to-date; missing or delayed alerts.
- **Why it’s hard:** Only occurs on mixed-length versions; looks like a data issue.
- **Static-analysis discoverability:** **Medium** (logic looks plausible).
- **Suggested detection:** Unit test for differing segment lengths.  
- **Note:** **Likely too easy unless direct test is removed** (there is a length-variance test).
- **Ranks:** Exercise **4**, Stealth **3**, Scorability **5**

### B13
- **Location:** `hofund-core/src/main/java/dev/logchange/hofund/connection/HofundConnectionsTable.java`  
  `checkVersions()` ~lines 71–76
- **Core relevance:** Table output is a primary diagnostic view for connection health.
- **Bug type:** Error handling gap / data integrity
- **Proposed change:** Change guard to `if (version.isUnspecified() && requiredVersion.isUnspecified()) return;`.
- **Trigger conditions:** One side is `N/A` or `UNKNOWN`, the other is set.
- **Expected symptom:** `IllegalArgumentException` thrown during comparison; row falls back to DOWN/UNKNOWN.
- **Why it’s hard:** Looks like a connection failure instead of version-logic failure.
- **Static-analysis discoverability:** **Low** (condition looks innocuous).
- **Suggested detection:** Unit test where required version is `N/A` and current is set.
- **Ranks:** Exercise **4**, Stealth **4**, Scorability **4**

### B14
- **Location:** `hofund-core/src/main/java/dev/logchange/hofund/connection/HofundConnectionsTable.java`  
  `print()` ~lines 37–51
- **Core relevance:** Table generation triggers live connection checks in many environments.
- **Bug type:** Performance regression / consistency
- **Proposed change:** Call `connection.getFun().get().getConnection()` twice (once for status, once for version).
- **Trigger conditions:** Slow or flaky endpoints; non-idempotent checks.
- **Expected symptom:** Double traffic per refresh; occasional mismatch between status and version in a single row.
- **Why it’s hard:** Nondeterministic; looks like remote flakiness.
- **Static-analysis discoverability:** **Medium** (duplicate calls are easy to overlook).
- **Suggested detection:** Integration test counting connection invocations per table render.
- **Ranks:** Exercise **5**, Stealth **4**, Scorability **3**

### B15
- **Location:** `hofund-core/src/main/java/dev/logchange/hofund/connection/HofundConnectionResult.java`  
  `parseResponseBody()` ~lines 42–45
- **Core relevance:** Version parsing feeds tags, alerts, and dashboards.
- **Bug type:** Encoding / data integrity
- **Proposed change:** Use `new InputStreamReader(..., StandardCharsets.US_ASCII)` (or ISO-8859-1).
- **Trigger conditions:** Response bodies with non-ASCII characters in JSON fields.
- **Expected symptom:** Version extraction fails or returns UNKNOWN; tags become inconsistent.
- **Why it’s hard:** Passes ASCII-only tests; fails only for localized payloads.
- **Static-analysis discoverability:** **Low** (charset choice looks intentional).
- **Suggested detection:** Unit test with UTF-8 response containing non-ASCII.
- **Ranks:** Exercise **4**, Stealth **5**, Scorability **4**

### B16
- **Location:** `hofund-core/src/main/java/dev/logchange/hofund/connection/HofundConnection.java`  
  `getTags()` ~lines 104–110
- **Core relevance:** Tags define graph edges and metric identity for all connections.
- **Bug type:** API contract drift / data integrity
- **Proposed change:** Swap the values for `source` and `target` tags.
- **Trigger conditions:** Grafana node graph or dashboards expecting `source=app` and `target=dependency`.
- **Expected symptom:** Edges appear reversed or missing; dependency graphs look inverted.
- **Why it’s hard:** Metrics still emit; only visualizations or query semantics break.
- **Static-analysis discoverability:** **Medium** (strings appear reasonable).
- **Suggested detection:** Graph rendering integration test with a known dependency edge.  
- **Note:** **Likely too easy unless direct test is removed** (there are tag tests).
- **Ranks:** Exercise **4**, Stealth **3**, Scorability **5**

### B17
- **Location:** `hofund-core/src/main/java/dev/logchange/hofund/graph/edge/HofundEdgeMeter.java`  
  `checkIdCollision()` ~lines 47–55
- **Core relevance:** Graph edges must be uniquely identified to render correctly.
- **Bug type:** Correctness / uniqueness validation
- **Proposed change:** Check collisions using `connection.toTargetTag()` instead of `connection.getEdgeId(infoProvider)`.
- **Trigger conditions:** Multiple edges to the same target with different types/descriptions.
- **Expected symptom:** Startup fails with “edge id must be unique” even when edges are distinct.
- **Why it’s hard:** Only in richer graphs; exception message points to data, not logic.
- **Static-analysis discoverability:** **Low**.
- **Suggested detection:** Unit test with two edges sharing target but different type/description.
- **Ranks:** Exercise **5**, Stealth **4**, Scorability **3**

### B18
- **Location:** `hofund-spring-boot-autoconfigure/src/main/java/dev/logchange/hofund/git/springboot/autoconfigure/HofundDefaultGitInfoProperties.java`  
  `@PropertySource` ~lines 6–8
- **Core relevance:** Git metadata is core to observability tagging.
- **Bug type:** Reliability / config loading
- **Proposed change:** Remove `ignoreResourceNotFound = true` from `@PropertySource`.
- **Trigger conditions:** Environments without `git.properties` (local/dev builds).
- **Expected symptom:** Application fails to start due to missing resource.
- **Why it’s hard:** CI/prod often have git.properties, so failure only in some envs.
- **Static-analysis discoverability:** **Medium**.
- **Suggested detection:** Boot test without git.properties on classpath.
- **Ranks:** Exercise **4**, Stealth **4**, Scorability **4**

### B19
- **Location:** `hofund-spring-boot-autoconfigure/src/main/java/dev/logchange/hofund/git/springboot/autoconfigure/HofundGitInfoAutoConfiguration.java`  
  `defaultIfEmpty()` ~lines 65–71
- **Core relevance:** Git metadata tags are used in dashboards and rollouts.
- **Bug type:** Config precedence / data integrity
- **Proposed change:** Invert the condition so defaults override explicit values.
- **Trigger conditions:** Users set custom git info values via properties.
- **Expected symptom:** Tags show stale/default values despite explicit configuration.
- **Why it’s hard:** Looks like binding or build-pipeline issue.
- **Static-analysis discoverability:** **Low** (logic reads plausibly).
- **Suggested detection:** Unit test asserting configured values win over defaults.
- **Ranks:** Exercise **4**, Stealth **4**, Scorability **4**

### B20
- **Location:** `hofund-spring-boot-autoconfigure/src/main/java/dev/logchange/hofund/web/springboot/autoconfigure/HofundWebServerInfoAutoConfiguration.java`  
  `tomcatHofundWebServerInfoProvider()` ~lines 26–30
- **Core relevance:** Web server metadata is core for runtime diagnostics.
- **Bug type:** Compatibility / dependency boundary
- **Proposed change:** Remove `@ConditionalOnClass(ServerInfo.class)` from the Tomcat provider.
- **Trigger conditions:** Running on Undertow/Netty without Tomcat classes.
- **Expected symptom:** `NoClassDefFoundError` at startup; auto-config fails.
- **Why it’s hard:** Works in Tomcat-based environments; fails only on other containers.
- **Static-analysis discoverability:** **Medium**.
- **Suggested detection:** Boot test with Undertow or WebFlux server.
- **Ranks:** Exercise **4**, Stealth **4**, Scorability **4**

---

## Top 10 recommended set

- **B11** – Locale-specific target mismatch; subtle and high-impact in i18n envs.
- **B12** – Version compare truncation; silently misses required-version alerts.
- **B13** – Unspecified-version guard bug; surfaces as false DOWN rows.
- **B14** – Double health checks per row; causes inconsistency and load spikes.
- **B15** – ASCII-only parsing; breaks version detection for UTF-8 payloads.
- **B16** – Source/target tag swap; graph semantics inverted without errors.
- **B17** – Edge collision false positives; breaks graph on realistic dependency sets.
- **B18** – Missing git.properties startup failure; environment-specific reliability trap.
- **B19** – Config precedence inversion; stale git metadata despite overrides.
- **B20** – Tomcat-only class usage; crashes on Undertow/Netty deployments.

---

## Next step (instances I01–I05)

Goal: Each instance has two bugs with **mixed types** (correctness/data, reliability/perf, security/time, concurrency/caching).

- **I01:** B11 (i18n correctness) + B18 (config loading)  
  *Locale sensitivity + environment-specific reliability.*
- **I02:** B12 (numeric correctness) + B16 (API contract drift)  
  *Version precision + graph semantics.*
- **I03:** B13 (error handling) + B20 (compatibility)  
  *Hidden exceptions + container-specific startup failures.*
- **I04:** B14 (performance/consistency) + B19 (config precedence)  
  *Load amplification + metadata drift.*
- **I05:** B15 (encoding/data integrity) + B17 (graph correctness)  
  *UTF-8 parsing + edge identity validation.*

