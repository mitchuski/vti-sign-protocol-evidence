# Runtime evidence: `keys/sign-jws/0.1` and `keys/sign-csr/0.1` in the VTA

This bundle answers trustoverip/dtgwg-trust-tasks-tf#752. That issue placed runtime evidence for the two tasks in the implementing repository. It is also shaped for the Scope C cases (C01–C10) in sankarshanmukhopadhyay/rahp-toolkit#994 (`docs/review/dtg-runtime-evidence/cases.yaml`, `scope-c-vti-2026-10-18-rebaseline.md`).

All tests were run locally on 2026-10-29 by an agent working in Claude Code. Nothing was deployed. Every "trace" below comes from an in-process VTA, built by the repository's own test fixture (`build_signing_test_app_state_with_sink`) and driven through its real Trust Task dispatch spine (`dispatch_trust_task_core`). That spine runs §7.2 validation, proof verification, the ACL and capability gates, the handler, the audit sink and response signing. It is runtime evidence for the code at the pinned commit. It is not a trace of any production deployment.

## Pinned sources

| Item | Value |
|---|---|
| Implementation | OpenVTC/verifiable-trust-infrastructure `origin/main` = `61c23f2091aab03d07271c142eabc0ba74dfe889` (2026-10-29), detached worktree. It contains #2057, merge `dda501dddf48540a3f4b1c36bc6ee917754f4c9b` (`git merge-base --is-ancestor` checked). |
| Files of #2057 changed since `dda501dd` | Yes, by later refactors and dependency bumps. The commit tested is `61c23f20`, not `dda501dd`. |
| trust-tasks-rs | 0.36.1, from VTI `Cargo.lock` |
| Specifications | trustoverip/dtgwg-trust-tasks-tf `specs/keys/sign-{jws,csr}/0.1/` — the blobs are identical on `main` (`484ad5f4`) and at tag `trust-tasks-rs-v0.36.1` (`adf09475`). Copies are in `specs/`. Blob ids: sign-jws spec.md `50ca798e`, schema `b5a11f18`, invalid-examples `8b75fcdc`; sign-csr spec.md `656cf5e6`, schema `0f8ddf0c`, invalid-examples `a47c730d`. |
| Toolchain | cargo 1.98.1 (797e8a9bc 2026-08-05), rustc 1.98.1 (48a229cea 2026-09-01), x86_64-pc-windows-msvc, Windows 11 |

### Build environment notes (not code changes)

- `OPENSSL_DIR=C:\Users\mitch\vcpkg\installed\x64-windows-static-md`, `OPENSSL_STATIC=1`. `webauthn-rs` pulls in `openssl-sys`, which finds no OpenSSL on Windows by default.
- `MSVC_SPECTRE_LIB_DIR=<MSVC 14.29>\lib\x64`. `regorus` 0.13 depends on `msvc_spectre_libs` with `features = ["error"]`, and the Spectre-mitigated MSVC libraries are not installed on this machine. With this variable the build links the ordinary MSVC libraries. That changes binary hardening only, not behaviour under test.
- `NO_COLOR=1 CLICOLOR=0 TERM=dumb CARGO_TERM_COLOR=never CARGO_INCREMENTAL=0`. The logs contain no ANSI escapes. `Compiling`/`Blocking` progress lines were filtered out of the test logs.

## What was run

