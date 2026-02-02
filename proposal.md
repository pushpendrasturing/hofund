# Repo map (core-focused)
- Connection checks and metrics (hofund-core/src/main/java/dev/logchange/hofund/connection)
  - AbstractHofundBasicHttpConnection, SimpleHofundHttpConnection: HTTP health checks, headers, timeouts, env-based disablement.
  - HofundConnection, HofundConnectionResult, Version: connection identity/tags, HTTP response parsing, version comparison.
  - HofundConnectionMeter, HofundConnectionsTable: Prometheus gauges and startup table output.
  - Status/Type/RequestMethod/RequestHeader: status values, tag typing, HTTP request method/headers.
- Graph metrics (hofund-core/src/main/java/dev/logchange/hofund/graph)
  - HofundNodeMeter, HofundEdgeMeter: Grafana node graph tags and collision checks.
- Info and metadata meters (hofund-core/src/main/java/dev/logchange/hofund/info, git, java, os, web)
  - HofundInfoMeter, HofundGitInfoMeter, HofundJavaInfoMeter, HofundOsInfoMeter, HofundWebServerInfoMeter.
- Utilities (hofund-core/src/main/java/dev/logchange/hofund)
  - AsciiTable, StringUtils, EnvProvider.

Hot path summary:
- Prometheus scrape -> hofund.connection gauge -> ConnectionFunction -> HTTP/DB checks.
- Tags/IDs from HofundConnection feed Grafana node/edge meters.
- Info/git/java/os/web meters provide global tags for observability.

# Additional core bug candidates (B41-B90)
These are new, core-only candidates beyond the previous 40. No duplicates in this list.

### B41 - Headers applied after connect (ignored by server)
- Location: hofund-core/.../connection/AbstractHofundBasicHttpConnection.java, testConnection() ~150-157
- Core relevance: HTTP checks are core health signals.
- Bug type: correctness / request construction
- Proposed change: call setRequestHeaders(urlConn) after urlConn.connect().
- Trigger conditions: any auth/header-required endpoint.
- Expected symptom: 401/403 or missing features only for header-dependent endpoints.
- Why its hard: default endpoints still OK; failures look like remote auth issues.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test with mock server expecting Authorization header.
- Rank (1-5): Exercise 4, Stealth 4, Scorability 4

### B42 - Treat only 2xx as success (3xx = DOWN)
- Location: hofund-core/.../connection/AbstractHofundBasicHttpConnection.java, testConnection() ~161-165
- Core relevance: HTTP status mapping drives hofund_connection.
- Bug type: correctness / boundary handling
- Proposed change: change success condition to responseCode >= 200 && < 300.
- Trigger conditions: endpoints returning 301/302 (redirects).
- Expected symptom: false DOWN for services behind redirects or TLS upgrades.
- Why its hard: depends on deployment routing; looks like infra issue.
- Static-analysis discoverability: Low/Medium.
- Suggested detection: integration test with 302 response.
- Rank (1-5): Exercise 3, Stealth 4, Scorability 4

### B43 - Case-sensitive env disable check and ignore "1"
- Location: hofund-core/.../connection/AbstractHofundBasicHttpConnection.java, isCheckingStatusInactiveByEnvs() ~198-203
- Core relevance: operational disablement is a key control.
- Bug type: config parsing
- Proposed change: use "true".equals(envVarValue) and remove the "1" check.
- Trigger conditions: env value "TRUE", "True", or "1".
- Expected symptom: disablement ignored only in some environments.
- Why its hard: looks like misconfiguration; no errors.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test with env values "TRUE" and "1".
- Rank (1-5): Exercise 4, Stealth 4, Scorability 4

### B44 - Cache URL at construction (stale on dynamic config)
- Location: hofund-core/.../connection/AbstractHofundBasicHttpConnection.java, constructor/getURL() ~24-114
- Core relevance: URL is the connection target for HTTP checks.
- Bug type: caching/invalidation
- Proposed change: store URL once in a field (new URL(getUrl())) and reuse it.
- Trigger conditions: getUrl() changes at runtime (dynamic config, rolling DNS).
- Expected symptom: checks continue hitting old endpoint after config update.
- Why its hard: only after runtime changes; looks like config not applied.
- Static-analysis discoverability: Low.
- Suggested detection: integration test that updates URL and expects new target.
- Rank (1-5): Exercise 4, Stealth 5, Scorability 4

