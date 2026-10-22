---
title: "Wallet Keys on the CLI: Why Changing a Key Breaks Everything"
date: 2026-10-08
draft: true
description: "Create a key, sign, send, and feel the EOA migration tax on a local anvil chain. Then rotate an owner on a tiny smart wallet without changing its address. Foundry commands with real output."
tags: ["post-quantum", "ethereum", "foundry", "crypto-agility", "wallets", "CLI", "PKI"]
series: ["Post-Quantum, Plainly"]
---

This is Part 3 of [Post-Quantum, Plainly](/posts/pq-plainly-1-the-locks/). Part 2 covered [the problem](/posts/pq-plainly-2-the-problem/). Here we make the key-change pain concrete.

**Safety.** Everything below ran on a local `anvil` chain with disposable keys. Testnet or local only. Never paste a real private key into a terminal command or a chat window. Never use these keys on mainnet. For real funds, use a hardware wallet. Your keys are your data.

I used Foundry **1.8.5** (`cast` / `forge` / `anvil`). Install, then confirm:

```bash
curl -L https://foundry.paradigm.xyz | bash
foundryup
cast --version
```

Start a local chain in another terminal and leave it running:

```bash
anvil
```

```bash
export RPC=http://127.0.0.1:8545
```

## 1) Create a wallet

```bash
cast wallet new
```

```
Successfully created new keypair.
Address:     0x2CF2721C71b56a59CbA33Cf8C3503eaEcbc85D65
Private key: 0x…   # yours will differ; treat it as a password
```

The private key is the key. The address is the last 20 bytes of `keccak256` of the uncompressed public key. Export your values (do not paste mine into anything that matters):

```bash
export ALICE_PK=<private-key-from-cast-wallet-new>
export ALICE_ADDR=$(cast wallet address --private-key $ALICE_PK)
```

## 2) Re-derive the address and the public key

```bash
cast wallet address --private-key $ALICE_PK
```

```
0x2CF2721C71b56a59CbA33Cf8C3503eaEcbc85D65
```

```bash
cast wallet public-key --raw-private-key $ALICE_PK
```

```
0x1d07a89c3e4594fb8b4891cd501d3fa75436880b5d5db39da9a34bda729d7d0f…
```

Same key, same address, every time. No server in the middle. Math, not a login form.

## 3) Fund Alice and check the balance

Anvil lets you set a balance without a faucet dance:

```bash
cast rpc anvil_setBalance $ALICE_ADDR 0x8AC7230489E80000 --rpc-url $RPC
```

```
null
```

That hex is 10 ETH in wei. Confirm:

```bash
cast balance $ALICE_ADDR --ether --rpc-url $RPC
```

```
10.000000000000000000
```

Alice can pay gas. Still local. Still fake ETH.

## 4) Sign a message and verify it

```bash
export SIG=$(cast wallet sign --private-key $ALICE_PK "hello from alice")
echo $SIG
```

```
0x2f2d84b1a23b775d22334ee7616dac177c3efe661549585c66f92a6cbaec684f…
```

```bash
cast wallet verify --address $ALICE_ADDR "hello from alice" $SIG
```

```
Validation succeeded. Address 0x2CF2721C71b56a59CbA33Cf8C3503eaEcbc85D65 signed this message.
```

ECDSA signatures carry enough information to recover the public key. Verification is not a password check against a server. It is math: does this signature match this address for this message? Don't trust. Verify.

## 5) Send a transaction (public key goes onchain)

Make a second wallet to receive funds, then send:

```bash
export BOB_PK=$(cast wallet new 2>/dev/null | cut -f2)
export BOB_ADDR=$(cast wallet address --private-key $BOB_PK)

cast send $BOB_ADDR --value 1ether --private-key $ALICE_PK --rpc-url $RPC
```

```
from                 0x2CF2721C71b56a59CbA33Cf8C3503eaEcbc85D65
status               1 (success)
transactionHash      0xc104f5db9b5d1d9869f07724ae2f968de12f3533de59630458967f64e27b5818
to                   0xeCE7BE161d8f6331AF8be2bBA3dcF5197feb52fA
```

Inspect the signature fields on that transaction:

```bash
cast tx 0xc104f5db9b5d1d9869f07724ae2f968de12f3533de59630458967f64e27b5818 --rpc-url $RPC
```

```
from                 0x2CF2721C71b56a59CbA33Cf8C3503eaEcbc85D65
r                    0x333990d8b721faeba162b8d7aad058d4fd5c123f8c34c76634dd198d65e9aa9a
s                    0x7f175ba8e05bee6e4ce9d016f2d82bb7d2afab70b4d6eba2768b1580e9bdaff1
yParity              1
```

Those `r`, `s`, and `yParity` values are enough to recover the public key. A never-spent address keeps that public key offchain; a spent one does not. That is the "fresh address" advice from [Part 2](/posts/pq-plainly-2-the-problem/), not a reason to panic-migrate.

## 6) The key-change problem on an EOA

On an externally owned account, the address *is* the key. You cannot rotate the key and keep the address. "Rotation" means: new wallet, move funds, update every place that knew the old address.

