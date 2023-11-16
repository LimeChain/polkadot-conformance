# Host Conformance Testing Suite using Zombienet Research

# Abstract

The research explores the utilization of Zombienet to create a host-agnostic conformance testing suite. We examine the necessary modifications to Substrate and Zombienet to facilitate testing across several Host modules, namely Trie Host API, State Trie, SCALE, BABE, and GRANDPA. The study is structured around the AAA pattern to present findings in a cohesive manner. Our results suggest that Zombienet shows promise for testing Trie Host API and State Trie through custom RPC methods, while BABE and GRANDPA benefit from a logging approach. Testing SCALE is more effective using an alternative approach. However, the development of a state transition tool or feature is essential before conducting any black-box Host testing. This research contributes to advancing host-agnostic conformance testing for Polkadot and blockchain systems in general, addressing specific knowledge gaps and providing practical insights into the field.

# **Introduction**

The Polkadot Protocol Conformance Tests Research Proposal responds to the open Polkadot Protocol Conformance Tests RFP, addressing the need for a comprehensive and fully automated testing suite in the Polkadot ecosystem. Polkadot has witnessed remarkable growth in recent years, with multiple Host implementations and a commitment to decentralization. However, with increased diversity comes the challenge of ensuring protocol compliance and functionality across these implementations. This research aims to address these challenges and propose a way forward towards a more robust, flexible, and extensible conformance testing suite.

## Content **Structure**

The research is structured around the following topics:

1. **Framing the Research**: The key conceptual requirements that will make or break the research outcome. Having these requirements met, it’d be feasible for Zombienet to be used as a framework for executing general conformance tests.
2. **Host API Research**: This milestone focuses on investigating the feasibility of using Zombienet as the framework for executing Host API tests, ensuring that these critical functions can be adequately tested in a host-agnostic manner.
3. **SCALE Research**: The research seeks to assess whether Zombienet can effectively conduct SCALE encoding and decoding tests, vital for efficient data transmission and communication within the Polkadot network.
4. **State Trie Research**: The evaluation of the State Trie component includes both encoding and decoding. This milestone will determine if Zombienet is suited to handle State Trie-related conformance tests.
5. **BABE Research**: BABE, an integral part of Polkadot's consensus module, requires validation and block import. The research will explore the compatibility of Zombienet with conducting BABE tests, considering the intricacies of block authorship and state changes.
6. **GRANDPA Research**: In the case of GRANDPA, a finalization protocol, the research will investigate the challenges of testing various conditions, such as chain finality, chain length, and primary block counts, using Zombienet.
7. **Conclusion**: The team will summarize the findings from the preceding topics and come to a conclusion whether Zombienet can be used to conduct conformance tests.
8. **Recommendations**: Being the final part of the research, the team will give our view of the way forward, towards building the conformance suite.

The outcomes of this research will provide valuable insights into the adaptability and suitability of Zombienet for Polkadot conformance testing. This research document aims to lay the groundwork for a potential implementation of the testing suite, ultimately contributing to the continued development and reliability of the Polkadot network.

# Framing the Research

When considering the research goal and topics from a more abstract perspective, one would likely conclude that the task is to test internal modules using a framework that lacks explicit access to or knowledge of these internals i.e black box testing. Therefore, in order to enable testing of that logic, there must be a channel though which, the Host and the framework can respectively read and write information without compromising the security characteristics of the former. BABE and GRANDPA are particularly interesting modules because they are hard to be put into a specific state. Consequently, another key moment that must be considered as part of the research is how the Host will arrive at the state in order to execute the test. Without a reliable approach to state transitions, the task would appear insurmountable.

The research has led the team to explore several approaches, which are defined below and discussed in more detail in the following sections.

## The AAA Pattern

The AAA (Arrange-Act-Assert) pattern has become a standard across the testing industry. It suggests that you should divide your test method into three sections: arrange, act and assert. Each one of them only responsible for the part in which they’re named after. Our team applied the AAA pattern to put frame around our research endeavours which led to a more comprehensible  approach.

### Arrange: Transitioning to the correct Host initial state

Reliable initial state replication of a test scenario is a critical characteristic of any conformance suite. Since every test case requires a specific chain state and probably other environment variables for the test scenario to be executed consistently, being able to perform such a transition requires a few steps:

**Transitioning**

Currently, Hosts don't provide any out-of-the-box mechanism to easily transition to a desired trie state. In other protocols this has been implemented using a tool that accepts a list of transactions or a definition of the desired state. In Polkadot, this means that after a test scenario is defined, the developer would have to launch a new node and manually execute transactions until the desired state is reached. This would be unreliable and time-consuming. For the purposes of the research, we assumed that such a tool exists and at the end of the document, the team will propose a solution to this challenge.

**State export & import**

