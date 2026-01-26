---
title: Transaction frames
description: Add frame abstraction for transaction validation, execution, and gas payment
author: Vitalik Buterin (@vbuterin), lightclient (@lightclient), Felix Lange (@fjl)
discussions-to: <URL>
status: Draft
type: Standards Track
category: Core
created: 2026-01-22
requires: 2718, 4844
---

## Abstract

We propose splitting the transaction scope into multiple frames: validations, execution, and post-operation logic. Transaction validity is determined by the result of the validation steps of a transaction.

We further separate transaction validation for the purposes of authorization and the gas fee payment, allowing one contract to pay gas for a transaction that will be executed from another contract.

## Motivation

This new transaction provides a native off ramp from the elliptic curve based cryptographic system used to authenticate transactions today, to post-quantum secure systems. It is defined in such an abstract manner that it can support all important use cases: PQ crypto, signature aggregation, native support for Inclusion Lists, etc.

## Specification

### Constants

| Name               | Value                                   |
| ------------------ | --------------------------------------- |
| `AA_TX_TYPE`       | `0x06`                                  |
| `AA_ENTRY_POINT`   | `address(0x7701)`                       |
| `AA_BASE_GAS_COST` | 15000                                   |
| `PER_FRAME_COST`   | 960 (cost of 60 nonzero calldata bytes) |

### Opcodes

| Name           | Value  |
| -------------- | ------ |
| `APPROVE`      | `0xaa` |
| `TXPARAMDLOAD` | `0xb0` |
| `TXPARAMSIZE`  | `0xb1` |
| `TXPARAMCOPY`  | `0xb2` |

### New Transaction Type

A new [EIP-2718](./eip-2718) transaction with `TransactionType` `AA_TX_TYPE` is introduced. Transactions of this type are referred to as "AA transactions".

The `TransactionPayload` is defined as the RLP serialization of the following:

```
[chain_id, nonce, sender, max_priority_fee_per_gas, max_fee_per_gas, max_fee_per_blob_gas, blob_versioned_hashes, frames, signature]

frames = [[flags, target, gas_limit, data], ...]
```

The field `max_fee_per_blob_gas` represents the maximum fee per blob gas the sender is willing to pay. The field `blob_versioned_hashes` represents a list of versioned hashes, as defined in [EIP-4844](./eip-4844.md). If no blobs are included, `blob_versioned_hashes` must be an empty list and `max_fee_per_blob_gas` must be `0`.

#### Flags

The `flags` field is interpreted as a bit field with three potential modes: `STATIC`, `REVERT`, and `AS_SENDER`.

| Bit | Name      | Summary                                     |
| --- | --------- | ------------------------------------------- |
| 0   | STATIC    | Frame is read-only.                         |
| 1   | REVERT    | Perform transaction-level revert if call fails. |
| 2   | AS_SENDER | Set `frame.caller` to `tx.sender`           |

##### `STATIC` Mode

Frame executes in read-only mode.

##### `REVERT` Mode

If the frame terminates without using the `APPROVE` opcode, perform a Transaction-level Revert (defined below).

##### `AS_SENDER` Mode

Frame caller is set to `tx.sender`.

#### Constraints

Some validity constraints can be determined statically. They are outlined below:

```python
assert tx.chain_id < 2**256
assert tx.nonce < 2**64
assert len(tx.frames) > 0
assert len(tx.sender) == 20
assert tx.frames[n].flags >> 3 == 0
assert len(tx.frames[n].target) == 20 or tx.frames[n].target is None
```

#### Receipt

The `ReceiptPayload` is defined as:

```
[frame_receipt, ...]
frame_receipt = [status, gas_used, sender, payer, logs]
```

### New Opcodes

#### `APPROVE` opcode (`0xaa`)

The `APPROVE` opcode functions equivalently to `RETURN`, except it takes an additional stack argument which describes the scope of the approval. The gas cost is the same as `RETURN`. The return data format is identical to `RETURN`.

The approval argument must be one of the following values:

1. `0x0`: Approval of execution - the sender contract approves future frames calling on its behalf. Only valid when `target` equals `tx.sender`.
2. `0x1`: Approval of payment - the contract approves funding the total gas cost for executing all frames.
3. `0x2`: Approval of execution and payment - combines both `0x0` and `0x1`.

If the approval argument is `>= 0x3`, execution results in an exceptional halt.

