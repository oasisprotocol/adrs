# Architectural Decision Records

This is a location to record all architecture decisions in the Oasis Core
project via [Architectural Decision Records] (ADRs).

<!-- markdownlint-disable line-length -->
[Architectural Decision Records]: https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions.html
<!-- markdownlint-enable line-length -->

## Format

Each record has a unique number associated with it to make it easier to
cross-reference them. The records are stored as Markdown files in this
directory, named using the following convention:

```
<adr-number>-<short-title>.md
```

Where:

* `<adr-number>` is the assigned zero-padded ADR number.
* `<short-title>` is the ADR's title in [Kebab case].

The content of each ADR should follow [the template]. In short, an ADR should
provide:

* a changelog of all modifications so far,
* context on the relevant goals and the current state,
* proposed changes to achieve the goals,
* summary of pros and cons,
* references and
* the decision that was made.

[Kebab case]: https://en.wikipedia.org/wiki/Letter_case#Special_case_styles
[the template]: template.md

## Process for Creating New ADRs

There is a lightweight process for proposing, discussing and deciding on ADRs:

* Create branch: If you have write permissions to the repository, you
  can create user-id prefixed branches (e.g. user/feature/foobar) in the main
  repository. Otherwise, fork the main repository and create your branches
  there.
  * Good habit: regularly rebase to the `HEAD` of `master` branch of the main
    repository to make sure you prevent nasty conflicts:

    ```bash
    git rebase <main-repo>/master
    ```

  * Push your branch to GitHub regularly so others can see what you are working
    on:

    ```bash
    git push -u <main-repo-or-your-fork> <branch-name>
    ```

    _Note that you are allowed to force push into your development branches._

* Use draft pull requests for work-in-progress:
  * The draft state signals that the code is not ready for review, but still
    gives a nice URL to track the ongoing work.

* In your branch, create a new ADR file following the convention and template
  specified above.
* Update the index of current records below.
* Create a pull request and mark it as ready for review. The commit message for
  introducing an ADR should have the title of the ADR, following by a short
  summary:

  ```
  ADR 0000: Architectural Decision Records

  Introduce architectural decision records (ADRs) for keeping track of
  architecture decisions in a transparent way.
  ```

* The ADR will be discussed by other members of the community and the project
  committers. After a sufficient amount of discussion, acceptance or rejection
  decision will be taken in accoordance with the governance process and the
  pull request will be merged, introducing a new ADR.

After the ADR is merged an implementation may be undertaken by following the
[contribution process].

<!-- markdownlint-disable line-length -->
[contribution process]:
  https://github.com/oasisprotocol/oasis-core/tree/master/CONTRIBUTING.md
<!-- markdownlint-enable line-length -->

## Current Records

The following records currently exist:

<!-- markdownlint-disable line-length -->
* [ADR 0000](0000-architectural-decision-records.md) - Architectural Decision Records
* [ADR 0001](0001-tm-multi-root-apphash.md) - Multiple Roots Under the Tendermint Application Hash
* [ADR 0002](0002-go-modules-compatible-git-tags.md) - Go Modules Compatible Git Tags
* [ADR 0003](0003-consensus-runtime-token-transfer.md) - Consensus/Runtime Token Transfer
* [ADR 0004](0004-runtime-governance.md) - Runtime Governance
* [ADR 0005](0005-runtime-compute-slashing.md) - Runtime Compute Node Slashing
* [ADR 0006](0006-consensus-governance.md) - Consensus Governance
* [ADR 0007](0007-improved-random-beacon.md) - Improved Random Beacon
* [ADR 0008](0008-standard-account-key-generation.md) - Standard Account Key Generation
* [ADR 0009](0009-ed25519-semantics.md) - Ed25519 Signature Verification Semantics
* [ADR 0010](0010-vrf-elections.md) - VRF-based Committee Elections
* [ADR 0011](0011-incoming-runtime-messages.md) - Incoming Runtime Messages
* [ADR 0012](0012-runtime-message-results.md) - Runtime Message Results
* [ADR 0013](0013-runtime-upgrades.md) - Runtime Upgrade Improvements
* [ADR 0014](0014-runtime-signing-tx-with-hardware-wallet.md) - Signing Runtime Transactions with Hardware Wallet
* [ADR 0015](0015-vrf-per-block-entropy.md) - Randomized Paratime Proposer Selection
* [ADR 0016](0016-consensus-parameters-change-proposal.md) - Consensus Parameters Change Proposal
<!-- markdownlint-enable line-length -->
