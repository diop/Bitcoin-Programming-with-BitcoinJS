# Coinjoin Transaction \(4 inputs, 4 outputs\) - Legacy P2PKH

> To follow along this tutorial
>
> * Execute all the transaction code in one go by typing `node code/filename.js`   
> * Or enter the commands step-by-step by `cd` into `./code` then type `node` in a terminal to open the Node.js REPL   
> * Open the Bitcoin Core GUI console or use `bitcoin-cli` for the Bitcoin Core commands
> * Use `bx` aka `Libbitcoin-explorer` as a handy complement

## What is a Coinjoin

The signatures, one per input, inside a transaction are completely independent of each other. This means that it's possible for Bitcoin users to agree on a set of inputs to spend, and a set of outputs to pay to, and then to individually and separately sign a transaction and later merge their signatures. The transaction is not valid and won't be accepted by the network until all signatures are provided, and no one will sign a transaction which is not to their liking.

To use this to increase privacy, the N users would agree on a uniform output size and provide inputs amounting to at least that size. The transaction would have N outputs of that size and potentially N more change outputs if some of the users provided input in excess of the target. All would sign the transaction, and then the transaction could be transmitted. No risk of theft at any point.

Consider the following transactions made at the same time: A purchases an item from B, C purchases an item from D, and E purchases an item from F. Without Coinjoin, the public blockchain ledger would record three separate transactions for each input-output match. With Coinjoin, only one single transaction is recorded. The ledger would show that bitcoins were paid from A, C, and E addresses to B, D, and F. By masking the deals made by all parties, an observer can’t, with full certainty, determine who sent bitcoins to whom.

