# aupm-donor-demo

A third-party donor rehearsal for AuPM on Algorand TestNet.

Each push to `master` and each pull request runs the `aupm-attest` action. The action posts
`package-lock.json` to the AuPM server and pays for the reviewed entries with the TestNet donor
key in the `AUPM_DONOR_MNEMONIC` secret. The job uploads the signed attestation as the
`aupm-attestation` artifact.

- `package-lock.json` is the real lockfile of `actions/checkout`.
- The repo variable `AUPM_ENDPOINT` holds the server origin.
- The job sets `NETWORK=testnet`. Without it, the donor code expects MainNet.
- The action fails open: a failed payment logs a warning, and the job stays green.

TestNet only. The reviews on the TestNet server are unread rehearsal records (AuPM ADR 0012).