### B45 - setDoOutput(true) for all requests before setRequestMethod
- Location: hofund-core/.../connection/AbstractHofundBasicHttpConnection.java, testConnection() ~150-154
- Core relevance: affects all HTTP checks.
- Bug type: correctness / protocol behavior
- Proposed change: add urlConn.setDoOutput(true) before setRequestMethod.
- Trigger conditions: GET/HEAD requests on some JDKs (doOutput may switch method to POST).
- Expected symptom: unexpected 405/400 on servers that disallow POST.
- Why its hard: JDK-specific and endpoint-dependent.
- Static-analysis discoverability: Low.
- Suggested detection: integration test verifying method stays GET for doOutput=true.
- Rank (1-5): Exercise 4, Stealth 4, Scorability 3

### B46 - withRequiredVersion resets request method to GET
- Location: hofund-core/.../connection/SimpleHofundHttpConnection.java, withRequiredVersion() ~95-97
- Core relevance: affects HTTP checks with required versions.
- Bug type: API contract drift
- Proposed change: construct new instance using RequestMethod.GET regardless of existing method.
- Trigger conditions: non-GET connections that also set required version.
- Expected symptom: endpoint fails only when requiredVersion is set.
- Why its hard: appears only for combined features.
- Static-analysis discoverability: Low/Medium.
- Suggested detection: unit test using POST + requiredVersion.
- Rank (1-5): Exercise 4, Stealth 4, Scorability 4

### B47 - Cache tags in HofundConnection (stale detected_version)
- Location: hofund-core/.../connection/HofundConnection.java, getTags() ~104-112
- Core relevance: tags power graph IDs and version reporting.
- Bug type: caching/invalidation
- Proposed change: cache the tags list on first call and return it for all future calls.
- Trigger conditions: detected_version changes over time.
- Expected symptom: version tag never updates after startup.
- Why its hard: status OK but version stale; subtle in dashboards.
- Static-analysis discoverability: Low.
- Suggested detection: integration test where target version changes.
- Rank (1-5): Exercise 5, Stealth 4, Scorability 4

### B48 - Source tag uses application version instead of name
- Location: hofund-core/.../connection/HofundConnection.java, getTags() ~106-108
- Core relevance: source tag ties edges to nodes.
- Bug type: data integrity / ID mismatch
- Proposed change: Tag.of("source", infoProvider.getApplicationVersion()).
- Trigger conditions: always.
- Expected symptom: edges no longer match node IDs; graph broken.
- Why its hard: metrics still emitted; graph issues look like Grafana config.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test for source tag value.
- Rank (1-5): Exercise 4, Stealth 4, Scorability 5

### B49 - Preserve description case in toTargetTag
- Location: hofund-core/.../connection/HofundConnection.java, toTargetTag() ~57-63
- Core relevance: target tag drives graph node identity.
- Bug type: data integrity
- Proposed change: remove .toLowerCase() for description.
- Trigger conditions: vendors described with mixed case (PostgreSQL vs postgresql).
- Expected symptom: duplicate nodes for same DB vendor across environments.
- Why its hard: only appears across mixed naming conventions.
- Static-analysis discoverability: Low.
- Suggested detection: integration test with same DB target, different case description.
- Rank (1-5): Exercise 3, Stealth 4, Scorability 4

### B50 - Edge ID omits type when description present
- Location: hofund-core/.../connection/HofundConnection.java, getEdgeId() ~71-77
- Core relevance: edge IDs must be unique across types.
- Bug type: data integrity / collision
- Proposed change: when description present, build id as appName + "-" + target + "_" + description (no type).
- Trigger conditions: same target + description across different types.
- Expected symptom: edge collisions and missing edges.
- Why its hard: requires multi-type targets; no direct errors.
- Static-analysis discoverability: Low.
- Suggested detection: integration test with HTTP and QUEUE same target+desc.
- Rank (1-5): Exercise 4, Stealth 4, Scorability 4

