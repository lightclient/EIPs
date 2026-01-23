---
title: Native Account Abstraction
description: Frame-based native account abstraction
author: Vitalik Buterin (@vbuterin), lightclient (@lightclient), Felix Lange (@fjl)
discussions-to: <URL>
status: Draft
type: Standards Track
category: Core
created: 2026-01-22
---

## Abstract

We propose splitting the Ethereum transaction scope into multiple steps:
validations, execution, and post-operation logic. Transaction validity is
determined by the result of the validation steps of a transaction.

We further separate transaction validation for the purposes of authorization
and the gas fee payment, allowing one contract to pay gas for a transaction
that will be executed from another contract.

## Motivation

Native Account Abstraction allows custom validation logic of a transaction and
custom gas payment logic, opening new use-cases and features for wallets and
dApps.

A more detailed motivation for this proposal can be found in the
[README document](../assets/eip-7701/README.md).

## Specification

### Constants

| Name               | Value                               |
| ------------------ | ----------------------------------- |
| `AA_TX_TYPE`       | `0x06`                              |
| `AA_ENTRY_POINT`   | `address(0x7701)`                   |
| `AA_BASE_GAS_COST` | 15000                               |
| `PER_FRAME_COST`   | 960 (cost of 60 nonzero calldata bytes) |

### Opcodes

| Name           | Value  |
| -------------- | ------ |
| `APPROVE`      | `0xaa` |
| `TXPARAMDLOAD` | `0xb0` |
| `TXPARAMSIZE`  | `0xb1` |
| `TXPARAMCOPY`  | `0xb2` |

### New Transaction Type

A new [EIP-2718](./eip-2718) transaction with type `AA_TX_TYPE` is introduced.
Transactions of this type are referred to as "AA transactions".

Their payload should be interpreted as:

```
AA_TX_TYPE || rlp([
  chain_id: int,
  nonce: int,
  sender: Address,
  max_priority_fee_per_gas: int,
  max_fee_per_gas: int,
  frames: List[Frame],
  allowed_pure_codes: List[Address]
])
```

`frames` must be a list of objects, all of the form:

```
[flags: int, target: address or null, gas: int, data: Bytes]
```

`flags` must not have any higher bits set than the flags defined below.
`allowed_pure_codes` must be a list of 0 or more addresses.

A transaction with an empty `frames` list is invalid. Frames with `gas: 0`
follow normal EVM semantics and may succeed if calling an account with no code.

### `APPROVE` opcode (`0xaa`)

We introduce an `APPROVE` opcode, which functions equivalently to `RETURN`,
except it takes an additional stack argument which describes the scope of the
approval. The gas cost is the same as `RETURN`. The return data format is
identical to `RETURN`.

The approval argument must be one of the following values:

1. `0x0`: Approval of execution - the sender contract approves future frames
   calling on its behalf. Only valid when `target` equals `tx.sender`.
2. `0x1`: Approval of payment - the contract approves funding the total gas
   cost for executing all frames.
3. `0x2`: Approval of execution and payment - combines both `0x0` and `0x1`.

If the approval argument is `>= 0x3`, execution results in an exceptional halt.

`APPROVE` may be called from nested calls within a frame, as long as the
approval propagates from the required contract (sender for `0x0`/`0x2`, payer
for `0x1`/`0x2`).

### `TXPARAM*` opcodes

The `TXPARAMDLOAD` (`0xb0`), `TXPARAMSIZE` (`0xb1`), and `TXPARAMCOPY` (`0xb2`)
opcodes follow the pattern of `CALLDATA*` / `RETURNDATA*` opcode families. Gas
cost follows standard EVM memory expansion costs.

Each `TXPARAM*` opcode takes two extra stack input values before the
`CALLDATA*` equivalent inputs. The values of these inputs are as follows:

| `in1` | `in2`       | Return value                        | Size    |
| ----- | ----------- | ----------------------------------- | ------- |
| 0x00  | must be 0   | current transaction type            | 32      |
| 0x01  | must be 0   | `nonce`                             | 32      |
| 0x02  | must be 0   | `sender`                            | 32      |
| 0x03  | must be 0   | `max_priority_fee_per_gas`          | 32      |
| 0x04  | must be 0   | `max_fee_per_gas`                   | 32      |
| 0x05  | must be 0   | max cost (basefee=max, all gas used)| 32      |
| 0x06  | must be 0   | `tx_hash_for_signature`             | 32      |
| 0x10  | must be 0   | `len(frames)`                       | 32      |
| 0x11  | must be 0   | currently executing frame index     | 32      |
| 0x12  | frame index | `target`                            | 32      |
| 0x13  | frame index | `data`                              | dynamic |
| 0x14  | frame index | `gas`                               | 32      |
| 0x15  | frame index | `flags`                             | 32      |
| 0x16  | frame index | `status` (error if current/future)  | 32      |
| 0x20  | must be 0   | `len(allowed_pure_codes)`           | 32      |
| 0x21  | index       | `allowed_pure_codes[in2]`           | 32      |

Notes:
- 0x03 and 0x04 have a possible future extension to allow indices for
  multidimensional gas.
- The `status` field (0x16) returns `0` for failure or `1` for success.
- Out-of-bounds access for frame index (`>= len(frames)`) or allowed_pure_codes
  index (`>= len(allowed_pure_codes)`) results in an exceptional halt.
- Invalid `in1` values (not defined in the table above) result in an
  exceptional halt.

The `tx_hash_for_signature` (0x06) is computed as:

```
keccak256(AA_TX_TYPE || rlp([
  chain_id,
  nonce,
  sender,
  max_priority_fee_per_gas,
  max_fee_per_gas,
  frames,
  allowed_pure_codes
]))
```

This is the hash that contracts should use for signature verification.

### Flags

The flag bits are defined as follows:

**Bit 0: IS_STATIC**

Call cannot modify state but can read (like a STATICCALL).

**Bit 1: IS_PURE**

Call cannot modify state and cannot read *any* state. The following opcodes are
prohibited: `CREATE`, `CREATE2`, `SSTORE`, `SELFDESTRUCT`, `SLOAD`, `BALANCE`,
`SELFBALANCE`, and any `CALL` operation that sends value (non-zero value
parameter).

Exceptions:
- `EXTCODE{COPY,SIZE,HASH}` of addresses in `allowed_pure_codes` is permitted.
- `DELEGATECALL` to addresses in `allowed_pure_codes` is permitted, but purity
  restrictions apply to the executed code.
- No exception for precompiles unless they are listed in `allowed_pure_codes`.

Cannot read environmental opcodes (e.g., `BLOCKHASH`, `COINBASE`, `TIMESTAMP`,
`NUMBER`, `GASLIMIT`, `BASEFEE`). Can use `TXPARAM`. Other frames see this
frame's data as empty.

Note: IS_STATIC and IS_PURE cannot both be 1 at the same time.

**Bit 2: TX_REVERT_ON_FAIL**

If the frame terminates without using the `APPROVE` opcode (i.e., exits via
`RETURN`, `REVERT`, `STOP`, `SELFDESTRUCT`, or due to an exception such as
out-of-gas), perform a Transaction-level Reversion (defined below).

Exception: When `TX_REVERT_ON_FAIL` and `IS_PURE` are both set, a successful
`RETURN` does not trigger a Transaction-level Reversion. This allows pure
verification frames to pass without requiring approval.

**Bit 3: CALL_FROM_SENDER**

