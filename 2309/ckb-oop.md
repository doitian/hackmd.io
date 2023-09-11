# Ideas on A General OOP Programming Model on CKB

## dApp Programming on UTXO is Hard

Separating the generator and verifier is a premature optimization. Typically, using a generator to recreate the result and check whether it matches the given result is enough in most situations. However, attempting to separate the generator and verifier can lead to unnecessarily complex design.

Another issue that arises is the signature, which can make it difficult for users to understand the transaction. The situation becomes even more complicated when working on the partial signing proposal, in which users must sign selected fields in a transaction. Our observations indicate that properly implementing partial signing can be challenging, and can increase the disparity between what the transaction actually does and what the user is signing.

## The New OOP Idea

These issues lead to an idea: Why not let users sign their intentions as messages sent to a cell? This approach is similar to Object-Oriented Programming (OOP), where users can sign and send messages to cells. The lock script can then verify the signature, and the typescript can handle message processing. During message processing, side effects such as creating, modifying, or deleting a cell can occur, and these ultimately determine the transaction outcome in a deterministic way.

The generator uses these side effects to create transactions in a deterministic manner. The verifier then recreates the transaction and verifies that it matches the original transaction.

```python
# Message: transfer(receiver, amount)
# Replay protection: sign the out point or nonce.
# Lock script: verify signature of the message

# Typescript:
def transfer(self, receiver: address, amount: u256):
  assert(self.amount >= amount)

  sdk.create_cell(
    Self,
    address=receiver,
    amount=amount
  )

  self.amount -= amount
  if self.amount == 0:
    sdk.delete_cell(self)
```

In the example above:

- `create_cell` adds an output
- `self.amount -= amount` modifies an cell. Since `self` is an input cell, this will create an output cell with the modified data.
- `delete_cell` will remove the cell. The cell `self` has been modified and added to the output, `delete_cell` removes it from the outputs.

There are currently open issues around how to handle the CKB capacity provider and change output.

## Borrowing Ideas from Move

We treat every cell as a first-class object and can borrow concepts from the Move language.

We can modify a cell, which will create an output for the input cell. Alternatively, we can delete the cell, which removes it from outputs and returns associated CKB. If we do not manipulate the cell, an output identical to the input cell is created. We can also create a new cell, which generates a new output.

All cells support native operations to handle the underlying CKB capacities.

We can create references to cells, and even save those references in cell data. The generator use references to find live cells to create the transaction, and the verifier locates the matched live cells in the transaction to determine the message receiver.

There are different ways to reference a cell, such as:

- Hard referencing, which uses the cell out point.
- Soft referencing, which utilizes the indexer pattern to locate any cells that match specific criteria. It must supports light client to find the matched cells.
- Local referencing, which targets cells within the transaction.

A cell can send messages to other cells, and the message handling method is determined by the cell's type.

The cell type also defines the ACL, which specifies that who can send messages. Some messages are private and require the owner's signature, whereas others are public and can be called by other cells. Some are internal and only available to the cell type's typescript.

There are three special messages in particular: new, update, and delete. The ACL on these messages specifies who can create, update, and delete the cell.

## Transaction Global Execution

All message processing is delegated to the first input. It's recommended to use the same type script group for all OOP cells; as such, the OOP type should be stored in either the lock script arg or in the data. Another solution would be to bypass the OOP typescript when it doesn't include the first input, but this still causes a wastage of VM resources. TODO: why?

## Similar Works

- Kuai Framework
- [Run Framework on Bitcoin](https://web.archive.org/web/20230328120426/https://run.network/docs/)
- [Programming Objects Tutorial Series | Sui Docs](https://docs.sui.io/build/programming-with-objects)