### B51 - Drop hyphens instead of converting to underscores
- Location: hofund-core/.../connection/HofundConnection.java, getEnvVarName() ~139-143
- Core relevance: env-based disablement is operationally important.
- Bug type: config parsing
- Proposed change: replace "-" with "" instead of "_".
- Trigger conditions: targets containing "-".
- Expected symptom: env var disablements silently ignored for dashed targets.
- Why its hard: looks like user error; no warnings.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test for target "payment-api".
- Rank (1-5): Exercise 3, Stealth 4, Scorability 4

### B52 - Type tag uses enum name (uppercase)
- Location: hofund-core/.../connection/HofundConnection.java, getTags() ~108-109
- Core relevance: type tag used in dashboards/queries.
- Bug type: API contract drift
- Proposed change: Tag.of("type", getType().name()) instead of toString().
- Trigger conditions: any deployment with dashboards expecting lowercase.
- Expected symptom: dashboards/alerts miss series due to label mismatch.
- Why its hard: looks like missing data rather than code bug.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test for type tag equals "http"/"database".
- Rank (1-5): Exercise 4, Stealth 4, Scorability 5

### B53 - Truncate HTTP response body to 512 chars
- Location: hofund-core/.../connection/HofundConnectionResult.java, parseResponseBody() ~42-51
- Core relevance: detected_version tag is core metadata.
- Bug type: data integrity / performance optimization
- Proposed change: stop reading after 512 chars.
- Trigger conditions: large JSON or version field after 512 chars.
- Expected symptom: detected_version becomes UNKNOWN intermittently.
- Why its hard: depends on response size/format.
- Static-analysis discoverability: Low.
- Suggested detection: unit test with large response body.
- Rank (1-5): Exercise 4, Stealth 4, Scorability 4

### B54 - Require a space in `"version": "`
- Location: hofund-core/.../connection/HofundConnectionResult.java, extractVersionFromResponse() ~69-76
- Core relevance: parsing version from JSON.
- Bug type: input parsing / format assumptions
- Proposed change: set versionKey to "\"version\": \"" (with space).
- Trigger conditions: minified JSON with no spaces.
- Expected symptom: detected_version UNKNOWN for many services.
- Why its hard: depends on JSON formatting style.
- Static-analysis discoverability: Low.
- Suggested detection: unit test with minified JSON.
- Rank (1-5): Exercise 3, Stealth 4, Scorability 4

### B55 - Use lastIndexOf("version") after applicationIndex
- Location: hofund-core/.../connection/HofundConnectionResult.java, extractVersionFromResponse() ~69-75
- Core relevance: detected_version tag accuracy.
- Bug type: correctness (parsing)
- Proposed change: use lastIndexOf(versionKey) from end of body.
- Trigger conditions: JSON with both application.version and build.version.
- Expected symptom: wrong detected_version (build version instead of app).
- Why its hard: values look plausible.
- Static-analysis discoverability: Low.
- Suggested detection: unit test with multiple version keys.
- Rank (1-5): Exercise 4, Stealth 4, Scorability 4

### B56 - Always read errorStream instead of inputStream
- Location: hofund-core/.../connection/HofundConnectionResult.java, parseResponseBody() ~42-53
- Core relevance: detected_version tag accuracy.
- Bug type: error handling
- Proposed change: read urlConn.getErrorStream() unconditionally.
- Trigger conditions: 2xx responses (errorStream null).
- Expected symptom: detected_version UNKNOWN even when UP.
- Why its hard: status OK; only version is wrong.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test with 200 response containing version.
- Rank (1-5): Exercise 3, Stealth 4, Scorability 4

### B57 - Remove try-with-resources in response parsing
- Location: hofund-core/.../connection/HofundConnectionResult.java, parseResponseBody() ~42-56
- Core relevance: repeated scrapes can leak resources.
- Bug type: resource leak
- Proposed change: remove try-with-resources and never close reader.
- Trigger conditions: long-running service with frequent scrapes.
- Expected symptom: gradual resource exhaustion or stalled connections.
- Why its hard: delayed, load-dependent.
- Static-analysis discoverability: Medium.
- Suggested detection: load test with many scrapes and fd leak monitoring.
- Rank (1-5): Exercise 4, Stealth 4, Scorability 3

