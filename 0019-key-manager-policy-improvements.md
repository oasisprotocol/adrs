# ADR 0019: Key Manager Policy Improvements

## Component

Oasis Core

## Changelog

- {date}: {changelog}

## Status

Proposed

## Context

The initial key manager design uses a dynamic policy for access control,
that is signed by a hard-coded set of signers.  While the current design
is functional, there are several "ease of use" type improvements that can
be made to how policies and their signatures are handled.

## Decision

### KM Enclave Status changes

To facilitate extra verification at the consensus layer, the per-key manager
instance runtime status field is to be extended as follows:

```golang
type Status struct {
  // ... existing fields omitted ...

  // BaseSigners is the list of allowed policy signers, as hard-coded in
  // the enclave binary.
  BaseSigners []signature.PublicKey `json:"base_signers,omitempty"`

  // BaseSignerThreshold is the number of valid signatures that are required
  // for a policy, as hard-coded in the enclave binary.
  BaseSignerThreshold uint32 `json:"base_signer_threshold,omitempty"`
}
```

The ordering of the elements in the `BaseSigners` list is not explicitly
specified beyond the fact that it MUST be globally consistent across all
deployments of a given enclave binary.

It is also emphasized that, the list is of the hard-coded policy signers,
as of the time the enclave binary was built.

### KM Policy Updates

The existing key manager `UpdatePolicy` method is to be deprecated, and
replaced with the following methods.

```golang

type PolicyProposal struct {
  // The unsigned policy.
	Policy PolicySGX `json:"policy"`

  // Manually collected signatures.
  Signatures []signature.Signatures `json:"signatures,omitempty"`

  // XXX: Specify some sort of voting period... or hardcode it? idk.
}

type PolicyVote struct {
  // The policy serial number that this vote is for.
  Serial    uint32 `json:"serial"`
  // The runtime ID that this vote is for.
  ID        common.Namespace `json:"id"`

  // The signature over the PolicyProposal.Policy.
  Signature signature.Signature `json:"signature"`
}

// TODO: Document the backend extensions.

var (
	MethodPolicyPropose = transaction.NewMethodName(ModuleName, "PolicyPropose", PolicyProposal{})
  MethodPolicyVote = transaction.NewMethodName(ModuleName, "PolicyVote", PolicyVote{})
)
```

Like with `UpdatePolicy`, the new update process is initiated by the owner
of the KM runtime in question, but now with a `PolicyProposal`.

If other entities support the update, they will sign the policy and submit
the signature as a `PolicyVote` transaction.

On the consensus side, votes:
- Must be for an open (not expired) `PolicyProposal`
- Must be signed by a public key that is part of `Status.BaseSigners`

Once the number of signatures (be them from `PolicyProposal.Signatures` or
`PolicyVote`s) is greater than `Status.BaseSignerThreshold`, the policy
is considered ratified, and will take effect on the key managers nodes.

As a special case to support legacy KM runtime deployments, `UpdatePolicy`
transactions submitted for existing runtimes, that both have an empty
`Status.BaseSigners` list AND a `0` `Status.BaseSignerThreshold`, are
treated as ratified iff `UpdatePolicy.Signatures` contains more than one
valid signature.

### KM Key Revocation

A new key manager method `PolicyRevokeKey` is to be added.

```golang
var (
  MethodPolicyRevokeKey = transaction.NewMethodName(ModuleName, "PolicyRevokeKey", struct{})
)

type Backend interface {
  // Existing methods omitted....

  GetRevocations(context.Context, int64) ([]signature.PublicKey, error)
}
```

Submitting a `PolicyRevokeKey` transaction, will invalidate the signing key
for the purpose of ratifying key manager policies.  Note that revocations
are expected to take effect the next time various nodes re-attest (the light
client MUST pull down the revocation list).

## Consequences

### Positive

It will be significantly easier to update key manager policies as the tabulation
of signatures will be done on-chain.

Revocation may be a more expedient emergency halt mechanism than the alternatives.

### Negative

This adds non-trivial amount of complexity for, what realitically is a process
that a handful of people will ever need to do.

### Neutral

## References

> Are there any relevant PR comments, issues that led up to this, or articles
> referenced for why we made the given design choice? If so, link them here.

- {reference link}
