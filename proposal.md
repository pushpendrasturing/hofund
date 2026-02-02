# Repo map
- Core metrics and connection engine (hofund-core)
  - connection/*: HTTP and DB connection checks, connection result parsing, status and type enums, version comparison, connection metrics and table (HofundConnection, HofundConnectionResult, AbstractHofundBasicHttpConnection, HofundConnectionMeter, HofundConnectionsTable, Version).
  - graph/*: Node and Edge meters for Grafana Node Graph (HofundNodeMeter, HofundEdgeMeter) that depend on connection IDs and tags.
  - info/git/os/java/web: info providers and meters for application, git metadata, OS, Java, web server (HofundInfoMeter, HofundGitInfoMeter, HofundOsInfoMeter, HofundJavaInfoMeter, HofundWebServerInfoMeter).
  - util: AsciiTable, StringUtils, EnvProvider.
- Spring integration (hofund-spring)
  - connection.spring.datasource: DataSource detection, DB vendor-specific parsing, test queries (DataSourceConnectionsProvider, DataSourceConnectionFactory, DatasourceConnection, OracleConnection, PostgreSQLConnection, H2Connection, UnknownDatasourceConnection).
  - connection.spring.http: Provider bridging AbstractHofundBasicHttpConnection beans into metrics (HofundBasicHttpConnectionProvider).
- Spring Boot autoconfigure (hofund-spring-boot-autoconfigure)
  - Auto-wiring of meters and providers; property binding for info and git; conditional enablement for Prometheus (HofundInfoAutoConfiguration/Properties, HofundGitInfoAutoConfiguration/Properties/Default, HofundConnectionAutoConfiguration, HofundGraphAutoConfiguration, HofundOsInfoAutoConfiguration, HofundJavaInfoAutoConfiguration, HofundWebServerInfoAutoConfiguration, ConnectionTabelAutoConfigure).
- E2E and tests (hofund-spring-boot-e2e, hofund-core/src/test, hofund-spring/src/test)
  - Integration examples and tests; not on runtime hot paths.
- Non-core assets
  - grafana-dashboards and changelog; docs and release metadata.

Hot paths and data flow:
- Prometheus scrape -> micrometer Gauge -> HofundConnectionMeter -> ConnectionFunction -> HTTP/DB checks.
- Tags and IDs flow from HofundConnection -> graph meters (node/edge) and dashboards.
- Auto-configuration wires providers, meters, and properties in Spring Boot apps.

# Bug candidates

### B01 - Treat 4xx as UP for HTTP checks
- Location: /workspace/hofund-core/src/main/java/dev/logchange/hofund/connection/AbstractHofundBasicHttpConnection.java, testConnection() ~135-166
- Core relevance: HTTP connection checks feed hofund_connection metrics and graph edges.
- Bug type: correctness (HTTP status classification)
- Proposed change: change success condition to `responseCode < 500` (allow 4xx as UP).
- Trigger conditions: endpoints returning 401/403/404/429.
- Expected symptom: connection metrics show UP while dependency is rejecting calls.
- Why its hard: 4xx can be intermittent and dashboards still show healthy.
- Static-analysis discoverability: Medium.
- Suggested detection: integration test hitting a 401 endpoint and asserting DOWN.
- Rank (1-5): Exercise value 4, Stealth 4, Scorability 5

### B02 - Skip disable checks until after connect
- Location: /workspace/hofund-core/src/main/java/dev/logchange/hofund/connection/AbstractHofundBasicHttpConnection.java, testConnection() ~135-151
- Core relevance: hot path for each HTTP dependency check.
- Bug type: reliability/perf (unnecessary network calls)
- Proposed change: move CheckingStatus and env-var checks below openConnection/connect.
- Trigger conditions: connection disabled by config or env.
- Expected symptom: network calls/timeouts despite connection marked INACTIVE.
- Why its hard: metrics still show INACTIVE; only visible in traffic or logs.
- Static-analysis discoverability: Low.
- Suggested detection: unit test with EnvProvider mocked verifying no connect.
- Rank (1-5): Exercise value 3, Stealth 4, Scorability 4

### B03 - Swap connect/read timeouts
- Location: /workspace/hofund-core/src/main/java/dev/logchange/hofund/connection/AbstractHofundBasicHttpConnection.java, testConnection() ~150-152
- Core relevance: timeouts are critical for connection reliability.
- Bug type: reliability/perf (timeout misconfiguration)
- Proposed change: set connect timeout from getReadTimeout and read timeout from getConnectTimeout.
- Trigger conditions: slow handshake vs slow response.
- Expected symptom: false DOWNs during slow connects or hangs during slow reads.
- Why its hard: manifests only under certain network conditions.
- Static-analysis discoverability: Low/Medium.
- Suggested detection: integration test with delayed connect vs delayed response.
- Rank (1-5): Exercise value 3, Stealth 4, Scorability 4

### B04 - Ignore custom request method and always use GET
- Location: /workspace/hofund-core/src/main/java/dev/logchange/hofund/connection/AbstractHofundBasicHttpConnection.java, testConnection() ~153
- Core relevance: HTTP checks support custom request methods.
- Bug type: API contract drift / correctness
- Proposed change: hardcode GET instead of getRequestMethod().
- Trigger conditions: endpoints requiring POST/HEAD or pre-signed requests.
- Expected symptom: connection DOWN only for those endpoints; others OK.
- Why its hard: most users use GET; failure appears endpoint-specific.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test using POST with a mock server.
- Rank (1-5): Exercise value 4, Stealth 3, Scorability 5

### B05 - Report INACTIVE connections as DOWN
- Location: /workspace/hofund-core/src/main/java/dev/logchange/hofund/connection/AbstractHofundBasicHttpConnection.java, testConnection() ~140-148
- Core relevance: inactive semantics affect alerting and graphs.
- Bug type: correctness (state mapping)
- Proposed change: return Status.DOWN instead of Status.INACTIVE when disabled.
- Trigger conditions: any connection disabled by config or env.
- Expected symptom: dashboards show outages for intentionally disabled checks.
- Why its hard: only seen in environments that disable checks.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test for INACTIVE mapping; alert rule check for -1.
- Rank (1-5): Exercise value 3, Stealth 3, Scorability 5

### B06 - Disconnect before reading response body
- Location: /workspace/hofund-core/src/main/java/dev/logchange/hofund/connection/AbstractHofundBasicHttpConnection.java, testConnection() ~157-163
- Core relevance: version extraction uses the same HttpURLConnection.
- Bug type: data integrity / resource handling
- Proposed change: call urlConn.disconnect() before HofundConnectionResult.http(Status.UP, urlConn).
- Trigger conditions: any UP response with version in body.
- Expected symptom: detected_version becomes UNKNOWN while status remains UP.
- Why its hard: status looks fine; version loss is subtle.
- Static-analysis discoverability: Low.
- Suggested detection: integration test asserting detected_version from a known JSON response.
- Rank (1-5): Exercise value 3, Stealth 4, Scorability 4

### B07 - Skip first request header
- Location: /workspace/hofund-core/src/main/java/dev/logchange/hofund/connection/AbstractHofundBasicHttpConnection.java, setRequestHeaders() ~86-95
- Core relevance: headers often carry auth or versioning.
- Bug type: input handling / off-by-one
- Proposed change: iterate headers starting at index 1 (skip first).
- Trigger conditions: first header is required (Authorization, API key).
- Expected symptom: some connections fail with 401/403 only when headers configured.
- Why its hard: order-dependent; with multiple headers only first is missing.
- Static-analysis discoverability: Low.
- Suggested detection: unit test with single Authorization header and mock server.
- Rank (1-5): Exercise value 3, Stealth 4, Scorability 4

### B08 - Env var name normalization drops underscores
- Location: /workspace/hofund-core/src/main/java/dev/logchange/hofund/connection/HofundConnection.java, getEnvVarName() ~139-143
- Core relevance: env-based disable is a key operational control.
- Bug type: config parsing / validation
- Proposed change: use regex "[^A-Za-z0-9]" so underscores are stripped.
- Trigger conditions: targets containing "_" (common in service names).
- Expected symptom: env var disable no longer works for those targets.
- Why its hard: only affects certain target names; no compile errors.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test verifying env var naming for targets with underscores/dashes.
- Rank (1-5): Exercise value 3, Stealth 4, Scorability 4

### B09 - Drop description from toTargetTag for DATABASE/QUEUE
- Location: /workspace/hofund-core/src/main/java/dev/logchange/hofund/connection/HofundConnection.java, toTargetTag() ~57-64
- Core relevance: tag identity drives node/edge uniqueness in Grafana graph.
- Bug type: data integrity / ID collision
- Proposed change: always return target + "_" + type for DATABASE/QUEUE (ignore description).
- Trigger conditions: multiple DBs/queues with same target but different vendor/description.
- Expected symptom: node/edge collisions; one connection overwrites another.
- Why its hard: only appears in multi-DB setups; metrics still exist but merged.
- Static-analysis discoverability: Low.
- Suggested detection: integration test with two datasources same target different vendor.
- Rank (1-5): Exercise value 4, Stealth 4, Scorability 4

### B10 - Edge ID uses target instead of toTargetTag
- Location: /workspace/hofund-core/src/main/java/dev/logchange/hofund/connection/HofundConnection.java, getEdgeId() ~71-77
- Core relevance: edge IDs must align with node IDs for graph.
- Bug type: correctness / ID mismatch
- Proposed change: build edge ID using getTarget() rather than toTargetTag().
- Trigger conditions: DB/QUEUE connections with descriptions.
- Expected symptom: graph edges disconnected or overwritten.
- Why its hard: only in graph view; connection metrics still look normal.
- Static-analysis discoverability: Low/Medium.
- Suggested detection: Grafana node-graph snapshot test comparing expected edge ids.
- Rank (1-5): Exercise value 4, Stealth 4, Scorability 4

### B11 - Swap detected_version and required_version tags
- Location: /workspace/hofund-core/src/main/java/dev/logchange/hofund/connection/HofundConnection.java, getTags() ~104-112
- Core relevance: version tags drive alerts and dashboards.
- Bug type: data integrity
- Proposed change: swap Tag.of("detected_version", ...) with "required_version".
- Trigger conditions: when required version is configured.
- Expected symptom: dashboards show required as detected and vice versa; alerts fire incorrectly.
- Why its hard: values look plausible; only careful comparison reveals swap.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test with requiredVersion set and expected tags.
- Rank (1-5): Exercise value 3, Stealth 4, Scorability 5

### B12 - "target" tag uses getTarget() instead of toTargetTag()
- Location: /workspace/hofund-core/src/main/java/dev/logchange/hofund/connection/HofundConnection.java, getTags() ~106-109
- Core relevance: "target" tag powers graph linkage and queries.
- Bug type: correctness / ID mismatch
- Proposed change: Tag.of("target", getTarget()).
- Trigger conditions: DB/QUEUE connections with description/vendor.
- Expected symptom: graph edges not matching nodes; duplicate targets.
- Why its hard: only in multi-DB setups; not obvious from metrics alone.
- Static-analysis discoverability: Low.
- Suggested detection: integration test verifying target tag for DB connection includes vendor.
- Rank (1-5): Exercise value 4, Stealth 4, Scorability 4

### B13 - Use Status.ordinal() for gauge value
- Location: /workspace/hofund-core/src/main/java/dev/logchange/hofund/connection/HofundConnectionMeter.java, bindTo() ~31-34
- Core relevance: gauge value feeds prometheus alerts.
- Bug type: numeric correctness
- Proposed change: map to status.ordinal() instead of getValue().
- Trigger conditions: any INACTIVE status.
- Expected symptom: INACTIVE becomes 2 instead of -1; alerts misfire.
- Why its hard: values still numeric; dashboards might not show obvious issue.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test mapping Status.INACTIVE to -1.
- Rank (1-5): Exercise value 4, Stealth 3, Scorability 5

### B14 - Cache connection result at registration (stale status)
- Location: /workspace/hofund-core/src/main/java/dev/logchange/hofund/connection/HofundConnectionMeter.java, bindTo() ~30-34
- Core relevance: hofund_connection is the primary health signal.
- Bug type: caching/invalidation
- Proposed change: compute HofundConnectionResult once and bind gauge to that constant.
- Trigger conditions: connection status changes after startup.
- Expected symptom: metrics never update; stale UP/DOWN values.
- Why its hard: only visible after changes; initial startup looks fine.
- Static-analysis discoverability: Low.
- Suggested detection: integration test toggling endpoint availability and expecting gauge changes.
- Rank (1-5): Exercise value 5, Stealth 4, Scorability 4

### B15 - Treat equal versions as too low
- Location: /workspace/hofund-core/src/main/java/dev/logchange/hofund/connection/HofundConnectionsTable.java, checkVersions() ~71-77
- Core relevance: connection table logs are operational signal.
- Bug type: correctness (comparison boundary)
- Proposed change: use <= instead of < when comparing versions.
- Trigger conditions: current version equals required version.
- Expected symptom: false error logs about version mismatch.
- Why its hard: only a log, not a failure; easily ignored.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test for equal versions not logging error.
- Rank (1-5): Exercise value 2, Stealth 3, Scorability 5

### B16 - Exception path marks INACTIVE instead of DOWN
- Location: /workspace/hofund-core/src/main/java/dev/logchange/hofund/connection/HofundConnectionsTable.java, print() catch block ~53-65
- Core relevance: table output used at startup to detect failures.
- Bug type: error handling / correctness
- Proposed change: use Status.INACTIVE in catch block.
- Trigger conditions: connection function throws (timeouts, SQL errors).
- Expected symptom: failures hidden as inactive; operators miss real outage.
- Why its hard: only visible in connection table logs.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test with failing connection function expecting DOWN.
- Rank (1-5): Exercise value 3, Stealth 3, Scorability 5

### B17 - Read only the first line of HTTP response
- Location: /workspace/hofund-core/src/main/java/dev/logchange/hofund/connection/HofundConnectionResult.java, parseResponseBody() ~42-52
- Core relevance: version extraction for HTTP connections.
- Bug type: edge-case input handling
- Proposed change: return after the first readLine() to optimize.
- Trigger conditions: pretty-printed JSON or multi-line responses.
- Expected symptom: detected_version becomes UNKNOWN for some services.
- Why its hard: depends on response formatting; minified JSON still works.
- Static-analysis discoverability: Low.
- Suggested detection: unit test with multi-line JSON body.
- Rank (1-5): Exercise value 3, Stealth 4, Scorability 4

### B18 - Search for "version" from the start of the body
- Location: /workspace/hofund-core/src/main/java/dev/logchange/hofund/connection/HofundConnectionResult.java, extractVersionFromResponse() ~64-76
- Core relevance: version tags in metrics.
- Bug type: correctness (parsing)
- Proposed change: remove applicationIndex and search from 0.
- Trigger conditions: response contains other "version" fields before application.version.
- Expected symptom: wrong detected_version (for example build.version).
- Why its hard: looks plausible; only manifests with extra fields.
- Static-analysis discoverability: Low.
- Suggested detection: unit test with JSON containing multiple "version" keys.
- Rank (1-5): Exercise value 3, Stealth 4, Scorability 4

### B19 - Skip parsing version for UP responses
- Location: /workspace/hofund-core/src/main/java/dev/logchange/hofund/connection/HofundConnectionResult.java, http(Status, HttpURLConnection) ~35-39
- Core relevance: detected_version is part of core metrics tags.
- Bug type: data integrity / performance optimization
- Proposed change: if status == UP, return UNKNOWN without reading body.
- Trigger conditions: any healthy HTTP connection with version info.
- Expected symptom: detected_version always UNKNOWN despite valid responses.
- Why its hard: status still UP; only version metrics degrade.
- Static-analysis discoverability: Medium.
- Suggested detection: integration test verifying version extraction when UP.
- Rank (1-5): Exercise value 3, Stealth 3, Scorability 4

### B20 - Compare versions lexicographically
- Location: /workspace/hofund-core/src/main/java/dev/logchange/hofund/connection/Version.java, compareTo() ~33-65
- Core relevance: required version enforcement in connection table.
- Bug type: numeric precision / comparison
- Proposed change: replace numeric split with String.compareTo().
- Trigger conditions: versions with multi-digit parts (10.0 vs 2.0).
- Expected symptom: false "version too low" or missed mismatch.
- Why its hard: only shows when digits exceed 9; looks correct for small versions.
- Static-analysis discoverability: Low/Medium.
- Suggested detection: unit test comparing 2.10 vs 2.2.
- Rank (1-5): Exercise value 4, Stealth 4, Scorability 5

### B21 - Treat UNKNOWN/N/A as equal instead of throwing
- Location: /workspace/hofund-core/src/main/java/dev/logchange/hofund/connection/Version.java, compareTo() ~39-46
- Core relevance: version checks gate alerting.
- Bug type: correctness / error handling
- Proposed change: when either version is unspecified, return 0 instead of throwing.
- Trigger conditions: required version missing or unknown.
- Expected symptom: checkVersions silently skips mismatches; no error logs.
- Why its hard: behavior is absence of signal rather than a failure.
- Static-analysis discoverability: Low.
- Suggested detection: unit test expecting IllegalArgumentException for UNKNOWN vs 1.0.
- Rank (1-5): Exercise value 3, Stealth 4, Scorability 4

### B22 - Ignore extra version segments
- Location: /workspace/hofund-core/src/main/java/dev/logchange/hofund/connection/Version.java, compareTo() ~52
- Core relevance: version ordering determines error logs.
- Bug type: correctness (boundary handling)
- Proposed change: use Math.min instead of Math.max for length.
- Trigger conditions: comparing 1.2 vs 1.2.1 or 2.0 vs 2.0.0.1.
- Expected symptom: versions with extra segments compare as equal.
- Why its hard: only shows with uneven segment counts.
- Static-analysis discoverability: Low/Medium.
- Suggested detection: unit test where 1.2.1 > 1.2.
- Rank (1-5): Exercise value 4, Stealth 4, Scorability 5

### B23 - Lowercase DB product name before enum mapping
- Location: /workspace/hofund-spring/src/main/java/dev/logchange/hofund/connection/spring/datasource/DataSourceConnectionFactory.java, of() ~23-28
- Core relevance: determines vendor-specific DB parsing and test queries.
- Bug type: correctness (classification)
- Proposed change: lowercase metaData.getDatabaseProductName() before DatabaseProductName.of.
- Trigger conditions: any database; enum names are capitalized.
- Expected symptom: all databases become NOT_RECOGNIZED, losing vendor-specific parsing.
- Why its hard: still functional via UnknownDatasourceConnection; silent behavior change.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test verifying PostgreSQLConnection is chosen for PostgreSQL.
- Rank (1-5): Exercise value 4, Stealth 3, Scorability 4

### B24 - Use `==` for string comparison in DatabaseProductName.of
- Location: /workspace/hofund-spring/src/main/java/dev/logchange/hofund/connection/spring/datasource/DatabaseProductName.java, of() ~17-21
- Core relevance: vendor detection path.
- Bug type: correctness (equality)
- Proposed change: replace equals with `==` in the filter.
- Trigger conditions: non-interned product names (most JDBC drivers).
- Expected symptom: vendor detection fails; falls back to NOT_RECOGNIZED.
- Why its hard: looks like micro-optimization; likely too easy unless direct test is removed.
- Static-analysis discoverability: High.
- Suggested detection: unit test for DatabaseProductName.of("PostgreSQL").
- Rank (1-5): Exercise value 2, Stealth 2, Scorability 5

### B25 - Deduplicate datasources by target only
- Location: /workspace/hofund-spring/src/main/java/dev/logchange/hofund/connection/spring/datasource/DataSourceConnectionsProvider.java, getConnections() ~35-43
- Core relevance: determines which DB connections are exposed.
- Bug type: data integrity / de-duplication
- Proposed change: treat two DatasourceConnection as duplicates when target matches, ignoring url/vendor.
- Trigger conditions: multiple DBs with same target name but different hosts.
- Expected symptom: missing connection metrics for one datasource.
- Why its hard: only surfaces in multi-datasource deployments.
- Static-analysis discoverability: Low.
- Suggested detection: integration test with two datasources same target, different URL.
- Rank (1-5): Exercise value 4, Stealth 4, Scorability 4

### B26 - Use `==` for result comparison in DB test
- Location: /workspace/hofund-spring/src/main/java/dev/logchange/hofund/connection/spring/datasource/DatasourceConnection.java, testConnection() ~54-55
- Core relevance: DB health check is critical for hofund_connection.
- Bug type: correctness (string comparison)
- Proposed change: replace Objects.equals(resultSet.getString(1), "1") with `==`.
- Trigger conditions: any DB check; string is not interned.
- Expected symptom: DB connections always DOWN.
- Why its hard: likely too easy; would be caught by existing tests if present.
- Static-analysis discoverability: High (likely too easy unless the direct test is removed).
- Suggested detection: unit/integration test that expects UP when query returns 1.
- Rank (1-5): Exercise value 3, Stealth 2, Scorability 5

### B27 - Set query timeout after executing query
- Location: /workspace/hofund-spring/src/main/java/dev/logchange/hofund/connection/spring/datasource/DatasourceConnection.java, testConnection() ~51-53
- Core relevance: controls DB check latency and stability.
- Bug type: reliability/perf
- Proposed change: move statement.setQueryTimeout(...) to after executeQuery().
- Trigger conditions: slow or hung DB connections.
- Expected symptom: checks hang longer than expected; thread pool starvation.
- Why its hard: only under DB slowness; no obvious code error.
- Static-analysis discoverability: Low.
- Suggested detection: integration test with delayed DB response and timeout assertion.
- Rank (1-5): Exercise value 4, Stealth 4, Scorability 4

### B28 - Cache a JDBC Connection in DatasourceConnection
- Location: /workspace/hofund-spring/src/main/java/dev/logchange/hofund/connection/spring/datasource/DatasourceConnection.java, testConnection() ~49-63
- Core relevance: DB checks run frequently under prometheus scrape.
- Bug type: concurrency / resource leak
- Proposed change: store Connection in a field and reuse across checks instead of try-with-resources.
- Trigger conditions: concurrent scrapes or connection close by pool.
- Expected symptom: intermittent SQLExceptions, leaked connections, or stale results.
- Why its hard: nondeterministic and load-dependent.
- Static-analysis discoverability: Medium.
- Suggested detection: load test with concurrent scrapes; leak detection in pool.
- Rank (1-5): Exercise value 5, Stealth 4, Scorability 3

### B29 - Off-by-one when parsing PostgreSQL DB name
- Location: /workspace/hofund-spring/src/main/java/dev/logchange/hofund/connection/spring/datasource/postgresql/PostgreSQLConnection.java, constructor ~26-32
- Core relevance: target name drives graph identity and metrics.
- Bug type: correctness (string slicing)
- Proposed change: set `to = url.lastIndexOf("?") - 1` when query exists.
- Trigger conditions: JDBC URL with query parameters.
- Expected symptom: target missing last character; graph IDs mismatch.
- Why its hard: only with query params; looks like similar name.
- Static-analysis discoverability: Low.
- Suggested detection: unit test for URL with query string.
- Rank (1-5): Exercise value 3, Stealth 4, Scorability 4

### B30 - Use first ":" instead of last ":" in H2 target parsing
- Location: /workspace/hofund-spring/src/main/java/dev/logchange/hofund/connection/spring/datasource/h2/H2Connection.java, constructor ~26-29
- Core relevance: target name for H2 DB.
- Bug type: correctness (string parsing)
- Proposed change: use url.indexOf(':') instead of lastIndexOf(':').
- Trigger conditions: typical H2 URLs (jdbc:h2:mem:test).
- Expected symptom: target becomes "h2" or "mem" instead of db name.
- Why its hard: only shows in graph/metrics labels; DB still UP.
- Static-analysis discoverability: Low.
- Suggested detection: unit test for target derivation from H2 URL.
- Rank (1-5): Exercise value 3, Stealth 4, Scorability 4

### B31 - Derive Oracle target from URL instead of username
- Location: /workspace/hofund-spring/src/main/java/dev/logchange/hofund/connection/spring/datasource/oracle/OracleConnection.java, constructor ~25-28
- Core relevance: target name used for ID and graph nodes.
- Bug type: correctness / data integrity
- Proposed change: parse target from URL segment instead of metadata username.
- Trigger conditions: Oracle deployments with multiple schemas or shared host.
- Expected symptom: collisions across schemas; metrics merge.
- Why its hard: only in multi-schema or shared Oracle hosts.
- Static-analysis discoverability: Low.
- Suggested detection: integration test with two Oracle schemas same host.
- Rank (1-5): Exercise value 3, Stealth 4, Scorability 4

### B32 - Use split("?") when stripping JDBC query params
- Location: /workspace/hofund-spring/src/main/java/dev/logchange/hofund/connection/spring/datasource/UnknownDatasourceConnection.java, deriveTarget() ~88-92
- Core relevance: fallback path for unknown datasources.
- Bug type: input parsing / regex pitfall
- Proposed change: replace split("\\?") with split("?").
- Trigger conditions: JDBC URL that includes "?" parameters.
- Expected symptom: noQuery becomes empty or malformed; target becomes "unknown".
- Why its hard: only affects unknown datasource path; regex bug is subtle.
- Static-analysis discoverability: Low.
- Suggested detection: unit test for deriveTarget with JDBC URL containing query params.
- Rank (1-5): Exercise value 3, Stealth 4, Scorability 4

### B33 - Log JDBC URL and username at INFO
- Location: /workspace/hofund-spring/src/main/java/dev/logchange/hofund/connection/spring/datasource/UnknownDatasourceConnection.java, constructor ~26-45
- Core relevance: runs in fallback path but still in core DB discovery.
- Bug type: security (information disclosure)
- Proposed change: add log.info(...) with discoveredUrl and discoveredUser.
- Trigger conditions: any datasource where URL includes credentials or sensitive hostnames.
- Expected symptom: sensitive information in logs.
- Why its hard: not a functional failure; only visible in logs and audits.
- Static-analysis discoverability: Medium.
- Suggested detection: security review or log-scrubbing tests.
- Rank (1-5): Exercise value 3, Stealth 3, Scorability 4

### B34 - Node ID tag uses getTarget() instead of toTargetTag()
- Location: /workspace/hofund-core/src/main/java/dev/logchange/hofund/graph/node/HofundNodeMeter.java, tagsForConnection() ~82-86
- Core relevance: node IDs must match edge target tags.
- Bug type: correctness / graph integrity
- Proposed change: Tag.of("id", connection.getTarget()).
- Trigger conditions: DB/QUEUE connections with description/vendor.
- Expected symptom: nodes not matching edges; graph partial.
- Why its hard: only in graph; connection metrics still look ok.
- Static-analysis discoverability: Low.
- Suggested detection: grafana node-graph snapshot test.
- Rank (1-5): Exercise value 4, Stealth 4, Scorability 4

### B35 - Collision check compares against application version
- Location: /workspace/hofund-core/src/main/java/dev/logchange/hofund/graph/node/HofundNodeMeter.java, checkIdCollision() ~51-53
- Core relevance: prevents ID collisions that break graph.
- Bug type: correctness (logic)
- Proposed change: compare ids.contains(infoProvider.getApplicationVersion()) instead of name.
- Trigger conditions: when a connection target matches the app name.
- Expected symptom: collision goes undetected; nodes merged.
- Why its hard: only if names overlap; exception removed.
- Static-analysis discoverability: Low/Medium.
- Suggested detection: unit test where target equals app name.
- Rank (1-5): Exercise value 3, Stealth 4, Scorability 4

### B36 - Edge collision check uses toTargetTag instead of edgeId
- Location: /workspace/hofund-core/src/main/java/dev/logchange/hofund/graph/edge/HofundEdgeMeter.java, checkIdCollision() ~50-55
- Core relevance: ensures unique edge IDs across the graph.
- Bug type: data integrity
- Proposed change: compare only connection.toTargetTag() for uniqueness.
- Trigger conditions: multiple edges from different sources to same target.
- Expected symptom: collisions not detected; edges overwritten.
- Why its hard: only visible in graph; subtle in metrics.
- Static-analysis discoverability: Low.
- Suggested detection: integration test with two services calling same target.
- Rank (1-5): Exercise value 3, Stealth 4, Scorability 4

### B37 - Stop lowercasing application name
- Location: /workspace/hofund-spring-boot-autoconfigure/src/main/java/dev/logchange/hofund/info/springboot/autoconfigure/HofundInfoAutoConfiguration.java, getApplicationName() ~37-39
- Core relevance: app name is a core tag and graph ID.
- Bug type: correctness / config contract drift
- Proposed change: return properties.getApplication().getName() without toLowerCase().
- Trigger conditions: app names with uppercase or mixed case.
- Expected symptom: graph IDs and target matching fail; env-var name mismatches.
- Why its hard: only manifests with case-sensitive comparisons across modules.
- Static-analysis discoverability: Medium.
- Suggested detection: integration test asserting lowercased id tag.
- Rank (1-5): Exercise value 4, Stealth 4, Scorability 4

### B38 - Swap application_name and application_version tags
- Location: /workspace/hofund-core/src/main/java/dev/logchange/hofund/info/HofundInfoMeter.java, tags() ~36-38
- Core relevance: hofund_info is a key metric for dashboards.
- Bug type: data integrity
- Proposed change: set application_name tag to provider.getApplicationVersion() and vice versa.
- Trigger conditions: any deployment.
- Expected symptom: labels are wrong; dashboards and alerts mislabel services.
- Why its hard: values look plausible unless checked carefully.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test verifying tag values from provider.
- Rank (1-5): Exercise value 3, Stealth 3, Scorability 5

### B39 - Reverse precedence in defaultIfEmpty
- Location: /workspace/hofund-spring-boot-autoconfigure/src/main/java/dev/logchange/hofund/git/springboot/autoconfigure/HofundGitInfoAutoConfiguration.java, defaultIfEmpty() ~65-71
- Core relevance: git metadata tags used widely in dashboards.
- Bug type: config/override handling
- Proposed change: if val is empty return val, else return defaultVal (reverse logic).
- Trigger conditions: custom hofund.git-info.* properties set by user.
- Expected symptom: user overrides ignored; tags show defaults.
- Why its hard: looks like user misconfiguration; no crashes.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test that overrides git-info properties and expects them to win.
- Rank (1-5): Exercise value 4, Stealth 4, Scorability 5

### B40 - Normalize build_time to local time without offset
- Location: /workspace/hofund-spring-boot-autoconfigure/src/main/java/dev/logchange/hofund/git/springboot/autoconfigure/HofundDefaultGitInfoProperties.java, getBuildTime() ~36-37
- Core relevance: build_time tag used for debugging deployments.
- Bug type: time/date/timezone
- Proposed change: parse git.build.time to LocalDateTime and return without timezone/offset.
- Trigger conditions: build_time contains offset or Z; deployments across timezones.
- Expected symptom: build_time appears shifted; hard to correlate with other systems.
- Why its hard: only visible when comparing across timezones; looks like input issue.
- Static-analysis discoverability: Low.
- Suggested detection: unit test with known build_time and expected offset preserved.
- Rank (1-5): Exercise value 3, Stealth 4, Scorability 4

# Top 10 recommended set
- B14 - Stale connection status caching in core metric; high impact and stealth.
- B25 - Datasource de-dup by target only; subtle data loss in multi-DS setups.
- B01 - 4xx treated as UP; misleading health signal with realistic triggers.
- B37 - Case-sensitive app name mismatch; cross-module and hard to spot.
- B20 - Lexicographic version compare; classic numeric precision trap.
- B27 - Query timeout set too late; reliability/perf under load.
- B39 - Git info override precedence reversed; config drift hard to spot.
- B40 - build_time timezone normalization; time drift across systems.
- B29 - PostgreSQL off-by-one parsing; edge-case string bug.
- B33 - JDBC URL logging at INFO; security training value.

# Instances (I01-I05)
Each instance mixes correctness, config, parsing, and reliability/perf issues and spreads higher-ranked bugs.

- I01: B14, B25, B07, B15, B30, B34, B19, B21
- I02: B01, B37, B06, B11, B23, B32, B12, B03
- I03: B20, B27, B02, B08, B17, B36, B24, B31
- I04: B39, B40, B05, B10, B18, B22, B35, B16
- I05: B29, B33, B04, B09, B13, B26, B28, B38
