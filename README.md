# Writing Smart Contracts in Rust

In today's lecture you'll be writing your first smart contract in Rust. There are multiple blockchain platforms that support Rust-based smart contracts, but we'll be using the Casper Network. Each platform has different standards for how smart contracts need to be written, and Casper is no different. Casper natively supports WebAssembly binaries, to which each Rust contract is compiled. While Casper smart contracts can be written in other languages, Rust is preferred due to its strict type safety and concision.

The Casper Network developers have prepared libraries that make writing smart contracts on Casper easy.
Today we'll take a look at how you can use Rust and these libraries to write your first smart contract.

Casper recommends using Linux to write and compile Casper compatible smart contracts. MacOS also works, but is not officially supported.

Compiling Casper smart contracts requires the use of Rust and the Cargo package manager.
Before installing these though, we need to install some dependencies.

Use `apt` in Ubuntu to install the following packages

```bash
sudo apt install curl build-essential pkg-config openssl libssl-dev
```

Then install cargo with

```bash
sudo apt install cargo
```

To install Rust, run

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

Then restart the terminal to refresh your `PATH` environment variable 

Now that you have Rust and cargo installed, you can install the relevant Casper cargo crates.
Casper has a command line tool for creating template contracts and associated tests, known as `cargo-casper`, install this tool by running:

```bash
cargo install cargo-casper
```

As for interfacing with the blockchain, another tool was created called `casper-client`. This tool allows you to query the status of the network like block data, validator information, and global state data. It also allows you to send deploys to the network; these deploys can either come in the form of contract installation, entrypoint invocation, or native functions like transferrals and delegations.

Install the `casper-client` by running

```bash
cargo install casper-client
```

We can use `cargo-casper` to prepare template contracts, and can use `casper-client` to install them, but what about compiling the contracts to the installable format, WebAssembly?

For this you can use `CMake`, a popular build tool. Install it by running

```bash
sudo apt install cmake
```

Now that you have all the necessary tools installed, let's get started writing your first smart contract.

Create a new smart contract template by opening a terminal window, navigate to the directory you'd like your project stored ( `cd ~`), and run

```bash
cargo casper (and the name of your project)
```

A new contract template will be created in *contract/src/main.rs*. Open this file in an editor of your choice, I'll be using VS Code.

Let's examine this template. As is, this is not a complete smart contract, or even a smart contract at all. What we currently have is known as "session code". Session code can be deployed to the blockchain just like a smart contract can, but it is ephemeral in the sense that, once deployed, there is nothing further a user can interact with. A smart contract, on the other hand, is stored on the blockchain and can be invoked by users at a later time. You'll learn how to construct a full-blown smart contract shortly, but first let's look at what this logic is doing.

On the first two lines, you'll see two Rust attributes, `#![no_std]` and `#![no_main]`. These indicate that the program does not include the Rust standard library and does not have the default entry point `"main"`, respectively. Next, we have what's known as a "configuration predicate", that checks the target compilation architecture and ensures that it's "wasm32". If it's not, the next line is executed, which throws an error notifying the developer to compile targeting the proper architecture for Casper's execution engine.

Next is the importation of dependencies. We start with `extern crate alloc` which imports the `alloc` crate to the project; this is needed because we need to access the `String` class and we're in a `no_std` environment (show that we do this on the next line).

On line 13 the `casper_contract` crate is imported. In the template, the  `runtime` and `storage` modules are imported from the `contract_api` module. These are important modules that consist of supporting functions for interacting with Casper's execution engine. `runtime` contains functions responsible for interacting with transient objects such as caller provided runtime arguments and session data, all of which are relevant only to the session's context. `storage` on the other hand contains functions responsible for accessing and mutating local and global state, such as creating permanent smart contracts on the blockchain and mutating other data like dictionaries and `URef`s.

In addition to the `runtime` and `storage` modules, `casper_contract` also holds the `account` and `system` modules, responsible for managing Casper accounts and interacting with immutable system smart contracts, respectively.

