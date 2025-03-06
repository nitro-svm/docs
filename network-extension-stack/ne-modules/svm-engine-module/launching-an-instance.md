# Launching an Instance

Please reach out to launch an SVM Engine instance, request an API key from the team, and start sending JSON RPC requests.

The two main endpoints that are available are for retrieving account information and sending SVM transactions.

#### Get Account Info

```json
{
 "jsonrpc": "2.0",
 "method": "getAccountInfo",
 "params": ["Di1NNkC7mEnsKFRRsGVzKEToGsm985JbKDvmsmC6DmNC"],
 "id": 1 
}
```

#### Send Transaction

```json
{
 "jsonrpc": "2.0",
 "method": "sendTransaction",
 "params": [ { "signatures": ..., "message": ... } ],
 "id": 1 
}
```

<details>

<summary>Sample Client Code</summary>

Use the following Rust snippet as a guide for interacting with the server.

{% code title="main.rs" %}
```rust
async fn main() -> Result<(), Error> {
    // Step 1: Set up the client
    let server_url = env::var("SERVER_URL").expect("SERVER_URL not found");
    let api_key = env::var("API_KEY").expect("API_KEY not found");
    let mut headers = HeaderMap::new();
    headers.insert(
        AUTHORIZATION,
        HeaderValue::from_str(&format!("Bearer {}", api_key.as_str())).unwrap(),
    );

    let client = HttpClientBuilder::default()
        .set_headers(headers)
        .build(server_url)?;

    // Step 2: Check the status of the account
    let params = rpc_params![PUBKEY];
    let account_info: AccountInfoResponse = client
        .request("getAccountInfo", params)
        .await?;
    println!("Account Info: {account_info:?}");

    // Step 3: Send an SPL transaction
    let seed: &[u8; 32] = b"an_example_fixed_seed_for_tests1";
    let payer = keypair_from_seed(seed).unwrap();
    let seed: &[u8; 32] = b"an_example_fixed_seed_for_tests2";
    let alice = keypair_from_seed(seed).unwrap();
    let seed: &[u8; 32] = b"an_example_fixed_seed_for_tests3";
    let bob = keypair_from_seed(seed).unwrap();
    let minter = Keypair::new();
    let mint_auth = Keypair::new();

    let transaction = prepare_spl_tx(&payer, &alice, &bob, &minter, &mint_auth);
    let params = rpc_params![transaction];
    let signature: Signature = client
        .request("sendTransaction", params)
        .await?;
    println!("Transaction Signature: {signature:?}");

    // Step 4: Check the status of the account again and verify that it has tokens
    let params = rpc_params![PUBKEY];
    let account_info: AccountInfoResponse = client
        .request("getAccountInfo", params)
        .await?;
    println!("Account Info: {account_info:?}");

    Ok(())
}
```
{% endcode %}

</details>