When combined, Zombienet and Polkadot Host provide a way to export and import state using database snapshots, a tutorial for that can be found [here](https://github.com/paritytech/polkadot-sdk/blob/master/substrate/zombienet/0001-basic-warp-sync/README.md). What’s better is that Zombienet supports loading state from custom database paths. It’s done using a not-so-well-documented parameter `db_snapshot`, which we found used [here](https://github.com/paritytech/polkadot-sdk/blob/master/substrate/zombienet/0001-basic-warp-sync/test-warp-sync.toml). Combined together, these features allow for Hosts to share database snapshot across test runs and be tested deterministically.

<aside>
⚠️ Because this approach uses database snapshots, it’s not 100% host-agnostic. For example, if Substrate exports it’s RocksDB snapshot, it won’t be importable in Gossamer because it uses LevelDB.

</aside>

## Act & Assert: Invoking and validating specific Host functions

Another challenge that any framework for testing a Host as a black box would face is function invocation and asserting. In order to overcome this limitation in the proposal, the team explores two options that are appropriate in different scenarios.

### Secure information channel from Host to Zombienet

Information about the internal state of Host modules can be exposed in a variety of ways. The team found 3 approaches to be suitable for creating a testing suite using Zombienet. In this section, they’re given a brief description, whereas in the following sections, they’re explored in more depth. 

<aside>
💡 Note: Examples of each approach can be found in the following research sections.

</aside>

**Log assertions**

The team believes that a simple, yet effective solution that doesn't add much overhead would be to add a special `conformance` feature flag. When a client is run with this flag, additional logs will be printed during the execution of functions of interest to the conformance suite. 

As a prerequisite, the team will have to craft precise chain states that will put the node in the desired state. Once this is achieved, the Host will be started and expected to naturally “**act”** and be put in the desired **assert** state. Following that, looking at the logs would be sufficient to assert on the behaviour of the node, as shown in some of the following research topics.

This methodology, however, has its drawbacks. The main one is that scenario crafting is going to be hard and unpredictable. Since Polkadot Hosts are a complex piece of software, there’s a high chance that some piece of logic may result in false or bad testing results. This makes the approach time consuming and unreliable for complex test cases. Another con is that this approach doesn’t solve the **act** phase, meaning that it should be paired with another method to **invoke** the functionality of interest.

**Conformance RPC methods**

The second approach requires development on each Host’s side. Each one should add a predefined list of additional RPC methods and expose the endpoints only when the `conformance` flag is enabled. Having RPC methods works around the black box problem. The tester still doesn’t know how the function handles, but is able to **invoke** specific functions and **assert** on their result. A similar approach has been successfully implemented in projects like [retesteth](https://github.com/ethereum/retesteth/wiki/RPC-Methods), proving its effectiveness in ensuring reliable testing. When combined with careful test scenario crafting, this approach should be able to cover complex test cases.

The main drawback is that it’s slower to implement than the previous one. In order to enable a conformance RPC method, one would first have to implement the said method in the specific Host implementation and provide access to the internal module. In some cases this won’t bloat the binary size if implemented behind a compile-time feature flag in Rust. Following, an update to Polkadot.js would have to be made in order to include the new method. Finally, Zombienet would have to update it’s dependency on Polkadot.js in order to be able to invoke the function from a Zombienet test. Alternatively, RPC methods can be invoked without Polkadot.js using HTTP requests.

# Trie **Host API Research**

## **Scope of Research**

The goal of this section is to prove or disprove that the requirements described in the previous section can be completed. While the primary focus of this research is the specific Trie Host API, it is important to note that our team firmly believes that the methodology used here can be extended to encompass the majority of other Host API families featuring **pure** functions. The principles and practices developed during this research may serve as a valuable framework for testing and verification across a broader spectrum of Host APIs.

## Research Findings

Trie Host API consists of the following functions:

- `ext_trie_blake2_256_root` - Compute a 256-bit Blake2 trie root formed from the iterated items.
- `ext_trie_blake2_256_ordered_root` - Compute a 256-bit Blake2 trie root formed from the enumerated items.
- `ext_trie_keccak_256_root` - Computes a 256-bit Keccak trie root formed from the iterated items.
- `ext_trie_keccak_256_ordered_root` - Computes a 256-bit Keccak trie root formed from the enumerated items.
- `ext_trie_blake2_256_verify_proof` - Verifies a key/value pair against a Blake2 256-bit merkle root.
- `ext_trie_keccak_256_verify_proof` - Verifies a key/value pair against a Keccak 256-bit merkle root.

All Trie Host API functions are inherently **pure,** meaning that they produce the same output when provided with the same input. This purity attribute renders them deterministic, simplifying the creation of test cases as they are independent of the Host’s internal state.

### Testing with logs

Assuming the initial state is successfully exported and loaded, the **assertion** poses no issues. It simply involves adding log entries at the beginning and end of each Host API function. These logs would document the input and output, facilitating testing and verification. Afterwards, assertions can be performed based on the logs determining whether the result is correct.

**Original Substrate code:**

```rust
/// Conduct a 256-bit Blake2 hash.
fn blake2_256(data: &[u8]) -> [u8; 32] {
	sp_core::hashing::blake2_256(data)
}
```

**Modified Substrate code with added conformance logs:**

```rust
/// Conduct a 256-bit Blake2 hash.
fn blake2_256(data: &[u8]) -> [u8; 32] {
	log::trace!(
		target: "conformance",
		"blake2_256_input: {}", hex::encode(&data),
	);
	let result = sp_core::hashing::blake2_256(data);
	log::trace!(
		target: "conformance",
		"blake2_256_output: {}", hex::encode(&result),
	);
	result
}
```

**Zombienet configuration and test:**

```jsx
[relaychain]
default_command = "polkadot"
default_args = ["--conformance"]
chain_spec_path = "blake2_256_conformance_chain_spec.json"

  [[relaychain.nodes]]
  name = "alice"
  validator = false
  db_snapshot="blake2_256_snapshot.tgz"
```

```jsx
Description: Blake2 256 Conformance
Network: ./0001-blake2_256_conformance.toml
Creds: consfig

# Check authority status.
alice: reports node_roles is 4

# Check for matching input
alice: log line matches glob "*blake2_256_input: 96c2a38c16b0403b5f7a09ecaf1e2d76d2c226538e343670a251bae9df26830362b07092ef01381789a2718c687cbe1ac8a8b97f918abde4c5a89930c0f561fc51fae18a67ce040c3fea81bb8e0f94d6" within 60 seconds
# Check for matching output
alice: log line matches glob "*blake2_256_output: c8a8b97f918abde4c5a89930c0f561fc" within 60 seconds
```

However, the Host API differs from other Host modules, primarily because its functions are triggered by the Runtime. This mechanic introduces complexities in devising test scenarios, as engineers must consider the specific circumstances under which a Host API function will be invoked. To address this challenge, the team suggests two solutions that would resolve the invocation issue:

- **Conformance Runtime**: This custom specialized runtime would grant access to trigger Host functions on-demand, with custom input, resulting in streamlining the testing process through Zombienet.
- **RPC Conformance methods**: These RPC methods would provide access to each Host API, and as mentioned above, the result can easily be asserted in a Zombienet test. More details on this approach can be found in the following section.

### Testing with RPC methods

As mentioned already, this approach would expose several new endpoints, one for each Host API function. This solves the challenge of invoking specific methods. 

**Example Trie API Blake2 256 Root RPC Request**:

```json
{
   "id": 1,
	"jsonrpc": "2.0",
  "method": "test_trieApiBlake256Root",
  "params": {
    "data": "0xecondedData",
  },
  "id": 1
}
```

**Example Substrate RPC trait**:

```json
#[rpc(client, server)]
pub trait TrieStateApi<Hash> {
	/// Call Trie Host API's Blake256 Root method
	#[method(name = "test_trieApiBlake256Root")]
	fn blake256Root(&self, data: String) -> RpcResult<String>;
}
```

**Example Trie API Blake2 256 Root Response**:

```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "result": "0xtrieRootResult"
}
```

# **SCALE Research**

## Scope of Research

The SCALE codec plays a pivotal role in ensuring the seamless operation of the Host. It is extensively utilised across various Host modules, with particular prominence in two key areas: the P2P and Host API modules.

In the P2P module, the SCALE codec is responsible for encoding and decoding network messages, enabling the exchange of data between network participants. Similarly to the Host API module, SCALE is used to encode and decode input and output data that flows between the Host and the Runtime. The research primarily focuses on developing an effective approach for testing SCALE within these two critical scenarios.

## Research Findings

### Initial Setup

All tests will be run through a JavaScript script and require only a simple network. The research will be using the following template for all methods:

```
Description: Scale Testing
Network: ./scale_network.toml
Creds: config

alice: js-script ./scale.js within 10 seconds
```

`simple_network.toml`:

```toml
[relaychain]
default_command = "polkadot"
default_args = ["--conformance"]

[[relaychain.nodes]]
name = "alice"
validator = true
```

To expose SCALE we can add additional RPC methods in the Substrate by following the steps below.

### Testing with HTTP RPC calls

We can add the SCALE testing methods to an RPC module. To do this in Substrate we can add them to an already existing trait (ideally in the future a new RPC group will be made) and then add it to the implementation.

```rust
/// Substrate system RPC API
#[rpc(client, server)]
pub trait SystemApi<Hash, Number> {
	#[method(name = "scale_decoding_and_encoding")]
	fn scale_decoding_and_encoding(&self, complex: String) -> RpcResult<()>;

	#[method(name = "scale_encoding")]
	fn scale_encoding(&self) -> RpcResult<()>;
```

```rust
use codec::{Encode, Decode};

/// Have a variety of fields for actual testing
#[derive(Encode, Decode)]
pub struct ComplexType {
	pub normal_integer: u64
}

#[async_trait]
impl<B: traits::Block> SystemApiServer<B::Hash, <B::Header as HeaderT>::Number> for System<B> {
	/// Scale Decoding and Decoding test
	/// We cannot test decoding by itself unless we encode it to some other format
	/// We could format the values individually and not as a struct, but things like
	/// lists are still going to be debugged differently in different languages
	fn scale_decoding_and_encoding(&self, complex: String) -> RpcResult<String> {
		let bytes = hex::decode(complex.as_str()).unwrap();
		let decoded: ComplexType = ComplexType::decode(&mut bytes.as_slice()).unwrap();
		Ok(hex::encode(decoded.encode()))
	}

	/// Scale Encoding
	fn scale_encoding(&self) -> RpcResult<String> {
		let complex = ComplexType {
			normal_integer: 42
		};
		Ok(hex::encode(complex.encode()))
	}
```

The Zombienet script will then make an RPC call via HTTP to the newly added methods:

```jsx
const assert = require("assert");

async function run(nodeName, networkInfo, args) {
  const {wsUri, userDefinedTypes} = networkInfo.nodesByName[nodeName];

  let scale_encode_result = await (await fetch("http://" + wsUri.slice(5), {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json'
    },
    body: '{"jsonrpc":"2.0","id":"id","method":"scale_encoding","params":[]}'
  })).text();

  if (scale_encode_result !== '{"jsonrpc":"2.0","result":"2a00000000000000","id":"id"}') {
    assert.fail("Result does not match");
  }

  assert.ok("Test passed");
}

module.exports = { run }
```

<aside>
⚠️ Attempting to define and call custom RPC methods via the exposed [ApiPromise](https://polkadot.js.org/docs/api/start/rpc.custom/) through the `zombie` helper yielded unsuccessful results, causing the script to timeout. Other attempts resulted in the methods just not being detected.

</aside>

### Testing with logs

To test with logs, we need our RPC calls to log the SCALE encoding instead of returning it as a result. This can be done by modifying our previous call to remove the `→ RpcResult<String>` from the trait definition and replacing it with a line to log like this:

```rust
log::info!(target: "conformance", "scale_encoding: {}", hex::encode(complex.encode()));
```

After we invoke the javascript for calling the RPC method we can add a line to the end of the Zombienet test asserting a matching log like this:

```jsx
alice: log line matches glob "*scale_encoding: 2a00000000000000*" within 10 seconds
```

# **State Trie Research**

## **Scope of Research**

The State Trie plays a crucial role in storing the state of the Polkadot chain. It serves as a critical component for ensuring the integrity of the Host's state and, by extension, the entire Polkadot network. This integrity is maintained by calculating the State Trie’s Merkle Root with every block inclusion.

The research specifically focuses on two fundamental aspects of the State Trie: the encoding and decoding of the Trie. The primary areas of logic investigated include:

- Encoding and decoding Node Headers
- Encoding and decoding Leaf Nodes
- Encoding and decoding Children Nodes
- Encoding and decoding Merkle Root

## Research Findings

### Testing with logs

Applying a logging approach similar to the one used in the Host API was considered. However, this approach presented challenges, mainly due to the multitude of variables involved that can influence the state trie logic. These variables such as different genesis specs can introduce unpredictability in trie node values, potentially leading to flaky tests, an undesirable outcome in the context of conformance testing. Although, if state transitions can be performed deterministically, the team believes that logging would be a viable solution.

### Testing with RPC Methods

The team suggests that a more effective approach for testing the State Trie would involve the implementation of specific RPC calls. These calls would allow for the invocation of State Trie functions and the execution of tests. By implementing RPC calls tailored to the State Trie functions, the testing process can be more controlled and predictable, reducing the risk of unreliable test results and contributing to the overall quality of the conformance testing suite.

**Example Trie Decode RPC Request**:

```json
{
   "id": 1,
	"jsonrpc": "2.0",
  "method": "test_stateTrieDecode",
  "params": {
    "data": "0xdata",
  },
  "id": 1
}
```

**Example Trie Decode RPC Response**:

```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "result": {
		"key": "0xkey"
		"value": "0xvalue",
		"children": [...],
		...
	}
}
```

**Example Merkle Root Decode Request**:

```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "method": "test_trieRootDecode",
  "params": {
    "data": {
			"proofs": ["0xproof1", "0xproof2"],
			"rootHash": ["0xrootHash"] 
		},
  },
}
```

**Example Merkle Root Decode Response**:

```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "result": {
		"key": "0xkey"
		"value": "0xvalue",
		"children": [...],
		...
	}
}
```

Following these examples, Trie encode and Merkle root encode methods would have the a similar implementation. The main difference is that the request and response body switched.

# **BABE Research**

## Scope of Research

BABE, serves as the block production engine. It functions on a slot-based algorithm, effectively breaking time into epochs, with each epoch further divided into slots. Each slot is designated for the creation of a block by one or more authors. These authors are classified into primary and secondary categories.

**Primary Authors** are randomly assigned slots using a verifiable random function (VRF). This function takes into account the **author's private key**, **an epoch randomness value** (derived from the VRF values of the blocks in the epoch two cycles before the current one), and the **slot number**. The VRF generates a pseudo-random number along with proof of its proper generation. This proof can be utilized by other network nodes to validate the authorship.

Validators within the network execute their VRF for each slot and compare the result against a predefined threshold. If the result falls below this threshold, the validator earns the right to act as the author for that slot. It's important to note that there is a possibility for multiple validators to generate numbers below the threshold and thus share the role of primary authors for the same slot. Any conflicts in block authorship are resolved by the GRANDPA mechanism.

In cases where no validator manages to produce a number below the agreed-upon threshold, every slot is assigned a secondary author. These secondary authors are tasked with producing a block for any slot that remains without a primary leader. This redundancy ensures the continued operation and block production within the network.

![Untitled](Host%20Conformance%20Testing%20Suite%20using%20Zombienet%20Res%2005eb42de213a40c8916d3d25735b9a58/Untitled.png)

## Testing BABE with Zombienet/Polkadot.js

**Primary Author Assignment Verification**:

- **Objective**: Confirm that the primary author for a slot was correctly chosen.
- **Requirements**:
    - The result of the VRF must fall below the agreed-upon threshold for that slot.
    - Validators should be able to provide proof that the number was generated correctly.

**Block Authorship Validation**:

- **Objective**: Determine if a slot with a primary author results in a block created by a primary author.
- **Requirements**:
    - Slots may have one or multiple primary authors or none.
    - If a slot has a primary author assigned, the block should be created by a primary author, not a secondary one.

**Secondary Author Assignment Confirmation**:

- **Objective**: Verify that slots were correctly assigned secondary authors as backups.
- **Requirements**:
    - If no validator generates a pseudo-random number below the agreed threshold, a secondary author should create a block in that slot.
    - No slot should be left without an author.

### **Key Findings and Requirements**:

**Required Information**:

- The agreed threshold for each slot.
- The pseudo-random number generated by the VRF.
- The proof generated by the VRF.

**Available Information**:

- Currently, you can access the assigned slots (as primary or secondary author) for each node in the current epoch via the [babe_epochAuthorship](https://polkadot.js.org/docs/polkadot/rpc#epochauthorship-hashmapauthorityid-epochauthorship) rpc method.
- You can get a block author using [the derive api](https://polkadot.js.org/docs/api/start/api.rpc#detour-into-derives), but the returned value is an `AccountId32`, which couldn’t be linked to a specific node.
- You can obtain all authorities for the current epoch via [this API](https://polkadot.js.org/docs/polkadot/storage/#authorities-vecspconsensusbabeapppublicu64).

**Missing Information**:

- It appears that there is no way to identify in which slot a block has been created.
- Information about the threshold for a specific slot is not available.
- There is no straightforward method to access the VRF result and compare it with the threshold for each slot.
- The documentation may lack clarity regarding the return values and their significance.

**Limitations**:

- Due to the nature of author assignment and the randomness factor of the VRF, it is not possible to deterministically assert specific cases. For example in some runs we may have slots without primary authors and in other runs all slots may have primary authors, which may make it impossible to assert that a block was correctly created by a secondary author.

**Example tests:**

```
Some of the API functions and object properties in the examples below
do not exist and are purely exemplary. They need to be implemented
in order to make BABE tests with Zombienet/Polkadot.js possible
```

```jsx
Description: BABE Conformance
Network: ./0002-babe-conformance.toml
Creds: config

# Check authority status.
alice: reports node_roles is 4

# Check for matching input
alice: js-script ./assert-primary-author-assignment.js return is 0 within 60 secs
alice: js-script ./assert-block-authorship.js return is 0 within 60 secs
alice: js-script /assert-secondary-author-assignment.js return 0 within 60 secs
```

**Test 1: Primary Author Assignment Verification**

```jsx
async function run(nodeName, networkInfo) {
  const { wsUri, userDefinedTypes } = networkInfo.nodesByName[nodeName];
  const api = await zombie.connect(wsUri, userDefinedTypes);

  // Fetch all slot numbers in the current epoch
  // The result is an array of Slot objects containing
  // slot number, agreed theshold for the VRF result, assigned primaries, secondary, etc.
  // [Slot { number: 0, threshold: 23, primaries: [], ... }, Slot { number: 1, threshold: 65, primaries: [], ... }, ...]
  const slots = await api.query.babe.epochSlots();

  for (const slot of slots) {
    // iterate throug all slot primaries and assert if the VRF result is below the threshold
    for (const primary of slot.primaries) {
      const vrfResult = await api.query.babe.vrfResult(slot.number, primary);
      assert(vrfResult < slot.threshold, "VRF result is above the threshold");
    }
  }
}

```

**Test 2: Block Authorship Validation**

```jsx
async function run(nodeName, networkInfo) {
  const { wsUri, userDefinedTypes } = networkInfo.nodesByName[nodeName];
  const api = await zombie.connect(wsUri, userDefinedTypes);

  // Fetch latest block and the slot it was created in
  const block = await api.derive.chain.getHeader();
  const slot = await api.query.babe.slot(block.hash);

  // For example purposes assuming the slot has primary author
  assert(
    slot.primaries.includes(block.author),
    "Block is not created by the primary author"
  );
}

```

**Test 3: Secondary Author Assignment Confirmation**

```jsx
async function run(nodeName, networkInfo) {
  const { wsUri, userDefinedTypes } = networkInfo.nodesByName[nodeName];
  const api = await zombie.connect(wsUri, userDefinedTypes);

  // Fetch all slots in the current epoch
  // [Slot { number: 0, threshold: 23, secondaries: [], ... }, Slot { number: 1, threshold: 65, secondaries: [], ... }, ...]
  const slots = await api.query.babe.epochSlots();

  // Iterate through slots and assert if the secondary author is assigned
  for (const slot of slots) {
    assert(slot.secondaries.length > 0, "Secondary author is not assigned");
  }
}

```

### Testing BABE with logs

The team believes this to be the optimal approach for testing BABE. There are several reasons for that:

- **Logs are flexible**: They can be added almost everywhere and show information about everything that’s happening on the protocol’s side.
- **Logs have very low overhead**: Adding a couple or even a bunch of logs won’t affect performance of the Host. Moreover, they can be added in Rust behind a feature flag and won’t bloat the size of other Host builds.

The drawback to this approach is that logic related to VRF may not be testable. For example - **Primary Author Assignment Verification** may lead to flaky tests because the threshold is directly affected by the VRF result, in some test instances, the author will be a primary, in other cases, it will be a secondary. However, there are other properties that the research didn’t have manage to comprehensively assess, but our team believes they could be tested with logs as well:

1. Primary and secondary block production correlating to the difficulty setting 
2. The boundary of the number of babe fork generated under a certain threshold for an epoch

### Testing BABE with Conformance RPC methods

Testing BABE is possible by creating a set of RPC methods that expose the state of BABE for a specific or the current block, utilizing the existing `chain_getHeader` and `chain_getBlock` methods. In contrast to the previous sections, in this one the idea of using RPC methods is to help with the **assert** phase, instead of the **act** one. 

The team believes that the main con of this approach is the specification of the testing methods. Since BABE is a fast-moving protocol, with a lot of messages and state changes happening every second, creating the methods would prove to be difficult. A method would have to access internal BABE state that may be encapsulated in a good way, so that external modules don’t have access.

# **GRANDPA Research**

## Research Scope

GRANDPA is designed to provide finality to the blocks in a blockchain. Finality means that once a block has been confirmed by GRANDPA, it cannot be reverted or changed. This is important for the security and stability of a blockchain network. While BABE has probabilistic finality, adding GRANDPA as Polkadot’s finality gadget, makes the protocol have deterministic finality.  What’s great about BABE and GRANDPA being tested with Zombienet is that Zombienet can easily be configured to spawn malicious nodes. This enables testing byzantine behaviour of the protocol.

In order for a chain to be finalized, it must meet several requirements:

- It must be built on top an already finalized block
- It must be the longest chain
- It must have the most primaries

![Untitled](Host%20Conformance%20Testing%20Suite%20using%20Zombienet%20Res%2005eb42de213a40c8916d3d25735b9a58/Untitled%201.png)

Our research focused on finding ways on how GRANDPA’s finality requirements can be tested.

## Testing GRANDPA with Zombienet/Polkadot.js

From GRANDPA specification the following test cases can be devised:

**GRANDPA correct chain choice validation**

- **Objective**: Verify that GRANDPA chooses the correct chain based on it’s rules:
    - A chain can have multiple forks
    - A chain can be built on non-finalized block
    - A block can be authored by either primary or secondary slot leader
- **Requirements**:
    - Host must be able to choose the best chain

**GRANDPA signature verification**

- **Objective**: Prove that Host is able to properly verify GRANDPA signatures
- **Requirements**:
    - Host is able to detect invalid signatures
    - Host is able to validate correct signatures

**GRANDPA state verification**

- **Objective**: Verify that Host transitions GRANDPA state correctly
- **Requirements**:
    - Host correctly transitions when a scheduled authority set change is applied
    - Host correctly transitions when a forced authority set change is applied

**GRANDPA equivocation detection**

- **Objective**: Validate that Host catches equivocations and reports them
- **Requirements**:
    - Host is able to detect equivocations coming from other nodes
    - Host reports equivocations to the runtime

### Findings

At the time of writing, Polkadot.js provides us with the following Grandpa API:

- **currentSetId(): `SetId`** **[](https://polkadot.js.org/docs/polkadot/runtime#currentsetid-setid)**
    - Get current GRANDPA authority set id
- **generateKeyOwnershipProof(setId: `SetId`, authorityId: `AuthorityId`): `Option<OpaqueKeyOwnershipProof>`[](https://polkadot.js.org/docs/polkadot/runtime#generatekeyownershipproofsetid-setid-authorityid-authorityid-optionopaquekeyownershipproof)**
    - Generates a proof of key ownership for the given authority in the given set
- **grandpaAuthorities(): `AuthorityList`**
    - Get the current GRANDPA authorities and weights. This should not change except for when changes are scheduled and the corresponding delay has passed.
- **submitReportEquivocationUnsignedExtrinsic(equivocationProof: `GrandpaEquivocationProof`, keyOwnerProof: `OpaqueKeyOwnershipProof`): `Option<Null>`**
    - Submits an unsigned extrinsic to report an equivocation

**GRANDPA correct chain choice validation**

In order to track forks, a Host would have to track them in real time using logs. The reason is that Polkadot.js doesn’t provide users with the ability to track forks or fetch information about them. Currently Polkadot.js doesn’t provide us API that allow us to conduct correct chain choice validations, as well as determine which block is created by a primary or a secondary slot leader.

**GRANDPA signature verification**

Signature verification is currently infeasible without additional `conformance` logs from the Host. Polkadot.js doesn’t provide access to such data.

**GRANDPA state verification**

Again, Polkadot.js doesn’t provide API for accessing GRANDPA state. Assessing is only possible with additional `conformance` logs that are emitted whenever the state changes.

**GRANDPA equivocation detection**

Equivocation can be submitted using **submitReportEquivocationUnsignedExtrinsic** function. However, the goal of detecting equivocations can’t be approached without additional `conformance` logs.

### Testing GRANDPA with logs

Similarly to BABE, logs are a flexible way to track state changes with little overhead.  The drawback in the case with GRANDPA is that in some cases the logs may turn out too verbose and hence hard to assert, or just not sufficient to perform a proper test. For example - **GRANDPA correct chain choice validation** will require to log all forks in a way that we can assert that GRANDPA has made the correct choice and is following the requirements.

### Testing GRANDPA with Conformance RPC methods

Potential solution to testing GRANDPA is adding a subscription conformance RPC method that is triggered each time a chain is finalized (similarly to `subscribeNewHeads`), which returns the candidate and the finalized chains, which we can then compare.

The con of this approach, as stated previously in the document, is the slower implementation, potentially not all methods could be possible to implement or not all needed data may be accessible. However, 

# **Conclusion**

Polkadot stands as a pioneering blockchain protocol that has multiple client implementations - a rare distinction. Yet, this achievement brings with it a set of distinct challenges, particularly in the world of conformance testing. Our research lead the team to several challenges that prevent the realization of this endeavour.

These challenges can be found in every step of the AAA pattern - the standard for conducting tests:

- **Arrange**: Hosts lack a reliable tool or approach to establish their initial state for testing.
- **Act**: Hosts lack a reliable way to trigger specific internal functions.
- **Assert**: While Zombienet possesses strong foundational assertion capabilities, there is room for enhanced flexibility to enable complex assertions.

While the idea of using Zombienet as the framework for conducting the conformance suite is not immediately feasible, the team believes that it’s still the right way forward. We stand behind the idea of the community of Polkadot, ourselves included, to work towards this achievement.

## Suggested Approaches

### Trie Host API, State Trie

The team suggest using the **Conformance RPC methods** solution for these Host modules. Although requiring more development, we believe that it provides great flexibility and level of abstraction at which the modules should be tested. Additionally, it solves the issue with invoking the functions. If more of a integration type tests are wanted, we believe that logs approach can be introduced so that it’s possible to assert on what’s happening inside the Host.

### BABE & GRANDPA

The team believes that BABE and GRANDPA protocols should be tested **primarily with logs**. The reasons is that they would be very hard to test in isolation, as is the case with the aforementioned Host modules. Their logic is moving very fast and we believe that, based on our research, logging the necessary info and asserting on that inside Zombienet to be the optimal solution.

### SCALE

While our research confirmed the viability of testing the SCALE codec through RPC methods and logging approaches (former method is preferred), the team is concerned that Zombienet might not offer the **right level of abstraction** for this module. Zombienet enables users to use black box testing, allowing them to test without knowledge of each Host's internals. While this is beneficial for host-agnostic testing, the team believes that, for SCALE codec testing, a different approach would be more practical. 

The proposed method involves generating input and output data for each testing scenario. The data is then provided to Host implementers for them to integrate into their respective implementation. This approach has several differences from the Zombienet one. The main one being that it inverses the control of tests - rather than the framework orchestrating and executing tests, the development team integrates them of each team to ensure codec compliance.

# **Recommendations**

Continuing from the conclusion that there are several challenges that need to be solved before a conformance testing suite using Zombienet can be achieved, the team proposes several action points that will pave the path forward towards achieving the goal:

## Solve Host State Transition (Arrange)

A challenge that came up in every research topic’s testing scenarios was: “How will the Host arrive at the desired initial testing state?”. The team considers this a critical missing piece of the conformance suite, regardless of the testing approach. The most naive solution would be to manually execute transactions until the desired state is reached. Afterwards, the database snapshot can be exported and imported in tests. Unfortunately, this approach is very time consuming, won’t work for complex scenarios that require careful crafting, and as mentioned earlier - it doesn’t work when Hosts use different DBs.

Our team recommends working towards enabling Hosts to import a specific pre-state, as well as export it. This is going to benefit the conformance suite in several ways:

- **Import state:** Hosts will be able to accept a trie state in a pre-specified format using a CLI parameter.
- **Export state**: Hosts will be able to export the trie state, making it accessible to external services and frameworks.
- **Host-agnostic**: The logic is going to use **trie state**, in contrast to the currently available ****approach with **database snapshots** which don’t work if databases aren’t the same across Hosts.

The design of the logic and changes to Hosts are out of scope for this research. However, the team believes that it’s feasible and expresses interest in building it. It will not only be useful for conformance testing, but for testing and replicating complex Host scenarios in general. The idea might sound similar to what Chopsticks has already done, however there are a few key differences:

- Chopsticks works with Parachains, whereas our team suggests a solution for Hosts.
- Chopsticks is a tool that allows developers to fork Substrate based networks and allows for the replaying of blocks to easily examine how extrinsics effect state. We propose a feature that enables Hosts to transitions to a **specific state** that can potentially be forked later using Chopstics.

## Conformance RPC methods (Act)

Another challenge that came up often was related to invoking specific internal Host functions. Being internal logic isn’t supposed to be accessed from the outside, that’s why we recommend creating a new group of RPC methods called `conformance`. These methods will provide the said access and will be only available when Host is launched with `conformance` flag. Technically, any function can be exposed using this approach, however it’s particularly useful when testing `pure` functions like the ones in Host API, State encoding and decoding and others.

This approach combined with the solution for transitioning state will syngergize well and make the foundational blocks for Host conformance testsing.

## Zombienet Improvements (Assert)

### DSL: More options for assertion logs

The team believes that Zombienet can be improved in terms of assertion logical operators. Something that pushed us to using workarounds was the lack of `after` operator. We envision this operator to function similarly to `within` but checks the validity of a statement **after** the time runs out.

This operator will be very useful in negative test cases where someone wants to assert that an event **hasn’t** happened. Right now, if you want to test that block production isn’t working and that’s the intended behaviour - the intuitive asserting would be (since there’s no operator that fits the case better):

```json
alice: block height equals 0 within 60 seconds
```

However, since every chain starts from block 0, therefore this statement is immediately true and the assert completes.

Adding `after` operator resolves this issue. Additionally, it will make the tests more readable since, the workaround our team implemented during the PVF conformance testing grant was to check after another event has happened:

```json
# Ensure parachain did not make progress.
alice: parachain 2000 block height = 0 within 10 seconds

# Check log for trap message
alice: log line matches glob "*wasm trap: out of bounds memory access*" within 120 seconds

# Ensure parachain did not make progress.
alice: parachain 2000 block height = 0 within 10 seconds
```

### Enable asserting logs inside TS/JS script

An alternative approach to the problem above is to pivot to using custom scripts for testing exclusively. However, this solution is not ideal because currently `log assertions` aren’t available inside the scripts, hindering the ability to make assertions. That’s the reason why we recommend improving Zombienet to allow for `log assertions` inside TS/JS scripts. This approach will give users greater flexibility and the ability to use a widely known language to write test cases. Most importantly, this will open up a lot of possibilities to what’s possible when users are able to combine Polkadot.js information and logs coming from the Host. Additionally, we believe that if not only log assertions, but other features of Zombienet’s DSL are integrated inside the custom scripts, users will have even greater freedom and ease of using Zombienet. 

Looking through the [LogMatch](https://github.com/paritytech/zombienet/blob/main/javascript/packages/orchestrator/src/test-runner/assertions.ts#L159-L173), [getNodes](https://github.com/paritytech/zombienet/blob/main/javascript/packages/orchestrator/src/network.ts#L253-L264) and [findPattern](https://github.com/paritytech/zombienet/blob/main/javascript/packages/orchestrator/src/networkNode.ts#L487-L551) functions, used for asserting the logs in Zombienet - they all boil down to getting the `tempDir` and `node name`, fetching the logs afterwards and pattern matching the lines for the specified pattern. In the script we have the node name via the `nodeName` param and the `tempDir` via the `networkInfo` parameter.

```tsx
async function run(nodeName, networkInfo) {
	const tempDir = networkInfo.tempDir;
}
```

It seems that it should be possible to expose log assertion inside the JS/TS script through the `zombie` helper or as a function of the `node` object:

```tsx

async function run(nodeName, networkInfo) {
	const tempDir = networkInfo.tempDir;
	const pattern = "*blake2_256_input:*";
	const result = zombie.matchLogs(node, tmpDir, pattern);
	
	assert(result === true, `Could not find pattern ${pattern} in logs`
}
```

or

```tsx
async function run(nodeName, networkInfo) {
	const node = networkInfo.nodesByName[nodeName];
	const pattern = "*blake2_256_input:*";
	const result = node.matchLogs(pattern);

	assert(result === true, `Could not find pattern ${pattern} in logs`
}
```

The same should be possible for the Prometheus metrics, because `prometheusUri` is also accessible through the `networkInfo` parameter for each node. Moreover, it should be possible to implement the metrics matchers through the zombie helper or directly through the node object:

```tsx
async function run(nodeName, networkInfo) {
	const { prometheusUri } = networkInfo.nodesByName[nodeName];
	const result = zombie.getMetric(nodeName, "node_roles");
	
	assert(result == 4, "node_role should be 4")
}
```

or

```tsx

async function run(nodeName, networkInfo) {
	const node = networkInfo.nodesByName[nodeName];
	const result = node.getMetric(nodeName, "node_roles");
	
	assert(result == 4, "node_role should be 4")
}

```

Having access to these matchers in the scripts will make it possible to create more complex test cases, in which users interact with the nodes through the exposed APIs and expect the changes to be properly logged.