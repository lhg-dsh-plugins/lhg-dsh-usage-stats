# v0.3.7 release notes

This release fixes OpenCode Go quota windows whose real usage is at most 1% being reported as fully exhausted.

## Changes

- OpenCode Go: `goWindowFromObject` no longer rescales percent-named values in (0, 1] as if they were 0..1 ratios. The dashboard's `usagePercent` is already a 0..100 percent (the OpenCode console computes `floor(min(100, usage / limit * 100))`), and the Bearer endpoint's percent fields are 0..100 as well, so a window at 1% real usage now stays at 1% used / 99% remaining instead of jumping to 100% used / 0% remaining. This affects both the dashboard (cookie) path and the API path when the usage object names the field `usagePercent`. Windows above 1% were already reported verbatim and are unchanged.

## Validation and compatibility

- New regression coverage in `scripts/test-subscriptions.mjs`: dashboard-shaped and API-shaped payloads with rolling 1% / weekly 0% / monthly 2% now normalize to exactly those values; against the pre-fix code the same cases fail with the session window at 100% used.
- All offline test suites pass with the fix; no other adapter, window kind, or wire shape changes.

Cache schema, credentials, and account monitor configuration are untouched. Costs remain estimates; unknown pricing and incomplete or mixed-currency amounts remain unknown rather than being guessed.

## Scope and safety

Only the OpenCode Go window normalization changes. Missing services, credentials, or unrecognized responses keep explicit unavailable/error states.
