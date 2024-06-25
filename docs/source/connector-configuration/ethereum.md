!!! note

    *This adapter relies on web3js 1.2.x that is the stable version coming from 1.0.0-beta.37*

This page introduces the Ethereum adapter suitable for all the Ethereum clients that expose the web3 RPC interface over HTTP.

!!! note

    *Hyperledger Besu and Geth are the current tested clients. The tests are driven via standard Ethereum JSON-RPC APIs so other clients should be compatible once docker configurations exist.*

!!! note

    *Hyperledger Besu does not provide wallet services so the `contractDeployerPassword` and `fromAddressPassword` options are not supported and the private key variants must be used.*

!!! note

    *Some highlights of the provided features:*

    - configurable confirmation blocks threshold

The page covers the following aspects of using the Ethereum adapter:

- how to assemble a connection profile file, a.k.a., the blockchain network configuration file;
- how to use the adapter interface from the user callback module;
- transaction data gathered by the adapter;
- and a complete example of a connection profile.

## Assembling the Network Configuration File

The JSON network configuration file of the adapter essentially defines which contract are expected to be on the network and which account the adapter should use to deploy the pointed contracts and which account use to invoke them. Contract necessary for the benchmark are automatically deployed at benchmark startup while the special registry contract needs to be already deployed on the network. ABI and bytecode of the registry contract are reported in the src/contract/ethereum/registry/registry.json.

### Connection profile example
We will provide an example of the configuration and then we’ll in deep key by key

```sh
{
    "caliper": {
        "blockchain": "ethereum",
        "command" : {
            "start": "docker-compose -f network/ethereum/1node-clique/docker-compose.yml up -d && sleep 3",
            "end" : "docker-compose -f network/ethereum/1node-clique/docker-compose.yml down"
          }
    },
    "ethereum": {
        "url": "http://localhost:8545",
        "contractDeployerAddress": "0xc0A8e4D217eB85b812aeb1226fAb6F588943C2C2",
        "contractDeployerAddressPassword": "password",
        "fromAddress": "0xc0A8e4D217eB85b812aeb1226fAb6F588943C2C2",
        "fromAddressPassword": "password",
        "transactionConfirmationBlocks": 12,
        "contracts": {
            "simple": {
                "path": "src/contract/ethereum/simple/simple.json"
            }
        }
    }
}
```

The top-level `caliper` attribute specifies the type of the blockchain platform, so Caliper can instantiate the appropriate adapter when it starts. To use this adapter, specify the `ethereum` value for the `blockchain` attribute.

Furthermore, it also contains two optional commands: a `start` command to execute once before the tests and an `end` command to execute once after the tests. Using these commands is an easy way, for example, to automatically start and stop a test network. When connecting to an already deployed network, you can omit these commands.

These are the keys to provide inside the configuration file under the `ethereum` one:

- URL of the node to connect to. Only http is currently supported.
- Deployer address with which to deploy required contracts.
- Deployer address private key: the private key of the deployer address.
- Deployer address password: to unlock the deployer address.
- Address from which to invoke methods of the benchmark.
- Private Key: the private key of the benchmark address.
- Password: to unlock the benchmark address.
- Number of confirmation blocks to wait to consider a transaction as successfully accepted in the chain.
- Contracts configuration.

The following sections detail each part separately. For a complete example, please refer to the example section or one of the example files in the `network/ethereum` directories

#### URL
The URL of the node to connect to. Any host and port can be used if it is reachable. Currently only HTTP is supported.

```sh
"url": "http://localhost:8545"
```

#### Deployer address

The address to use to deploy contracts of the network. Without particular or specific needs it can be set to be equal to the benchmark address. Its private key must be hold by the node connected with URL and it must be provided in the checksum form (the one with both lowercase and uppercase letters).

```sh
"contractDeployerAddress": "0xc0A8e4D217eB85b812aeb1226fAb6F588943C2C2"
```

#### Deployer address private key

The private key for the deployer address. If present then transactions are signed inside caliper and sent “raw” to the ethereum node.

```sh
"contractDeployerAddressPrivateKey": "0x45a915e4d060149eb4365960e6a7a45f334393093061116b197e3240065ff2d8"
```

#### Deployer address password

The password to use to unlock deployer address. If there isn’t an unlock password, this key must be present as empty string. If the deployer address private key is present this is not used.

```sh
"contractDeployerAddressPassword": "gottacatchemall"
```

#### Benchmark address

The address to use while invoking all the methods of the benchmark. Its private key must be hold by the node connected with URL and it must be provided in the checksum form (the one with both lowercase and uppercase letters).

```sh
"fromAddress": "0xc0A8e4D217eB85b812aeb1226fAb6F588943C2C2"
```

#### Benchmark address private key

The private key for the benchmark address. If present then transactions are signed inside caliper and sent “raw” to the ethereum node.

```sh
"fromAddressPassword": "0x45a915e4d060149eb4365960e6a7a45f334393093061116b197e3240065ff2d8"
```

#### Benchmark address password

The password to use to unlock benchmark address. If there isn’t an unlock password, this key must be present as empty string. If the benchmark address private key is present this is not used.

```sh
"fromAddressPassword": "gottacatchemall"
```

#### Confirmation blocks

It is the number of blocks the adapter will wait before warn Caliper that a transaction has been successfully executed on the network. You can freely tune it from 1 to the desired confirmations. Keep in mind that in the Ethereum main net (PoW), 12 to 20 confirmations can be required to consider a transaction as accepted in the blockchain. If you’re using different consensus algorith (like clique in the example network provided) it can be safely brought to a lower value. In any case it is up to you.