```bash
export NEW_PK=$(cast wallet new 2>/dev/null | cut -f2)
export NEW_ADDR=$(cast wallet address --private-key $NEW_PK)

cast send $NEW_ADDR --value 8ether --private-key $ALICE_PK --rpc-url $RPC
cast balance $ALICE_ADDR --ether --rpc-url $RPC
cast balance $NEW_ADDR --ether --rpc-url $RPC
```

```
0.999960621324958000
8.000000000000000000
```

Funds moved. The old address still exists. Every allowance, multisig signer slot, payroll entry, and allowlist that pointed at Alice is now wrong until you fix it by hand. That is the botched-migration risk Vitalik named in [Part 2](/posts/pq-plainly-2-the-problem/): more money lost to migrations than to hacks.

## 7) The agile alternative: key as a setting

A smart account can keep one address and treat the owner key as a field you update. Tiny contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract SimpleWallet {
    address public owner;
    error NotOwner();
    error ZeroAddress();
    event OwnerChanged(address indexed oldOwner, address indexed newOwner);

    constructor(address initialOwner) {
        if (initialOwner == address(0)) revert ZeroAddress();
        owner = initialOwner;
        emit OwnerChanged(address(0), initialOwner);
    }

    receive() external payable {}

    modifier onlyOwner() {
        if (msg.sender != owner) revert NotOwner();
        _;
    }

    function changeOwner(address newOwner) external onlyOwner {
        if (newOwner == address(0)) revert ZeroAddress();
        address old = owner;
        owner = newOwner;
        emit OwnerChanged(old, newOwner);
    }

    function execute(address to, uint256 value, bytes calldata data)
        external onlyOwner returns (bytes memory)
    {
        (bool ok, bytes memory ret) = to.call{value: value}(data);
        require(ok, "call failed");
        return ret;
    }
}
```

Init a forge project, drop that file in `src/`, test it, deploy it:

```bash
forge init swappable-wallet
# put SimpleWallet.sol in src/, then:
forge test
```

```
[PASS] testAddressStaysSame()
[PASS] testOwnerCanChange()
[PASS] testStrangerCannotChange()
```

```bash
forge create src/SimpleWallet.sol:SimpleWallet \
  --rpc-url $RPC --private-key $ALICE_PK --broadcast \
  --constructor-args $ALICE_ADDR
```

```
Deployer: 0x2CF2721C71b56a59CbA33Cf8C3503eaEcbc85D65
Deployed to: 0xE4B0bA66022e027bA86e443262ed2d2BFDf95a8b
```

```bash
export WALLET=0xE4B0bA66022e027bA86e443262ed2d2BFDf95a8b
cast send $WALLET --value 0.5ether --private-key $ALICE_PK --rpc-url $RPC
cast call $WALLET "owner()(address)" --rpc-url $RPC
```

```
0x2CF2721C71b56a59CbA33Cf8C3503eaEcbc85D65
```

Change the owner to the new key. Address and balance stay put:

```bash
cast send $WALLET "changeOwner(address)" $NEW_ADDR \
  --private-key $ALICE_PK --rpc-url $RPC
cast call $WALLET "owner()(address)" --rpc-url $RPC
cast balance $WALLET --ether --rpc-url $RPC
```

```
0xb750a355068BB26728975F422487d0D1b5Ba3eCd
0.500000000000000000
```

Alice can no longer change the owner (`NotOwner`). The new owner can spend:

```bash
cast send $WALLET "execute(address,uint256,bytes)" $BOB_ADDR 0.1ether 0x \
  --private-key $NEW_PK --rpc-url $RPC
cast balance $WALLET --ether --rpc-url $RPC
```

```
status               1 (success)
0.400000000000000000
```

Swap the lock. Keep the door. That is crypto-agility at account level.

## 8) Frames and Hegotá

EIP-7702 delegation is a related path for EOAs; Foundry's `cast wallet sign-auth` speaks that dialect. Frames ([EIP-8141](https://eips.ethereum.org/EIPS/eip-8141)), headed for Hegotá, aim to make swappable verification native for every account, including post-quantum signature schemes later. Hegotá is not the PQ fork; it is the fork that decides whether the PQ forks happen on time. Details and citations are in [Part 2](/posts/pq-plainly-2-the-problem/).

## What you should feel in your hands

1. A wallet key is a big random number; the address is a fingerprint of the public side.
2. Signing proves control without giving the key away, but the onchain signature leaks the public key.
3. On an EOA, changing keys means changing identity.
4. On a smart account, changing keys can be a state update.
5. Post-quantum migration without that property is a fleet of botched moves waiting to happen.

Agility is the product. Algorithms are the consumables. Lock designs again: [Part 1](/posts/pq-plainly-1-the-locks/). Threat model and citations: [Part 2](/posts/pq-plainly-2-the-problem/). Hybrid TLS and SSH practice: [Assume the Math Will Move](/posts/crypto-agility-vitalik/).

*Views are my own and do not represent my employer. Nothing here is financial advice.*
