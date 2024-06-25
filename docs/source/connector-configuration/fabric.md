## Overview

This page introduces the Fabric adapter that utilizes the Common Connection Profile (CCP) feature of the Fabric SDK to provide compatibility and a unified programming model across different Fabric versions.

!!! note

    *The latest supported version of Hyperledger Fabric is v1.4.X*

The adapter exposes many SDK features directly to the user callback modules, making it possible to implement complex scenarios.

!!! note

    *Some highlights of the provided features:*

    - supporting multiple orderers
    - automatic load balancing between peers and orderers
    - supporting multiple channels and chaincodes
    - metadata and private collection support for chaincodes
    - support for TLS and mutual TLS communication
    - dynamic registration of users with custom affiliations and attributes
    - option to select the identity for submitting a TX/query
    - transparent support for every version of Fabric since v1.0 (currently up to v1.4.X)
    - detailed execution data for every TX

## Installing dependencies
To install the Fabric SDK dependencies of the adapter, configure the binding command as follows:

- Set the `caliper-bind-sut` setting key to `fabric`
- Set the `caliper-bind-sdk` setting key to a supported SDK binding.

You can set the above keys either from the command line:

```sh
user@ubuntu:~/caliper-benchmarks$ npx caliper bind --caliper-bind-sut fabric --caliper-bind-sdk 1.4.0
```
or from environment variables:
```sh
user@ubuntu:~/caliper-benchmarks$ export CALIPER_BIND_SUT=fabric
user@ubuntu:~/caliper-benchmarks$ export CALIPER_BIND_SDK=1.4.0
user@ubuntu:~/caliper-benchmarks$ npx caliper bind
```
or from various other sources.

## Runtime settings

### Common settings
Some runtime properties of the adapter can be set through Caliper’s runtime configuration mechanism. For the available settings, see the `caliper.fabric` section of the default configuration file and its embedded documentation.

The above settings are processed when starting Caliper. Modifying them during testing will have no effect. However, you can override the default values *before Caliper* starts from the usual configuration sources.

!!!note

    *An object hierarchy in a configuration file generates a setting entry for every leaf property. Consider the following configuration file:*
    ```sh
    caliper:
        fabric:
            usegateway: true
            discovery: true
    ```
    *After naming the project settings file `caliper.yaml` and placing it in the root of your workspace directory, it will override the following two setting keys with the following values:*

    - *Setting `caliper-fabric-usegateway` is set to true*
    - *Setting `caliper-fabric-discovery` is set to true*

    **The other settings remain unchanged.**

### Skip channel creation

Additionally, the adapter provides a dynamic setting (family) for skipping the creation of a channel. The setting key format is `caliper-fabric-skipcreatechannel-<channel_name>`. Substitute the name of the channel you want to skip creating into `<channel_name>`, for example:

```sh
user@ubuntu:~/caliper-benchmarks$ export CALIPER_FABRIC_SKIPCREATECHANNEL_MYCHANNEL=true
```

!!!note

    This settings is intended for easily skipping the creation of a channel that is specified in the network configuration file as “not created”. However, if you know that the channel always will be created during benchmarking, then it is recommended to denote this explicitly in the network configuration file.

Naturally, you can specify the above setting multiple ways (e.g., command line argument, configuration file entry).

## The adapter API
The user callback modules interact with the adapter at two phases of the tests: during the initialization of the user module (in the `init` callback), and when submitting invoke or query transactions (in the `run` callback).

### The `init` callback

The first argument of the `init` callback is a `blockchain` object. We will discuss it in the next section for the `run` callback.