### B58 - Math.abs on status value
- Location: hofund-core/.../connection/HofundConnectionMeter.java, bindTo() ~31-34
- Core relevance: hofund_connection is primary health signal.
- Bug type: numeric correctness
- Proposed change: return Math.abs(status.getValue()).
- Trigger conditions: INACTIVE status (-1).
- Expected symptom: INACTIVE shows as UP (1).
- Why its hard: looks like a healthy dependency.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test mapping INACTIVE to -1.
- Rank (1-5): Exercise 4, Stealth 3, Scorability 5

### B59 - Reuse tags from first connection for all gauges
- Location: hofund-core/.../connection/HofundConnectionMeter.java, bindTo() ~30-34
- Core relevance: tags identify each connection series.
- Bug type: data integrity
- Proposed change: compute tags once outside the loop and reuse for all gauges.
- Trigger conditions: multiple connections configured.
- Expected symptom: metrics collapse into one series with wrong tags.
- Why its hard: only visible when multiple connections exist.
- Static-analysis discoverability: Medium.
- Suggested detection: integration test with two connections verifying distinct tags.
- Rank (1-5): Exercise 4, Stealth 4, Scorability 4

### B60 - Remove unspecified-version guard in checkVersions
- Location: hofund-core/.../connection/HofundConnectionsTable.java, checkVersions() ~71-75
- Core relevance: startup table used for readiness diagnostics.
- Bug type: error handling
- Proposed change: delete the isUnspecified() early return.
- Trigger conditions: required version N/A or UNKNOWN.
- Expected symptom: compareTo throws, catch block logs DOWN and UNKNOWN.
- Why its hard: appears as intermittent connection failure.
- Static-analysis discoverability: Low.
- Suggested detection: unit test with UNKNOWN required version.
- Rank (1-5): Exercise 4, Stealth 4, Scorability 4

### B61 - Re-run connection check for version
- Location: hofund-core/.../connection/HofundConnectionsTable.java, print() ~39-52
- Core relevance: connection checks run at startup.
- Bug type: reliability/perf
- Proposed change: call connection.getFun().get().getConnection() again inside checkVersions instead of using connectionResult.
- Trigger conditions: slow or flaky dependencies.
- Expected symptom: duplicate network calls; inconsistent status vs version.
- Why its hard: only visible under load or flakiness.
- Static-analysis discoverability: Low/Medium.
- Suggested detection: unit test asserting single connection invocation.
- Rank (1-5): Exercise 4, Stealth 4, Scorability 3

### B62 - Empty version becomes UNKNOWN instead of N/A
- Location: hofund-core/.../connection/Version.java, of() ~14-18
- Core relevance: version semantics used for comparisons/logging.
- Bug type: data semantics
- Proposed change: return UNKNOWN for null/empty instead of NOT_APPLICABLE.
- Trigger conditions: DB connections or missing version.
- Expected symptom: version logs change behavior; comparisons behave differently.
- Why its hard: manifests as subtle changes in logs.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test for Version.of("") == N/A.
- Rank (1-5): Exercise 3, Stealth 3, Scorability 5

### B63 - Missing segments treated as -1 (1.2 < 1.2.0)
- Location: hofund-core/.../connection/Version.java, compareTo() ~52-56
- Core relevance: version ordering drives warnings.
- Bug type: correctness (boundary handling)
- Proposed change: use -1 when part is missing rather than 0.
- Trigger conditions: comparing 1.2 to 1.2.0 or 2.0 to 2.0.1.
- Expected symptom: false "too low" warnings.
- Why its hard: only for uneven version lengths.
- Static-analysis discoverability: Low.
- Suggested detection: unit test for 1.2 == 1.2.0.
- Rank (1-5): Exercise 4, Stealth 4, Scorability 5

