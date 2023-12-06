# ADR 0023: Secret Sharing Schemes (CHURP)

## Component

Oasis Core

## Changelog

- 2023-12-01: Initial proposal

## Status

Draft

## Context

Currently, key managers derive keys from either master secrets or ephemeral
secrets, which are unique to the key manager runtime and shared among key
manager nodes. We acknowledge that this approach is not comprehensive,
as a compromise of a single key manager enclave would reveal past secrets,
potentially leading to the decryption of the internal state of runtimes
using them. However, this key derivation method is straightforward and,
as a result, exceptionally fast.

While master secret rotations and ephemeral secrets aim to rotate the secrets
to mitigate the impact of a secret compromise, we also want to support
different types of key derivation, each offering varying levels of security.
Examples include verifiable secret sharing schemes and dynamic-committee
proactive secret sharing, where key managers would only hold a share
of the secret.

This proposal aims to introduce support for the CHUrn-Robust Proactive
secret sharing scheme (CHURP) and Key Derivation Center (KDC).

## Key manager apps

To facilitate the straightforward addition of new features to the key manager,
we must first generalize it to support the concurrent execution of multiple
(independent) applications.

Not all key manager runtimes are required to support all apps, and likewise,
not all nodes need to run all the apps. In fact, the committee nodes for each
application should be dynamic, allowing for the addition or removal of nodes
based on specific requirements.

### Current situation

Currently, the key manager supports two applications: one for generating,
distributing, and storing master secrets, and the other for ephemeral secrets.
However, these two applications are not independent, as they both share
the same key manager policy for secret replication and key derivation.

Issues:

- In the runtime, the logic for key manager status, policy, master secrets,
  and ephemeral secrets should be decoupled.

- On the host, each application should have its own worker (e.g., a master
  secret app should have a dedicated worker responsible for participating
  in the master secret protocol).

### Example apps

Current and future applications:

- Master secrets (for generation and replication master secrets)

- Ephemeral secrets (for generation and replication of ephemeral secrets)

- CPU change (for detecting wether the CPU has changed)

- CHURP (secret sharing scheme)

- Key derivation center (secret sharing scheme)

### App trait

Every application should implement the following trait.

```rust
/// Key manager application.
pub trait App {
  /// Register RPC methods to the dispatcher.
  ///
  /// Registered methods are exposed via enclave RPC to the local host
  /// or remote clients.
  fn register(&self, ...);

  /// Initialize the application on startup.
  ///
  /// Use runtime ID, untrusted local storage, host data, consensus verifier,
  /// runtime identity, etc. to start the application.
  fn init(&self, ...);
}
```

Each application should register RPC methods and adhere to the naming
convention `app.Method`.

#### Example 1

```rust
/// Master secrets key manager application.
pub trait MasterSecrets {
  fn generate(&self);
  fn load(&self);
  fn replicate(&self);
  fn key_pair(&self);
  fn private_key(&self);
  fn public_key(&self);
  fn symmetric_key(&self);
  fn update_status(&self);
}
```

Methods:

- `MasterSecrets.Generate`

- `MasterSecrets.Load`

- `MasterSecrets.Replicate`

- `MasterSecrets.KeyPair`

- `MasterSecrets.PrivateKey`

- `MasterSecrets.PublicKey`

- `MasterSecrets.SymmetricKey`

- `MasterSecrets.UpdateStatus`

#### Example 2

```rust
/// CPU change detection key manager application.
pub trait CPUChangeDetection {
  fn encrypt(&self);
  fn decrypt(&self);
}
```

Methods:

- `CpuChange.Encrypt`

- `CpuChange.Decrypt`

### App worker

Each key manager application should have a dedicated worker on the host node
responsible for communicating with the app and ensuring its consensus view
is up-to-date.

```go
// AppWorker is a key manager application worker.
type AppWorker interface {
  // Start starts the worker.
  Start()
  // Stop stops the worker.
  Stop()
}
```

