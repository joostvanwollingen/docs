# Testing Contracts

As part of the Casper local Rust contract development environment, we provide an in-memory virtual machine against which you can run your contract. You do not need to set up a full node for testing. We provide a testing framework that simulates deploy execution, enables monitoring global state changes using assertions, and confirms a successful deploy of the smart contract.

Here is the main testing flow:

1.  Initialize the system and create a deploy.
2.  Execute the deploy, which will call the smart contract.
3.  Query the context for changes and verify that the result matches the expected values.

It is also possible to build scripts with this environment and set up continuous integration for contract code. This environment enables the testing of blockchain-enabled systems from end to end.

## Initialize the System and Create a Deploy

The following steps guide you through the initialization of the system and deploy creation.

1. Define global variables and constants.
2. Import builders and constants.
3. Create a deploy item.

### Declarations and Imports

You can use global variables and constants in later steps to derive values and create components.

**Global Variables and Constants**

```rust
use std::path::PathBuf;
const MY_ACCOUNT: [u8; 32] = [7u8; 32];
const KEY: &str = "my-key-name";
const VALUE: &str = "hello world";
const RUNTIME_ARG_NAME: &str = "message";
const CONTRACT_WASM: &str = "contract.wasm";
```

#### Imports

We derive imports from the dependencies in the Cargo.toml file. If you see problems while importing, fix the dependency settings in the Cargo.toml file.

```rust
use casper_engine_test_support::{
    DeployItemBuilder, ExecuteRequestBuilder, InMemoryWasmTestBuilder,
    ARG_AMOUNT, DEFAULT_ACCOUNT_ADDR, DEFAULT_ACCOUNT_INITIAL_BALANCE, DEFAULT_GENESIS_CONFIG, DEFAULT_PAYMENT,  DEFAULT_RUN_GENESIS_REQUEST,
};

use casper_types::{
    account::AccountHash, runtime_args, Key, PublicKey, RuntimeArgs, SecretKey, U512,
};
```

### Create a Deploy Item

**Declaring Local Variables**

```rust
let secret_key = SecretKey::ed25519_from_bytes(MY_ACCOUNT).unwrap();
let public_key = PublicKey::from(&secret_key);
let account_addr = AccountHash::from(&public_key);
let session_code = PathBuf::from(CONTRACT_WASM);
let session_args = runtime_args! {
            RUNTIME_ARG_NAME => VALUE,
};
```

-   _secret_key and public_key_ : Used to derive account address

-   _account address_ : Used to get the authorization key and location

-   _session_code_ : Gets the path to your actual contract wasm file on your system

-   _session_args_ Get the runtime arg value

**Create Deploy Item**

Before deploying the contract in the framework, you need to have a `Deploy Item` to send to the request. `DeployItemBuilder` will directly instantiate the deploy item using associate builder methods.

```rust
let deploy_item = DeployItemBuilder::new()
    .with_empty_payment_bytes(runtime_args! {ARG_AMOUNT => *DEFAULT_PAYMENT})
    .with_session_code(session_code, session_args)
    .with_authorization_keys(&[account_addr])
    .with_address(account_addr)
    .build();
```

Deploy items contain the following elements:

-   _`payment details`_ : This can be standard payments or custom payments. Standard payment is the bare payment amount a user wishes to pay for the deploy. Custom payment comes with payment codes or functions that indicate payments by module bytes. `empty_payment_bytes` implies the module bytes inside the deploy item's payment part are empty. It directs the framework to use the standard payment contract that is the original amount (DEFAULT_PAYMENT).

<p align="center"><img src="/static/image/EmptyModuleBytes.png" width="300"/></p>

-   _session_code_ : Sets the session code for the deploy using session_code and session_args

    -   PathBuff : Helps to find the compiled WASM file in your WASM directory. This is a mutable path with some extended functionalities.

-   _authorization_keys_ : Sets authorization keys to authorize the deploy

-   _address_ : Sets the address of the deploy

## Deploy the Smart Contract

Follow these steps to deploy the smart contract:

1. Create the builder.
2. Create an execute request.
3. Deploy the contract.

### Create the Builder

`InMemoryWasmTestBuilder` functions as the builder inside this framework, and it is the builder for a simple WASM test that uses the state held entirely in memory. InMemoryWasmTestBuilder provides methods to simulate deploys to the blockchain array and make queries to whatever state you find in the global state.

```rust
let mut builder = InMemoryWasmTestBuilder::default();
        builder.run_genesis(&DEFAULT_RUN_GENESIS_REQUEST).commit();
```

`Genesis` : Run genesis call will initialize the blockchain network to get your first block. When you initialize a blockchain network, there needs to be a genesis block as a starting point to the incoming blocks. The subsequent set of deploys will execute after the execution of the genesis block.

### Create an Execute Request

After creating the `deploy_item`, wrap it in an `ExecuteRequest` created by the `ExecuteRequestBuilder`; then, the builder instance will build the deploy item and return the execute_request.

```rust
let execute_request = ExecuteRequestBuilder::from_deploy_item(deploy_item).build();
```

### Deploy the Contract

InMemoryWasmTestBuilder instance will execute the _execute_request_, which carries the deploy details.

```rust
builder.exec(execute_request).commit().expect_success();
```

-   _Commit()_ - This will process the execution result of the previous execute_request on the latest post-state hash, which is the hash function's output.

