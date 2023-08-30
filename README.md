# Polkadot Parachain PVF Conformance Testing Suite 🛡️

# Introduction
The Polkadot Parachain PVF Conformance Testing Suite is a comprehensive toolset designed to test the conformance of Polkadot parachains with the Polkadot Validating Function (PVF) specification. It provides a collection of test cases and utilities to verify the correctness and compatibility of parachains in the Polkadot ecosystem.

This testing suite aims to ensure that parachains adhere to the PVF specification, enabling developers to build and deploy secure and interoperable parachains on the Polkadot network. By running the provided test cases, developers can identify any issues or deviations from the PVF standard and make necessary adjustments to ensure compatibility.

# GitHub Pipeline
A GitHub pipeline is set up to run automatically once a day or whenever changes are made to the `tests` folder. This ensures continuous monitoring and validation of the testing suite, further enhancing the reliability and robustness of the Polkadot Parachain PVF Conformance Testing process.

# Testing suite
- code size of the PVF is too large. Different hosts have different hard limit, so we are testing with more than the highest permited PVF size for all hosts.
- execution time of the PVF is more than 2 seconds. Our tests are executed using a modified WASM runtime that has injected infinite loop in the beginning of the `validate_block` function
- allocating a large amount of heap memory using malloc to get heap overflow.
- allocating a large amount of memory on the stack to cause stack overflow


# Runtimes and host implementations
We are running the tests on the following host implementations:
- [Kagome](https://github.com/soramitsu/kagome)
- [Polkadot](https://github.com/paritytech/polkadot)
For each host implementation we are testing using two different runtimes:
- [Cumulus](https://github.com/paritytech/cumulus)
- [Moonbeam](https://github.com/moonbeam-foundation/moonbeam)


# Usage
To run the testing suite locally you need to fetch the following binaries on your machine:
- `kagome`: you can fetch the latest binary through their docker container using the following command:
```sh
docker run -v <desired_destination_on_local_machine>:/kagome --rm soramitsu/kagome:latest sh -c  "cp /usr/local/bin/kagome . && chmod +x kagome"
```
- [polkadot](https://github.com/paritytech/polkadot/releases)
- [polkadot-parachain](https://github.com/paritytech/cumulus/releases)
- [zombienet](https://github.com/paritytech/zombienet/releases)
- [wasm-injector](https://github.com/LimeChain/wasm-injector)
- `moonbeam`: you can fetch the latest binary through their docker container using the following command:
```sh
 docker run --user root -v <desired_destination_on_local_machine>:/hostdir --rm --entrypoint sh moonbeamfoundation/moonbeam:<desired_version> -c "cp /moonbeam/moonbeam /hostdir/moonbeam && chmod +x /hostdir/moonbeam"
```
You can export the runtime WASM from polkadot-parachain using: `polkadot-parachain export-genesis-wasm > fetched.wasm`
Or from moonbeam using: `moonbeam export-genesis-wasm --chain moonbase-local > fetched.wasm`
To generate the injected WASM modules you need to run the following commands:
```sh
wasm-injector convert fetched.wasm ./wasm/orig.wasm.hex --hexified
wasm-injector inject noops --size 50 validate_block fetched.wasm ./wasm/code-size.wasm.hex --hexified
wasm-injector inject stack-overflow validate_block fetched.wasm ./wasm/stack-overflow.wasm.hex --hexified
wasm-injector inject heap-overflow validate_block fetched.wasm ./wasm/heap-overflow.wasm.hex --hexified
wasm-injector inject infinite-loop validate_block fetched.wasm ./wasm/infinite-loop.wasm.hex --hexified
```
You can run the tests using the following command. The options for runtimes are `cumulus`, `moonbeam`, `acala` and for hosts are `kagome`, `polkadot`:
```sh
git clone https://github.com/LimeChain/polkadot-conformance
cd polkadot-conformance
julia scripts/runTests.jl [runtime] [host]
```

For example:
```sh
# Execute all tests for all runtimes and hosts
julia scripts/runTests.jl

# Execute only the tests for a specific runtime
julia scripts/runTests.jl cumulus

# Execute only the tests for a specific runtime and host
julia scripts/runTests.jl acala kagome
```


# Contributing
Contributions to the Polkadot Parachain PVF Conformance Testing Suite are welcome! If you want to contribute, please follow these steps:

1. Fork the repository.
2. Create a new branch: git checkout -b feature/my-feature.
3. Make your changes and commit them: git commit -m 'Add new feature'.
4. Push to the branch: git push origin feature/my-feature.
5. Submit a pull request describing your changes.

Please ensure that your contributions adhere to the existing coding style.
