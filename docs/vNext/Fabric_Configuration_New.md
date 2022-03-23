---
layout: vNext
title:  "Fabric"
categories: config
permalink: /vNext/fabric-config/new/
order: 3
---

## Table of contents

{:.no_toc}

- TOC
{:toc}

## Overview

This page introduces the Fabric adapter that utilizes the Common Connection Profile (CCP) feature of the Fabric SDK to provide compatibility and a unified programming model across different Fabric versions.

> The latest supported version of Hyperledger Fabric is v2.2 and later 2.x releases (eg 2.4)



The adapter exposes many SDK features directly to the user callback modules, making it possible to implement complex scenarios.

> Some highlights of the provided features:
> * supports multiple channels and contracts
> * supports multiple organizations
> * supports multiple identities
> * private collection support for contracts
> * support for TLS and mutual TLS communication
> * option to select the identity for submitting a TX/query

## Installing dependencies

You must bind Caliper to a specific Fabric SDK to target the corresponding (or compatible) SUT version. Refer to the [binding documentation](./Installing_Caliper.md#the-bind-command) for details.

### Binding with Fabric SDK 1.4

It is confirmed that a 1.4 Fabric SDK is compatible with a Fabric 2.2 and later Fabric 2.x SUTs.

Note that when using the binding target for the Fabric SDK 1.4 there are capability restrictions:
> * The 1.4 SDK does not support administration actions. It it not possible to create/join channels, nor install/instantiate a contract. Consequently the 1.4 binding only facilitates operation with a `--caliper-flow-only-test` flag
> * Currently setting `discover` to `true` in the network configuration file is not supported if you don't specify `--caliper-fabric-gateway-enabled` when bound to the Fabric 1.4 SUT
> * Detailed execution data for every transaction is only available if you don't enable the `gateway` option

### Binding with Fabric SDK 2.2

It is confirmed that a 2.2 Fabric SDK is compatible with later Fabric 2.x SUTs.

> Note that when using the binding target for the Fabric SDK 2.x there are capability restrictions:
> * The 2.x SDK does not facilitate administration actions. It it not possible to create/join channels, nor install/instantiate contract. Consequently the 2.2 binding only facilitates operation with a `--caliper-flow-only-test` flag
> * Detailed execution data for every transaction is not available.

## Runtime settings

### Common settings

Some runtime properties of the adapter can be set through Caliper's [runtime configuration mechanism](./Runtime_Configuration.md). For the available settings, see the `caliper.fabric` section of the [default configuration file](https://github.com/hyperledger/caliper/blob/main/packages/caliper-core/lib/common/config/default.yaml) and its embedded documentation.

The above settings are processed when starting Caliper. Modifying them during testing will have no effect. However, you can override the default values _before Caliper starts_ from the usual configuration sources.

> __Note:__ An object hierarchy in a configuration file generates a setting entry for every leaf property. Consider the following configuration file:
> ```yaml
> caliper:
>   fabric:
>     gateway:
>       localhost: false
> ```
> After naming the [project settings](./Runtime_Configuration.md#project-level) file `caliper.yaml` and placing it in the root of your workspace directory, it will override the following two setting keys with the following values:
> * Setting `caliper-fabric-gateway-localhost` is set to `false`
>
> __The other settings remain unchanged.__
>
> Alternatively you can change this setting when you launch caliper with the CLI options of
> ```
> --caliper-fabric-gateway-localhost false
> ```

## The connector API

The [workload modules](./Workload_Module.md) interact with the adapter at three phases of the tests: during the initialization of the user module (in the `initializeWorkloadModule` callback), when submitting invoke or query transactions (in the `submitTransaction` callback), and at the optional cleanup of the user module (in the `cleanupWorkloadModule` callback).

### The `initializeWorkloadModule` function

See the [corresponding documentation](./Workload_Module.md#initializeworkloadmodule) of the function for the description of its parameters.

The last argument of the function is a `sutContext` object, which is a platform-specific object provided by the backend blockchain's connector. The context object provided by this connector is a `FabricConnectorContext` instance but this doesn't provide anything of use at this time.

For the current details/documentation of the API, refer to the [source code](https://github.com/hyperledger/caliper/blob/main/packages/caliper-fabric/lib/FabricConnectorContext.js).

### The `submitTransaction` function

The `sutAdapter` object received (and saved) in the `initializeWorkloadModule` function is of type [`ConnectorInterface`](https://github.com/hyperledger/caliper/blob/main/packages/caliper-core/lib/common/core/connector-interface.js). Its `getType()` function returns the `fabric` string value.

The `sendRequests` method of the connector API allows the workload module to submit requests to the SUT. It takes a single parameter: an object or array of objects containing the settings of the requests.

The settings object has the following structure:

* `contractId`: _string. Required._ The ID of the contract to call.
* `contractFunction`: _string. Required._ The name of the function to call in the contract.
* `contractArguments`: _string[]. Optional._ The list of __string__ arguments to pass to the contract.
* `readOnly`: _boolean. Optional._ Indicates whether the request is a TX or a query. Defaults to `false`.

* `transientMap`: _Map<string, byte[]>. Optional._ The transient map to pass to the contract.
* `invokerIdentity`: _string. Optional._ The name of the user who should invoke the contract. If not provided a user will be selected from the organization defined by `invokerMspId` or the first organization in the network configuration file if that property is not provided
* `invokerMspId`: _string. Optional._ The mspid of the user organization who should invoke the contract. Defaults to the first organization in the network configuration file.
* `targetPeers`: _string[]. Optional._ An array of endorsing peer names as the targets of the transaction proposal. If omitted, the target list will be chosen for you. If discovery is used then the node sdk uses discovery to determine the correct peers.
* `targetOrganizations`: _string[]. Optional._ An array of endorsing organizations as the targets of the invoke. If both targetPeers and targetOrganizations are specified then targetPeers will take precedence
* `channel`: _string. Optional._ The name of the channel on which the contract to call resides.
* `timeout`: _number. Optional._ [**Only applies to 1.4 binding when not enabling gateway use**] The timeout in seconds to use for this request.
* `orderer`: _string. Optional._ [**Only applies to 1.4 binding when not enabling gateway use**] The name of the target orderer for the transaction broadcast. If omitted, then an orderer node of the channel will be automatically selected.

So invoking a contract looks like the following:

```js
let requestSettings = {
    contractId: 'marbles',
    contractFunction: 'initMarble',
    contractArguments: ['MARBLE#1', 'Red', '100', 'Attila'],
    invokerIdentity: 'client0.org2.example.com',
    timeout: 10
};

await this.sutAdapter.sendRequests(requestSettings);
```

> __Note:__ `sendRequests` also accepts an array of request settings. However, Fabric does not support submitting an atomic batch of transactions like Sawtooth, so there is no guarantee that the order of these transactions will remain the same, or whether they will reside in the same block.

## Gathered TX data

The previously discussed  `sendRequests` function returns the result (or an array of results) for the submitted request(s) with the type of [TxStatus](https://github.com/hyperledger/caliper/blob/main/packages/caliper-core/lib/transaction-status.js). The class provides some standard and platform-specific information about its corresponding transaction.

The standard data provided are the following:
* `GetID():string` returns the transaction ID.
* `GetStatus():string` returns the final status of the transaction, either `success` or `failed`.
* `GetTimeCreate():number` returns the epoch when the transaction was submitted.
* `GetTimeFinal():number` return the epoch when the transaction was finished.
* `IsVerified():boolean` indicates whether we are sure about the final status of the transaction. Unverified (considered failed) transactions could occur, for example, if the adapter loses the connection with every Fabric event hub, missing the final status of the transaction.
* `GetResult():Buffer` returns one of the endorsement results returned by the contract as a `Buffer`. It is the responsibility of the user callback to decode it accordingly to the contract-side encoding.

The adapter also gathers the following platform-specific data (if observed) about each transaction, each exposed through a specific key name. The placeholders `<P>` and `<O>` in the key names are node names taking their values from the top-level peers and orderers sections from the network configuration file (e.g., `endorsement_result_peer0.org1.example.com`). The `Get(key:string):any` function returns the value of the observation corresponding to the given key. Alternatively, the `GetCustomData():Map<string,any>` returns the entire collection of gathered data as a `Map`.

### Available data keys for all Fabric SUTs
The adapter-specific data keys that are available when binding to any of the Fabric SUT versions are :

|            Key name            | Data type | Description                                                                                                                                                                                                                                                                  |
|:------------------------------:|:---------:|:-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|         `request_type`         |  string   | Either the `transaction` or `query` string value for traditional transactions or queries, respectively. |

## Available data keys for the Fabric 1.4 SUT when gateway is not enabled
The adapter-specific data keys that only the v1.4 SUT when not enabling the gateway makes available are :

|            Key name            | Data type | Description                                                                                                                                                                                                                                                           |
|:------------------------------:|:---------:|:-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|         `time_endorse`         |  number   | The Unix epoch when the adapter received the proposal responses from the endorsers. Saved even in the case of endorsement errors.                                                                                                                                   |
|        `proposal_error`        |  string   | The error message in case an error occurred during sending/waiting for the proposal responses from the endorsers.                                                                                                                                                            |
| `proposal_response_error_<P>`  |  string   | The error message in case the endorser peer `<P>` returned an error as endorsement result.                                                                                                                                                                                   |
|    `endorsement_result_<P>`    |  Buffer   | The encoded contract invocation result returned by the endorser peer `<P>`. It is the user callback's responsibility to decode the result.                                                                                                                                  |
| `endorsement_verify_error_<P>` |  string   | Has the value of `'INVALID'` if the signature and identity of the endorser peer `<P>` couldn't be verified. This verification step can be switched on/off through the [runtime configuration options](#runtime-settings).                                                    |
| `endorsement_result_error<P>`  |  string   | If the transaction proposal or query execution at the endorser peer `<P>` results in an error, this field contains the error message.                                                                                                                                        |
|     `read_write_set_error`     |  string   | Has the value of `'MISMATCH'` if the sent transaction proposals resulted in different read/write sets.                                                                                                                                                                       |
|       `time_orderer_ack`       |  number   | The Unix epoch when the adapter received the confirmation from the orderer that it successfully received the transaction. Note, that this isn't the actual ordering time of the transaction.                                                                                 |
|     `broadcast_error_<O>`      |  string   | The warning message in case the adapter did not receive a successful confirmation from the orderer node `<O>`. Note, that this does not mean, that the transaction failed (e.g., a timeout occurred while waiting for the answer due to some transient network delay/error). |
| `broadcast_response_error_<O>` |  string   | The error message in case the adapter received an explicit unsuccessful response from the orderer node `<O>`.                                                                                                                                                                |
|       `unexpected_error`       |  string   | The error message in case some unexpected error occurred during the life-cycle of a transaction.                                                                                                                                                                             |
|      `commit_timeout_<P>`      |  string   | Has the value of `'TIMEOUT'` in case the event notification about the transaction did not arrive in time from the peer node `<P>`.                                                                                                                                           |
|       `commit_error_<P>`       |  string   | Contains the error code in case the transaction validation fails at the end of its life-cycle on peer node `<P>`.                                                                                                                                                            |
|      `commit_success_<P>`      |  number   | The Unix epoch when the adapter received a successful commit event from the peer node `<P>`. Note, that transactions committed in the same block have nearly identical commit times, since the SDK receives them block-wise, i.e., at the same time.                         |
|     `event_hub_error_<P>`      |  string   | The error message in case some event hub connection-related error occurs with peer node `<P>`.                                                                                                                                                                               |

You can access these data in your workload module after calling `sendRequests`:

```js
let requestSettings = {
    contractId: 'marbles',
    contractVersion: '0.1.0',
    contractFunction: 'initMarble',
    contractArguments: ['MARBLE#1', 'Red', '100', 'Attila'],
    invokerIdentity: 'client0.org2.example.com',
    timeout: 10
};

// single argument, single return value
const result = await this.sutAdapter.sendRequests(requestSettings);

let shortID = result.GetID().substring(8);
let executionTime = result.GetTimeFinal() - result.GetTimeCreate();
console.log(`TX [${shortID}] took ${executionTime}ms to execute. Result: ${result.GetStatus()}`);
```

### The cleanupWorkloadModule function
The `cleanupWorkloadModule` function is called at the end of the round, and can be used to perform any resource cleanup required by your workload implementation.

## Network configuration file reference

The YAML network configuration file of the adapter mainly describes the organizations and the identities associated with those organizations, It also provides explicit information about the channels in your fabric network and the contracts (chaincode) deployed to those channels. It will reference Common Connection Profiles for each organization (as common connection profiles are specific to a single organization). These are the same connection profiles that would be consumed by the node-sdk. Whoever creates the fabric network and channels would be able to provide appropriate profiles for each organization.

The following sections detail each part separately. For a complete example, please refer to the [example section](#network-configuration-example) or one of the files in the [Caliper repository](https://github.com/hyperledger/caliper), such as the caliper-fabric test folder

<details><summary markdown="span">__name__
</summary>
_Required. Non-empty string._ <br>
The name of the configuration file.

```yaml
name: Fabric
```
</details>

<details><summary markdown="span">__version__
</summary>
_Required. Non-empty string._ <br>
Specifies the YAML schema version that the Fabric SDK will use. Only the `'2.0.0'` string should be specified.

```yaml
version: '2.0.0'
```
</details>

<details><summary markdown="span">__caliper__
</summary>
_Required. Non-empty object._ <br>
Contains runtime information for Caliper. Can contain the following keys.

*  <details><summary markdown="span">__blockchain__
   </summary>
   _Required. Non-empty string._ <br>
   Only the `"fabric"` string is allowed for this adapter.

   ```yaml
   caliper:
     blockchain: fabric
   ```
   </details>

*  <details><summary markdown="span">__sutOptions__
   </summary>
   _Optional. Non-empty object._ <br>
   These are sut specific options block, the following are specific to the fabric implementation

   *  <details><summary markdown="span">__mutualTls__
      </summary>
       _Optional. Boolean._ <br>
       Indicates whether to use client-side TLS in addition to server-side TLS. Cannot be set to `true` without using server-side TLS. Defaults to `false`.

       ```yaml
       caliper:
         blockchain: fabric
         sutOptions:
           mutualTls: true
       ```
      </details>
   </details>

*  <details><summary markdown="span">__command__
   </summary>
   _Optional. Non-empty object._ <br>
   Specifies the start and end scripts. <br>
   > Must contain __at least one__ of the following keys.

   *  <details><summary markdown="span">__start__
      </summary>
      _Optional. Non-empty string._ <br>
      Contains the command to execute at startup time. The current working directory for the commands is set to the workspace.

      ```yaml
      caliper:
        command:
          start: my-startup-script.sh
      ```
      </details>

   *  <details><summary markdown="span">__end__
      </summary>
      _Optional. Non-empty string._ <br>
      Contains the command to execute at exit time. The current working directory for the commands is set to the workspace.

      ```yaml
      caliper:
        command:
          end: my-cleanup-script.sh
      ```
      </details>
   </details>
</details>

<details><summary markdown="span">__info__
</summary>
_Optional. Object._ <br>
 Specifies custom key-value pairs that will be included as-is in the generated report. The key-value pairs have no influence on the runtime behavior.

```yaml
info:
  Version: 1.1.0
  Size: 2 Orgs with 2 Peers
  Orderer: Solo
  Distribution: Single Host
  StateDB: CouchDB
```
</details>

<details><summary markdown="span">__organizations__
</summary>
 _Required. Non-empty object._ <br>
Contains information about 1 or more organizations that will be used when running a workload. Even in a multi-organization fabric network, workloads would usually only be run from a single organization so it would be common to only see 1 organization defined. However it does support defining multiple organizations for which a workload can explicitly declare which organization to use. The first Organization in the network configuration will be the default organization if no explicit organization is requested.

```yaml
organizations:
  - mspid: Org1MSP
    identities:
      wallet:
        path: './org1wallet'
        adminNames:
        - admin
      certificates:
      - name: 'User1'
        clientPrivateKey:
          pem: |-
            -----BEGIN PRIVATE KEY-----
            ...
            -----END PRIVATE KEY-----
        clientSignedCert:
          pem: |-
            -----BEGIN CERTIFICATE-----
            ...
            -----END CERTIFICATE-----
    connectionProfile:
      path: './Org1ConnectionProfile.yaml'
      discover: true
  - mspid: Org2MSP
    connectionProfile:
      path: './Org2ConnectionProfile.yaml'
      discover: false
    identities:
      wallet:
        path: './org2wallet'
        adminNames:
        - admin

```

Each organization must have `mspid`, `connectionProfle` and `identities` provided and at least 1 cerficate or wallet definition in the identities section so that at least 1 identity is defined
*  <details><summary markdown="span">__mspid__
   </summary>
   _Required. Non-empty string._ <br>
   The unique MSP ID of the organization.

   ```yaml
   organizations:
     - mspid: Org1MSP
   ```
   </details>

*  <details><summary markdown="span">__connectionProfile__
   </summary>
   _Required. Non-empty object._ <br>
   Reference to a fabric network Common Connection Profile. These profiles are the same profiles that the fabric SDKs would consume in order to interact with a fabric network. A Common Connection Profile is organization specific so you need to ensure you point to a Common Connection Profile that is representive of the organization it is being included under. Connection Profiles also can be in 2 forms. A static connection profile will contain a complete description of the fabric network, ie all the peers and orderers as well as all the channels that the organization is part of. A dynamic connection profile will contain a minimal amount of information usually just a list of 1 or more peers belonging to the organization (or is allowed to access) in order to discover the fabric network nodes and channels.

   ```yaml
   organizations:
     - mspid: Org1MSP
     connectionProfile:
      path: './test/sample-configs/Org1ConnectionProfile.yaml'
      discover: true
   ```
   *  <details><summary markdown="span">__path__
      </summary>
      _Required. Non-empty string._ <br>
      The path to the connection profile file

      ```yaml
      organizations:
        - mspid: Org1MSP
        connectionProfile:
          path: './test/sample-configs/Org1ConnectionProfile.yaml'
      ```
      </details>
   *  <details><summary markdown="span">__discover__
      </summary>
      _Optional. Boolean._ <br>
      A value of `true` indicates that the connection profile is a dynamic connection profile and discovery should be used. If not specified then it defaults to false. You can only set this value to true if you plan to use the `gateway` option

      ```yaml
      organizations:
        - mspid: Org1MSP
        connectionProfile:
          path: './test/sample-configs/Org1ConnectionProfile.yaml'
          discover: true
      ```
      </details>
   </details>

*  <details><summary markdown="span">__identities__
   </summary>
   _Required. Non-empty object._ <br>
   Defines the location of 1 or more identities available for use. Currently only supports explicit identities by providing a certificate and private key as PEM or an SDK wallet that contains 1 or more identities on the file system. At least 1 identity must be provided via one of the child properties of identity.

   ```yaml
   identities:
      wallet:
        path: './wallets/org1wallet'
        adminNames:
        - admin
      certificates:
      - name: 'User1'
        clientPrivateKey:
          pem: |-
            -----BEGIN PRIVATE KEY-----
            ...
            -----END PRIVATE KEY-----
        clientSignedCert:
          pem: |-
            -----BEGIN CERTIFICATE-----
            ...
            -----END CERTIFICATE-----
    ```
    *  <details><summary markdown="span">certificates
       </summary>
       _Optional. A List of non-empty objects._ <br>
       Defines 1 or more identities by providing the PEM information for the client certificate and client private key as either an embedded PEM, a base64 encoded string of the PEM file contents or a path to individual PEM files

       ```yaml
       certificates:
       - name: 'User1'
         clientPrivateKey:
            path: path/to/privateKey.pem
         clientSignedCert:
            path: path/to/cert.pem
       - name: 'Admin'
         admin: true
         clientPrivateKey:
          pem: |-
            -----BEGIN PRIVATE KEY-----
            MIGHAgEAMBMGByqGSM49AgEGCCqGSM49AwEHBG0wawIBAQQgIRZo3SAPXAJnGVOe
            jRALBJ208m+ojeCYCkmJQV2aBqahRANCAARnoGOEw1k+MtjHH4y2rTxRjtOaKWXn
            FGpsALLXfBkKZvxIhbr+mPOFZVZ8ztihIsZBaCuCIHjw1Tx65szJADcO
            -----END PRIVATE KEY-----
         clientSignedCert:
          pem: |-
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
       - name: 'User3'
         clientPrivateKey:
          pem: LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS0tCk1JR0hBZ0VBTUJNR0J5cUdTTTQ5QWdFR0NDcUdTTTQ5QXdFSEJHMHdhd0lCQVFRZ0lSWm8zU0FQWEFKbkdWT2UKalJBTEJKMjA4bStvamVDWUNrbUpRVjJhQnFhaFJBTkNBQVJub0dPRXcxaytNdGpISDR5MnJUeFJqdE9hS1dYbgpGR3BzQUxMWGZCa0tadnhJaGJyK21QT0ZaVlo4enRpaElzWkJhQ3VDSUhqdzFUeDY1c3pKQURjTwotLS0tLUVORCBQUklWQVRFIEtFWS0tLS0tCg==
         clientSignedCert:
          pem: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUNXRENDQWY2Z0F3SUJBZ0lSQU1wU2dXRmpESE9vaFhhMFI2ZTlUSGd3Q2dZSUtvWkl6ajBFQXdJd2RqRUwKTUFrR0ExVUVCaE1DVlZNeEV6QVJCZ05WQkFnVENrTmhiR2xtYjNKdWFXRXhGakFVQmdOVkJBY1REVk5oYmlCRwpjbUZ1WTJselkyOHhHVEFYQmdOVkJBb1RFRzl5WnpFdVpYaGhiWEJzWlM1amIyMHhIekFkQmdOVkJBTVRGblJzCmMyTmhMbTl5WnpFdVpYaGhiWEJzWlM1amIyMHdIaGNOTWpBd09UQTNNVEUwTWpBd1doY05NekF3T1RBMU1URTAKTWpBd1dqQjJNUXN3Q1FZRFZRUUdFd0pWVXpFVE1CRUdBMVVFQ0JNS1EyRnNhV1p2Y201cFlURVdNQlFHQTFVRQpCeE1OVTJGdUlFWnlZVzVqYVhOamJ6RVpNQmNHQTFVRUNoTVFiM0puTVM1bGVHRnRjR3hsTG1OdmJURWZNQjBHCkExVUVBeE1XZEd4elkyRXViM0puTVM1bGVHRnRjR3hsTG1OdmJUQlpNQk1HQnlxR1NNNDlBZ0VHQ0NxR1NNNDkKQXdFSEEwSUFCTWRMdlNVRElqV1l1Qnc0WVZ2SkVXNmlmRkx5bU9BWDdHS1k2YnRWUERsa2RlSjh2WkVyWExNegpKV2ppdnIvTDVWMlluWnF2ME9XUE1NZlB2K3pIK1JHamJUQnJNQTRHQTFVZER3RUIvd1FFQXdJQnBqQWRCZ05WCiBIU1VFRmpBVUJnZ3JCZ0VGQlFjREFnWUlLd1lCQlFVSEF3RXdEd1lEVlIwVEFRSC9CQVV3QXdFQi96QXBCZ05WCkhRNEVJZ1FnNWZPaHl6d2FMS20zdDU0L0g0YjBhVGU3L25HUHlKWk5oOUlGUks2ZkRhQXdDZ1lJS29aSXpqMEUKQXdJRFNBQXdSUUloQUtFbnkvL0pZN0dYWi9USHNRSXZVVFltWHNqUC9iTFRJL1Z1TFg3VHpjZWZBaUJZb1N5WQp5OTByZHBySTZNcDZSUGlxalZmMDJQNVpDODZVa1AwVnc0cGZpUT09Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
       ```

       *  <details><summary markdown="span">__name__
          </summary>
          _Required. Non-empty string._ <br>
          Specifies a name to associate with this identity. This name doesn't have to match anything within the certificate itself but must be unique

          ```yaml
          certificates:
            - name: 'User1'
          ```

       *  <details><summary markdown="span">__admin__
          </summary>
          _Optional. Boolean._ <br>
          Indicates if this identity can be considered an admin identity for the organization. Defaults to false if not provided
          This only needs to be provided if you plan to create channels and/or install and instantiate contracts (chaincode)

          ```yaml
          certificates:
            - name: 'User2'
              admin: true
          ```

       *  <details><summary markdown="span">__clientPrivateKey__
          </summary>
          _Required. Non-empty object._ <br>
          Specifies the identity's private key for the organization.
          > Must contain __at most one__ of the following keys.

          *  <details><summary markdown="span">__path__
              </summary>
              _Optional. Non-empty string._ <br>
              The path of the file containing the private key.

              ```yaml
              clientPrivateKey:
                path: path/to/cert.pem
              ```
             </details>

          *  <details><summary markdown="span">__pem__
              </summary>
              _Optional. Non-empty string._ <br>
              The content of the private key file either in exact PEM format (which must split into multiple lines for yaml, or contain newline characters for JSON), or it could be a base 64 encoded version of the PEM (which will also encode the required newlines) as a single string. This single string format makes it much easier to embed into the network configuration file especially for a JSON based file

              ```yaml
              clientPrivateKey:
                pem: |
                  -----BEGIN PRIVATE KEY-----
                   MIGHAgEAMBMGByqGSM49AgEGCCqGSM49AwEHBG0wawIBAQQgIRZo3SAPXAJnGVOe
                   jRALBJ208m+ojeCYCkmJQV2aBqahRANCAARnoGOEw1k+MtjHH4y2rTxRjtOaKWXn
                   FGpsALLXfBkKZvxIhbr+mPOFZVZ8ztihIsZBaCuCIHjw1Tx65szJADcO
                   -----END PRIVATE KEY-----
              ```
             </details>
          </details>

       *  <details><summary markdown="span">__clientSignedCert__
          </summary>
          _Required. Non-empty object._ <br>
          Specifies the identity's certificate for the organization.
          > Must contain __at most one__ of the following keys.

          *  <details><summary markdown="span">__path__
             </summary>
             _Optional. Non-empty string._ <br>
             The path of the file containing the certificate.

             ```yaml
             clientSignedCert:
               path: path/to/cert.pem
             ```
             </details>

          *  <details><summary markdown="span">__pem__
             </summary>
             _Optional. Non-empty string._ <br>
              The content of the certificate file either in exact PEM format (which must split into multiple lines for yaml, or contain newline characters for JSON), or it could be a base 64 encoded version of the PEM (which will also encode the required newlines) as a single string. This single string format makes it much easier to embed into the network configuration file especially for a JSON based file

             ```yaml
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
          </details>
       </details>

    *  <details><summary markdown="span">wallet
       </summary>
       _Optional. Non-empty object_ <br>
       Provide the path to a file system wallet. Be aware that the persistence format used between v1.x and v2.x of the node sdks changed so make sure you provide a wallet created in the appropriate format for the version of SUT you bind to.

       *  <details><summary markdown="span">__path__
          </summary>
          _Required. Non-empty string._ <br>
          The path to the file system wallet

          ```yaml
          identities:
            wallet:
              path: './wallets/org1wallet'
          ```
          </details>

       *  <details><summary markdown="span">__adminNames__
          </summary>
          _Oprional. List of strings._ <br>
          1 or more names in the wallet that are identitied as organization administrators.
          This only needs to be provided if you plan to create channels and/or install and instantiate contracts (chaincode)

          ```yaml
          identities:
            wallet:
              path: './wallets/org1wallet'
              adminNames:
              - admin
              - another_admin
          ```
          </details>

       </details>
   </details>
</details>

<details><summary markdown="span">__channels__
</summary>
_Required. A list of objects._ <br>
Contains one or more unique channels with associated information about the contracts (chaincode) that will be available on the channel

```yaml
channels:
  - channelName: mychannel
    contracts:
    - id: marbles
      contractID: myMarbles

  - channelName: somechannel
    contracts:
    - id: basic
```

*  <details><summary markdown="span">__channelName__
   </summary>
   _Required. Non-empty String._ <br>
   The name of the channel.

   ```yaml
   channels:
     - channelName: mychannel
   ```

*  <details><summary markdown="span">__contracts__
   </summary>
   _Required. Non-sparse array of objects._ <br>
   Each array element contains information about a contract in the channel.

   > __Note:__ the `contractID` value of __every__ contract in __every__ channel must be unique on the configuration file level! If `contractID` is not specified for a contract then its default value is the `id` of the contract.

   ```yaml
   channels:
     mychannel:
       contracts:
       - id: simple
       - id: smallbank
   ```

   *  <details><summary markdown="span">__id__
      </summary>
      _Required. Non-empty string._ <br>
      The ID of the contract.

      ```yaml
      channels:
        mychannel:
          contracts:
          - id: simple
      ```
      </details>


   *  <details><summary markdown="span">__contractID__
      </summary>
      _Optional. Non-empty string._ <br>
      The Caliper-level unique ID of the contract. This ID will be referenced from the user callback modules. Can be an arbitrary name, it won't effect the contract properties on the Fabric side.

      If omitted, it defaults to the `id` property value.

      ```yaml
      channels:
        mychannel:
          contracts:
          - id: simple
          - contractID: simpleContract
      ```
      </details>
   </details>
</details>

## Network Configuration Example

The following example is a Fabric network configuration for the following network topology and artifacts:
* two organizations `Org1MSP` and `Org2MSP`;
* one channel named `mychannel`;
* `asset-transfer-basic` contract deployed to `mychannel`;
* the nodes of the network use TLS communication, but not mutual TLS;
* the fabric samples test network is started and terminated automatically by Caliper;

```yaml
name: Fabric
version: "2.0.0"

caliper:
  blockchain: fabric
  sutOptions:
    mutualTls: false
  command:
    start: ../fabric-samples/test-network/network.sh up createChannel && ../fabric-samples/test-network/network.sh deployCC -ccn basic -ccl javascript
    end: ../fabric-samples/test-network/network.sh down

info:
  Version: 1.1.0
  Size: 2 Orgs
  Orderer: Raft
  Distribution: Single Host
  StateDB: GoLevelDB

channels:
  - channelName: mychannel
    contracts:
    - id: basic
      contractID: BasicOnMyChannel

organizations:
  - mspid: Org1MSP
    identities:
      certificates:
      - name: 'admin.org1.example.com'
        admin: true
        clientPrivateKey:
          pem: |-
            -----BEGIN PRIVATE KEY-----
            ...
            -----END PRIVATE KEY-----
        clientSignedCert:
          pem: |-
            -----BEGIN CERTIFICATE-----
            ...
            -----END CERTIFICATE-----
    connectionProfile:
      path: './Org1ConnectionProfile.yaml'
      discover: true
  - mspid: Org2MSP
    connectionProfile:
    identities:
      certificates:
      - name: 'admin.org2.example.com'
        admin: true
        clientPrivateKey:
          pem: |-
            -----BEGIN PRIVATE KEY-----
            ...
            -----END PRIVATE KEY-----
        clientSignedCert:
          pem: |-
            -----BEGIN CERTIFICATE-----
            ...
            -----END CERTIFICATE-----
      path: './Org2ConnectionProfile.json'
      discover: true

```

## License

The Caliper codebase is released under the [Apache 2.0 license](./LICENSE.md). Any documentation developed by the Caliper Project is licensed under the Creative Commons Attribution 4.0 International License. You may obtain a copy of the license, titled CC-BY-4.0, at http://creativecommons.org/licenses/by/4.0/.