---
created: 2020-08-21
updated: 2020-08-27
author: ian yang
tags: ckb
---

# CKB Transactions Management Guideline

[![hackmd-github-sync-badge](https://hackmd.io/lA8KVycmQU6oqWjPzQZNYA/badge)](https://hackmd.io/lA8KVycmQU6oqWjPzQZNYA)

UTXO transactions management is hard because they may be stuck in or even dropped from the CKB node transactions pool when the fee is too low.

This guideline is for any transactions generator which wants to trace the transaction status before finally confirmed in the chain.

## Transaction States

The transaction lifecycle starts after creation and ends when it is finally confirmed or conflicts with a confirmed transaction. During the lifecycle, the transaction is in one of the following states.

* Pending
* Confirming
* Confirmed
* Conflicting
* Conflictive
* Reverted
* Abandon

The list does not include orphan transactions that depend on unknown cells, because transactions generator can ignore them.

Different states require different strategies as described in the following chapters.

![transaction-state-diagram](https://raw.githubusercontent.com/r763/uPic/master/202008/IERrxl/transaction-state-diagram.png)

### Pending Transaction

The newly created transaction is in the Pending state.

The generator must store the Pending transactions locally and send them to CKB nodes at regular intervals. It's essential because CKB nodes may drop the transactions in their pools. The generator is not recommended to use the output cells of the pending transactions unless the transaction is stuck and requires fee bumping. If the generator does need chaining pending transactions, it must limit the length of the pending transactions chain.

Once the transaction appears in a block in the canonical chain, it turns to Confirming.

If the Pending transaction or any or its ancestor transaction[^1] has conflict input cells with a transaction in the canonical chain, it becomes conflicting.

If a user or an app decides to give up a stuck transaction, they can mark the Pending transaction is Abandon.

### Confirming Transaction

The Confirming transactions are found in the chain but are not qualified as Confirmed yet.

The generator must store the Confirming transactions locally but does not need to send them to CKB regularly. Same as Pending transactions, it is not recommended to use the output cells of Confirming transactions.

The block containing the transaction gives one confirmation. Each descendant block in the chain gives an extra confirmation. Although any block can be reverted in theory, it is safe to assume that the transaction will never be reverted after X confirmations, where X depends on various factors. The generator has to decide the value of X and adjust it regularly.

The Confirming transaction becomes Confirmed when it gets at least X confirmations.

If a chain reorganization occurs and the block containing the transaction is reverted, the transaction transforms into Pending.

### Confirmed Transaction

A Confirmed transaction is in the chain and has at least X confirmations.

The generator can remove Confirmed transactions from local storage or keep the latest ones as a history for reference. The generator can use the output cells of Confirmed transactions freely.

If a chain reorganization occurs and the block containing the transaction is reverted, the transaction becomes Reverted.

### Conflicting Transaction

A transaction is in the state Conflicting if it conflicts with any transaction in the chain. Two different transactions conflict when

1. They have conflict input cells.
2. A transaction has conflict input cells with any ancestor of another transaction.
3. They are descendants of a pair of conflict transactions.

The generator must store them in the local storage because there is a chance they may become Pending. The generator should not use the output cells of Conflicting transactions.

If all the blocks containing the conflict transactions have been reverted, the Conflicting transactions become Pending again.

If any conflict transaction gets X confirmations, the Conflicting transactions turn to Conflicted. See State "Confirming" for the discussion of X.

### Conflictive Transaction

A Conflictive Transaction conflict with an on-chain transaction that gets at least X confirmations. See State "Confirming" for the discussion of X.

The generator can remove Conflictive transactions from the local storage or keep recent ones for reference. The generator should not use the output cells of Conflictive transactions.

If all the blocks containing the conflict transactions have been reverted, the Conflicted transactions become Pending again.

### Reverted Transaction

State Reverted is an alias of Pending, it is a prompt that the transaction has ever been Confirmed, but is reverted.

The generator must store and send Reverted transactions regularly. The state transformations are the same as Pending.

### Abandon

The generator should allow users to mark a Pending or Reverted transaction as Abandon. The generator stops sending the Abandon transactions to CKB nodes. Nothing can stop a block containing an abandoned transaction. It just means the user has given up the efforts to commit the 
transaction.

The generator must store enough time range of recent Abandon transactions. The generator can clean the old Abandon transactions only when the disk storage is a concern. The reason is mentioned in the RBF fee bumping method below.

User is allowed to mark an Abandon transaction to Pending.

Abandon transactions transform to Confirming or Conflicting just like Pending and Reverted transactions do.

[^1]: If a transaction A is a parent of B if B uses an output cell of A as its input cell. An ancestor is either the parent or an ancestor of any parent.