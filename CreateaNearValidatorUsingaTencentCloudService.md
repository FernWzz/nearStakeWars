# Create a Near Validator Using a Tencent Cloud Service

The NEAR Protocol is a decentralized application platform designed to enable applications with similar usability as they are on the web today. The network runs on a proof-of-stake (PoS) consensus mechanism called Nightshade, designed to provide dynamic scalability and stable fee rates.

NEAR is the native utility token within the network, which has the following use cases:

- Fees for processing transactions and storing data.

- Stake NEAR tokens to run network validators.

- Used for network governance voting to decide how network resources are allocated and the future technical direction of the protocol.



The NEAR protocol creates a range of tools for developers and users, including:

- NEAR SDKs: A complete SDK toolkit including standard data structures, examples and test tools for Rust and AssemblyScript.

- NEAR Gitpod: NEAR uses Gitpod to create a great onboarding experience for developers.

- NEAR Wallet: A wallet reference implementation designed to work seamlessly with a progressive security model that enables application developers to design more efficient user experiences.

- NEAR Browser: Assists in contract debugging and understanding of network performance.

- NEAR Console Tools: A set of simple command line tools that allow developers to easily create, test and deploy applications from their local environment.

This article will introduce how to use Tencent Cloud to create and run a near validator

## 1. Use the web plugin to create a near wallet
- open the link (https://wallet.shardnet.near.org/);
- You can import or create a new wallet in the avatar position in the upper right corner of the web page;
- If you are a new user, you can choose to create a wallet. To create a wallet, you can use your email address, mobile phone number, annotated words and Ledger hardware wallet. For Web3 users who are more accustomed to the latter two methods, this article will use the mnemonic method to create wallets;
- Then you need to create a unique Account ID, which has a unified format AccountName.shardnet.near;
- After creating a unique Account ID, a set of 12-word mnemonics will be generated, which need to be stored properly. Owning the mnemonic is equivalent to using it for all near tokens in the account;
- In order to ensure that the user has saved the mnemonic, the web wallet will randomly check a mnemonic;
- Finally, enter the mnemonic, and you can successfully create a wallet. Note that the wallet address created on this page is for testing, and the near in this address is only for testing and has no actual value;

![Alt](https://github.com/FernWzz/nearStakeWars/blob/main/picture/wallet.jpg)

## 2. Create a validator
- This tutorial uses ubuntu 20.04 to run the validator, to make sure the program uses the latest version, update the server with the command
```bash
sudo apt update && sudo apt upgrade -y
````

- Since near-cli is installed using npm, you first need to install node, this tutorial uses node 18.x
```bash
curl -sL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install build-essential nodejs
PATH="$PATH"
````

Use `node -v` to check node version and `npm -v` to check npm version

![Alt](https://github.com/FernWzz/nearStakeWars/blob/main/picture/nodeVersion.jpg)

- install near-cli
```bash
sudo npm install -g near-cli
````

![Alt](https://github.com/FernWzz/nearStakeWars/blob/main/picture/nearCliVersion.jpg)

Use `near proposals` to view all validators in the proposal state, `near validators current` to view all current validators, and `near validators next` to view nodes that will become validators

- Check whether the server configuration meets the requirements, the recommended configuration for the test network is 4-Core CPU with AVX support/8GB DDR4/500G SSD
```bash
lscpu | grep -P '(?=.*avx )(?=.*sse4.2 )(?=.*cx16 )(?=.*popcnt )' > /dev/null \
  && echo "Supported" \
  || echo "Not supported"
````

- Install dev tools, Python pip and rust
```bash
# install development tools
sudo apt install -y git binutils-dev libcurl4-openssl-dev zlib1g-dev libdw-dev libiberty-dev cmake gcc g++ python docker.io protobuf-compiler libssl-dev pkg-config clang llvm cargo

# install python3-pip
sudo apt install python3-pip

# Configure environment variables
USER_BASE_BIN=$(python3 -m site --user-base)/bin
export PATH="$USER_BASE_BIN:$PATH"

# Configure the development environment
sudo apt install clang build-essential make

# Install rust, use the default installation method
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
# configure rust environment variables
source $HOME/.cargo/env
````

- Download near source code
```bash
git clone https://github.com/near/nearcore
cd nearcore
git fetch

# switch to the latest branch
git checkout <commit>

# compile source code
cargo build -p neard --release --features shardnet
````

- Initialize the near node
```bash
./target/release/neard --home ~/.near init --chain-id shardnet --download-genesis
````
The `--home` parameter specifies the initialization file location and data storage location. It is recommended to use an expandable magnetic field to store block data.

- Start the near full node
```bash
cd ~/nearcore
./target/release/near --home ~/.near run
````

- Authorize the wallet
```bash
near login
````
This step will generate a link, which needs to be opened in the browser. Those who use cloud services can open the link locally by modifying the ip address in the url to complete the authorization.

![Alt](https://github.com/FernWzz/nearStakeWars/blob/main/picture/authorizeWallet.jpg)

- Create validator_key.json
```bash
near generate-key <pool_id>
````
This command creates `YOUR_WALLET.json` under `~/.near-credentials/shardnet/`, you can use the command `cp ~/.near-credentials/shardnet/YOUR_WALLET.json ~/.near/validator_key.json` Copy to `.near` directory
- Use the daemon to start the node, the server file can refer to
```bash
[Unit]
Description=NEARd Daemon Service

[Service]
Type=simple
User=<USER>
#Group=near
WorkingDirectory=/home/<USER>/.near
ExecStart=/home/<USER>/nearcore/target/release/near run
Restart=on-failure
RestartSec=30
KillSignal=SIGINT
TimeoutStopSec=45
KillMode=mixed

[Install]
WantedBy=multi-user.target
````

- Deploy the Staking Pool contract
```bash
near call factory.shardnet.near create_staking_pool '{"staking_pool_id": "<pool id>", "owner_id": "<accountId>", "stake_public_key": "<public key>", "reward_fee_fraction": {"numerator" : 5, "denominator": 100}, "code_hash":"DD428g9eqLL8fWUxv8QSpVFzyHi1Qd16P8ephYCTmMSZ"}' --accountId="<accountId>" --amount=30 --gas=300000000000000
````

![Alt](https://github.com/FernWzz/nearStakeWars/blob/main/picture/deploy.jpg)

- make a mortgage
```bash
near call <staking_pool_id> deposit_and_stake --amount <amount> --accountId <accountId> --gas=300000000000000
````
Domestic cloud services sometimes have problems with linking nodes, but the nodes will automatically reconnect. If there is a network request error, you can wait patiently.

- De-collateralize
```bash
near call <staking_pool_id> unstake '{"amount": "<amount yoctoNEAR>"}' --accountId <accountId> --gas=300000000000000
````

- Redeem all proceeds
```bash
near call <staking_pool_id> withdraw_all --accountId <accountId> --gas=3000000000000000
````

## 3. Monitoring program
- Install monitoring program
```bash
sudo apt install curl jq
# View monitor version
curl -s http://127.0.0.1:3030/status | jq .version
````

![Alt](https://github.com/FernWzz/nearStakeWars/blob/main/picture/check.jpg)

- View delegation and pledge information
```bash
near view <your pool>.factory.shardnet.near get_accounts '{"from_index": 0, "limit": 10}' --accountId <accountId>.shardnet.near
````
- View generated block information
```bash
curl -s -d '{"jsonrpc": "2.0", "method": "validators", "id": "dontcare", "params": [null]}' -H 'Content-Type: application/json ' 127.0.0.1:3030 | jq -c '.result.current_validators[] | select(.account_id | contains ("POOL_ID"))'
````
- See why a node was kicked
```bash
curl -s -d '{"jsonrpc": "2.0", "method": "validators", "id": "dontcare", "params": [null]}' -H 'Content-Type: application/json ' 127.0.0.1:3030 | jq -c '.result.prev_epoch_kickout[] | select(.account_id | contains ("<POOL_ID>"))' | jq .reason
````
