# Source Distribution Notice Gate

An Intelligent Contract that turns a publisher's declared license obligations and distribution artifacts into a small, consensus-checked release decision.

Public repository: https://github.com/an8442780-debug/source-distribution-notice-gate

## Live Deployment

Studio Dev deployment completed after the exact-revision PRE-DEPLOY approval.

- Network: `studio-dev` / Chain ID `61997`
- Deployer/publisher address: `0xef469ea73e17e629523e08ceaba67df0b1759845`
- Contract: `0x9C0Cfd624B4e406353279C473c86407964B4FF5e`
- Contract Explorer: https://explorer-studio-dev.genlayer.com/address/0x9C0Cfd624B4e406353279C473c86407964B4FF5e
- Deployment transaction: `0x4fcd5530cc8d49f9d3fcb53ac342c594daccb4f837a17aa7d028cfed55439db2` ([Explorer](https://explorer-studio-dev.genlayer.com/tx/0x4fcd5530cc8d49f9d3fcb53ac342c594daccb4f837a17aa7d028cfed55439db2))
- Deployment receipt: finalized, consensus `MAJORITY_AGREE`, execution accepted
- Deployed contract source revision: `sha256:e1e91649e85d6e268073a0865d3e3aa536a3aed7c33af3c1fd5c9cbda0659e4e`
- Live E2E release `sdng-live-1`: register → seal → submit evidence → assess all finalized/accepted; authoritative readback is `ASSESSED` / `RELEASE_ENABLED`, obligation mask `2`, observed artifact mask `6`.
- Live E2E write transactions: `0xa0012be5b867551c70e41f24beea04917bff4592a3600ad06c47d56a68f00e1c`, `0x1cfe1e874377c2fb642065c1bb84f6cb4614fe9b19992393ed7333b40b26405a`, `0xad63eade3353690d3c6d44334aa132278883118ed8edb5d75b2d5d94a8583d52`, `0xec0408b5a3f57c56431c04f9176a6c339eeb7cc9ec634d49821a524932b14b70` (inputs in `samples/sdng-live-1.json`).
- Negative authorization transaction: `0x4af127c678fc3c3b27e408419e4b335a8a12890e3927618561f3ccd81c3cbdab` ([Explorer](https://explorer-studio-dev.genlayer.com/tx/0x4af127c678fc3c3b27e408419e4b335a8a12890e3927618561f3ccd81c3cbdab)). It finalized with consensus `MAJORITY_AGREE` but leader execution `FINISHED_WITH_ERROR` / `UNAUTHORIZED_PUBLISHER`; readback remains `NOT_FOUND` for the attempted release and `RELEASE_ENABLED` for `sdng-live-1` (input in `samples/sdng-unauthorized.json`).

### Public E2E Evidence

| Scenario | Public input | Result | Evidence |
|---|---|---|---|
| Authorized release assessment | [`samples/sdng-live-1.json`](samples/sdng-live-1.json) | Finalized consensus; `ASSESSED` / `RELEASE_ENABLED`; obligation mask `2`, observed artifact mask `6` | [Assess transaction](https://explorer-studio-dev.genlayer.com/tx/0xec0408b5a3f57c56431c04f9176a6c339eeb7cc9ec634d49821a524932b14b70) |
| Unauthorized publisher write | [`samples/sdng-unauthorized.json`](samples/sdng-unauthorized.json) | Finalized consensus with leader `FINISHED_WITH_ERROR` / `UNAUTHORIZED_PUBLISHER`; attempted release remains `NOT_FOUND` | [Authorization transaction](https://explorer-studio-dev.genlayer.com/tx/0x4af127c678fc3c3b27e408419e4b335a8a12890e3927618561f3ccd81c3cbdab) |

## Problem and GenLayer fit

Release tooling often declares a distribution mode while omitting a notice or source-offer artifact. The contract records the package/version, canonical license hashes, bounded obligation mask, and expected artifact mask, then uses validator consensus to interpret untrusted license text. Deterministic code derives `RELEASE_ENABLED`, `HOLD_NOTICE_SET`, or `LICENSE_SCOPE_UNCLEAR` from the agreed obligation mask and submitted evidence.

This is not legal certification. A conventional backend is enough when the license obligations are already machine-readable and trusted; GenLayer is useful when independent validators must interpret bounded, hostile license text before a release gate changes state.

## How it works

1. The publisher registers a package, version, distribution mode, license hashes, obligation mask, and expected artifact mask.
2. The publisher seals the release and submits exact observed version/mode/artifact evidence.
3. Validators independently assess the untrusted license text and must agree on the integer obligation mask.
4. Deterministic code compares the agreed mask with the publisher declaration and checks missing artifacts.

## State model and invariants

`REGISTERED -> SEALED -> EVIDENCE_SUBMITTED -> ASSESSED`

Only the deployer/publisher may write. Release IDs are single-use; exact replays are no-ops. Modes are limited to `BINARY`, `SOURCE_AVAILABLE`, and `DUAL`. Masks are limited to `ATTRIBUTION=1`, `NOTICE=2`, and `SOURCE_OFFER=4`. Version and mode mismatches fail closed, and observed artifact masks must satisfy the selected mode. Prompt-instruction markers are rejected before any oracle call. Corrections can reopen only an assessed non-enabled release; correction replays are idempotent.

## Public API and oracle surface

Writes: `register_release`, `seal_release`, `submit_evidence`, `assess_release`, and `correct_evidence`.

Views: `read_status`, `read_result`, `read_obligation_mask`, `read_artifact_mask`, `read_release`, and `read_publisher`. `read_release` returns deterministic JSON with the package, canonical hash array, masks, observed evidence and digest. Integrators can use `read_result(release_id)` as the deterministic release-gate oracle and `read_status` to avoid consuming a result before assessment.

## Consensus Binding Matrix

| Field | Source | Stored? | Downstream effect | Validator check | Binding mode | Differential test |
|---|---|---:|---|---|---|---|
| `obligation_mask` | independent license-text assessment | yes | enabled/hold/unclear | independent second assessment, exact integer equality | strict integer equality | validator disagreement leaves no state transition |
| `observed_artifact_mask` | publisher evidence | yes | missing-obligation hold | mode invariant plus deterministic cross-field check against agreed and expected masks | exact stored mask | missing notice/attribution and incomplete-mode tests |
| `package_version` / observed version | publisher declaration/evidence | yes | scope validity | deterministic exact equality | exact binding | wrong-version test |
| `mode` / observed mode | fixed enum and evidence | yes | distribution semantics | deterministic exact equality and mode-mask rules | exact binding | wrong-mode rejection |
| `license_hashes` | publisher declaration | yes | identifies assessed license set | canonicalized, bounded, duplicate-free | exact canonical bytes | duplicate/invalid hash rejection |
| `result` | deterministic derivation | yes | release gate decision | derived only from bound fields | deterministic derivation | mask/artifact differential cases |

## Security and failure behavior

License text is untrusted data delimited in the prompt; embedded instructions are ignored by the prompt and known instruction markers fail closed before the oracle call. Malformed oracle output, validator disagreement, or a non-final consensus result must not be treated as a release approval. Missing required artifacts produce `HOLD_NOTICE_SET`; license-scope disagreement, version/mode mismatch, or injection produces `LICENSE_SCOPE_UNCLEAR`.

## Tests

```text
$tool = "E:\\Genlayer-Tools\\studio-next-toolchain"
$env:GENVM_VERSION = "v0.6.0-rc5"
& "$tool\\.venv\\Scripts\\genvm-lint.exe" check contracts/source_distribution_notice_gate.py
& "$tool\\.venv\\Scripts\\gltest.exe" tests/ -v
```

The direct suite covers registration/sealing/submission replay, missing notices, source-offer success, wrong versions, prompt injection, correction, and consensus-bound integer decisions. Studio Dev deployment and public evidence are recorded above and in `samples/`.

The pinned Windows RC runner may hit a temporary-file cleanup `WinError 32` before test execution; this repository does not claim that behavior as a plain-run PASS. The contract source is independently linted and compiled, and the deployed Studio Dev evidence above is authoritative for the live revision.

## Consensus Engineering Lessons

- Use integer masks for consequential obligations; do not store free-form model text as a decision.
- Re-run the same bounded assessment in validators and compare the exact decision field.
- Derive the release status deterministically after consensus; never let explanation prose mutate state.
- Treat license text and artifact evidence as hostile input with size and marker limits.
- Keep missing evidence as a visible hold state instead of silently enabling a release.

## Reusable integrations

- CI/CD can call `read_result` before publishing a package.
- A registry can store the returned masks and display the exact release decision without trusting an off-chain service.
- An audit workflow can use `read_release` to bind a package/version to canonical license hashes and evidence digest.

## Limitations

The contract recognizes only three bounded obligation bits and three fixed distribution modes. It does not certify legal compliance, fetch repositories, inspect binary contents, or guarantee that a publisher's evidence is truthful beyond the recorded masks and consensus assessment.

## Repository structure

```text
contracts/source_distribution_notice_gate.py
tests/test_source_distribution_notice_gate.py
samples/sdng-live-1.json
samples/sdng-unauthorized.json
README.md
requirements.txt
.gitignore
```

## License

MIT
