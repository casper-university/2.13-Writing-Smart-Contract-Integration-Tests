# Writing Casper Smart Contract Integration Tests

Hi everyone I hope you're all having a good day. Thanks for joining me today as you'll learn to write integration tests to test smart contracts for the Casper Network without having to deploy them to the network itself. The tests that you will write will execute locally in a mock VM that adheres to the same rules that the full-fledged Casper networks do. This way, you can perform tests to ensure expected outcomes without needing to wait for deploys, specific blocks or conditions, and without needing to fund accounts. For today's example, we will use the "counter" smart contract that we built previously. Since we wrote this smart contract three weeks ago we will take a moment to go over the contract before we write the tests, but in essence, all it does is store a number and expose an entrypoint that increments it.

Begin by opening the counter smart contracts directory in an editor like VS Code. If you don't have the project on standby, you can clone it from its [git repository](https://github.com/casper-university/2.8-Writing-Smart-Contracts-In-Rust).

Open */contract/src/main.rs*, this is the counter contract.

Take a look at the `call` function. It starts by defining a new value `0u32` at a newly generated URef, then inserts it into the contract's named keys under `"count_key"`. Next it adds the entrypoint `increment_count`, which is defined as a function below, then takes the entry points object along with the named keys and creates a new smart contract out of it.

The `storage::new_contract` function returns the contract hash, which we then store under `counter_contract_hash` within the deploying account's named keys.

Now the next function represents the `increment_count` entrypoint. All this function does is acquire the `count` value from the contract's named key `"count_key"`, and directly increment the value by `1`.

Now that we've reviewed the counter contract we can begin writing tests for it. For such a simple smart contract such as this, testing on a testnet or NCTL would not really be a problem, but for more complex contracts, you may want to perform a series of tests that would be impractical to test repeatedly on a live network.

In the left side panel you'll notice a directory *tests/*. Expand this folder, then the *src/* folder, and open the *integration_tests.rs* file.

There are a few new concepts and Rust libraries we'll be exploring here, so bear with me.

On line 5 you'll see the Rust attribute `#[cfg(test)]`. This is a common attribute in Rust projects, and lets the compiler know that this module should only be compiled and run when we execute `cargo test` in the command line, and not while compiling debug or release versions of the project.

On line 6 a new module `tests` is defined. This is the module that the `#[cfg(test)]` attribute acts on. Everything within this module is related to our tests.

Line 7 contains an import of the `PathBuf` module, which we'll use to import the compiled WebAssembly smart contract from our local directory.

Next, the `casper_engine_test_support` crate is imported. This, alongside the `casper_execution_engine` crate that we'll look at next, is an important create in testing Casper smart contracts. This crate provides an interface to write tests and interact with an instance of the execution engine.

The next imported crate, `casper_execution_engine`, enables execution of compiled smart contracts within the test framework, effectively preparing a mock VM and simulating what would occur in a live Casper network environment.

The last import is `casper_types`, which we've used previously and allows us to import datatypes that are used in the smart contract.

Go ahead and delete the existing test functions.

Now create a new test function `test_counting` and prefix it with the `#[test]` attribute:

```rust
#[test]
fn test_counting() {
  
}
```

We need to begin the test by invoking an instance of the execution engine, which we can do by instantiating a new `InMemoryWasmTestBuilder`:

```rust
let mut builder = InMemoryWasmTestBuilder::default();
```

Now we can initiate the simulated blockchain by running the `.run_genesis` command on the `WasmTestBuilder`:

```rust
builder
	.run_genesis(&PRODUCTION_RUN_GENESIS_REQUEST)
	.commit();
```

Now we can build an installation deploy, effectively constructing a signed deployment of the counter smart contract:

```rust
let counter_installation_request = ExecuteRequestBuilder::standard(
	*DEFAULT_ACCOUNT_ADDR,
	"contract.wasm",
	runtime_args! {},
).build();
```

We do this by using the `ExecuteRequestBuilder` module and providing as arguments the `DEFAULT_ACCOUNT_ADDR` which is the default Casper account used for testing, the name of the compiled smart contract binary which you can see under */target/wasm32-unknown-unknown/release/contract.wasm* ***Show this in left side panel***, and an empty runtime arguments object. Don't forget to place an `*` (asterisk) before `DEFAULT_ACCOUNT_ADDR` as we want to dereference it. Then we tag on the `.build()` function at the end to convert the object into an `ExecuteRequest` which is the type it will need to be in to submit it to the mock VM.

Now we can use the `WasmTestBuilder` we prepared at the beginning of the test to execute this deployment request:

```rust
builder
	.exec(counter_installation_request)
	.expect_success()
	.commit();
```

