---
published: true
title: TSEC CodeStorm E-Voting Decentralized App
date: 2018-11-11
categories: writeup
tags: blockchain solidity ethereum node javascript 
---

This post is a journal of my work at the TSEC CodeStorm 2018 while at the same time serves as a case study into the
application of Blockchain for solving Voting.

# What is a Smart Contract?
A smart contract is a computer program on the blockchain which allows secured, verifiable and immutable records of all transactions.

# The Problem Statement
We were given the follwing problem statement, with some additional info.

```
Electronic voting (e-voting), which uses electronic systems to aid casting and counting votes in an election, has been a research topic of interest for the past few decades in cryptography. 
 
Remote-voting can be viewed as special case of secure multi-party computation. Develop a secure, transparent and decentralized application for  remote voting system as a medium for voting for various purposes including views on different issues or political voting.

Showcase your idea, and propose an e-voting protocol based on the blockchain technology.
```

It was then elaborated like this:

- The system should allow eligible voters to cast their votes and display results of votes graphically.
- Design a permissioned model for registration of the eligible voters  by mutual consensus and allow any voter to stand for the election of the leader.
- Provide a synchronous consensus algorithm and unique identity authentication algorithm for the implementation of your idea that stresses on the motivation for a paperless, pollution free system.

# The Approach

![E-Voting-DApp](../../assets/images/e-voting-dapp.png "E-Voting-DApp")

The voter registration/authentication layer is separate from the blockchain layer. No trace of the user's identity is present on the blockchain.

The transactions only consist of the increment in vote count for a candidate, voter's ether address, which here is not used for voter auth but instead, VoterID and Aadhar + Biometrics are used for voter authentication.

The voter first enters their Aadhaar Info and registers their thumbprint to prove they're actually present at the time of casting their vote. Then, we use their VoterID to check a database of eligible voters, and if they're eligible, they proceed to the vote casting page.

The purpose of not including ether wallet address in the auth process was that verifying the addresses would require storing the addresses on a database which was likely to make the voter traceable, which we wanted to avoid at all costs.

For the purpose of the demo we had the contract deployer assigning rights to a particular user's wallet address.
We made use of `plot.ly` for creating graphs of vote results.

# The Flaws

We do not record wallet addresses being used to vote anywhere, this can allow a single voter to use the same wallet address to cast multiple votes.

We also do not store whether a eligible voter has cast his vote or not. Even if we record wallet addresses, the voter will be able to use multiple wallets to cast votes.

# Countermeasures

![E-Voting-DApp-countermeasures](../../assets/images/e-voting-dapp-countermeasures.png "E-Voting-DApp-countermeasures ")

We include 2 different databases in our existing architecture.

1. A database of voters who have voted.
2. A database of ethereum addresses which have been used.

This allows us to maintain anonymity of the voters, separate databases mean the data cannot be co-related easily, and allows us to counter the flaw mentioned above of a single wallet address being able to vote multiple times, or a single voter using different wallet addresses to vote multiple times.


# Conclusion

Voting inherently works on trust, like in traditional banking, you trust the Bank to keep your money safe, and here you trust the Election Commission to maintain fair voting environment. But in the case of banking, the trust can & has been removed by cryptocurrencies and replaced with miners solving mathematical problems, called proof-of-work to verify transactions. Unfortunately, in the scenario of voting, we cannot verify voters with a proof-of-work model. It's the job of Election Commission, and cannot be automated with computers in a near perfect manner, yet.

There are other solutions proposed, such as using facial recognition to match a voter's selfie to the image on his Voter-ID, which I don't believe will be successful in India since our voter ID photos are taken from potatoes, while our selfies are much higher quality. Training a machine learning model to match from those two will make a great blogpost of its own.

The problem statement mentions "a permissioned model for registration of the eligible voters  by mutual consensus", permissioned models will usually involve use of Hyperledger for smart contracts which allows some info to be private while keeping the rest public on the same blockchain. I didn't take a thorough look into this part as we were running out of time, but I do wonder how mutual consensus can help with voter registration. Having one voter verify the other? Doesn't sound very robust.

The code I wrote for this project is [available here](https://github.com/ArionMiles/E-Voting-dApp).

# Conclusion Conclusion

Using any kind of database to authenticate users moves away from the idea of decentralization. By putting your faith in a central authority to maintain that database, you introduce trust in a system meant to be trustless.

The authentication layer is maintained by the Government here, and is non-transparent. This moves away from the idea of trustless system. The only purpose blockchain serves here is to maintain a immutable record of votes, which can be verified by the public.

All of this comes from very brief 32 hours I had for this project at the hackathon. I realize I might have some concepts wrong here, and I'd be happy to know if I've said something incorrect. If you would like to discuss this further, feel free to contact me on twitter!