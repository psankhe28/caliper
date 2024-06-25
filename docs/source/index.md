# Caliper

Caliper is a blockchain performance benchmark framework, which allows users to test different blockchain solutions with predefined use cases, and get a set of performance test results.

## Supported Blockchain Solutions

Currently supported blockchain solutions:

- [Hyperledger Burrow](https://github.com/hyperledger/burrow)
- [Hyperledger Composer](https://github.com/hyperledger/composer)
- [Ethereum](https://github.com/ethereum/)
- [Hyperledger Fabric](https://github.com/hyperledger/fabric)
- [FISCO BCOS](https://github.com/FISCO-BCOS/FISCO-BCOS)
- [Hyperledger Iroha](https://github.com/hyperledger/iroha)
- [Hyperledger Sawtooth](https://github.com/hyperledger/sawtooth-core)

Steps for configuring a benchmark that targets a supported blockchain technology are given in the following pages:

- [Burrow](https://hyperledger.github.io/caliper/v0.2/burrow-config/)
- [Composer](https://hyperledger.github.io/caliper/v0.2/composer-config/)
- [Ethereum](https://hyperledger.github.io/caliper/v0.2/ethereum-config/)
- [Fabric](https://hyperledger.github.io/caliper/v0.2/fabric-config/)
- [FISCO BCOS](https://hyperledger.github.io/caliper/v0.2/fisco-config/)
- [Iroha](https://hyperledger.github.io/caliper/v0.2/iroha-config/)
- [Sawtooth](https://hyperledger.github.io/caliper/v0.2/sawtooth-config/)

## Supported Performance Indicators

Currently supported performance indicators:

- Success rate
- Transaction/Read throughput
- Transaction/Read latency (minimum, maximum, average, percentile)
- Resource consumption (CPU, Memory, Network IO, …)

See [PSWG](https://wiki.hyperledger.org/groups/pswg/performance-and-scale-wg) to find out the definitions and corresponding measurement methods.

## Architecture

See the [Architecture Introduction](overview/architecture.md) page.

## Installing Caliper

See the [Installing and Running Caliper](overview/installing-and-running-caliper.md) page.

## Caliper Flow Control

The default Caliper lifecycle is:

1. Start SUT (using a `start` command in the network configuration file)
2. Initialize the SUT
3. Install smart contracts
4. Perform the benchmark test
5. Tear down SUT (using an `end` command in the network configuration file)

In some cases, it is desirable to run a subset of the complete Caliper lifecycle. It is possible to do this by using command line flags:

To skip a phase

- `caliper-flow-skip-start` - skip the start command within the network configuration file
- `caliper-flow-skip-init` - skip the blockchain `init()` function that is used to configure an existing blockchain network
- `caliper-flow-skip-install` - skip the installation of any smart contracts listed in the network configuration file
- `caliper-flow-skip-test` - skip the benchmark test
- `caliper-flow-skip-end` - skip the end command within the network configuration file

One or more `skip` flags may be specified at a time, but may not be used in conjunction with `only` flags.

To only run a single phase

- `caliper-flow-only-start` - only run the start command within the network configuration file
- `caliper-flow-only-init` - only run the blockchain `init()` function that is used to configure an existing blockchain network
- `caliper-flow-only-install` - only install the smart contracts listed in the network configuration file
- `caliper-flow-only-test` - only run the benchmark test
- `caliper-flow-only-end` - only run the end command within the network configuration file

Only one `only` flag is permitted to be supplied at a time, and may not be used in conjunction with a `skip` flag. For instance, given that a blockchain network has been created, configured, and has smart contracts installed, to run the benchmark test, the following command would be used:

```sh
caliper benchmark run --caliper-workspace ./packages/caliper-samples --caliper-benchconfig benchmark/simple/config.yaml --caliper-networkconfig network/fabric-v1.4/2org1peercouchdb/fabric-node.yaml --caliper-flow-only-test
```

## Sample Networks
See the Sample Networks page.

## How to Contribute
See Contributing

## License
The Caliper codebase is released under the Apache 2.0 license. Any documentation developed by the Caliper Project is licensed under the Creative Commons Attribution 4.0 International License. You may obtain a copy of the license, titled CC-BY-4.0, at [http://creativecommons.org/licenses/by/4.0/](http://creativecommons.org/licenses/by/4.0/).