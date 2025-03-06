# Private Decryption

# Fairblock Private Decryption Standard

## Abstract
This document defines the standard interface and expected behavior for Private Decryption Contracts deployed on FairyRing. It provides a structured approach to managing private identities, private decryption keys, and encrypted data storage, ensuring developers can build interoperable and secure applications.

**Two main things are provided:**

1. A 'interface' for FRC-03, the FairyRing Request for Comment standard representing private decryption contracts. It has comments within it that help developers understand what hooks need to be maintained and possible ideas on how to implement custom logic for their respective cApp.
2. An accompanying set of sequence diagrams to help developers understand the typical transaction flow anticipated using the FRC03 within a cApp design. The sequence diagrams can be found here for [now](https://drive.google.com/file/d/1akjomwW2-tL-pJV616lZ52vpFyVLA-ty/view). Make sure to open it with `draw.io`.

## Motivation
This standard outlines a template design pattern for contracts handling private decryption on FairyRing. The typical transaction flow is outlined within the sequence diagram below, and further details to accompany the sequence diagram are within this document.

---
## Design Pattern Typical Flow

The below design pattern is written from the perspective of the developers, both back-end and front-end. It is assumed that developers creating cApps will create the smart contracts for cApps, and coordinate different solutions for front-ends.

For front-end development, we recommend using the `ts-lib` repo hosted within the Fairblock github organization to interact with FairyRing, in tandem with the typical integration wtih smart contracts, but on FairyRing.

### 1. Contract Interface
A Fairblock Private Decryption contract, following the guidelines of `FRC03.rs`, are to follow these guidelines:

#### State Vars

The following `IdentityRecord` struct is used in a mapping called `RECORDS` to keep track of all state related to encrypted transactions within this contract. It is updated throughout the main contract.

From `state.rs` module, the `IdentityRecord` struct consists of the following, where comments help define each object:

```rust
pub struct IdentityRecord {
    pub identity: String,  // Unique ID from FairyRing
    pub pubkey: String,  // Master Public Key
    pub creator: String,  // The user who created this identity
    pub encrypted_data: String,  // Data stored under this identity
    pub price: Coin,  // Cost paid for identity creation
    pub private_keyshares: HashMap<String, Vec<IndexedEncryptedKeyshare>>,  // Private decryption keyshares
}
```

**How `RECORDS` is used within the contract**

- **`RECORDS`** is a CosmWasm storage map storing identity-related information.
- **Key:** `&str` (FairyRing unique identity).
- **Value:** `IdentityRecord` struct containing:
  - Identity, public key, creator, encrypted data, price, and private keyshares.

**Usage:**
- **Saving Identity:** In `reply()`, upon receiving a response from FairyRing.
- **Storing Encrypted Data:** In `store_encrypted_data()`, when users upload encrypted content.
- **Requesting Keyshares:** In `execute_request_keyshare()`, before making the request.
- **Receiving Keyshares:** In `execute_private_keys()`, upon response from FairyRing.
- **Querying Identities:** Via `query_identity()` and `query_all_identities()`.


<!-- TODO: Question for ap0calpyse: the Master Public Key here... is it the specific one associated to the identity (condition) for this transaction? In the case of contracts that use `general conditions` my understanding is that conditions aren't really defined/associated directly with FairyRing / pep module. It is directly in the smart contract now. This is a VIP question because I've seen Master Public Key, Public Keys, and just need to be sure on this. THAT FAIRYRING MAKES LOTS AND LOTS OF PUBLIC KEYS BASED ON WHATEVER ID IS REQUESTED AND GENERATED. EVERY UNIQUE ID IS ASSOCIATED TO A MASTER PUBLIC KEY THAT CAN BE USED TO ENCRYPT AND WHATNOT. BY DEFAULT FAIRYRING USES BLOCK HEIGHT AS ITS CONDITION/ID - SO FOR THOSE IDS, THEY GET MASTER PUBLIC KEYS. FOR GENERAL CONTRACTS, IDS ARE UNIQUE IDS - NOT CONDITIONS. SO WE GET MASTER PUBLIC KEYS FOR THOSE UNIQUE IDS. 

REALLY THE KEY QUESTION: What is the master public key, and how is it different than public keys. Or are they the same. I'm not talking about public keys associated to wallets. I'm talking about keys associated to FairyRing.
-->


#### Step 1a - `request_identity(price: Coin)`
**Purpose:** Called by a user to request a private identity for a transaction. The `reply()` processes the async response from FairyRing. See [`reply()` function](TODO-GetLink).

**Behavior:**
- Increments `LAST_REPLY_ID`.
- Stores contract address as the `creator` for the specific id
- Stores the pending request (tracks sender and price).
- Sends a `MsgRequestPrivateIdentity` to the chain. Specifically via `CosmosMsg::Any(any_msg.clone())` within a submessage, and the response is handled asynchronously using the reply mechanism.
- Next step is that contract, with `reply()` will process the response from FairyRing.

<!-- TODO: Questions - **Security Considerations:**
- Is there a cost to an identity? If this is a design area, would it be a good idea to not have an identity be granted for free unless explicitly permitted to prevent spamming from the contract? -->

**Gotchas:**
- The requester must wait for a reply before their identity is usable.
- IDs are unique in the 'global' scheme for the FairyRing ecosystem because FairyRing ultimately processes the ID that is given to it from `execute_request_identity()` and ensures it is unique.
   - In a cross-chain world, there could be identital external contract addresses and potential collision. Thus, FairyRing has to ensure that each identity for a specific contract and transaction is globally unique.

<!-- TODO: Questions for Ap0calypse: 
1. FairyRing actually ensures unique IDs per transaction right?
2. For this function, `execute_request_identity()`, it looks like all parts of it are important tbh.
3. Developer can go ahead and add extra stuff to it, but overall it must do everything that is in it for:
   - Storing PENDING_REQUESTS, Keeping track of LAST_REPLY_ID, proposing a req_id start and preparing it in the msg sent to FairyRing, work with the `reply()` to obtain the end result. -->


---

#### Step 1b Reply Handling
Fairblock decryption contracts receive asynchronous replies for identity and keyshare requests.

##### `reply(msg: Reply)`
**Purpose:** Processes responses from the chain.

**Behavior:**
- Extracts identity or keyshare response data.
- Updates `RECORDS` and removes the request from `PENDING_REQUESTS`.

**Gotchas:**
- If the response data is missing or invalid, the contract must handle the failure gracefully.
- When receiving identities, the new unique identity from FairyRing is stored and used throughout the contract.

#### Step 2 - Receive Unique ID from FairyRing

This is from the response from Step 1a, and is done within step 1b. The unique ID is then used for further processing of this transaction until it is decrypted.

#### Step 3 - Register Contract and Transaction to `pep`

<!-- TODO: ap0alpyse what's up with the registering tx. It is directly to pep, and registering the transaction with the respective contract to it? Shouldn't it already have the unique id that FAIRYRING generated on hand? Or is this step to ensure that only the contract can actually be the caller for getting keyshares for any IDs that come out of it (for Private Decryption)? Otherwise, I'm not sure when or why we'd have to register a tx... I'd think that the request_id function and processing on FairyRing's side would take care of that.  -->

#### Step 4 - Encrypt transaction data

Done with FairyRing, and is done by the front end.

#### Step 5 - Call function `store_encrypted_data()`

Contract function that is used to store the encrypted data for an encrypted transaction. This is the final storage of the encrypted transaction with the unique ID obtained.

#### Step 6 - START OF USER 2 COMING ALONG AND DECRYPTING THE TRANSACTION FROM USER 1 

#### `request_private_keyshare(identity: String, secp_pubkey: String)`
**Purpose:** Requests private keyshares for a given identity.

**Behavior:**
- Increments `LAST_REPLY_ID` (to track asynchronous responses).
- Stores the pending request in `PENDING_REQUESTS`.
- Creates an empty keyshare entry for the requester.
- Sends a `MsgRequestPrivateDecryptionKey` to the chain.

**Gotchas:**
- The requester must already have a valid identity for keyshares to be granted.
- Keyshares are not available immediately—this request is asynchronous.


<!-- TODO: Questions for ap0calpyse: why isn't ENCRYPTED_KEYSHARE_VALUE automatically obtained from the contract call / in the `reply()` function within the `contract.rs`? Right now we have the bash script reading the state and then parsing it for the private keyshares.
 -->

#### Step 8: Wallet-2 aggregates keyshares to obtain the final decryption key

This is done off-chain and likely by the front-end using `ts-lib`, where the obtained keyshares are aggregated using `fairyringd aggregate-keyshares()`.

#### Step 9: Wallet-2 decrypts the stored data and compares with original encrypted message

This is done off-chain and likely by the front-end using `ts-lib`, where the obtained keyshares are aggregated using `fairyringd query pep decrypt-data()`.
<!-- 
---

TODO: BELOW ARE NOT PART OF MY REWRITE SO FAR. IGNORE FOR NOW

### 1.1 Initialization
#### `instantiate`
**Purpose:** Initializes the contract by storing the chain’s master public key.

**Parameters:**
- `pubkey: String` – The master public key of the chain.

**Behavior:**
- Stores the `pubkey` on contract deployment.
- Initializes `LAST_REPLY_ID` to track asynchronous replies.

**Edge Cases:**
- If the `pubkey` is not set properly, identity and decryption key requests will fail.

---

### 1.2 Execution Messages
Each execute message represents an action that modifies contract state.

#### `update_pubkey(pubkey: String)`
**Purpose:** Updates the master public key stored in the contract.

**Behavior:**
- Replaces the existing public key.

**Security Considerations:**
- Should be restricted to an authorized actor to prevent unauthorized key updates.
---

#### `store_encrypted_data(identity: String, data: String)`
**Purpose:** Stores encrypted data linked to a private identity.

**Behavior:**
- Updates the `RECORDS` entry for the specified identity.
- Stores data under `encrypted_data`.

**Gotchas:**
- The data must be pre-encrypted before sending.
- If the identity does not exist, this transaction fails.

---

#### `execute_private_keys(identity: String, private_decryption_key: PrivateDecryptionKey)`
**Purpose:** Stores private keyshares received from the PEP module.

**Behavior:**
- Updates `RECORDS[identity]` with the `private_keyshares` mapped to the requester.

**Security Considerations:**
- Only authorized entities should be able to call this function.
- Keyshares should be verified before being stored.

---

## 3. Queries
Fairblock Private Decryption contracts expose read-only queries for retrieving stored identities.

#### `get_identity(identity: String) -> IdentityResponse`
**Purpose:** Fetches the stored details of a given identity.

**Returns:**
- Identity details, including `pubkey`, `creator`, `price`, and `private_keyshares`.

---

#### `get_all_identities() -> AllIdentitiesResponse`
**Purpose:** Fetches all identities stored in the contract.

**Returns:**
- A list of all registered identities.

---

## 4. Common Design Patterns & Gotchas

### 4.1 Asynchronous Calls & Replies
- Requests for identities and keyshares do not return immediately. Developers must wait for replies.
- Each request is tracked by a unique `LAST_REPLY_ID`.

### 4.2 Handling Private Data
- Data must be encrypted off-chain before being stored.
- Keyshares are only useful after decryption.

### 4.3 Permissions & Security
- Only authorized actors should update the master public key.
- Storing invalid keyshares should be prevented.

---

## 5. Summary
This standard defines the expected behavior of Private Decryption Contracts on FairyRing, ensuring consistency, security, and best practices. Following this pattern will help engineers build interoperable and reliable private decryption systems. -->
