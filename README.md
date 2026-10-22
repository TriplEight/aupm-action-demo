# aupm-action-demo

The AuPM Action installs your npm packages through the AuPM registry and donates USDC to the
reviewers of the packages that a human reviewed. This repository is a working example.

Cost: free for packages with no review. A reviewed package costs 10,000 microUSDC ($0.01) per
lockfile entry at the default price.

## Prerequisites

- A GitHub repository with a `package-lock.json`.
- Node 22 or later.
- A funded donor account to donate. See
  [the donor procedure](https://github.com/TriplEight/AuPM/blob/main/docs/ONBOARDING.md#2-donor)
  for funding.

## Quickstart

1. Fork this repository. To use your own repository, copy `.github/workflows/ci.yml` instead.
2. In a fork, open the Actions tab. Click "I understand my workflows, go ahead and enable them".
3. Create the donor key:

   ```bash
   npm install -g aupm-cli
   aupm donor init
   ```

   The key file is `~/.config/aupm/donor.key`.
4. Store the key as a repository secret:

   ```bash
   gh secret set AUPM_DONOR_MNEMONIC_MAINNET < ~/.config/aupm/donor.key
   ```

   Without `gh`: open Settings > Secrets and variables > Actions > New repository secret. Never
   paste the words into a workflow file, an issue or a log.
5. Push a commit, or run the workflow `CI` from the Actions tab.

WARNING: a run with the MainNet secret donates real USDC on Algorand MainNet.

## The workflow

The Action replaces the `npm ci` step:

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

Copy the pin from `.github/workflows/ci.yml` of this repository. Never use a branch name, a tag
or a short SHA.

For free use, omit `donate` and `donor-secret`. The Action then installs through AuPM and
donates nothing.

The Action runs `npm ci` through the AuPM registry. If the AuPM registry fails, it logs a
warning and runs plain `npm ci`. An install failure fails the job, as with plain `npm ci`. An
AuPM failure never fails the job. The Action does not change the Node version or `PATH` of later
steps. The input reference is in
[the Action README](https://github.com/TriplEight/AuPM/blob/main/.github/actions/aupm/README.md).

## Verify the run

A paid run shows these results:

- The annotation `::notice::AuPM donation: <N> microUSDC, settlement txid <txid>`.
- The table "AuPM donation" in the job summary.
- The artifact `aupm-receipt-<network>`. It holds the signed `aupm-receipt.json`.

A missing notice means that nothing was paid. A failed donation logs a `::warning::`, and the
job stays green.

> [!NOTE]
> A pull request from a branch of the same repository gets the secret, so its run pays. The merge to the default branch is a push, and that run pays again. You can put `[skip ci]` in the merge commit message to skip the payment.

- Do not add a `schedule` trigger to a workflow that donates.
- Do not re-run a job that paid. The re-run donates again.
- A pull_request from a fork gets no secrets. The Action installs the packages, logs a
  `::warning::`, writes no receipt and donates nothing. The job stays green.

## Cost of one run

A run donates 10,000 microUSDC ($0.01) for each entry of `package-lock.json` that has a review on
the server of that network. The multiplier range is x0.1 (1,000 microUSDC, $0.001) to x100. A
package with no review is free. On MainNet, this lockfile has 30 reviewed entries. One run at
x1 donates 300,000 microUSDC ($0.30).

The two servers have different review sets. The cost of one run differs on each network.

To count the reviewed entries before a run, run the Action with no donor secret. It logs
`N reviewed package(s) withheld`. The run costs N × 10,000 microUSDC at x1.

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

## Rehearse on TestNet

A TestNet run donates TestNet USDC (ASA 10458941). It has no real-money value.

Before the run, set these values in Settings > Secrets and variables > Actions:

- The secret `AUPM_DONOR_MNEMONIC_TESTNET`: a TestNet donor with TestNet USDC and a little
  TestNet ALGO.
- The variable `AUPM_ENDPOINT_TESTNET`: the TestNet server origin. If it is empty, the Action
  installs from the npm registry, logs a warning and donates nothing.

In the GitHub web UI:

1. Open Actions, then the workflow `CI`.
2. Click Run workflow.
3. Select the branch `master` and set `network` to `testnet`.
4. Click Run workflow. The run name ends with `(TestNet rehearsal)`.

With the GitHub CLI:

```bash
gh workflow run ci.yml --repo TriplEight/aupm-action-demo -f network=testnet
```

The artifact of a TestNet run is `aupm-receipt-testnet`. To run on MainNet by hand, set `network`
to `mainnet`. This donates real USDC.

## Lockfile

`package.json` and `package-lock.json` are the dependencies of `actions/checkout` v7.0.1. The
reviews on the TestNet server are unread rehearsal records. The reviews on the MainNet server are
human reviews.
