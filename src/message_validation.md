# Message Validation

## Validation steps

 - Syntax validation: This should be mostly covered by the decoding
 - Semantic validation: Checks to make sure a message is valid in a given context (signature verification, epoch number check, etc.)
 - Group policy validation: checks about handshake type, etc.
 - AS/policy validation: Checks to see whether syntactically and semantically correct messages should be adopted or dropped (Is a member allowed to add another member? Is a member allowed to remove another member?)

## Detailed list of validation steps

### Semantic validation of message framing

| ValidationStep | Description                                                 | Implemented | Tested | Test File                                    |
| -------------- | ----------------------------------------------------------- | ----------- | ------ | -------------------------------------------- |
| `ValSem1`      | Wire format                                                 | ✅          | ✅     | `openmls/src/group/tests/test_validation.rs` |
| `ValSem2`      | Group id                                                    | ✅          | ✅     | `openmls/src/group/tests/test_validation.rs` |
| `ValSem3`      | Epoch                                                       | ✅          | ✅     | `openmls/src/group/tests/test_validation.rs` |
| `ValSem4`      | Sender: Member: check the sender points to a non-blank leaf | ✅          | ✅     | `openmls/src/group/tests/test_validation.rs` |
| `ValSem5`      | Application messages must use ciphertext                    | ✅          | ✅     | `openmls/src/group/tests/test_validation.rs` |
| `ValSem6`      | Ciphertext: decryption needs to work                        | ✅          | ✅     | `openmls/src/group/tests/test_validation.rs` |
| `ValSem7`      | Membership tag presence                                     | ✅          | ✅     | `openmls/src/group/tests/test_validation.rs` |
| `ValSem8`      | Membership tag verification                                 | ✅          | ✅     | `openmls/src/group/tests/test_validation.rs` |
| `ValSem9`      | Confirmation tag presence                                   | ✅          | ✅     | `openmls/src/group/tests/test_validation.rs` |
| `ValSem10`     | Signature verification                                      | ✅          | ✅     | `openmls/src/group/tests/test_validation.rs` |

### Semantic validation of proposals covered by a Commit

| ValidationStep | Description                                                                                 | Implemented    | Tested | Test File |
| -------------- | ------------------------------------------------------------------------------------------- | -------------- | ------ | --------- |
| `ValSem100`    | Add Proposal: Identity in proposals must be unique among proposals                          | ✅             | ❌     | TBD       |
| `ValSem101`    | Add Proposal: Signature public key in proposals must be unique among proposals              | ✅             | ❌     | TBD       |
| `ValSem102`    | Add Proposal: HPKE init key in proposals must be unique among proposals                     | ✅             | ❌     | TBD       |
| `ValSem103`    | Add Proposal: Identity in proposals must be unique among existing group members             | ✅             | ❌     | TBD       |
| `ValSem104`    | Add Proposal: Signature public key in proposals must be unique among existing group members | ✅             | ❌     | TBD       |
| `ValSem105`    | Add Proposal: HPKE init key in proposals must be unique among existing group members        | ✅             | ❌     | TBD       |
| `ValSem106`    | Add Proposal: required capabilities                                                         | ❌<sup>1</sup> | ❌     | TBD       |
| `ValSem107`    | Remove Proposal: Removed member must be unique among proposals                              | ✅             | ❌     | TBD       |
| `ValSem108`    | Remove Proposal: Removed member must be an existing group member                            | ✅             | ❌     | TBD       |
| `ValSem109`    | Update Proposal: Identity must be unchanged between existing member and new proposal        | ✅             | ❌     | TBD       |
| `ValSem110`    | Update Proposal: HPKE init key must be unique among existing members                        | ✅             | ❌     | TBD       |

<sup>1</sup> Partly implemented, see `TODO`s in `openmls/src/group/core_group/validation.rs`.

### Commit message validation

| ValidationStep | Description                                                                            | Implemented | Tested | Test File |
| -------------- | -------------------------------------------------------------------------------------- | ----------- | ------ | --------- |
| `ValSem200`    | Commit must not cover inline self Remove proposal                                      | ✅          | ❌     | TBD       |
| `ValSem201`    | Path must be present, if Commit contains Removes or Updates                            | ❌          | ❌     | TBD       |
| `ValSem202`    | Path must be the right length                                                          | ❌          | ❌     | TBD       |
| `ValSem203`    | Path secrets must decrypt correctly                                                    | ❌          | ❌     | TBD       |
| `ValSem204`    | Public keys from Path must be verified and match the private keys from the direct path | ✅          | ❌     | TBD       |
| `ValSem205`    | Confirmation tag must be successfully verified                                         | ✅          | ❌     | TBD       |

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
