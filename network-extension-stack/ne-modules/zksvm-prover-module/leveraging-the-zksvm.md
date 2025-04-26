# Leveraging the zkSVM

Please reach out to request an API key and generate a proof in only a few minutes.

### **Proof Generation**

The high-level flow is to start the session, send requests containing SVM transactions, end the session to trigger proof generation, and poll the endpoint to retrieve summary statistics.

{% stepper %}
{% step %}
#### Install Rust Dependencies (Optional)

If using the zkSVM Rust client, install the latest version of the necessary crates.

{% code title="Cargo.toml" %}
```toml
[dependencies]
zksvm-api-types = "0.1.x"
zksvm-client = "0.1.x"
```
{% endcode %}
{% endstep %}

{% step %}
#### Start Session

Kick off a session and initialize the genesis accounts that should exist before transaction processing. The request will be rejected if it doesn't contain a valid access key.

{% tabs %}
{% tab title="Rust SDK" %}
```rust
use zksvm_client::session::Session;

let mut session = Session::new(SERVER_URL, SERVER_PORT, API_KEY)?;
let session_id = session.start(genesis_accounts).await?;
```
{% endtab %}

{% tab title="HTTP Request" %}
Request

```
POST /session
```

```json
{
  "api_key": "3ca8a552-f347-4692-a391-f2affa1f01c3",
  "genesis_accounts": {
    "AVxHjnUapzK8C3hQuiyCz7R22bag2CWGNor9v6YZQuJh": {
      "lamports": 1000000000000,
      "data": [],
      "owner": "11111111111111111111111111111111",
      "executable": false,
      "rentEpoch": 124
    },
    "C7zRFydqurVQArmveAKnB11j6WHkFERyH4FYMzSQDRMh": {
      "lamports": 1000000000000,
      "data": [],
      "owner": "11111111111111111111111111111111",
      "executable": false,
      "rentEpoch": 124
    }
  }
}
```

Response

```json
{  
  "session_id": "f8f694eb-377f-48f6-a14b-3f4c56ad3edc"
}
```
{% endtab %}
{% endtabs %}
{% endstep %}

{% step %}
#### Send Transaction

Send SVM transactions that the prover should include in the ZK proof.

There's a max cap of 50 due to memory constraints of the underlying zkVM, and the server will return how many transactions are remaining for the current session.&#x20;

{% tabs %}
{% tab title="Rust SDK" %}
```rust
let remaining_txs = session.send_transaction(tx).await?;
assert_eq!(49, remaining_txs);
```
{% endtab %}

{% tab title="HTTP Request" %}
_Request_

```
POST /session/<session_id>/transactions
```

```json
{  
    "transaction": [...]
}
```

_Response_

```json
{
  "remaining_transactions_in_session": 25
}
```
{% endtab %}
{% endtabs %}
{% endstep %}

{% step %}
#### End Session