`APPROVE` may be invoked from nested calls within a frame. `APPROVE(0x1)` can be invoked at any time and the invoker will pay for the gas in the transaction. Only code executing in the context of `tx.sender` can invoke `APPROVE(0x0)` and `APPROVE(0x02)` successfully. Once `APPROVE` is executed, future calls of the instruction will not change the initial approve status. If `APPROVE` is called again with a value it has previously been called with in an earlier frame (i.e. `0x0`, then `0x2` or `0x1` and again `0x1`) perform a transaction-level revert. If `APPROVE` is called outside a valid context, perform a transaction-level revert.

The status of a call returning with `APPROVE` has three new potential status codes.

| Code | Result               | Description                                      |
| ---- | -------------------- | ------------------------------------------------ |
| 0    | `FAIL`               | Call reverted                                    |
| 1    | `SUCCESS`            | Call completed successfully                      |
| 2    | `APPROVED_EXECUTION` | Call approved execution successfully             |
| 3    | `APPROVED_PAYMENT`   | Call approved payment successfully               |
| 4    | `APPROVED_BOTH`      | Call approved execution and payment successfully |

*Note: codes `0` and `1` already exist today and are replicated here for completeness.*

#### `TXPARAM*` opcodes

The `TXPARAMDLOAD` (`0xb0`), `TXPARAMSIZE` (`0xb1`), and `TXPARAMCOPY` (`0xb2`) opcodes follow the pattern of `CALLDATA*` / `RETURNDATA*` opcode families. Gas cost follows standard EVM memory expansion costs.

Each `TXPARAM*` opcode takes two extra stack input values before the `CALLDATA*` equivalent inputs. The values of these inputs are as follows:

| `in1` | `in2`       | Return value                         | Size    |
| ----- | ----------- | ------------------------------------ | ------- |
| 0x00  | must be 0   | current transaction type             | 32      |
| 0x01  | must be 0   | `nonce`                              | 32      |
| 0x02  | must be 0   | `sender`                             | 32      |
| 0x03  | must be 0   | `max_priority_fee_per_gas`           | 32      |
| 0x04  | must be 0   | `max_fee_per_gas`                    | 32      |
| 0x05  | must be 0   | max cost (basefee=max, all gas used, includes blob cost) | 32      |
| 0x06  | must be 0   | `tx_hash_for_signature`              | 32      |
| 0x07  | must be 0   | `signature`                          | dynamic |
| 0x08  | must be 0   | `max_fee_per_blob_gas`               | 32      |
| 0x09  | must be 0   | `len(blob_versioned_hashes)`         | 32      |
| 0x0a  | blob index  | `blob_versioned_hashes[blob index]`  | 32      |
| 0x10  | must be 0   | `len(frames)`                        | 32      |
| 0x11  | must be 0   | currently executing frame index      | 32      |
| 0x12  | frame index | `target`                             | 32      |
| 0x13  | frame index | `data`                               | dynamic |
| 0x14  | frame index | `gas_limit`                          | 32      |
| 0x15  | frame index | `flags`                              | 32      |
| 0x16  | frame index | `status` (exceptional halt if current/future) | 32      |

Notes:
- 0x03 and 0x04 have a possible future extension to allow indices for multidimensional gas.
- The `status` field (0x16) returns `0` for failure or `1` for success.
- Out-of-bounds access for frame index (`>= len(frames)`) and blob index results in an exceptional halt.
- Invalid `in1` values (not defined in the table above) result in an exceptional halt.

The `tx_hash_for_signature` (0x06) is computed as:

```
keccak256(AA_TX_TYPE || rlp([chain_id, nonce, sender, max_priority_fee_per_gas, max_fee_per_gas, max_fee_per_blob_gas, blob_versioned_hashes, frames]))
```

This is the hash that contracts should use for signature verification. Note that `blob_versioned_hashes` is included in this hash, binding the signature to the specific blobs attached to this transaction.

### Processing flow

When processing a frame transaction, perform the following steps.

Perform stateful validation check:
- Ensure `tx.nonce == state[tx.sender].nonce`

Initialize with transaction-scoped variables:
- `payer_approved: bool = false`
- `sender_approved: bool = false`

Then for each call frame:

1. Execute a `call` with the specified `flags`, `target`, `gas_limit`, and `data`.
   - If `AS_SENDER` is set, check if `sender_approved == true`. If so, set the `caller` to `tx.sender`. If not, perform a Transaction-level Revert.
   - If `AS_SENDER` is not set, set the `caller` to `AA_ENTRY_POINT`.
   - If `target` is null, set the call target to `tx.sender`.
   - The `ORIGIN` opcode returns the `caller` throughout all call depths (including nested calls within the frame).
