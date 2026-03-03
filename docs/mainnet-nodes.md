# Mainnet Public Nodes User Manual

## Mainnet Public Nodes’ Addresses
#### node1

```bash
"/ip4/52.52.69.223/tcp/8228/p2p/QmZCfzENZqWrWwifJj9BFDvxQWFyYw5GjdB4vN7Ynd4FxY"
```

#### node2

```bash
"/ip4/54.178.252.1/tcp/8228/p2p/QmZ73KHvZ5GFxf6XhHZ3icPeKFo93rk86kZ8qauox3avJP"
```



## Overview

In this guide, we will deploy two local nodes (nodeA and nodeB) and connect them to the public relay nodes (node1 and node2) to perform multi-hop payments. The overall topology is:

```
┌───────┐        ┌───────┐        ┌───────┐        ┌───────┐
│ nodeA │ ─────▶ │ node1 │ ─────▶ │ node2 │ ─────▶ │ nodeB │
│:8227  │        │public │        │public │        │:8237  │
└───────┘        └───────┘        └───────┘        └───────┘
  sender        relay node 1     relay node 2      receiver
```

In this demo, nodeA and nodeB run on the same machine for convenience. In a real-world scenario, they would be on separate machines, each connecting only to a public relay node. The benefit is that local nodes do not need to expose a publicly reachable address — they can send and receive payments through the public relay nodes.



## Local Node Deployment

