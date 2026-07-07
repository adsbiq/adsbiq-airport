# Code signing & antivirus-safety

This product installs a **USB driver**, runs as a **background service**, makes
**network connections**, and **auto-updates** — exactly the behaviors Windows
SmartScreen and heuristic AV engines flag. **Unsigned → "Windows protected your
PC" walls + occasional false positives. Signed → clean for the vast majority.**
Signing is load-bearing, not optional. Three things get us there.

## 1. Get a certificate — SignPath Foundation (free for OSS)

SignPath donates code-signing to open-source projects.

- **Eligibility:** public OSS repo (`github.com/adsbiq/adsbiq-airport` ✓) with an
  OSI-approved license (`MIT` ✓).
- **Apply:** <https://signpath.org/> → *Foundation* → submit the project (repo URL,
  license, short description, maintainers). Turnaround is typically 1–2 weeks —
  **apply now** so it's ready when the installer ships.
- **What you get:** an OV ("organization validated") certificate + a cloud signing
  service you drive from CI (no private key ever touches a dev box).
- OV clears the "unknown publisher" wall and builds SmartScreen reputation over
  time. (EV certs give *instant* SmartScreen trust but cost money + a hardware
  token — revisit only if download volume justifies it.)

### CI integration (activate after the SignPath project exists)
Add a signing step to the installer-build job. SignPath's action uploads the
artifact, signs it in their cloud, and returns the signed file:
```yaml
  - name: Sign the installer
    uses: signpath/github-action-submit-signing-request@v1
    with:
      api-token: ${{ secrets.SIGNPATH_API_TOKEN }}
      organization-id: ${{ vars.SIGNPATH_ORG_ID }}
      project-slug: adsbiq-airport
      signing-policy-slug: release-signing
      artifact-configuration-slug: installer
      github-artifact-id: ${{ steps.upload.outputs.artifact-id }}
      wait-for-completion: true
      output-artifact-directory: signed/
```
Secrets to add once approved: `SIGNPATH_API_TOKEN` (secret) + `SIGNPATH_ORG_ID`
(var). The signing policy + artifact config are defined in the SignPath portal.

## 2. Build SmartScreen / Defender reputation

- After the first signed build, submit the installer to Microsoft as a developer:
  **<https://www.microsoft.com/en-us/wdsi/filesubmission>** (choose "software
  developer" → false-positive/review). This pre-empts Defender false positives.
- SmartScreen reputation then accrues automatically from a **consistent signed
  publisher** + download volume. Early downloads may still warn until it builds;
  keeping the same cert/publisher across releases is what compounds trust.
- Before every public release, scan the artifacts on **VirusTotal** and chase down
  any engine hits (usually the driver installer `wdi-simple`).

## 3. Clean-build hygiene (so we don't look malicious)

- **Sign everything**, not just `setup.exe`: the agent, `wdi-simple.exe`, the
  decoders, and the installer. An installer that drops unsigned exes still trips AV.
- **Embed metadata** (company, product, version) in each exe. Go: add a
  `versioninfo` resource (e.g. `josephspurrier/goversioninfo`) so the agent isn't
  a metadata-less binary. Inno already sets publisher/version.
- **No packing / obfuscation / UPX** — packers are a top AV heuristic trigger.
- **Reproducible, public CI builds** (already the case) — transparency helps
  reputation and lets anyone verify the binary matches the source.
- **Publish SHA-256 checksums** with each release.

## Bottom line
Ship **signed** (SignPath) + Defender-submitted + clean-built, and the installer
is virus-scan-safe for the overwhelming majority of users. Ship unsigned and
expect friction on nearly every install. The single highest-leverage action is
**applying to SignPath Foundation today** — the clock on approval starts now.