2. If the call fails (reverts, runs out of gas, or hits an exception), revert the call frame as normal and skip to step 4.
3. If the call exits with `APPROVE`, update approval state based on the argument:
   - `0x0` (execution approval): If `target` equals `tx.sender`, set `sender_approved = true`.
   - `0x1` (payment approval): If `payer_approved` is `false`, increment the sender's nonce, collect the total gas cost from `target`, and set `payer_approved = true`. The total gas cost is defined as `sum(frame.gas_limit for all frames) × effective_gas_price + blob_gas_cost`, where `effective_gas_price` is calculated per EIP-1559 and `blob_gas_cost` is calculated as `len(blob_versioned_hashes) × GAS_PER_BLOB × blob_base_fee` per EIP-4844. If `target` has insufficient balance, perform a Transaction-level Revert.
   - `0x2` (both): Apply both of the above rules.
4. If `REVERT` is set and the frame did not terminate via `APPROVE`, perform a Transaction-level Revert.

After executing all frames, verify that `payer_approved == true`. If it is, refund any unpaid gas to the gas payer. If it is not, the whole transaction is invalid.

**Transaction-level Revert** is defined as follows:

- If `payer_approved == false`, the whole transaction is invalid.
- If `payer_approved == true`, revert all frames up until, but not including, the frame that flipped `payer_approved` to `true`.

### Validation-only processing flow

To *validate* the transaction without *executing* it, run the above only until the point that `payer_approved` is set to `true`, then stop.

If desired, we can set a `MAX_VALIDATION_GAS` (e.g., 400,000), and exit with failure if `gas_spent_so_far + frame.gas_limit > MAX_VALIDATION_GAS`.

The likely two use cases of this are:

- **Mempools**: mempools will need to validate in order to verify that the transaction can pay fees.
- **FOCIL**: FOCIL nodes will need to validate AA transactions to accept them. Attesters will need to validate AA transactions that are in the FOCIL but not in the block, and verify that all validations *fail* (much like they need to verify that validations of all non-included EOA transactions fail).

### Intrinsic Gas

The intrinsic gas for an AA transaction is calculated as:

```
AA_BASE_GAS_COST + (len(frames) × PER_FRAME_COST) + calldata_cost
```

Where `calldata_cost` is calculated per standard EVM rules (4 gas per zero byte, 16 gas per non-zero byte). This intrinsic gas is deducted during state transition before frame execution begins.

### Gas Accounting

Each frame has its own `gas_limit` allocation. Unused gas from a frame is **not** available to subsequent frames. After all frames execute, the gas refund is calculated as:

```
refund = sum(frame.gas_limit for all frames) - total_gas_used
```

This refund is returned to the gas payer (the `target` that called `APPROVE(0x1)` or `APPROVE(0x2)`) and added back to the block gas pool.

Note: This refund mechanism is separate from EIP-3529 storage refunds.

### Blob Gas Accounting

Frame transactions follow the same blob gas accounting rules as EIP-4844 blob transactions:

- The transaction is only valid if `max_fee_per_blob_gas >= get_base_fee_per_blob_gas(block.header)`
- Blob gas is calculated as `len(blob_versioned_hashes) × GAS_PER_BLOB`
- The blob fee is deducted from the payer at the time of `APPROVE(0x1)` or `APPROVE(0x2)` and is **not refundable**
- Blob gas usage counts toward the block's `blob_gas_used` and is subject to `MAX_BLOB_GAS_PER_BLOCK`

The existing `BLOBHASH` opcode (0x49) from EIP-4844 functions identically for frame transactions, returning `tx.blob_versioned_hashes[index]` or zero if out of bounds.

## Rationale

The above rules are sufficient to enable all core goals of account abstraction, including transaction sponsorship, and they are even easily extensible to support quantum-resistant signature aggregation.

### Example 1: Simple Transaction

| Frame | Caller         | Target       | Data      | Flags     |
| ----- | -------------- | ------------ | --------- | --------- |
| 0     | AA_ENTRY_POINT | User account | None      | REVERT    |
| 1     | User account   | Target addr  | User data | AS_SENDER |

Frame 0 verifies the signature and exits with `APPROVE(0x2)` to approve both execution and payment. Frame 1 executes and exits normally via `RETURN`.

The mempool can process this transaction with the following static validations:

