:warning: **OUTDATED:** This script uses an older version of the flashbots RPC endpoint, and will not work as is today. :warning:

---

searcher-sponsored-tx
=====================

**Forked from [flashbots org](https://github.com/flashbots/searcher-sponsored-tx/) and tweaked to add an `UnstakeAndTransferERC20` engine**

The `UnstakeAndTransfer` engine is similar to the `TransferERC20`, but it sends a `withdraw` transaction to a staking contract before transferring the tokens. It was used for rescuing tokens from a compromised account which had tokens locked up in [this contract](https://etherscan.io/address/0xe8f063c4dc60b2f6c2c900d870ddcdae7daab7f6). The relevant addresses for this use case are hardcoded in a `src/settings` file, make sure to change them for your own run.

This fork also adds a `burner` script that can be used for burning all ETH sent to the compromised account, so a hacker cannot send funds to it to pay for the transactions used to move the tokens elsewhere. By keeping the `burner` running, you make sure that the only transactions that can be sent from the compromised account are txs with zero gas price - the ones that are sent as part as your flashbot bundle. This prevents the hacker from doing any action, giving time for your flashbot bundle to be mined. You can run the burner with `yarn burn`, using the same env vars for configuration as the main script.

:warning: **Please only run this if you understand what you are doing**. You risk sending the funds you want to rescue to a wrong address or burning your own ETH, if you incorrectly set up these scripts. Last, keep in mind that this is not bulletproof. If your private key was compromised, the hacker has the same opportunities as you do, and they could use this very script to steal your funds.

---

This repository contains a simple Flashbots "searcher" for submitting a transaction from account X, but paying for the transaction from account Y. This is accomplished by submitting a Flashbots transaction bundle, with the first transaction(s) executing from account X, and the last, single transaction calling a contract which verifies the early transactions ran successfully, then pays the miner.

We hope you will use this repository as an example of how to integrate Flashbots into your own Flashbot searcher (bot). For more information, see the [Flashbots Searcher FAQ](https://github.com/flashbots/pm/blob/main/guides/flashbots-alpha.md)

Use case
========
The use case for this multi-transaction setup is to make calls from an account that has a compromised private key. Since transferring in any ETH to this compromised wallet will immediately be swept by bots that monitor that account. Transferring in funds will also give any attacker the ability to withdraw tokens that are held by that account.

Using this searcher, you can create a bundle of 0-gas-price transaction that execute against the compromised account, then reward the miner in a later transaction from a non-compromised account. The last transaction should execute a contract function to ensure the earlier transaction ran successfully (both to avoid mistakes AND to ensure the miner can't simply take the reward transaction without processing the 0-gas-price transactions).

Environment Variables
=====================
- ETHEREUM_RPC_URL - Ethereum RPC endpoint. Can not be the same as FLASHBOTS_RPC_URL
- PRIVATE_KEY_ZERO_GAS - Private key for the compromised Ethereum EOA that owns assets that needs to be transferred
- PRIVATE_KEY_DONOR - Private key for an account that has ETH that will be used to fund the miner for the "ZERO_GAS" transactions 
- FLASHBOTS_SECRET - Private key with no funds used to sign bundles (can be a random key)
- RECIPIENT - Ethereum EOA to receive assets from ZERO_GAS account

Setting Miner Reward
====================
Inside `src/index.ts` is :
```
const MINER_REWARD_IN_WEI = ETHER.div(1000).mul(12); // 0.012 ETH
```

This is the amount, in `wei`, sent to `block.coinbase` from the DONOR EOA. This value is specified in TypeScript, instead of an ENVIRONMENT variable, due to how important it is. Setting this to a very large value could result in massive losses of funds to the DONOR EOA. For safety, **do not store large amounts of eth in DONOR EOA**


Selecting a different "engine"
==============================
This system can operate against different protocols by swapping a new "engine" class that adheres to "Base" functionality in the `main()` function. Available engines:
- `TransferERC20`
- `UnstakeAndTransferERC20`
- `CryptoKitties`

An engine accepts relevant parameters during construction and provides functions to retrieve transaction descriptions to be passed in to Flashbots. Selecting and configuring a different engine requires directly modifying the source, uncommenting the engine and setting the necessary variables.


Usage
======================
```
$ npm install
$ PRIVATE_KEY_ZERO_GAS=__COMPROMISED_PRIVATE_KEY__ \
    PRIVATE_KEY_DONOR=__FUNDED_PRIVATE_KEY__ \
    RECIPIENT=__ADDRESS_THAT_RECEIVES_ASSETS__ \
    FLASHBOTS_KEY_ID=__YOUR_PERSONAL_KEY_ID__ \
    FLASHBOTS_SECRET=__YOUR_PERSONAL_SECRET__ \
      npm run start
```
