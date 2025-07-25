# Transaction latency testing tool

A Go-based tool for testing and comparing transaction latency performance between different [Flashblocks-enabled Base node](https://blog.base.dev/accelerating-base-with-flashblocks) endpoints.

Flashblocks are sub-blocks issued by the block builder every 200ms, allowing for early confirmation times and making Base 10x faster for real-time applications.

## Requirements

### Node compatibility

For **synchronous** transaction sending testing, the node endpoints must support the `eth_sendRawTransactionSync` RPC method. See [EIP-7966: eth_sendRawTransactionSync Method](https://eips.ethereum.org/EIPS/eip-7966).

For **asynchronous** transaction sending testing, the node endpoints do not have to support the `eth_sendRawTransactionSync` RPC method and will rely on separate polling for confirmed transactions.

[Chainstack Base nodes](https://chainstack.com/build-better-with-base/) for Mainnet & Testnet support both the synchronous one with `eth_sendRawTransactionSync` and the asynchronous one.

## Configuration

Create a `.env` file based on `.env.example`:

```bash
PRIVATE_KEY=your_private_key_here
TO_ADDRESS=0xc2F695613de0885dA3bdd18E8c317B9fAf7d4eba

# If you are going with the asynchronous method and poll for the transaction receipts separately
POLLING_INTERVAL_MS=50

BASE_NODE_ENDPOINT_1=https://your-node-endpoint-1.com
BASE_NODE_ENDPOINT_2=https://your-node-endpoint-2.com

# This a simple label for the resulting filename to remember where you sent the test transactions from. Not used in any node routing.
REGION=singapore

NUMBER_OF_TRANSACTIONS=10

# With `true`, the synchronous eth_sendRawTransactionSync method is used. With `false`, the asynchronous method is used with POLLING_INTERVAL_MS=50
SEND_TXN_SYNC=true

RUN_ENDPOINT2_TESTING=true
```

### Configuration options

#### Transaction sending testing modes

`SEND_TXN_SYNC=true`: 
- Uses `eth_sendRawTransactionSync` method
- Provides instant confirmation receipt when transaction is included in a Flashblock
- Transactions wait for immediate inclusion confirmation

`SEND_TXN_SYNC=false`: 
- Uses standard `eth_sendTransaction` method
- Polls for transaction receipts using `POLLING_INTERVAL_MS`
- Traditional async transaction sending

#### Polling configuration

`POLLING_INTERVAL_MS`:
- Used only when `SEND_TXN_SYNC=false`
- Defines how frequently (in milliseconds) to check for transaction receipts
- Default: 50ms (recommended for accurate latency measurements)
- Lower values = more frequent polling = faster detection but higher load
- Not used when `SEND_TXN_SYNC=true` since confirmations are immediate

#### Endpoint testing

- `BASE_NODE_ENDPOINT_1`: Primary endpoint (e.g., a Flashblocks-enabled)
- `BASE_NODE_ENDPOINT_2`: Secondary endpoint (e.g., a standard non-Flashblocks endpoint)

The tool will:
1. Send transactions to `BASE_NODE_ENDPOINT_1` first
2. Then send transactions to `BASE_NODE_ENDPOINT_2` (if `RUN_ENDPOINT2_TESTING=true`)
3. Compare performance between both endpoints

## Usage

```bash
# Build the container
docker build -t transaction-latency .

# Run the test
docker run -v $(pwd)/data:/data --env-file .env --rm -it transaction-latency
```

### Note about .env file loading

When running with Docker, you may see the log message:
```
Error loading .env file
```

This is expected and harmless. The application uses two methods to load environment variables:
1. **Docker's `--env-file` flag** (recommended): Sets environment variables at container runtime
2. **Direct .env file loading**: Attempts to read the `.env` file from inside the container

Since we use the `--env-file` approach for security (keeping secrets out of the image), the direct file loading fails but the environment variables are still available via Docker. The application continues normally and all configuration works as expected.

## Results

After completion, results are saved to the `./data/` directory:

- `endpoint1-{region}.csv`: Results from `BASE_NODE_ENDPOINT_1`
- `endpoint2-{region}.csv`: Results from `BASE_NODE_ENDPOINT_2` (if enabled)

### CSV format

Each CSV contains:
- `sent_at`: Timestamp when transaction was sent
- `txn_hash`: Transaction hash
- `included_in_block`: Block number where transaction was included
- `inclusion_delay_ms`: Time from sending to confirmation (milliseconds)

## Performance expectations

Flashblocks are produced at 200ms intervals, but actual confirmation times will be higher due to network latency:

- **Standard endpoint**: ~2000ms (2-second block time)
- **Flashblocks endpoint**: ~300-500ms (200ms Flashblock interval + network travel time)

The actual confirmation time depends on:
- Network latency to/from the Base node
- Transaction processing time
- Time until next Flashblock (up to 200ms)
- Network travel time for confirmation response

## Optional eth_getBlockByNumber comparison

On a Flashblocks-enabled node, the following standard RPC methods will retrieve the Flashblocks data instead of full blocks data:

  * `eth_getBlockByNumber` with `pending` tag
  * `eth_getTransactionReceipt`
  * `eth_getBalance` with `pending` tag
  * `eth_getTransactionCount` with `pending` tag
  * `eth_getTransactionByHash` with `pending` tag
  * `eth_sendRawTransactionSync`

For a quick comparison, see:

* `eth-get-block-by-number-pending-examples/preconfirmed-flashblock.log` — the result of running `"method":"eth_getBlockByNumber","params":["pending",true]` on a Flashblocks-enabled node as [block 33228756](https://basescan.org/block/33228756) was forming.
* `eth-get-block-by-number-pending-examples/confirmed-block.log` — the result of running of running `"method":"eth_getBlockByNumber","params":["pending",true]` on a non-Flashblocks-enabled node as [block 33228756](https://basescan.org/block/33228756) was formed.

You will see the key differences in the results:
* Transaction count — 52 in the Flashblock vs. 167 in the fully formed block.
* `stateRoot` — empty in the Flashblock vs. computed in the fully formed block.
* `blockHash` — different in the Flashblock and the fully formed block. The finalized transactions will have the hash of the fully formed block attributed to them.
* `receiptsRoot` — different in the Flashblock and the fully formed block.

## Learn more

- [Base Flashblocks documentation](https://docs.base.org/base-chain/flashblocks/apps)
- [We’re making Base 10x faster with Flashblocks](https://blog.base.dev/accelerating-base-with-flashblocks)