Caller is set to `tx.sender` (otherwise it's `AA_ENTRY_POINT`).

### AA transaction processing flow

Initialize with transaction-scoped variables:
- `payer_approved: bool = false`
- `sender_approved: bool = false`

For each call frame:

1. Execute a `call` with the specified `flags`, `target`, `gas`, and `data`.
   - If `CALL_FROM_SENDER` is set, check if `sender_approved == true`. If so,
     set the `caller` to `tx.sender`. If not, perform a Transaction-level
     Reversion.
   - If `CALL_FROM_SENDER` is not set, set the `caller` to `AA_ENTRY_POINT`.
   - If `target` is null, set the call target to `tx.sender`.
   - The `ORIGIN` opcode returns the `caller` throughout all call depths
     (including nested calls within the frame).
2. If the call fails (reverts, runs out of gas, or hits an exception), revert
   the call frame as normal and skip to step 4.
3. If the call exits with `APPROVE`, update approval state based on the
   argument:
   - `0x0` (execution approval): If `target` equals `tx.sender`, set
     `sender_approved = true`.
   - `0x1` (payment approval): If `payer_approved` is `false`, increment the
     nonce, collect the total gas cost from `target`, and set
     `payer_approved = true`. The total gas cost is defined as
     `sum(frame.gas for all frames) × effective_gas_price`, where
     `effective_gas_price` is calculated per EIP-1559. If `target` has
     insufficient balance, perform a Transaction-level Reversion.
   - `0x2` (both): Apply both of the above rules.
4. If `TX_REVERT_ON_FAIL` is set and the frame did not terminate via
   `APPROVE`, perform a Transaction-level Reversion.

After executing all frames, verify that `payer_approved == true`. If it is,
refund any unpaid gas to the gas payer. If it is not, the whole transaction is
invalid.

**Transaction-level Reversion** is defined as follows:

- If `payer_approved == false`, the whole transaction is invalid.
- If `payer_approved == true`, revert all frames up until, but not including,
  the frame that flipped `payer_approved` to `true`.

### Validation-only processing flow

To *validate* the transaction without *executing* it, run the above only until
the point that `payer_approved` is set to `true`, then stop.

If desired, we can set a `MAX_VALIDATION_GAS` (e.g., 400,000), and exit with
failure if `gas_spent_so_far + frame.gas > MAX_VALIDATION_GAS`.

The likely two use cases of this are:

- **Mempools**: mempools will need to validate in order to verify that the
  transaction can pay fees.
- **FOCIL**: FOCIL nodes will need to validate AA transactions to accept them.
  Attesters will need to validate AA transactions that are in the FOCIL but
  not in the block, and verify that all validations *fail* (much like they
  need to verify that validations of all non-included EOA transactions fail).

### Intrinsic Gas

The intrinsic gas for an AA transaction is calculated as:

```
AA_BASE_GAS_COST + (len(frames) × PER_FRAME_COST) + calldata_cost
```

Where `calldata_cost` is calculated per standard EVM rules (4 gas per zero byte,
16 gas per non-zero byte). This intrinsic gas is deducted during state
transition before frame execution begins.

### Gas Accounting

Each frame has its own `gas` allocation. Unused gas from a frame is **not**
available to subsequent frames. After all frames execute, the gas refund is
calculated as:

```
refund = sum(frame.gas for all frames) - total_gas_used
```

This refund is returned to the gas payer (the `target` that called
`APPROVE(0x1)` or `APPROVE(0x2)`) and added back to the block gas pool.

Note: This refund mechanism is separate from EIP-3529 storage refunds.

## Rationale

The above rules are sufficient to enable all core goals of account
abstraction, including transaction sponsorship, and they are even easily
extensible to support quantum-resistant signature aggregation.

### Example 1: Simple Transaction

| Frame | Caller         | Target       | Data      | Flags              |
| ----- | -------------- | ------------ | --------- | ------------------ |
| 0     | AA_ENTRY_POINT | User account | Signature | TX_REVERT_ON_FAIL  |
| 1     | User account   | Target addr  | User data | none               |

Frame 0 verifies the signature and exits with `APPROVE(0x2)` to approve both
execution and payment. Frame 1 executes and exits normally via `RETURN`.

The mempool can process this transaction with the following static validations:

- Verify that it has 2 frames, and the first has the needed flags (the flags
  of the second don't matter).
- Verify that the call of frame 0 succeeds, and does not violate the AA
  mempool rules (similar to [ERC-7562](https://eips.ethereum.org/EIPS/eip-7562)).

### Example 2: Paymaster Transaction (ERC20 Payment)

| Frame | Caller         | Target       | Data           | Flags             |
| ----- | -------------- | ------------ | -------------- | ----------------- |
| 0     | AA_ENTRY_POINT | User account | Signature      | TX_REVERT_ON_FAIL |
| 1     | AA_ENTRY_POINT | Paymaster    | Paymaster data | TX_REVERT_ON_FAIL |
| 2     | User account   | ERC20        | Transfer call  | none              |
| 3     | User account   | Target addr  | User data      | none              |
| 4     | AA_ENTRY_POINT | Paymaster    | postOp call    | none              |

- Frame 0: Verifies signature and exits with `APPROVE(0x0)` to authorize
  execution from sender.
- Frame 1: Verifies the previous frame's status via `TXPARAM(0x16, 0)`, checks
  that the user has enough ERC20 tokens, and that the next frame is an ERC20
  send of the right size to the paymaster. Exits with `APPROVE(0x1)` to
  authorize payment.
- Frame 2: Sends tokens to paymaster.
- Frame 3: User's intended call.
- Frame 4 (optional): Check unpaid gas, refund tokens, possibly convert tokens
  to ETH on an AMM.

If the contract is not yet deployed, in all cases, prepend a frame calling the
factory deploying the contract. The mempool would have to whitelist factories.

### Example 3: Quantum-Safe Aggregation

| Frame | Caller         | Target       | Data       | Flags                      |
| ----- | -------------- | ------------ | ---------- | -------------------------- |
| 0     | AA_ENTRY_POINT | User account | Signature  | TX_REVERT_ON_FAIL, IS_PURE |
| 1     | AA_ENTRY_POINT | User account | Public key | TX_REVERT_ON_FAIL          |
| 2     | User account   | Target addr  | User data  | none                       |

- Frame 0: Verifies the signature using the public key from frame 1's data
  (accessed via `TXPARAM(0x13, 1)`). Exits via `RETURN`. Because both
  `TX_REVERT_ON_FAIL` and `IS_PURE` are set, a successful `RETURN` does not
  trigger a Transaction-level Reversion.
- Frame 1: Verifies the previous frame passed via `TXPARAM(0x16, 0)`. Checks
  the public key and nonce. Exits with `APPROVE(0x2)` if correct.
- Frame 2: User's intended call. Exits normally via `RETURN`.

The total purity of frame 0, and the fact that other frames are not allowed to
access its calldata, means that frame 0 can theoretically be elided and
replaced with a STARK proof of its correct execution. This is what enables
future efficient ZK-STARK support: users can include (highly expensive)
quantum-safe signatures or even STARKs in this phase, and the large calldata
can be completely elided from the block.

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
| Frames wrapper                    | 1     |
| Sender validation frame: target   | 1     |
| Sender validation frame: gas      | 2     |
| Sender validation frame: calldata | 65    |
| Sender validation frame: flags    | 1     |
| Execution frame: target           | 20    |
| Execution frame: gas              | 1     |
| Execution frame: calldata         | 0     |
| Execution frame: flags            | 1     |
| Allowed pure codes wrapper        | 1     |
| **Total**                         | 127   |

Notes: Nonce assumes < 65536 prior sends. Fees assume < 1099 gwei. Validation
frame target is 1 byte because target is tx.sender. Validation gas assumes
<= 65,536 gas. Calldata is 65 bytes for ECDSA signature.

This is almost equivalent in size to an existing transaction; the only extra
overhead is the need to specify the sender explicitly.

**First transaction from an account (add deployment frame):**

| Field                      | Bytes |
| -------------------------- | ----- |
| Deployment frame: target   | 20    |
| Deployment frame: gas      | 3     |
| Deployment frame: calldata | 100   |
| Deployment frame: flags    | 1     |
| **Total additional**       | 124   |

Notes: Gas assumes cost < 2^24. Calldata assumes small proxy.

**Trustless pay-with-ERC20 paymaster (add these frames):**

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

Notes: Paymaster can read info from other fields. ERC-20 transfer call is
68 bytes.

There is some inefficiency in the paymaster case, because the same paymaster
address must appear in three places (paymaster validation, send to paymaster
inside ERC-20 calldata, post op frame), and the ABI is inefficient (~12 + 24
bytes wasted on zeroes). This is difficult to mitigate in a "clean" way,
because one of the duplicates is inside the ERC-20 call, "opaque" to the
protocol. However, it is much less inefficient than ERC-4337, because not all
of the data takes the hit of the 32-byte-per-field ABI overhead.

## Backwards Compatibility

The `ORIGIN` opcode behavior changes for AA transactions, returning the frame's
caller rather than the traditional transaction origin. This is consistent with
the precedent set by EIP-7702, which already modified `ORIGIN` semantics.
Contracts that rely on `ORIGIN == tx.origin` for security checks (a discouraged
pattern) may behave differently under AA transactions.

## Security Considerations

TODO

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
