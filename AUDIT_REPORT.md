# NagramXF Canary audit and hardening report

**Target base:** `canary`  
**Working branch:** `audit-fixes/canary-2026-10-19`  
**Audit status:** Source-level review in progress; **not release-validated**.  
**Merge status:** Not merged.

## Scope and method

Reviewed the current `canary` baseline and the patch branch for targeted correctness, lifecycle, remote-data parsing, file-import, native loading, installer fallback, and authentication-token handling. The changes below are based on source inspection. This environment could not clone the repository for a local checkout and has no Android SDK/Gradle installation, so no local compile, unit test, instrumentation test, or device smoke test has been run. GitHub reported no status checks/workflow runs for the branch head at the time of review.

## Fixes committed on this branch

- **ReplyMessageLine:** correct sticker drawable detach lifecycle; use the selected palette for collectible gradient colors; clear stale state when the palette is empty.
- **PagePreviewRulesHelper:** correct cache TTL units; preserve in-flight loading state; clear loading on error/null response; handle malformed cached payloads and invalid regex patterns without crashing.
- **BaseRemoteHelper:** route resolution and unexpected-response failures through error callbacks instead of silently leaving callers waiting.
- **UpdateHelper:** handle absent ABI selection without unboxing a null value and only attach valid document IDs.
- **ApplicationLoader:** report APK installer success only after the launch call succeeds, preserving fallback behavior after launch failures.
- **NativeLoader:** allow native-library fallback attempts to continue after a failed load.
- **StickersActivity:** stream content URI imports with resource cleanup and enforce the 3 MiB cap during copying; remove partial files after failure.
- **OpenChatReceiver / OpenAttachedMenuBotReceiver:** finish activity on invalid input and caught failures.
- **FileRefController:** avoid skipping common reference-update handling when recovering a bot profile media reference.
- **NekoMessageCell:** reset interrupted press/long-press state on detach.
- **AuthTokensHelper:** remove a debug log that emitted `future_auth_token` bytes as UTF-8 text; remove the now-unused import.

## Remaining risks / decisions

### High priority — token backup policy needs an explicit product/security decision
`BackupAgent` backs up the `saved_tokens` and `saved_tokens_login` SharedPreferences. `AuthTokensHelper` stores serialized `TL_auth_authorization` objects in the latter and requests Android backup after saving them. This may be intentional for the app's token-recovery feature, but it means authentication-related material is included in the Android backup path. Confirm the intended threat model and Android backup/data-extraction rules before changing this behavior; no backup policy change was made in this patch.

### Medium priority — cleartext traffic is allowed globally
The manifest sets `android:usesCleartextTraffic="true"`, and `network_security_config.xml` permits cleartext in the base config. The manifest comment says this supports opening `http://` links. Prefer handling external HTTP links in a browser/intent without granting the entire app cleartext access, but this requires a compatibility audit of internal HTTP clients and link handling. No behavior change was made here.

### Medium priority — imported database/archive size is not globally bounded
The database import/ZIP extraction path identified during review does not enforce an aggregate uncompressed-size cap. The existing entry-name filtering helps with path traversal, but very large inputs could still exhaust storage or memory. Establish a limit based on realistic backup sizes and enforce both per-entry and total extracted bytes before shipping. No limit was added because a safe product limit needs evidence about legitimate user backup sizes.

### Low priority — reaction counter rendering order
A possible draw-order issue was observed where alpha may be assigned after a drawable's draw call. This has not been reproduced, and it should be confirmed with a focused visual/runtime test before changing rendering behavior.

## Verification checklist

- [ ] Run the repository's supported Canary Gradle build with its documented Java/Android SDK/NDK versions.
- [ ] Run available tests and inspect lint/compiler output.
- [ ] Smoke-test: reply line/sticker detach and dark collectible gradient; cached preview rules (fresh, stale, corrupt, malformed regex); remote-helper error callbacks; update metadata without ABI/document; APK install failure fallback; native load fallback; 3 MiB sticker import boundary and copy failure cleanup; deep-link receiver invalid inputs; bot media reference recovery; message-cell detach during long press.
- [ ] Decide whether token preferences should be included in Android backup and whether cleartext policy can be narrowed.
- [ ] Add an aggregate database import/extraction size cap after choosing a compatible threshold.
- [ ] Re-check GitHub Actions on the final PR head before merge.

## Recommendation

Keep this branch isolated until build and targeted regression checks pass. This report is not a claim of a complete exhaustive review of every file or a release-ready validation.


## Cross-check against the upstream Canary baseline

I also checked the exact `canary` base commit (`385dc54a0f60682311a423f6f7eeee0b3efad9ab`) rather than assuming a similar upstream implementation had already fixed these defects. The source at that baseline still contains the following patterns that this branch changes:

- `ReplyMessageLine`: the sticker's detach callback calls `attach()`; collectible gradient values are read from `p.colors` even when a dark palette was selected.
- `PagePreviewRulesHelper`: `onError` is empty; the TTL is `15 * 60` while timestamps use milliseconds; the loading guard is reset immediately after starting asynchronous loading; cached payload decoding is not fully guarded.
- `BaseRemoteHelper`: several remote-channel resolution and response-type failure paths do not invoke the error delegate.
- `AuthTokensHelper`: debug logging prints `future_auth_token` content. This branch removes that log.

These were verified against the exact base commit's source, so the changes are targeted corrections rather than assuming an upstream commit already contains them. The repository's `canary` branch is the PR target; no merge or release is performed by this audit.