#### Example

A master secret worker should be responsible for participating in the master
secret protocol.

## CHURP

CHURP is a proactive secret sharing scheme in which the committee of nodes
storing a secret can change over time.

### Requirements

- Support multiple secrets.

  - Each group of clients can share a unique secret.

  - Secrets may vary in terms of committee size, handoff intervals,
    security levels, etc.

  - State re-encryption should be possible with keys derived from
    a new shared secret.

- Key manager restarts should not lose shares or committed data.

### Protocol

#### Offline

- Assign non-zero ID numbers to all nodes and associate them with their
  public keys.

- The key manager owner selects a Schnorr group based on security
  requirements and prepares the CHURP configuration and access policy.

  - The Schnorr group can also be generated by one of the enclaves.
    In this case, the owner only needs to define security parameters.

#### Initialization

- The key manager owner publishes the configuration in the consensus layer.

- Key manager nodes interested in participating update their configuration
  file and restart the node.

  - To prevent the need for restarting the node, a command could be added
    to the CLI.

- Following the restart, the node's enclave prepares a non-zero-hole
  verification matrix for the dealing phase and registers with its checksum
  (public commitment).

#### Dealing phase

- Starts at the beginning of the first epoch with a sufficient number
  of registered nodes.

- The first t+2 nodes serve as dealers.

  - Make sure to include enough nodes to prevent dealer corruption.

  - Verification matrices of other nodes are disregarded, and their entropy
    will not be included in the secret. However, they will still receive
    a share.

- The construction of the shared secret and dealing occurs offline through
  a peer-to-peer network, following the specified enclave policy.

- Each registered node (dealing):

  - requests its bivariate shares (polynomials and non-zero-hole
    verification matrices) from the dealers,

  - validates received shares,

  - verifies non-zero-hole verification matrices against the consensus layer,

  - combines shares (adds polynomials and merges non-zero-hole verification
    matrices),

  - seals the result (full share) and stores it locally in the enclave's
    confidential storage,

  - sends a transaction containing the checksum of the merged matrix
    to the consensus layer, confirming receipt of all shares.

- If the timeout/epoch expires or checksums do not match, the registry
  is cleared, and nodes must re-register.

- Upon receiving confirmations from all registered nodes, the consensus layer
  announces the new committee and begins collecting registrations for the
  first handoff.

  - Nodes can register for the handoff no earlier than one epoch in advance.

- The dealers delete dealing data.

- The committee starts serving requests.

#### Serving

- To construct a derived key, a key manager client must contact at least
  the threshold number of nodes within the committee to acquire the necessary
  derived key shares.

- The committee responds exclusively to nodes defined in the policy.

- Blame detection should/can be added later (expensive).

#### Handoff

- Starts if sufficient time has elapsed since the last handoff/dealing
  and an adequate number of nodes have prepared a zero-hole verification
  matrix and registered for the new committee.

- Each registered node (share reduction):

  - requests switch data points for constructing the dimension switched
    polynomial and the merged verification matrix from the current committee,

  - validates received points,

  - verifies the merged verification matrix against the consensus layer,

  - combines the points into a polynomial (reduced share).

- Each registered node (proactive randomization):

  - requests its bivariate shares (polynomials and zero-hole
    verification matrices) from the new committee members,

  - validates received shares,

  - verifies zero-hole verification matrices against the consensus layer,

  - applies shares to the secret polynomial and to the merged verification
    matrix.

- Each registered node (full share distribution):

  - requests switch data points for constructing the dimension switched
    polynomial and the proactive verification matrix from the new committee
    members,

  - verifies received points,

  - combines the points into a polynomial (full share),

  - seals the result (full share) and stores it locally in the enclave's
    confidential storage,

  - sends a transaction containing the checksum of the proactive verification
    matrix to the consensus layer confirming that the full share was received.

- If the committee hasn't changed, skip the share reduction and full node
  distribution steps, and only execute proactive randomization.

