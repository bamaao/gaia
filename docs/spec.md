# Gaia Planning

**May 25, 2017**

The initial production release of the Cosmos network, codenamed *Gaia*, will be a Tendermint-based blockchain created for the sole purpose of letting Atom holders decide on a set of initial validators for Cosmos Genesis (aka. the "delegation game").

Before launching Gaia, we will need to build:
- Delegation ABCI app (should allow candidates to announce themselves, and delegators to submit their delegation choices)
- Desktop application (should allow users to easily view/verify candidates, and delegate to them)
- Gaia testnet (we should put up a deployment of this network on our own nodes during development)
- Gaia mainnet (we will need to coordinate with members of the community to run Gaia's validator nodes in order to run the delegation game)

## Delegation Game

Many members of the community are eager to take part in the Cosmos consensus process by running a validator node, but due to technical limitations there can only be 100 of them. The delegation game will let the Atom holders sign their *intent to delegate*, which includes the validator to delegate to, and the amount of Atoms to bond. Once a certain date and time is reached, the game will end and the 100 validators with the most Atoms delegated will lock in as the validator set for Cosmos Genesis.

### Announcing Candidacy

Validator candidates will broadcast a `CandidacyTx` transaction to the Gaia network, which contains:
- Validator public key
- Signature (using corresponding validator private key)
- Display name
- Candidacy message (a string to display in user interfaces)
- ID verification
- Interest commission rate

**TODO:** `CandidacyTx` rate-limiting. We won't be able to use transaction fees since Atoms won't be transferrable yet, but we want to prevent people from spamming these transactions. Maybe we should require a minimum amount of Atoms to be delegated along with the `CandidacyTx`? (However, this may discourage candidates who do not hold Atoms).

**ID Verification**

It will be important for candidates to prove their identity through external means which can be validated by all delegators. Without this, candidates could pose as a certain company or public figure to attract delegators. Verification could include publishing the validator pubkey through these channels:
- Serving via HTTPS (to prove domain name ownership)
- Social media accounts, e.g. Twitter (to prove account ownership)
- PGP signature (to link to a known PGP pubkey)

### Delegating

Atom holders may sign their intent to delegate using the pubkey associated with their publicly-known Cosmos address, apportioning their Atoms to one or more candidate validators. They are not required to delegate all of their Atoms, and the un-delegated amount will stay liquid at the time of Cosmos Genesis. This will be done by broadcasting a `DelegationIntentTx`, which includes:
- Publicly-known Cosmos address
- Public key associated with Cosmos address
- Signature (using corresponding Cosmos private key)
- List of candidates to delegate to, and an amount of Atoms for each
- Sequence number (delegators may make updates to their delegation during the Delegation Game)

Nodes already know the suggested Atom allocation based on the donations made in the Cosmos Fundraiser, so they will be able to look up the Atom amount for each address.

To prevent DoS by spamming these transactions, a rule could be implemented to rate-limit `DelegationIntentTx` creation by Cosmos address (e.g. only one transaction can be made for a certain Cosmos address per 100 blocks).

### Game End

The Delegation Game will end at a predetermined time, which should be chosen to give Atom holders plenty of time to research candidates and make decisions.

**TODO:** Should we plan for Genesis to happen as soon as the Delegation Game ends? If so, that will affect our choice for the end time.

### Gaia Validators

While Gaia exists to choose the initial Cosmos Genesis validators, the validator nodes of the Gaia network itself must be chosen somehow. The decision of who to accept as a Gaia validator will be made by the Cosmos developers, but will be done via a public channel to be as transparent as possible: anyone may apply to be a Gaia validator by opening a Pull Request on a public Github repository, and anyone may discuss the decisions via comments.

### Potential Cryptoeconomic Attacks

**TODO** Anything we need to worry about? E.g. Gaia validators manipulating the results somehow.

## ABCI App Design

The Gaia ABCI app should support the `CandidacyTx` and `DelegationIntentTx` behavior as described above. Writing the functionality as a Basecoin plugin is not needed since there will be no transferrable coins on Gaia.

### Data Types

```go
type CandidacyTx struct {
  ValidatorPubKey crypto.PubKey
  Signature crypto.Signature
  Name string
  Message string
  IDProofs []IDProof
  InterestCommission uint32 // max 32-bit uint = 100%, 0 = 0%
}

// IDProof provides an external identity source (e.g. domain names, Twitter, Estonian e-citizenship), and a link to proof of ownership of that ID.
type IDProof struct {
  Type string // e.g. "twitter", "HTTPS"
  ID string // e.g. Twitter handle, domain name
  Proof string // e.g. Tweet URL containing
}
```

**TODO:** Include proof of Atom ownership so candidates can show that they will be bonding their own coins.

```go
// DelegationIntentTx is a way for an Atom holder to announce the validator delegations they would like to make. Once Gaia ends, the last state of delegations will be bonded on the Genesis chain.
type DelegationIntentTx struct {
  PubKey crypto.PubKey
  Signature crypto.Signature
  Delegations []Delegation
  Sequence uint32
}

type Delegation struct {
  ValidatorPubKey crypto.PubKey
  Amount uint64
}
```

### State

The Delegation Game ABCI app will maintain the list of validator candidates and delegation intent.

**Candidate List**
The list of validator candidates can be stored as an array, containing the data in the `CandidacyTx` along with the sum of coins delegated to this candidate. The array is unbounded in size, so care must be taken to prevent attackers from cheaply generating many "candidates" to inflate the list (e.g. by requiring a *delegation intent* to be included with a minimum number of Atoms to be delegated to the candidate).

**Delegations**
Delegations can be tracked in a Merkleeyes store, using pubkey addresses as keys, and the information in `DelegationIntentTx` as the value (minus the signature).

## TODO
- Desktop app UX