The `.expect_success()` function expects that the deployment will succeed and will otherwise panic, and the `.commit()` function will commit the deployment to the mock blockchain.

Next, we need to obtain the contract hash of the smart contract, just like we do in a real environment:

```rust
let contract_hash = builder
	.get_expected_account(*DEFAULT_ACCOUNT_ADDR)
	.named_keys()
	.get("counter_contract_hash")
	.expect("must have contract hash key as part of contract creation")
	.into_hash()
	.map(ContractHash::new)
	.expect("must get contract hash");
```

Let's break down all these function calls:

* `get_expected_account` gets the deploying account
* `named_keys` gets that accounts named keys
* `get("counter_contract_hash")` gets the contract hash value from the named keys, which as you may remember we added the contract hash to the account's named keys on line 59 in the counter smart contract 
* `expect` Expects this value to be present, which it will be in this case. Otherwise this function panics and the test fails
* `into_hash` Converts the named key value into a `Hash` object
* `map(ContractHash::new)` converts the `Hash` into a `ContractHash` object which is a recognizable Casper type
* `.expect("must get contract hash");` Expects this all to have worked properly, otherwise panics and the test fails

Now we can officially test the increment entrypoint. To do this we can use the `contract_call_by_hash` function on the `ExecuteRequestBuilder` module to prepare the deployment request:

```rust
let increment_request = ExecuteRequestBuilder::contract_call_by_hash(
	*DEFAULT_ACCOUNT_ADDR,
	contract_hash,
	"increment_count",
	runtime_args! {},
).build();
```

This looks very similar to our installation request that we created a few minutes ago, but with some notable differences. Instead of providing the path to the compiled smart contract, we instead provide the smart contract hash, and the entrypoint we would like to call, which is in this case `"increment_count"`.

Then, just like with installing the contract, we run the `builder.exec` function and expect success:

```rust
builder
	.exec(increment_request)
	.expect_success()
	.commit();
```

Great, we've now completed the first test, which just installs the contract and invokes the `increment_count` entrypoint. We can test this by heading back to the terminal and running `make test`. ***Do this***

You'll see that our test succeeded.

Let's go back to our code and add some logic to the test. We want to assert that the count variable is equal to `1` after we call the entrypoint. Since everytime we run the test, a new instance of the blockchain is spun up, and a new contract is installed, it will always start at `0`, and after the `increment_count` entrypoint is invoked, will be `1`.

First, we need to acquire the `count_key` from the named keys of the smart contract:

```rust
let count_key = builder
	.get_contract(contract_hash)
	.expect("Not able to find contract")
	.named_keys()
	.get("count_key")
	.expect("Unable to find count_key")
	.clone();
```

In this command we call `get_contract` on the `WasmTestBuilder`, providing the contract hash we obtained above, then use `expect` to force unwrap the option. Then we get its named keys, and from them the `"count_key"`. We `expect` that this key exists, and finally clone the object, because as it is it is only a reference to the `Key` object.

Now we can get the value from the named key:

```rust
let count_key_value = builder
	.query(None, count_key, &[])
	.expect("should be stored value.")
	.as_cl_value()
	.expect("should be cl value.")
	.clone()
	.into_t::<u32>()
	.expect("should be u32.");
```

Here, the `.query` function queries the value of a `Key`, in our case the `count_key`. The first argument accepts either a post hash, similar to a block hash if you're familiar, or `None`. By putting a post hash here you could get the value at a certain time within the blockchain's history, but for our purposes we just want the current value, so can use `None`. The empty array reference in the last parameter is for the path to the value. Since our value isn't nested in something like a dictionary or mapping, we can leave this blank.

This query returns a `Result`, so we can use `expect` to make sure that it's successful, and panic otherwise. Then we convert this value into a `CLValue`, which is a value implemented by all of the casper types. This again returns an option value, so we can `expect` that it's not `None`. The object returned is a reference to a `CLValue`, so we need to `clone` it. Then we can turn it into a `u32`, which is the type it is in the smart contract, and finally `expect` that that type casting will work properly.

Now that we finally have the count value, we can `assert` that it is equal to `1`:

```rust
assert_eq!(count_key_value, 1);
```

And we're done! Save the file and head back to the terminal and re-run `make test`. You should see that the test works properly.

If we go back to our tests and change the value in `assert_eq!` to `2`, save and re-run the test, you'll notice that the test fails as we expect it to, because the value stored in global state is `1` and not `2`.

That's it! Now you know how to write integration tests for Casper smart contracts. This knowledge can be expanded to write more complex tests for more complex contracts, using the same principles.

I'd now like to open the floor for any questions you may have regarding today's lecture, the Casper smart contract testing frameworks, or how this contract interacts with them.