End the session to trigger proof generation and select the [SP1 proof type](https://docs.succinct.xyz/docs/generating-proofs/proof-types). Supported types are `core`, `compressed`, and `groth16`. The last option is recommended for most uses, including on-chain verification.

Since proving is async, the response only contains a successful acknowledgement of the request but not the completed proof details.

{% tabs %}
{% tab title="Rust SDK" %}
```rust
use zksvm_api_types::common::ProofType;

session.end(ProofType::Groth16).await?;
```
{% endtab %}

{% tab title="HTTP Request" %}
_Request_

```
PUT /session/<session_id>
```

```json
{
    "proof_type": "groth16"
}
```

_Response_

```
{}
```


{% endtab %}
{% endtabs %}
{% endstep %}

{% step %}
#### Poll Session

Poll the session to see the latest details on its status.

{% tabs %}
{% tab title="Rust SDK" %}
```rust
let res = session.poll_status().await?;
assert_eq!(session_id, res.session.session_id);
assert_eq!(ProofState::InProgress, res.session.proof.state);
```
{% endtab %}

{% tab title="HTTP Request" %}
_Request_

```
GET /session/<session_id>I 
```

_Response_

```json
{
  "session": {
    "session_id": "f8f694eb-377f-48f6-a14b-3f4c56ad3edc",
    "api_key": "3ca8a552-f347-4692-a391-f2affa1f01c3",
    "proof": {
      "proving_time": 123,
      "cycle_count": 456,
      ...
    },
    ...
  },
  "transactions": [...]
}
```
{% endtab %}
{% endtabs %}
{% endstep %}
{% endstepper %}

### **Proof Verification**

Once the prover has completed proof generation, the proof can be verified either locally or via an on-chain program.

<details>

<summary>Local Verification</summary>

The proof can be verified locally to check its integrity.

{% code title="Cargo.toml" %}
```toml
[dependencies]
bincode = "1.3.3"
sp1-sdk = "4.0.0"
sp1-verifier = "4.0.0"
zksvm-client = "0.1.0-alpha"
```
{% endcode %}

{% code title="main.rs" %}
```rust
use zksvm_client::Session;
use sp1_sdk::{HashableKey, SP1ProofWithPublicValues, SP1VerifyingKey};
use zksvm_client::verify::verify_locally;

#[tokio::main]
async fn main() {
	let session = Session::existing(SERVER_URL, SERVER_PORT, API_KEY, SESSION_ID).unwrap();
	let status = session.poll_status().await.unwrap();
	
	let proof = res.session.proof;
	let bytes = proof.proof.unwrap();
	let groth_proof = bincode::deserialize::<SP1ProofWithPublicValues>(bytes.as_slice()).unwrap();
	assert_eq!(64, groth_proof.public_values.to_vec().len());

	let res = verify_locally(proof);
	assert!(res.is_ok());
}
```
{% endcode %}

</details>

<details>

<summary>On-chain Verification</summary>

If it's a `Groth16` proof, it can also be verified on-chain via a verifier program.

{% code title="Cargo.toml" %}
```toml
[dependencies]
solana-client = "2.1.0"
solana-program = "2.1.0"
solana-sdk = "2.1.0"
zksvm-client = "0.1.0-alpha"
```
{% endcode %}

{% code title="main.rs" %}
```rust
use solana_client::rpc_client::RpcClient;
use zksvm_client::verify::verify_on_chain;

#[tokio::main]
async fn main() {
	// assume zkproof from the server exist
	verify_on_chain(
            // (change this URL if using mainnet or testnet)
            RpcClient::new(RPC_URL.to_string()),
            verifier_program_id,
            payer,
            zkproof,
        )
        .expect("Failed to verify proof on chain");
}
```
{% endcode %}

</details>



### Test Utilities

The library provides a few utilities to initialize program accounts and generate sample SVM transactions for convenience.

<details>

<summary>Initialize Program Accounts</summary>

{% code title="Cargo.toml" %}
```toml
[dependencies]
solana-client = "2.1.0"
solana-program = "2.1.0"
solana-sdk = "2.1.0"
zksvm-client = "0.1.0-alpha"
```
{% endcode %}

{% code title="main.rs" %}
```rust
use zksvm_client::test_utils::bpf_loader_upgradeable_program_accounts;

fn main() {
	// assume program bytecode and pubkey exist
	let [(_, program_acc), (programdata_pubkey, programdata_acc)] =
	bpf_loader_upgradeable_program_accounts(
	        &program_pubkey,
	        bytecode.as_slice(),
	        &Rent::default()
	    );
	
	let genesis_accounts: BTreeMap<Pubkey, Account> = BTreeMap::from([
	    (program_pubkey, program_acc),
	    (programdata_pubkey, programdata_acc),
	]);
}
```
{% endcode %}

</details>

<details>

<summary>Prepare SVM Test Transactions</summary>

{% code title="Cargo.toml" %}
```toml
[dependencies]
solana-client = "2.1.0"
solana-program = "2.1.0"
solana-sdk = "2.1.0"
zksvm-client = "0.1.0-alpha"
```
{% endcode %}

{% code title="main.rs" %}
```rust
use zksvm_client::test_utils::{prepare_deploy_program_transactions, prepare_sol_tx, prepare_spl_tx};

fn main() {
  // assume a set of all keypairs exist
  let sol_tx = prepare_sol_tx(&payer, &alice, &bob, &source);
  let spl_tx = prepare_spl_tx(&payer, &alice, &bob, &minter, &mint_authority);
  
  // assume program bytecode and keypair exist
  let deploy_program_txs = prepare_deploy_program_transactions(
      payer,
      &program_keypair,
      program_authority,
      bytecode.to_vec(),
      // assume latest blockhash exists
      latest_blockhash,
  )
  .expect("Failed to create transactons for deploying a program");
}
```
{% endcode %}

</details>



### **System Constraints**

There are a few limitations to keep in mind. The system currently places a hard cap of 50 transactions per session due to memory constraints of the underlying zkVM, which is around 24B cycles. The server will reject requests that send additional transactions beyond 50. Not that if you start a new session, old data from previous sessions are not persisted.

The zkSVM contains several pre-installed Solana Program Library (SPL) programs, e.g. the `token` and `token-2022` programs, but any custom programs need to be deployed before they can be interacted with.