| # | Command (cwd) | Log | Result |
|---|---|---|---|
| 00 | `cargo test -p vta-service --lib --no-run` (VTI worktree) | `logs/00-build-vta-service-tests.log` | built (2 pre-existing warnings, not in touched files) |
| 10 | `cargo test -p trust-tasks-rs --no-default-features --features keys,validate --test conformance -- keys::sign_jws keys::sign_csr` (dtgwg-trust-tasks-tf @ `adf09475`) | `logs/10-trust-tasks-rs-conformance-sign-jws-csr.log` | **8 passed, 0 failed** |
| 20 | `cargo test -p vta-service --lib -- operations::sign_formats::tests operations::keys::tests::sign_jws operations::keys::tests::sign_csr operations::keys::tests::protocol_signing trust_tasks::conformance::every_published_dispatched_uri_has_a_witness trust_tasks::conformance::every_witnessed_task_round_trips_through_its_generated_types` (VTI, unmodified tests from #2057) | `logs/20-vta-service-upstream-tests.log` | **12 passed, 0 failed** |
| 21 | `cargo test -p vti-common --lib -- acl::tests::sign_jws_and_sign_csr_follow_sign_and_narrow_alone`; `cargo test -p vta-sdk --lib -- signing_domain_tests retry_safety` | `logs/21-vti-common-vta-sdk-upstream-tests.log` | **1 + 13 passed, 0 failed** |
| 30 | `VTI_EVIDENCE_DIR=<bundle>/traces cargo test -p vta-service --lib -- trust_tasks::keys::sign_protocol_tests --test-threads 1` (VTI + the uncommitted tests below) | `logs/30-vta-service-new-wire-tests.log` | **18 passed, 0 failed** |

Row 10 needs `--features validate`. Without it, cargo compiles out the generated `rejects_invalid_examples` and `rejects_invalid_response_examples` tests and runs only the 4 round-trip tests. That was the first run, which was superseded.

## New tests (candidate upstream contribution, uncommitted)

The tests from #2057 work at the operation layer (`operations::keys::sign_jws`/`sign_csr`, `sign_formats`). None of them drives `handle_sign_jws` or `handle_sign_csr`, so none asserts what reaches the wire: the `#response` shape, the `trust-task-error` code and `details`, and the absence of a JWS or CSR in a refusal. The specifications' invalid examples had also never been sent to the VTA itself.

`patch/vti-sign-jws-csr-wire-tests.patch` (against `61c23f20`, test-only) adds:

- `vta-service/src/trust_tasks/sign_protocol_tests.rs` — 18 tests, mounted from `trust_tasks/keys.rs` with `#[cfg(test)] #[path] mod sign_protocol_tests;`
- `vta-service/src/trust_tasks/sign_protocol_fixtures/sign-{jws,csr}-0.1.payload.invalid-examples.json` — verbatim copies of the spec files, same blobs

The trace writer runs only when `VTI_EVIDENCE_DIR` is set.

## Fixtures and traces

`traces/<case>.{request,response,audit}.json` holds one file of each kind per exchange: 69 exchanges plus one summary (`C07-invalid-examples-summary`, request file `null`), 210 files in all.

- **request**: the signed Trust Task document sent.
- **response**: the VTA's signed answer.
- **audit**: the `keys.sign*` rows the exchange added. An empty array means nothing was signed.

Redaction: none was needed. No private key appears anywhere. The request proofs come from test-only Ed25519 seeds `[0x71;32]` … `[0x75;32]` and `[0x7A;32]` (the fixture super-admin). The VTA's keys come from the fixture's throwaway seed in a temporary directory that was deleted after the run. Every DID is a `did:key` of those test keys. Payload values (nonces, subjects) are the specifications' own example values. The `C04`/`C06` traces deliberately show requests the VTA signs.

## Requirement → test → result

`U` = an existing #2057 test (logs 20/21). `N` = a new wire test (log 30). `T` = generated trust-tasks-rs conformance (log 10). Every test listed passed.

### keys/sign-jws/0.1

| Requirement (spec) | Accepted path | Rejected path | Result |
|---|---|---|---|
| C1 validate per §7.2: schema, `additionalProperties: false`, exactly one of `payload`/`payloadBase64Url` | N `sign_jws_spec_request_is_answered…` | N `the_specifications_invalid_examples_are_refused_by_the_vta`: all 9 request examples get `malformedRequest`, nothing signed (`C07-sign-jws-invalid-example-*`). T `rejects_invalid_examples` | PASS |
| C2 permission: `permissionDenied`; `sign-jws` required; `sign` not accepted (framing custodian, SHOULD NOT) | N C01 (holder of `sign-jws` only) | N `sign_jws_without_entitlement…` (`sign` only, `sign-csr` only, Reader role). U `protocol_signing_takes_its_own_capability_only`. U `sign_jws_and_sign_csr_follow_sign_and_narrow_alone` | PASS |
| C3 `keys:notFound` (unknown key) | — | N `sign_jws_key_refusals…`: super-admin gets `keys:notFound`. **To a non-admin producer an unknown key is `permissionDenied`** (N `an_unknown_or_out_of_scope_key…`, see Findings) | PASS / observation |
| C3 `failedPrecondition` (status not active) | — | N revoked → `keys/sign-jws:failedPrecondition`. U `sign_jws_declared_refusals` | PASS |
| C4 `alg` from key type; `keyTypeUnsupported` with `details.keyType` | N P-256 → `ES256` (C01), Ed25519 → `EdDSA`. U `sign_jws_signs_a_jws_that_verifies…` | N x25519 and mldsa44 → `keys/sign-jws:keyTypeUnsupported`, `details.keyType` set | PASS (this VTA signs JWS with p256 and ed25519 only. ML-DSA is refused as unimplemented, which the spec allows: "or the custodian does not implement") |
| C5 forbidden `alg crit b64 jwk jku x5u` → `headerMemberForbidden`, `details.member` | N extra non-forbidden members (`typ`, `kid`) accepted. U `forbidden_and_mistyped…` (`profile_version` ok) | N `sign_jws_forbidden_header_members_are_refused_on_the_wire`: each of the 6 gives a `trust-task-error`, `details.member`, `retryable:false`, no JWS, no audit row (`C02-*`) | PASS |
| C6 mistyped registered member / foreign `x5c` → `keys:invalidArgument` | U `sign_jws_declared_refusals` (an `x5c` certifying the named key is signed) | N `typ`, `kid`, `x5t`, non-certificate `x5c`. U wrong-key `x5c` | PASS |
| C7 build signing input itself (RFC 8785), no framing; `payloadInvalid` | N C01: the returned first two segments are **byte-identical to the spec example**, SHA-256 `e0c236e3…031257`. N `payloadBase64Url` carried unchanged. U `the_spec_example_signing_input_is_reproduced` | N non-canonical `payloadBase64Url` and integer > 2^53 → `keys/sign-jws:payloadInvalid` | PASS |
| C8 return `#response` {keyId, alg, jws}, never key material | N C01: `#response`, `threadId`, VTA proof, the payload validates against the response schema, JWS verifies under the VTA key (and not the spec's key) | T `rejects_invalid_response_examples` (incl. `privateKey`). N all 4 response examples fail the schema the VTA answers against | PASS |
| Audit SHOULD: producer, key, alg, typ, signing-input digest; MUST NOT payload | N C01 audit row: `keys.sign-jws`, actor = producer DID, resource = key, `detail = alg=ES256 typ=oauth-authz-req+jwt sha256=e0c236e3…`, no nonce. U `protocol_signing_audits_the_digest_not_the_payload` | N every refusal leaves no `keys.sign*` row | PASS |
| Correlation SHOULD: bind key to protocol; MAY: restrict `typ` (C04) | N `sign_jws_purpose_restrictions_are_not_enforced_here`: one key signs two `typ`s and a CSR | — | **NOT ENFORCED (observed)** |
| C08: a `keys/sign` signature is not a JWS | — | N `a_keys_sign_signature_does_not_verify_as_a_jws`: a `sign` holder's signature over the exact JWS signing input fails `verify_compact` and verifies only over `vti.vta.opaque-signing.v1‖0x00‖input`. U `signing_domain_tests` | PASS |
| Spec example itself | N `sign_jws_spec_example_verifies` (the spec JWS verifies under the stated key; digest matches) | — | PASS |

### keys/sign-csr/0.1

| Requirement (spec) | Accepted path | Rejected path | Result |
|---|---|---|---|
| C1 validate per §7.2 (C07: no member for an extension, DER or DN string) | N C05 | N all 14 request invalid examples → `malformedRequest`, nothing signed, incl. `extensions` (#08) and `certificationRequestInfo` (#07). T `rejects_invalid_examples` | PASS |
| C2 permission; `sign-csr` required; `sign` not accepted | N C05 | N `sign_csr_without_entitlement…` (`sign` only, `sign-jws` only, Reader). U `protocol_signing_takes_its_own_capability_only` | PASS |
| C3 `keys:notFound` / `failedPrecondition` | — | N super-admin gets `keys:notFound`; revoked → `keys/sign-csr:failedPrecondition`. Non-admin unknown key → `permissionDenied` (as for JWS) | PASS / observation |
| C4 only `p256`, `ed25519`; else `keyTypeUnsupported` | N P-256 (C05), Ed25519 (`sign-csr-ed25519-accepted`). U both | N x25519 and mldsa44 → `keys/sign-csr:keyTypeUnsupported` with `details.keyType` | PASS |
| C5 subject table, control chars, no repair → `subjectInvalid` {member, index, reason} | N C05 subject | N `sign_csr_subject_rules_are_refused_on_the_wire`: C lower-case, 3-letter C, CN of 65 chars, C0 and C1 control characters, `serialNumber` outside PrintableString, `emailAddress` without `@` — each `details.member=subject`, `index=1`. U `subject_rules_refuse_without_repairing` | PASS |
| C5 `subjectAltNames` rules | N C05 (dns, uri) | Enforced by the schema only (wildcard, `ip`, empty object, URI with a space → `malformedRequest` via invalid examples #09–#12). The VTA adds no SAN check of its own beyond the schema; the spec defines none beyond the schema | PASS (schema) |
| C6 build request itself: the key's own SPKI, encodings, only the SAN extension | N `sign_csr_spec_example_info_is_reproduced`: the VTA's `CertificationRequestInfo` for the spec request and spec key is **byte-identical** to the spec example's, and the spec signature verifies over it (so PrintableString/UTF8String choices and SAN order match). N C05: the returned CSR's info equals the build for the VTA key; exactly 1 requested extension. U `sign_csr_builds_a_request_a_ca_accepts` | N/T C07 | PASS |
| C7 PEM (64-col, LF) + `publicKey`, never key material | N C05: PEM armour, ≤64 columns, no CR, `publicKey` = key record, the response validates against the schema, `x509-parser` verifies the signature | T `rejects_invalid_response_examples`. N all 3 response examples fail the schema | PASS |
| Audit SHOULD: subject, SANs, digest of DER info | N C05: `keys.sign-csr`, `detail = subject=… san=dns:…,uri:… sha256=<sha256 of info>` | N refusals leave no row | PASS |
| MAY restrict subject (C06) | N `sign_csr_does_not_judge_whose_name_the_subject_is`: a well-formed subject naming another organisation is signed | — | **NOT RESTRICTED (observed)**. Form is validated; authority over the name is the CA's job per spec |
| SHOULD issue CSRs only for keys bound to a certificate protocol | N C04 trace `C04-sign-csr-same-key-accepted` | — | **NOT ENFORCED (observed)** |

### RAHP Scope C case map

| Case | Traces | Disposition from this run |
|---|---|---|
| C01-valid-sign-jws | `C01-sign-jws-accepted` | PASS (reproduced at runtime; signing-input digest = spec's) |
| C02-sign-jws-prohibited-header | `C02-sign-jws-header-{alg,crit,b64,jwk,jku,x5u}-refused` | FAIL_CLOSED as expected |
| C03-sign-jws-replay | — | not run (needs a consumer freshness/replay contract; out of task scope) |
| C04-sign-jws-wrong-purpose | `C04-*` | NOT ENFORCED (observed; the SHOULD/MAY controls are absent) |
| C05-valid-sign-csr | `C05-sign-csr-accepted`, `sign-csr-ed25519-accepted` | PASS |
| C06-sign-csr-wrong-subject | `C06-sign-csr-other-organisation-accepted` + `sign-csr-subjectInvalid-*` | form refused; authority not judged (out of task guarantee) |
| C07-sign-csr-arbitrary-extension | `C07-sign-csr-invalid-example-08-refused` (+ all invalid examples) | FAIL_CLOSED at the VTA (`malformedRequest`) |
| C08-generic-sign-substitution | `C08-keys-sign-over-jws-signing-input` + `C09-*-sign-only-refused` | FAIL_CLOSED |
| C09-requester-no-authority | `C09-*` | FAIL_CLOSED for: capability missing, wrong capability, read-only role, unknown key, key in another context. Not covered: revoked or expired ACL entry, authority freshness |
| C10-audit-reconstruction | `*.audit.json` | partial: emitted rows shown (fields, no payload for JWS; subject/SAN for CSR). Storage, retention and access are not examined |

## Findings

1. **No wire-level test existed** for either handler at the pinned commit. The declared codes, `details`, the `trust-task-error` vs `#response` split and the schema rejections were asserted only one layer down, or in trust-tasks-rs. The 18 tests in `patch/` close that.
2. **`keys:notFound` reaches only a super-admin.** A producer that is not a super-admin gets `permissionDenied` both for an unknown `keyId` and for a key in a context it cannot act in. This is deliberate in the code (`load_record_in_caller_scope`: an absent key and an out-of-scope key are the same refusal). It is defensible under Conformance item 2, which runs first and covers "the key is one the producer may name". Read literally, item 3 says otherwise, so the spec could state the ordering.
3. **C04 and the sign-csr key binding SHOULD are not enforced.** #2057 states this, and the traces show it.
4. **ML-DSA keys are refused for JWS** as `keyTypeUnsupported`. The spec permits that, but it is a capability limit worth stating.
5. All refusals observed are `retryable:false`. None leaves a signing audit row; refusals are not audited as signing events.

## Integrity

`SHA256SUMS` lists every file in this bundle (`sha256sum`, sorted by path) except three:

- `SHA256SUMS` itself;
- `README.md` and `DRAFT-752-comment.md`, which state the root. Covering them would make the root depend on itself.

The bundle root is the SHA-256 of the `SHA256SUMS` file. Its lines are sorted by path, LF line endings.

    root = b28163aeb6a55bd49972dddf9eb048e951fd90a15e9a3d1fa7c681a46f3511ea

Check it with `sha256sum -c SHA256SUMS && sha256sum SHA256SUMS`.
