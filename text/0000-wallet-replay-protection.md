- Title: `wallet-(re)play-protection`
- Authors: Phyro, John Tromp, Yeastplume
- Start date: `Aug 9, 2020`
- RFC PR: Edit if merged: [mimblewimble/grin-rfcs#0000](https://github.com/mimblewimble/grin-rfcs/pull/0000)
- Tracking issue: [Edit if merged with link to tracking github issue]

---

## Summary
[summary]: #summary

Given certain conditions, UTXOs that belong to a user can be moved without their explicit permission. This document outlines the fixes that prevents this behaviour from occuring by introducing additional rules a wallet should follow.

## Motivation
[motivation]: #motivation

A new class of attacks was found on Grin that we call (re)play attacks. A Mimblewimble transaction that already happened can be replayed if the exact conditions are recreated. In order to be able to replay a transaction the following conditions must be true:
1. The inputs must be in the utxo set
2. The outputs of the transaction must not be in the utxo set (Grin does not allow duplicate outputs in the utxo set)

This means that if Alice sent some coins to Bob and the outputs have been spent, anyone that saw their original transaction could replay the transaction if the same inputs existed in the utxo set. But why would the same inputs exist on the chain in the first place? Alice could send those coins to Bob again, using different inputs but the same output for Bob, and trick Bob into thinking that this constitutes a new payment. If Bob accepts it as such, then Alice can replay Bob's spend of the recreated output at an opportune moment. Similarly, a _play_ attack comes from the same reasoning, but with the difference that a transaction never made it to the chain for some reason. One such reason could be that an input was spent before it was broadcasted which would make the current transaction invalid. In this document, we propose new wallet behaviours that make it robust in the face of (re)play attacks so the end users can't be victims of these attacks.


## Community-level explanation
[community-level-explanation]: #community-level-explanation

We propose that wallets follow the following 2 rules to fully protect users from (re)play attacks.

- Spend safely
- Cancel safely

An `anchor` is a 0-valued output (created by this wallet) that is never spent.
A safe (from this wallet's viewpoint) transaction is a transaction that either creates an anchor, or that spends an output from an earlier safe transaction.
Unsafe receives are allowed, but safe receives (that are generally payjoins) are preferred as long as the receiver gets to finalize, which minimizes the risk of utxo spoofing. Sends to others should always be a safe transaction (self-spends could be unsafe).

A safe cancel requires an immediate self-spend of an input of the tx to be canceled.

The wallet rules for spending safely would happen behind the scenes, so the end user would not be aware of any changes. Canceling a transaction would be a different experience than it is now, but it is required to change it to prevent the play attacks from occuring. A possible cancel flow would be to make the cancellation a longer event which is not finished until the self-spend transaction has been confirmed on the chain.  

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### General technical overview

As we have already mentioned, a transaction can be replayed if the inputs are recreated and its outputs don't exist. One way to stop this attack would be to make it impossible to recreate an input. Of course the owner can always recreate it, but the problem is really if anyone else recreates it by replaying the transaction that created the input. Luckily, there is a way to make an input that is protected and can't be recreated by anyone else except by the owner. This seems impossible because you can always just go one transaction back and recreate the outputs that created the inputs and keep repeating this until you hit a transaction that can be replay at which point you can replay all the next transactions and thus successfully recreate the target input. To prevent this, we create a 'stopping' transaction `T` for which we _know_ it can't be recreated. We achieve this by creating an output in `T` that we call an `anchor` output which will never be spent. Since Grin does not allow duplicate outputs, it means that the transaction that created `anchor` output can't be recreated because it would attempt to insert a duplicate output in the UTXO set which is invalid by Grin rules. Along with an `anchor` output we create additional outputs `O1` and `O2` in `T` which we call `Protected` outputs - it's not obvious why they have this name yet.

Let's say we have a transaction `T`
```
I1  -> anchor
    -> O1
    -> O2
```

We now use `O2` in a new transaction `T2` as an input to send money to Bob.

```
O2  -> O3 (change output)
    -> Bob_output
```

The key idea here is to observe that `T2` can't be replayed by anyone else. Only we can create the input `O2` needed for the transaction to be replayed. The attacker could attempt to replay the previous transaction `T` to create the  `O2` input, but `T` can't be replayed because it has an `anchor` output that has not been spent and hence can't be valid. This means that our transaction `T2` is protected against other parties replaying it. 

If we again used `O3` as an input in a new transaction `T3` to send money to Charlie, the same protection would be put in place. The attacker would need to replay `T2` to recreate `O3` input, but we have already shown that `T2` can't be replayed.

```
O3  -> O4 (change output)
    -> Charlie_output
```

This is the main idea used in this proposal to protect against replay attacks. This is why we name the outputs we created along side the `anchor` output `Protected` outputs. They provide protection for subsequent transactions we make if we use them as inputs to our transactions. Key thing to note here is that while we started with 2 protected outputs `O1` and `O2`, the outputs that we create in a transaction to which we add a protected input automatically become `Protected` outputs as well. This property is what allows us to continue creating new protected outputs without the need to create more than one `anchor` output.


### Wallet rules for replay protection

Most common transaction types are:
1. A `Regular` tx which has inputs from a single party - the sender
2. A `PayJoin` tx which has inputs from both parties - both sender and receiver contribute at least one input each

Let's call an output `Protected` if it is a part of the graph that leads back to an anchor output. A transaction is protected from replays if a party contributes a Protected output as an input or creates an anchor output. It's not enough that some other party does this. We are only really protected from replay attacks if we make the transaction safe from replays ourselves. This way, our safety does not depend on the honesty and following of the wallet rules of other participants.

If we will be doing both `Regular` and `PayJoin` transactions, then only some of our outputs will be protected. The first problem we have is that after a wallet restore, we can't tell whether an output is protected or not. It would be nice if we could label our outputs as either `Protected` or `Unprotected` after a wallet restore.

#### Output types/labels

We need a clear separation of outputs that are protected from those that are not and we need to be able to recognize an anchor output. Let's define three possible output types: `Protected`, `Unprotected` and `Anchor`. Each of the output types also has a function `create(amount)` with which we can create an output of that type that holds some amount of coins. An `Anchor` output is an exception because it has a form `0*H + r*G`, so the `create` function takes no amount value. Only the owner of the output should be able to see the output's type. For this reason we need a way for the owner to read the type of an output from the output itself.

##### Storing an output type

It seems necessary to have output type information on the output itself if we want it to be consistent with different wallets reusing the same seed.  Since we have 3 different types, we need at least two bits to store the type. We suggest storing the output type information in the available bits in the Bulletproofs where the currently unused bits are set to 0. This means that the bit representation of the `Unprotected` type needs to be `00` to account for all the previous unprotected outputs. Mapping of other types to bits is left to the wallet implementation. The message that is encoded in the Bulletproof is already private so only the owner would be able see the type of an output.

_NOTE: Solutions that use wallets local storage to store the output types lose track of the types on a wallet restore or when the same seed is used across multiple devices. Solving this problem is non-trivial as it requires either storing and syncing type information between wallet instances or user manually assigning types to their outputs, both of which are much less friendly._

#### Wallet transaction rules

##### Replay protection with utilization of an anchor

Whenever we want to transact safely, but lack a confirmed protected output (we might have unconfirmed ones), we create another (possibly the first) anchor and some configurable number of protected change outputs. An anchor output is created from our `Anchor.create` implementation. Our set of newly created `Protected` outputs is generated from our implementation of `Protected.create` which labels the outputs as protected.

_Note: Grin has a rule that an output that already exists in the UTXO set cannot be created - a transaction that would attempt to do that is invalid. This is why transactions that contain an anchor output can't be replayed. All the outputs that were created along the anchor output are also safe from being recreated through a replay which allows us to use them as new protected outputs._

If we are worried that the number of created outputs could hint that we are creating an anchor, we can create an anchor in an isolated transaction that gets aggregated with the transaction we want to protect. For receiving transactions this would mean we need to contribute some inputs, outputs and a kernel to simulate an aggregated transaction. Since we only need to have send transactions safe, we never really need to create an anchor in a receiving transaction. If we have no protected outputs available and we want to create a safe send transaction, we can first create an _unsafe_ send transaction and in parallel create another self-spend transaction that creates an anchor output. An unsafe send transaction should not be broadcasted. We can aggregate the unsafe send transaction with the self-spend transaction that creates an anchor to obtain a safe aggregated transaction which can be broadcasted. This requires at least another unprotected output available that can be used as an input in the anchoring transaction and costs slightly more fees.

##### Replay protection with utilization of Protected outputs

If we have a protected output available, we can make a transaction safe by adding a protected output as an input to the transaction. This prevents any malicious replay attacks. As we already mentioned, receives can be made through unsafe transactions so we really only need to make safe transactions when we are sending money to someone because these are the transactions that move the coins away from us. Self-spends are an exception and can be left unsafe because we don't really mind them being replayed since the transaction doesn't give coins to anyone else.

This means that:
1. We can send money through a regular 1-2 transaction if our input is of type `Protected`
2. We can receive money through a regular 1-2 transaction

The only thing we need to be aware is that our output that is created in a 1-2 receive transaction will be labeled as unprotected and will hence need to be spent in a safe transaction. A more general rule is that unprotected outputs need to be spent in a safe transaction (except self-spends) - in most cases this means along side some of our `Protected` inputs. In theory, it should be impossible to replay a transaction that followed this rule.

_Note: If we are the sender in a 1-2 transaction where we use a `Protected` output as an input, we create a change output which is also a `Protected` output. Since we spent 1 protected input and created another one, it means that the sender can chain such 1-2 transactions safely._

This protects the user from all malicious replay attacks and it can happen behind the scenes without the user knowing there are different types of outputs.

Wallet pseudo-code:
```rust
// The general idea is to also have at least a handful of protected output available most of
// the time. Might want to have a dozen of these in case of high number of transactions being
// performed which would be useful for parties that perform concurrent transactions e.g. exchanges

// Suppose we have outputs separated by type and the following interfaces available:
// struct Protected {
//   create(value) -> Output         // Creates a new protected output holding 'value' coins
//   create_multi(value) -> []Output // Create multiple protected outputs with the same value
//   outputs() -> {}Output           // A set of outputs of type Protected and sufficiently confirmed 
// }

// Create a safe transaction by creating an anchor and an initial set of protected outputs PS
fn anchor_tx() -> ([]Output, []Output) {
  // It's not needed to create any protected output because the anchor output will always be joined
  // by another output we own - change output as a sender and value output as a receiver. This
  // output will be protected because of the anchor
  anchor = Anchor.create()        // generates an anchor
  PS = Protected.create_multi(0)  // generates a few protected 0-value outputs
  return [], [anchor] + PS
}

// Returns inputs and outputs that should be added to make a transaction safe. In case
// there is no available protected output, it creates an anchor and PS outputs. If we
// have a protected output available, we make it a safe transaction by adding a protected
// output as an input
fn safe_tx() -> ([]Output, []Output) {
  // If we have no available protected output, contribute anchor + PS outputs for replay protection
  // NOTE: this might be too obvious from the chain analysis perspective. If we are worried about this
  // we could decide to create anchors in a separate transaction that gets aggregated with our
  // transaction to obtain a safe aggregated transaction.
  if !Protected.outputs() {
    // Safe transaction with an anchor
    return anchor_tx()
  }
  // Safe transaction with a protect output as an input
  return [Protected.outputs.pop()], []
}

// Receive txs don't need to be safe so we flip a coin to determine whether
// it will be a regular or safe transaction based on the wallet configuration
// receive-only wallets could have safe_receive_prob=0.0 to always have 1-2 receive transactions
fn receive(value: int64) -> ([]Output, []Output) {
  is_safe = random() <= config.safe_receive_prob
  if !is_safe{
    // If the receive transaction is unsafe, then we simply add an unprotected output
    return [], [Unprotected.create(value)]
  }
  // A safe receive is just a safe tx to which we add a protected output that holds our received value
  receiver_inputs, receiver_outputs = safe_tx()
  // If we have added a protected output as an input, we need to add its value to the value output
  new_value = value + sum(input.v for input in receiver_inputs)
  // We add a protected output to a safe transaction regardless whether it has an anchor or
  // a protected input (safe PayJoin)
  return receiver_inputs, receiver_outputs + [Protected.create(new_value)]
}

// A 'send' transaction needs to always be protected from replays so it MUST be a safe transaction.
// We need to either add a protected output as an input or create an anchor if we have no available
// protected outputs
fn send(value: int64) -> ([]Output, []Output) {
  // Start with a safe tx
  inputs, outputs = safe_tx()
  // Add the inputs whose sum of values is enough to perform the tx - favor unprotected inputs
  inputs = pick_inputs_sum(inputs, value)  // we pick additional inputs to get the sum coins >= value
  change_value = sum(input.v for input in inputs) - value  // fee is missing from here
  // Add a protected change output - send is always a safe tx
  change_output = Protected.create(change_value)
  outputs += [change_output]
  return inputs, outputs
}

// NOTE: There are many possible strategies that can keep the number of protected outputs
// higher than the specified n_protected_outputs configuration. These can be created by:
// 1. doing self-spend 1-2 transactions (more private but comes at a cost of tx fees)
// 2. adding new outputs to existing transactions
// 3. attaching outputs to transactions that pass by and have overpaid the fees
// The strategy choice is left to the wallet implementation.

// There are a number of improvements that can be made with regards to picking specific protected
// outputs e.g. we might want to pick low value outputs as protected inputs in a receive transaction
```

##### Protection with wallet history

A wallet can keep a history of spent outputs. This way, if a spent output reappears, the default wallet behaviour could be to ignore such output and not count it in the balance. Wallet configuration could allow users to see these outputs and decide to either accept it or refresh it through a self-spend transaction.

The downside of this approach is that malicious replay attacks are still possible in which case it would mean that a user would need to make a manual choice what to do with outputs that are unknown to the history of the wallet. Unknown outputs are also those created from the same seed on a different device and from possible child wallets since they each have their own history. The cross device scenario could be mitigated by export and import of wallet history.

#### Receive-only wallets
 
Any kind of automated receiving should default to 1-2 transactions and thus creating _unprotected_ outputs to avoid utxo spoofing attack which would reveal our inputs. Always performing 1-2 receive transactions can be achieved by setting the configuration `safe_receive_prob = 0.0` which is explained in the next section.

#### Transaction building configuration

```yaml
// Defines a list of wallet configurations to define different behaviour that we do as a receiver of a
// transactions. All wallet configurations inherit the values from the 'default' option that can be
// overriden. A wallet should allow the user to generate a one-time SlatepackAddress based on a chosen
// configuration. 
transaction_building:
  configs:
    - name: default
      // Probability that we will protect a 'receive' transaction from replay attacks. If an anchor does not
      // exist, the transaction will be protected by generating an anchor and an initial set of protected outputs.
      // If such an anchor already exists, we contribute a protected output as an input which makes it a PayJoin
      // transaction. By default, half of the receive transaction will be PayJoin transactions. This is to help mask
      // the sender inputs. Additionally, PayJoin transactions are cheaper and more healthy for the network because
      // they don't generate new outputs.
      // If PayJoin transactions are not wanted due to privacy concerns, you can set this value to 0.0 in which case
      // receive transactions will never contribute an input.
      // Default: safe_receive_prob: 0.9
      safe_receive_prob: 0.9

      // The number of protected outputs we want to have available at any time. This is to be able to have concurrent
      // transactions that are safe from replay attacks. If we had only 1 protected output and wanted to make two
      // 'send' transactions, we would need to do them sequentially because we could only protect a single
      // transaction at once. This defines the minimal set size of protected outputs. Exchanges would want to set
      // this to higher values to avoid blocking transactions.
      // Default: n_protected_outputs: 5
      n_protected_outputs: 5

    // PayJoins configuration
    - name: all_payjoins
      safe_receive_prob: 1.0

    // No PayJoins configuration - e.g. could be used for withdrawals from exchanges or other untrusted parties
    - name: no_payjoins
      safe_receive_prob: 0.0
    
  // We can predefine some fixed transaction building configurations for specific addresses, but we should note
  // that this comes at a privacy risk due to the SlatepackAddress reuse.
  address_configurations:
    grin1dhvv9mvarqrqwerfderuxp3qgl6qpphvc9p4u24asdfec0mvvg6342q4w6: default
    grin1hhb34mfsrqwlfderuxpfdql6qpphvc9p4u24fd47ec0mvvg6342q1asdxc: all_payjoins
    grin1fsdf9fd83e3fjvcxruxp3qgl6qpphvc9p4u24347ec0mvvg6342fdoiosl: no_payjoins
```

_NOTE: it is highly recommended that all payjoin transactions require a manual confirmation from the receiver. This should limit the scope of the UTXO spoofing attacks which allow the sender to find out all the inputs of the receiver - something that is possible with automated payjoins. Having all of the transactions require manual confirmation prevents also dusting attacks. Perhaps the most important advantage of manually confirming the transactions the user wants to receive is that the user gains a complete control over what they have in the wallet - something that can't be guaranteed without interactivity because it requires the receiver confirmation in some form which makes it interactive. It is possible to limit the UTXO spoofing attacks without manual confirmation if the SlatepackAddress can be used only once._


#### Wallet accounts

Each wallet account should have its  `anchor` outputs protecting it.

#### Exchanges scenarios

##### Exchange configuration

As mentioned in the configuration section, exchanges would likely need a bigger set of protected outputs so they would need to adjust the `n_protected_outputs` configuration. It would be encouraged that exchanges have `safe_receive_prob` set to `1.0` to help mask the user input.

##### User withdrawing from an exchange configuration

We probably don't want to be doing a PayJoin transaction when we are withdrawing from an exchange because we'd be showing them one of our inputs. To avoid this, we can generate a `SlatepackAddress` from a configuration option that has 0% chance to do a PayJoin receive transaction.

#### PayJoin transactions

PayJoin transactions are one possible way to break the [common-input-ownership heuristic](https://en.bitcoin.it/wiki/Common-input-ownership_heuristic). They achieve this by having the receiver contribute some inputs to the transaction. It might seem like payjoin transactions only benefit the sender while hurting the privacy of the receiver, but the receiver also wants to spend outputs in an input set with a mixed ownership. Each transaction is an opportunity for the receiver to spend outputs on the input side that will have mixed owners. Doing so comes at a cost of showing the contributed receiver's inputs (usually just one) to the sender, but also at a benefit of having multiple owners of inputs which should in most cases be in the interest of both the sender and the receiver. Important thing to note here is that if there are very few or no payjoins happening on the network, this same receiver could in a future transaction spend multiple inputs together and show their spent inputs not only to the transacting party, but to everyone else as well, which seems like a worse privacy tradeoff than doing payjoins. Spending multiple inputs together is not uncommon, so we want most of the transactions be payjoins to help us obfuscate our own inputs when we spend them. A positive side effect of doing a payjoin as a receiver is that we consolidate some of the outputs to create an output that holds more coins, so we are less likely to come in a situation where we need to spend multiple inputs together. Having the majority of the transactions be payjoins is also beneficial for the whole network because it creates probabilistic input ownership which makes backwards chain analysis _much_ harder to do since we can't tell whether an input belongs to the sender of the receiver.

Currently, PayJoin transactions are cheaper than regular transactions because they have more inputs. A 2-2 PayJoin transaction does not create a new output as opposed to a regular 1-2 transaction, which means that it can be thought of as an replacement of the value of the current output with a new value which does not increase the UTXO set size.

_PayJoin transactions also allow for the possibility of the receiver paying their share of fees. Whether users would find this useful is not clear yet._

#### Play attack protection

The difference between a Play and Replay attack is that in a Play attack, the transaction never lands on the chain. Suppose Alice wants to pay Bob, but after constructing the transaction and broadcasting it, the transaction does not land on the chain for some reason. Alice and Bob recreate a new transaction where Alice uses different inputs. This second transaction goes through and both Alice and Bob are happy. However, if Bob saw the first transaction, if it becomes valid at some point, then it's possible for him to 'play' it or broadcast it on the chain after the second transaction was already confirmed on the chain. This way, Bob receives his payment twice. A Play attack attacks the sender by tricking them into signing a transaction multiple times. Transactions that don't make it to the chain should _always_ be cancelled by the user before sending a new transaction.

To protect against this, a user should have an option to cancel a transaction which would immediately create a self-spend transaction with the same inputs that the canceled transaction used and broadcast it to the network. The reason why the sender would want to reuse the inputs is to make the previous transaction invalid. So now, only 1 of the two transactions that have been signed can make it to the chain. As already mentioned, the cancel would not be confirmed until the self-spend transaction has been confirmed on the chain.

It is advised to have a detection mechanism to catch a potentially malicious attempt from the receiver. A wallet needs to remember the inputs and outputs that the receiver contributed to the transaction and check if our transaction kernel does _not_ exist on the chain and at least one of the two is true:
1. At least one of the receiver contributed outputs appears on the chain - transaction is invalid due to duplicate outputs being forbidden
2. At least one of the receiver contributed inputs is spent - transaction is invalid due to an input being already spent

If our transaction is not present and any of the two conditions is true, then our transaction is invalid and we should immediately spend the inputs of the transaction. On a transaction cancel, a wallet could present the user with an option to replace the transaction with another payment. Similarly, if the user initiated a new payment to a user for which a transaction is already pending, it could offer the user an option to respend the transaction. Both of these options would not produce an additional kernel because the inputs are used in a non self-spend transaction.

_Note: If we make another transaction to the same user, we should either wait for the cancel transaction to get enough confirmations or use the new outputs created in the cancel transaction (self-spend) as inputs for this new transaction. This is to avoid a possible short reorg attack which could remove the self-spend transaction and publish both transaction which would result in a double-spend._

## Drawbacks
[drawbacks]: #drawbacks

### Replay attacks solution

It requires an `anchor` input that is never spent which increases the chain 700 bytes per wallet. These outputs might be easier to identify because they never move. How easy/hard would it be to identify them is unclear. Most wallets are expected to have only a few such outputs and a lot of wallets will get lost and hence a lot of outputs will never move.

### Play attacks solution

Cancelling a transaction now takes some time because the user should wait for the self-spend to be confirmed. Adding another transaction on the chain for every cancelled transaction means that a cancel costs fees and adds a kernel on the chain. The side effect is a bit more kernel bloat, though that's very likely to be negligible and could be avoided by offering the user to respend the transaction.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

There are also other ideas on how to protect against replay attacks that solve the problem at the consensus level. Both wallet and consensus level solutions have their own tradeoffs. The main benefit of solutions at the wallet level is that they can't introduce a consensus failure and because they are only a change in the wallet behaviour, they can later be replaced by a consensus level solution if needed.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

- Is it worth implementing other solutions as well (e.g. output history + sweeping) which seem to have more problems and introduce complexity to wallet handling?
- Should the user have a transaction configuration option that, when enabled, would require a manual confirmation of the receiving transactions? The user manually confirming the outputs they would receive prevents dusting attacks (regardless of the cost) and severely limits any other utxo spoofing methods. It also allows the user to have full control over what outputs they will have in their wallet.

## Future possibilities
[future-possibilities]: #future-possibilities

The options of transaction building configuration and programmability are only limited by our imagination. PayJoins could be researched a bit more and used more often if we don't interact with parties that are known to collect data on their users.

## References
[references]: #references

[Replay attacks and possible mitigations](https://forum.grin.mw/t/replay-attacks-and-possible-mitigations/7415)

[PayJoins for replay protection](https://forum.grin.mw/t/payjoins-for-replay-protection/7544)