-   _expect_success()_ - Assert this is a successful execution of a deploy. If it is not successful, it will crash the test.

## Query and Assert

Query and assertion steps are as below:

1. Pre-assert the state
2. Deploy the contract
3. Post-assertion

The smart contract creates a new value _hello world_ under the key _my-key-name_. It is possible to extract this value from the global state of the blockchain using the `query_result`.

### Pre-Assertion

Pre-assertions are helpful to confirm the existing state that you expect to change has not changed before the test execution. To accomplish this, create a query result using the builder and assert that the query's result should not be equal to the value of KEY.

```rust
  let result_of_query = builder.query(None, Key::Account(account_addr), &[KEY.to_string()]);
  assert!(result_of_query.is_err());
```

The parameters are: post-state, not defined in this case base_key, which is the actual namespace of the value the global KEY constant

### Deploy the Contract

Deploy is done by executing the previously created execute the request.

```rust
builder.exec(execute_request).commit().expect_success();
```

-   _Commit()_ - Commit effects of previous execution call on the latest post-state hash.

-   _expect_success()_ - Assert this was a successful execution of a deploy. If it's not successful, it will crash your test.

### Post-Assertion to confirm Deploy

This will query the post-deploy value and assert for its change.

```rust
  let result_of_query = builder
            .query(None, Key::Account(account_addr), &[KEY.to_string()])
            .expect("should be stored value.")
            .as_cl_value()
            .expect("should be cl value.")
            .clone()
            .into_t::<String>()
            .expect("should be string.");
```

-   _query()_ - Queries state for a given value.

-   _expect()_ - The validation for the query which contains the output message. This will unwrap the value; the test will panic and crash if the value can't be unwrapped. The string value inside the argument will output as the reason to crash.

-   _as_cl_value()_ : Returns a wrapped [CLValue](/docs/design/serialization-standard#serialization-standard-values) if this is a CLValue variant.

-   _clone()_ - To break the reference to the CLValue so that it will provide brand new CLValue

-   Into_t() - Cast the CLValue back to the original type (i.e., a String type in this sample). Note that the `expected_value` is a `String` type lifted to the `Value` type. It is also possible to map `returned_value` to the `String` type.

**Assertion**

Assert that the query's result matches the expected value; here, the expected value is "hello world".

```rust
assert_eq!(result_of_query, VALUE);
```

## Final Test Sample

The code below is the simple test generated by [cargo-casper](https://crates.io/crates/cargo-casper) (found in `tests/src/integration_tests.rs` of a project created by the tool).

```rust
#[cfg(test)]
mod tests {
   use casper_engine_test_support::{
        DeployItemBuilder, ExecuteRequestBuilder, InMemoryWasmTestBuilder, ARG_AMOUNT,
        DEFAULT_ACCOUNT_ADDR, DEFAULT_ACCOUNT_INITIAL_BALANCE, DEFAULT_GENESIS_CONFIG,
        DEFAULT_PAYMENT, DEFAULT_RUN_GENESIS_REQUEST,
    };
    use casper_types::{
        account::AccountHash, runtime_args, Key, PublicKey, RuntimeArgs, SecretKey, U512,
    };
    use std::path::PathBuf;
const MY_ACCOUNT: [u8; 32] = [7u8; 32];
    // Define `KEY` constant to match that in the contract.
    const KEY: &str = "my-key-name";
    const VALUE: &str = "hello world";
    const RUNTIME_ARG_NAME: &str = "message";
    const CONTRACT_WASM: &str = "contract.wasm";
#[test]
    fn should_store_hello_world() {
        let secret_key = SecretKey::ed25519_from_bytes(MY_ACCOUNT).unwrap();
        let public_key = PublicKey::from(&secret_key);
        let account_addr = AccountHash::from(&public_key);
        let session_code = PathBuf::from(CONTRACT_WASM);
        let session_args = runtime_args! {
            RUNTIME_ARG_NAME => VALUE,
        };
let deploy_item = DeployItemBuilder::new()
            // .with_payment_bytes(module_bytes, args)
            .with_empty_payment_bytes(runtime_args! {
                ARG_AMOUNT => *DEFAULT_PAYMENT
            })
            .with_session_code(session_code, session_args)
            .with_authorization_keys(&[account_addr])
            .with_address(account_addr)
            .build();
let execute_request = ExecuteRequestBuilder::from_deploy_item(deploy_item).build();
let mut builder = InMemoryWasmTestBuilder::default();
        builder.run_genesis(&DEFAULT_RUN_GENESIS_REQUEST).commit();
// prepare assertions.
        let result_of_query = builder.query(None, Key::Account(account_addr), &[KEY.to_string()]);
        assert!(result_of_query.is_err());
// deploy the contract.
        builder.exec(execute_request).commit().expect_success();
// make assertions
        let result_of_query = builder
            .query(None, Key::Account(account_addr), &[KEY.to_string()])
            .expect("should be stored value.")
            .as_cl_value()
            .expect("should be cl value.")
            .clone()
            .into_t::<String>()
            .expect("should be string.");
assert_eq!(result_of_query, VALUE);
    }
}

fn main() {
    panic!("Execute "cargo test" to test the contract, not "cargo run".");
}
```

You can see the result of the above test in this screen capture:

<p align="center"><img src="/static/image/test-compile-result.png" width="700"/></p>
