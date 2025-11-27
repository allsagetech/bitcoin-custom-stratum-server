# Drivechain Stratum Server

*A minimal Stratum-v1 pool server for BIP300/301 (Drivechain)
experimentation using USB/ASIC miners.*

This project provides a small, Python-based Stratum server intended for
**home mining**, **sidechain R&D**, and **Drivechain developers**
running a Drivechain-patched Bitcoin node (BIP300/301).\
It allows real miners (cgminer/bfgminer/ASICs/USB miners) to mine blocks
locally while the server automatically inserts the **BMM Accept
OP_RETURN** commitment for your sidechain.

> **Important:**\
> This is experimental software. It is *not* production-grade, not
> hardened, and assumes a trusted LAN setup.

## Features

-   Minimal **Stratum v1** server compatible with cgminer/bfgminer.
-   Automatic creation of **coinbase transactions** including:
    -   BIP34 block height
    -   Extranonce1 / extranonce2
    -   Optional **BMM Accept OP_RETURN** (BIP301)
-   Reconstructs **full blocks** from share submissions and submits them
    to your Drivechain-patched mainchain node.
-   Fetches h\* (sidechain block hash) from your sidechain daemon.
-   Fully configurable using a `.env` file.
-   Simple, readable Python implementation---easy to modify.

## Requirements

-   Python 3.8+
-   A Drivechain-enabled Bitcoin node (`bitcoind --drivechain=1`)
    running with RPC enabled.
-   (Optional) A sidechain node such as Thunder / CUSF sidechain with
    RPC enabled.
-   cgminer/bfgminer or any USB/ASIC miner that supports Stratum v1.

## Installation

Clone and run directly---no pip dependencies required:

    python3 drivechain_stratum_server.py

The server listens on the configured `STRATUM_HOST:STRATUM_PORT`
(default `0.0.0.0:3333`).

## Configuration (`.env`)

All settings can be overridden via a `.env` file placed in the same
directory.

### **Network**

    NETWORK=regtest     # mainnet | testnet | regtest

### **Mainchain RPC (Drivechain-patched Bitcoin)**

    RPC_HOST=127.0.0.1
    RPC_PORT=18443
    RPC_USER=rpcuser
    RPC_PASSWORD=rpcpassword

### **Sidechain RPC**

(Optional but required for BMM OP_RETURN)

    ENABLE_SIDECHAIN=1
    SC_RPC_HOST=127.0.0.1
    SC_RPC_PORT=18554
    SC_RPC_USER=scrpcuser
    SC_RPC_PASSWORD=scrpcpassword
    SIDECHAIN_NUMBER=0     # 0-255

### **Mining Output (Miner Payout)**

Provide a **20-byte pubkey hash** (NOT a full address):

    MINER_PKH_HEX=1a2b3c... (40 hex chars)

If left blank, the server inserts an **OP_TRUE** anyone-can-spend output
for testing.

### **Stratum Server**

    STRATUM_HOST=0.0.0.0
    STRATUM_PORT=3333
    POOL_DIFFICULTY=1.0
    JOB_REFRESH_INTERVAL=10

## Running a Miner

Example cgminer command:

    cgminer -o stratum+tcp://<server-ip>:3333 -u worker -p x

bfgminer:

    bfgminer -o stratum+tcp://<server-ip>:3333 -u worker -p x

The server automatically: - Sends new jobs every `JOB_REFRESH_INTERVAL`
seconds - Provides extranonce1 / extranonce2 - Accepts shares -
Assembles full blocks when shares meet network target - Submits valid
blocks to the Drivechain node

## BMM (Blind Merged Mining)

If `ENABLE_SIDECHAIN=1`, the server: 1. Calls the sidechain RPC
`getbestblockhash` 2. Wraps the 32-byte hash (h\*) in a **BIP301 BMM
Accept** OP_RETURN:
`OP_RETURN <4-byte header> <1-byte sidechain ID> <32-byte h*>` 3.
Inserts it as a second output in the coinbase transaction.

If the sidechain RPC fails, the code falls back to a zeroed h\*.

## How It Works (High-Level)

1.  Miner connects → sends `mining.subscribe`
2.  Server returns extranonce1 + size of extranonce2
3.  Miner authorizes with `mining.authorize`
4.  Server sends:
    -   Difficulty
    -   New Stratum job (`mining.notify`)
5.  Miner submits shares
6.  Server reconstructs header + full block
7.  If share meets *network* target:
    -   Block is submitted via `submitblock`

## Testing Without a Miner

You can simulate requests using `nc` or Python---ask if you'd like a
testing harness.

## Disclaimer

This is **experimental software** for **Drivechain research and home-lab
setups**.\
It is not optimized, audited, or production-ready. Assumes trusted
miners and LAN environment.
