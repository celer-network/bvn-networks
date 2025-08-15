# Validator Manual

This manual describes the process of spinning up a BVN node and joining the network as a validator.

**NOTE**: _The Ethereum account to join BVN must be a bonded validator of SGN. This doc assumes the operator already has access to the bonded SGN validator key._

## Table of Contents
- [Prepare machine and install dependencies](#prepare-machine-and-install-dependencies)
- [Setup binary, config and accounts](#setup-binary-config-and-accounts)
- [Run validator with systemd](#run-validator-with-systemd)
- [Register and bond validator](#register-and-bond-validator)

If you only intend to run an BVN (witness) node to sync and verify the blocks, stop after [run node with systemd](#run-validator-with-systemd).

## Prepare machine and install dependencies

We run on Ubuntu Linux amd64 with Amazon EC2 as an example. Feel free to experiment with other VPS or physical server setups on your own.

1. Start an EC2 machine with the Ubuntu 20.04 LTS image. We recommend using `c6a.2xlarge` if available in the region or `c5.2xlarge` (8 vCPUs, 16GB RAM and 10Gbps network bandwidth) with an EBS volume of at least 500GB. Use the appropriate security groups and a keypair that you have access to.

2. Install go (at least 1.16):

    ```sh
    sudo snap install go --classic
    ```

    Install gcc and make :

    ```sh
    sudo apt update && sudo apt install gcc make
    ```

    (Optional) If you are on Ubuntu and choose to use cleveldb, install libleveldb-dev.
    NOTE: Even though cleveldb has been reported to be more performant, we have
    observed possible memory leak and DO NOT recommend using it.

    ```sh
    sudo apt install libleveldb-dev
    ```

3. Set \$GOBIN and add \$GOBIN to \$PATH. Edit `$HOME/.profile` and add:

    ```sh
    export GOBIN=$HOME/go/bin; export GOPATH=$HOME/go; export PATH=$PATH:$GOBIN
    ```

    to the end, then:

    ```sh
    source $HOME/.profile
    ```

4. (Optional) Install geth:

    ```sh
    sudo add-apt-repository -y ppa:ethereum/ethereum
    sudo apt-get update
    sudo apt-get install ethereum
    ```

## Setup binary, config and accounts

1. From the `/home/ubuntu` directory, download and install the `bvnd` & `mockplugin.so` binaries:

    ```sh
    curl -L https://github.com/celer-network/bvn-networks/releases/download/v1.0.2/bvnd-v1.0.2-goleveldb-linux-amd64.tar.gz | tar -xz
    # To use with cleveldb on Ubuntu, download https://github.com/celer-network/bvn-networks/releases/download/v1.0.2/bvnd-v1.0.2-cleveldb-ubuntu-linux-amd64.tar.gz
    mv bvnd $GOBIN
    mv mockplugin.so $GOBIN
    ```

2. From the `/home/ubuntu` directory, clone the `bvn-networks` repository:

    ```sh
    git clone https://github.com/celer-network/bvn-networks
    cd bvn-networks/bvn
    ```

3. Copy the config files:

    ```sh
    mkdir -p $HOME/.bvnd/config
    cp * $HOME/.bvnd/config
    ```

4. Initialize the new validator node:

    `node-name` is a name you specify for the node.

    ```sh
    # Remove existing genesis.json first
    rm $HOME/.bvnd/config/genesis.json
    # Initialize default genesis.json and config.toml
    bvnd init <node-name> --chain-id bvn --home $HOME/.bvnd
    # Overwrite genesis.json and config.toml with the ones from bvn-networks
    cp genesis.json config.toml $HOME/.bvnd/config
    # Create an empty Tendermint snapshots directory
    mkdir -p $HOME/.bvnd/data/snapshots
    ```

    Backup the generated Tendermint key files `$HOME/.bvnd/config/node_key.json` and `$HOME/.bvnd/config/priv_validator_key.json` securely. Make sure the keys are **never** committed to any repo.

5. Fill out the fields in the Tendermint config file `$HOME/.bvnd/config/config.toml` with the correct values:

    | Field | Description |
    | ----- | ----------- |
    | moniker | The `node-name` you decided |
    | external_address| `<public-ip:26656>`, where `public-ip` is the public IP of the machine hosting the node |
    | db_backend | `goleveldb` or `cleveldb` depending on the binary used |

    Currently, the Celer foundation nodes restrict the access to port 26656-26657, so please **report your public IP to the Celer team** to get whitelisted.

6. Add a Cosmos SDK / Tendermint validator account:

    ```sh
    bvnd keys add <node-name> --keyring-backend file --keyring-dir $HOME/.bvnd
    ```

    Input a passphrase for the keyring. Backup the passphrase along with the displayed mnemonic phrase securely. Make sure they are **never** committed to any repo.

    To view the account created, run:

    ```sh
    bvnd keys list --keyring-backend file --keyring-dir $HOME/.bvnd
    ```

    Make a note of the **bvn-prefixed account address**.

7. Prepare an Ethereum gateway URL. You can use services like [Infura](https://infura.io/),
[Alchemy](https://alchemyapi.io/) or run your own node.

8. Prepare validator Ethereum address, which is the same address with the bonded SGN validator. Copy the validator keystore json file from SGN to BVN 
    ```sh
      mkdir $HOME/.bvnd/eth-ks
      cp <path-to-keystore-json> $HOME/.bvnd/eth-ks/val.json
    ```

9. Prepare an Ethereum key as the **signer key**, which will be used for signing cross-chain transactions and needs to stay online. You can either reuse the signer key from your SGN validator, or create a new key by following the same instructions in the [SGN validator manual](https://github.com/celer-network/sgn-v2-networks/blob/main/docs/validator.md#setup-binary-config-and-accounts).

10. Fill out the fields in the BVN-specific config file `$HOME/.bvnd/config/bvn.toml` with the correct values:

    | Field | Description |
    | ----- | ----------- |
    | eth.gateway | The Ethereum gateway URL obtained from step 7 |
    | eth.signer_keystore | The path to the signer Ethereum keystore file in step 9, or the format required by AWS KMS setup |
    | eth.signer_passphrase | The passphrase of the signer keystore, or the format required by AWS KMS setup |
    | eth.validator_address | The **Ethereum address** of the validator key prepared in step 8 |
    | bvnd.passphrase | The **Cosmos keyring passphrase** you typed in step 6 |
    | bvnd.validator_account | The **bvn-prefixed validator Cosmos SDK account** added in step 6 |

11. Fill in the missing gateway URLs in `$HOME/.bvnd/config/multichain.toml` with the corresponding JSON-RPC URLs for the chains. In general, we recommend using paid provider services instead of the public endpoints for better reliability.

## Run validator with systemd

We recommend using systemd to run your validator. Feel free to experiment with other setups on your own.

1. Prepare the bvnd system service:

    ```sh
    sudo mkdir -p /var/log/bvnd
    sudo touch /var/log/bvnd/tendermint.log
    sudo touch /var/log/bvnd/app.log
    sudo touch /etc/systemd/system/bvnd.service
    ```

    Add the following to `/etc/systemd/system/bvnd.service`:

    ```
    [Unit]
    Description=BVN daemon
    After=network-online.target

    [Service]
    Environment=HOME=/home/ubuntu
    ExecStart=/home/ubuntu/go/bin/bvnd start \
      --home /home/ubuntu/.bvnd/
    StandardOutput=append:/var/log/bvnd/tendermint.log
    StandardError=append:/var/log/bvnd/app.log
    Restart=always
    RestartSec=3
    User=ubuntu
    Group=ubuntu
    LimitNOFILE=4096

    [Install]
    WantedBy=multi-user.target
    ```

    With this setup, `tendermint.log` contains Tendermint logs (i.e. block height, peer info, etc.) while `app.log`
    contains BVN-specific logs.

2. Create `/etc/logrotate.d/bvn` and add the following:

    ```
    /var/log/bvnd/*.log {
        compress
        copytruncate
        daily
        maxsize 30M
        rotate 30
    }
    ```

3. Add an entry to `/etc/crontab` to make logrotate run every 6 hours:

    ```
    30 */6  * * *   root    logrotate -f /etc/logrotate.conf
    ```

4. Currently, we require using Tendermint state sync to sync your node. Follow the instructions in [this doc](state_sync.md) to prepare the node
receiving the snapshot with `up-to-date-node-ip`s taken from the `seeds` field in `$HOME/.bvnd/config/config.toml`. Stop short of starting the node.

5. Enable and start the bvnd service:

    ```sh
    sudo systemctl enable bvnd.service
    sudo systemctl start bvnd.service
    ```

    Now the node should start the state sync. Monitor `tendermint.log` for the progress:

    ```sh
    tail -f /var/log/bvnd/tendermint.log
    ```

    You can tell the node is synced when a new block shows up about every 5 seconds.

6. If you choose not to setup state sync, the node will perform a traditional "fast sync" instead.
In this mode it replays and verifies all historical transactions starting from genesis.

## Register and bond validator

1. Regsiter BVN validator by calling [registerBvnValidator](https://github.com/celer-network/sgn-v2-contracts/blob/0dce0d165cb09bba2092709968561bebfd29a287/contracts/staking/restaking/BvnRestaking.sol#L55) on the `BvnRestaking` contract

    - **For validator key on local keystore JSON file**: use command line

      ```sh
      bvnd ops validator register --keystore ~/.bvnd/eth-ks/val.json --passphrase <val-ks-passphrase>
      ```

    - **For validator key on MetaMask / hardware wallet**: call the `registerBvnValidator` method through web explorer.

      To find the `_bvnAddr` value, run the command `bvnd ops validator address`, user the output value after bvn acct address in hex.

2. Update description and verify the status:

    ```sh
    echo | bvnd tx staking edit-description --website "your-website" --contact "email-address"
    bvnd query staking validator <val-eth-address>
    ```
