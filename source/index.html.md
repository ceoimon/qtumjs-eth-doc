---
title: QtumJS-Eth API Reference

language_tabs:
  - typescript
  - javascript

search: true
---

# Introduction

> To install qtumjs-eth

```
npm install qtumjs-eth
```

QtumJS-Eth is a JavaScript library for developing DApp on the Ethereum blockchain. You can use this library to develop frontend UI that runs in the browser, as well as backend server scripts that run in NodeJS.

The main classes are:

Class | Description
--------- | -----------
RPCRaw | Direct access to blockchain RPC service.
[EthRPC](#ethrpc) | Wrapper for `RPCRaw`, to provide some [Ethereum JSON RPC API](https://github.com/ethereum/wiki/wiki/JSON-RPC) wrapper.
[Contract](#contract) | An abstraction for interacting with smart contracts. Handles [ABI encoding/decoding](https://github.com/ethereum/wiki/wiki/Ethereum-Contract-ABI).

QtumJS-Eth is developed using [TypeScript](https://www.typescriptlang.org/), and as such, comes with robust type definitions for all the APIs. We recommend using [VSCode](https://code.visualstudio.com/) to take advantage of language support, such as type hinting and autocompletion.

But you can also choose to use plain JavaScript and notepad if you prefer.

This document is the reference for QtumJS-Eth API, and its basic uses. 

# ERC20 Example

```ts
import {
  Ethereum,
} from "qtumjs-eth"

const repoData = require("./solar.json")
const ethereum = new Ethereum("http://localhost:8545", repoData)

const myToken = ethereum.contract("CappedToken")

async function transfer(fromAddr, toAddr, amount) {
  const tx = await myToken.send("transfer", [toAddr, amount], {
    from: fromAddr
  })

  console.log("transfer tx:", tx.txid)
  console.log(tx)

  await tx.confirm(3) 
  console.log("transfer confirmed")
}
```

Assuming that `solar.json` contains information about your deployed contracts,
you can use qtumjs to call the token contract's method to transfer tokens.

An example [solar.json](https://github.com/ceoimon/qtumbook-mytoken-qtumjs-eth-cli/blob/master/solar.development.json.example). This can be generated automatically using the [solar](https://github.com/qtumproject/solar) deployment tool.

The complete example: [ceoimon/qtumbook-mytoken-qtumjs-eth-cli](https://github.com/ceoimon/qtumbook-mytoken-qtumjs-eth-cli)

For contract deployment, see [Solar Smart Contract Deployment Tool](https://github.com/qtumproject/solar).

# Ethereum

```ts
const repoData = require("./solar.json")
const ethereum = new Ethereum("http://localhost:8545", repoData, '0x90f8bf6a479f320ead074411a4b0e7944ea8c9c1')
```

The `Ethereum` class is an instance of the `qtumjs-eth` API. It provides two main features:

+ Access to the RPC service. It is a subclass of [EthRPC](#ethrpc).
+ A factory method to instantiate [Contract](#contract-2) instances, for interacting with deployed contracts.

Arg | Type
--------- | -----------
url | string
  | URL of the RPC service
repoData | [IContractsRepoData](#icontractsrepodata)
  | Information about Solidity contracts.
sender | string
  | *optional* Default ETH sender (default: `$first_account_of_eth_node`)

The `repoData` contains the ABI definitions of all the deployed contracts and libraries, as well as deploy addresses. This information is used to instantiate `Contract` instances.

`Contract` instantiated with `Ethereum`'s factory method is able to decode all event types found in `repoData`. Whereas a `Contract` constructed manually is only able to decode event types defined in its scope, a limitation due to how the Solidity compiler output ABI definitions.

It is recommended that you use `Ethereum` to instantiate `Contract` instances.

## contract

```ts
const myToken = ethereum.contract("CappedToken")
```

> This instantiates the Contract using information [here](https://github.com/ceoimon/qtumbook-mytoken-qtumjs-eth-cli/blob/master/solar.development.json.example#L3).

A factory method to instantiate a `Contract` instance using the ABI definitions and address found in `repoData`. The Contract instance is configured with an event log decoder that can decode all known event types found in `repoData`.

Arg | Type
--------- | -----------
name | string
  | Used as key into the `repoData.contracts` map to get contract information.
@return | [Contract](#contract)
  | The Contract instance corresponding to the specific key.

# Contract

A class abstraction for interacting with a Smart Contract.

This is a more convenient API than using `EthRPC` to directly call the RPC's `sendcontract` and `calltocontract` methods. It handles ABI encoding, to convert between JS and Solidity values.

* API for confirming transactions.
* API for invoking contract's methods using `call` or `send` .
* API for getting contract's log events.

## constructor

```js
const rpc = new EthRPC("http://localhost:8545")

const myToken = new Contract(rpc, repo.contracts.CappedToken)
```

> The contract [info](https://github.com/ceoimon/qtumbook-mytoken-qtumjs-eth-cli/blob/master/solar.development.json.example#L3) may be generated by [solar](https://github.com/qtumproject/solar).

Arg | Type | Description
--------- | ----------- | -----------
rpc | EthRPC | The RPC instance used to interact with the contract.
info | [IContractInfo](#icontractinfo) | Information for the deployed contract

It is recommended that you use [Ethereum#contract](#contract) instead of this constructor.

## call

```js
async function totalSupply() {
  const result = await myToken.call("totalSupply")

  // supply is a BigNumber instance (see: bn.js)
  const supply = result[0]

  console.log("supply", supply.toNumber())
}
```

Executes contract method on your own local testrpc node as a "simulation" using `callcontract`. It is free, and does not actually modify the blockchain.

Arg | Type
--------- | -----------
method | string
  | Name of the contract method.
args | any[]
  | Arguments for calling the method
opts | [IContractCallRequestOptions](#icontractcallrequestoptions)
  | *optional* call options
@return | Promise\<any[]>
  | call result, ABI decoded

## send

```ts
async function mint(toAddr, amount) {
  // Submit a `sendtocontract` transaction, invoking the `mint` method.
  const tx = await myToken.send("mint", [toAddr, amount])

  console.log("tx:\n", tx)

  // Wait for 3 confirmations. The callback receives the
  // updated transaction info for each additional confirmation.
  //
  // Both arguments are optional. `await tx.confirm()` would do.
  const receipt = await tx.confirm(3, (updatedTx) => {
    console.log("new confirmation", updatedTx.txid)
  })
  console.log("tx receipt:\n", JSON.stringify(receipt, null, 2))
}
```

> Example output:

```js
tx:
{ hash: '0xc215218a28d0b9ff83e5b8d0d75078e1d25afb32363b6c6b2beb44b009940d4e',
  nonce: '0x01',
  blockHash: '0x4a591600314e15fc01911c4d04500ec262b65342774dcef088c10eadc52dc981',
  blockNumber: '0x02',
  transactionIndex: '0x0',
  from: '0x90f8bf6a479f320ead074411a4b0e7944ea8c9c1',
  to: '0xe78a0f7e598cc8b0bb87894b0f60dd2a88d6a8ab',
  value: '0x0',
  gas: '0x030d40',
  gasPrice: '0x04a817c800',
  input: '0x40c10f19000000000000000000000000a9639ccae212729afce8717099020a8d07a9d87f00000000000000000000000000000000000000000000000000000000000003e8',
  txid: '0xc215218a28d0b9ff83e5b8d0d75078e1d25afb32363b6c6b2beb44b009940d4e',
  method: 'mint',
  confirm: [Function: confirm] }
```

> The callback would print 3 times, for each confirmation:

```
new confirmation 0xc215218a28d0b9ff83e5b8d0d75078e1d25afb32363b6c6b2beb44b009940d4e
new confirmation 0xc215218a28d0b9ff83e5b8d0d75078e1d25afb32363b6c6b2beb44b009940d4e
new confirmation 0xc215218a28d0b9ff83e5b8d0d75078e1d25afb32363b6c6b2beb44b009940d4e
```

> The returned transaction receipt after confirmation:

```json
tx receipt:
{
  "transactionHash": "0xc215218a28d0b9ff83e5b8d0d75078e1d25afb32363b6c6b2beb44b009940d4e",
  "transactionIndex": "0x0",
  "blockHash": "0x4a591600314e15fc01911c4d04500ec262b65342774dcef088c10eadc52dc981",
  "blockNumber": "0x2",
  "gasUsed": "0x10c1c",
  "cumulativeGasUsed": "0x10c1c",
  "contractAddress": null,
  "status": "0x1",
  "logsBloom": "0x00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000200000000020000000000008000000000000000000000000000000000000000000000000020000000000000000000800000000002000400000000010100000000000000000000000001000000000000000000000000000000000100000000000000080000000000000000000000000000000400000000000000000000000000000000002000000000000000000000000000000000000000000000000000020000000000000000000000000000000000000000000000000000000000000000000",
  "from": "0x90f8bf6a479f320ead074411a4b0e7944ea8c9c1",
  "to": "0xe78a0f7e598cc8b0bb87894b0f60dd2a88d6a8ab",
  "logs": [
    {
      "0": "3e8",
      "amount": "3e8",
      "to": "0xa9639ccae212729afce8717099020a8d07a9d87f",
      "_eventName": "Mint"
    },
    {
      "0": "3e8",
      "value": "3e8",
      "from": "0x0000000000000000000000000000000000000000",
      "to": "0xa9639ccae212729afce8717099020a8d07a9d87f",
      "_eventName": "Transfer"
    }
  ],
  "rawlogs": [
    {
      "logIndex": "0x0",
      "transactionIndex": "0x0",
      "transactionHash": "0xc215218a28d0b9ff83e5b8d0d75078e1d25afb32363b6c6b2beb44b009940d4e",
      "blockHash": "0x4a591600314e15fc01911c4d04500ec262b65342774dcef088c10eadc52dc981",
      "blockNumber": "0x2",
      "address": "0xe78a0f7e598cc8b0bb87894b0f60dd2a88d6a8ab",
      "data": "0x00000000000000000000000000000000000000000000000000000000000003e8",
      "topics": [
        "0x0f6798a560793a54c3bcfe86a93cde1e73087d944c0ea20544137d4121396885",
        "0x000000000000000000000000a9639ccae212729afce8717099020a8d07a9d87f"
      ],
      "type": "mined"
    },
    {
      "logIndex": "0x1",
      "transactionIndex": "0x0",
      "transactionHash": "0xc215218a28d0b9ff83e5b8d0d75078e1d25afb32363b6c6b2beb44b009940d4e",
      "blockHash": "0x4a591600314e15fc01911c4d04500ec262b65342774dcef088c10eadc52dc981",
      "blockNumber": "0x2",
      "address": "0xe78a0f7e598cc8b0bb87894b0f60dd2a88d6a8ab",
      "data": "0x00000000000000000000000000000000000000000000000000000000000003e8",
      "topics": [
        "0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef",
        "0x0000000000000000000000000000000000000000000000000000000000000000",
        "0x000000000000000000000000a9639ccae212729afce8717099020a8d07a9d87f"
      ],
      "type": "mined"
    }
  ]
}
```

Creates a transaction that executes contract method globally on the network, changing the blockchain.

This costs gas.

There are two asynchronous steps to a transaction:

1. You submit the the transaction to the network.
2. Once submitted, wait for a required number of confirmations.

After successful confirmation, the transaction receipt ([ITransactionReceipt](#itransactionreceipt)) with ABI decoded event logs is returned.

Arg | Type
--------- | -----------
method | string
  | Name of the contract method.
args | any[]
  | Arguments for calling the method
opts | [IContractSendRequestOptions](#icontractsendrequestoptions)
  | *optional* send options
@return | Promise\<[IContractSendResult](#icontractsendresult)>
  | call result, with ABI decoded outputs

## Method Overloading

If there is no ambiguity, use the method name to call/send a method. If the same method name has multiple definitions, use the method signature to call/send a method.

> The name foo may have multiple method definitions:

```ts
function foo();
function foo(int256 _a);
function foo(uint256 _a, uint256 _b);
function foo(int256 _a, int256 _b);
```

> `foo` methods with arity 0 and arity 1 have no ambiguity. Can call directly.

```ts
contract.call("foo")
contract.call("foo", [1])
```

> `foo` methods with arity 2 are ambiguous, must call with full method signature:

```ts
contract.call("foo(uint256,uint256)", [1, 2])
contract.call("foo(int256,int256)", [1, 2])
```

## getlogs

```js
async function getLogs(fromBlock=0, toBlock="latest") {
  const logs = await myToken.getlogs({
    fromBlock,
    toBlock
  })

  console.log(JSON.stringify(logs, null, 2))
}
```

> Example Output

```json
[
  {
    "logIndex": "0x00",
    "transactionIndex": "0x00",
    "transactionHash": "0xd7f0ba07750626ffd82fa8ad909e2657bc3b181a7896e69212c55f8f83ae89f7",
    "blockHash": "0xdf328377bbbe15edb59928bb0507a9f7e252e973bcd75decd579e94ee6d9e4af",
    "blockNumber": "0x02",
    "address": "0xe78a0f7e598cc8b0bb87894b0f60dd2a88d6a8ab",
    "data": "0x00000000000000000000000000000000000000000000000000000000000003e8",
    "topics": [
      "0x0f6798a560793a54c3bcfe86a93cde1e73087d944c0ea20544137d4121396885",
      "0x00000000000000000000000090f8bf6a479f320ead074411a4b0e7944ea8c9c1"
    ],
    "type": "mined",
    "event": {
      "0": "3e8",
      "amount": "3e8",
      "to": "0x90f8bf6a479f320ead074411a4b0e7944ea8c9c1",
      "_eventName": "Mint"
    }
  },
  {
    "logIndex": "0x01",
    "transactionIndex": "0x00",
    "transactionHash": "0xd7f0ba07750626ffd82fa8ad909e2657bc3b181a7896e69212c55f8f83ae89f7",
    "blockHash": "0xdf328377bbbe15edb59928bb0507a9f7e252e973bcd75decd579e94ee6d9e4af",
    "blockNumber": "0x02",
    "address": "0xe78a0f7e598cc8b0bb87894b0f60dd2a88d6a8ab",
    "data": "0x00000000000000000000000000000000000000000000000000000000000003e8",
    "topics": [
      "0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x00000000000000000000000090f8bf6a479f320ead074411a4b0e7944ea8c9c1"
    ],
    "type": "mined",
    "event": {
      "0": "3e8",
      "value": "3e8",
      "from": "0x0000000000000000000000000000000000000000",
      "to": "0x90f8bf6a479f320ead074411a4b0e7944ea8c9c1",
      "_eventName": "Transfer"
    }
  },
  {
    "logIndex": "0x00",
    "transactionIndex": "0x00",
    "transactionHash": "0x0e96facc85351227e5fc68c3688713dd6fe6385a7cac4a63cb6c0f9ca5be0a92",
    "blockHash": "0xabc83d54866df78b9014048df29589ed6d3ab36ea851594dca79b88b67ecb1b2",
    "blockNumber": "0x03",
    "address": "0xe78a0f7e598cc8b0bb87894b0f60dd2a88d6a8ab",
    "data": "0x0000000000000000000000000000000000000000000000000000000000000064",
    "topics": [
      "0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef",
      "0x00000000000000000000000090f8bf6a479f320ead074411a4b0e7944ea8c9c1",
      "0x000000000000000000000000ffcf8fdee72ac11b5c542428b35eef5769c409f0"
    ],
    "type": "mined",
    "event": {
      "0": "64",
      "value": "64",
      "from": "0x90f8bf6a479f320ead074411a4b0e7944ea8c9c1",
      "to": "0xffcf8fdee72ac11b5c542428b35eef5769c409f0",
      "_eventName": "Transfer"
    }
  }
]
```

Get [Solidity event logs](http://solidity.readthedocs.io/en/develop/abi-spec.html#events) generated by the contract.

The options can limit events log query to a block number range by specifying `fromBlock` and `toBlock`. For example, you could query for event logs between block 1000 to 1500.

Arg | Type
--------- | -----------
opts | [IGetLogsRequest](#igetlogsrequest)
  | *optional* Event logs query parameters
@return | Promise\<[IContractEventLog](#icontracteventlog)[]>
  | Log query result, with ABI decoded outputs

## onLog

```js
myToken.onLog((entry) => {
  console.log(entry)
})
```

Subscribe to contract's new events. The callback is invoked each time a new event is received. By default, `onLog` start listening for logs from the tip of the blockchain. Use `fromBlock` to also receive older events.

Note: it will include the latest block's logs if `fromBlock` is not specified or set to `latest`.

Arg | Type
--------- | -----------
callback | (entry: [IContractEventLog](#icontracteventlog)) => void
opts | [IGetLogsRequest](#igetlogsrequest)
  | *optional* Event logs query parameters
@return | () => void
  | cancel the listener

## logEmitter

```js
this.emitter = myToken.logEmitter({ minconf: 1 })

this.emitter.on("Mint", (event) => {
  // ...
})

this.emitter.on("Transfer", (event) => {
  // ...
})

this.emitter.on("?", (event) => {
  // all un-decodeable events
})
```

Subscribe to contract's new events, using the [EventsEmitter](https://github.com/primus/eventemitter3) interface. The events emitted are instances of [IContractEventLog](#icontracteventlog)

The Solidity events names are used as the emitted event names.

Events that lack ABI definitions (thus cannot be parsed) are emitted as "?".

Arg | Type
--------- | -----------
opts | [IGetLogsRequest](#igetlogsrequest)
  | *optional* Event logs query parameters

## receipt

```ts
const txid = "0xd7f0ba07750626ffd82fa8ad909e2657bc3b181a7896e6"
const receipt = await qrcToken.receipt(txid)
if (receipt) {
  console.log(JSON.stringify(receipt, null, 2))
}
```

Get the receipt for a transaction that had been accepted by the network. If the transaction had not been confirmed, null is returned.

The event logs for that transaction are ABI-encoded.

Arg | Type
--------- | -----------
txid | string
  | Transaction ID
@return | Promise\<[ITransactionReceipt](#itransactionreceipt) &#124; `null`>
  | Transaction receipt, with event logs. return `null` if transaction is not yet confirmed.

# EthRPC

```ts
const rpc = new EthRPC('http://localhost:8545', '0x90f8bf6a479f320ead074411a4b0e7944ea8c9c1');
```

This is a JSON-RPC client for direct access to the Ethereum RPC API. It does not handle any ABI-encoding or decoding for you.

Arg | Type
--------- | -----------
url | string
  | URL of the RPC service
sender | string
  | *optional* Default ETH sender (default: `$first_account_of_eth_node`)

## rawCall

Makes a JSON-RPC method call, and return the result. This method throws an error if the JSON API returns a non-200 HTTP result.

Arg | Type
--------- | -----------
method | string
  | RPC API method name
params | any[]
  | arguments for api
opts | [IRPCCallOption](#irpccalloption)
  | *optional* extra options

> Using `try...catch` to handle error:

```ts
async function main() {
  try {
    const result = await rpc.rawCall("unknown-method-hohoho")
  } catch (err) {
    console.log("err", err)
  }
}
```

## getSender

Arg | Type
--------- | -----------
@return | Promise\<string>
  | default ETH sender

Get the default ETH sender, return the `sender` parameter if provided in constructor calling.

## getGasPrice

Arg | Type
--------- | -----------
@return | Promise\<string>
  | current network's gas price

Get current network's gas price.

## getBlockNumber

Arg | Type
--------- | -----------
@return | Promise\<number>
  | latest block number

Get the latest block number.

## getAccounts

Arg | Type
--------- | -----------
@return | Promise\<string[]>
  | accounts

Get accounts.

## getNetId

Arg | Type
--------- | -----------
@return | Promise\<string>
  | [net id](https://github.com/ethereum/wiki/wiki/JSON-RPC#net_version)

Get the net id (aka net version).

## getTransactionCount

Arg | Type
--------- | -----------
address | string
  | address to query
block | [typeBlockTags](#typeblocktags)
  | block to query
@return | Promise\<number>
  | transaction count

Get account's transaction count.

## getBalance

Arg | Type
--------- | -----------
address | string
  | address to query
block | [typeBlockTags](#typeblocktags)
  | block to query
@return | Promise\<string>
  | balance

Get ETH balance

## 

# Types Lexicon

## IContractInfo

```ts
export interface IContractInfo {
  /**
   * Contract's ABI definitions, produced by solc.
   */
  abi: IABIMethod[]

  /**
   * Contract's address
   */
  address: string

  /**
   * The owner address of the contract
   */
  sender?: string
}
```

The minimal deployment information necessary to interact with a deployed contract.

## IContractCallRequestOptions

Options for [Contract#call](#call)

```ts
/**
 * Options for `call` to a contract method.
 */
export interface IContractCallRequestOptions {
  /**
   * The quantum/ethereum address that will be used as sender.
   */
  from?: string

  gasLimit?: string | number
  gasPrice?: string | number
  value?: string | number
  blockNumber?: typeBlockTags
}
```

### References

+ [typeBlockTags](#typeblocktags)

## IContractSendRequestOptions

Options for [Contract#send](#send)

```ts
/**
 * Options for `send` to a contract method.
 */
export interface IContractSendRequestOptions {
  /**
   * The amount in Ether to send. eg 0.1, default: 0
   */
  value?: number | string

  /**
   * gasLimit, default: 200000
   */
  gasLimit?: number | string

  /**
   * gasPrice
   */
  gasPrice?: number | string

  /**
   * The ethereum address that will be used as sender.
   */
  from?: string

  nonce?: number | string
}
```

## IContractSendResult

```ts
export interface IContractSendResult extends IGetTransactionResult {
  /**
   * Name of contract method invoked.
   */
  method: string

  /**
   * Wait for transaction confirmations.
   */
  confirm: IContractSendConfirmFunction

  txid: string
}
```

Return value of [Contract#send](#send).

### References

+ [IGetTransactionResult](#igettransactionresult)
+ [IContractSendConfirmFunction](#icontractsendconfirmfunction)

## IContractSendConfirmFunction

Used to wait for transaction confirmations.

Arg | Type
--------- | -----------
n | number
  | *optional* Number of confirmations to wait for. (default: 3)
callback | [IContractSendConfirmationHandler](#icontractsendconfirmationhandler)
  | *optional* The callback function invoked for each additional confirmation
@return | [ITransactionReceipt](#itransactionreceipt)
  | The transaction receipt after specific confirmations

## IContractSendConfirmationHandler

The callback for [`IContractSendConfirmFunction`](#icontractsendconfirmfuncction)

Arg | Type
--------- | -----------
updatedTx | [IGetTransactionResult](#igettransactionresult)
  | Basic information about a transaction submitted to the network.
receipt | [ITransactionReceipt](#itransactionreceipt)
  | Additional information about a confirmed transaction.

## IGetTransactionResult

```ts
export interface IGetTransactionResult {
  hash: string
  nonce: string
  from: string
  to: string
  value: string
  gas: string
  gasPrice: string
  input: string
  blockHash?: string
  blockNumber?: string
  transactionIndex?: string
}
```

Basic information about a transaction submitted to the network.

## ITransactionReceipt

The transaction receipt for [Contract#send](#send), with the event logs decoded.

```ts
export interface ITransactionReceipt extends IGetTransactionReceiptBase {
  /**
   * logs decoded using ABI
   */
  logs: IParsedLog[],

  /**
   * undecoded logs
   */
  rawlogs: ITransactionLog[],
}
```

### References

* [IGetTransactionReceiptBase](#igettransactionreceiptbase)
* [IParsedLog](#iparsedlog)
* [ITransactionLog](#itransactionlog)

## IParsedLog

```ts
/**
 * A decoded Solidity event log
 */
export interface IParsedLog {
  _eventName: string
  [key: string]: any
}
```

## ITransactionLog

```ts
export interface ITransactionLog {
  address: string
  topics: string[]
  data: string
}
```

## IGetLogsRequest

```ts
export interface IGetLogsRequest {
  /**
   * The block number to start looking for logs.
   */
  fromBlock?: typeBlockTags

  /**
   * The block number to stop looking for logs.
   */
  toBlock?: typeBlockTags

  /**
   * contract address
   */
  address?: string[] | string

  /**
   * filter topics
   */
  topics?: Array<string | null>
}
```

### References

+ [typeBlockTags](#typeblocktags)

## IContractEventLogs

```ts
/**
 * Query result of a contract's event logs.
 */
export interface IContractEventLogs {
  /**
   * Event logs, ABI decoded.
   */
  entries: IContractEventLog[]

  /**
   * Number of event logs returned.
   */
  count: number

  /**
   * The block number to start query for new event logs.
   */
  nextblock: number
}
```

Query result of a contract's event logs.

To query for new logs that have not yet been seen, use `nextblock` as `startBlock` when querying for event logs.

* [IContractEventLog](#icontracteventlog)

## IContractEventLog

A decoded contract event log.

```ts
export interface IContractLogEntry extends ILogEntry {
  /**
   * Solidity event, ABI decoded. Null if no ABI definition is found.
   */
  event?: IParsedLog | null
}
```

### References

+ [ILogEntry](#ilogentry)
+ [IParsedLog](#iparsedlog)

## ILogEntry

```ts
/**
 * The raw log data returned by rpc, not ABI decoded.
 */
export interface ILogEntry {
  /**
   *  `true` when the log was removed, due to a chain reorganization. `false` if
   * its a valid log.
   */
  removed: boolean

  /**
   * integer of the log index position in the block. `null` when its pending.
   */
  logIndex: string | null

  /**
   * integer of the transactions index position log was created from. `null`
   * when its pending
   */
  transactionIndex: string | null

  /**
   * hash of the transactions this log was created from. `null` when its pending
   */
  transactionHash: string | null

  /**
   * hash of the block where this log was in. `null` when its pending.
   */
  blockHash: string | null

  /**
   * the block number where this log was in. `null` when its pending
   */
  blockNumber: string | null

  /**
   * address from which this log originated
   */
  address: string

  /**
   * contains one or more 32 Bytes non-indexed arguments of the log.
   */
  data: string

  /**
   * Array of 0 to 4 32 Bytes data of indexed log arguments. (In solidity: The
   * first topic is the hash of the signature of the event, except you declared
   * the event with the `anonymous` specifier
   */
  topics: string[]
}
```

## IGetTransactionReceiptBase

Receipt for a transaction accepted by the network. It is returned by the `eth_getTransactionReceipt` RPC call.

```ts
export interface IGetTransactionReceiptBase {
  blockHash: string
  blockNumber: string

  transactionHash: string
  transactionIndex: string

  from: string
  to: string

  cumulativeGasUsed: string
  gasUsed: string

  contractAddress: string | null
  logsBloom: string
  status?: TRANSACTION_STATUS // number in js, 0 is success, 1 is failed
}
```
### References

+ [ITransactionReceipt](#itransactionreceipt)

## IContractsRepoData

Information about contracts

```ts
export interface IContractsRepoData {
  /**
   * Information about deployed contracts
   */
  contracts: {
    [key: string]: IContractInfo,
  },

  /**
   * Information about deployed libraries
   */
  libraries: {
    [key: string]: IContractInfo,
  }

  /**
   * Information of contracts referenced by deployed contract/libraries, but not deployed
   */
  related: {
    [key: string]: {
      abi: IABIMethod[],
    },
  }
}

/**
 * The minimal deployment information necessary to interact with a
 * deployed contract.
 */
export interface IContractInfo {
  /**
   * Contract's ABI definitions, produced by solc.
   */
  abi: IABIMethod[]

  /**
   * Contract's address
   */
  address: string

  /**
   * The owner address of the contract
   */
  sender?: string
}

export interface IABIMethod {
  name: string,
  type: string,
  payable: boolean,
  inputs: IABIInput[],
  outputs: IABIOutput[],
  constant: boolean,
  anonymous: boolean,
}
```

This can be generated automatically using the [solar](https://github.com/qtumproject/solar) deployment tool.

An example [solar.json](https://github.com/ceoimon/qtumbook-mytoken-qtumjs-eth-cli/blob/master/solar.development.json.example).

## IRPCCallOption

```ts
export interface IRPCCallOption {
  cancelToken?: CancelToken
}
```

You can provider a [Axios `CancelToken`](https://github.com/axios/axios#cancellation) to cancel a http request.

## TRANSACTION_STATUS

```ts
export enum TRANSACTION_STATUS {
  FAILED,
  SUCCESS
}
```

## typeBlockTags

```ts
export type typeBlockTags = number | "latest" | "pending" | "earliest"
```