- If the timeout/epoch expires or checksums do not match, the registry
  is cleared, and nodes must re-register.

- Upon receiving confirmations from all registered nodes, the consensus layer
  announces the new committee and begins collecting registrations for the
  next handoff.

- The old committee deletes obsolete full shares.

- The committee starts serving requests.

### Identification

```go
// NodeToID assigns a unique ID to a node for use in the CHURP protocol.
func NodeToID(nodeID signature.PublicKey) []byte {
  id := nodeID
  return id[:]
}
```

### Configuration

```go
// SchnorrPrime is a Schnorr prime p, where p = qr + 1, for some prime q.
type SchnorrPrime struct {
  // P is the prime number p.
  P []byte `json:"p"`
  // Q is the prime factor of p-1.
  Q []byte `json:"q"`
  // R is the reminder of p-1 when divided by q.
  R uint32 `json:"r"`
}

// SchnorrGroup is a large prime-order subgroup of the multiplicative group
// of integers modulo Schnorr prime p.
type SchnorrGroup struct {
  // Modulus is a Schnorr prime.
  Modulus SchnorrPrime `json:"modulus"`
  // Generator is a generator of the group.
  Generator []byte `json:"generator"`
}

// Config contains the CHURP configuration.
type Config struct {
  // Group is a Schnorr group.
  Group SchnorrGroup `json:"group"`

  // Threshold is the minimum number of distinct shares required to reconstruct a secret.
  Threshold uint8 `json:"threshold"`

  // HandoffInterval is the time interval in epochs between handoffs.
  //
  // Zero value disables handoffs.
  HandoffInterval beacon.EpochTime `json:"handoff_interval,omitempty"`

  // Verification is true iff the dealing and the handoffs should be verified.
  Verification bool `json:"verification,omitempty"`

  // BlameAssignment is true iff the responses should be checked
  // for corrupted shares.
  BlameAssignment bool `json:"blame_assignment,omitempty"`
}

// PolicySGX is a CHURP access control policy.
type PolicySGX struct {
  // Serial is the monotonically increasing policy serial number.
  Serial uint32 `json:"serial"`

  // ID is the runtime ID that this policy is valid for.
  RuntimeID common.Namespace `json:"runtime_id"`

  // ID is the CHURP instance ID that this policy is valid for.
  ChurpID uint8 `json:"churp_id"`

  // MayQuery is the map of runtime IDs to the vector of enclave IDs that
  // may query private key material.
  MayQuery map[common.Namespace][]sgx.EnclaveIdentity `json:"may_query,omitempty"`

  // MayShare is the vector of enclave IDs from which a share can be obtained
  // during handouts.
  MayShare []sgx.EnclaveIdentity `json:"may_share,omitempty"`

  // MayJoin is the vector of enclave IDs that may join the new committee
  // during handoffs.
  MayJoin []sgx.EnclaveIdentity `json:"may_join,omitempty"`
}

// SignedPolicySGX is a signed SGX CHURP access control policy.
type SignedPolicySGX struct {
  Policy PolicySGX `json:"policy"`

  Signatures []signature.Signature `json:"signatures"`
}
```

### Consensus transactions

```go
var (
  // MethodChurpCreate is the method name for creating a new CHURP instance.
  MethodChurpCreate = transaction.NewMethodName(
    ModuleName, "Churp.Create", Config{},
  )

  // MethodChurpUpdatePolicy is the method name for CHURP policy updates.
  MethodChurpUpdatePolicy = transaction.NewMethodName(
    ModuleName, "Churp.UpdatePolicy", SignedPolicySGX{},
  )

  // MethodChurpRegister is the method name for node registration with the given checksum.
  MethodChurpRegister = transaction.NewMethodName(
    ModuleName, "Churp.Register", hash.Hash{},
  )
)
```

### Churp application status

