# Additional Production Scenarios

## Scenario 31 — Bean initialization performs remote I/O

**Symptoms:** Startup is slow or fails when a dependency is temporarily unavailable.

**Investigation:** Inspect constructors, @PostConstruct methods, runners, and bean initialization traces for network calls.

**Resolution:** Keep bean construction cheap. Move optional loading to a bounded, observable startup phase or a controlled background process, depending on readiness requirements.

**Trade-offs:** Deferring work improves startup but can move failures into live traffic. Make readiness reflect actual ability to serve.

**Prevention:** Keep constructors side-effect-light and add startup tests that reveal accidental network calls.

**Follow-up:** What work belongs in a constructor?

## Scenario 32 — Auto-configuration unexpectedly disappears

**Symptoms:** A default client or converter bean is missing after adding a dependency or custom bean.

**Investigation:** Inspect condition evaluation, classpath, properties, and user bean definitions.

**Resolution:** Identify which condition failed or which custom bean caused back-off; configure the replacement intentionally.

**Prevention:** Test auto-configuration both with and without user overrides.

**Follow-up:** Why is @ConditionalOnMissingBean useful for starters?

## Scenario 33 — Property value differs from YAML

**Symptoms:** A timeout or endpoint differs from the checked-in file.

**Investigation:** Inspect active profiles, environment variables, command-line arguments, imported configuration, and deployment injection.

**Resolution:** Correct the authoritative source and document precedence; never log secret values to debug the issue.

**Prevention:** Validate configuration at startup and add deployment checks.

**Follow-up:** Why is a profile not a secret store?

## Scenario 34 — Bean overriding hides a wiring defect

**Symptoms:** Application starts but uses an unexpected implementation.

**Investigation:** Inspect bean names, conditions, imports, and override settings.

**Resolution:** Give implementations explicit names/qualifiers and remove accidental duplicate definitions.

**Prevention:** Test the selected bean and avoid relying on overriding as configuration strategy.

**Follow-up:** How does @Primary differ from bean overriding?

## Scenario 35 — AOP advice runs twice

**Symptoms:** Duplicate audit records or duplicate timing measurements.

**Investigation:** Check overlapping pointcuts, nested proxies, duplicate aspects, and whether the same operation is instrumented at multiple layers.

**Resolution:** Narrow pointcuts and define ordering/ownership of instrumentation.

**Prevention:** Test one representative method and inspect proxy configuration.

**Follow-up:** Why can broad pointcuts be dangerous?

## Scenario 36 — Final method bypasses class proxy

**Symptoms:** Transaction/cache advice does not run on one method.

**Investigation:** Determine proxy type, target class finality, method visibility, and call path.

**Resolution:** Make the interception boundary proxy-compatible or use a suitable interface-based design.

**Prevention:** Avoid relying on proxy interception for methods that cannot be intercepted.

**Follow-up:** What constraints differ between JDK and subclass proxies?

## Scenario 37 — API serializes internal fields

**Symptoms:** A new entity field appears in JSON unexpectedly.

**Investigation:** Check whether controllers return persistence entities, serialization annotations, mixins, and DTO mappings.

**Resolution:** Return explicit response DTOs and define the API contract.

**Prevention:** Add serialization contract tests and review API changes.

**Follow-up:** Why can DTO separation reduce lazy-loading issues?

## Scenario 38 — Content type mismatch

**Symptoms:** Clients receive 406 or 415 responses.

**Investigation:** Check Accept and Content-Type headers, message converters, request mapping consumes/produces constraints, and payload format.

**Resolution:** Align the API contract and supported media types; do not disable validation blindly.

**Prevention:** Add contract tests for supported and unsupported media types.

**Follow-up:** What is the difference between 406 and 415?

## Scenario 39 — Offset pagination becomes slow

**Symptoms:** Deep pages take longer as the dataset grows.

**Investigation:** Inspect generated SQL, query plan, sort indexes, and offset depth.

**Resolution:** Consider keyset/cursor pagination with a stable ordering key and suitable index.

**Trade-offs:** Cursor APIs complicate arbitrary page jumps and require clear consistency semantics.

**Prevention:** Establish query-plan and latency thresholds for deep-page requests, and document the cursor's stable sort key.

**Follow-up:** Why must cursor sorting be deterministic?

## Scenario 40 — Entity graph creates huge joins

**Symptoms:** Query returns duplicate rows or consumes excessive memory after eager fetching.

**Investigation:** Inspect joins, collection cardinality, generated SQL, and result-set size.

**Resolution:** Fetch only the needed shape, use projections or staged queries where appropriate, and measure alternatives.

**Prevention:** Avoid blanket eager loading.

**Follow-up:** Why doesn't eager loading universally solve N+1?

## Scenario 41 — Read-only transaction still writes

**Symptoms:** Code changes data despite readOnly transaction metadata.

**Investigation:** Check transaction manager, database, driver, provider, and whether the call was intercepted.

**Resolution:** Enforce permissions/constraints at the database where writes must be prohibited; treat readOnly as a hint/optimization unless the platform guarantees more.

**Prevention:** Test the real database behavior.

**Follow-up:** Why is readOnly not an authorization mechanism?

## Scenario 42 — Transaction spans remote call

**Symptoms:** Database connections remain occupied while a downstream service responds slowly.

**Investigation:** Trace transaction duration and identify remote I/O inside the transaction.

**Resolution:** Shorten the database transaction and model remote coordination explicitly, often using outbox/saga patterns where required.

**Trade-offs:** Moving the call changes consistency semantics; define recovery and compensation.

**Prevention:** Instrument transaction duration and review code paths that perform remote I/O while a database transaction is active.

**Follow-up:** Why are distributed transactions not implied by @Transactional?

## Scenario 43 — Test passes only when run alone

**Symptoms:** Test suite order changes results.

**Investigation:** Look for shared static state, mutable singleton test beans, database residue, reused ports, async tasks, and context cache interactions.

**Resolution:** Isolate fixtures and clean persistent state; make asynchronous work observable and bounded.

**Prevention:** Run tests repeatedly and in parallel where supported.

**Follow-up:** What is Spring test context caching intended to optimize?

## Scenario 44 — @SpringBootTest suite is too slow

**Symptoms:** CI spends most time booting contexts.

**Investigation:** Group tests by context configuration, identify unique property overrides, and inspect unnecessary full-context usage.

**Resolution:** Use unit tests and slices where sufficient; reuse context configuration intentionally.

**Trade-offs:** A faster test pyramid still needs some full-context integration coverage.

**Prevention:** Track CI duration by test class and require justification for adding unique context configurations or full-context tests.

**Follow-up:** What can invalidate context cache reuse?

## Scenario 45 — Test mocks no longer replace beans after upgrade

**Symptoms:** A test loads the real collaborator or fails to compile.

**Investigation:** Verify Boot/Framework version, mock annotation package, test slice inclusion, and context bean name/type.

**Resolution:** Use the mock-bean API supported by the selected version and migrate imports deliberately.

**Prevention:** Include dependency compatibility checks in upgrade reviews.

**Follow-up:** Why should version-specific test APIs be documented?