The second argument of the `init` callback is a `context`, which is a platform-specific object provided by the backend blockchain’s adapter. The context object provided by this adapter exposes the following properties:
- The `context.networkInfo` property is a `FabricNetwork` instance that provides simple string-based “queries” and results about the network topology, so user callbacks can be implemented in a more general way. For the current details/documentation of the API, refer to the [source code](https://github.com/hyperledger/caliper/blob/v0.2.0/packages/caliper-fabric/lib/fabricNetwork.js).

### The run callback
The `blockchain` object received (and saved) in the `init` callback is of type [Blockchain](https://github.com/hyperledger/caliper/blob/v0.2.0/packages/caliper-core/lib/blockchain.js) and it wraps the adapter instance of type Fabric. The `blockchain.bcType` property has the `fabric` string value.

The two main functions of the adapter are `invokeSmartContract` and `querySmartContract`, sharing a similar API.

#### Invoking a chaincode
To submit a transaction, call the `blockchain.invokeSmartContract` function. It takes five parameters:
1. `context`: the same `context` object saved in the `init` callback.
2. `contractID`: the unique (Caliper-level) ID of the chaincode.
3. `version`: the chaincode version. **Unused.**
4. `invokeSettings`: an object containing different TX-related settings
5. `timeout`: the timeout value for the transaction **in seconds.**

The `invokeSettings` object has the following structure:
- `chaincodeFunction`: *string. Required*. The name of the function to call in the chaincode.
- `chaincodeArguments`: *string[]. Optional.* The list of **string** arguments to pass to the chaincode.
- `transientMap`: *Map<string, byte[]>. Optional.* The transient map to pass to the chaincode.
- `invokerIdentity`: *string. Optional.* The name of the user who should invoke the chaincode. If an admin is needed, use the organization name prefixed with a `#` symbol (e.g., `#Org2`). Defaults to the first client in the network configuration file.
- `targetPeers`: *string[]. Optional.* An array of endorsing peer names as the targets of the transaction proposal. If omitted, the target list will include endorsing peers selected according to the specified load balancing method.
- `orderer:` *string. Optional.* The name of the target orderer for the transaction broadcast. If omitted, then an orderer node of the channel will be used, according to the specified load balancing method.

So invoking a chaincode looks like the following (with automatic load balancing between endorsing peers and orderers):

```sh
let settings = {
    chaincodeFunction: 'initMarble',
    chaincodeArguments: ['MARBLE#1', 'Red', '100', 'Attila'],
    invokerIdentity: 'client0.org2.example.com'
};

return blockchain.invokeSmartContract(context, 'marbles', '', settings, 10);
```
`invokeSmartContract` also accepts an array of `invokeSettings` as the second arguments. However, Fabric does not support submitting an atomic batch of transactions like Sawtooth, so there is no guarantee that the order of these transactions will remain the same, or whether they will reside in the same block.

Using “batches” also increases the expected workload of the system, since the rate controller mechanism of Caliper cannot account for these “extra” transactions. However, the resulting report will accurately reflect the additional load.

#### Querying a chaincode
To query the world state, call the `blockchain.bcObj.querySmartContract` function. It takes five parameters:
1. `context`: the same `context` object saved in the `init` callback.
2. `contractID`: the unique (Caliper-level) ID of the chaincode.
3. `version`: the chaincode version. *Unused.*
4. `querySettings`: an object containing different query-related settings
5. `timeout`: the timeout value for the transaction *in seconds.*

The `querySettings` object has the following structure:
- `chaincodeFunction`: *string. Required.* The name of the function to call in the chaincode.
- `chaincodeArguments`: *string[]. Optional.* The list of string arguments passed to the chaincode.
- `transientMap`: *Map<string, byte[]>. Optional.* The transient map passed to the chaincode.
- `invokerIdentity`: *string. Optional.* The name of the user who should invoke the chaincode. If an admin is needed, use the organization name prefixed with a `#` symbol (e.g., `#Org2`). Defaults to the first client in the network configuration file.
- `targetPeers`: *string[]. Optional.* An array of endorsing peer names as the targets of the query. If omitted, the target list will include endorsing peers selected according to the specified load balancing method.
- `countAsLoad:` *boolean. Optional.* Indicates whether the query should be counted as workload and reflected in the generated report. If specified, overrides the adapter-level `caliper-fabric-countqueryasload` setting.

!!! note

    *Not counting a query in the workload is useful when occasionally retrieving information from the ledger to use as a parameter in a transaction (might skew the latency results). However, count the queries into the workload if the test round specifically targets the query execution capabilities of the chaincode.*

So querying a chaincode looks like the following (with automatic load balancing between endorsing peers):

```sh
let settings = {
    chaincodeFunction: 'readMarble',
    chaincodeArguments: ['MARBLE#1'],
    invokerIdentity: 'client0.org2.example.com'
};

return blockchain.querySmartContract(context, 'marbles', '', settings, 3);
```
`querySmartContract` also accepts an array of `querySettings` as the second arguments. However, Fabric does not support submitting an atomic batch of queries, so there is no guarantee that their execution order will remain the same (although it’s highly probably, since queries have shorter life-cycles than transactions).

Using “batches” also increases the expected workload of the system, since the rate controller mechanism of Caliper cannot account for these extra queries. However, the resulting report will accurately reflect the additional load.

## Gathered TX data
The previously discussed `invokeSmartContract` and `querySmartContract` functions return an array whose elements correspond to the result of the submitted request(s) with the type of [TxStatus](https://github.com/hyperledger/caliper/blob/v0.2.0/packages/caliper-core/lib/transaction-status.js). The class provides some standard and platform-specific information about its corresponding transaction.

The standard data provided are the following:
- `GetID():string` returns the transaction ID.
- `GetStatus():string` returns the final status of the transaction, either `success` or `failed`.
- `GetTimeCreate():number` returns the epoch when the transaction was submitted.
- `GetTimeFinal():number` return the epoch when the transaction was finished.
- `IsVerified():boolean` indicates whether we are sure about the final status of the transaction. Unverified (considered failed) transactions could occur, for example, if the adapter loses the connection with every Fabric event hub, missing the final status of the transaction.
- `GetResult():Buffer` returns one of the endorsement results returned by the chaincode as a `Buffer`. It is the responsibility of the user callback to decode it accordingly to the chaincode-side encoding.

The adapter also gathers the following platform-specific data (if observed) about each transaction, each exposed through a specific key name. The placeholders `<P>` and `<O>` in the key names are node names taking their values from the top-level peers and orderers sections from the network configuration file (e.g., `endorsement_result_peer0.org1.example.com`). The `Get(key:string):any` function returns the value of the observation corresponding to the given key. Alternatively, the `GetCustomData():Map<string,any>` returns the entire collection of gathered data as a `Map`.

The adapter-specific data keys are the following:

| Key name                        | Data type | Description                                                                                                                                                        |
|---------------------------------|-----------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `request_type`                    | string    | Either the `transaction` or `query` string value for traditional transactions or queries, respectively.                                                               |
| `time_endorse`                    | number    | The Unix epoch when the adapter received the proposal responses from the endorsers. Saved even in the case of endorsement errors.                                   |
| `proposal_error`                  | string    | The error message in case an error occurred during sending/waiting for the proposal responses from the endorsers.                                                  |
| `proposal_response_error_<P>`     | string    | The error message in case the endorser peer `<P>` returned an error as endorsement result.                                                                           |
| `endorsement_result_<P>`          | Buffer    | The encoded chaincode invocation result returned by the endorser peer `<P>`. It is the user callback’s responsibility to decode the result.                         |
| `endorsement_verify_error_<P>`    | string    | Has the value of `'INVALID'` if the signature and identity of the endorser peer `<P>` couldn’t be verified.                                                           |
| `endorsement_result_error<P>`     | string    | If the transaction proposal or query execution at the endorser peer `<P>` results in an error, this field contains the error message.                               |
| `read_write_set_error`            | string    | Has the value of '`MISMATCH'` if the sent transaction proposals resulted in different read/write sets.                                                              |
| `time_orderer_ack`                | number    | The Unix epoch when the adapter received the confirmation from the orderer that it successfully received the transaction.                                         |
| `broadcast_error_<O>`             | string    | The warning message in case the adapter did not receive a successful confirmation from the orderer node `<O>`. Note, that this does not mean, that the transaction failed (e.g., a timeout occurred while waiting for the answer due to some transient network delay/error).                    |
| `broadcast_response_error_<O>`    | string    | The error message in case the adapter received an explicit unsuccessful response from the orderer node `<O>`.                                                       |
| `unexpected_error`                | string    | The error message in case some unexpected error occurred during the life-cycle of a transaction.                                                                  |
| `commit_timeout_<P>`              | string    | Has the value of `'TIMEOUT'` in case the event notification about the transaction did not arrive in time from the peer node `<P>`.                                     |
| `commit_error_<P>`                | string    | Contains the error code in case the transaction validation fails at the end of its life-cycle on peer node `<P>`.                                                   |
| `commit_success_<P>`              | number    | The Unix epoch when the adapter received a successful commit event from the peer node `<P>`. Note, that transactions committed in the same block have nearly identical commit times. |
| `event_hub_error_<P>`             | string    | The error message in case some event hub connection-related error occurs with peer node `<P>`.                                                                      |

You can access these data in your user callback after calling `invokeSmartContract` or `querySmartContract`:

```sh
let settings = {
    chaincodeFunction: 'initMarble',
    chaincodeArguments: ['MARBLE#1', 'Red', '100', 'Attila'],
    invokerIdentity: 'client0.org2.example.com'
};

// "results" is of type TxStatus[]
// DO NOT FORGET "await"!!
let results = await blockchain.invokeSmartContract(context, 'marbles', '', settings, 10);

// do something with the data
for (let result of results) {
    let shortID = result.GetID().substring(8);
    let executionTime = result.GetTimeFinal() - result.GetTimeCreate();
    console.log(`TX [${shortID}] took ${executionTime}ms to execute. Result: ${result.GetStatus()}`);
}

// DO NOT FORGET THIS!!
return results;
```

!!! note

    *Do not forget to return the result array at the end of the `run` function!*

## Network configuration file reference
The YAML network configuration file of the adapter builds upon the CCP of the Fabric SDK while adding some Caliper-specific extensions. The definitive documentation for the base CCP is the [corresponding Fabric SDK documentation page](https://hyperledger.github.io/fabric-sdk-node/release-1.4/tutorial-network-config.html). However, this page also includes the description of different configuration elements to make this documentation as self-contained as possible.

The following sections detail each part separately. For a complete example, please refer to the example section or one of the files in the [Caliper benchmarks](https://github.com/hyperledger/caliper-benchmarks/tree/v0.2.0/networks/fabric) repository. Look for files in the `fabric-v*` directories that start with `fabric-*`.

!!! note

    *Unknown keys are not allowed anywhere in the configuration. The only exception is the `info` property and when network artifact names serve as keys (peer names, channel names, etc.).*

<details>
  <summary><b>name</b></summary>

  <i>Required. Non-empty string.</i>
   <br/>
   The name of the configuration file.
   ```sh
   name: Fabric
   ```
</details>

<details>
  <summary><b>version</b></summary>

  <i>Required. Non-empty string.</i>
   <br/>
    Specifies the YAML schema version that the Fabric SDK will use. Only the `'1.0'` string is allowed.
   ```sh
   version: '1.0'
   ```
</details>

<details>
  <summary><b>mutual-tls</b></summary>

  <i>Optional. Boolean.</i>
   <br/>
    Indicates whether to use client-side TLS in addition to server-side TLS. Cannot be set to `true` without using server-side TLS. Defaults to `false`.
   ```sh
   mutual-tls: true
   ```
</details>

<details>
  <summary><b>caliper</b></summary>

  <i>Required. Non-empty object.</i>
  <br/>

  Contains runtime information for Caliper. Can contain the following keys.

  <ul>
    <li>
      <details>
        <summary><b>blockchain</b></summary>

        <i>Required. Non-empty string.</i>
        <br/>

        Only the <code>"fabric"</code> string is allowed for this adapter.

        ```sh
        caliper:
            blockchain: fabric
        ```
      </details>
    </li>
    <li>
      <details>
        <summary><b>command</b></summary>

        <i>Optional. Non-empty object.</i>
        <br/>

        Specifies the start and end scripts.
        <br/>

        <div style="border-left: 4px solid #2196F3; padding-left: 10px; margin: 10px 0;">
          <strong>Note:</strong>
          <p>Must contain at least one of the following keys.</p>
        </div>

        <ul>
          <li>
            <details>
              <summary><b>start</b></summary>

              <i>Optional. Non-empty string.</i><br/>

              Contains the command to execute at startup time. The current working directory for the commands is set to the workspace.

              ```sh
              caliper:
                command:
                  start: my-startup-script.sh
              ```
            </details>
          </li>
          <li>
            <details>
              <summary><b>end</b></summary>

              <i>Optional. Non-empty string.</i><br/>

              Contains the command to execute at exit time. The current working directory for the commands is set to the workspace.

              ```sh
              caliper:
                command:
                  end: my-cleanup-script.sh
              ```
            </details>
          </li>
        </ul>
      </details>
    </li>
  </ul>
</details>

<details>
  <summary><b>info</b></summary>

  <i>Optional. Object.</i>
   <br/>
    Specifies custom key-value pairs that will be included as-is in the generated report. The key-value pairs have no influence on the runtime behavior.

   ```sh
    info:
      Version: 1.1.0
      Size: 2 Orgs with 2 Peers
      Orderer: Solo
      Distribution: Single Host
      StateDB: CouchDB
   ```
</details>

<details>
  <summary><b>wallet</b></summary>

  <i>Optional. Non-empty string.</i>
   <br/>
   Specifies the path to an exported <code>FileSystemWallet</code>. If specified, all interactions will be based on the identities stored in the wallet, and any listed client keys within the <code>clients</code> section must correspond to an existing identity within the wallet.
   ```sh
   wallet: path/to/myFileWallet
   ```
</details>

<details>
  <summary><b>certificateAuthorities</b></summary>

  <i>Optional. Non-empty object.</i>
   <br/>
    The adapter supports the Fabric-CA integration to manage users dynamically at startup. Other CA implementations are currently not supported. If you don’t need to register or enroll users using Caliper, you can omit this section. <br/>

    The top-level <code>certificateAuthorities</code> section contains one or more CA names as keys (matching the <code>ca.name</code> or <code>FABRIC_CA_SERVER_CA_NAME</code> setting of the CA), and each key has a corresponding object (sub-keys) that describes the properties of that CA. The names will be used in other sections to reference a CA.

   ```sh
   certificateAuthorities:
    ca.org1.example.com:
      # properties of CA
    ca.org2.example.com:
      # properties of CA
   ```

    <ul>
    <li>
      <details>
        <summary><b>url</b></summary>

        <i>Required. Non-empty URI string.</i>
        <br/>

        The endpoint of the CA. The protocol must be either <code>http://</code> or <code>https://</code>. Must be <code>https://</code> when using TLS.

        ```sh
        certificateAuthorities:
          ca.org1.example.com:
            url: https://localhost:7054
        ```
      </details>
    </li>
    <li>
      <details>
        <summary><b>httpOptions</b></summary>

        <i>Optional. Object.</i>
        <br/>

        The properties specified under this object are passed to the <code>http</code> client verbatim when sending the request to the Fabric-CA server.

      ```sh
      certificateAuthorities:
        ca.org1.example.com:
          httpOptions:
            verify: false
      ```
      </details>
    </li>
    <li>
      <details>
        <summary><b>tlsCACerts</b></summary>

        <i>Required for TLS. Object.</i>
        <br/>

        Specifies the TLS certificate of the CA for TLS communication. Forbidden to set for non-TLS communication.

        <div style="border-left: 4px solid #2196F3; padding-left: 10px; margin: 10px 0;">
          <strong>Note:</strong>
          <p>Must contain <b>at most one</b> of the following keys.</p>
        </div>

        <ul>
        <li>
        <details>
        <summary><b>path</b></summary>

        <i>Optional. Non-empty string.</i>
        <br/>

        The path of the file containing the TLS certificate.
        ```sh
          certificateAuthorities:
            ca.org1.example.com:
              tlsCACerts:
                path: path/to/cert.pem
        ```
        </details>
        </li>
        <li>
        <details>
        <summary><b>pem</b></summary>

        <i>Optional. Non-empty string.</i>
        <br/>

        The content of the TLS certificate file.
        ```sh
          certificateAuthorities:
            ca.org1.example.com:
              tlsCACerts:
                pem: |
                  -----BEGIN CERTIFICATE-----
                  MIICSDCCAe+gAwIBAgIQfpGy5OOXBYpKZxg89x75hDAKBggqhkjOPQQDAjB2MQsw
                  CQYDVQQGEwJVUzETMBEGA1UECBMKQ2FsaWZvcm5pYTEWMBQGA1UEBxMNU2FuIEZy
                  YW5jaXNjbzEZMBcGA1UEChMQb3JnMS5leGFtcGxlLmNvbTEfMB0GA1UEAxMWdGxz
                  Y2Eub3JnMS5leGFtcGxlLmNvbTAeFw0xODA5MjExNzU3NTVaFw0yODA5MTgxNzU3
                  NTVaMHYxCzAJBgNVBAYTAlVTMRMwEQYDVQQIEwpDYWxpZm9ybmlhMRYwFAYDVQQH
                  Ew1TYW4gRnJhbmNpc2NvMRkwFwYDVQQKExBvcmcxLmV4YW1wbGUuY29tMR8wHQYD
                  VQQDExZ0bHNjYS5vcmcxLmV4YW1wbGUuY29tMFkwEwYHKoZIzj0CAQYIKoZIzj0D
                  AQcDQgAED4FM1+iq04cjveIDyn4uj90lJlO6rASeOIzm/Oc2KQOjpRRlB3H+mVnp
                  rXN6FacjOp0/6OKeEiW392dcdCMvRqNfMF0wDgYDVR0PAQH/BAQDAgGmMA8GA1Ud
                  JQQIMAYGBFUdJQAwDwYDVR0TAQH/BAUwAwEB/zApBgNVHQ4EIgQgPQRWjQR5EUJ7
                  xkV+zbfY618IzOYGIpfLaV8hdlZfWVIwCgYIKoZIzj0EAwIDRwAwRAIgYzk8553v
                  fWAOZLxiDuMN9RiHve1o5aAQad+uD+eLpxMCIBmv8CtXf1C60h/0zyG1D6tTTnrB
                  H8Zua3x+ZQn/kqVv
                  -----END CERTIFICATE-----
        ```
        </details>
        </li>
        </ul>
      </details>
    </li>
    <li>
      <details>
        <summary><b>registrar</b></summary>

        <i>Required. Non-empty, non-spares array.</i>
        <br/>

        A collection of registrar IDs and secrets. Fabric-CA supports dynamic user enrollment via REST APIs. A “root” user, i.e., a registrar is needed to register and enroll new users. Note, that currently <b>only one registrar per CA</b> is supported (regardless of the YAML list notation).

        ```sh
        certificateAuthorities:
          ca.org1.example.com:
            registrar:
            - enrollId: admin
              enrollSecret: adminpw
        ```
        <ul>
        <li>
        <details>
        <summary><b>[item].enrollId</b></summary>

        <i>Required. Non-empty, string.</i>
        <br/>

        The enrollment ID of the registrar. Must be unique on the collection level.
      </details>
        </li>
        <li>
        <details>
        <summary><b>[item].enrollSecret</b></summary>

        <i>Required. Non-empty, string.</i>
        <br/>

        The enrollment secret of the registrar.
      </details>
        </li>
        </ul>
      </details>
    </li>
    </ul>
</details>

<details>
  <summary><b>peers</b></summary>

  <i>Required. Non-empty object.</i>
   <br/>
    Contains one or more, arbitrary but unique peer names as keys, and each key has a corresponding object (sub-keys) that describes the properties of that peer. The names will be used in other sections to reference a peer.
    <br/><br/>
    Can be omitted if only the start/end scripts are executed during benchmarking.

   ```sh
   peers:
    peer0.org1.example.com:
      # properties of peer
    peer0.org2.example.com:
      # properties of peer
   ```

   A peer object (e.g., <code>peer0.org1.example.com</code>) can contain the following properties.
     <ul>
    <li>
      <details>
        <summary><b>url</b></summary>

        <i>Required. Non-empty URI string.</i>
        <br/>

        The (local or remote) endpoint of the peer to send the requests to. If TLS is configured, the protocol must be <code>grpcs://</code>, otherwise it must be <code>grpc://</code>.

        ```sh
        peers:
          peer0.org1.example.com:
            url: grpcs://localhost:7051:
        ```
      </details>
    </li>
    <li>
      <details>
        <summary><b>eventUrl</b></summary>

        <i>Required. Non-empty URI string.</i>
        <br/>

        The (local or remote) endpoint of the peer event service used by the event hub connections. If TLS is configured, the protocol must be <code>grpcs://</code>, otherwise it must be <code>grpc://</code>. Either all peers must contain this setting, or none of them.

        ```sh
        peers:
          peer0.org1.example.com:
            eventUrl: grpcs://localhost:7051:
        ```
      </details>
    </li>
    <li>
      <details>
        <summary><b>grpcOptions</b></summary>

        <i>Optional. Object.</i>
        <br/>

        The properties specified under this object set the gRPC settings used on connections to the Fabric network. See the available options in the gRPC settings tutorial of the Fabric SDK.

        ```sh
        peers:
          peer0.org1.example.com:
            grpcOptions:
              ssl-target-name-override: peer0.org1.example.com
              grpc.keepalive_time_ms: 600000
        ```
      </details>
    </li>
    <li>
      <details>
        <summary><b>tlsCACerts</b></summary>

        <i>Required for TLS. Object.</i>
        <br/>

        Specifies the TLS certificate of the peer for TLS communication. Forbidden to set for non-TLS communication.
        <br/>

        <div style="border-left: 4px solid #2196F3; padding-left: 10px; margin: 10px 0;">
          <strong>Note:</strong>
          <p>Must contain <b>at most one</b> of the following keys.</p>
        </div>

        <ul>
          <li>
            <details>
              <summary><b>path</b></summary>

              <i>Optional. Non-empty string.</i><br/>

              The path of the file containing the TLS certificate.

              ```sh
              peers:
                peer0.org1.example.com:
                  tlsCACerts:
                    path: path/to/cert.pem
              ```
            </details>
          </li>
          <li>
            <details>
              <summary><b>pem</b></summary>

              <i>Optional. Non-empty string.</i><br/>

              Contains the command to execute at exit time. The current working directory for the commands is set to the workspace.

               The content of the TLS certificate file.
              
              ```sh
              peers:
                peer0.org1.example.com:
                  tlsCACerts:
                    pem: |
                    -----BEGIN CERTIFICATE-----
                    MIICSDCCAe+gAwIBAgIQfpGy5OOXBYpKZxg89x75hDAKBggqhkjOPQQDAjB2MQsw
                    CQYDVQQGEwJVUzETMBEGA1UECBMKQ2FsaWZvcm5pYTEWMBQGA1UEBxMNU2FuIEZy
                    YW5jaXNjbzEZMBcGA1UEChMQb3JnMS5leGFtcGxlLmNvbTEfMB0GA1UEAxMWdGxz
                    Y2Eub3JnMS5leGFtcGxlLmNvbTAeFw0xODA5MjExNzU3NTVaFw0yODA5MTgxNzU3
                    NTVaMHYxCzAJBgNVBAYTAlVTMRMwEQYDVQQIEwpDYWxpZm9ybmlhMRYwFAYDVQQH
                    Ew1TYW4gRnJhbmNpc2NvMRkwFwYDVQQKExBvcmcxLmV4YW1wbGUuY29tMR8wHQYD
                    VQQDExZ0bHNjYS5vcmcxLmV4YW1wbGUuY29tMFkwEwYHKoZIzj0CAQYIKoZIzj0D
                    AQcDQgAED4FM1+iq04cjveIDyn4uj90lJlO6rASeOIzm/Oc2KQOjpRRlB3H+mVnp
                    rXN6FacjOp0/6OKeEiW392dcdCMvRqNfMF0wDgYDVR0PAQH/BAQDAgGmMA8GA1Ud
                    JQQIMAYGBFUdJQAwDwYDVR0TAQH/BAUwAwEB/zApBgNVHQ4EIgQgPQRWjQR5EUJ7
                    xkV+zbfY618IzOYGIpfLaV8hdlZfWVIwCgYIKoZIzj0EAwIDRwAwRAIgYzk8553v
                    fWAOZLxiDuMN9RiHve1o5aAQad+uD+eLpxMCIBmv8CtXf1C60h/0zyG1D6tTTnrB
                    H8Zua3x+ZQn/kqVv
                    -----END CERTIFICATE-----
              ```
            </details>
          </li>
        </ul>
      </details>
    </li>
  </ul>
</details>

<details>
  <summary><b>orderers</b></summary>
  <i>Required. Non-empty object.</i>
   <br/>
Contains one or more, arbitrary but unique orderer names as keys, and each key has a corresponding object (sub-keys) that describes the properties of that orderer. The names will be used in other sections to reference an orderer.
<br/>
Can be omitted if only the start/end scripts are executed during benchmarking, or if discovery is enabled.

   ```sh
   orderers:
    orderer1.example.com:
      # properties of orderer
    orderer2.example.com:
      # properties of orderer
   ```
  An orderer object (e.g., <code>orderer1.example.com</code>) can contain the following properties.
  <ul>
    <li>
      <details>
        <summary><b>url</b></summary>

        <i>Required. Non-empty URI string.</i>
        <br/>
        The (local or remote) endpoint of the orderer to send the requests to. If TLS is configured, the protocol must be <code>grpcs://</code>, otherwise it must be <code>grpc://</code>.

        ```sh
        orderers:
          orderer1.example.com:
            url: grpcs://localhost:7050:
        ```
      </details>
    </li>
    <li>
      <details>
        <summary><b>grpcOptions</b></summary>

        <i>Optional. Object.</i>
        <br/>
        The properties specified under this object set the gRPC settings used on connections to the Fabric network. See the available options in the gRPC settings tutorial of the Fabric SDK.

        ```sh
        orderers:
          orderer1.example.com:
            grpcOptions:
              ssl-target-name-override: orderer1.example.com
              grpc.keepalive_time_ms: 600000
        ```
      </details>
    </li>
    <li>
      <details>
        <summary><b>tlsCACerts</b></summary>

        <i>Required for TLS. Object.</i>
        <br/>
        Specifies the TLS certificate of the orderer for TLS communication. Forbidden to set for non-TLS communication.

       <div style="border-left: 4px solid #2196F3; padding-left: 10px; margin: 10px 0;">
          <strong>Note:</strong>
          <p>Must contain <b>at most one</b> of the following keys.</p>
        </div>

        <ul>
        <li>
        <details>
        <summary><b>path</b></summary>

        <i>Optional. Non-empty string.</i>
        <br/>

        The path of the file containing the TLS certificate.
        ```sh
          orderers:
            orderer1.example.com:
              tlsCACerts:
                path: path/to/cert.pem
        ```
        </details>
        </li>
        <li>
        <details>
        <summary><b>pem</b></summary>

        <i>Optional. Non-empty string.</i>
        <br/>

        The content of the TLS certificate file.
        ```sh
          orderers:
            orderer1.example.com:
              tlsCACerts:
                pem: |
                  -----BEGIN CERTIFICATE-----
                  MIICSDCCAe+gAwIBAgIQfpGy5OOXBYpKZxg89x75hDAKBggqhkjOPQQDAjB2MQsw
                  CQYDVQQGEwJVUzETMBEGA1UECBMKQ2FsaWZvcm5pYTEWMBQGA1UEBxMNU2FuIEZy
                  YW5jaXNjbzEZMBcGA1UEChMQb3JnMS5leGFtcGxlLmNvbTEfMB0GA1UEAxMWdGxz
                  Y2Eub3JnMS5leGFtcGxlLmNvbTAeFw0xODA5MjExNzU3NTVaFw0yODA5MTgxNzU3
                  NTVaMHYxCzAJBgNVBAYTAlVTMRMwEQYDVQQIEwpDYWxpZm9ybmlhMRYwFAYDVQQH
                  Ew1TYW4gRnJhbmNpc2NvMRkwFwYDVQQKExBvcmcxLmV4YW1wbGUuY29tMR8wHQYD
                  VQQDExZ0bHNjYS5vcmcxLmV4YW1wbGUuY29tMFkwEwYHKoZIzj0CAQYIKoZIzj0D
                  AQcDQgAED4FM1+iq04cjveIDyn4uj90lJlO6rASeOIzm/Oc2KQOjpRRlB3H+mVnp
                  rXN6FacjOp0/6OKeEiW392dcdCMvRqNfMF0wDgYDVR0PAQH/BAQDAgGmMA8GA1Ud
                  JQQIMAYGBFUdJQAwDwYDVR0TAQH/BAUwAwEB/zApBgNVHQ4EIgQgPQRWjQR5EUJ7
                  xkV+zbfY618IzOYGIpfLaV8hdlZfWVIwCgYIKoZIzj0EAwIDRwAwRAIgYzk8553v
                  fWAOZLxiDuMN9RiHve1o5aAQad+uD+eLpxMCIBmv8CtXf1C60h/0zyG1D6tTTnrB
                  H8Zua3x+ZQn/kqVv
                  -----END CERTIFICATE-----
        ```
        </details>
        </li>
        </ul>
      </details>
    </li>
  </ul>
</details>

<details>
  <summary><b>organizations</b></summary>

  <i>Required. Non-empty object.</i>
   <br/>
Contains one or more, arbitrary but unique organization names as keys, and each key has a corresponding object (sub-keys) that describes the properties of the organization. The names will be used in other sections to reference an organization.

<br/>
Can be omitted if only the start/end scripts are executed during benchmarking.

   ```sh
   organizations:
    Org1:
      # properties of the organization
    Org2:
      # properties of the organization
   ```
   An organization object (e.g., <code>Org1</code>) can contain the following properties.
  <ul>
    <li>
      <details>
        <summary><b>mspid</b></summary>

        <i>Required. Non-empty string.</i>
        <br/>
        The unique MSP ID of the organization.

        ```sh
        organizations:
          Org1:
            mspid: Org1MSP
        ```
      </details>
    </li>
    <li>
      <details>
        <summary><b>peers</b></summary>

        <i>Optional. Non-empty, non-sparse array of strings.</i>
        <br/>
        The list of peer names (from the top-level <code>peers</code> section) that are managed by the organization. Cannot contain duplicate or invalid peer entries.

        ```sh
        organizations:
          Org1:
            peers:
            - peer0.org1.example.com
            - peer1.org1.example.com
        ```
      </details>
    </li>
    <li>
      <details>
        <summary><b>certificateAuthorities</b></summary>

        <i>Optional. Non-empty, non-sparse array of strings.</i>
        <br/>
        The list of CA names (from the top-level <code>certificateAuthorities</code> section) that are managed by the organization. Cannot contain duplicate or invalid CA entries. Note, that currently <b>only one CA</b> is supported.

        ```sh
        organizations:
          Org1:
            certificateAuthorities:
            - ca.org1.example.com
        ```
      </details>
    </li>
    <li>
      <details>
        <summary><b>adminPrivateKey</b></summary>

        <i>Optional. Object.</i>
        <br/>
        Specifies the admin private key for the organization. Required, if an initialization step requires admin signing capabilities (e.g., creating channels, installing/instantiating chaincodes, etc.).

       <div style="border-left: 4px solid #2196F3; padding-left: 10px; margin: 10px 0;">
          <strong>Note:</strong>
          <p>Must contain <b>at most one</b> of the following keys.</p>
        </div>

        <ul>
        <li>
        <details>
        <summary><b>path</b></summary>

        <i>Optional. Non-empty string.</i>
        <br/>

        The path of the file containing the private key.
        ```sh
        organizations:
          Org1:
            adminPrivateKey:
              path: path/to/cert.pem
        ```
        </details>
        </li>
        <li>
        <details>
        <summary><b>pem</b></summary>

        <i>Optional. Non-empty string.</i>
        <br/>

        The content of the private key file.

        ```sh
          organizations:
            Org1:
              adminPrivateKey:
                pem: |
                  -----BEGIN CERTIFICATE-----
                  MIICSDCCAe+gAwIBAgIQfpGy5OOXBYpKZxg89x75hDAKBggqhkjOPQQDAjB2MQsw
                  CQYDVQQGEwJVUzETMBEGA1UECBMKQ2FsaWZvcm5pYTEWMBQGA1UEBxMNU2FuIEZy
                  YW5jaXNjbzEZMBcGA1UEChMQb3JnMS5leGFtcGxlLmNvbTEfMB0GA1UEAxMWdGxz
                  Y2Eub3JnMS5leGFtcGxlLmNvbTAeFw0xODA5MjExNzU3NTVaFw0yODA5MTgxNzU3
                  NTVaMHYxCzAJBgNVBAYTAlVTMRMwEQYDVQQIEwpDYWxpZm9ybmlhMRYwFAYDVQQH
                  Ew1TYW4gRnJhbmNpc2NvMRkwFwYDVQQKExBvcmcxLmV4YW1wbGUuY29tMR8wHQYD
                  VQQDExZ0bHNjYS5vcmcxLmV4YW1wbGUuY29tMFkwEwYHKoZIzj0CAQYIKoZIzj0D
                  AQcDQgAED4FM1+iq04cjveIDyn4uj90lJlO6rASeOIzm/Oc2KQOjpRRlB3H+mVnp
                  rXN6FacjOp0/6OKeEiW392dcdCMvRqNfMF0wDgYDVR0PAQH/BAQDAgGmMA8GA1Ud
                  JQQIMAYGBFUdJQAwDwYDVR0TAQH/BAUwAwEB/zApBgNVHQ4EIgQgPQRWjQR5EUJ7
                  xkV+zbfY618IzOYGIpfLaV8hdlZfWVIwCgYIKoZIzj0EAwIDRwAwRAIgYzk8553v
                  fWAOZLxiDuMN9RiHve1o5aAQad+uD+eLpxMCIBmv8CtXf1C60h/0zyG1D6tTTnrB
                  H8Zua3x+ZQn/kqVv
                  -----END CERTIFICATE-----
        ```
        </details>
        </li>
 </details>
        <li>
      <details>
        <summary><b>signedCert</b></summary>

        <i>Optional. Object.</i>
        <br/>
        Specifies the admin certificate for the organization. Required, if an initialization step requires admin signing capabilities (e.g., creating channels, installing/instantiating chaincodes, etc.).

       <div style="border-left: 4px solid #2196F3; padding-left: 10px; margin: 10px 0;">
          <strong>Note:</strong>
          <p>Must contain <b>at most one</b> of the following keys.</p>
        </div>

        <ul>
        <li>
        <details>
        <summary><b>path</b></summary>

        <i>Optional. Non-empty string.</i>
        <br/>

        The path of the file containing the certificate.
        ```sh
        organizations:
          Org1:
            signedCert:
              path: path/to/cert.pem
        ```
        </details>
        </li>
        <li>
        <details>
        <summary><b>pem</b></summary>

        <i>Optional. Non-empty string.</i>
        <br/>

        The content of the certificate.

        ```sh
          organizations:
            Org1:
              signedCert:
                pem: |
                  -----BEGIN CERTIFICATE-----
                  MIICSDCCAe+gAwIBAgIQfpGy5OOXBYpKZxg89x75hDAKBggqhkjOPQQDAjB2MQsw
                  CQYDVQQGEwJVUzETMBEGA1UECBMKQ2FsaWZvcm5pYTEWMBQGA1UEBxMNU2FuIEZy
                  YW5jaXNjbzEZMBcGA1UEChMQb3JnMS5leGFtcGxlLmNvbTEfMB0GA1UEAxMWdGxz
                  Y2Eub3JnMS5leGFtcGxlLmNvbTAeFw0xODA5MjExNzU3NTVaFw0yODA5MTgxNzU3
                  NTVaMHYxCzAJBgNVBAYTAlVTMRMwEQYDVQQIEwpDYWxpZm9ybmlhMRYwFAYDVQQH
                  Ew1TYW4gRnJhbmNpc2NvMRkwFwYDVQQKExBvcmcxLmV4YW1wbGUuY29tMR8wHQYD
                  VQQDExZ0bHNjYS5vcmcxLmV4YW1wbGUuY29tMFkwEwYHKoZIzj0CAQYIKoZIzj0D
                  AQcDQgAED4FM1+iq04cjveIDyn4uj90lJlO6rASeOIzm/Oc2KQOjpRRlB3H+mVnp
                  rXN6FacjOp0/6OKeEiW392dcdCMvRqNfMF0wDgYDVR0PAQH/BAQDAgGmMA8GA1Ud
                  JQQIMAYGBFUdJQAwDwYDVR0TAQH/BAUwAwEB/zApBgNVHQ4EIgQgPQRWjQR5EUJ7
                  xkV+zbfY618IzOYGIpfLaV8hdlZfWVIwCgYIKoZIzj0EAwIDRwAwRAIgYzk8553v
                  fWAOZLxiDuMN9RiHve1o5aAQad+uD+eLpxMCIBmv8CtXf1C60h/0zyG1D6tTTnrB
                  H8Zua3x+ZQn/kqVv
                  -----END CERTIFICATE-----
        ```
        </details>
        </li>
        </ul>
      </details>
    </li>
  </ul>
</details>

<details>
  <summary><b>clients</b></summary>

  <i>Required. Non-empty object.</i>
   <br/>
    Contains one or more unique client names as keys, and each key has a corresponding object (sub-keys) that describes the properties of that client. These client names can be referenced from the user callback modules when submitting a transaction to set the identity of the invoker.

  ```sh
    clients:
      client0.org1.example.com:
        # properties of the client
      client0.org2.example.com:
        # properties of the client
  ```

  For every client name, there is a single property, called <code>client</code> ( to match the expected format of the SDK), which will contain the actual properties of the client. So the <code>clients</code> section will look like the following:

  ```sh
    clients:
      client0.org1.example.com:
        client:
          # properties of the client
      client0.org2.example.com:
        client:
          # properties of the client
  ```

  The <code>client</code> property of a client object (e.g., <code>client0.org1.example.com</code>) can contain the following properties. The list of required/forbidden properties depend on whether a wallet is configured or note.

  <div style="border-left: 4px solid #2196F3; padding-left: 10px; margin: 10px 0;">
    <strong>Note:</strong> the following constraints apply when a <b>file wallet is configured</b>:</p>
    <ul>
    <li>
    The <code>credentialStore</code>, <code>clientPrivateKey</code>, <code>clientSignedCert</code>, <code>affiliation</code>, <code>attributes</code> and <code>enrollmentSecret</code> properties are forbidden.
    </li>
    <li>
    Each client name <b>must</b> correspond to one of the identities within the provided wallet. If you wish to specify an <code>Admin Client</code> then the naming convention is that of <code>admin.<orgname></code>. If no explicit admin client is provided for an organisation, it is assumed that the first listed client for that organisation is associated with an administrative identity.
    </li>
    </ul>
  </div>

  <div style="border-left: 4px solid #2196F3; padding-left: 10px; margin: 10px 0;">
    <strong>Note:</strong> the following constraints apply when <b>a file wallet is not configured</b>:</p>
    <ul>
    <li>
    <code>credentialStore </code> is required.
    </li>
    <li>
    The following set of properties are mutually exclusive:
    <ul>
    <li>
    <code>clientPrivateKey</code>/<code>clientSignedCert</code> (if one is set, then the other must be set too)
    </li>
    <li>
    <code>affiliation</code>/<code>attributes</code> (if <code>attributes</code> is set, then <code>affiliation</code> must be set too)
    </li>
    <li>
    <code>enrollmentSecret</code>
    </li>
    </li>
    </ul>
  </div>

  <ul>
      <li>
      <details>
      <summary><b>organization</b></summary>

      <i>Required. Non-empty string.</i>
      <br/>

      The name of the organization (from the top-level <code>organizations</code> section) of the client.
      ```sh
        clients:
          client0.org1.example.com:
            client:
              organization: Org1
      ```
      </details>
      </li>
      <li>
      <details>
        <summary><b>credentialStore</b></summary>

        <i>Required without file wallet. Non-empty object.</i>
        <br/>
        The implementation-specific properties of the key-value store. A <code>FileKeyValueStore</code>, for example, has the following properties.
      
     <ul>
        <li>
        <details>
        <summary><b>path</b></summary>
        <i>Required. Non-empty string.</i>
        <br/>

        Path of the directory where the SDK should store the credentials.
        ```sh
        clients:
          client0.org1.example.com:
            client:
              credentialStore:
                path: path/to/store
        ```
        </details>
        </li>
        <li>
        <details>
        <summary><b>cryptoStore</b></summary>

        <i>Required. Non-empty object.</i>
        <br/>
        The implementation-specific properties of the underlying crypto key store. A software-based implementation, for example, requires a store path.
        <ul>
        <li>
        <details>
        <summary><b>path</b></summary>
        <i>Required. Non-empty string.</i>
        <br/>

        The path of the crypto key store directory.
        ```sh
        clients:
          client0.org1.example.com:
            client:
              credentialStore:
                cryptoStore:
                  path: path/to/crypto/store
        ```
        </details>
        </li>
        </ul>
        </details>
        </li>
        </ul>
      </details>
    </li>
    <li>
      <details>
        <summary><b>clientPrivateKey</b></summary>

        <i>Optional. Object.</i>
        <br/>
        Specifies the private key for the client identity.

        <div style="border-left: 4px solid #2196F3; padding-left: 10px; margin: 10px 0;">
          <strong>Note:</strong>
          <p>Must contain <b>at most one</b> of the following keys.</p>
        </div>
      
     <ul>
        <li>
        <details>
        <summary><b>path</b></summary>
        <i>Optional. Non-empty string.</i>
        <br/>

        The path of the file containing the private key.
        ```sh
        clients:
          client0.org1.example.com:
            client:
              clientPrivateKey:
                path: path/to/cert.pem
        ```
        </details>
        </li>
        <li>
        <details>
        <summary><b>pem</b></summary>

        <i>Required. Non-empty string.</i>
        <br/>
        The content of the private key file.
        ```sh
        clients:
          client0.org1.example.com:
            client:
              clientPrivateKey:
                pem: |
                  -----BEGIN CERTIFICATE-----
                  MIICSDCCAe+gAwIBAgIQfpGy5OOXBYpKZxg89x75hDAKBggqhkjOPQQDAjB2MQsw
                  CQYDVQQGEwJVUzETMBEGA1UECBMKQ2FsaWZvcm5pYTEWMBQGA1UEBxMNU2FuIEZy
                  YW5jaXNjbzEZMBcGA1UEChMQb3JnMS5leGFtcGxlLmNvbTEfMB0GA1UEAxMWdGxz
                  Y2Eub3JnMS5leGFtcGxlLmNvbTAeFw0xODA5MjExNzU3NTVaFw0yODA5MTgxNzU3
                  NTVaMHYxCzAJBgNVBAYTAlVTMRMwEQYDVQQIEwpDYWxpZm9ybmlhMRYwFAYDVQQH
                  Ew1TYW4gRnJhbmNpc2NvMRkwFwYDVQQKExBvcmcxLmV4YW1wbGUuY29tMR8wHQYD
                  VQQDExZ0bHNjYS5vcmcxLmV4YW1wbGUuY29tMFkwEwYHKoZIzj0CAQYIKoZIzj0D
                  AQcDQgAED4FM1+iq04cjveIDyn4uj90lJlO6rASeOIzm/Oc2KQOjpRRlB3H+mVnp
                  rXN6FacjOp0/6OKeEiW392dcdCMvRqNfMF0wDgYDVR0PAQH/BAQDAgGmMA8GA1Ud
                  JQQIMAYGBFUdJQAwDwYDVR0TAQH/BAUwAwEB/zApBgNVHQ4EIgQgPQRWjQR5EUJ7
                  xkV+zbfY618IzOYGIpfLaV8hdlZfWVIwCgYIKoZIzj0EAwIDRwAwRAIgYzk8553v
                  fWAOZLxiDuMN9RiHve1o5aAQad+uD+eLpxMCIBmv8CtXf1C60h/0zyG1D6tTTnrB
                  H8Zua3x+ZQn/kqVv
                  -----END CERTIFICATE-----
        ```
        </details>
        </li>
        </ul>
      </details>
    </li>
    <li>
      <details>
        <summary><b>clientSignedCert</b></summary>

        <i>Optional. Object.</i>
        <br/>
        Specifies the certificate for the client identity.

        <div style="border-left: 4px solid #2196F3; padding-left: 10px; margin: 10px 0;">
          <strong>Note:</strong>
          <p>Must contain <b>at most one</b> of the following keys.</p>
        </div>
      
     <ul>
        <li>
        <details>
        <summary><b>path</b></summary>
        <i>Optional. Non-empty string.</i>
        <br/>

        The path of the file containing the certificate.
        ```sh
        clients:
          client0.org1.example.com:
            client:
              clientSignedCert:
                path: path/to/cert.pem
        ```
        </details>
        </li>
        <li>
        <details>
        <summary><b>pem</b></summary>

        <i>Required. Non-empty string.</i>
        <br/>
        The content of the certificate file.
        ```sh
        clients:
          client0.org1.example.com:
            client:
              clientSignedCert:
                pem: |
                  -----BEGIN CERTIFICATE-----
                  MIICSDCCAe+gAwIBAgIQfpGy5OOXBYpKZxg89x75hDAKBggqhkjOPQQDAjB2MQsw
                  CQYDVQQGEwJVUzETMBEGA1UECBMKQ2FsaWZvcm5pYTEWMBQGA1UEBxMNU2FuIEZy
                  YW5jaXNjbzEZMBcGA1UEChMQb3JnMS5leGFtcGxlLmNvbTEfMB0GA1UEAxMWdGxz
                  Y2Eub3JnMS5leGFtcGxlLmNvbTAeFw0xODA5MjExNzU3NTVaFw0yODA5MTgxNzU3
                  NTVaMHYxCzAJBgNVBAYTAlVTMRMwEQYDVQQIEwpDYWxpZm9ybmlhMRYwFAYDVQQH
                  Ew1TYW4gRnJhbmNpc2NvMRkwFwYDVQQKExBvcmcxLmV4YW1wbGUuY29tMR8wHQYD
                  VQQDExZ0bHNjYS5vcmcxLmV4YW1wbGUuY29tMFkwEwYHKoZIzj0CAQYIKoZIzj0D
                  AQcDQgAED4FM1+iq04cjveIDyn4uj90lJlO6rASeOIzm/Oc2KQOjpRRlB3H+mVnp
                  rXN6FacjOp0/6OKeEiW392dcdCMvRqNfMF0wDgYDVR0PAQH/BAQDAgGmMA8GA1Ud
                  JQQIMAYGBFUdJQAwDwYDVR0TAQH/BAUwAwEB/zApBgNVHQ4EIgQgPQRWjQR5EUJ7
                  xkV+zbfY618IzOYGIpfLaV8hdlZfWVIwCgYIKoZIzj0EAwIDRwAwRAIgYzk8553v
                  fWAOZLxiDuMN9RiHve1o5aAQad+uD+eLpxMCIBmv8CtXf1C60h/0zyG1D6tTTnrB
                  H8Zua3x+ZQn/kqVv
                  -----END CERTIFICATE-----
        ```
        </details>
        </li>
        </ul>
      </details>
    </li>
    <li>
      <details>
      <summary><b>affiliation</b></summary>

      <i>Required. Non-empty string.</i>
      <br/>

      If you dynamically register a user through a Fabric-CA, you can provide an affiliation string. The adapter will create the necessary affiliation hierarchy, but you must provide a registrar for the CA that is capable of registering the given affiliation (hierarchy).
      ```sh
        clients:
          client0.org1.example.com:
            client:
              affiliation: org3.sales
      ```
      </details>
    </li>
    <li>
      <details>
        <summary><b>attributes</b></summary>

        <i>Optional. Non-empty, non-spare array of objects.</i>
        <br/>

        If you dynamically register a user through a Fabric-CA, you can provide a collection of key-value attributes to assign to the user.

        ```sh
        clients:
          client0.org1.example.com:
            client:
              attributes:
              - name: departmentId
                value: sales
                ecert: true
        ```
        Each attribute has the following properties.
     <ul>
        <li>
        <details>
        <summary><b>[item].name</b></summary>

        <i>Required. Non-empty string.</i>
        <br/>

        The unique name of the attribute in the collection.
        </details>
        </li>
        <li>
        <details>
        <summary><b>[item].value</b></summary>

        <i>Required. String.</i>
        <br/>

        The string value of the attribute.
        </details>
        </li>
        <li>
        <details>
        <summary><b>[item].ecert</b></summary>

        <i>Optional. Boolean.</i>
        <br/>

        A value of <code>true</code> indicates that this attribute should be included in an enrollment certificate by default.
        </details>
        </li>
        
      </ul>
      </details>
    </li>
    <li>
      <details>
      <summary><b>enrollmentSecret</b></summary>

      <i>Required. Non-empty string.</i>
      <br/>

      For registered (but not enrolled) users you can provide the enrollment secret, and the adapter will enroll the user and retrieve the necessary crypto materials. However, this is a rare scenario.
      ```sh
        clients:
          client0.org1.example.com:
            client:
              enrollmentSecret: secretString
      ```
      </details>
    </li>
    <li>
  <details>
    <summary><b>connection</b></summary>
    <i>Optional. Non-empty object.</i>
    <br/>
    Specifies connection details for the client. Currently, it includes timeout values for different node services. Must contain the following key.
    <ul>
      <li>
        <details>
          <summary><b>timeout</b></summary>
          <i>Required. Non-empty object.</i>
          <br/>
          <div style="border-left: 4px solid #2196F3; padding-left: 10px; margin: 10px 0;">
            <strong>Note:</strong>
            <p>Must contain <b>at least one</b> of the following keys.</p>
          </div>
          <ul>
            <li>
              <details>
                <summary><b>peer</b></summary>
                <i>Optional. Non-empty object.</i>
                <br/>
                Specifies timeout values for peer services.
                <div style="border-left: 4px solid #2196F3; padding-left: 10px; margin: 10px 0;">
                  <strong>Note:</strong>
                  <p>Must contain <b>at least one</b> of the following keys.</p>
                </div>
                <ul>
                  <li>
                  <details>
                  <summary><b>endorser</b></summary>
                  <i>Optional. Positive integer.</i>
                  <br/>
                  Timeout value in seconds to use for peer requests.
                  <pre><code>
                  clients:
                    client0.org1.example.com:
                      client:
                        connection:
                          peer:
                            endorser: 120
                  </code></pre>
                  </details>
                  </li>
                  <li>
                  <details>
                  <summary><b>eventhub</b></summary>
                  <i>Optional. Positive integer.</i>
                  <br/>
                  Timeout value in seconds to use for waiting for registered events to occur.
                  <pre><code>
                  clients:
                    client0.org1.example.com:
                      client:
                        connection:
                          peer:
                            eventHub: 120
                  </code></pre>
                  </details>
                  </li>
                  <li>
                  <details>
                  <summary><b>eventReg</b></summary>
                  <i>Optional. Positive integer.</i>
                  <br/>
                  Timeout value in seconds to use when connecting to an event hub.
                  <pre><code>
                  clients:
                    client0.org1.example.com:
                      client:
                        connection:
                          peer:
                            eventReg: 120
                  </code></pre>
                  </details>
                  </li>
                </ul>
              </details>
            </li>
            <li>
              <details>
                <summary><b>orderer</b></summary>
                <i>Optional. Positive integer.</i>
                <br/>
                Timeout value in seconds to use for orderer requests.
                <pre><code>
                clients:
                  client0.org1.example.com:
                    client:
                     connection:
                        orderer: 30
                </code></pre>
              </details>
            </li>
          </ul>
        </details>
      </li>
    </ul>
  </details>
</li>

  </ul>

</details>

<details>
  <summary><b>channels</b></summary>

  <i>Required. Non-empty object.</i>
   <br/>
    Contains one or more unique channel names as keys, and each key has a corresponding object (sub-keys) that describes the properties of the channel.
    <br/>
    Can be omitted if only the start/end scripts are executed during benchmarking.

   ```sh
   channels:
    mychannel:
      # properties of the channel
    yourchannel:
      # properties of the channel-tls: true
   ```
   A channel object (e.g., <code>mychannel</code>) can contain the following properties.

   <ul>
    <li>
      <details>
        <summary><b>created</b></summary>

        <i>Optional. Boolean.</i>
        <br/>

       Indicates whether the channel already exists or not. If <code>true</code>, then the adapter will not try to create the channel again (which would fail). Defaults to <code>false</code>.

        <div style="border-left: 4px solid #2196F3; padding-left: 10px; margin: 10px 0;">
          <strong>Note:</strong>when a channel needs to be created, either <code>configBinary</code> or <code>definition</code> must be set!
        </div>

        ```sh
        channels:
          mychannel:
            created: false
        ```
      </details>
    </li>
    <li>
      <details>
        <summary><b>configBinary</b></summary>

        <i>Optional. Non-empty string.</i>
        <br/>

        If a channel doesn’t exist yet, the adapter will create it based on the provided path of a channel configuration binary (which is typically the output of the configtxgen tool).

        <div style="border-left: 4px solid #2196F3; padding-left: 10px; margin: 10px 0;">
          <strong>Note:</strong>if <code>created</code> is false, and the <code>definition</code> property is provided, then this property is forbidden.
        </div>

      ```sh
      channels:
        mychannels:
          configBinary: my/path/to/binary.tx
      ```
      </details>
    </li>
    <li>
      <details>
        <summary><b>definition</b></summary>

        <i>Optional. Object.</i>
        <br/>

        If a channel doesn’t exist yet, the adapter will create it based on the provided channel definition, consisting of multiple properties.

        ```sh
        channels:
          mychannel:
            definition:
              capabilities: []
              consortium: SampleConsortium
              msps: ['Org1MSP', 'Org2MSP']
              version: 0
        ```

     <ul>
        <li>
        <details>
        <summary><b>capabilities</b></summary>

        <i>Required. Non-sparse array of strings.</i>
        <br/>

        List of channel capabilities to include in the configuration transaction.
        </details>
        </li>
        <li>
        <details>
        <summary><b>consortium</b></summary>

        <i>Required. Non-empty string.</i>
        <br/>

        The name of the consortium.
        </details>
        </li>
        <li>
        <details>
        <summary><b>msps</b></summary>

        <i>Required. Non-sparse array of unique strings.</i>
        <br/>
        The MSP IDs of the organizations in the channel.

        </details>
        </li>
        <li>
        <details>
        <summary><b>version</b></summary>

        <i>Required. Non-negative integer.</i>
        <br/>
        The version number of the configuration.
        </details>
        </li>
        </ul>
      </details>
    </li>
    <li>
      <details>
        <summary><b>orderers</b></summary>

        <i>Optional. Non-spares array of unique string.</i>
        <br/>

        a list of orderer node names (from the top-level <code>orderers</code> section) that participate in the channel.

        <div style="border-left: 4px solid #2196F3; padding-left: 10px; margin: 10px 0;">
          <strong>Note:</strong>if discovery is disabled, then this property is required!
        </div>

      ```sh
      channels:
        mychannel:
          orderers: ['orderer1.example.com', 'orderer2.example.com']
      ```
      </details>
    </li>
    <li>
      <details>
        <summary><b>peers</b></summary>

        <i>Required. Object.</i>
        <br/>

        Contains the peer node names as properties (from the top-level <code>peers</code> section) that participate in the channel.

        ```sh
        channels:
          mychannel:
            peers:
              peer0.org1.example.com:
                # use default values when object is empty
              peer0.org2.example.com:
                endorsingPeer: true
                chaincodeQuery: true
                ledgerQuery: false
                eventSource: true
        ```
      Each key is an object that can have the following properties.
     <ul>
        <li>
        <details>
        <summary><b>endorsingPeer</b></summary>

        <i>Optional. Boolean.</i>
        <br/>

        Indicates whether the peer will be sent transaction proposals for endorsement. The peer must have the chaincode installed. Caliper can also use this property to decide which peers to send the chaincode install request. Default: true
        </details>
        </li>
        <li>
        <details>
        <summary><b>chaincodeQuery</b></summary>

        <i>Optional. Boolean.</i>
        <br/>
        Indicates whether the peer will be sent query proposals. The peer must have the chaincode installed. Caliper can also use this property to decide which peers to send the chaincode install request. Default: true
        </details>
        </li>
        <li>
        <details>
        <summary><b>ledgerQuery</b></summary>

        <i>Optional. Boolean.</i>
        <br/>
        Indicates whether the peer will be sent query proposals that do not require chaincodes, like `queryBlock()`, `queryTransaction()`, etc. Currently unused. Default: true

        </details>
        </li>
        <li>
        <details>
        <summary><b>eventSource</b></summary>

        <i>Optional. Boolean.</i>
        <br/>
        Indicates whether the peer will be the target of an event listener registration. All peers can produce events but Caliper typically only needs to connect to one to listen to events. Default: true

        </details>
        </li>
        </ul>
      </details>
    </li>
    <li>
      <details>
        <summary><b>chaincodes</b></summary>

        <i>Required. Non-sparse array of objects.</i>
        <br/>
        Each array element contains information about a chaincode in the channel.

        <div style="border-left: 4px solid #2196F3; padding-left: 10px; margin: 10px 0;">
          <strong>Note:</strong>
          <p>the <code>contractID</code> value of <b>every</b> chaincode in <b>every</b> channel must be unique on the configuration file level! If <code>contractID</code> is not specified for a chaincode then its default value is the <code>id</code> of the chaincode.</p>
        </div>

        ```sh
        channels:
          mychannel:
            chaincodes:
            - id: simple
              # other properties of simple CC
            - id: smallbank
              # other properties of smallbank CC
        ```
        Some properties are required depending on whether a chaincode needs to be deployed. The following constraints apply:

        <div style="border-left: 4px solid #2196F3; padding-left: 10px; margin: 10px 0;">
          <strong>Note:</strong>
          <h5>Constraints for Installing Chaincodes:</h5>
          <ul>
            <li>If <code>metadataPath</code> is provided, <code>path</code> is also required.</li>
            <li>If <code>path</code> is provided, <code>language</code> is also required.</li>
          </ul> 

          <h3>Constraints for Instantiating Chaincodes:</h3>

          <ul>
            <li>if any of the following properties are provided, <code>language</code> is also needed:<code>init</code>,<code>function</code>,<code>initTransientMap</code>,<code>collections-config</code>,<code>endorsement-policy</code>. Each element can contain the following properties.</li>
          </ul>

        </div>
        <ul>
        <li>
        <details>
        <summary><b>[item].id</b></summary>

        <i>Required. Non-empty string.</i>
        <br/>

        The ID of the chaincode.
        ```sh
          channels:
            mychannel:
              chaincodes:
              - id: simple
                # other properties:
        ```
        </details>
        </li>
        <li>
        <details>
        <summary><b>[item].version</b></summary>

        <i>Required. Non-empty string.</i>
        <br/>

        The version string of the chaincode.
        ```sh
          channels:
            mychannel:
              chaincodes:
              - version: v1.0
                # other properties:
        ```
        </details>
        </li>
        <li>
        <details>
        <summary><b>[item].contractID</b></summary>

        <i>Optional. Non-empty string.</i>
        <br/>

        The Caliper-level unique ID of the chaincode. This ID will be referenced from the user callback modules. Can be an arbitrary name, it won’t effect the chaincode properties on the Fabric side.
        <br/>
        If omitted, it defaults to the <code>id</code> property value.
        
        ```sh
          channels:
            mychannel:
              chaincodes:
              - contractID: simpleContract
                # other properties:
        ```
        </details>
        </li>
        <li>
        <details>
        <summary><b>[item].language</b></summary>

        <i>Optional. Non-empty string.</i>
        <br/>

        The path to the chaincode directory. For golang chaincodes, it is the fully qualified package name (relative to the <code>GOPATH/src</code> directory). Note, that <code>GOPATH</code> is temporarily set to the workspace directory by default. To disable this behavior, set the <code>caliper-fabric-overwritegopath</code> setting key to <code>false</code>.

        ```sh
          channels:
            mychannel:
              chaincodes:
              - path: contracts/mycontract
                # other properties:
        ```
        </details>
        </li>
        <li>
        <details>
        <summary><b>[item].metadataPath</b></summary>

        <i>Optional. Non-empty string.</i>
        <br/>

        The directory path for additional metadata for the chaincode (like CouchDB indexes). Only supported since Fabric v1.1.
        ```sh
          channels:
            mychannel:
              chaincodes:
              - metadataPath: contracts/mycontract/metadata
                # other properties:
        ```
        </details>
        </li>
        <li>
        <details>
        <summary><b>[item].init</b></summary>

        <i>Optional. Non-sparse array of string.</i>
        <br/>

        The list of string arguments to pass to the chaincode’s <code>Init</code> function during instantiation.

        ```sh
          channels:
            mychannel:
              chaincodes:
              - init: ['arg1', 'arg2']
                # other properties:
        ```
        </details>
        </li>
        <li>
        <details>
        <summary><b>[item].function</b></summary>

        <i>Optional. Non-empty string.</i>
        <br/>

        The function name to pass to the chaincode’s <code>Init</code> function during instantiation.

        ```sh
          channels:
            mychannel:
              chaincodes:
              - function: 'init'
                # other properties:
        ```
        </details>
        </li>
        <li>
        <details>
        <summary><b>[item].initTransientMap</b></summary>

        <i>Optional. Object containing string keys associated with string values.</i>
        <br/>

        The transient key-value map to pass to the Init function when instantiating a chaincode. The adapter encodes the values as byte arrays before sending them.
        ```sh
          channels:
            mychannel:
              chaincodes:
                - initTransientMap:
                  pemContent: |
                    -----BEGIN PRIVATE KEY-----
                    MIGHAgEAMBMGByqGSM49AgEGCCqGSM49AwEHBG0wawIBAQQgQDk37WuVcnQUjE3U
                    NTW7PpPfcp54q/KBKNrtFXjAtUChRANCAAQ0xnSUxoocDsb2YIrmtFIKZ4XAiwqu
                    V0BCfsl+ByVKUUdXypNrluQfm28AxX7sEDQLKtHVmuMi/BGaKahZ6Snk
                    -----END PRIVATE KEY-----
                  stringArg: this is also passed as a byte array
                # other properties
        ```
        </details>
        </li>
        <li>
        <details>
        <summary><b>[item].collections-config</b></summary>

        <i>Optional. Non-empty, non-sparse array of objectss.</i>
        <br/>

       List of private collection definitions for the chaincode or a path to the JSON file containing the definitions. For details about the content of such definitions, refer to the <a href="https://hyperledger.github.io/fabric-sdk-node/release-1.4/tutorial-private-data.html" target='_blank'>SDK page</>.
        ```sh
          channels:
            mychannel:
              chaincodes:
              - language: node
                # other properties:
        ```
        </details>
        </li>
        <li>
        <details>
        <summary><b>[item].endorsement-policy</b></summary>

        <i>Optional. Object.</i>
        <br/>

        The endorsement policy of the chaincode as required by the Fabric Node.js SDK. If omitted, then a default N-of-N policy is used based on the target peers (thus organizations) of the chaincode.

        ```sh
          channels:
            mychannel:
              chaincodes:
              - endorsement-policy:
                identities:
                - role:
                    name: member
                    mspId: Org1MSP
                - role:
                    name: member
                    mspId: Org2MSP
                policy:
                  2-of:
                  - signed-by: 0
                  - signed-by: 1
                # other properties:
        ```
        </details>
        </li>
        <li>
        <details>
        <summary><b>[item].targetPeers</b></summary>

        <i>Optional. Non-empty, non-sparse array of strings.</i>
        <br/>

        Specifies custom target peers (from the top-level <code>peers</code> section) for chaincode installation/instantiation. Overrides the peer role-based channel-level targets.
        ```sh
          channels:
            mychannel:
              chaincodes:
              - targetPeers
                - peer0.org1.example.com
                - peer1.org1.example.com
                - peer0.org2.example.com
                - peer1.org2.example.com
                # other properties:
        ```
        </details>
        </li>
        </ul>
        </details>
</details>

## Connection Profile Example

- two organizations;
- each organization has two peers and a CA;
- the first organization has one user/client, the second has two (and the second user is dynamically registered and enrolled);
- one orderer;
- one channel named `mychannel` that is created by Caliper;
- `marbles@v0` chaincode installed and instantiated in `mychannel` on every peer;
- Nodes of the network use TLS communication, but not mutual TLS;
- The local network is deployed and cleaned up automatically by Caliper.

```sh
name: Fabric
version: "1.0"

mutual-tls: false

caliper:
  blockchain: fabric
  command:
    start: docker-compose -f network/fabric-v1.1/2org2peergoleveldb/docker-compose-tls.yaml up -d;sleep 3s
    end: docker-compose -f network/fabric-v1.1/2org2peergoleveldb/docker-compose-tls.yaml down;docker rm $(docker ps -aq);docker rmi $(docker images dev* -q)

info:
  Version: 1.1.0
  Size: 2 Orgs with 2 Peers
  Orderer: Solo
  Distribution: Single Host
  StateDB: GoLevelDB

clients:
  client0.org1.example.com:
    client:
      organization: Org1
      credentialStore:
        path: "/tmp/hfc-kvs/org1"
        cryptoStore:
          path: "/tmp/hfc-cvs/org1"
      clientPrivateKey:
        path: network/fabric-v1.1/config/crypto-config/peerOrganizations/org1.example.com/users/User1@org1.example.com/msp/keystore/key.pem
      clientSignedCert:
        path: network/fabric-v1.1/config/crypto-config/peerOrganizations/org1.example.com/users/User1@org1.example.com/msp/signcerts/User1@org1.example.com-cert.pem
  client0.org2.example.com:
    client:
      organization: Org2
      credentialStore:
        path: "/tmp/hfc-kvs/org2"
        cryptoStore:
          path: "/tmp/hfc-cvs/org2"
      clientPrivateKey:
        path: network/fabric-v1.1/config/crypto-config/peerOrganizations/org2.example.com/users/User1@org2.example.com/msp/keystore/key.pem
      clientSignedCert:
        path: network/fabric-v1.1/config/crypto-config/peerOrganizations/org2.example.com/users/User1@org2.example.com/msp/signcerts/User1@org2.example.com-cert.pem
  client1.org2.example.com:
    client:
      organization: Org2
      affiliation: org2.department1
      role: client
      credentialStore:
        path: "/tmp/hfc-kvs/org2"
        cryptoStore:
          path: "/tmp/hfc-cvs/org2"

channels:
  mychannel:
    configBinary: network/fabric-v1.1/config/mychannel.tx
    created: false
    orderers:
    - orderer.example.com
    peers:
      peer0.org1.example.com:
        endorsingPeer: true
        chaincodeQuery: true
        ledgerQuery: true
        eventSource: true
      peer1.org1.example.com:
      peer0.org2.example.com:
      peer1.org2.example.com:

    chaincodes:
    - id: marbles
      version: v0
      targetPeers:
      - peer0.org1.example.com
      - peer1.org1.example.com
      - peer0.org2.example.com
      - peer1.org2.example.com
      language: node
      path: src/marbles/node
      metadataPath: src/marbles/node/metadata
      init: []
      function: init

      initTransientMap:
        pemContent: |
          -----BEGIN PRIVATE KEY-----
          MIGHAgEAMBMGByqGSM49AgEGCCqGSM49AwEHBG0wawIBAQQgQDk37WuVcnQUjE3U
          NTW7PpPfcp54q/KBKNrtFXjAtUChRANCAAQ0xnSUxoocDsb2YIrmtFIKZ4XAiwqu
          V0BCfsl+ByVKUUdXypNrluQfm28AxX7sEDQLKtHVmuMi/BGaKahZ6Snk
          -----END PRIVATE KEY-----
        stringArg: this is also passed as a byte array

      endorsement-policy:
        identities:
        - role:
            name: member
            mspId: Org1MSP
        - role:
            name: member
            mspId: Org2MSP
        policy:
          2-of:
          - signed-by: 0
          - signed-by: 1

organizations:
  Org1:
    mspid: Org1MSP
    peers:
    - peer0.org1.example.com
    - peer1.org1.example.com
    certificateAuthorities:
    - ca.org1.example.com
    adminPrivateKey:
      path: network/fabric-v1.1/config/crypto-config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp/keystore/key.pem
    signedCert:
      path: network/fabric-v1.1/config/crypto-config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp/signcerts/Admin@org1.example.com-cert.pem

  Org2:
    mspid: Org2MSP
    peers:
    - peer0.org2.example.com
    - peer1.org2.example.com
    certificateAuthorities:
    - ca.org2.example.com
    adminPrivateKey:
      path: network/fabric-v1.1/config/crypto-config/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp/keystore/key.pem
    signedCert:
      path: network/fabric-v1.1/config/crypto-config/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp/signcerts/Admin@org2.example.com-cert.pem

orderers:
  orderer.example.com:
    url: grpcs://localhost:7050
    grpcOptions:
      ssl-target-name-override: orderer.example.com
      grpc-max-send-message-length: 15
    tlsCACerts:
      path: network/fabric-v1.1/config/crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem

peers:
  peer0.org1.example.com:
    url: grpcs://localhost:7051
    grpcOptions:
      ssl-target-name-override: peer0.org1.example.com
      grpc.keepalive_time_ms: 600000
    tlsCACerts:
      path: network/fabric-v1.1/config/crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/msp/tlscacerts/tlsca.org1.example.com-cert.pem

  peer1.org1.example.com:
    url: grpcs://localhost:7057
    grpcOptions:
      ssl-target-name-override: peer1.org1.example.com
      grpc.keepalive_time_ms: 600000
    tlsCACerts:
      path: network/fabric-v1.1/config/crypto-config/peerOrganizations/org1.example.com/peers/peer1.org1.example.com/msp/tlscacerts/tlsca.org1.example.com-cert.pem

  peer0.org2.example.com:
    url: grpcs://localhost:8051
    grpcOptions:
      ssl-target-name-override: peer0.org2.example.com
      grpc.keepalive_time_ms: 600000
    tlsCACerts:
      path: network/fabric-v1.1/config/crypto-config/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/msp/tlscacerts/tlsca.org2.example.com-cert.pem

  peer1.org2.example.com:
    url: grpcs://localhost:8057
    grpcOptions:
      ssl-target-name-override: peer1.org2.example.com
      grpc.keepalive_time_ms: 600000
    tlsCACerts:
      path: network/fabric-v1.1/config/crypto-config/peerOrganizations/org2.example.com/peers/peer1.org2.example.com/msp/tlscacerts/tlsca.org2.example.com-cert.pem

certificateAuthorities:
  ca.org1.example.com:
    url: https://localhost:7054
    httpOptions:
      verify: false
    tlsCACerts:
      path: network/fabric-v1.1/config/crypto-config/peerOrganizations/org1.example.com/tlsca/tlsca.org1.example.com-cert.pem
    registrar:
    - enrollId: admin
      enrollSecret: adminpw

  ca.org2.example.com:
    url: https://localhost:8054
    httpOptions:
      verify: false
    tlsCACerts:
      path: network/fabric-v1.1/config/crypto-config/peerOrganizations/org2.example.com/tlsca/tlsca.org2.example.com-cert.pem
    registrar:
    - enrollId: admin
      enrollSecret: adminpw
```