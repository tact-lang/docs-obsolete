# Tact spec

Here's a quick recap of some features aimed at multi-actor contracts.

## Named contracts

Contracts are defined using `contract T { ... }` syntax. This permits defining multiple interoperating contracts in one file.

## Storage declaration

Variables are declared in a subset of TL-B with a contemporary syntax. 
Compiler checks that there is enough space for all references.

```
contract T {
    let a: Int32;       // no default value, must be specified within stateinit
    let b: Int32 = 0;   // default value
    let mut c: Int32;   // mutable field
    let x: Ref[X];      // cell references are explicit
}
```

## Methods

Methods are declared with `fn` keyword and an optional `->` for return type:

```
fn <method_name>(<args>) -> <return_type> {

}
```

## Visibility and access control

All the storage and methods are accessible to the contract itself and not visible to outside users.

Storage and methods that should be accessible outside are marked with `pub` prefix:

```
contract Wallet {
   pub let pubkey: PublicKey;
   pub fn get_pubkey() -> PublicKey { return self.pubkey; }
}
```

We will most likely have to adopt [Swift access control model](https://docs.swift.org/swift-book/LanguageGuide/AccessControl.html). 

I suggest we put nuanced options (when needed) as arguments to `pub(...)` keyword, similar to how Rust does `pub(crate)`. E.g. `pub(subtype)` to make the entity visible in the inheriting subtype, or `pub(file)` to make it accessible in the same source file (akin to Swift's `fileprivate`).


## Mutability

All storage is immutable by default and could be assigned only once.

Mutable storage is marked with `mut` keyword.

```
let mut seqno: Int64;
```

Methods cannot mutate the storage unless marked as mutating via `mut` keyword:

```
fn mut receive_external() {
  ...
}
```


## Message routing

Contract may define catch-all handlers for internal and external messages, or specific messages based on "opcode" prefix.

```
contract T {
   action foo#ad9f7d51( ... ) { ... }        // hex prefix
   action bar$010110( ... ) { ... }          // binary prefix
   action(external) baz#...( ... ) { ... }   // external message handler
   action(bounced) foo#...( ... ) { ... }    // handler for the bounced message

   action default( ... ) { ... }             // handler for all unhandled internal messages
   action(external) default( ... ) { ... }   // handler for all unhandled external messages
   action(bounced) default( ... ) { ... }    // handler for all unhandled bounced messages
}
```

Every action under the hood is a `pub fn mut` method with compile-time attributes and added runtime checks (see below).

Internal actions are the default. `action` is a shortcut for `action(internal)`.

To handle raw message and override the default routing one may use `fn mut receive_internal` and `fn mut receive_external`:

```
contract T {
   fn mut receive_internal(msg) { ... }
   fn mut receive_external(msg) { ... }
}
```

"Get methods" are any exported (`pub`) non-mutating methods:

```
contract T {
  pub fn get_pubkey() { ... }
}
```

## self

Keyword `self` refers to the instance of the current contract within a method.

Storage can be accessed via dot `.` operator:

```
self.seqno += 1;
```

Self is mutable in the mutating (`fn mut`) methods.


## msg

Within the action handler keyword `msg` gives access to the `CommonMsgInfo` fields, such as `value` and `sender`.

```
action foo() {
    verify(msg.sender == self.owner);
    verify(msg.value.coins > 10.TON);
    ...
}
```

In the context of sending a message, `msg.*` is used to set some of these parameters.

```
send addr.foo(msg.value: 10.TON);
```


## Sending messages

Messages are sent using `send` keyword to differentiate between ordinary method calls:

```
send addr.message(..., msg.mode: InboundCoins);
```

In here the `addr` could be an address, or a contract instance (stateinit):

```
send Token(master: USDT, owner: user1_addr).transfer(destination: user2_addr);
```

To send a raw message:

```
send_raw_msg(addr: Address, body: Cell, mode: SendMode)
```



## Contract inheritance

The key feature is inheritance of the contracts. This permits packaging common patterns and security features in reusable modules, where compiler prevent incorrect and unsafe use.

```
contract T: X {
   // T inherits storage and methods from X
}
```

Contract inheritance is similar to Swift's inheritance of classes.

* Data layouts are concatenated.
* Methods are concatenated or overriden.
* Constructors


## Constructors and initialization

In TON initialization of a contract is a source of confusion. Initial state of the contract (`StateInit`, a pair of the code and data) defines the contract address and allows the contract to be "deployed" by anyone on the network. More precisely one may think that the contract exists eternally, just its initial data and code are not known to the network until someone publishes them explicitly by attaching the `stateinit` to a message.

Some contracts depend on being "initialized" by authorized parties. E.g. a collectible item must be marked as "initialized" only when its parent collection contract says so. Or a fungible token gets its balance increased only through a message from its minter or a sibling token contract.

Users who might need to define their "on-chain" initializers should not be confused by in-process initialization helpers (constructors).

We suggest using term `constructor` for such in-process helpers that aim at making StateInit. This liberates term `init` for user-defined messages and works nicely w.r.t. TL-B terminology.

```
contract Token {
    let minter: Address;
    let balance: Uint64;
    
    constructor(minter: Address) {
        self.minter = minter;
        self.balance = 0;
    }
}
```

Usage:

```
let t = Token(minter: 0xfad7e94710...);
```

Default constructor is provided when all the fields either have static defaults, or publicly visible.


```
contract Token {
    pub let minter: Address; // public
    let balance: Uint64 = 0; // private, but has default
    
    // auto-generated constructor
    constructor(minter: Address) {
        self.minter = minter;
        self.balance = 0;
    }
}
```

Open questions:

* Sort out how we support multiple constructors (See [Swift initializers](https://docs.swift.org/swift-book/LanguageGuide/Initialization.html)). Maybe we could start with just one constructor per contract for simplicity.

* How to reconcile inheritance and default constructors? If we allow default constructors in a subtype, then would it inherit all parameters from the parent?



## Standard tokens

Standard library comes with three contracts meant to be overriden by the users:

* OnceToken
* UniqueToken
* FungibleToken

These are sorted in the order of flexibility.

`OnceToken` is a "one-shot" instance that may be "issued" and then "returned" back to the issuer. When the main contract needs to store some of its state outside its data storage (e.g. in the user's hands, but not necessarily), it may issue a one-use token. The token may stay idle or permute its state, and when conditions are ready, the token is destroyed and its latest state is securely communicated to the issuing contract.

`UniqueToken` is similar to `OnceToken` but it also has an owner and implements a `transfer` method that allows changing ownership.

`FungibleToken` implements Jetton standard and permits transferring units between instances of these tokens.


## Example: multisig contract

Multisig contract stores pending requests in separate tokens. These "request tokens" accumulate votes from other parties and when completed, signals the contract to forward the requested message.

```
struct SendMessage {
   dest: AnyAddress,
   rawmsg: RawMessage,
   sendmode: SendMode,
}

contract Multisig {
  let parties: Hashmap[Address, Uint32];
  let threshold: Unit32;
  let timeout: TimeInterval = 24*3600;
  let mut request_id: Uint64 = 0;
    
  action initiate_request#ffddeeaa(message: SendMessage) {
    verify(self.parties.contains(msg.sender), "Not authorized");
    
    let reqid = self.request_id.increment();
    
    send Request(owner, reqid, message).init(
           expiration: now() + self.timeout, 
           parties, 
           threshold
         );
  }
  
  action process_request#ffe441(request: Request) {
    let m = request.message;
    send_raw_msg(address: m.dest, message; m.rawmsg, mode: m.sendmode);
  }
}


contract Request: OnceToken {
  let id: Uint32;
  let message: SendMessage;
  let expiration: TimeInterval = 0;
  let mut parties: Hashmap[Address, Uint32] = Hashmap.Empty;
  let mut threshold: Uint32 = 0;
  
  action init(expiration: TimeInterval, parties: Hashmap[Address, Uint32], threshold: Uint32) {
    verify(msg.sender == self.owner);
    self.expiration = expiration;
    self.parties = parties;
    self.threshold = threshold;
  }
  
  // This method should not be reached until Request is initialized
  action vote() {
    verify(now() < self.expiration, "Request expired");
    let weight = self.parties.remove(msg.sender) or fail("Not authorized");
    this.threshold = this.threshold - weight or 0;

    if this.threshold == 0 {
      self.redeem(Multisig.process_request);
    }
  }
  
  clear() {
    verify(now() >= self.expiration, "Request is still active");
    self.destroy();
  }
}
```


where OnceToken implementation is the following:

```
contract OnceToken {
  pub let owner: Address;
  
  pub fn redeem(action: Action()) {
    // TODO: we need to define how "token: Request" is getting deserialized and validated in the context of `Multisig` contract
  }
  
}

```