Directly after the importation of the `casper_contract` crate is that of the `casper_types` crate, the other pertinent crate for writing Casper smart contracts. `casper_types` is an extensive crate containing modules, macros, structs, enums, constants, and functions, all related to the datatyping of objects.

Next is the creation of a user error enum. This logic allows the developer to create his or her own errors; useful for debugging. Each error represents a `u16` value, which can be logged to a deploy response in the event of a revert.

Finally, we have the `call` function, which is the function executed upon deployment of the binary to the Casper Network. Before the definition of the function, the `#[no_mangle]` attribute is present. This is necessary so the compiler does not alter the function name in the interest of efficiency. If the `#[no_mangle]` attribute weren't present, the compiler may alter the name making it invisible to the execution engine.

Within the `call` function is where all the necessary setup resides. In the example template, it is first checked if a given key exists in the session, which it should not, as it's not been defined. If logic were prepended that created that key, the call would revert with the error `KeyAlreadyExists`.

Next, a new constant is defined, `value`, that is assigned the `String` value provided as a runtime argument by the deployer, and on the next line, it is stored in a new universal reference, or " `URef`" in global state. If left unchanged, this value will propagate through the blockchains global state forever, always referencable. On the following line, this `URef` is stored under a "named key", which is a unique key associated with a comprehensible string in the caller's account data.

Lastly, it is validated that the named key was stored in the calling account's data; if for whatever reason it wasn't, the value would unwrap to `None` instead of `Some(value)` and execution would be reverted. The stored key is then compared to the initial key constant, and if there is a discrepancy, execution would revert with `KeyMismatch`, however as the code is written, this would never happen.

Now that we've reviewed the template, delete everything within the `call` function as you prepare to write your own smart contract logic.

For today's example, we'll be writing a simple "Counter" smart contract, that will contain a single integer `count`, initially set to zero, and a public entrypoint that allows any account to increment the count.

Start by importing the necessary dependencies:

```rust
use alloc::string::{String, ToString};
use alloc::vec::Vec;

use casper_contract::{
    contract_api::{runtime, storage},
    unwrap_or_revert::UnwrapOrRevert,
};
use casper_types::{ApiError, Key, URef, CLType, EntryPoint, EntryPoints, EntryPointAccess, EntryPointType, contracts::NamedKeys};
```

Begin the `call` function by defining the `count` variable and setting it to 0:

```rust
let count = storage::new_uref(0u32);
```

The value of this new variable count is referenced by a new `URef`. We need to store this `URef` somewhere so we can later reference the value of `count`. On Casper there is tool for saving these references to objects, known as "Named Keys". Named Keys can be stored under accounts and alongside smart contracts. In this case we want to store them with the smart contract. Create a new `NamedKeys` object and store the "count" `URef` under the key "count_key":

```rust
let mut named_keys = NamedKeys::new();
named_keys.insert(String::from("count_key"), count.into());
```

As mentioned, our smart contract will have one entrypoint that will increment the count, which will appropriately be named `increment_count`. Casper's execution engine expects us to explicitly publicize this entrypoint, which can be done by creating a new `EntryPoints` object, and adding a new entrypoint to it:

```rust
let mut entry_points = EntryPoints::new();

entry_points.add_entry_point(EntryPoint::new(
	"increment_count",
	Vec::new(),
	CLType::Unit,
	EntryPointAccess::Public,
	EntryPointType::Contract,
));
```

All that's left in the `call` function is to create a new smart contract in the blockchain's global state, and store a reference to it in the deploying account's named keys.

```rust
let (contract_hash, contract_version) = storage::new_contract(
	entry_points,
	Some(named_keys),
	Some("counter_package".to_string()),
	Some("counter_access_uref".to_string()),
);

runtime::put_key("counter_contract_hash", contract_hash.into());
```

Now we can write the `increment_count` function. Start by creating a new public, external function `increment_count`, and supply the `#[no_mangle]` attribute just before the function declaration:

