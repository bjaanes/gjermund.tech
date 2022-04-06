---
title: "Making Ledger work on REStake"
description: A guide to making ledger work for auto-compounding on REStake
images:
- 
date: 2022-04-07T00:08:05+02:00
draft: false
keywords:
- cosmos
- cosmos ecosystem
- ledger
- blockchain
- cosmos hub
- eco stake
- auto-compounding
---

Not long ago, the multi-chain validator [Eco Stake](https://www.ecostake.com/) released an amazing tool for auto-compounding in the Cosmos Ecosystem called [REStake](https://restake.app/).

This tool was enabled by a recent addition to the Cosmos SDK called authz. Authz enables an account to give any other account permissions
to perform actions on their behalf. Eco Stake leverages this by letting users give validators permissions to withdraw and re-delegate on their behalf.

For Ledger users, however, the joy was short-lived because unfortunately the authz module has a bug that makes this difficult to do properly.

## TL;DR

There is now a workaround to use Ledger with REStake, but it requires some command line usage. 

I have however made a guide and little program to make the process a little easier.

If you're not interested in the why and how, go directly to the [guide](#guide)

## Why doesn't it work?

In short, because of a bug with Authz that causes messages to be created in an incompatible format.

If you want to read more in-depth about this, the following links will go more in-depth:

- https://medium.com/confio/authz-and-ledger-signing-an-executive-summary-8754a0dc2a88
- https://github.com/eco-stake/restake/issues/144

## How did I make it work?

As the above-mentioned article states, it should be possible to grant Authz permissions using the chain clis directly.

This however, turned out to be a truth with some modifications. It does indeed work to grant _some_ types of permissions with the clis, but not all.
In particular, so-called "generic" grants work fine, but the more specialized delegation grants don't.

The difference between the generic and the specialized delegation grant is that the delegation grant has more options to for instance limit which validators can be granted access.
The generic grants simply give someone permission to call specific messages on the blockchain on your behalf.

What I discovered is that you can still use the generic grants to give permission to the necessary messages.

There are, however, two drawbacks:
1. You have to download the cli for each, find all the correct parameters and interact with the chain in a quite manual way.
2. Using the generic grant you cannot limit which validators the grantee have permission to re-delegate to.

    That being said, the REStake scripts only re-delegates the rewards from themselves. 
    But, in theory, there is nothing stopping them from withdrawing and delegating all funds to any validator they choose.  
    
    That also being said, you can revoke the permissions at any time, limiting the potential "harm" (which already is very limited thanks to authz only giving the limited permissions you explicitly grant)
  
After figuring out how to make the correct commands, I also found that the REStake script was explicitly looking for the delegation grant and did not recognize the genric one. 
To correct this problem I made a PR to the REStake repository and got help from Tom at Eco Stake to test the changes. And it finally worked!
My ledger accounts were finally being auto-compounded.

So, after having done all the work to learn and figure out how to make the correct authz grants manually
I decided to write a cli that helps you find all the information you need to be able to run these commands yourself.

The details on how to use the cli can be found in the guide below, 
and the code for the cli can be found here: https://github.com/empowerchain/restake-authz-ledger

## Guide

### Overview

The steps you'll need to take are:

1. Download the cli for the chain you want to enable REStake on
2. Find the correct addresses for the validator(s) you want to grant access
3. Run a command to grant access to withdrawing staking rewards (on your behalf, they won't actually get the rewards in their wallet!)
4. Run a command to grant access to delegating (again, on your behalf. This is the only direct access they'll have to your funds)

To make these steps easier, I've made a simple CLI that fetches all the information you need and tells you which commands you need to write.
However, in case you want to do this process on your own I will be detailing where you can go to manually find the information you need.

We'll go through each of the steps and the command in detail to make sure you understand exactly the changes you are making.

> A note: the cli is attempting to correctly set the parameters for every supported chain, and thus not all of them
have been tested thoroughly (if you feel like donating funds for every chain in Cosmos, I'm happy to accept :)).
>
> The point being: if the commands fail, you might have to make adjustments. Probably around the fees. If you
> need any help, reach out to me on [twitter @gjermundbjaanes](https://twitter.com/gjermundbjaanes) or on [Eco Stake's discord server](https://discord.gg/HddWzvqgHk) (gjermund#1586).

### The actual step-by-step guide

Have your wallet public key and Ledger device ready.

In the step-by-step guide I will use the Cosmos Hub as the example, but you can use it on any chain other chain 
(although, if you're not staking Atom by now, what are you even doing?)

#### 1. Download and extract the restake-authz-ledger cli

Download the latest version of the cli and extract it.

You'll find executables for linux, mac and windows here: https://github.com/empowerchain/restake-authz-ledger/releases/tag/latest

You can also build the cli from source if you prefer, see instructions in the repo itself.

#### 2. Run the restake-authz-ledger cli

Open a terminal in the same folder where you extracted the cli binary.

> Don't worry, you won't be making any transactions just yet. The cli does not do that for you.

Run the command `./restake-authz-ledger grant` and the cli will guide you through selecting the correct network, wallet address and validator.

{{< youtube yF_zqUXpi5k >}}

After running the command, you should be presented with some instructions and commands. 

It might look something like this:
```
Instructions to create the necessary authz grants to enable REStake:

First you need to get the cli for the chain.
If you want to build the cli from source, the source code is here: https://github.com/cosmos/gaia (reported recommended version is v6.0.4)
Linux amd64: https://github.com/cosmos/gaia/releases/download/v6.0.4/gaiad-v6.0.4-linux-amd64
Linux arm64: https://github.com/cosmos/gaia/releases/download/v6.0.4/gaiad-v6.0.4-linux-arm64
Darwin (macOS) amd64: https://github.com/cosmos/gaia/releases/download/v6.0.4/gaiad-v6.0.4-darwin-amd64
Windows amd64: https://github.com/cosmos/gaia/releases/download/v6.0.4/gaiad-v6.0.4-windows-amd64.exe

After getting the cli, you need to run 3 commands:

1: Add you ledger to your keys in gaiad with the following command:
$ /path/to/binary/gaiad keys add ledger --ledger --keyring-backend file


2: Grant your validator access to withdraw rewards (to your own wallet, not theirs) on your behalf:
$ /path/to/binary/gaiad tx authz grant cosmos195mm9y35sekrjk73anw2az9lv9xs5mzt6k0xuh generic --msg-type /cosmos.distribution.v1beta1.MsgWithdrawDelegatorReward --from ledger --ledger --chain-id cosmoshub-4 --node https://rpc-cosmoshub.blockapsis.com:443 --keyring-backend file --gas auto --gas-prices 0uatom --gas-adjustment 1.5

3: Grant your validator access to delegate on your behalf:
$ /path/to/binary/gaiad tx authz grant cosmos195mm9y35sekrjk73anw2az9lv9xs5mzt6k0xuh generic --msg-type /cosmos.staking.v1beta1.MsgDelegate --from ledger --ledger --chain-id cosmoshub-4 --node https://rpc-cosmoshub.blockapsis.com:443 --keyring-backend file --gas auto --gas-prices 0uatom --gas-adjustment 1.5
```

We'll go through each of them next.

#### 3. Download the chain cli

You can download the source code and build the binary yourself if you are familiar with that process, or you can use the binary downloads provided directly.

> It is worth noting that some chains don't provide this information to the chain registry (our it might be outdated). 
> Juno is such an example at the current time of writing.
> 
> In such cases, it should be fairly easy to find the repository by a quick google search like for instance `juno github` which should give you https://github.com/CosmosContracts/juno

#### 4. Add Ledger to the cli's keyring

The first command is a relatively harmless one. You'll simply add your Ledgers public key to the chain cli key management.

Make sure your Ledger is connected, unlocked and that the Cosmos app is open on it.

```shell
$ /path/to/binary/gaiad keys add ledger --ledger --keyring-backend file
```

The command adds a new key named `ledger`, using Ledger (`--ledger`) and the keyring backend on the file system (`--keyring-backend file`).

When you run this command, you will need to confirm the key on your Ledger device.

#### 5. Grant permission to withdraw delegator rewards

Now, things are starting to get fun. It's time to add the first grant to your validator.

If you are not using the restake-authz-ledger cli you can find your validators restake address here: https://github.com/eco-stake/validator-registry

```shell
$ /path/to/binary/gaiad tx authz grant cosmos195mm9y35sekrjk73anw2az9lv9xs5mzt6k0xuh generic --msg-type /cosmos.distribution.v1beta1.MsgWithdrawDelegatorReward --from ledger --ledger --chain-id cosmoshub-4 --node https://rpc-cosmoshub.blockapsis.com:443 --keyring-backend file --gas auto --gas-prices 0uatom --gas-adjustment 1.5
```

Let's go through each parameter here:
* `/path/to/binary/gaiad` this is the path to where you downloaded the chain cli. If your current working directory is the same, you simply need to replace this with `./gaiad`
* `tx` tells the cli we're making a transaction
* `authz` more specifically, an authz transaction
* `grant` that the authz transaction is to grant permissions
* `cosmos195mm9y35sekrjk73anw2az9lv9xs5mzt6k0xuh` is the wallet address we're granting permissions _to_. This is the validators REStake bot address (the wallet they use to run the REStake script that redelegates for you)
* `generic` this it the type of authz permission. As mentioned before, we need to use the generic type. It also requires `--msg-type` to be specified
* `--msg-type /cosmos.distribution.v1beta1.MsgWithdrawDelegatorReward` which action we are giving the grantee permission to perform on our behalf. In this case it is the withdraw delegator reward action (often named "claim" in wallets)
* `--from ledger` this is the name of the key we added earlier to the keyrin
* `--ledger` this tells the cli that the transaction will be signed using a Ledger device
* `--chain-id cosmoshub-4` The chain identifier (there should only be one correct option here for a mainnet transaction)
* `--node https://rpc-cosmoshub.blockapsis.com:443` this lets the cli know where to broadcast the transaction
* `--keyring-backend file` the keyring backend to use (where to find the `--from` key name)
* `--gas auto` this lets the cli try to figure out the correct amount of gas needed for the transaction
* `--gas-prices 0uatom` the minimum gas fee "per unit of gas"
* `--gas-adjustment 1.5` this adjusts the auto gas up by 50% to give a buffer.

> If you experience an error message related to insufficient gas fees, try to increase `--gas-prices` parameter.

> If you exprience and error message related to "out of gas", try to increase the `--gas-adjustment` parameter.

> In some cases, the output command from the restake-authz-ledger cli is not able to determine the correct fee price and/or denom.
> In those case you might see something like this `--gas-prices CHANGEFEEchangedenom` and you will have to find the correct minimum gas fee and denom for the chain

#### 6. Grant permission to delegate

The second grant command is almost identical to the first one, with only a change to the `--msg-type` parameter.

In this step we are granting the validators REStake bot the permission to delegate on our behalf.

```shell
$ /path/to/binary/gaiad tx authz grant cosmos195mm9y35sekrjk73anw2az9lv9xs5mzt6k0xuh generic --msg-type /cosmos.staking.v1beta1.MsgDelegate --from ledger --ledger --chain-id cosmoshub-4 --node https://rpc-cosmoshub.blockapsis.com:443 --keyring-backend file --gas auto --gas-prices 0uatom --gas-adjustment 1.5
```

For a detailed explanation of each paramater, see the previous step.

## Final notes

This is certainly not the most attractive UX, it at least provides a workaround that functions.

In the future, this should be 100% unnecessary, as the fix to the authz bug will be shipped in a future version of the Cosmos SDK.
Until then, this allows us Ledger users a way to enjoy the benefits of auto-compounding.

If you need any help, feel free to reach out to me on [twitter @gjermundbjaanes](https://twitter.com/gjermundbjaanes) or on [Eco Stake's discord server](https://discord.gg/HddWzvqgHk) (gjermund#1586).

You can also open issues on the GitHub repo for the restake-authz-ledger repo itself: https://github.com/empowerchain/restake-authz-ledger/issues