1. Download fnn

   Please visit the [releases page](https://github.com/nervosnetwork/fiber/releases) to download and use the latest version of fnn.

   ```bash
   mkdir tmp && cd tmp
   tar xzvf fnn-latest.tar.gz
   ```



2. Export the account private key to the fiber node’s ckb directory

   Here, the ckb-cli is used to create an account, which will later be used to pay for opening  channels between the local node and the public mainnet node. If ckb-cli is not installed, please download it from the [releases](https://github.com/nervosnetwork/ckb-cli/releases).

   ```bash
   # Create a local node directory named nodeA
   mkdir -p mainnet-fnn/nodeA/ckb
   ./ckb-cli account new
   ./ckb-cli account export --lock-arg 0xd4cf2823703d170f923549d8efeb34260fc0f3ba --extended-privkey-path exported-key-a
   head -n 1 ./exported-key-a > mainnet-fnn/nodeA/ckb/key
   chmod 600 mainnet-fnn/nodeA/ckb/key
   # check nodeA key
   ./ckb-cli util key-info --privkey-path mainnet-fnn/nodeA/ckb/key
   ```

   ```bash
   # Create a local node directory named nodeB
   mkdir -p mainnet-fnn/nodeB/ckb
   ./ckb-cli account new
   ./ckb-cli account export --lock-arg 0xaa7f14d92341d5570f5680e49ad738e0c990bdba --extended-privkey-path exported-key-b
   head -n 1 ./exported-key-b > mainnet-fnn/nodeB/ckb/key
   chmod 600 mainnet-fnn/nodeB/ckb/key
   # check nodeB key
   ./ckb-cli util key-info --privkey-path mainnet-fnn/nodeB/ckb/key
   ```




3. Copy config.yml and modify rpc_url

   ```bash
   cp config/mainnet/config.yml mainnet-fnn/nodeA
   cp config/mainnet/config.yml mainnet-fnn/nodeB
   ```
	As the comment says, "[use a trusted CKB RPC node](https://github.com/nervosnetwork/fiber/blob/2ab20ffb50243c25109a62ef2ac18b7e4f1a9e70/config/mainnet/config.yml#L55)" should be changed to the RPC endpoint of a CKB node you trust.
   For convenience, I used [Public JSON RPC nodes](https://github.com/nervosnetwork/ckb/wiki/Public-JSON-RPC-nodes) here.
   
    ``` bash
   sed -i.bak 's|rpc_url:.*|rpc_url: "https://mainnet.ckbapp.dev/"|' mainnet-fnn/nodeA/config.yml
   # check rpc_url
   grep rpc_url mainnet-fnn/nodeA/config.yml
    ```

   For nodeB, also modify `rpc_url` and change the listening ports to avoid conflicts with nodeA:

    ```bash
   sed -i.bak 's|rpc_url:.*|rpc_url: "https://mainnet.ckbapp.dev/"|' mainnet-fnn/nodeB/config.yml
   sed -i.bak 's|/ip4/0.0.0.0/tcp/8228|/ip4/0.0.0.0/tcp/8238|' mainnet-fnn/nodeB/config.yml
   sed -i.bak 's|127.0.0.1:8227|127.0.0.1:8237|' mainnet-fnn/nodeB/config.yml
   # check nodeB config
   grep -E 'rpc_url|listening' mainnet-fnn/nodeB/config.yml
    ```




4. Start the nodes

   You need to set a `FIBER_SECRET_KEY_PASSWORD` environment variable in the startup command to encrypt your wallet private key file. I used `123` here for demo purposes, but I recommend using a strong password.
   
   ```bash
   FIBER_SECRET_KEY_PASSWORD='123' RUST_LOG=info ./fnn -c mainnet-fnn/nodeA/config.yml -d mainnet-fnn/nodeA > mainnet-fnn/nodeA/a.log 2>&1 &
   FIBER_SECRET_KEY_PASSWORD='123' RUST_LOG=info ./fnn -c mainnet-fnn/nodeB/config.yml -d mainnet-fnn/nodeB > mainnet-fnn/nodeB/b.log 2>&1 &
   ```




## Establishing a CKB Channel: nodeA ⟺ node1


1. Establish a network connection between nodeA and node1

   ```bash
   curl -s --location 'http://127.0.0.1:8227' --header 'Content-Type: application/json' --data '{
       "id": 1,
       "jsonrpc": "2.0",
       "method": "connect_peer",
       "params": [
           {
               "address": "/ip4/52.52.69.223/tcp/8228/p2p/QmZCfzENZqWrWwifJj9BFDvxQWFyYw5GjdB4vN7Ynd4FxY"
           }
       ]
   }'
   ```

   ```json
   {"jsonrpc":"2.0","result":null,"id":1}
   ```



2. Establish a channel with 499ckb: nodeA (400ckb) ⟺ node1 (151ckb)

   _Node1 has open_channel_auto_accept_min_ckb_funding_amount set at 400ckb, so please input 499ckb or more._

   ```bash
   curl -s --location 'http://127.0.0.1:8227' --header 'Content-Type: application/json' --data '{
       "id": 2,
       "jsonrpc": "2.0",
       "method": "open_channel",
       "params": [
           {
               "peer_id": "QmZCfzENZqWrWwifJj9BFDvxQWFyYw5GjdB4vN7Ynd4FxY",
               "funding_amount": "0xb9e459300",
               "public": true
           }
       ]
   }'
   ```

   ```json
   {"jsonrpc":"2.0","id":2,"result":{"temporary_channel_id":"0x42bf93237d88952441172198fc600ccc49a8a841b0895f8a8573cf550bad8516"}}
   ```




3. Query the channels between nodeA and node1

   ```bash
   curl -s --location 'http://127.0.0.1:8227' --header 'Content-Type: application/json' --data '{
       "id": 3,
       "jsonrpc": "2.0",
       "method": "list_channels",
       "params": [
           {
               "peer_id": "QmZCfzENZqWrWwifJj9BFDvxQWFyYw5GjdB4vN7Ynd4FxY"
           }
       ]
   }'
   ```

   Wait until the state_name changes to `CHANNEL_READY`.

   **Note: When the channel has just changed to the CHANNEL_READY state and you attempt to use send_payment, you may still encounter an error: `Failed to build route`. It is advisable to wait for some time before trying again.**

   ```json
   {"jsonrpc":"2.0","id":3,"result":{"channels":[{"channel_id":"0xa1cda836a83c0eabf287b150a69c6dfad4be9646c7be5d813e2b10f823363971","is_public":true,"is_acceptor":false,"is_one_way":false,"channel_outpoint":"0x33afc31a3fed0555ba6f5d89a8051024a6fba07590c10353c929cb64820394ff00000000","peer_id":"QmZCfzENZqWrWwifJj9BFDvxQWFyYw5GjdB4vN7Ynd4FxY","funding_udt_type_script":null,"state":{"state_name":"CHANNEL_READY"},"local_balance":"0x9502f9000","offered_tlc_balance":"0x0","remote_balance":"0x38407b700","received_tlc_balance":"0x0","pending_tlcs":[],"latest_commitment_transaction_hash":"0x7dac79d29c7d4c6e1389b44cae62a6103ec0267ee67821462c72142cfaa404ec","created_at":"0x19caeac41cc","enabled":true,"tlc_expiry_delta":"0xdbba00","tlc_fee_proportional_millionths":"0x3e8","shutdown_transaction_hash":null,"failure_detail":null}]}}
   ```

   **Why is the local_balance 0x9502f9000 (40,000,000,000) and the remote_balance 0x38407b700 (15,100,000,000)?**

	This channel was established with nodeA contributing 499 CKB and node1 contributing 250 CKB.

	Each side must reserve 99 CKB (98 CKB for <a href="https://github.com/nervosnetwork/fiber/blob/2ab20ffb50243c25109a62ef2ac18b7e4f1a9e70/crates/fiber-lib/src/fiber/config.rs#L23">commitment lock occupied capacity</a> + 1 CKB for <a href="https://github.com/nervosnetwork/fiber/blob/2ab20ffb50243c25109a62ef2ac18b7e4f1a9e70/crates/fiber-lib/src/fiber/config.rs#L18">shutdown transaction fee</a>) to ensure sufficient funds for on-chain settlement when the channel closes. This reserved amount is not available for off-chain payments.

	Actual available funds in the channel:

	nodeA: 499 CKB - 99 CKB = 400 CKB (local_balance is 0x9502f9000)

	node1: 250 CKB - 99 CKB = 151 CKB (remote_balance is 0x38407b700)




## Establishing a CKB Channel: nodeB ⟺ node2


1. Establish a network connection between nodeB and node2

   ```bash
   curl -s --location 'http://127.0.0.1:8237' --header 'Content-Type: application/json' --data '{
       "id": 1,
       "jsonrpc": "2.0",
       "method": "connect_peer",
       "params": [
           {
               "address": "/ip4/54.178.252.1/tcp/8228/p2p/QmZ73KHvZ5GFxf6XhHZ3icPeKFo93rk86kZ8qauox3avJP"
           }
       ]
   }'
   ```

   ```json
   {"jsonrpc":"2.0","result":null,"id":1}
   ```



2. Establish a channel with 499ckb: nodeB (400ckb) ⟺ node2 (151ckb)

   _Node2 has open_channel_auto_accept_min_ckb_funding_amount set at 400ckb, so please input 499ckb or more._

   ```bash
   curl -s --location 'http://127.0.0.1:8237' --header 'Content-Type: application/json' --data '{
       "id": 2,
       "jsonrpc": "2.0",
       "method": "open_channel",
       "params": [
           {
               "peer_id": "QmZ73KHvZ5GFxf6XhHZ3icPeKFo93rk86kZ8qauox3avJP",
               "funding_amount": "0xb9e459300",
               "public": true
           }
       ]
   }'
   ```

   ```json
   {"jsonrpc":"2.0","id":2,"result":{"temporary_channel_id":"0x9475bdfc2f88d20752139dfc77eef3d2b3197f9c529401b9d322756f936838ee"}}
   ```




3. Query the channels between nodeB and node2

   ```bash
   curl -s --location 'http://127.0.0.1:8237' --header 'Content-Type: application/json' --data '{
       "id": 3,
       "jsonrpc": "2.0",
       "method": "list_channels",
       "params": [
           {
               "peer_id": "QmZ73KHvZ5GFxf6XhHZ3icPeKFo93rk86kZ8qauox3avJP"
           }
       ]
   }'
   ```

   Wait until the state_name changes to `CHANNEL_READY`.

   ```json
   {"jsonrpc":"2.0","id":3,"result":{"channels":[{"channel_id":"0x4b86abb452b35b25aabf0215dc4dbadc998dec8b2dc762f50284c7091d542b1e","is_public":true,"is_acceptor":false,"is_one_way":false,"channel_outpoint":"0xba4d16d112368d757b725d494f60451a6316540a630cd9db2d79b5cd8ebb9e6d00000000","peer_id":"QmZ73KHvZ5GFxf6XhHZ3icPeKFo93rk86kZ8qauox3avJP","funding_udt_type_script":null,"state":{"state_name":"CHANNEL_READY"},"local_balance":"0x9502f9000","offered_tlc_balance":"0x0","remote_balance":"0x38407b700","received_tlc_balance":"0x0","pending_tlcs":[],"latest_commitment_transaction_hash":"0x552aad4fade10b03182e561aea66a4ab2e86fc9fb261748093bc3d6974938a8c","created_at":"0x19caf2674a0","enabled":true,"tlc_expiry_delta":"0xdbba00","tlc_fee_proportional_millionths":"0x3e8","shutdown_transaction_hash":null,"failure_detail":null}]}}
   ```

   nodeB: 499 CKB - 99 CKB = 400 CKB (local_balance is 0x9502f9000)

   node2: 250 CKB - 99 CKB = 151 CKB (remote_balance is 0x38407b700)



## Multi-Hop CKB Payment: nodeA → node1 → node2 → nodeB

Now that both channels are established, we can send a multi-hop payment from nodeA to nodeB through the public nodes (see topology in the [Overview](#overview) section). Since node1 and node2's RPC are not publicly accessible, we can only query balance changes on nodeA (port 8227) and nodeB (port 8237). The total fee charged by the intermediate nodes (node1 + node2) can be inferred from the difference.


1. Generate an invoice on nodeB

   Set the amount to 0x5f5e100 (100,000,000 shannon), equivalent to 1 CKB. The payment_preimage should be a unique 32-byte hexadecimal number.

   ```bash
   # Generate a 32-byte random number and represent it in hexadecimal
   payment_preimage="0x$(openssl rand -hex 32)"
   echo $payment_preimage
   ```

   ```bash
   curl -s --location 'http://127.0.0.1:8237' --header 'Content-Type: application/json' --data '{
       "id": 1,
       "jsonrpc": "2.0",
       "method": "new_invoice",
       "params": [
           {
               "amount": "0x5f5e100",
               "currency": "Fibb",
               "description": "test invoice generated by nodeB",
               "expiry": "0xe10",
               "final_cltv": "0x28",
               "payment_preimage": "'$payment_preimage'",
               "hash_algorithm": "sha256"
           }
       ]
   }'
   ```

   Record the `invoice_address` from the response.



2. Query channel balances before payment

   nodeA ⟺ node1

   ```bash
   curl -s --location 'http://127.0.0.1:8227' --header 'Content-Type: application/json' --data '{
       "id": 2,
       "jsonrpc": "2.0",
       "method": "list_channels",
       "params": [
           {
               "peer_id": "QmZCfzENZqWrWwifJj9BFDvxQWFyYw5GjdB4vN7Ynd4FxY"
           }
       ]
   }'
   ```

   nodeB ⟺ node2

   ```bash
   curl -s --location 'http://127.0.0.1:8237' --header 'Content-Type: application/json' --data '{
       "id": 2,
       "jsonrpc": "2.0",
       "method": "list_channels",
       "params": [
           {
               "peer_id": "QmZ73KHvZ5GFxf6XhHZ3icPeKFo93rk86kZ8qauox3avJP"
           }
       ]
   }'
   ```

   As shown in the channel establishment sections, the initial balances are:

   nodeA ⟺ node1: `{"local_balance":"0x9502f9000","remote_balance":"0x38407b700"}`

   nodeB ⟺ node2: `{"local_balance":"0x9502f9000","remote_balance":"0x38407b700"}`



3. Send payment from nodeA to nodeB

   Pass in the previously recorded `invoice_address` to the `send_payment` request on nodeA.

   ```bash
   curl -s --location 'http://127.0.0.1:8227' --header 'Content-Type: application/json' --data '{
       "id": 3,
       "jsonrpc": "2.0",
       "method": "send_payment",
       "params": [
           {
               "invoice": "<invoice_address from step 1>"
           }
       ]
   }'
   ```



4. Repeat Steps 1 and 3 two more times

   Perform two additional `new_invoice` (on nodeB) and `send_payment` (on nodeA) requests, keeping the amount set to 0x5f5e100.



5. Query channel balances after payments

   nodeA ⟺ node1

   Balances changed from `{"local_balance":"0x9502f9000","remote_balance":"0x38407b700"}` to `{"local_balance":"0x93e44c414","remote_balance":"0x395f282ec"}`

   nodeB ⟺ node2

   Balances changed from `{"local_balance":"0x9502f9000","remote_balance":"0x38407b700"}` to `{"local_balance":"0x962113300","remote_balance":"0x372261400"}`



   This means the channel balances have changed as follows before and after the payments:

   - Before payments

     nodeA (40000000000) ⟺ node1 (15100000000)

     nodeB (40000000000) ⟺ node2 (15100000000)

   - After payments

     nodeA (39699399700) ⟺ node1 (15400600300)

     nodeB (40300000000) ⟺ node2 (14800000000)

   Funds changes:

   ​	nodeA: 39699399700 - 40000000000 = -300600300

   ​	nodeB: 40300000000 - 40000000000 = +300000000

   ​	Total intermediate fees (node1 + node2): 300600300 - 300000000 = 600300

   **Conclusion: Three CKB payments of 100,000,000 shannon (1 CKB) each from nodeA → node1 → node2 → nodeB were successfully completed. nodeA paid a total of 300,600,300 shannon, nodeB received 300,000,000 shannon, and the two intermediate relay nodes (node1 + node2) earned a combined fee of 600,300 shannon.**



6. Close the channels

   Close nodeA's channel with node1:

   ```bash
   curl -s --location 'http://127.0.0.1:8227' --header 'Content-Type: application/json' --data '{
       "id": 6,
       "jsonrpc": "2.0",
       "method": "shutdown_channel",
       "params": [
           {
               "channel_id": "0xa1cda836a83c0eabf287b150a69c6dfad4be9646c7be5d813e2b10f823363971",
               "close_script": {
                   "code_hash": "0x9bd7e06f3ecf4be0f2fcd2188b23f1b9fcc88e5d4b65a8637b17723bbda3cce8",
                   "hash_type": "type",
                   "args": "0xd4cf2823703d170f923549d8efeb34260fc0f3ba"
               },
               "fee_rate": "0x3FC"
           }
       ]
   }'
   ```

   Close nodeB's channel with node2:

   ```bash
   curl -s --location 'http://127.0.0.1:8237' --header 'Content-Type: application/json' --data '{
       "id": 6,
       "jsonrpc": "2.0",
       "method": "shutdown_channel",
       "params": [
           {
               "channel_id": "0x4b86abb452b35b25aabf0215dc4dbadc998dec8b2dc762f50284c7091d542b1e",
               "close_script": {
                   "code_hash": "0x9bd7e06f3ecf4be0f2fcd2188b23f1b9fcc88e5d4b65a8637b17723bbda3cce8",
                   "hash_type": "type",
                   "args": "0xaa7f14d92341d5570f5680e49ad738e0c990bdba"
               },
               "fee_rate": "0x3FC"
           }
       ]
   }'
   ```

   After channel closure, the on-chain settlement will reflect the final balances. You can verify on the CKB explorer that:
   - nodeA's address received CKB reflecting its remaining channel balance
   - nodeB's address received CKB reflecting its accumulated payments

