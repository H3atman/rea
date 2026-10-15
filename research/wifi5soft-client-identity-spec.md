# WiFi5-Soft client identity: architecture and verification

## Scope and current status

- Firmware scope: **24.10.4**, the version supplied for this investigation; no running router version was verified.
- Selected artifact: `/home/rgsv/w5s/root/soft/index.o`.
- SHA-256: `4d2cc3cc0e4930dea932b6916ae2bd8d62c77b259c55ed03e250710f1ec4bdfd` (locally checked).
- This is a static architecture interpretation and a future verification specification, not a runtime report.
- Existing dumps and frontend analysis are investigation inputs; no test receipts or runtime artifacts described below exist yet.
- **All proposed cases are not-run. No original-runtime identity or accounting verification has been performed.**
- Work performed here is documentation and read-only source inspection only; no target execution, installation, or test implementation.

## Authority and reading conventions

| Label                    | Meaning                                                                                           |
| ------------------------ | ------------------------------------------------------------------------------------------------- |
| **observed static**      | Instruction, constant, or structure present in the inspected dump/bundle; not observed execution. |
| **derived**              | Interpretation or reconstructed flow from those static observations; not original source.         |
| **unknown**              | Available evidence does not establish the behavior or relationship.                               |
| **proposed requirement** | Desired property or future test procedure, not a firmware fact.                                   |