- Verify that it has 2 frames, and the first has the needed flags (the flags of the second don't matter).
- Verify that the call of frame 0 succeeds, and does not violate the AA mempool rules (similar to [ERC-7562](https://eips.ethereum.org/EIPS/eip-7562)).

### Example 2: Paymaster Transaction (ERC20 Payment)

| Frame | Caller         | Target       | Data           | Flags     |
| ----- | -------------- | ------------ | -------------- | --------- |
| 0     | AA_ENTRY_POINT | User account | None           | REVERT    |
| 1     | AA_ENTRY_POINT | Paymaster    | Paymaster data | REVERT    |
| 2     | User account   | ERC-20       | Transfer call  | AS_SENDER |
| 3     | User account   | Target addr  | User data      | AS_SENDER |
| 4     | AA_ENTRY_POINT | Paymaster    | postOp call    | none      |

- Frame 0: Verifies signature and exits with `APPROVE(0x0)` to authorize execution from sender.
- Frame 1: Verifies the previous frame's status via `TXPARAM(0x16, 0)`, checks that the user has enough ERC-20 tokens, and that the next frame is an ERC20 send of the right size to the paymaster. Exits with `APPROVE(0x1)` to authorize payment.
- Frame 2: Sends tokens to paymaster.
- Frame 3: User's intended call.
- Frame 4 (optional): Check unpaid gas, refund tokens, possibly convert tokens to ETH on an AMM.

If the contract is not yet deployed, in all cases, prepend a frame calling the factory deploying the contract. The mempool would have to whitelist factories.

### Data Efficiency

**Basic transaction sending ETH from a smart account:**

| Field                             | Bytes |
| --------------------------------- | ----- |
| Tx wrapper                        | 1     |
| Chain ID                          | 1     |
| Nonce                             | 2     |
| Sender                            | 20    |
| Max priority fee                  | 5     |
| Max fee                           | 5     |
| Max fee per blob gas              | 1     |
| Blob versioned hashes (empty)     | 1     |
| Signature                         | 65    |
| Frames wrapper                    | 1     |
| Sender validation frame: target   | 1     |
| Sender validation frame: gas      | 2     |
| Sender validation frame: data     | 0     |
| Sender validation frame: flags    | 1     |
| Execution frame: target           | 20    |
| Execution frame: gas              | 1     |
| Execution frame: data             | 0     |
| Execution frame: flags            | 1     |
| **Total**                         | 128   |

Notes: Nonce assumes < 65536 prior sends. Fees assume < 1099 gwei. Validation frame target is 1 byte because target is tx.sender. Validation gas assumes <= 65,536 gas. Calldata is 65 bytes for ECDSA signature. Blob fields assume no blobs (empty list, zero max fee).

This is almost equivalent in size to an existing transaction; the only extra overhead is the need to specify the sender explicitly.

**First transaction from an account (add deployment frame):**

| Field                      | Bytes |
| -------------------------- | ----- |
| Deployment frame: target   | 20    |
| Deployment frame: gas      | 3     |
| Deployment frame: data     | 100   |
| Deployment frame: flags    | 1     |
| **Total additional**       | 124   |

Notes: Gas assumes cost < 2^24. Calldata assumes small proxy.

**Trustless pay-with-ERC-20 paymaster (add these frames):**

| Field                                | Bytes |
| ------------------------------------ | ----- |
| Paymaster validation frame: target   | 20    |
| Paymaster validation frame: gas      | 3     |
| Paymaster validation frame: calldata | 0     |
| Paymaster validation frame: flags    | 1     |
| Send to paymaster frame: target      | 20    |
| Send to paymaster frame: gas         | 3     |
| Send to paymaster frame: calldata    | 68    |
| Send to paymaster frame: flags       | 1     |
| Paymaster post op frame: target      | 20    |
| Paymaster post op frame: gas         | 3     |
| Paymaster post op frame: calldata    | 0     |
| Paymaster post op frame: flags       | 1     |
| **Total additional**                 | 140   |

Notes: Paymaster can read info from other fields. ERC-20 transfer call is 68 bytes.

There is some inefficiency in the paymaster case, because the same paymaster address must appear in three places (paymaster validation, send to paymaster inside ERC-20 calldata, post op frame), and the ABI is inefficient (~12 + 24 bytes wasted on zeroes). This is difficult to mitigate in a "clean" way, because one of the duplicates is inside the ERC-20 call, "opaque" to the protocol. However, it is much less inefficient than ERC-4337, because not all of the data takes the hit of the 32-byte-per-field ABI overhead.

## Backwards Compatibility

The `ORIGIN` opcode behavior changes for AA transactions, returning the frame's caller rather than the traditional transaction origin. This is consistent with the precedent set by EIP-7702, which already modified `ORIGIN` semantics. Contracts that rely on `ORIGIN == tx.origin` for security checks (a discouraged pattern) may behave differently under AA transactions.

## Security Considerations

TODO

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
