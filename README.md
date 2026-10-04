# aupm-action-demo

A third-party donor demo for AuPM on Algorand.

The workflow `.github/workflows/aupm-attest.yml` runs the `aupm-attest` action. The action posts
`package-lock.json` to the AuPM server and pays for the reviewed entries. The job uploads the
signed attestation as the `aupm-attestation-<network>` artifact.

WARNING: push and pull_request runs target Algorand MainNet and pay real USDC.

## Networks

The run name, the job name and the step names show the network of each run.

| Trigger | Network | Endpoint | Donor secret |
| --- | --- | --- | --- |
| push to `master`, pull_request | MainNet | `https://aupm.fyi` | `AUPM_DONOR_MNEMONIC_MAINNET` |
| workflow_dispatch, `network: mainnet` (default) | MainNet | `https://aupm.fyi` | `AUPM_DONOR_MNEMONIC_MAINNET` |
| workflow_dispatch, `network: testnet` | TestNet | repo variable `AUPM_ENDPOINT_TESTNET` | `AUPM_DONOR_MNEMONIC_TESTNET` |

The job sets `NETWORK` to the selected network. The donor code in the action reads it. A
TestNet secret never pays on MainNet, and a MainNet secret never pays on TestNet.

## Cost of one run

A run pays 1,000 microUSDC for each entry of `package-lock.json` that has a review on the server
of that network. A package with no review is free. The two servers have different review sets,
so the cost of one run is different on each network.

To count the reviewed entries before a run, run the free partial attestation with no donor key:

    aupm attest package-lock.json

It prints `withheld N reviewed entries`. The run costs N × 1,000 microUSDC.

Each push to `master` and each pull_request update pays again. A re-run of a job that paid
pays again.

## Set up

1. Add the repository secrets (Settings, Secrets and variables, Actions):
   - `AUPM_DONOR_MNEMONIC_MAINNET`: a MainNet donor that holds USDC and a little ALGO.
   - `AUPM_DONOR_MNEMONIC_TESTNET`: a TestNet donor, for rehearsals only.
2. Add the repository variable `AUPM_ENDPOINT_TESTNET` with the TestNet server origin.
3. Keep the action pinned to a full 40-character commit SHA of `TriplEight/AuPM`. Never use a
   branch name or a short SHA. The SHA must contain AuPM PR #45 (merge
   `ef335ce9a5a3ecd11231831e32241ea505b02bce`) or a later commit. An older SHA builds the
   MainNet payment with TestNet parameters, and the facilitator rejects it.

## Run on TestNet

A TestNet run pays TestNet USDC (ASA 10458941). It has no real-money value.

In the GitHub web UI:

1. Open Actions, then the workflow `aupm-attest`.
2. Click Run workflow.
3. Select the branch `master` and set `network` to `testnet`.
4. Click Run workflow. The run name is `aupm-attest on testnet`.

With the GitHub CLI:

    gh workflow run aupm-attest.yml --repo TriplEight/aupm-action-demo -f network=testnet

Before the run, make sure that:

- The secret `AUPM_DONOR_MNEMONIC_TESTNET` holds a TestNet donor with TestNet USDC and a little
  TestNet ALGO.
- The variable `AUPM_ENDPOINT_TESTNET` holds the TestNet server origin. If it is empty, the action
  logs `::warning::` and pays nothing.

The artifact of a TestNet run is `aupm-attestation-testnet`.

To run on MainNet by hand, use the same steps with `network` set to `mainnet`. This pays real USDC.

## Results

- A paid run logs `::notice::AuPM donation: <N> microUSDC, settlement txid <txid>`. The job
  summary shows the same line.
- The action fails open. A failed payment logs a `::warning::`, and the job stays green. A
  missing notice is the signal that nothing was paid.
- A pull_request from a fork gets no secrets. The action logs a `::warning::`, writes no
  attestation and pays nothing. The upload step then logs a warning, and the job stays green.

`package-lock.json` is the real lockfile of `actions/checkout`. The reviews on the TestNet
server are unread rehearsal records (AuPM ADR 0012). The reviews on the MainNet server are
human reviews.
