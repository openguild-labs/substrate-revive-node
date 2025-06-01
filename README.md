<div align="center">

# Tutorial: Configuring pallet-revive on a Substrate node 

## Objectives

Teach developers how to integrate and configure pallet-revive in a Substrate-based node (such as Asset Hub) to enable live EVM debugging capabilities


<img height="70px" alt="Polkadot SDK Logo" src="https://github.com/paritytech/polkadot-sdk/raw/master/docs/images/Polkadot_Logo_Horizontal_Pink_White.png#gh-dark-mode-only"/>
<img height="70px" alt="Polkadot SDK Logo" src="https://github.com/paritytech/polkadot-sdk/raw/master/docs/images/Polkadot_Logo_Horizontal_Pink_Black.png#gh-light-mode-only"/>

> This is a template for creating a [parachain](https://wiki.polkadot.network/docs/learn-parachains) based on Polkadot SDK.
>
> This template is automatically updated after releases in the main [Polkadot SDK monorepo](https://github.com/paritytech/polkadot-sdk).

</div>

## Table of Contents

- [Building and running the custom node locally](#building-and-running-the-custom-node-locally)

- [How to add pallet-revive on runtime](#how-to-add-pallet-revive-on-runtime)

- [Step by step how to deploy and interact solidity smart contract on PolkaVM ](#step-by-step-how-to-deploy-and-interact-solidity-smart-contract-on-polkavm)


## Building and running the custom node locally

### Step 1: Install Rust 

- 👉 Check the
  [Rust installation instructions](https://www.rust-lang.org/tools/install) for your system.

- 🛠️ Depending on your operating system and Rust version, there might be additional
  packages required to compile this template - please take note of the Rust compiler output.

### Step 2: Fetch parachain template code

```sh
git clone https://github.com/paritytech/polkadot-sdk-parachain-template.git
```

### Step 3: Build the project 

```sh
cargo build --release 
```

### Step 4: Install `polkadot-omni-node`

```sh
cargo install --locked polkadot-omni-node@0.5.0
```

### Step 5:  Install `staging-chain-spec-builder`
```sh
cargo install --locked staging-chain-spec-builder@10.0.0 
```


### Step 6:  Use `chain-spec-builder` to generate the `chain_spec.json` file

```sh
chain-spec-builder create --relay-chain "rococo-local" --para-id 1000 --runtime \
    target/release/wbuild/parachain-template-runtime/parachain_template_runtime.wasm named-preset development
```

### Step 7:  Run Omni Node

```bash
polkadot-omni-node --chain chain_spec.json --dev --dev-block-time 1000
```

### Step 8: Go to Polkadot Explorer JS with local node 

Link: https://polkadot.js.org/apps/?rpc=ws%3A%2F%2F127.0.0.1%3A9944#/explorer


## How to add pallet-revive on runtime

### Step 1: Add `pallet-revive` package in `Cargo.toml` 

```rust
polkadot-sdk = { workspace = true, features = [..., "pallet-revive"], default-features = false }
```

Reference: https://github.com/openguild-labs/substrate-node-revive/commit/2c1bd2d0966331c9f0cd9e5a2db2f0f7fbdbae25

### Step 2: Add `pallet-revive` in  runtime 

+ Define `pallet-revive` in `construct runtime` 
```rust
	// Revive
	#[runtime::pallet_index(60)]Add commentMore actions
	pub type Revive = pallet_revive;
```

+ Implement `pallet-revive` runtime for main Runtime 

```rust
const ETH: u128 = 1_000_000_000_000_000_000;

pub const fn deposit(items: u32, bytes: u32) -> Balance {
	(items as Balance * UNIT + (bytes as Balance) * (5 * MILLI_UNIT / 100)) / 10
}

parameter_types! {Add commentMore actions
	pub ChainId: u64 = u32::from(crate::genesis_config_presets::PARACHAIN_ID) as u64;
	pub const DepositPerItem: Balance = deposit(1, 0);
	pub const DepositPerByte: Balance = deposit(0, 1);
	pub const DefaultDepositLimit: Balance = deposit(1024, 1024 * 1024);
	// 30 percent of storage deposit held for using a code hash.
	pub const CodeHashLockupDepositPercent: Perbill = Perbill::from_percent(30);
	pub const NativeToEthRatio: u32 = (ETH/UNIT) as u32;
}


impl pallet_revive::Config for Runtime {
	type AddressMapper = pallet_revive::AccountId32Mapper<Self>;
	// No runtime dispatchables are callable from contracts.
	type CallFilter = Nothing;
	type ChainExtension = ();
	type ChainId = ChainId;
	type CodeHashLockupDepositPercent = CodeHashLockupDepositPercent;
	type Currency = Balances;
	type DepositPerByte = DepositPerByte;
	type DepositPerItem = DepositPerItem;
	type InstantiateOrigin = EnsureSigned<Self::AccountId>;
	// 1 ETH : 1_000_000 UNIT
	type NativeToEthRatio = NativeToEthRatio;
	// 512 MB. Used in an integrity test that verifies the runtime has enough memory.
	type PVFMemory = ConstU32<{ 512 * 1024 * 1024 }>;
	type RuntimeCall = RuntimeCall;
	type RuntimeEvent = RuntimeEvent;
	type RuntimeHoldReason = RuntimeHoldReason;
	// 128 MB. Used in an integrity that verifies the runtime has enough memory.
	type RuntimeMemory = ConstU32<{ 128 * 1024 * 1024 }>;
	type Time = Timestamp;
	// Disables access to unsafe host fns such as xcm_send.
	type UnsafeUnstableInterface = ConstBool<false>;
	type UploadOrigin = EnsureSigned<Self::AccountId>;
	type WeightInfo = pallet_revive::weights::SubstrateWeight<Self>;
	type WeightPrice = TransactionPayment;
	type Xcm = ();
	type EthGasEncoder = ();
    type FindAuthor = <Runtime as pallet_authorship::Config>::FindAuthor;
}

impl TryFrom<RuntimeCall> for pallet_revive::Call<Runtime> {
	type Error = ();

	fn try_from(value: RuntimeCall) -> Result<Self, Self::Error> {
		match value {
			RuntimeCall::Revive(call) => Ok(call),
			_ => Err(()),
		}
	}
}
```
Reference: https://github.com/openguild-labs/substrate-node-revive/commit/f6a0b5feb8516f0a8061e692b367ed0cb46e500a


### Step 3: Convert `eth` transaction to extrinsic substrate type 

```rust
/// EthExtra converts an unsigned Call::eth_transact into a CheckedExtrinsic.
#[derive(Clone, PartialEq, Eq, Debug)]
pub struct EthExtraImpl;

impl pallet_revive::evm::runtime::EthExtra for EthExtraImpl {
    type Config = Runtime;
    type Extension = TxExtension;

    fn get_eth_extension(nonce: u32, tip: Balance) -> Self::Extension {
        (
            frame_system::CheckNonZeroSender::<Runtime>::new(),
            frame_system::CheckSpecVersion::<Runtime>::new(),
            frame_system::CheckTxVersion::<Runtime>::new(),
            frame_system::CheckGenesis::<Runtime>::new(),
            frame_system::CheckMortality::from(generic::Era::Immortal),
            frame_system::CheckNonce::<Runtime>::from(nonce),
            frame_system::CheckWeight::<Runtime>::new(),
            pallet_transaction_payment::ChargeTransactionPayment::<Runtime>::from(tip),
            frame_metadata_hash_extension::CheckMetadataHash::<Runtime>::new(false),
        )
            .into()
    }
}

/// Unchecked extrinsic type as expected by this runtime.
pub type UncheckedExtrinsic =
    pallet_revive::evm::runtime::UncheckedExtrinsic<Address, Signature, EthExtraImpl>;

```
> **_NOTE:_**  Unchecked Extrinsic  : these are signed transactions that require some validation check before they can be accepted in the transaction pool. Any unchecked extrinsic contains the signature for the data being sent plus some extra data.


Reference: https://github.com/openguild-labs/substrate-node-revive/commit/59dcf99b2c8328d661d7fa84ab863e5a09a71965




### Step 4: Add revive runtime API

+ Get evm balance 
+ Get block gas limit 
+ Get gas price 
+ Get nonce 
+ `eth_transact` 
+ `call` 
+ `instantiate` 
+ Upload code 
+ Get on-chain storage
+ Trace  

Reference: https://github.com/openguild-labs/substrate-node-revive/commit/328bdf3b79b691de0e12da72270379097592b686

### Step 5: Fix bug 

Reference : https://github.com/openguild-labs/substrate-node-revive/commit/72ccd3bdba2d6ef8469e057e275b9df8bdcb98e2



## Step by step how to deploy and interact solidity smart contract on PolkaVM

### Step 1: Install `revive-eth-rpc` 

```sh
cargo install pallet-revive-eth-rpc@0.4.0
```

### Step 2: Run `substrate-revive-node` and `revive-eth-rpc` in dev mode 

```sh
polkadot-omni-node --chain chain_spec.json --dev -lruntime::revive=debug --dev-block-time 1000
```

```sh
./eth-rpc --dev
```


### Step 3: Check RPC 

```sh
curl -X POST http://127.0.0.1:8545 \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc":"2.0",
    "method":"eth_getBlockByNumber",
    "params":["latest", false],
    "id":1
  }'
```

Result: 

```bash
{"jsonrpc":"2.0","id":1,"result":{"baseFeePerGas":"0x3e8","difficulty":"0x0","extraData":"0x","gasLimit":"0xab87c85a2e8","gasUsed":"0x0","hash":"0x79ff71b978bab8f2e4cd3d6d3e8163d8869427d7d5061846f38c0cc3a67bb453","logsBloom":"0x00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000","miner":"0x0000000000000000000000000000000000000000","mixHash":"0x0000000000000000000000000000000000000000000000000000000000000000","nonce":"0x0000000000000000","number":"0x3b9","parentHash":"0x33c1cdc034d00f814119c3fd2f10e5a31b2b8e7aec6d5689739cd3a3d9e0982f","receiptsRoot":"0x4a9a161b4ff8849269a6a75e08fbbbbc0ca80eede262c8bf035bdbfe6dc5749a","sha3Uncles":"0x0000000000000000000000000000000000000000000000000000000000000000","size":"0x0","stateRoot":"0x45145e2f329608888ec06ea39fcecade07ee59903060e65290698e0e9df4e4a7","timestamp":"0x0","transactions":[],"transactionsRoot":"0x4a9a161b4ff8849269a6a75e08fbbbbc0ca80eede262c8bf035bdbfe6dc5749a","uncles":[]}}
```

### Step 4: Add local network in metamask 

| Properties | Network Details |
|------------|----------------|
| Network name | Substrate Node Revive |
| RPC | http://127.0.0.1:8545 |
| Chain ID | 1000 |
| Symbol | REVIVE |



### Step 5: Mapping Substrate's Account to ETH account 

Go to Polkadot JS Explorer -> Developer -> Choose `Revive` pallet -> Choose `mapAccount` extrinsic

> **_NOTE:_**  Alice's Substrate PrivateKey ( dev purpose)  : 0xe5be9a5092b81bca64be81d212e7f2f9eba183bb7a90954f7b76361f6edb5c0a

### Step 6: Transfer funds to ETH Account using Polkadot JS explorer 


Go to Polkadot JS Explorer -> Developer -> Choose `Revive` pallet -> Choose `call` extrinsic

| Params  |  Value |
|------------|----------------|
| dest | Is your ETH address  |
| value | 1000000000000000(1000 REVIVE) |
| storageDepositLimit | 1000000000000000 |
| data | 0x |

### Step 7: Deploy simple smart contract using Remix Polkadot 

Link: https://remix.polkadot.io/#lang=en&optimize=false&runs=200&evmVersion=null&version=soljson-v0.8.28+commit.7893614a-revive-0.1.0-dev.12.js







