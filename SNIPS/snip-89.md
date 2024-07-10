---
snip: -
title: Safe Transfer for Fungible Tokens
author: Jack Boyuan Xu <jxu@ethsign.xyz>
discussions-to: -
status: Draft
type: Standards Track
category: SRC
created: 2024-6-24
requires: SNIP-2, SNIP-5, SNIP-6, SNIP-64
---

## Abstract

This SNIP aims to introduce interface support checks for [SNIP-2](snip-2.md) token transfers that prevent potential token loss caused by the sender transferring tokens to an empty address or a contract that does not properly acknowledge inbound [SNIP-2](snip-2.md) token transfers.

## Motivation

With the irreversible nature of blockchain transactions and counterintuitive format of addresses, too many financial assets are lost on a daily basis due to user error such as address typos. [SNIP-3](snip-3.md) already recognizes this issue and has a built-in safe transfer function that ensures the recipient acknowledges the inbound token transfer. There is no reason why this mechanism doesn't exist for [SNIP-2](snip-2.md).

## Specification

The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in RFC 2119.

_"ERC-20" is used interchangeably with "SNIP-2" due to its prevalence within the Starknet ecosystem._

The [SNIP-89](SNIP-89.md) receiver interface ID MUST be defined as follows:

```cairo
/// Extended function selector as defined in SRC-5
const ISNIP89_RECEIVER_ID: felt252 = selector!("fn_on_erc20_received(ContractAddress,ContractAddress,u256,Span<felt252>)->felt252)");
```

An [SNIP-89](SNIP-89.md)-compliant [SNIP-2](snip-2.md)/ERC-20 token contract MUST:

- Call `on_erc20_received` or `onERC20Received` on the recipient and compare the return value against the [SNIP-89](SNIP-89.md) interface ID. If there is a mismatch, the transfer has failed. Revert, return false, or perform any other appropriate actions for failed token transfers.

An [SNIP-89](SNIP-89.md)-compliant [SNIP-2](snip-2.md)/ERC-20 token recipient MUST satisfy one of the following criteria:

- Through [SNIP-5](snip-5.md), self-identify as an account with an [SNIP-6](snip-6.md) interface ID.
- Through [SNIP-5](snip-5.md), confirm support for [SNIP-89](SNIP-89.md) interface ID and implement the following `ISNIP89Receiver` trait.

```cairo
#[starknet::interface]
trait ISNIP89Receiver<TContractState> {
    fn on_erc20_received(
        self: @TContractState,
        operator: ContractAddress,
        from: ContractAddress,
        amount: u256,
        data: Span<felt252>
    ) -> felt252;

    fn onERC20Received(
        self: @TContractState,
        operator: ContractAddress,
        from: ContractAddress,
        amount: u256,
        data: Span<felt252>
    ) -> felt252;
}
```

## Implementation

Below is a snippet of an OpenZeppelin-based example implementation of a `safe_transfer_from` function:

```cairo
fn safe_transfer_from(
    ref self: ContractState,
    sender: ContractAddress,
    recipient: ContractAddress,
    amount: u256,
    data: Span<felt252>,
) {
    self.erc20.transfer_from(sender, recipient, amount);
    if !self._check_on_erc20_received(sender, recipient, amount, data) {
        self._throw_invalid_receiver(recipient);
    }
}

fn _check_on_erc20_received(
    self: @ContractState,
    sender: ContractAddress,
    recipient: ContractAddress,
    amount: u256,
    data: Span<felt252>,
) -> bool {
    let src5_dispatcher = ISRC5Dispatcher { contract_address: recipient };
    if src5_dispatcher.supports_interface(ISNIP89_RECEIVER_ID) {
        ISNIP89ReceiverDispatcher { contract_address: recipient }
            .on_erc20_received(
                get_caller_address(), sender, amount, data
            ) == ISNIP89_RECEIVER_ID
    } else {
        src5_dispatcher.supports_interface(ISRC6_ID)
    }
}

/// SNIP-64 error standard
fn _throw_invalid_receiver(self: @ContractState, receiver: ContractAddress) {
    let data: Array<felt252> = array!['ERC20InvalidReceiver', receiver.into(),];
    panic(data);
}
```

See the full repository [here](https://github.com/boyuanx/starknet-erc20-safetransfer).

## Security

As with calling any external contract, calling this receive hook may result in reentrancy attacks. Use proper design patterns and/or reentrancy guards if needed.

## Copyright

Copyright and related rights waived via [MIT](../LICENSE).