> To read more about Coinjoin
>
> * [Bitcoin Developer Guide - Coinjoin](https://bitcoin.org/en/developer-guide#coinjoin)
> * Top-notch fungibility framework [ZeroLink](https://github.com/nopara73/ZeroLink)
> * The current most advanced Coinjoin wallet [Wasabi Wallet](https://www.wasabiwallet.io)

## What we will do

Let's create a Coinjoin transaction mixing four different transactions.

We will do the following:

|   Inputs   | -\> |   Outputs   |
|------------|-----|-------------|
| Alice\_1   | -\> |   Bob\_1    |
| Carol\_1   | -\> |  Dave\_1    |
| Eve\_1     | -\> |  Mallory\_2 |
| Mallory\_1 | -\> |  Alice\_2   |

The four signers agree on a uniform output amount of 0.2 BTC.

We will create four UTXOs to spend from, with different amounts but with at least 0.2 BTC. If someone spend a UTXO that is more than 0.2 BTC, we will create a additional output for his change.

## Create UTXOs to spend from

Import libraries, test wallets and set the network
```javascript
const bitcoin = require('bitcoinjs-lib')
const { alice, bob, carol, dave, eve, mallory } = require('./wallets.json')
const network = bitcoin.networks.regtest
```

First let's create four UTXOs with a single transaction.
> Send 0.2 BTC to alice\_1 P2PKH address.  
> Send 0.2 BTC to carol\_1 P2PKH address.  
> Send 0.25 BTC to eve\_1 P2PKH address.  
> Send 0.3 BTC to mallory\_1 P2PKH address.  
```shell
sendmany "" '{"n4SvybJicv79X1Uc4o3fYXWGwXadA53FSq":0.2, "mh1HAVWhKkzcvF41MNRKfakVvPV2sfaf3R":0.2, "mqbYBESF4bib4VTmsqe6twxMDKtVpeeJpt":0.25, "mwkyEhauZdkziHJijx9rwZgShr9gYi9Hkh": 0.3}' 
```
&nbsp;

Generate a block to dave\_1 P2WPKH address.
```shell
generatetoaddress 1 bcrt1qnqud2pjfpkqrnfzxy4kp5g98r8v886wgvs9e7r
```
&nbsp;

Then we need to know which UTXO corresponds to which address Get the output indexes \(vout\) of the transaction, so that we have the four outpoints \(txid / vout\).
> Find the output index \(or vout\) under `details > vout`.
```shell
gettransaction TX_ID
```
&nbsp;

We can also check UTXOs of specific addresses.
```shell
scantxoutset start '["addr(n4SvybJicv79X1Uc4o3fYXWGwXadA53FSq)", "addr(mh1HAVWhKkzcvF41MNRKfakVvPV2sfaf3R)", "addr(mqbYBESF4bib4VTmsqe6twxMDKtVpeeJpt)", "addr(mwkyEhauZdkziHJijx9rwZgShr9gYi9Hkh)"]'
```

## Creating the Coinjoin transaction

Now let's spend the UTXOs and create four new ones. A facilitator, let's say alice\_1, will create the transaction, sign his input, then pass the partially-signed transaction to the next participant and so on until everybody has signed and someone broadcast it.

Create a key pair for each spender \(alice\_1, carol\_1, eve\_1, mallory\_1\), so that they can sign.
```javascript
const keyPairAlice1 = bitcoin.ECPair.fromWIF(alice[1].wif, network)
const keyPairCarol1 = bitcoin.ECPair.fromWIF(carol[1].wif, network)
const keyPairEve1 = bitcoin.ECPair.fromWIF(eve[1].wif, network)
const keyPairMallory1 = bitcoin.ECPair.fromWIF(mallory[1].wif, network)
```
&nbsp;

Create an address for each recipient \(bob\_1, dave\_1, mallory\_2, alice\_2\), so that they can receive.
```javascript
const keyPairBob1 = bitcoin.ECPair.fromWIF(bob[1].wif, network)
const p2pkhBob1 = bitcoin.payments.p2pkh({pubkey: keyPairBob1.publicKey, network})

const keyPairDave1 = bitcoin.ECPair.fromWIF(dave[1].wif, network)
const p2pkhDave1 = bitcoin.payments.p2pkh({pubkey: keyPairDave1.publicKey, network})

const keyPairMallory2 = bitcoin.ECPair.fromWIF(mallory[2].wif, network)
const p2pkhMallory2 = bitcoin.payments.p2pkh({pubkey: keyPairMallory2.publicKey, network})

const keyPairAlice2 = bitcoin.ECPair.fromWIF(alice[2].wif, network)
const p2pkhAlice2 = bitcoin.payments.p2pkh({pubkey: keyPairAlice2.publicKey, network})
```
&nbsp;

We also have two more recipients that will get back their change \(eve\_1 and mallory\_1\).
```javascript
const p2pkhEve1 = bitcoin.payments.p2pkh({pubkey: keyPairEve1.publicKey, network})
const p2pkhMallory1 = bitcoin.payments.p2pkh({pubkey: keyPairMallory1.publicKey, network})
```
&nbsp;

Create a BitcoinJS transaction builder object.
```javascript
const txb = new bitcoin.TransactionBuilder(network)
```
&nbsp;

Add each inputs by providing the outpoints.
```javascript
txb.addInput('TX_ID', TX_VOUT)
txb.addInput('TX_ID', TX_VOUT)
txb.addInput('TX_ID', TX_VOUT)
txb.addInput('TX_ID', TX_VOUT)
```
&nbsp;

Add each 0.2 BTC payment outputs.
```javascript
txb.addOutput(p2pkhBob1.address, 2e7)
txb.addOutput(p2pkhDave1.address, 2e7)
txb.addOutput(p2pkhMallory2.address, 2e7)
txb.addOutput(p2pkhAlice2.address, 2e7)
```
&nbsp;

Add the change outputs of eve\_1 and mallory\_1. These UTXOs will not be coinjoined. One might reasonably assume that the 4950000 satoshis UTXO belongs to eve\_1 and that the 9950000 satoshis UTXO belongs to mallory\_1. Let's also subtract the mining fees \(0.0005 BTC each\) from them. We can have different policies regarding who has to pay for the mining fees, it is just easier that way for our example.
```javascript
txb.addOutput(p2pkhEve1.address, 5e6 - 5e4)
txb.addOutput(p2pkhMallory1.address, 1e7 - 5e4)
```
&nbsp;

> The miner fee is calculated by subtracting the outputs from the inputs. \(20000000 + 20000000 + 25000000 + 30000000\)ins - \(20000000 + 20000000 + 20000000 + 20000000 + 4950000 + 9950000\)outs = 100 000 100 000 satoshis equals 0,001 BTC, this is the mining fee.
&nbsp;

Each participant signs their input with the default `SIGHASH_ALL` flag, which prevents inputs or outputs from being manipulated after the fact.
```javascript
txb.sign(0, keyPairAlice1)
txb.sign(1, keyPairCarol1)
txb.sign(2, keyPairEve1)
txb.sign(3, keyPairMallory1)
```
&nbsp;

Finally we can build the transaction and get the raw hex serialization.
```javascript
const tx = txb.build()
console.log('Transaction hexadecimal:')
console.log(tx.toHex())
```
&nbsp;

Inspect the raw transaction with Bitcoin Core CLI, check that everything is correct.
```shell
decoderawtransaction TX_HEX
```

## Broadcasting the transaction

It's time to broadcast the transaction via Bitcoin Core CLI.
```shell
sendrawtransaction TX_HEX
```
&nbsp;

Inspect the transaction.
> Don't forget the second argument to returns a detailed json object.
```shell
getrawtransaction TX_ID true
```

## What's Next?

Continue "Part Two: Pay To Public Key Hash" with [Native Segwit P2WPKH](../p2wpkh/).