```go
// ChurpStatus is the current CHURP status.
type ChurpStatus struct {
  // ID is a unique identifier of this CHURP instance.
  ID uint8 `json:"id,omitempty"`

  // Config is configuration of this CHURP instance.
  Config Config `json:"config"`

  // Policy is the CHURP policy.
  Policy *SignedPolicySGX `json:"policy"`

  // Round is the current round.
  //
  // Zero round is the dealer round.
  Round uint64 `json:"round,omitempty"`

  // Committee is a list of nodes holding a share in the current round.
  Committee []signature.PublicKey `json:"committee,omitempty"`

  // Registrations is a map from node ID to the checksum of verification matrices.
  Registrations map[signature.PublicKey]hash.Hash `json:"registrations,omitempty"`
}

// ChurpStatuses are the current CHURP statuses.
type ChurpStatuses struct {
  // Statuses is a map from CHURP ID to the statuess.
  Statuses map[uint8]ChurpStatus `json:"statuses,omitempty"`
}
```

### Key manager worker

```go
// Config is the keymanager worker configuration structure.
type Config struct {
  ...

  // Churp is map of CHURP configurations that the node will participate in. 
  Churp map[uint8]ChurpConfig `yaml:"churp"`
}

// ChurpConfig is configuration for CHURP.
type ChurpConfig struct {}
```

### Key manager runtime application

```rust
/// Key manager application that implements churn-robust proactive secret
/// sharing scheme (CHURP).
pub trait Churp {
  /// Prepare CHURP for participation in the given round of the protocol.
  /// 
  /// Initialization randomly selects a bivariate polynomial for the given
  /// round, computes the corresponding verification matrix and its checksum,
  /// and signs the latter.
  ///
  /// Bivariate polynomial:
  ///     B(x,y) = \sum_{i=0}^{t_n} \sum_{j=0}^{t_m} a_{i,j} x^i y^j
  ///
  /// Verification matrix: 
  ///     A = [g^{a_{i,j}} \mod{p}]
  ///
  /// Checksum:
  ///     H = KMAC256(A, runtime ID, round)
  ///
  /// In the zero round (dealing phase), the bivariate polynomial is always
  /// a zero-hole polynomial.
  ///
  /// This method must be called locally.
  fn init(&self, churp_id: u8, round: u64) -> SignedChecksum;

  /// Return bivariate verifiable secret sharing verification matrix.
  ///
  /// Verification matrix is a matrix of dimension t_n x t_m containing
  /// encrypted coefficients of the secret bivariate polynomial.
  ///
  /// Verification matrix: 
  ///     A = [g^{a_{i,j}} \mod{p}]
  ///
  /// Bivariate polynomial:
  ///     B(x,y) = \sum_{i=0}^{t_n} \sum_{j=0}^{t_m} a_{i,j} x^i y^j
  ///
  /// The matrix is used to verify that polynomials and data points derived
  /// from the bivariate polynomial are valid.
  ///
  /// This method can be called over an insecure channel as the matrix
  /// doesn't contain any sensitive information.
  fn verification_matrix(&self, churp_id: u8, round: u64);

  /// Return bivariate share (polynomial) for the calling node.
  /// 
  /// The polynomial is partial evaluation of the bivariate polynomial
  /// at a given x or y value (caller id).
  /// 
  /// Polynomial:
  ///     g_{node_id}(y) = B(caller_id,y) (dealing phase)
  ///     g_{node_id}(x) = B(x,caller_id) (proactive randomization)
  ///
  /// Bivariate polynomial:
  ///     B(x,y) = \sum_{i=0}^{t_n} \sum_{j=0}^{t_m} a_{i,j} x^i y^j
  ///
  /// WARNING: This method must be called over a secure channel as
  /// the polynomial needs to be kept secret and generated only
  /// for authorized nodes.
  fn bivariate_share(&self, churp_id: u8, round: u64);

  /// Construct a full share from the verified bivariate shares obtained
  /// from the dealers.
  ///
  /// Full secret:
  ///      s(y) =  B(node_id,y) = \sum_g_{node_id}(y)
  fn dealing(&self, churp_id: u8, round: u64);

  /// Construct a reduced share from the switch data points obtained from
  /// the nodes in the current committee.
  ///
  /// Reduced share:
  ///      r(x) = B(x, node_id)
  fn share_reduction(&self, churp_id: u8, round: u64);

  /// Randomize reduced share using bivariate shares obtained from
  /// the nodes in the new committee.
  ///
  /// Proactive reduced share:
  ///      r'(x) = B(x, node_id) + \sum Q(x, node_id)
  fn proactive_randomization(&self, churp_id: u8, round: u64);

  /// Construct a full share from the switch data points obtained from 
  /// the nodes in the new committee.
  ///
  /// Full share:
  ///      s'(y) = B'(node_id,y)
  fn full_share_distribution(&self, churp_id: u8, round: u64);

  /// Return switch data point (integer) for the calling node.
  /// 
  /// The point is evaluation of the shared secret (polynomial)
  /// at the given y value (caller id).
  /// 
  /// Switch point:
  ///     P = g(caller_id) = B(node_id,caller_id)
  ///
  /// WARNING: This method must be called over a secure channel as
  /// the polynomial needs to be kept secret and generated only
  /// for authorized nodes.
  fn switch_point(&self, churp_id: u8, round: u64) -> Integer;

  /// Return the key share for the given round.
  ///
  /// To construct a derived key, the caller is required to collect 
  /// at least the threshold number of key shares and combine them
  /// locally to construct the derived key.
  ///
  /// Key share:
  ///     K = H(runtime_id, key_pair_id)^{r g(0)}
  ///
  ///
  /// The caller should always provide a proof which can be independently
  /// verified before key shares are released to the caller.
  ///
  /// WARNING: This method must be called over a secure channel as
  /// the key share needs to be kept secret and generated only
  /// for authorized nodes.
  fn key_share(&self, churp_id: u8, round: u64, runtime_id: Namespace, 
    key_pair_id: KeyPairId, proof: Proof) -> Integer;
}
```

