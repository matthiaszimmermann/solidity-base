# Solidity Base Repository

This repository provides a complete development environment for building and testing Solidity smart contracts using Foundry within a VSCode DevContainer setup.

## Purpose 

The setup includes OpenZeppelin V5 contracts and upgradeable contracts, allowing developers to easily create, deploy, and test upgradeable smart contracts. 

The repository features a pre-configured dev container setup to ensuring consistency across environments. 
A sample upgradeable ERC20 contract and corresponding test cases are included to serve as a blueprint on how to use the setup.

## Sample Contracts 

The repository provides two sample Solidity smart contracts in subfolder `src`.

- **Counter**: A very simple smart contract
- **MultiSigWallet**: A simple multi sig wallet contract from Cyfrin
- **Token**: An upgradeable token contract using OpenZeppelin V5

Corresponding tests are located in subfolder `test`.

## Preparation

To use this repository the following software is required:

- Git
- Docker
- VSCode

## Initial Setup

1. Clone repository into directory of your choice
1. Open VSCode in this directory
1. Run Devcontainer setup


### Git Submodules Checkout

The project makes use of Git submodules for dependencies.
After cloning the repository also checkout the submodules using the command shown below. 

```shell
git submodule update --init --recursive
```

To update the submodules, use the following command.

```shell
git submodule update --recursive
```

## Using Foundry

### Forge Format

Formats Solidity files to common standard.
The command `forge fmt --check` should not show any errors before pushing to Github.

```shell
$ forge fmt
$ forge fmt --check
```

### Forge Build

```shell
$ forge build
$ forge build --sizes
```

### Forge Test

Runs tests. 
The command `forge test` should not show any errors before pushing to Github.

```shell
forge test
forge test --mt test_Token
forge test --gas-report
forge coverage
```

### Contract Storage Layout

Check storage layout for classical (non upgradeable) contracts.

```
forge clean
forge build
forge inspect MultiSigWallet storage
```

The storage layout of upgradeable contracts like `Token` cannot be inspected directly. 
In addition, OpenZeppelin V5, the namespace storage layout is not yet supported by `forge inspect`, see [foundry issue #7662](https://github.com/foundry-rs/foundry/issues/7662).

A partial workaround is shown below (does not show struct organization).

```
forge inspect ERC20StorageInspector storage
```

## Interact with Deployed Contracts

### Start a Local Anvil Chain

1. Open a new terminal
1. Reset the `.env` file with a newly created account
1. Unset old env variables
1. Set new mnemonic variable
1. Start `anvil` with it

```shell
uv run python3 app/init.py > .env
unset ETH_ADDRESS ETH_PRIVATE_KEY ETH_MNEMONIC
export ETH_MNEMONIC=$(grep ETH_MNEMONIC .env | cut -d\" -f2)
anvil --mnemonic $ETH_MNEMONIC
```

### Deploy Token Contract to Local Anvil

1. Open a new shell
1. Run deploy script with `forge script` using the new private key in the `.env` file
1. Record the address of the newly deployed contract (used as `<contract-address>` below)

```shell
forge script script/UpgradeableToken.s.sol --fork-url http://127.0.0.1:8545 --broadcast --private-key $(grep ETH_PRIVATE_KEY .env | cut -d= -f2)
```

### Interact with the Token Contract

1. set the `ETH_MNEMONIC` variable using the `.env` file
1. Start a python shell

```shell
eval $(echo export $(grep MNEMONIC .env))
cd app
uv run python
```

Follow the steps in the python shell below.

```python
import os

from web3 import Web3
from web3utils.node import Node
from web3utils.contract import Contract
from web3utils.wallet import Wallet

# verify that w3_uri matches with your anvil chain
w3_uri = "http://127.0.0.1:8545"
w3 = Web3(Web3.HTTPProvider(w3_uri))
node = Node(w3)

# print network info
node.chain_id
node.latest_block
node.timestamp

# create account using the env variable mnemonic
w = Wallet.from_mnemonic(w3, os.getenv('ETH_MNEMONIC'))
a = Wallet.create(w3)

# transfer 1 eth and check balance
w.transfer(a, 10**18)
a.balance()/10**18

# offline tx creation and signing
tx = w.transfer(a, 1, sign_and_send=False)
tx_singed = w.sign(tx)
tx_hash = w.send(tx_singed)

# create token object
token_address = "<contract-address>"
token = Contract(w3, "IERC20Metadata", token_address)

# print token balance of deployer (in full tokens)
token.name()
token.balanceOf(w)/10**token.decimals()

# transfer some tokens
tx_hash = token.transfer(a, 42*10**token.decimals(), {'from':w})

# offline contract tx creation and signing
tx = token.transfer(a, 42, {'nonce': None})
tx_signed = w.sign(tx)
tx_hash = w.send(tx_signed)

# extract logs example
# for filter parameter see https://ethereum.org/en/developers/docs/apis/json-rpc/#eth_getlogs 
logs = token.get_logs({'fromBlock':'latest'})
la = token.contract.events.Transfer.process_log(logs[0])
la.args
la.address
la.blockNumber
```
