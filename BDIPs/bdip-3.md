---
bdip: 3
title: Wallet API
author: Nick Fujita <nick.fujita@bitcoin.com>
type: Standard Track
status: Draft
Created: 2018-11-18
---

=================

Table of Contents
* [Simple Summary](#simple-summary)
* [Abstract](#abstract)
* [Motivation](#motivation)
* [Specification](#specification)
  * [Methods](#methods)
    * [getAddress](#getAddress)
    * [sendAssets](#sendAssets)
    * [payInvoice](#payInvoice)
  * [Communication Protocol](#communication-protocol)
* [Rationale](#rationale)
* [References](#references)
* [Copyright](#copyright)

## Simple Summary
A standard interface for blockchain connected applications to interact with 3rd party user wallets.

## Abstract
There is currently no standardized interface that will allow applications to communicate with wallets. There is currently an existing interface that is implemented by the Badger wallet, and this proposal seeks to expand on those methods.

## Motivation
Currently, there is documentation for the existing version of interface methods with the Badger wallet, but it can be expanded further to accommodate additional wallet providers. The additional expanded interface will look to accommodate the existing Badger wallet interface under the hood, while also making it more flexible to allow for wallets that support multiple accounts and HD wallets.

## Specification

All methods on the wallet api will be async in order to accommodate transaction handling and user input from the wallet side.

### Methods

#### getAddress

Return an address for a specified protocol.

##### Method Interface

```
function getAddress(GetAccountInput): Promise<GetAccountOutput>
```


##### Input arguments
```
interface GetAccountInput {
  protocol: string; // BCH/SLP/BTC or any future protocol
}
```


##### Success return value
```
interface GetAccountOutput {
  address: string; // Address for the given protocol
  label?: string; // A label the users has set to identify their wallet
}
```

##### Error return value
```
interface Error {
  type: string; // `NO_PROVIDER`|`CONNECTION_DENIED`
  description: string;
  data: string;
}
```

##### Example
```
getAddress({
  protocol: 'BCH',
})
.then((data: GetAccountOutput) => {
  const {
    address,
    label,
  } = data;

  console.log('User address: ' + address);
  console.log('User address label (Optional): ' + label);
})
.catch(({type: string, description: string, data: any}) => {
  switch(type) {
    case NO_PROVIDER:
      console.log('No provider available.');
      break;
    case CANCELED:
      console.log('The user has canceled this request.');
      break;
    case PROTOCOL_ERROR:
      console.log('The provided protocol is not supported by this wallet.');
      break;
  }
});
```


##### Provider Request Handling

Depending on the account structure of the wallet, user handling may be different. In all cases, the wallet MUST return an address & protocol, or an error message. Each wallet may choose to provide back a default address in all cases, while others with multiple accounts and HD keys, may prompt a user to select an account from which to provide a new address.

Example
```
{
  address: 'bitcoincash:qrd9khmeg4nqag3h5gzu8vjt537pm7le85lcauzezc',
  label: 'My Spending Wallet'
}
```

#### sendAssets

Allows for a simple send of assets from user to specified address. The use case is where another application would like to request payment from the user wallet directly, without alerting any 3rd party server (merchant server).

##### Method Interface

```
function sendAssets(SendAssetsInput): Promise<SendAssetsOutput>
```

##### Input arguments

```
interface SendAssetsInput {
  to: string; // Address of the receiver of the assets to be sent
  protocol: string; // BCH/SLP/BTC or any future protocol
  assetId?: string; // Optional in the case of BCH or BTC. Required in the case of SLP, and will be token id
  value: string; // The amount of coins or assets to be sent, in Satoshis
}
```


##### Success return value

```
interface SendAssetsOutput {
  txid: string; // Transaction id of the sent assets
}
```


##### Error return value

```
interface Error {
  type: string; // `NO_PROVIDER`|`PROTOCOL_ERROR`|`SEND_ERROR`|`MALFORMED_INPUT`|`CANCELED`
  description: string;
  data: string;
}
```


##### Example

```
sendAssets({
  to: 'bitcoincash:qrd9khmeg4nqag3h5gzu8vjt537pm7le85lcauzezc',
  protocol: 'BCH',
  value: '1000',
})
.then((data: SendAssetsOutput) => {
  const {
    txid,
  } = data;

  console.log('Completed transaction id: ' + txid);
})
.catch(({type: string, description: string, data: any}) => {
  switch(type) {
    case NO_PROVIDER:
      console.log('No provider available.');
      break;
    case PROTOCOL_ERROR:
      console.log('The provided protocol is not supported by this wallet.');
      break;
    case SEND_ERROR:
      console.log('There was an error when broadcasting this transaction to the network.');
      break;
    case MALFORMED_INPUT:
      console.log('The input provided is not valid.');
      break;
    case CANCELED:
      console.log('The user has canceled this transaction request.');
      break;
  }
});
```


##### Provider Request Handling

- Validate the input parameters to make sure that the address provided is valid, and that it matches the provided protocol.
- Present the request to the user, and allow them to pick from which of their accounts they would like to send the funds (for some wallets they may only have a single account)
- If the user accepts the transaction, sign & broadcast the transaction, then return the transaction id to the app.
- If the user rejects the transaction, communicate back to the app that the user canceled the request.

Example
```
{
  txid: '31065a7ab53d5fa8e66ef1680e51c8485953c77e069293889b06d2b0b4934205',
}
```

#### payInvoice

Provides an interface for an application to request payment of a BIP70 invoice to a users wallet.

##### Method Interface

```
function payInvoice(PayInvoiceInput): Promise<PayInvoiceOutput>
```

##### Input arguments

```
interface PayInvoiceInput {
  url: string; // Url to retrieve the BIP70 payment request from the merchant server
}
```


##### Success return value

```
interface PayInvoiceOutput {
  memo: string; // Message from merchant server upon receiving payment
}
```


##### Error return value

```
interface Error {
  type: string; // `NO_PROVIDER`|`PROTOCOL_ERROR`|`MERCHANT_ERROR`|`MALFORMED_INPUT`|`CANCELED`
  description: string;
  data: string;
}
```

##### Example

```
payInvoice({
  url: 'bitcoincash:?r=https://bitpay.com/i/LHQmUTjzAcqX1NU47Nk1mJ',
})
.then((data: PayInvoiceOutput) => {
  const {
    memo,
  } = data;

  console.log('Payment processed memo from merchant server: ' + memo);
})
.catch(({type: string, description: string, data: any}) => {
  switch(type) {
    case NO_PROVIDER:
      console.log('No provider available.');
      break;
    case PROTOCOL_ERROR:
      console.log('The provided protocol is not supported by this wallet.');
      break;
    case MERCHANT_ERROR:
      console.log('There was an error when sending this transaction to the merchant.');
      break;
    case MALFORMED_INPUT:
      console.log('The input provided is not valid.');
      break;
    case CANCELED:
      console.log('The user has canceled this transaction request.');
      break;
  }
});
```

##### Provider Request Handling

- Fetch payment request from merchant server
- Present payment request information to user for approval
- Upon approval, submit transaction and send confirmation to merchant server
- Return payment confirmation memo from merchant server to connected application
- In the case there is an error in the

Example
```
{
  memo: 'Payment of 1 BTC for eleven tribbles accepted for processing.',
}
```

### Communication Protocol

The description and method interfaces above are Typscript psudocode assuming usage in an NPM package. However depending on the wallet, the communication protocol for these methods may differ.

#### JS methods

In the case of JS methods, all methods on the wallet API will be asynchronous, and accept an input object, and return a Promise where the argument will be output object.

#### Deep Link

For deep links can also be used as a communication medium for this wallet interface.

The general interface will be as follows:
```
<app_scheme>:<method>?<query_params>
```

Where app to app communication will have an additional `callback` query param which will provide a callback to the calling app, and web to app will have an additional `webhook` and `callback` parameters.

A more detailed example of what the communication flow would look like can be found [here](https://hackmd.io/@0GocXL2UTk-75Bt3XDppsA/BJObJntjQ?type=view#Web-App-to-Wallet-App-Communication)

## Rationale

## References

## Copyright
