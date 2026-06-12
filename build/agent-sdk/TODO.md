# TODO — follow-up on runtime change before this PR ships

This branch (`tyler/agent-sdk-build`, PR #321012) currently writes
`{version, url, sha256}` per platform into `product.agentSdks.<sdk>`.
That shape is going away.

## Why

1. **macOS Universal** ships arm64 + x64 binaries inside one bundle
   sharing one `product.json`. A per-platform `url`/`sha256` cannot be
   correct for both halves at once — the wrong-arch tarball would
   download or sha-verify fail.
2. **Sha256 in product.json is belt-and-suspenders we don't need.**
   product.json's own integrity is covered by `product.checksums`
   inside the signed app bundle. The URL is HTTPS to a
   Microsoft-controlled CDN. The sha only guards "trusted URL string
   but tampered bytes from edge" — that requires breaking TLS or
   Microsoft's edge, a harder attack than tampering with product.json
   itself (which `product.checksums` already guards).

## New shape

```ts
interface IAgentSdkProductConfig {
  readonly version: string;
  readonly urlTemplate: string; // format2 template, {sdkTarget} placeholder
}
```

Resolved at runtime via `getCurrentSdkTarget(sdk)` (mirrors build-side
`getSdkTargetForBuild`), which handles per-SDK musl variance (claude on
Alpine → `linux-*-musl`, codex on Alpine → `linux-*`).

## What this branch needs to change once runtime PR lands

1. `build/agent-sdk/produce.ts`:
   - `IAgentSdkResults` per-SDK entry: `{version, urlTemplate}` instead
     of `{version, url, sha256}`.
   - Drop the `sha256` from the produced JSON. (Keep computing it
     in `buildOne` — `upload.ts` still uses it for blob `metadata.sha256`
     to keep HEAD-then-skip pipeline idempotency.)
   - All platform jobs now emit the SAME `{version, urlTemplate}` per
     SDK — no per-target variance. The split-by-job design still
     stands because each job still UPLOADS its own platform's tarball;
     only the runtime metadata is shared.

2. `build/agent-sdk/common.ts`:
   - New `buildCdnUrlTemplate(sdk, sdkVersion)` returning the template
     string `https://main.vscode-cdn.net/agent-sdk/<sdk>/<version>/{sdkTarget}.tgz`.
   - Keep `buildCdnUrl()` for upload.ts logging? Probably not needed.
   - `IAgentSdkResults` entry shape change.

3. `build/agent-sdk/upload.ts`:
   - Unchanged. Still uploads tarballs, still writes `metadata.sha256`,
     still HEAD-then-skip.

4. README — explain the urlTemplate model + why no sha.

## Runtime PR (separate branch off main)

Tracking work:
- `IAgentSdkProductConfig` → `{version, urlTemplate}`
- Add runtime `getCurrentSdkTarget(sdk)` that mirrors build-side logic
- Substitute `{sdkTarget}` via `format2()` to get download URL
- Remove sha256 verification + cache re-verify
- `isAvailable(sdk)` becomes `!!cfg?.urlTemplate && !!getCurrentSdkTarget(sdk)`
- Update tests/mocks for the new shape

## Order

Runtime PR ships first. Build PR rebases onto it and changes
`produce.ts` shape to match. vscode-distro doesn't enable agentSdks
anywhere yet, so the order is purely about which one merges into a
consistent shape.
