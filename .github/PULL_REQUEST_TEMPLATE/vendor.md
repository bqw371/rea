<!--
Vendor PR template. Use when a PR adds, updates, or removes a `third_party/<name>/` upstream.
Pick this template from GitHub's selector, or append `?template=vendor.md` to the compare URL.
Full policy: docs/vendor-policy.md
-->

## Vendor integration

<!-- One sentence: what upstream this is and what REA uses it for. -->

## Immediate items (unwaivable — docs/vendor-policy.md §Supply-chain hardening)

- **Upstream URL:** <!-- canonical repo REA pulls from -->
- **Pinned commit SHA (40 chars):** <!-- no tag, no branch, no `HEAD` -->
- **SPDX license id:** <!-- e.g. MIT, Apache-2.0, GPL-3.0-only -->
- **Scrub ran clean:** yes / no <!-- if yes, paste the grep output below; if any
  match is a benign upstream test fixture, annotate it -->

```text
<!-- paste output of the grep commands from docs/vendor-policy.md §Scrub here -->
```

- **GPL-family subtree?** yes / no <!-- if yes, confirm no REA source
  under src/ or bridge/ imports from third_party/<name>/ (vendor-boundary CI
  enforces this; this field is the author-side ack) -->