Methods:

- `Churp.Init`

- `Churp.VerificationMatrix`

- `Churp.BivariateShare`

- `Churp.Dealing`

- `Churp.ShareReduction`

- `Churp.ProactiveRandomization`

- `Churp.FullShareDistribution`

- `Churp.SwitchPoint`

- `Churp.KeyShare`

### Key manager client

```rust
/// Key manager client for CHURP.
pub trait Client {
  /// Return the key for the given round.
  ///
  /// The key is combined from key shares obtained from the key manager nodes
  /// in the current CHURP committee.
  fn key(&self, churp_id: u8, round: u64, runtime_id: Namespace, key_pair_id: KeyPairId) -> Vec<u8>;
}
```

## Key Derivation Center

Key Derivation Center (KDC) is a secret sharing scheme based on
the verifiable secret sharing scheme (VSS) where every node possesses
only a share of the master secret. To derive a key from the master secret,
one needs to obtain at least t+1 threshold number of key shares from distinct
nodes and reassemble them locally.

```rust
/// Key manager application that implements key derivation center (KDC).
pub trait KDC {
  /// TODO: Define methods if we decide to implement KDC also.
}
```

## Consequences

### Positive

CHURP:

- High security, as the master secret is shared among key manager nodes.

- Supports proactive randomization (share refresh).

- Dynamic committees.

KDC:

- High security, as the master secret is shared among key manager nodes.

- Supports proactive randomization (share refresh).

### Negative

CHURP:

- Handoffs are computationally intensive.

KDC:

- The number of key manager nodes that share a master secret is fixed and
  cannot be changed once shares are generated. Consequently, if too many
  nodes are destroyed, the secret cannot be recovered.

- Support for replicating a share to a specific node is needed.

### Neutral

- Issuing derived key shares with CHURP should be slightly slower compared
  to KDC.
