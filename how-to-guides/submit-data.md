---
prev:
  text: "Blobstream rollups"
  link: "/how-to-guides/blobstream-rollups"
---

# Submitting data blobs to Celestia

<!-- markdownlint-disable MD013 -->

To submit data to Celestia, users submit blob transactions (`BlobTx`). Blob
transactions contain two components, a standard Cosmos-SDK transaction called
`MsgPayForBlobs` and one or more `Blob`s of data.

## Maximum blob size

The maximum total blob size in a transaction is just under **2 MiB
(1,973,786 bytes)**, based on a 64x64 share grid (4096 shares).
With one share for the PFB transaction, 4095 shares remain:
1 at 478 bytes and 4094 at 482 bytes each.

This is subject to change based on governance parameters.
Learn more on [the Mainnet Beta page under "Maximum bytes"](/how-to-guides/mainnet.md#maximum-bytes).

It is advisable to submit transactions where the total blob size is significantly
smaller than 1.8 MiB (e.g. 500 KiB) in order for your transaction to get included
in a block quickly. If a tx contains blobs approaching 1.8 MiB then there will
be no room for any other transactions. This means that your transaction will
only be included in a block if it has a higher gas price than every other
transaction in the mempool.

## Fee market and mempool

Celestia makes use of a standard gas-priced prioritized mempool. By default,
transactions with gas prices higher than that of other transactions in the mempool
will be prioritized by validators.

### Fee estimation

Celestia-node provides flexible fee estimation options for submitting transactions:

1. **Default estimation**: By default, fee estimation relies on the consensus node to which the node is connected.

2. **Third-party estimation**: Users can specify a separate endpoint for fee estimation using the `--core.estimator.address` flag in the CLI. This allows using a dedicated service for fee estimation.

3. **Maximum gas price**: Users can set a maximum gas price they're willing to pay for transactions using the `--max.gas.price` flag. If the estimated gas price exceeds this maximum, the transaction will not be submitted. The default maximum is set to 100 times the minimum gas price (0.2 TIA).

:::tip NOTE
Fee estimation differs by transaction type:

- PayForBlobs (PFB) transactions: Only the gas price (cost per unit) is dynamically estimated. The gas usage (number of units) uses a fixed calculation method.
- All other transactions: Both gas price and gas usage are dynamically estimated.
  :::

:::warning IMPORTANT
When using third-party estimation, the consensus endpoint must be running celestia-app v3.8.1 or higher.
:::

### Fees and gas limits

As of version v1.0.0 of the application (celestia-app), there is no protocol
enforced minimum fee (similar to EIP-1559 in Ethereum). Instead, each
consensus node running a mempool uses a locally configured gas price threshold
that must be met in order for that node to accept a transaction, either directly
from a user or gossiped from another node, into its mempool.

As of version v1.0.0 of the application (celestia-app), gas is not refunded.
Instead, transaction fees are deducted by a flat fee, originally specified by
the user in their tx (where fees = gasLimit \* gasPrice). This means that
users should use an accurate gas limit value if they do not wish to overpay.

Under the hood, fees are currently handled by specifying and deducting a flat
fee. However gas price is often specified by users instead of calculating the
flat fee from the gas used and the gas price. Since the state machine does not
refund users for unused gas, gas price is calculated by dividing the total fee
by the gas limit.

#### Estimating PFB gas

Generally, the gas used by a PFB transaction involves a static fixed cost and
a dynamic cost based on the size of each blob in the transaction.

:::tip NOTE
For a general use case of a normal account submitting a PFB, the static
costs can be treated as such. However, due to the description above of how gas
works in the Cosmos-SDK this is not always the case. Notably, if a
vesting account or the `feegrant` modules are used, then these static costs change.
:::

The fixed cost is an approximation of the gas consumed by operations outside
the function `GasToConsume` (for example, signature verification, tx size, read
access to accounts), which has a default value of 65,000 gas.

:::tip NOTE
The first transaction sent by an account (sequence number == 0) has an
additional one time gas cost of 10,000 gas. If this is the case, this
should be accounted for.
:::

Each blob in the PFB contributes to the total gas cost based on its size. The
function `GasToConsume` calculates the total gas consumed by all the blobs
involved in a PFB, where each blob's gas cost is computed by first determining
how many shares are needed to store the blob size. Then, it computes the product
of the number of shares, the number of bytes per share, and the `gasPerByte`
parameter. Finally, it adds a static amount per blob.

The [`blob.GasPerBlobByte`](https://github.com/celestiaorg/celestia-app/blob/ad91a5b2ca2d562f0b8c9c2e1ed6b4d3098a2e8e/specs/src/specs/params.md?plain=1#L27)
and [`auth.TxSizeCostPerByte`](https://github.com/celestiaorg/celestia-app/blob/ad91a5b2ca2d562f0b8c9c2e1ed6b4d3098a2e8e/specs/src/specs/params.md?plain=1#L25)
are parameters that
could potentially be adjusted through the system's governance mechanisms. Hence,
actual costs may vary depending on the current state of these parameters.

#### Gas fee calculation

The total fee for a transaction is calculated as the product of the gas limit
for the transaction and the gas price set by the user:

$\text{Total Fee} = \text{Gas Limit} \times \text{Gas Price}$

The gas limit for a transaction is the maximum amount of gas that a user is
willing to spend on a transaction. It is determined by both a static
fixed cost (FC) and a variable dynamic cost based on the size of each blob involved
in the transaction:

$\text{Gas Limit} = FC + \sum_{i=1}^{n} SSN(B_i) \times SS \times GCPBB$

Where:

- $FC$ = Fixed Cost, is a static value (65,000 gas)
- $\sum_{i=1}^{n} SSN(B_i)$ = SparseSharesNeeded for the $i$th Blob, is the number of shares needed for the $i$th blob in the transaction
- $SS$ = Share Size, is the size of each share
- $GCPBB$ = Gas Cost Per Blob Byte, is a parameter that could potentially be adjusted through the system's governance mechanisms.

The gas fee is set by the user when they submit a transaction. The fee is often
specified by users directly. The total cost for the transaction is then
calculated as the product of the estimated gas limit and the gas price.
Since the state machine does not refund users for unused gas,
it's important for users to estimate the gas limit accurately to
avoid overpaying.

For more details on how gas is calculated per blob, refer to the
[`PayForBlobs` function](https://github.com/celestiaorg/celestia-app/blob/32d247971386c1944d44bec1faeb000b1ff1dd51/x/blob/keeper/keeper.go#L53)
that consumes gas based on the blob sizes. This function uses the
[`GasToConsume` function](https://github.com/celestiaorg/celestia-app/blob/32d247971386c1944d44bec1faeb000b1ff1dd51/x/blob/types/payforblob.go#L157-L167)
to calculate the extra gas charged to pay for a set of blobs in a `MsgPayForBlobs`
transaction. This function calculates the total shares used by all blobs and
multiplies it by the `ShareSize` and `gasPerByte` to get the total gas to consume.

For estimating the total gas required for a set of blobs, refer to the
[`EstimateGas` function](https://github.com/celestiaorg/celestia-app/blob/32d247971386c1944d44bec1faeb000b1ff1dd51/x/blob/types/payforblob.go#L169-L181).
This function estimates the gas based on a linear model that is dependent on
the governance parameters: `gasPerByte` and `txSizeCost`. It assumes other
variables are constant, including the assumption that the `MsgPayForBlobs`
is the only message in the transaction. The `DefaultEstimateGas` function
runs `EstimateGas` with the system defaults.

#### Estimating gas programmatically

Users can estimate an efficient gas limit by using this function:

```go
import (
    blobtypes "github.com/celestiaorg/celestia-app/x/blob/types"
)
gasLimit := blobtypes.DefaultEstimateGas([]uint32{uint32(sizeOfDataInBytes)})
```

If using a celestia-node light client, then this function is automatically
called for you when submitting a blob. This function works by breaking down the
components of calculating gas for a blob transaction. These components consist
of a flat costs for all PFBs, the size of each blob and how many shares each
uses and the parameter for gas used per byte. More information about how gas is
used can be found in the [gas
specs](https://github.com/celestiaorg/celestia-app/blob/d17e231ae3a0150b50a1854f3e9a268c34502b6b/specs/src/specs/resource_pricing.md)
and the exact formula can be found in the [blob
module](https://github.com/celestiaorg/celestia-app/blob/d17e231ae3a0150b50a1854f3e9a268c34502b6b/x/blob/types/payforblob.go#L157-L181).

### Submitting multiple transactions in one block from the same account

The mempool Celestia uses works by maintaining a fork of the canonical state
each block. This means that each time we submit a transaction to it, it will
update the sequence number (aka nonce) for the account that submitted the
transaction. If users wish to submit a second transaction, they can, but must
specify the nonce manually. If this is not done, the new transactions will not
be able to be submitted until the first transaction is reaped from the mempool (i.e. included in a block), or dropped due to timing out.

By default, nodes will drop a transaction if it does not get included in 5
blocks (roughly 30 seconds). At this point, the user must resubmit their
transaction if they want it to eventually be included.

As of v1.0.0 of the application (celestia-app), users are unable to replace an
existing transaction with a different one with higher fees. They must instead
wait 5 blocks from the original submitted time and then resubmit the
transaction. Again, community members have already suggested solutions and a
willingness to accept changes to fix this issue.

## API

Users can currently create and submit `BlobTx`s in six ways.

### The celestia-app consensus node CLI

```bash
celestia-appd tx blob PayForBlobs <hex-encoded namespace> <hex-encoded data> [flags]
```

### The celestia-node light node CLI

Using `blob.Submit`:

```bash
celestia blob submit <hex-encoded namespace> <hex-encoded data> [flags]
```

Available flags:

- `--core.estimator.address string`: Specifies the endpoint of the third-party service for gas price and gas estimation. Format: `<address>:<port>`. If not provided, the default connection to the consensus node will be used.
- `--max.gas.price`: Sets the maximum gas price you're willing to pay for the transaction. If the estimated gas price exceeds this value, the transaction will not be submitted. Default is 0.2 TIA (100x the minimum gas price).

Learn more in the [node tutorial](/tutorials/node-tutorial.md).

### The celestia-node API golang client

See the [golang client tutorial](/tutorials/golang-client-tutorial.md).

### The celestia-node API Rust client

See the [Rust client tutorial](/tutorials/rust-client-tutorial.md).

### RPC to a celestia-node

Using the JSON RPC API, submit data using the following methods:

- [blob.Submit](https://node-rpc-docs.celestia.org/#blob.Submit)
- [state.SubmitPayForBlob](https://node-rpc-docs.celestia.org/#state.SubmitPayForBlob)

Learn more in the [celestia-node API docs](https://node-rpc-docs.celestia.org/).

### Post a blob directly from Celenium

Celenium provides a user-friendly interface to view and interact with data on Celestia,
and allows for submitting blobs directly from the explorer interface.

To submit a blob from Celenium, follow these steps:

1. Navigate to the [Celenium explorer](https://celenium.io) and connect your wallet.
2. Click on the terminal button in the top right corner of the screen and select the "Submit data blob" option.
3. Next, ensure the file you are submitting is in a supported format and upload it.
4. In the "Namespace" field, input the namespace you want to use for the blob in hex format.
5. Finally, click on the "Continue" button to submit your blob, then approve the transaction in your wallet.

Once the blob is submitted, you will see its hash on the screen.
You can also use Celenium’s search bar to search for the blob's hash and view its details.

<!-- markdownlint-disable MD013 -->