### B64 - Parse version part with Integer.parseInt(part)
- Location: hofund-core/.../connection/Version.java, parseInt() ~68-79
- Core relevance: version comparisons are core correctness.
- Bug type: robustness / parsing
- Proposed change: replace numeric prefix parsing with Integer.parseInt(part).
- Trigger conditions: versions with suffixes like "1.0-RC1" or "1-SNAPSHOT".
- Expected symptom: IllegalArgumentException from compareTo; table logs DOWN.
- Why its hard: only affects pre-release versions.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test with "1.0-RC1".
- Rank (1-5): Exercise 4, Stealth 4, Scorability 4

### B65 - Treat UNKNOWN/N/A as greater than any regular version
- Location: hofund-core/.../connection/Version.java, compareTo() ~39-46
- Core relevance: version checks should skip unspecified values.
- Bug type: correctness / error handling
- Proposed change: if unspecified, return 1 instead of throwing or equal.
- Trigger conditions: required version unspecified.
- Expected symptom: version checks never flag mismatches (UNKNOWN wins).
- Why its hard: absence of error looks like success.
- Static-analysis discoverability: Low.
- Suggested detection: unit test where UNKNOWN should not satisfy 1.0.
- Rank (1-5): Exercise 4, Stealth 4, Scorability 4

### B66 - Collision check uses getTarget instead of toTargetTag
- Location: hofund-core/.../graph/node/HofundNodeMeter.java, checkIdCollision() ~44-49
- Core relevance: node IDs must be unique for graph.
- Bug type: data integrity
- Proposed change: compare ids against connection.getTarget() only.
- Trigger conditions: multiple DB connections with same target but different vendor/description.
- Expected symptom: collisions not detected; nodes overwritten.
- Why its hard: only in multi-DB setups.
- Static-analysis discoverability: Low.
- Suggested detection: integration test with two DB vendors same target.
- Rank (1-5): Exercise 4, Stealth 4, Scorability 4

### B67 - Type tag uses enum name (uppercase)
- Location: hofund-core/.../graph/node/HofundNodeMeter.java, tagsForConnection() ~92-93
- Core relevance: type tags drive Grafana node-graph queries.
- Bug type: API contract drift
- Proposed change: Tag.of("type", connection.getType().name()).
- Trigger conditions: dashboards expecting lowercase type.
- Expected symptom: missing nodes in graph filters.
- Why its hard: looks like dashboard misconfig.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test asserting type tag "http"/"database".
- Rank (1-5): Exercise 3, Stealth 4, Scorability 5

### B68 - Subtitle drops type when description present
- Location: hofund-core/.../graph/node/HofundNodeMeter.java, tagsForConnection() ~87-90
- Core relevance: node subtitles are used to differentiate nodes.
- Bug type: data integrity / UI correctness
- Proposed change: use description only, removing type from subtitle.
- Trigger conditions: DB/QUEUE connections with descriptions.
- Expected symptom: nodes become ambiguous; operator confusion.
- Why its hard: visual-only regression.
- Static-analysis discoverability: Low.
- Suggested detection: snapshot test of grafana node-graph tags.
- Rank (1-5): Exercise 2, Stealth 4, Scorability 3

### B69 - Info node id uses application version
- Location: hofund-core/.../graph/node/HofundNodeMeter.java, tagsForInfo() ~73-76
- Core relevance: the main node id ties edges to the app.
- Bug type: correctness / ID mismatch
- Proposed change: Tag.of("id", infoProvider.getApplicationVersion()).
- Trigger conditions: always.
- Expected symptom: edges do not attach to the app node.
- Why its hard: graph breaks silently.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test for id == application name.
- Rank (1-5): Exercise 4, Stealth 4, Scorability 5

### B70 - Edge collision check is case-insensitive
- Location: hofund-core/.../graph/edge/HofundEdgeMeter.java, checkIdCollision() ~50-54
- Core relevance: prevents duplicate edge IDs.
- Bug type: data integrity
- Proposed change: compare ids using toLowerCase() on both sides.
- Trigger conditions: targets that differ only by case.
- Expected symptom: false positives, startup errors in some envs.
- Why its hard: only with case-sensitive naming.
- Static-analysis discoverability: Low.
- Suggested detection: unit test with two connections differing by case.
- Rank (1-5): Exercise 3, Stealth 4, Scorability 4

