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

User has to fix the fee issue via fee bumping. How to bump to fee highly depends on how the miners prioritize transactions for the next block.

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