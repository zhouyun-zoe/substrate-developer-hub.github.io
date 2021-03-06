---
title: Constructing a Runtime!
id: version-pre-v2.0-3e65111-construct_runtime
original_id: construct_runtime
---

The `construct_runtime!` macro is where you declare all the runtime pallets you would like to include into your blockchain's runtime. This includes any pallets from FRAME or even custom pallets you may have written.

You can find an example of a `construct_runtime!` definition from the Substrate [node-template](https://github.com/paritytech/substrate/blob/master/bin/node-template/runtime/src/lib.rs):

```rust
construct_runtime!(
	pub enum Runtime with Log(InternalLog: DigestItem<Hash, Ed25519AuthorityId>) where
		Block = Block,
		NodeBlock = opaque::Block,
		UncheckedExtrinsic = UncheckedExtrinsic
	{
		System: system::{default, Log(ChangesTrieRoot)},
		Timestamp: timestamp::{Module, Call, Storage, Config<T>, Inherent},
		Consensus: consensus::{Module, Call, Storage, Config<T>, Log(AuthoritiesChange), Inherent},
		Aura: aura::{Module},
		Indices: indices,
		Balances: balances,
		Sudo: sudo,
	}
);
```

## Defining the Pallets Used

In the example above, we only use FRAME pallets, but you can see that there are different syntactical ways that you can define the use of a pallet and its types. These different patterns are managed by the [macro definition](https://github.com/paritytech/substrate/blob/master/frame/support/src/runtime.rs).

### Naming Your Pallet

Each line in your `construct_runtime!` macro defines a pallet you are using in your runtime.

```rust
System: system::{default, Log(ChangesTrieRoot)},
```

The lowercase `system` points to the Rust pallet where you have defined your runtime logic (following the standard Rust style guidelines). The uppercase `System` is a friendly name that you can define for your pallet which gets exposed through the runtime APIs.

### Supported Types

The `construct_runtime!` macro provides support for the following types in a pallet:

 - [`Module`](#module)
 - [`Call`](#call)
 - [`Storage`](#storage)
 - [`Event`](#event) or `Event<T>` (if the event is [generic](https://doc.rust-lang.org/rust-by-example/generics.html))
 - [`Origin`](#origin) or `Origin<T>` (if the origin is [generic](https://doc.rust-lang.org/rust-by-example/generics.html))
 - [`Config`](#config) or `Config<T>` (if the config is [generic](https://doc.rust-lang.org/rust-by-example/generics.html))
 - [`Log`](#log)
 - [`Inherent`](#inherent)

### No Types or `default`

In the definition of the `Balances` or `Sudo` pallet, you see that we do not specify any of the types used within the pallet. In this case, the macro automatically expands this definition to include the following:

```rust
Module, Call, Storage, Event<T>, Config<T>
```
    
So these two lines are equivalent:

```rust
Balances: balances,
// Same as...
Balances: balances::{Module, Call, Storage, Event<T>, Config<T>},
```    

If you want to include these default types in addition to other types you define, you can use the `default` keyword as shown in the `System` module. Thus these two lines are equivalent as well:

```rust
System: system::{default, Log(ChangesTrieRoot)},
// Same as...
System: system::{Module, Call, Storage, Event<T>, Config<T>, Log(ChangesTrieRoot)},
```
    
## When To Use Different Types

As you can see, the types exposed by your various pallets are what ultimately power the runtime. Most of these types are automatically generated by other macros used in the runtime development.

### Module

The `Module` type is required by all pallets in your runtime. This type gets generated through the `decl_module!` macro. This is where all the public functions your pallet exposes are defined. For example, in the [`Sudo`](https://github.com/paritytech/substrate/blob/master/frame/sudo/src/lib.rs) pallet which controls the management of "admin" access to your chain:

```rust
decl_module! {
	// Simple declaration of the `Module` type. Lets the macro know what its working on.
	pub struct Module<T: Trait> for enum Call where origin: T::Origin {
		fn deposit_event<T>() = default;

		fn sudo(origin, proposal: Box<T::Proposal>) {
			// This is a public call, so we ensure that the origin is some signed account.
			let sender = ensure_signed(origin)?;
			ensure!(sender == Self::key(), "only the current sudo key can sudo");

			let ok = proposal.dispatch(system::RawOrigin::Root.into()).is_ok();
			Self::deposit_event(RawEvent::Sudid(ok));
		}

		fn set_key(origin, new: <T::Lookup as StaticLookup>::Source) {
			// This is a public call, so we ensure that the origin is some signed account.
			let sender = ensure_signed(origin)?;
			ensure!(sender == Self::key(), "only the current sudo key can change the sudo key");
			let new = T::Lookup::lookup(new)?;

			Self::deposit_event(RawEvent::KeyChanged(Self::key()));
			<Key<T>>::put(new);
		}
	}
}
```

You can learn more about `decl_module!` [here](runtime/macros/decl_module.md).

### Call

The `Call` type is an enum generated by the `decl_module!` macro which contains a list of all the callable functions and their arguments. For example, in the [`Sudo`](https://github.com/paritytech/substrate/blob/master/frame/sudo/src/lib.rs) pallet above, we should expect an enum similar to:

```rust
enum Call<T> {
    sudo(proposal: Box<T::Proposal>),
    set_key(new: <T::Lookup as StaticLookup>::Source),
}
```

We will dig deeper into this topic [here] (Coming Soon).

### Storage

The `Storage` type is exposed whenever your runtime uses the `decl_storage!` macro. This macro is used to save data used by your pallet into the chain state.

We can look at the [`Sudo`](https://github.com/paritytech/substrate/blob/master/frame/sudo/src/lib.rs) pallet to see an example:

```rust
decl_storage! {
	trait Store for Module<T: Trait> as Sudo {
		Key get(key) config(): T::AccountId;
	}
}
```

You can learn more about [`decl_storage!`](https://substrate.dev/rustdocs/pre-v2.0-3e65111/frame_support_procedural/macro.decl_storage.html)

### Event

The `Event` type needs to be exposed whenever your pallet deposits events using the `decl_event!` macro. Events can be used to easily report changes or conditions in your runtime to external entities like users, chain explorers, or dApps.

If you want to expose generic types defined by your pallet, you will need to make your type generic (`Event<T>`), as shown here:

```rust
/// An event in this pallet.
decl_event!(
	pub enum Event<T> where AccountId = <T as system::Trait>::AccountId {
		/// A sudo just took place.
		Sudid(bool),
		/// The sudoer just switched identity; the old key is supplied.
		KeyChanged(AccountId),
	}
);
```

Otherwise, you can simply use `Event` for non-generic types. You can learn more about `decl_event!` [here] (Coming Soon).

### Origin

The `Origin` type is needed whenever your pallet declares a custom `Origin` enum to be used within your pallet.

Every function call to your runtime has an origin which specifies where the extrinsic was generated from. In the case of a signed extrinsic (transaction), the origin contains an identifier for the caller. The origin can be empty in the case of an inherent extrinsic.

You can see an example of a customized `Origin` in the [council motion pallet](https://github.com/paritytech/substrate/blob/master/frame/council/src/motions.rs):

```rust
/// Origin for the council pallet.
#[derive(PartialEq, Eq, Clone)]
#[cfg_attr(feature = "std", derive(Debug))]
pub enum Origin {
	/// It has been condoned by a given number of council members.
	Members(u32),
}
```

We will dig deeper into this topic [here] (Coming Soon).

### Config

The `Config` type is needed whenever your pallet defines an initial state in the genesis configuration. By default, all storage variables are left in an 'unassigned' state, but using the `config()` keyword while declaring the storage variable allows you to initialize that state.

An example of that initialization can be found in the Substrate `node-template`, where Alice is set as the upgrade `Key` for `Sudo`. You can see from the `Storage` section that `Key` does have the `config()` parameter, and the value for it gets set in the `GenesisConfig` within [chain_spec.rs](https://github.com/paritytech/substrate/blob/master/bin/node-template/src/chain_spec.rs):

```rust
GenesisConfig {
    ...
    sudo: Some(SudoConfig {
        key: root_key,
    }),
}
```

We will dig deeper into this topic [here] (Coming Soon).

### Log

The `Log` type is needed if your pallet generates logs entries for the runtime. Within the pallet, you must expose both a `Log` type and a `RawLog` enum which defines what the pallet is logging. Logs must satisfy the following requirements:

1) The binary representation of all supported 'system' log items should stay the same. Otherwise, the native code will be unable to read log items generated by previous runtime versions.

2) The support of 'system' log items should never be dropped by runtime. Otherwise, native code will lose its ability to read items of this type even if they were generated by the versions which have supported these items.

The [`Consensus` pallet](https://github.com/paritytech/substrate/blob/master/frame/consensus/src/lib.rs) shows an example of how to implement a log:

```rust
pub type Log<T> = RawLog<
	<T as Trait>::SessionKey,
>;

/// Add logs in this pallet.
#[cfg_attr(feature = "std", derive(Serialize, Debug))]
#[derive(Encode, Decode, PartialEq, Eq, Clone)]
pub enum RawLog<SessionKey> {
	/// Authorities set has been changed. Contains the new set of authorities.
	AuthoritiesChange(Vec<SessionKey>),
}

impl<SessionKey: Member> RawLog<SessionKey> {
	/// Try to cast the log entry as AuthoritiesChange log entry.
	pub fn as_authorities_change(&self) -> Option<&[SessionKey]> {
		match *self {
			RawLog::AuthoritiesChange(ref item) => Some(item),
		}
	}
}
```

It is customary to expose a `deposit_log()` function within your pallet which creates a log:

```rust
/// Deposit one of this pallet's logs.
fn deposit_log(log: Log<T>) {
    <system::Module<T>>::deposit_log(<T as Trait>::Log::from(log).into());
}
```

We will dig deeper into this topic [here] (Coming Soon).

### Inherent

The `Inherent` type is needed when your pallet defines an implementation of the `ProvideInherent` trait.

This custom implementation is needed whenever your pallet wants to provide an inherent extrinsic and/or wants to verify an inherent extrinsic. If your pallet requires extra data for creating the inherent extrinsic, the data needs to be passed as `InherentData` into the runtime.

For example, in the [`Timestamp` pallet](https://github.com/paritytech/substrate/blob/master/frame/timestamp/src/lib.rs), an inherent data is used to set the timestamp for a given block by creating an extrinsic. As a peer authority of the block author, we confirm that the time proposed in the inherent extrinsic is within an acceptable period from our clock.

```rust
impl<T: Trait> ProvideInherent for Module<T> {
	type Call = Call<T>;
	type Error = InherentError;
	const INHERENT_IDENTIFIER: InherentIdentifier = INHERENT_IDENTIFIER;

	fn create_inherent(data: &InherentData) -> Option<Self::Call> {
		let data: T::Moment = extract_inherent_data(data)
			.expect("Gets and decodes timestamp inherent data")
			.saturated_into();

		let next_time = cmp::max(data, Self::now() + <MinimumPeriod<T>>::get());
		Some(Call::set(next_time.into()))
	}

	fn check_inherent(call: &Self::Call, data: &InherentData) -> result::Result<(), Self::Error> {
		const MAX_TIMESTAMP_DRIFT: u64 = 60;

		let t: u64 = match call {
			Call::set(ref t) => t.clone().saturated_into::<u64>(),
			_ => return Ok(()),
		};

		let data = extract_inherent_data(data).map_err(|e| InherentError::Other(e))?;

		let minimum = (Self::now() + <MinimumPeriod<T>>::get()).saturated_into::<u64>();
		if t > data + MAX_TIMESTAMP_DRIFT {
			Err(InherentError::Other("Timestamp too far in future to accept".into()))
		} else if t < minimum {
			Err(InherentError::ValidAtTimestamp(minimum))
		} else {
			Ok(())
		}
	}
}
```

We will dig deeper into this topic [here] (Coming Soon).
