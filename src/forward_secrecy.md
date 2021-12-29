# Forward Secrecy

To achieve forward secrecy, OpenMLS drops key material immediately after a given
key is no longer required by the protocol. For some keys this is simple, as they
are used only once and there is no need to store them for later use. However,
for other keys, the time of deletion is a result of a trade-off between
functionality and forward secrecy. For example, it can be desirable to keep the
`SecretTree` of past epochs for a while to allow decryption of straggling
application messages sent in previous epochs.

In this chapter, we detail how we achieve forward secrecy for the different types of keys used throughout MLS.

## Ratchet Tree

The ratchet tree contains the secret key material of the client's leaf, as well
(potentially) that of nodes in its direct path. The secrets in the tree are
changed in the same way as the tree itself: via the merge of a previously
prepared diff.

### Commit Creation

Upon the creation of a commit, any fresh key material introduced by the
committer is stored in the diff. It exists alongside the key material of the
ratchet tree before the commit until the client merges the diff, upon which the
key material in the original ratchet tree is dropped.

### Commit Processing

Upon receiving a commit from another group member, the client processes the
commit until they have a `StagedCommit`, which in turn contains a ratchet tree
diff. The diff contains any potential key material they decrypted from the
commit, as well as any potential key material that was introduced to the tree as
part of an update that someone else committed for them. The key material in the
original ratchet tree is dropped as soon as the `StagedCommit` (and thus the
diff) is merged into the tree.