### B71 - Reuse tags from last connection for all edges
- Location: hofund-core/.../graph/edge/HofundEdgeMeter.java, bindTo() ~41-44
- Core relevance: edge tags determine graph integrity.
- Bug type: data integrity
- Proposed change: compute tags once outside loop and reuse.
- Trigger conditions: multiple connections.
- Expected symptom: all edges share the same tags; missing edges.
- Why its hard: only visible with multiple edges.
- Static-analysis discoverability: Medium.
- Suggested detection: integration test with two edges and distinct tags.
- Rank (1-5): Exercise 4, Stealth 4, Scorability 4

### B72 - Lowercase application name with default locale
- Location: hofund-core/.../info/HofundInfoMeter.java, tags() ~36-38
- Core relevance: app id must match other tags and env names.
- Bug type: locale/timezone sensitivity
- Proposed change: apply toLowerCase() without Locale.ROOT.
- Trigger conditions: Turkish locale, names containing "I".
- Expected symptom: id mismatch in tags and env var naming.
- Why its hard: only in specific locales.
- Static-analysis discoverability: Low.
- Suggested detection: unit test with Turkish locale.
- Rank (1-5): Exercise 4, Stealth 5, Scorability 4

### B73 - application_version tag uses application type
- Location: hofund-core/.../info/HofundInfoMeter.java, tags() ~36-38
- Core relevance: hofund_info labels are fundamental to dashboards.
- Bug type: data integrity
- Proposed change: Tag.of("application_version", provider.getApplicationType()).
- Trigger conditions: any deployment.
- Expected symptom: version label shows "app"/"backend" instead of version.
- Why its hard: values look plausible.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test for application_version tag.
- Rank (1-5): Exercise 3, Stealth 3, Scorability 5

### B74 - Truncate commit_id to 7 chars without length check
- Location: hofund-core/.../git/HofundGitInfoMeter.java, tags() ~36
- Core relevance: git metadata is core to debugging deployments.
- Bug type: robustness
- Proposed change: provider.getCommitId().substring(0, 7) without guard.
- Trigger conditions: short/empty commit_id (local builds).
- Expected symptom: runtime exception; git info metric missing. Likely too easy unless test removed.
- Why its hard: only in non-standard builds.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test with empty commit_id.
- Rank (1-5): Exercise 3, Stealth 2, Scorability 4

### B75 - build_time tag uses build_host
- Location: hofund-core/.../git/HofundGitInfoMeter.java, tags() ~38-40
- Core relevance: build_time used for timeline correlation.
- Bug type: data integrity
- Proposed change: Tag.of("build_time", provider.getBuildHost()).
- Trigger conditions: always.
- Expected symptom: build_time label shows hostnames.
- Why its hard: labels still look like strings; not obviously wrong.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test for build_time tag value.
- Rank (1-5): Exercise 3, Stealth 3, Scorability 5

### B76 - Swap runtime_name and runtime_version tags
- Location: hofund-core/.../java/HofundJavaInfoMeter.java, tags() ~42-44
- Core relevance: Java info supports runtime debugging.
- Bug type: data integrity
- Proposed change: set runtime_name to info.getRuntime().getVersion() and runtime_version to getName().
- Trigger conditions: any deployment.
- Expected symptom: runtime tags inverted; looks plausible.
- Why its hard: values are both strings; not obvious.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test for runtime tag values.
- Rank (1-5): Exercise 3, Stealth 3, Scorability 5

### B77 - Swap jvm_vendor and vendor_name tags
- Location: hofund-core/.../java/HofundJavaInfoMeter.java, tags() ~39-47
- Core relevance: JVM vendor metadata.
- Bug type: data integrity
- Proposed change: use info.getJvm().getVendor() for vendor_name.
- Trigger conditions: any deployment.
- Expected symptom: vendor labels inconsistent across metrics.
- Why its hard: values are similar; may go unnoticed.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test with known vendor values.
- Rank (1-5): Exercise 2, Stealth 3, Scorability 4

### B78 - Remove toLowerCase in OS detection
- Location: hofund-core/.../os/HofundOsInfo.java, getOsFamily/getManufacturer() ~31-74
- Core relevance: OS tags used for fleet diagnostics.
- Bug type: input normalization
- Proposed change: remove osName = osName.toLowerCase().
- Trigger conditions: os.name with uppercase (most systems).
- Expected symptom: OS family falls back to raw name or Unknown manufacturer.
- Why its hard: only visible in labels; no failures.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test with "Linux" and "Windows".
- Rank (1-5): Exercise 3, Stealth 3, Scorability 5