The [source index](#source-index) gives absolute filenames and exact **file-line** ranges, not bytecode offsets or original-source lines.
The `.dec.txt` tables and frontend literal view are derived pseudo-source; the V8 dumps are static bytecode/constant-pool evidence.
Dummy source strings in the V8 dump are not recovered original JavaScript and must not be presented as such.
Existing dump verification metadata concerns dump structure only, not runtime correctness [H0].

Future case oracles use three separate labels; a desired property must never silently become a source claim:

- **E — source expectation:** behavior suggested by a cited static path, conditional on reaching that path.
- **C — characterization:** measure an unknown outcome without inventing the expected policy.
- **D — desired acceptance:** explicitly proposed product/safety property; failure does not contradict static evidence by itself.
  A receipt records each E/C/D oracle separately; one successful identity check cannot satisfy an accounting oracle.

## Identity glossary

| Term                            | Meaning and boundary                                                                                                                         |
| ------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| Current MAC                     | Network address returned for this request by `dhcp.getmac(IP)`; may change with MAC randomization.                                           |
| IP                              | Request lookup input, selected from `x-real-ip` or `socket.remoteAddress`; not a durable browser identifier [H1].                            |
| DHCP lease                      | Parsed lease-file record with its own expiration and `id`; not automatically the cookie identity [D2].                                       |
| Lookup expiration               | `Date.now() + 43200000` returned by `getmac`; neither lease expiration nor paid-time balance [D1].                                           |
| MAC-selected record             | `uniqueCode[MAC]`, selected by the current network MAC; selection and identity value are different concepts [H5].                            |
| Record `.id`                    | Identity value assigned from successful cookie decoding in the inspected callback; entitlement meaning is unknown [H5].                      |
| Cookie name / `global.idKey`    | Request namespace: starts with `global.prefix`, overridden by truthy `global.idKey`; `cookieSig` is a binding, not a literal wire name [H1]. |
| Cookie ciphertext               | Browser-carried encrypted representation passed to the decoding helper; availability depends on the browser context [H3].                    |
| Fixed AES passphrase            | Derived role of `88932009YYE`: HTTP supplies it as the first helper argument; helpers forward `(a1, a0)` to AES [H3, H4, S1].                |
| Socket `auth.id`                | Frontend `random(8)` value at socket/module initialization; do not substitute it for record `.id` [F1, F2].                                  |
| Entitlement/accounting identity | Owner of paid time, usage, charges, or authorization in the runtime ledger; linkage to `.id` remains unknown.                                |

## Static architecture

### Request and lookup boundary

1. **observed static:** the handler selects the cookie-name binding from `global.prefix`, then truthy `global.idKey` [H1].
2. **observed static:** it selects `x-real-ip`, with `socket.remoteAddress` fallback, and invokes `dhcp.getmac` [H1].
3. **derived:** proxy-header provenance matters; merely possessing a valid cookie does not establish the request's MAC.
4. **derived:** `getmac` requires both `variables.ip_list[IP]` and `variables.cIfname[MAC]`; either missing map entry returns null [D1].
5. **derived:** a successful lookup returns `mac`, `ip`, `ifname`, and a lookup expiration twelve hours after lookup time [D1].
6. **observed static:** the handler checks lookup expiration against `Date.now()` and requires a usable MAC before continuing [H1].
7. **observed static:** the `timeok` gate precedes cookie identity processing; the failure response names time updating or no internet [H1].
8. **derived:** a valid cookie alone cannot bypass the inspected MAC-resolution/time gates. Other handlers are outside this claim.

**unknown:** trust validation for `x-real-ip`, effective proxy topology, and map freshness/update ownership.
Original lease/getmac dump anchors accompany the derived interpretation [D0].
The exported `ip_list` and `cIfname` begin as empty objects [D3]; initialization alone does not establish subsequent population.
A separate lease parser prefers `/mnt/wifi5/dhcp.leases`, otherwise `/usr/dhcp.leases`, if present [D2].
It parses timestamp/expiration, MAC, IP, host, and lease `id`; router config names `/mnt/wifi5/dhcp.leases` [D2, D4].
**unknown:** the edge from that parser to the lookup maps. A lease file entry alone is not proof of successful `getmac` resolution.

### Cookie decoding, association, and custom remapping

- **observed static:** HTTP builds `88932009` plus `YYE` as the first `CSTRdecrypt` argument and supplies the cookie value second [H3].
- **derived:** the helper forwards `AES.decrypt(a1, a0)`, renders UTF-8, and supplies null on its caught failure path [S1].
- **observed static:** a truthy decoded callback argument can become `.id` of the existing MAC-selected record [H5].
- **observed static:** the absent-record branch constructs a MAC-keyed record with decoded `.id`, lowercased MAC, and current date [H6].
- **observed static:** metadata includes `info` and `userAgent`; records are serialized with `JSON.stringify` and passed to `fs.writeFile` [H6, H7].
- **derived:** reusing a decoded identity on a newly selected MAC is a possible association path, conditional on the gates and branch.
- **unknown:** whether this reassociation migrates, restores, or preserves entitlement/accounting state.

**Critical correction:** assigning the same cookie-derived `.id` to a new MAC record does **not** prove preservation of remaining paid time.
It also does not prove uninterrupted authorization, correct accounting attribution, or prevention of duplicate spending.
Case T07 therefore has independent ledger, elapsed-time, accounting, and authorization checks, not just `.id` equality.

A custom-remap path clears its custom flag, assigns `newID`, deletes a property, and initiates a serialized record write [H2].
A continuation supplies `newID` to a closure [H8]. These are scoped custom-path observations, not a general allocation algorithm.
The setter also contains existing-record and `random2(8)` encryption branches [H4].
**unknown:** complete persistent-ID allocation/fallback, collision handling, and write-failure handling across all branches.
Duplicate-ID collection logic is visible [H9], but a collection or log string does not establish enforcement policy.
**unknown:** whether copied cookies coexist, transfer ownership, reject a client, merge records, or duplicate entitlement consumption.

### Encryption boundary

- **derived:** the system module loads `crypto-js`; `CSTRencrypt(a0,a1,a2)` forwards `AES.encrypt(a1,a0)` [S1].
- **derived:** in the inspected HTTP calls, identity/ciphertext is the data argument and `88932009YYE` is the passphrase argument [H3, H4, S1].
- The recovered parameter order is evidence; calling `global.idKey` an AES key would conflate unrelated roles.
- **unknown:** effective crypto parameters and security properties of the deployed helper/library combination.
- **proposed requirement:** future tests must not treat decryptability as proof of authentication, device ownership, or authorization.

## Configuration, persistence, and cookie scope

| Static input or setting  | Scoped interpretation                                                                                                    | Authority/source                |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------ | ------------------------------- |
| `/mnt/wifi5/idbase-rule` | Initial `idbase=false`; exact contents `false` keep false; other successful contents set true; read exception sets true. | derived [I1]                    |
| `/mnt/wifi5/idbase-key`  | Truthy read sets `global.idKey` and `/mnt/wifi5/client-cookie/<value>/` as `cookiepath`.                                 | derived [I2]                    |
| Empty/key read failure   | No fallback assignment is shown in the inspected key-read slice.                                                         | observed static [I2]            |
| Default `cookiepath`     | Separate default literal `/mnt/wifi5/uniquecID/`; do not attribute it to the key-read catch.                             | observed static [G1]            |
| MAC record write         | Selected `cookiepath` plus normalized MAC filename; JSON serialization and asynchronous write invocation.                | derived [H6, H7]                |
| Cookie `maxAge`          | `259200000000` milliseconds, nominally 3000 days; not observed effective browser lifetime.                               | observed static / derived [H10] |
| Cookie `httpOnly`        | `true`.                                                                                                                  | observed static [H10]           |
| Cookie `sameSite`        | `strict`.                                                                                                                | observed static [H10]           |
| Cookie `path`            | `/client`.                                                                                                               | observed static [H10]           |
| Cookie clearing          | A `clearCookie` operation and cookie-name binding occur in the setter area.                                              | observed static [H11]           |

Original config names/paths are visible in the constant pool [I0].
Key contents are compared literally; do not trim `false\n` to `false` when describing the rule read [I1].
A namespace change K1 → K2 changes the observed cookie selector and persistence-directory selector, not demonstrated AES-key rotation.
**unknown:** branch-dependent effects of `idbase`, path validation, namespace migration, and behavior with unavailable directories.
**unknown:** successful write completion, crash durability, startup reload, and reconciliation of files with other state stores.
The cookie-option pool does not prove browser acceptance, effective expiry, host/domain scope, transport security, or clear-cookie effectiveness.
Do not infer uninspected cookie attributes or other handlers from this table.

## Browser and captive-portal boundary

The original frontend bundle is minified on line 1 [F1]; the retained literal view is a derived representation [F2].
The derived view shows socket `auth.id=random(8)`, disconnect/connect listeners, and conditional reload on reconnect [F2].
The disconnect callback sets the flag read by the connect callback's conditional reload [F2]; this is derived flow, not verified runtime lifecycle.
Inspected theme/media storage use is scoped frontend evidence, not an inventory proving all durable identity mechanisms absent.
**unknown:** socket-server association with cookie records and whether other scripts, workers, or stores participate.

**proposed requirement:** record origin, host, route, browser/container, and cookie acceptance/send outcomes for each future case.
Normal browsers, captive assistants, embedded WebViews, private sessions, and different profiles are separate contexts until measured.
Do not assume a cookie jar is shared between a captive assistant and a normal browser, or between apps on one device.
Exercise `/client` requests explicitly; compare other routes without assuming the cookie is sent there.
`HttpOnly` means page JavaScript is not an appropriate cookie-read oracle; use approved browser/network observations in memory.
A socket ID may change on module reinitialization, but random collisions are possible: do not demand inequality on every reload.
Record socket initialization/lifecycle separately from persistent-record and ledger observations.

## Future verification setup — not executed

Every item below is a **proposed requirement**, subject to operator approval before any action.
Use an isolated, disposable router/environment with a matching artifact digest, synthetic clients, and no production users or payments.
The operator must approve network isolation, supported configuration changes, grant creation, restart, and cleanup procedures.
Do not install tools, change router configuration, or execute any scenario merely to prepare this document.
Use existing approved tooling; unavailable facilities produce a blocked receipt rather than implicit installation or privilege escalation.
Use a supported synthetic grant mechanism; do not invent entitlement state by editing unknown ledger fields.
Prepare two controllable clients, fresh DHCP outcomes, and explicit browser/captive profiles; keep the current MAC separate from a client label.
An observation point must independently expose sanitized identity records, ledger ownership, balance, accounting, and authorization.
If ledger observability is unavailable, T07 is blocked even if association T02 can be characterized.
Before any writes, document backups/reset expectations and a supported restoration or disposal plan.
An interrupted run records cancellation and partial observations; failed cleanup leaves explicit ownership and a recovery plan.

### Shared evidence and clock rules

- Assign a fresh run ID and synthetic client labels A/B; correlate MAC/IP/cookie/record observations only within that run.
- Record config branch outcomes and sanitized namespace labels K1/K2 without storing namespace credential values.
- Confirm actual DHCP/map lookup outcome, not only a lease request or file appearance.
- Record router wall-clock and observer monotonic offsets, clock-sync state, request intervals, and balance units/resolution.
- Define clock tolerance **before** execution from measured clock offset, timestamp granularity, and bounded observation latency.
- Freeze neither the router clock nor expected consumption without a supported, separately approved method.
- Repeat deterministic cases at least three independent times; concurrency cases need three fresh paired trials with a start barrier.
- Report actual repeat count and each trial; interrupted/blocked trials do not count as passes.
- Capture before/action/after and failure/teardown outcomes, including whether writes and authorization checks completed.
- Do not claim architecture-wide equivalence from a finite scenario matrix.

## Case matrix

All cases below currently have status **not-run**. Artifacts are proposed sanitized receipts, not existing outputs.
Each case inherits the shared evidence, sensitivity, clock, cancellation, and teardown contracts.

### P0: request gates, association, and entitlement continuity

**T01 — Same MAC, valid cookie**

- Preconditions: known lookup maps, `timeok=true`, established record and cookie in one browser context.
- Actions: repeat `/client` request without changing MAC/context; compare before/after record and cookie-send outcome.
- E: the decoding callback can assign the decoded ID to the selected record [H1, H3, H5]; C: actual selected branch and response.
- Artifacts: ordered requests, resolved network tuple, decoded-ID pseudonym, record projection, response/Set-Cookie projection.

**T02 — Rotated MAC, same browser cookie**

- Preconditions: T01-like baseline; controllable MAC change, fresh lookup resolution, unchanged cookie jar.
- Actions: change MAC using approved means, complete DHCP resolution, request `/client`, inspect both old/new record projections.
- E: truthy decoded ID can populate the newly selected record [H6]; C: old-record disposition and persistence result.
- D: association is explainable without an accidental new identity; no claim about remaining time is allowed from this case alone.
- Artifacts: before/after MAC aliases, map/lease outcome, cookie-send result, both record projections and write outcome.

**T03 — Missing cookie on same versus new MAC**

- Preconditions: independent fresh profiles; one existing MAC record and one new resolved MAC.
- Actions: issue requests with cookie absent; run the two conditions independently, without clearing unrelated browser state.
- C: allocation/fallback, response, record reuse/creation, and new cookie issuance; setter branches alone are not the full allocator [H4].
- D: no silent transfer of another synthetic client's entitlement; artifacts include cookie absence and identity/ledger deltas.

**T04 — Malformed or corrupted cookie**

- Preconditions: resolved MAC and `timeok=true`; approved malformed ciphertext variants with synthetic data only.
- Actions: send truncated, corrupted, and non-decodable values; inspect response, mutation, and cookie-clear/set outcomes.
- E: caught decrypt failure supplies null [S1]; C: malformed values that decode without throwing and handler continuation.
- D: no unrelated entitlement grant or ledger corruption; artifact records variant labels, errors, and state deltas, not raw values.

**T05 — Valid cookie with failed MAC lookup**

- Preconditions: valid cookie; independently induce missing IP-map entry and missing MAC/interface-map entry on disposable setup.
- Actions: issue `/client` requests; capture actual lookup failure and resulting response/state.
- E: either missing map entry can return null and the handler's MAC gate blocks this path [D1, H1].
- C: error response and state side effects; artifacts distinguish missing mappings from missing lease assumptions.

**T06 — `timeok=false` control**

- Preconditions: supported isolated control of the gate; otherwise mark blocked, not simulated firmware verification.
- Actions: hold cookie/MAC valid, request with gate false, restore approved time state, repeat as positive control.
- E: the inspected gate returns its time/internet error before cookie processing [H1]; artifacts include gate state and record deltas.

**T07 — Paid-time and accounting continuity (independent of T02)**

- Preconditions: supported synthetic grant with known ledger owner/balance; independent ledger/accounting and authorization probes.
- Actions: record baseline; consume a controlled interval; rotate MAC with same cookie and fresh lookup; observe before/after ledger.
- Then disconnect/reconnect, consume a second interval, and inspect accounting attribution and authorization again.
- C: discover actual entitlement owner and charging model; no static source establishes that `.id` is their durable key.
- D: remaining balance follows baseline minus independently expected consumption within predeclared tolerance, without reset or free grant.
- D: authorization follows the grant; attribution remains with the intended synthetic entitlement, with no lost usage or duplicate charge.
- D: no simultaneous double spending; a copied-cookie concurrent check is separately exercised in T11.
- Artifacts: ledger owner aliases, balance/unit/time samples, ordered usage records, grant reference alias, authorization outcomes.
- `.id` equality is only association evidence. Missing balance/ledger/authorization coverage makes this case blocked, not passed.

### P1: configuration, persistence, concurrency, and browser boundaries

**T08 — Rule-file branches**

- Preconditions/actions: separately exercise exact `false`, another value (including newline variant), and approved read failure.
- E: static read/exception interpretation from [I1]; C: effect on request/remap behavior, not merely config parsing.
- Artifacts: branch labels, observed gate/branch state, record/ledger deltas; restore or dispose approved configuration.

**T09 — Namespace K1 → K2 → K1**

- Preconditions/actions: approved namespace configurations and independently observable cookie/persistence selectors.
- E: truthy key read selects name/directory [I2]; C: restart/loading requirements, fallback, migration, and return-to-K1 behavior.
- D: no unexpected entitlement merge across namespaces; artifacts use namespace labels and sanitized path components.
- Include empty/read-failure characterization separately; do not call this crypto-key rotation or claim the catch sets a default.

**T10 — Custom remap**

- Preconditions/actions: supported custom-remap trigger, known starting record; inspect new/old records and write completion.
- E: scoped `newID` assignment/deletion/write/continuation [H2, H8]; C: all linked ledger and old-record effects.
- D: intended ownership and accounting remain explainable; artifacts distinguish this custom path from a general allocator.

**T11 — Copied cookie on two concurrent clients**

- Preconditions: synthetic cookie, independently resolved MACs A/B, ledger/authorization observation and agreed safety policy.
- Actions: copy cookie in memory using approved tooling; synchronize requests/usage at a barrier; repeat with reversed ordering.
- C: coexistence/transfer/rejection policy and old/new record outcomes; collection logic [H9] does not decide the oracle.
- D: no duplicate entitlement spending or unauthorized balance increase; report deviations separately from characterization.
- Artifacts: barrier/order trace, two record projections, shared entitlement/accounting deltas, both authorization outcomes.

**T12 — Disposable-target restart persistence**

- Preconditions/actions: approved restart and supported observation; wait for confirmed write completion, restart, reconnect and inspect.
- C: restoration and reconciliation; `fs.writeFile` invocation is not durability/startup-load proof [H6].
- D: acknowledged identity/grant state survives a supported clean restart; artifacts include shutdown/startup and ledger checks.
- Abrupt interruption/crash is a separate approved trial, not implied by clean-restart success; record cleanup uncertainty.

**T13 — Android/iOS browser versus captive assistant**

- Preconditions/actions: real supported OS/browser versions, fresh explicit profiles; test each context and transitions independently.
- C: cookie acceptance/send, jar sharing/isolation, route/host changes, reopen behavior, and MAC-randomization interaction.
- D: supported-context identity/entitlement behavior meets declared product policy; unsupported contexts are labeled, never assumed.
- Artifacts: OS/browser/container/version labels, sanitized navigation/cookie outcomes, record/ledger observations.

**T14 — Socket lifecycle versus record identity**

- Preconditions/actions: one established record; reload, disconnect/reconnect, and change browser context in separate trials.
- E: derived socket initialization/reload mechanism [F2]; C: server linkage and transport-ID lifecycle.
- D: transport recreation alone does not reset entitlement; artifacts separate socket-ID, record-ID, and ledger-owner aliases.
- Random-ID collisions do not fail a strict-inequality oracle: verify initialization/flow, not guaranteed unique random output.

## Future receipt and artifact contract

Each run produces a machine-readable sanitized receipt plus a short human-readable case view; neither exists yet.
Required receipt fields:

- Run/case/trial IDs, scenario version, operator approval reference, firmware label and measured target SHA-256.
- Environment isolation, client/browser/OS/container versions, context/origin/route labels, supported/unsupported boundaries.
- E/C/D oracle definitions, cited source ranges, actual observations, comparison policy, per-oracle outcome and limitations.
- Authority: `controlled-replay` only for actual original-target runs; simulated fixtures must be labeled as such and cannot prove those runs.
- Overall status: `pass`, `fail`, `characterized`, `blocked`, or `not-run`; repeat target/count and complete per-trial outcomes.
- `pass`: all required E/D checks met with complete observations; `fail`: measured required-oracle violation.
- `characterized`: C outcomes measured, without claiming desired acceptance; `blocked`: missing setup/authority/observability or interruption.
- `not-run`: no execution; never replace this with a fabricated empty successful trace.
- Ordered action/request/barrier trace, DHCP and map outcome, sanitized before/after identity and ledger/accounting projections.
- Clock/units/tolerance policy and measured offsets, authorization probes, grant alias and independent consumption calculation.
- Errors, malformed/cancellation outcomes, missing sections, write completion, teardown/restoration status and retained cleanup owners.
- Sanitizer version/policy, artifact filenames and SHA-256 digests of **sanitized** persisted bytes only.
  Finite receipts prove only their declared cases, artifact, environment, and measured authority; no global absence or equivalence claims.

## Sensitivity and pre-persistence sanitization

This investigation explicitly designates **raw cookies, device identifiers, MACs, and namespace/config credentials as sensitive** for future capture.
This is a caller-selected capture policy, not REA's default inference from names: the repository default does not infer secrecy by name/pattern.
Apply it before any persistence, including logs, temporary files, screenshots, exception text, profiles, and tool transcript output.

- Capture only necessary projections in memory; replace sensitive values with stable **per-run** pseudonyms before serialization.
- Use a fresh in-memory mapping or keyed pseudonym scheme each run; determinism is within a run, never across runs.
- Preserve equality/difference within a semantic domain (e.g., decoded cookie ID versus record ID), not accidental cross-domain matches.
- Sanitize IPs, user-agent/device details, paths containing identifiers, and other linkable metadata under the declared capture policy too.
- Persist sanitized field presence, types, branch outcomes, and correlation aliases; redact credential-bearing header/query values.
- Do not persist raw HAR, browser profiles, Cookie/Set-Cookie headers, or decrypted identifiers and then sanitize afterward.
- Retain neither the raw-to-alias map nor pseudonym secrets; known low-entropy identifiers must not become unsalted persistent hashes.
- Verify sanitizer leakage handling before router tests using synthetic canaries; unsafe collectors block capture rather than write raw data.
- Persist only sanitized artifact hashes; do not use raw sensitive-value digests as replacement identifiers.
- If safe in-memory browser observation is unavailable, record that cookie/ledger oracle as blocked.

## Source index

These anchors refer to existing local investigation inputs; they are not generated by the proposed tests.
`H1`–`H11` are original dump file lines; `I/D/S/F2` pseudo-source interpretations are explicitly derived.

- **H0**: `/home/rgsv/w5s/decomp/httpserver.full.d8.verification.json`: lines 1–18, 41–44 (existing dump structure metadata).
- **H1**: `/home/rgsv/w5s/decomp/httpserver.full.d8.txt`: lines 17291–17368, 17713–17729 (namespace, IP, lookup and gates; pool labels).
- **H2**: `/home/rgsv/w5s/decomp/httpserver.full.d8.txt`: lines 17455–17458, 17501–17568 (custom flag/remap/write path).
- **H3**: `/home/rgsv/w5s/decomp/httpserver.full.d8.txt`: lines 17574–17587, 18277–18279 (decrypt argument order and literals).
- **H4**: `/home/rgsv/w5s/decomp/httpserver.full.d8.txt`: lines 17777–17843, 17850–17852 (setter encryption branches and literals).
- **H5**: `/home/rgsv/w5s/decomp/httpserver.full.d8.txt`: lines 18428–18538 (truthy decode, existing record and metadata).
- **H6**: `/home/rgsv/w5s/decomp/httpserver.full.d8.txt`: lines 18662–18752 (absent record, metadata, serialization and write invocation).
- **H7**: `/home/rgsv/w5s/decomp/httpserver.full.d8.txt`: lines 18979–19009, 19101–19135 (record/write names and object boilerplates).
- **H8**: `/home/rgsv/w5s/decomp/httpserver.full.d8.txt`: lines 18250–18262 (`newID` continuation).
- **H9**: `/home/rgsv/w5s/decomp/httpserver.full.d8.txt`: lines 18776–18828 (duplicate-ID collection, not a verified enforcement policy).
- **H10**: `/home/rgsv/w5s/decomp/httpserver.full.d8.txt`: lines 17910–17931, 18000–18021 (cookie options).
- **H11**: `/home/rgsv/w5s/decomp/httpserver.full.d8.txt`: lines 18201–18203 (`clearCookie` and binding names).
- **I0**: `/home/rgsv/w5s/decomp/clean/index.clean.txt`: lines 4479–4515 (original config constant-pool names/paths; behavior interpretation uses I1/I2).
- **I1**: `/home/rgsv/w5s/decomp/lib/index.dec.txt`: lines 61–106 (derived rule-file read/compare/catch).
- **I2**: `/home/rgsv/w5s/decomp/lib/index.dec.txt`: lines 108–148 (derived namespace-file read and path assignment).
- **G1**: `/home/rgsv/w5s/decomp/clean/global-path.clean.txt`: lines 64–65 (separate default `cookiepath` pool literals).
- **D0**: `/home/rgsv/w5s/decomp/clean/dhcp.clean.txt`: lines 368–400, 428–543 (original lease constants and getmac dump; derived flow uses D1/D2).
- **D1**: `/home/rgsv/w5s/decomp/lib/dhcp.dec.txt`: lines 50–118 (derived map lookup and lookup-expiration return).
- **D2**: `/home/rgsv/w5s/decomp/lib/dhcp.dec.txt`: lines 120–169, 202–269 (derived lease paths and parsed record).
- **D3**: `/home/rgsv/w5s/decomp/lib/variables.dec.txt`: lines 7–24 (derived exported empty maps).
- **D4**: `/home/rgsv/w5s/root/etc/config/dhcp`: lines 15–26 (router lease-file configuration).
- **S1**: `/home/rgsv/w5s/decomp/lib/system.dec.txt`: lines 50–54, 520–583 (derived crypto module and helper argument forwarding).
- **F1**: `/home/rgsv/w5s/public_out/public/client/newTheme/static/js/main.34905dd2.js`: line 1 (original minified frontend input).
- **F2**: `/home/rgsv/w5s/rea-out/main.literal-view.txt`: line 2 (retained derived socket/options/lifecycle and scoped theme storage view).
