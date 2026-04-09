# Fiber Network Nodes

> **Important:** Bootnodes are used for **peer discovery only** and cannot be used to create payment channels. To open/create a payment channel, you must connect to a **public node**.

## Mainnet

### Bootnodes

| Name | WSS Address |
|------|-------------|
| garrosh | `/dns4/garrosh.fiber.channel/tcp/443/wss/p2p/QmZ2gCTfEF6vKsiYFF2STPeA2rRLRim9nMtzfwiE7uMQ4v` |
| sylvanas | `/dns4/sylvanas.fiber.channel/tcp/443/wss/p2p/QmcMLnWraRyxd7PFRgvn1QeYRQS2DGsP6fPFCQjtfMs5b2` |

### Public Nodes (can create channels)

The public nodes' CKB addresses do not hold any USDI yet, so UDT channels cannot be created at this time.

| Name | TCP Address | WSS Address | CKB Address | Pubkey | open_channel_auto_accept_min_ckb_funding_amount | auto_accept_channel_ckb_funding_amount | UDT: USDI script | UDT: USDI auto_accept_amount |
|------|-------------|-------------|-------------|--------|--------------------------------------------------|----------------------------------------|-------------------|-------------------------------|
| fiber-mainnet-public-ca | `/ip4/52.52.69.223/tcp/8228/p2p/QmZCfzENZqWrWwifJj9BFDvxQWFyYw5GjdB4vN7Ynd4FxY` | `/dns4/ca.fiber.channel/tcp/443/wss/p2p/QmZCfzENZqWrWwifJj9BFDvxQWFyYw5GjdB4vN7Ynd4FxY` | `ckb1qzda0cr08m85hc8jlnfp3zer7xulejywt49kt2rr0vthywaa50xwsqdjsydf39sklhfvtfnk57vvd7d2vn4kxwqpr3prn` | `03a8d7da8d0934363dbc17f52c872e8d833016415266eabb3527439c5dd17adc6b` | `40000000000` (≥ 400 CKB auto-accepted) | `25000000000` (250 CKB inbound liquidity) | code_hash: `0xbfa35a9c38a676682b65ade8f02be164d48632281477e36f8dc2f41f79e56bfc`, hash_type: `type`, args: `0xd591ebdc69626647e056e13345fd830c8b876bb06aa07ba610479eb77153ea9f` | `10000000` (≥ 10,000,000 auto-accepted) |
| fiber-mainnet-public-tokyo | `/ip4/54.178.252.1/tcp/8228/p2p/QmZ73KHvZ5GFxf6XhHZ3icPeKFo93rk86kZ8qauox3avJP` | `/dns4/tokyo.fiber.channel/tcp/443/wss/p2p/QmZ73KHvZ5GFxf6XhHZ3icPeKFo93rk86kZ8qauox3avJP` | `ckb1qzda0cr08m85hc8jlnfp3zer7xulejywt49kt2rr0vthywaa50xwsqv47n6dc9ay34npwktvlpp5huzjvd07t4qhspqyz` | `033a69e5be369dab43aefa96fa729d83c571ccb066f312136c6ab2d354fcc028f9` | `40000000000` (≥ 400 CKB auto-accepted) | `25000000000` (250 CKB inbound liquidity) | code_hash: `0xbfa35a9c38a676682b65ade8f02be164d48632281477e36f8dc2f41f79e56bfc`, hash_type: `type`, args: `0xd591ebdc69626647e056e13345fd830c8b876bb06aa07ba610479eb77153ea9f` | `10000000` (≥ 10,000,000 auto-accepted) |

## Testnet

### Bootnodes

| Name | WSS Address |
|------|-------------|
| thrall | `/dns4/thrall.fiber.channel/tcp/443/wss/p2p/Qmes1EBD4yNo9Ywkfe6eRw9tG1nVNGLDmMud1xJMsoYFKy` |
| onyxia | `/dns4/onyxia.fiber.channel/tcp/443/wss/p2p/QmdyQWjPtbK4NWWsvy8s69NGJaQULwgeQDT5ZpNDrTNaeV` |

### Public Nodes (can create channels)

| Name | TCP Address | WSS Address | CKB Address | Pubkey | open_channel_auto_accept_min_ckb_funding_amount | auto_accept_channel_ckb_funding_amount | UDT: RUSD script | UDT: RUSD auto_accept_amount |
|------|-------------|-------------|-------------|--------|--------------------------------------------------|----------------------------------------|-------------------|-------------------------------|
| fiber-testnet-public-bottle | `/ip4/18.162.235.225/tcp/8119/p2p/QmXen3eUHhywmutEzydCsW4hXBoeVmdET2FJvMX69XJ1Eo` | `/dns4/bottle.fiber.channel/tcp/443/wss/p2p/QmXen3eUHhywmutEzydCsW4hXBoeVmdET2FJvMX69XJ1Eo` | `ckt1qzda0cr08m85hc8jlnfp3zer7xulejywt49kt2rr0vthywaa50xwsqfy4w0gqjsm0ulnq0l4ft6hu6spztrj72sjtcnx4` | `02b6d4e3ab86a2ca2fad6fae0ecb2e1e559e0b911939872a90abdda6d20302be71` | `40000000000` (≥ 400 CKB auto-accepted) | `25000000000` (250 CKB inbound liquidity) | code_hash: `0x1142755a044bf2ee358cba9f2da187ce928c91cd4dc8692ded0337efa677d21a`, hash_type: `type`, args: `0x878fcc6f1f08d48e87bb1c3b3d5083f23f8a39c5d5c764f253b55b998526439b` | `2000000000` (≥ 20 RUSD auto-accepted) |
| fiber-testnet-public-bracer | `/ip4/18.163.221.211/tcp/8119/p2p/QmbKyzq9qUmymW2Gi8Zq7kKVpPiNA1XUJ6uMvsUC4F3p89` | `/dns4/bracer.fiber.channel/tcp/443/wss/p2p/QmbKyzq9qUmymW2Gi8Zq7kKVpPiNA1XUJ6uMvsUC4F3p89` | `ckt1qzda0cr08m85hc8jlnfp3zer7xulejywt49kt2rr0vthywaa50xwsqwgkulmcyxtv2vgcmgatupg2r02k8n4mjcmm9f5m` | `0291a6576bd5a94bd74b27080a48340875338fff9f6d6361fe6b8db8d0d1912fcc` | `40000000000` (≥ 400 CKB auto-accepted) | `25000000000` (250 CKB inbound liquidity) | code_hash: `0x1142755a044bf2ee358cba9f2da187ce928c91cd4dc8692ded0337efa677d21a`, hash_type: `type`, args: `0x878fcc6f1f08d48e87bb1c3b3d5083f23f8a39c5d5c764f253b55b998526439b` | `2000000000` (≥ 20 RUSD auto-accepted) |