### B79 - Manufacturer derived from os.arch instead of os.name
- Location: hofund-core/.../os/HofundOsInfo.java, get() ~19-28
- Core relevance: OS metadata tags.
- Bug type: data integrity
- Proposed change: pass osArch into getManufacturer().
- Trigger conditions: any deployment.
- Expected symptom: manufacturer becomes "Unknown".
- Why its hard: labels look plausible; no failures.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test expecting "Microsoft" for Windows.
- Rank (1-5): Exercise 2, Stealth 3, Scorability 4

### B80 - Swap name and version tags in web server info
- Location: hofund-core/.../web/HofundWebServerInfoMeter.java, tags() ~36-38
- Core relevance: web server info used for diagnostics.
- Bug type: data integrity
- Proposed change: Tag.of("name", info.getVersion()) and Tag.of("version", info.getName()).
- Trigger conditions: any deployment.
- Expected symptom: name shows "9.0.x" and version shows "Apache Tomcat".
- Why its hard: still plausible strings.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test verifying tag values.
- Rank (1-5): Exercise 3, Stealth 3, Scorability 5

### B81 - Lowercase web server name/version on creation
- Location: hofund-core/.../web/HofundWebServerInfo.java, create() ~20-23
- Core relevance: web server tag identity.
- Bug type: data normalization
- Proposed change: return new HofundWebServerInfo(name.toLowerCase(), version.toLowerCase()).
- Trigger conditions: any deployment.
- Expected symptom: unexpected label changes; case-sensitive dashboards break.
- Why its hard: only visible in label matching.
- Static-analysis discoverability: Low.
- Suggested detection: unit test for name casing.
- Rank (1-5): Exercise 2, Stealth 3, Scorability 4

### B82 - Silently pad/truncate rows in AsciiTable.addRow
- Location: hofund-core/.../AsciiTable.java, addRow() ~18-22
- Core relevance: connection table output at startup.
- Bug type: data integrity / presentation
- Proposed change: if columns length differs, pad/truncate instead of throwing.
- Trigger conditions: callers pass wrong number of columns.
- Expected symptom: misaligned table with shifted columns.
- Why its hard: only visible in logs; no exception to highlight bug.
- Static-analysis discoverability: Low.
- Suggested detection: unit test with incorrect column count.
- Rank (1-5): Exercise 2, Stealth 4, Scorability 3

### B83 - Column widths based on headers only
- Location: hofund-core/.../AsciiTable.java, printTable() ~34-38
- Core relevance: connection table readability.
- Bug type: presentation correctness
- Proposed change: remove loop that updates columnWidths from rows.
- Trigger conditions: any row value longer than header.
- Expected symptom: truncated or misaligned columns in logs.
- Why its hard: only visual; not caught by tests.
- Static-analysis discoverability: Low.
- Suggested detection: snapshot test of table rendering.
- Rank (1-5): Exercise 2, Stealth 4, Scorability 3

### B84 - Last column padded with width-1
- Location: hofund-core/.../AsciiTable.java, printRow() ~58-60
- Core relevance: connection table output.
- Bug type: off-by-one (formatting)
- Proposed change: padRight(row.get(i), columnWidths[i] - 1) for last column.
- Trigger conditions: any row.
- Expected symptom: subtle alignment drift on last column.
- Why its hard: visual-only and easy to miss.
- Static-analysis discoverability: Low.
- Suggested detection: golden file test for table output.
- Rank (1-5): Exercise 2, Stealth 4, Scorability 3

### B85 - emptyIfNull returns literal "null"
- Location: hofund-core/.../StringUtils.java, emptyIfNull() ~5-10
- Core relevance: used in info tags and DB metadata.
- Bug type: data integrity
- Proposed change: return "null" instead of empty string.
- Trigger conditions: any null vendor/version fields.
- Expected symptom: labels contain "null" instead of empty.
- Why its hard: looks like real value; no errors.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test for emptyIfNull(null) == "".
- Rank (1-5): Exercise 3, Stealth 4, Scorability 5

