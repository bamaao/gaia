# Slashing protocol

We had a long discussion of how to handle the mechanics of slashing during the team retreat.  Many of the proposals would result in O(N) storage of the validator info, increasing with each delegator, and O(N) db accesses to pay out the interest or to slash.

The basic idea to solve this was to consider bonds a special class of currency, stored on each delegator account. In addition, there is one key that stores the general info of the bond type - how many bonds exist, what their value relative to atoms are, and some other parameters to control payout to the validator owner, separate from the delegates.

## Requirements

We iterated over several designs for this structure, and over the course of that, several requirements came up that informed this design:

* All operations (bonding, unbonding, slashing) must be O(1), that is independent of the number of delegators or validators.
* All delegated bonds must increase in value in proportion to the interest rate (minus the owner's commission) over the time they invested
* On evidence of malice, all delegated bonds must be slashed, bonded or unbonding
* The owner's commission must be paid out in atoms, that is liquidity to pay for the running costs of the system, with no unbonding period
* The owner cannot suddenly debond all their atoms and trick their delegators
* The owner needs some way to unbond everyone if eg. their hard drive burns and they lose their private key

## Design

There is a new struct called a `BondInfo` that defines all the global fields for the currency itself

```Go
type BondInfo struct {
  Name string // unique name to refer to this

  // Total*Value is the weight of the validator in atoms
  Total int64 // how many bonds exist
  Value int32 // 1,000,000 = 1 atom, decimal arithmetic

  Owner []byte // address of the owner who gets commissions
  Commision int32 // 1,000,000 = 100%, decimal arithmetic
  PledgedBond int64 // how many bonds the owner pledges to delegate

  // ShutDown time to unbond all safely???
}
```

The owner can set a commission fee to get liquid atoms. Every time a delegator bonds, they burn X atoms and create Y bonds in their account according to the Value of the bond.  The Total is also adjustred properly.

When a payout comes around, say a 10% bonus to this validator, first the commission is extracted, then the rest increases the value of the bond, rewarding all delegators with one simple operation.

Example:
There is a 20% commision, and 1,000,000 bonds, value is 1.2.
An interest payout of 10% comes around.  We apply this to the commision and the total value of the bonds: `interest * commission * bonds * value` and calcualte 24,000 atoms to be sent directly to the owner's account.  The remaining part increases the value of the bond `value = value * (1 + (interest * (1 - commission))`


On every account there is a new field:

```Go
type Account struct {
  Coins types.Coins //...
  Bonds Bond
}
```

The Bond must contain some fields...

```Go
type Bond struct {
  Name string
  Amount int64
  // TODO: unbonding times???
}
```

The following tx are allowed:

* bond -> immediately convert atoms to one bond at the given rate
* unbond -> mark X bonds to be unbonding, no longer count as power or get rewards
* cashout -> once the unbonding atoms have passed their time, then you can convert them to atoms at the (current rate, rate at unbonding start?? no interest, but yes slashes???)

* shutdown -> the owner can trigger unbonding for all atoms including delegates

Note: the owner cannot every have less than PledgedBond delegated, s/he can increase or decrease their bond, but only above that limit.  If they want to cash out completely, they need to invoke shutdown
