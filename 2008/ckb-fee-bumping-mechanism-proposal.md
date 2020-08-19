---
created: 2020-08-19
updated: 2020-08-19
author: ian y.
---

# CKB Fee Bumping Mechanism Proposal

[![hackmd-github-sync-badge](https://hackmd.io/TzsvOkzmTwG4LHzIs_pDRg/badge)](https://hackmd.io/TzsvOkzmTwG4LHzIs_pDRg)

As transaction volume continues to increase, so does the demand for block space, which is limited by both transaction serialized size and consumed cycles in CKB.

User pays a fee as the bidding to the block space. Setting the fee too low may cause the transaction delayed or never been confirmed.

Which makes the matters worse, it seems unlikely that user will always set a proper fee. There are mainly two reasons. First, The fee estimate algorithm can not predict the future fee surge caused by sudden congestion . The second, in some applications such as lighting network, the transaction was created long before it is revealed.

User has to fix the fee issue via fee bumping. This article is a proposal of the fee bumping mechanism for CKB.

## How Miners Prioritize Transactions

The miners select transactions to fill the limited block space which give the higest fee. Because there are two different limits, serialized size and consumed cycles, the selection algorithm is a [multi-dimensional knapsack problem](https://en.wikipedia.org/wiki/Knapsack_problem#Multi-dimensional_knapsack_problem).

### Virtual Bytes

Introducing the transaction virtual bytes converts the multi-dimensioanl knapsack to a typical knapsack problem, which has a simple greedy algorithm.

Using the following notations

* $L_s$ is the block serialized size limit.
* $L_c$ is the block consumed cycles limit.
* $s_t$ is the serialized size of a transaction $t$, and
* $c_t$ is its consumed cycles.

The virtual bytes $b_t$ is

\\[
b_t = L_s \times \max(\frac{c_t}{L_c}, \frac{s_t}{L_s}).
\\]

The greedy algorithm sorts the transactions in decreasing order of fee rate, $R_t = F_t/b_t$ in which $F_t$ is the fee paid by the transaction $t$. It then proceeds to insert them into the block as long as there are remaining space of both serialized size and consumed cycles.

### Ancestor Weight

A transaction A is parent of another transaction B if at least one of B's input cells is A's output cell. B is called the child transaction of A.

An ancestor of a transaction is either its parent transaction, or an ancesor of any its parent.

Miners are not allowed to add a transaction to the next block unless all its ancestors are already in the chain or will be added to the next block together.

Instead of sorting the transactions by fee rate and try them one by one, miners have to process transactions in packages.

The ancestor package $A[t]$ of a transaction $t$ is a set which includes itself and all its ancestors remaining in the pool. The weight of the ancesor package $W_{A[t]}$ is the lesser of the transaction fee rate and the package fee rate as defined in the following fomular, where

* $b_t$ is the virtual bytes of a transaction $t$.
* $F_t$ is the fee it pays.

\\[
W_{A[t]} = \min(\frac{F_t}{b_t}, \frac{\sum_{i \in A[t]}F_i}{\sum_{i \in A[t]}b_i})
\\]

### Descendant Weight

---

Survey Outline

- Terminology
    - Virtual bytes
        - The virtual bytes of a transaction is the greater of its serialized size in bytes and its consumed cycles multiplying 0.000,170,571,4
    - Fee rate
        - Fee rate = fee / virtual bytes
    - Parent and child transaction
        - If transaction P uses an output of transaction P as its input, P is a parent transaction of C, C is a child transaction of P.
    - Ancestor transaction
        - A parent transaction is also an ancestor transaction.
        - If a transaction A is an ancestor of a transaction D, all the parents of A are also ancestors of D.
    -  Descendant transaction
        - A child transaction is also a descendant transaction.
        - If a transaction D is a descendant of a transaction A, all the children of D are also descendants of A.
    - Ancestor package
        - The ancestor package of a transaction includes the transaction itself and all its ancestors that are in the mempool.
    - Descendant package
        - The descendant package of a transaction includes the transaction itself and all its descendants that are in the mempool.
    - Ancestor score
        - The ancestor score of a transaction is the lesser of its fee rate and its ancestor package fee rate
    - Descendant score
        - The descendant score of a transaction is the greater of its fee rate and its descendant package fee rate
- Mempool
    - If a transaction is in the pool, any of its parent is either in the chain or in the pool.
    * Mempool capacity
    * Mempool priority
    - Mempool requirements
        - Light client synchronizes pool.
        - Sanity check on fee when submitting transaction via RPC.
        - It allows bypassing pool limit during reorg
        - Time locked transactions (because of maturity or since) can be added to pool 2 blocks in advance.
            - Because the future epoch length and timestamp is unpredicatble, sometimes it’s impossible to propose the transaction in advance.
        - After verified an invalid transaction, unload cache for its referenced cells if these cells are just loaded for this transaction.
        - Double spend verification cache
            - Check mempool first, then chain.
        - Reorg has to check the time locked transactions (since, header dep, cellbase)
            - The verification result is cached, just need to ensure time lock verification passes in the new block.
            - After reorg, the transactions removed from chain can be added to the pool, but ensure the time lock verification passes for the next block.
        - Allow prioritize a transaction by modify the fee manually via RPC.
        - Mempool entry info
            - Add fields about ancestor and descendant package information
            - Allow getting entry info of all ancestors/all descendants
        - Dry run to get fee, cycles, bytes, virtual bytes, fee rate info
        - Mempool transaction events subscription
        - Allow user to rebroadcast or paste the transaction in 3rd party service to rebroadcast on regular internval
- Two Fee Bumping Methods
    - The weight of a transaction depends on its own fee rate and its ancestor or descendent package fee rate.
        * CPFP, Child Pays For Parent
        - RBF, Replace By Fee
            - RBF replace the existing transactions in the pool by creating a transaction which has conflict inputs.
            - Bitcoin allows opt-out of transaction replacement by setting nSequence > MAX_BIP125_RBF_SEQUENCE (SEQUENCE_FINAL-2) on all inputs.
            * Limitation and workaround
    - There are other bumping methods, but bitcoin core rejects them.
        https://en.bitcoin.it/wiki/Replace_by_fee
        - First-Seen Safe RBF
            https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2015-May/008248.html
        - Opt-in RBF
        - Delayed RBF