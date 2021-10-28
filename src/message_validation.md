# Message Validation

## Validation steps

 - Syntax validation: This should be mostly covered by the decoding
 - Semantic validation: Checks to make sure a message is valid in a given context (signature verification, epoch number check, etc.)
 - Group policy validation: checks about handshake type, etc.
 - AS/policy validation: Checks to see whether syntactically and semantically correct messages should be adopted or dropped (Is a member allowed to add another member? Is a member allowed to remove another member?)

## Detailed list of validation steps

 - MLSCiphertext:
   - group id must exist (usually checked by the DS already)
   - epoch must be within bounds
   - AAD can be extracted/evaluated
   - attempt decryption -> return MLSPlaintext and go to phase 2 (note that this modifies the SecretTree and is only guaranteed to work once, persistence)
 - MLSPlaintext:
   - Phase 1 (only if it was *not* decrypted from an MLSCiphertext)
     - group id must exist (usually checked by the DS already)
     - epoch must be within bounds
     - content_type must not be application
     - AAD can be extracted/evaluated
     - IFF sender is a group member, membership_tag must be present & successfully validated
   - Phase 2
     - IFF content_type is a commit, confirmation_tag must be present
     - Signature verification: Requires the AS (potentially blocking & persistence step might be required)
     - subsequent steps depend on content_type
 - Content type: Application
 - Content type: Proposal
   - Semantic checks:
     - Add Proposal: Double join
     - Remove Proposal: Ghost removal
     - Update Proposal: Identity must be unchanged
   - Operations like Add, Remove must be validated by AS: (potentially blocking & persistence step might be required)
 - Content type: Commit
   - all proposals (either inline or referenced) must be valid (see Content type: Proposal) potentially blocking & persistence step might be required)
   - Commit must not cover inline self Remove proposal
   - Path must be present, unless Commit only covers Add Proposals
   - Path must be the right length
   - Staging step: proposals must be applied to modify the provisional tree
   - Path must be applied and decrypt correctly
   - New public keys from Path must be verified and match the private keys from the direct path
   - Confirmation tag must be successfully verified

## API

