# aupm-action-demo

A demo of the AuPM GitHub Action in a third-party repository.

The AuPM Action replaces the `npm ci` step of a workflow:

```yaml
# before
- run: npm ci

# after
- uses: TriplEight/AuPM/.github/actions/aupm@<40-character SHA>
  with:
    endpoint: https://aupm.fyi
    donate: 'true'
    donor-secret: ${{ secrets.AUPM_DONOR_MNEMONIC_MAINNET }}
```

The Action does these steps:

1. It runs `npm ci` through the AuPM registry. Packages with no review install free.
2. If the AuPM registry fails, it logs a warning and runs plain `npm ci`.
3. It checks the lockfile against the reviewed packages and writes the signed receipt
   `aupm-receipt.json`.
4. With `donate: 'true'`, it donates 1,000 microUSDC for each reviewed package in the lockfile.

An install failure fails the job, as with plain `npm ci`. An AuPM failure never fails the job.
The Action does not change the Node version or `PATH` of later steps.

The workflow is `.github/workflows/ci.yml`. The job uploads the signed receipt as the
`aupm-receipt-<network>` artifact.

WARNING: push and pull_request runs donate real USDC on Algorand MainNet.

## Networks

The run name and the job name show `(TestNet rehearsal)` for a TestNet run. A name without it is
a MainNet run.

| Trigger | Network | Endpoint | Donor secret |
| --- | --- | --- | --- |
| push to `master`, pull_request | MainNet | `https://aupm.fyi` | `AUPM_DONOR_MNEMONIC_MAINNET` |
| workflow_dispatch, `network: mainnet` (default) | MainNet | `https://aupm.fyi` | `AUPM_DONOR_MNEMONIC_MAINNET` |
| workflow_dispatch, `network: testnet` | TestNet | repo variable `AUPM_ENDPOINT_TESTNET` | `AUPM_DONOR_MNEMONIC_TESTNET` |

The job sets `NETWORK` to the selected network. The donor code in the Action reads it. A TestNet
secret never pays on MainNet, and a MainNet secret never pays on TestNet.

## Cost of one run

A run donates 1,000 microUSDC for each entry of `package-lock.json` that has a review on the
server of that network. A package with no review is free. The two servers have different review
sets, so the cost of one run is different on each network. On MainNet, this lockfile has 30
reviewed entries, so one run donates 30,000 microUSDC.

To count the reviewed entries before a run, run the Action with no
donor secret. It logs `N reviewed package(s) withheld`. The run costs N × 1,000 microUSDC.

Each push to `master` and each pull_request update donates again. A re-run of a job that paid
donates again. To push a commit with no run, put `[skip ci]` in the commit message.

## Set up

1. Add the repository secrets (Settings, Secrets and variables, Actions). Each value is the
   25-word mnemonic of an Algorand account:
   - `AUPM_DONOR_MNEMONIC_MAINNET`: a MainNet donor that holds USDC and a little ALGO.
   - `AUPM_DONOR_MNEMONIC_TESTNET`: a TestNet donor, for rehearsals only.
2. Add the repository variable `AUPM_ENDPOINT_TESTNET` with the TestNet server origin.
3. Keep the Action pinned to a full 40-character commit SHA of `TriplEight/AuPM`. Never use a
   branch name or a short SHA.

## Run on TestNet

A TestNet run donates TestNet USDC (ASA 10458941). It has no real-money value.

In the GitHub web UI:

1. Open Actions, then the workflow `CI`.
2. Click Run workflow.
3. Select the branch `master` and set `network` to `testnet`.
4. Click Run workflow. The run name ends with `(TestNet rehearsal)`.

With the GitHub CLI:

    gh workflow run ci.yml --repo TriplEight/aupm-action-demo -f network=testnet

Before the run, make sure that:

- The secret `AUPM_DONOR_MNEMONIC_TESTNET` holds a TestNet donor with TestNet USDC and a little
  TestNet ALGO.
- The variable `AUPM_ENDPOINT_TESTNET` holds the TestNet server origin. If it is empty, the
  Action installs from the npm registry, logs a warning and donates nothing.

The artifact of a TestNet run is `aupm-receipt-testnet`.

To run on MainNet by hand, use the same steps with `network` set to `mainnet`. This donates real
USDC.

## Results

- A paid run logs `::notice::AuPM donation: <N> microUSDC, settlement txid <txid>`. The job
  summary shows the same line.
- A failed donation logs a `::warning::`, and the job stays green. A missing notice is the signal
  that nothing was paid.
- A pull_request from a fork gets no secrets. The Action installs the packages, logs a
  `::warning::`, writes no receipt and donates nothing. The job stays green.

`package.json` and `package-lock.json` are the dependencies of `actions/checkout` v7.0.1. The
reviews on the TestNet server are unread rehearsal records (AuPM ADR 0012). The reviews on the
MainNet server are human reviews.
