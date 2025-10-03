---
title: Do not create empty accounts
description: Avoid creating empty, one-off accounts.
author: lightclient (@lightclient)
discussions-to: <URL>
status: Draft
type: Standards Track
category: Core
created: 2025-10-03
---

## Abstract

If an account is created with an empty code hash and no storage, do not write it
to the trie.

## Motivation

Nearly 50% of new accounts are never called after they are created. Because
the `CREATE` operation is the only mechanism that allow "scripting" to users, it
is frequently used for one-off operations.

Because the goal of the script is achieved by execution of the initcode itself,
the return size is zero. Although the caller would prefer to not create an
account, this is cheapest way to executing arbitrary code.

## Specification

After an `initcode` frame returns, check the following:

- the nonce is `1`
- the balance is `0`
- the storage is empty
- the return value is empty

If all above are true, delete the new account from state.

## Rationale

### Verifying nonce is `1`

If the initcode deploys other contracts with `CREATE` it's nonce will be
incremented. In these cases, we will continue to write the account to state to
avoid a scenario where `CREATE` can be called multiple times with same nonce,
but different code.

## Backwards Compatibility

No backward compatibility issues found.

## Test Cases

TBD

## Reference Implementation

TBD

## Security Considerations

Needs discussion.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