```rust
#[no_mangle]
pub extern "C" fn increment_count() {
	
}
```

Within this function, we need to acquire the `URef` from the contract's named keys, use it to get the `count` value, increment it, and rewrite the new value to the count's `URef`. We can use `runtime::get_key("count_key")` to get the wrapped count key. `get_key` returns an `Option` value, which could result in `None`, so we can unwrap it safely using `unwrap_or_revert`. Once it's unwrapped to a `Key` object, we can convert it into its `URef` representation using `into_uref`, which also returns an option, so it needs to be safely unwrapped as well:

```rust
let count_uref: URef = runtime::get_key("count_key")
	.unwrap_or_revert()
	.into_uref()
	.unwrap_or_revert();
```

Now we can use `storage::add` to increment the count by one:

```rust
storage::add(count_uref, 1);
```

Congratulations! You've just written and understood the workings of a basic smart contract on the Casper Network. Now let's deploy this smart contract to the blockchain.

We can use the `casper-client` command line tool to deploy the Counter contract. Before doing so we need to compile the contract and create or import a funded signing account's private key. We'll deploy the contract on the Casper Testnet, so we can use the testnet's faucet to get free test tokens. Start by using `casper-client` to generate an account keypair:

```bash
casper-client keygen .
```

This creates a file *secret_key.pem* containing the private key of a randomly generated account.

From here we can open a Chromium based browser, [install the Casper Wallet](https://chromewebstore.google.com/detail/casper-wallet/abkahkcbhngaebpcgfmhkoioedceoigp), and import the new *secret_key.pem* file. Then you can head to *testnet.cspr.live*, click "More" then "Faucet". Sign in using the Casper Wallet, fill-out the captcha, and click request tokens.

Once the deployment of test tokens succeed, which should take one block, or about 16 seconds, we can prepare the `casper-client` `put-deploy` command which will send the smart contract to the blockchain.

In order to interface with the blockchain, we need the IP address of a live Casper node, which we can obtain from the *Peers* section on *cspr.live*. Click "More", "Connected Peers", then copy the IP address of one of the nodes.

Now open a terminal first compile the smart contract with:

```bash
make prepare
make build-contract
```

then execute `casper-client put-deploy` to deploy the smart contract to the blockchain:

```bash
casper-client put-deploy \
--node-address http://95.216.1.154:7777/rpc \
--chain-name casper-test \
--secret-key ./secret_key.pem \
--payment-amount 40000000000 \
--session-path ./contract/target/wasm32-unknown-unknown/release/contract.wasm
```

Running this command will output a "deploy hash", which is a reference to the deployment on the blockchain. Copying this hexadecimal string and pasting it into *testnet.cspr.live* will show us the execution results.

Now that the deployment succeeded, we can visit the deploying account, navigate to its Named Keys, and copy the contract hash. We can also click on the contract hash, view the smart contract's named keys, click on the "count_key" that we defined in the contract, and note that the count is set to 0, as it should be.

Going back to the terminal we can prepare an invocation of the `increment_count` entrypoint using the same `casper-client` command line tool. In this case, instead of providing a `--session-path`, we can provide a `--session-hash` with the contract hash we copied a minute ago:

```bash
casper-client put-deploy \
--node-address http://95.216.1.154:7777/rpc \
--chain-name casper-test \
--secret-key ./secret_key.pem \
--payment-amount 1000000000 \
--session-hash "" \
--session-entry-point "increment_count"
```

This command will also provide a deploy hash which you can copy and paste into cspr.live to see the results.

Once the calling of the `increment_count` entrypoint has succeeded, we can go to the smart contract's named keys, to the count variable, and discover that the value is now `1`. Every time this entrypoint is invoked, the count will increase.

Now that we've covered the essentials of writing a basic smart contract on the Casper Network, I'd like to open the floor for any questions you may have. Whether it's about the contract we've written today, blockchain specifics, or smart contract development in general, feel welcome to ask.