```sh
"transactionConfirmationBlocks": 12
```

#### Contract configuration

It is the list, provided as a json object, of contracts to deploy on the network before running the benchmark. You should provide a json entry for each contract; the key will represent the contract identifier to invoke methods on that contract. For each key you must provide a JSON object with the `path` field pointing to the contract definition file.

```sh
"contracts": {
    "simple": {
        "path": "src/contract/ethereum/simple/simple.json"
    },
    "second": {
        "path": "src/contract/ethereum/second/second.json"
    }
}
```

#### Contract definition file

Contract definition file is a simple JSON file containing basic information to deploy and use an Ethereum contract. Four keys are required:

- Name
- ABI
- Bytecode
- Gas

Here is an example:
```sh
{
    "name": "The simplest workload contract",
    "abi": [{"constant":true,"inputs":[{"nam......ype":"function"}],
    "bytecode": "0x608060405.........b0029",
    "gas": 259823
}
```
**Name**
It is a name to display in logs when the contract gets deployed. It is only a description name.

**ABI**
It is the ABI generated when compiling the contract. It is required in order to invoke methods on a contract.

**Bytecode**
It is the bytecode generated when compiling the contract. Note that since it is an hexadecimal it must start with the `0x`.

**Gas**
It is the gas required to deploy the contract. It can be easily calculated with widely used solidity development kits or querying to a running Ethereum node.

### Using the Adapter Interface

The user callback modules interact with the adapter at two phases of the tests: during the initialization of the user module (the `init` callback), and when submitting invoke or query transactions (the `run` callback).

#### The *init* Callback
The first argument of the `init` callback is a `blockchain` object. We will discuss it in the next section for the `run` callback.

The second argument of the `init` callback is a `context`, which is a platform-specific object provided by the backend blockchain’s adapter. The context object provided by this adapter is the following:

```
{
  fromAddress: "0xA89....7G"
  web3: Web3
}
```

The `fromAddress` property is the benchmark address while web3 is the configured instance of the Web3js client.

#### The *run* Callback

The `blockchain` object received (and saved) in the `init` callback is of type `Blockchain` you can find in `packages/caliper-core/lib/blockchain.js`, and it wraps the adapter object (if you know what you are doing, it can be reached at the `blockchain.bcObj` property). The `blockchain.bcType` property has the `ethereum` string value.

##### Invoking a contract method

To submit a transaction, call the `blockchain.invokeSmartContract` function. It takes five parameters: the previously saved `context` object, the `contractID` of the contract (that is the key specified here), an unused contract version (you can put whatever), the `invokeData` array with methods data and a `timeout` value in seconds that is currently unimplemented (the default web3js timeout is used).

The `invokeData` parameter can be a single or array of JSON, that must contain:

- `verb`: *string. Required.* The name of the function to call on the contract.
- `args`: *mixed[]. Optional.* The list of arguments to pass to the method in the correct order as they appear in method signature. It must be an array.

```sh
let invokeData = [{
    verb: 'open',
    args: ['sfogliatella', 1000]
},{
    verb: 'open',
    args: ['baba', 900]
}];

return blockchain.invokeSmartContract(context, 'simple', '', invokeData, 60);
```
Currently each method call inside `invokeData` is sent separately, that is, they are NOT sent as a batch of calls on RPC.

##### Querying a contract

To query a state on a contract state, call the `blockchain.querySmartContract` function that has exactly same arguments as the `blockchain.invokeSmartContract` function. The difference is that it can’t produce any change on the blockchain and node will answer with its local view of data. For backward compatibility reasons also the `blockchain.queryState` is present. It takes five parameters: the previously saved `context` object, the `contractID` of the contract, an unused contract version, the key to request information on and fcn to point which function call on the contract. fcn function on the contract must accept exactly one parameter; the `key` string will be passed as that parameter. `fcn` can also be omitted and the adapter will search on the contract for a function called `query`.

!!! note

    *Querying a value in a contract is always counted in the workload. So keep it in mind that if you think to query a value when executing a workload.*

Querying a chaincode looks like the following:

```sh
let queryData = [{
    verb: 'queryPastryBalance',
    args: ['sfogliatella']
},{
    verb: 'queryPastryBalance',
    args: ['baba']
}];

return blockchain.querySmartContract(context, 'pastryBalance', '', queryData, 60);
```

or

```sh
let balance = blockchain.queryState(context, 'simple', '', 'sfogliatella');
```

assuming that `simple` contract has the `query` function that takes only one argument.

Like invoke, currently there is not any support for batch calls.

## Transaction Data Gathered by the Adapter

The previously discussed `invokeSmartContract` and `queryState` functions return an array whose elements correspond to the result of the submitted request(s) with the type of TxStatus. The class provides some standard and platform-specific information about its corresponding transaction.

The standard information provided by the type are the following:

- `GetID():string` returns the transaction ID for invokeSmartContract, null for queryState and querySmartContract.
- `GetStatus():string` returns the final status of the transaction, either success or failed.
- `GetTimeCreate():number` returns the epoch when the transaction was submitted.
- `GetTimeFinal():number` return the epoch when the transaction was finished.
- `IsVerified():boolean` indicates whether we are sure about the final status of the transaction. Always true for successful transactions. False in all other cases.