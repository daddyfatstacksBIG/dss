# Multi Collateral Dai

This repository contains the core smart contract code for Multi
Collateral Dai. This is a high level description of the system, assuming
familiarity with the basic economic mechanics as described in the
whitepaper.

## Additional Documentation

`dss` is also documented in the [wiki](https://github.com/makerdao/dss/wiki) and in [DEVELOPING.md](https://github.com/makerdao/dss/blob/master/DEVELOPING.md)

## Design Considerations

- Token agnostic
  - system doesn't care about the implementation of external tokens
  - can operate entirely independently of other systems, provided an authority assigns
    initial collateral to users in the system and provides price data.

- Verifiable
  - designed from the bottom up to be amenable to formal verification
  - the core cdp and balance database makes *no* external calls and
    contains *no* precision loss (i.e. no division)

- Modular
  - multi contract core system is made to be very adaptable to changing
    requirements.
  - allows for implementations of e.g. auctions, liquidation, CDP risk
    conditions, to be altered on a live system.
  - allows for the addition of novel collateral types (e.g. whitelisting)


## Collateral, Adapters and Wrappers

Collateral is the foundation of Dai and Dai creation is not possible
without it. There are many potential candidates for collateral, whether
native ether, ERC20 tokens, other fungible token standards like ERC777,
non-fungible tokens, or any number of other financial instruments.

Token wrappers are one solution to the need to standardise collateral
behaviour in Dai. Inconsistent decimals and transfer semantics are
reasons for wrapping. For example, the WETH token is an ERC20 wrapper
around native ether.

In MCD, we abstract all of these different token behaviours away behind
*Adapters*.

Adapters manipulate a single core system function: `slip`, which
modifies user collateral balances.

Adapters should be very small and well defined contracts. Adapters are
very powerful and should be carefully vetted by MKR holders. Some
examples are given in `join.sol`. Note that the adapter is the only
connection between a given collateral type and the concrete on-chain
token that it represents.

There can be a multitude of adapters for each collateral type, for
different requirements. For example, ETH collateral could have an
adapter for native ether and *also* for WETH.


## The Dai Token

The fundamental state of a Dai balance is given by the balance in the
core (`vat.dai`, sometimes referred to as `D`).

Given this, there are a number of ways to implement the Dai that is used
outside of the system, with different trade offs.

*Fundamentally, "Dai" is any token that is directly fungible with the
core.*

In the Kovan deployment, "Dai" is represented by an ERC20 DSToken.
After interacting with CDPs and auctions, users must `exit` from the
system to gain a balance of this token, which can then be used in Oasis
etc.

It is possible to have multiple fungible Dai tokens, allowing for the
adoption of new token standards. This needs careful consideration from a
UX perspective, with the notion of a canonical token address becoming
increasingly restrictive. In the future, cross-chain communication and
scalable sidechains will likely lead to a proliferation of multiple Dai
tokens. Users of the core could `exit` into a Plasma sidechain, an
Ethereum shard, or a different blockchain entirely via e.g. the Cosmos
Hub.


## Price Feeds

Price feeds are a crucial part of the Dai system. The code here assumes
that there are working price feeds and that their values are being
pushed to the contracts.

Specifically, the price that is required is the highest acceptable
quantity of CDP Dai debt per unit of collateral.


## Liquidation and Auctions

An important difference between SCD and MCD is the switch from fixed
price sell offs to auctions as the means of liquidating collateral.

The auctions implemented here are simple and expect liquidations to
occur in *fixed size lots* (say 10,000 ETH).


## Settlement

Another important difference between SCD and MCD is in the handling of
System Debt. System Debt is debt that has been taken from risky CDPs.
In SCD this is covered by diluting the collateral pool via the PETH
mechanism. In MCD this is covered by dilution of an external token,
namely MKR.

As in collateral liquidation, this dilution occurs by an auction
(`flop`), using a fixed-size lot.

In order to reduce the collateral intensity of large CDP liquidations,
MKR dilution is delayed by a configurable period (e.g 1 week).

Similarly, System Surplus is handled by an auction (`flap`), which sells
off Dai surplus in return for the highest bidder in MKR.


## Authentication

The contracts here use a very simple multi-owner authentication system,
where a contract totally trusts multiple other contracts to call its
functions and configure it.

It is expected that modification of this state will be via an interface
that is used by the Governance layer.

## DSS Current Deployment

### Mainnet

```
# dss deployment on mainnet from 0xdDb108893104dE4E1C6d0E47c42237dB4E617ACc
# Thu 14 Nov 2019

export DEPLOYER=0xdDb108893104dE4E1C6d0E47c42237dB4E617ACc
export MULTICALL=0x5e227AD1969Ea493B43F840cfF78d08a6fc17796
export FAUCET=0x0000000000000000000000000000000000000000
export MCD_DEPLOY=0xbaa65281c2FA2baAcb2cb550BA051525A480D3F4
export MCD_GOV=0x9f8F72aA9304c8B593d555F12eF6589cC3A579A2
export MCD_ADM=0x9eF05f7F6deB616fd37aC3c959a2dDD25A54E4F5
export MCD_VAT=0x35D1b3F3D7966A1DFe207aa4514C12a259A0492B
export MCD_JUG=0x19c0976f590D67707E62397C87829d896Dc0f1F1
export MCD_CAT=0x78F2c2AF65126834c51822F56Be0d7469D7A523E
export MCD_VOW=0xA950524441892A31ebddF91d3cEEFa04Bf454466
export MCD_JOIN_DAI=0x9759A6Ac90977b93B58547b4A71c78317f391A28
export MCD_FLAP=0xdfE0fb1bE2a52CDBf8FB962D5701d7fd0902db9f
export MCD_FLOP=0xBE00FE8Dfd9C079f1E5F5ad7AE9a3Ad2c571FCAC
export MCD_PAUSE=0xbE286431454714F511008713973d3B053A2d38f3
export MCD_PAUSE_PROXY=0xBE8E3e3618f7474F8cB1d074A26afFef007E98FB
export MCD_GOV_ACTIONS=0x4F5f0933158569c026d617337614d00Ee6589B6E
export MCD_DAI=0x6B175474E89094C44Da98b954EedeAC495271d0F
export MCD_SPOT=0x65C79fcB50Ca1594B025960e539eD7A9a6D434A3
export MCD_POT=0x197E90f9FAD81970bA7976f33CbD77088E5D7cf7
export MCD_END=0xaB14d3CE3F733CACB76eC2AbE7d2fcb00c99F3d5
export MCD_ESM=0x0581A0AbE32AAe9B5f0f68deFab77C6759100085
export PROXY_ACTIONS=0x82ecD135Dce65Fbc6DbdD0e4237E0AF93FFD5038
export PROXY_ACTIONS_END=0x069B2fb501b6F16D1F5fE245B16F6993808f1008
export PROXY_ACTIONS_DSR=0x07ee93aEEa0a36FfF2A9B95dd22Bd6049EE54f26
export CDP_MANAGER=0x5ef30b9986345249bc32d8928B7ee64DE9435E39
export GET_CDPS=0x36a724Bd100c39f0Ea4D3A20F7097eE01A8Ff573
export PROXY_FACTORY=0xA26e15C895EFc0616177B7c1e7270A4C7D51C997
export PROXY_REGISTRY=0x4678f0a6958e4D2Bc4F1BAF7Bc52E8F3564f3fE4
export ETH=0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2
export VAL_ETH=0x81FE72B5A8d1A857d176C3E7d5Bd2679A9B85763
export PIP_ETH=0x81FE72B5A8d1A857d176C3E7d5Bd2679A9B85763
export MCD_JOIN_ETH_A=0x2F0b23f53734252Bda2277357e97e1517d6B042A
export MCD_FLIP_ETH_A=0xd8a04F5412223F513DC55F839574430f5EC15531
export BAT=0x0D8775F648430679A709E98d2b0Cb6250d2887EF
export VAL_BAT=0xB4eb54AF9Cc7882DF0121d26c5b97E802915ABe6
export PIP_BAT=0xB4eb54AF9Cc7882DF0121d26c5b97E802915ABe6
export MCD_JOIN_BAT_A=0x3D0B1912B66114d4096F48A8CEe3A56C231772cA
export MCD_FLIP_BAT_A=0xaA745404d55f88C108A28c86abE7b5A1E7817c07
export SAI=0x89d24A6b4CcB1B6fAA2625fE562bDD9a23260359
export PIP_SAI=0x54003DBf6ae6CBa6DDaE571CcdC34d834b44Ab1e
export MCD_JOIN_SAI=0xad37fd42185Ba63009177058208dd1be4b136e6b
export MCD_FLIP_SAI=0x5432b2f3c0DFf95AA191C45E5cbd539E2820aE72
export PROXY_PAUSE_ACTIONS=0x6bda13D43B7EDd6CAfE1f70fB98b5d40f61A1370
export PROXY_DEPLOYER=0x1b93556AB8dcCEF01Cd7823C617a6d340f53Fb58
export SAI_TUB=0x448a5065aeBB8E423F0896E6c5D525C040f59af3
export MIGRATION=0xc73e0383F3Aff3215E6f04B0331D58CeCf0Ab849
export MIGRATION_PROXY_ACTIONS=0xe4B22D484958E582098A98229A24e8A43801b674
```