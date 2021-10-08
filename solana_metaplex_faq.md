---
layout: page
title: Solana and Metaplex Candy Machine FAQ
tags: [software, blockchain, Solana, Metaplex]
---

## Solana

Q: What is the difference between `devnet` and `testnet`?

A: `devnet` is targeted at developers to let them test their smart contracts and applications before deploying to mainnet. `testnet` is used primarily to test new Solana releases. See the [Solana docs](https://docs.solana.com/cluster/rpc-endpoints) for more details.

--

Q: What free RPC endpoints are available? 

A: For devnet:

* https://api.devnet.solana.com

For mainnet: 

* https://api.mainnet-beta.solana.com
* https://solana-api.projectserum.com 

 See the [Solana docs](https://docs.solana.com/cluster/rpc-endpoints) for more details such as rate limits.

 For production, you should consider using a custom RPC endpoint.

--

 Q: What custom RPC endpoints are out there?

 A: You can consider [running your own](https://docs.solana.com/running-validator) or use one of the following:

 * https://www.quicknode.com/chains/sol

 * https://rpcpool.com/#/

 * https://figment.io/datahub/solana/

   


## Metaplex Candy Machine

Q: Why has the price for uploading NFTs to a candy machine changed?

A: The developers changed uploads to start costing a fixed amount per file for Arweave. Currently that's 0.0023 SOL. 

--

Q: How much does it cost to deploy X NFTs using Candy Machine?

A: I've started a spreadsheet with reported costs from the Metaplex Discord users. See it [here](https://docs.google.com/spreadsheets/d/1tEHPIUN1GccLyTsd5PS0tAQMC6ihjq48jlPPz0FK9Yg/edit?usp=sharing).

--

Q: What is the candy machine PDA for mainnet and is it the same for devnet?

A: It's [cndyAnrLdpjq1Ssp1z8xxDsB8dxe7u4HL5Nxi2K5WXZ](https://solscan.io/account/cndyAnrLdpjq1Ssp1z8xxDsB8dxe7u4HL5Nxi2K5WXZ) for both.

--

Q: Why doesn't my candy machine cli command work?

A: Things to check:

* Are you on the latest version of the main branch?

* Are you in the same directory as your .cache file?

* Are you running the command on the correct network? 

  (Commands default to `devnet`. Use `--env mainnet-beta` for `mainnet`.)

  