### B86 - isEmpty returns false for null
- Location: hofund-core/.../StringUtils.java, isEmpty() ~13-18
- Core relevance: used in tag and description logic.
- Bug type: correctness / null handling
- Proposed change: return false when description == null.
- Trigger conditions: null description values.
- Expected symptom: null treated as non-empty; tags built with nulls.
- Why its hard: only for nulls; errors show in tags.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test for isEmpty(null) == true.
- Rank (1-5): Exercise 3, Stealth 3, Scorability 5

### B87 - EnvProvider lowercases env var name
- Location: hofund-core/.../EnvProvider.java, SystemEnvProvider.getEnv() ~9-11
- Core relevance: env-based disablement of connection checks.
- Bug type: config parsing
- Proposed change: System.getenv(name.toLowerCase()).
- Trigger conditions: environment variables with uppercase names (standard).
- Expected symptom: disable env vars never match on Linux/Unix.
- Why its hard: looks like env misconfiguration.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test with mock EnvProvider.
- Rank (1-5): Exercise 4, Stealth 4, Scorability 4

### B88 - RequestHeader.of swaps name and value
- Location: hofund-core/.../connection/RequestHeader.java, of() ~12-14
- Core relevance: HTTP checks rely on headers for auth.
- Bug type: correctness
- Proposed change: new RequestHeader(value, name).
- Trigger conditions: any custom headers.
- Expected symptom: auth failures, but only when headers are used.
- Why its hard: only headered connections fail; looks like auth issue.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test verifying header name/value on connection.
- Rank (1-5): Exercise 4, Stealth 4, Scorability 4

### B89 - RequestMethod.HEAD returns "GET"
- Location: hofund-core/.../connection/RequestMethod.java, enum constant ~7-9
- Core relevance: HTTP checks for HEAD should not fetch body.
- Bug type: correctness / API drift
- Proposed change: set HEAD("GET").
- Trigger conditions: when users configure HEAD.
- Expected symptom: servers may reject GET or respond differently.
- Why its hard: only on HEAD usage; subtle performance impact.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test asserting HEAD maps to "HEAD".
- Rank (1-5): Exercise 3, Stealth 3, Scorability 4

### B90 - Type.toString returns uppercase name()
- Location: hofund-core/.../connection/Type.java, toString() ~17-19
- Core relevance: type tag used widely in queries.
- Bug type: API contract drift
- Proposed change: return name() instead of lowercase name.
- Trigger conditions: any deployment with dashboards expecting lowercase.
- Expected symptom: missing metrics in dashboards due to label mismatch.
- Why its hard: looks like missing data rather than code bug.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test for type tag values.
- Rank (1-5): Exercise 4, Stealth 4, Scorability 5

# Top 10 recommended set (from B41-B90)
- B44 - Stale cached URL; core health checks ignore dynamic config changes.
- B47 - Cached tags lead to stale detected_version across the system.
- B53 - Response truncation causes intermittent UNKNOWN versions.
- B56 - errorStream-only parsing hides version on healthy responses.
- B58 - Math.abs status hides INACTIVE; subtle alerting break.
- B60 - compareTo exception path masks real issues as DOWN.
- B64 - parseInt throws on pre-release versions; hard to detect.
- B66 - Node collision check misses duplicates; graph integrity issues.
- B70 - Edge collision case-insensitive; rare and confusing failures.
- B72 - Locale-sensitive lowercasing causes ID mismatch only in some locales.

# Instances (I01-I05)
Top 40 selected from B41-B90 (excluding: B74, B75, B77, B79, B81, B82, B83, B84, B86, B89).
Each instance mixes correctness, parsing, config, and data-integrity issues.

- I01: B41, B44, B49, B53, B58, B63, B66, B87
- I02: B42, B45, B47, B51, B54, B59, B68, B85
- I03: B43, B46, B50, B55, B60, B64, B69, B80
- I04: B48, B52, B56, B61, B65, B70, B72, B90
- I05: B57, B62, B67, B71, B73, B76, B78, B88
