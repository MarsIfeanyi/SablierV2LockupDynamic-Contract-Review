### SablierV2LockupDynamic Contract Walkthrough

- This is a review and walkthrough of the SablierV2LockupDynamic smart contract, which uses Segments, milestones and delta for calculation of the custom streaming curve, which determines how the payment rate varies over time.

- [Sablier](https://sablier.com/) is a token streaming protocol developed with Ethereum smart contracts, designed to facilitate `by-the-second` payments for cryptocurrencies, specifically `ERC20`assets.
- The protocol employs a set of persistent and non-upgradable smart contracts that prioritize security, censorship resistance, self-custody, and functionality without the need for trusted intermediaries who may selectively restrict access.

### My Observation

At the time of my review (October 2023), I didn't find any bug in the `SablierV2LockupDynamic.sol` contract.

Also from my observation Sablier labs has the BEST codebase from other protocols i have worked with.
The files and folder structure are well organized.
I also love the excellent folder structure for the smart contract test files.