*[temporary discussion doc](https://docs.google.com/document/d/1mtA7nqfh1v2guvh-7fGYEYAWaFDdLvf4i9n602RL0iE/edit?usp=sharing)*

```rust
impl ManagedGroup {
  fn parse_message(&mut self, mls_message: MlsMessage) -> Result<UnverifiedMessage, ManagedGroupError> {
    /*
     - epoch must be within bounds
     - AAD can be extracted/evaluated
     - decryption
     - IFF content_type is a commit, confirmation_tag must be present
    */
  }

  fn process_unverified_message(&self, message: UnverifiedMessage, signature_key: Option<SignatureKey>) -> Result<ProcessedMessage, ManagedGroupError> {
    /*
     - Signature verification, either with leaf key or optional parameter
     - IF Commit:
       - Extract all inline & pending proposals
     - Semantic validation of all proposals
       - IF Add Proposal: Double join check
       - IF Remove Proposal: Ghost removal check
       - IF Update Proposal: Identity must be unchanged
     - IF Commit:
       - Commit must not cover inline self Remove proposal
       - Path must be present, unless Commit only covers Add Proposals
       - Path must be the right length
       - Staging step: proposals must be applied to modify the provisional tree
       - Path must be applied and decrypt correctly
       - New public keys from Path must be verified and match the private keys from the direct path
       - Confirmation tag must be successfully verified
    */
  }

  fn store_pending_proposal(&mut self, pending_proposal: PendingProposal) -> () {
  /*
   - Store proposal in pending proposal list
  */
  }

  fn merge_staged_commit(&mut self, staged_commit: StagedCommit, psks: &[Psks]) -> Result<(), ManagedGroupError> {
    /*
     - Merge staged Commit values into internal group stage
    */
  }
}

enum MlsMessage {
  Ciphertext(MlsCiphertext),
  Plaintext(MlsPlaintext),
}

impl UnverifiedMessage {
  fn aad(&self) -> &[u8] {}
  fn credential(&self) -> &Credential {}
}

enum ProcessedMessage {
  ApplicationMessage(Vec<u8>),
  ProposalMessage(PendingProposal),
  StagedCommitMessage(StagedCommit),
}

enum PendingProposal {
  Add(PendingAddProposal),
  Remove(PendingRemoveProposal),
  Update(PendingRemoveProposal),
  Psk(PendingPskProposal),
}

impl StagedCommit {
  fn adds(&self) -> &[PendingAddProposal] {}
  fn removes(&self) -> &[PendingRemoveProposal] {}
  fn updates(&self) -> &[PendingUpdate] {}
  fn psks(&self) -> &[PendingPskProposal] {}
}
```

### Example API usage

```rust
// Parse either MlsPlaintext or MlsCiphertext
let unverified_message = group.pares_message(mls_message).expect("E1");

// Inspect AAD (optional)
if unverified_message.aad() == &[1, 2, 3] {}

// Inspect credential and fetch signature key from AS (optional)
let signature_key = AuthenticationService::get_key(unverified_message.credential());

let processed_message = group.process_unverified_message(unverified_message, Some(signature_key)).expect("E2");

match processed_message {
  ApplicationMessage(bytes) => {
    // Do something with application message.
    // No further interaction with the group is needed.
  },
  ProposalMessage(pending_proposal) => {
    // We can optionally inspect the proposal before we store it in the group:
    match &pending_proposal {
      Add(pending_add_proposal) => {
        // Do a policy check on that proposal
      },
      // Other type of propsals
      _ => {},
    }
    // After the optional inspection we store the pending proposal in the group
    group.store_pending_proposal(pending_proposal);
  }
  StagedCommitMessage(staged_commit) => {
    // We can optionally inspect all proposals covered by the Commit message before we merge it into the group:
    for add_proposal in &staged_commit.adds() {
      // Inspect add proposals here
    }
    // Inspect PSKs and get them from a store:
    let psks = staged_commit.psks.iter().map(|p| PskStore::fetch_psk(p)).collect();
    // Merge the staged commit into the group
    group.merge_staged_commit(staged_commit, psks).expect("E3");
  }
}
```

# Legacy stuff

### Validation function

The validation function has access to the inner state of a `ManagedGroup` and takes any inbound message from the DS as input. The messages are processed as follows:

#### Application messages

The message is checked and marked as either valid or invalid.

#### Proposals

The message is checked and marked as either valid or invalid. Proposals are stored inside a `ManagedGroup` and only evaluated when a Commit message is processed.

#### Commits

The message is checked and marked as either valid, invalid or pending validation. If the Commit message covers proposals that require validation by the AS, a `PendingValidationMessage` type message is returned. The list of operations in that message covers all relevant proposals that were previously received in the current epoch.
```rust
impl ManagedGroup {
    ...
    fn validate(&mut self, mls_message: MlsMessage) -> EvaluatedMessage { ... }
    ...
}

enum MlsMessage {
    Plaintext(MlsPlaintext),
    Ciphertext(MlsCiphertext),
}

enum EvaluatedMessage {
    ValidMessage(ValidPlaintextMessage),
    PendingValidation(PendingValidationMessage),
    InvalidMessage(InvalidMessageDetails),
}
```

### Valid messages

Valid messages are either automatically valid if they passed all checks, or can be converted from `PendingValidationMessage` by manual validation.

```rust
struct ValidPlaintextMessage(MlsPlaintext)
```

### Messages pending validation

Messages are marked as pending validation if input from the AS is required.

```rust
struct PendingValidationMessage {
    message: MlsPlaintext,
    operations: Vec<OperationType>,
}

impl PendingValidationMessage {
    fn validate(&self) -> ValidPlaintextMessage { ... }
}

enum OperationType {
    Add(AddOperation),
    Remove(RemoveOperation),
    Join(JoinOperation),
    Leave(LeaveOperation),
    Psk(PskOperation),
}

struct AddOperation { ... }
...
```

### Invalid messages

Messages that didn't pass the syntactic & semantic checks are marked as invalid with a reason for their invalidity.

```rust
struct InvalidMessageDetails {
    sender: Option<Sender>,
    reason: InvalidMessageReason,
}

enum InvalidMessageReason {
    InvalidGroup,
    InvalidGeneration,
    InvalidCiphertext,
    InvalidSignature,
    InvalidMembershipTag,
    DoubleJoin,
    GhostRemoval,
    InvalidUpdate,
    ...
}
```
