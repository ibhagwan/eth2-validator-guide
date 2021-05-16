## Objective

### Running a validator on a low-cost VPS on Void Linux (`runit`)

At the most basic level, an Eth2 validator runs 3 services:

- Eth1 client (geth, openetereum, besu, nethermind)
- Eth2 client (prysm, lighthouse, teku, nimbus)
    + Beacon node
    + Validator client

Running a node at home is great and probably even better for decentralization but unfortunately I don't trust my provider to not have the occasional outage. Although missing a few attestations here and there isn't a big deal, I am aiming for 100% uptime, so I chose to host my server on a [vultr.com high compute](https://www.vultr.com/products/high-frequency-compute) instance at the [$48/mo tier](https://www.vultr.com/products/high-frequency-compute/#pricing) which provided me with 3 CPUs, 8GB RAM, 256GB NVMe and 4GB of monthly bandwidth.

Early on in my research I realized the rather large storage requirements for the Eth1 client isn't going to work for my 256GB storage VPS which is currently at ~1TB and soon may even surpass that. That amount of storage lease can quickly ramp up in price and can easily cost > $500.

Although I could choose lower quality hosting for cheap storage, I didn't want to compromise on C-level hosting that uses HDDs and other sub-optimal setups so I decided to "outsource" my Eth1 client services to one of the [many Eth1 providers](https://ethereumnodes.com).

Theoretically this approach had one pitfall: relying on a Eth1 provider you have no control of, as proven by the now famous [Infura Nov 2020 outage](https://blog.infura.io/infura-mainnet-outage-post-mortem-2020-11-11/). Thankfully this was easily solved using [`lighthouse` built-in redundancy](https://lighthouse-book.sigmaprime.io/redundancy.html) using 2 or more Eth1 providers.

Lastly, I wanted to run my validator on a minimal linux distribution ([Void Linux](https://voidlinux.org) was my choice) which uses `runit` rather than `systemd` (Ubuntu) as most guides out there recommend. This guide is a result of my note taking during the process. Note that **there is absolutely nothing wrong with Ubuntu or `systemd`**, this is simply a personal preference.

Special thanks to all the other guides and especially [**/u/SomerEsat**](https://www.reddit.com/user/SomerEsat/) who's guides I consulted and served as an inspiration to this guide.


## Index

1. [Requirements](#requirements)
2. [Create Eth1 node cloud accounts](#create-eth1-node-cloud-accounts)
3. [Generate Eth2 keystore](#generate-eth2-keystore)
4. [Install Eth2 client](#install-eth2-client)
    1. [Create the runit beacon node service](#create-the-runit-beacon-node-service)
    2. [Import validator keys](#import-validator-keys)
    3. [Create the runit validator client service](#create-the-runit-validator-client-service)
5. [Deposit ETH into the validator](#deposit-eth-into-the-validator)

## Appendix

- [Verifying your mnemonic](#verifying-your-mnemonic)
- [Adding validators](#adding-validators)
- [Monitoring your validators](#monitoring-your-validators)
- [Validator effectiveness](#validator-effectiveness)
- [Slashing protection](#slashing-protection)
- [Migrating a node](#migrating-a-node)
- [Using the lighthouse wallet](#using-the-lighthouse-wallet)
    + [Recovery using the lighthouse wallet](#recovery-using-the-lighthouse-wallet)
    + [Add validators using the lighthouse wallet](#add-validators-using-the-lighthouse-wallet)
- [How to get Goerli ETH](#how-to-get-goerli-eth)
- [Enable `ntpd` on Void Linux](#enable-ntpd-on-void-linux)

## <a id="requirements">Requirements</a>

- Basic knowledge of Linux OS and using the terminal
- A server running a Linux distribution with the [`runit` initialization system](https://wiki.archlinux.org/title/Runit)
- A cloud VPS with minimum requirements:
    + 2 CPUs
    + 8GB RAM
    + 128 GB SSD
-  Time-server synchronization service enabled (see [how to enable `ntpd` on Void Linux](#enable-ntpd-on-void-linux)).

> **Note:** This guide **does not cover** the Linux installation, communication with your VPS using `ssh` or firewall setup.

## <a id="create-eth1-node-cloud-accounts">Create Eth1 node cloud accounts</a>

Create accounts in the provider(s) of your choice (I recommend at least 2 providers for Eth1 node redundancy), create your Eth1 nodes and write down the appropriate endpoints:

| Provider | Model | Limits | Geolocation |
| --- | --- | --- | --- |
|[**infura.io**](https://infura.io/)|Freemium|100k requests per day|AWS Ashburn VA|
|[**alchemyapi.io**](https://alchemyapi.io/)|Freemium|25M "compute units" per month (A node is ~1M units/day)|AWS Ashburn VA|
|[**rivet.cloud**](https://rivet.cloud)|Freemium|3M requests per month|AWS Columbus OH|
|[**chainstack.com**](https://chainstack.com)|Freemium, Requires CC|3M/mo, 14 days/mo + pay as you go|Google Singapore|
|[**anyblock.tools**](https://anyblock.tools)|Freemium|120 calls per hour|Cloudflare San Francisco CA|
|[**quicknode.io**](https://quicknode.io)|Paid|$9/mo 300k - $99/mo 20M||
|[**archivenode.io**](https://archivenode.io)|Free, requires social media verification||

> **Notes:**
>
>  - The free account in Infura daily limit of 100k requests is sufficient to run a node
>  - The free account in Alchemy has a limit of 25M "compute units", running a node utilizes about 1M units daily requests thus the free account should last about 25 days after which it refuses to answer requests
>  - **I personally run Infura as the main EP and Alchemy / Rivet as a backups, since Alchemy is only used as a backup EP it does not require more than the free tier on both providers.**
>  - For more free Eth1 node alternatives checkout [**https://ethereumnodes.com/**](https://ethereumnodes.com/).

### Examples of how to test your Eth1 nodes:

<details><summary><b>Alchemy</b></summary>

```console
❯ curl -X POST -H "Content-Type: application/json" \
    --data '{"jsonrpc":"2.0","method":"web3_clientVersion","params":[],"id":67}' \
    https://eth-mainnet.alchemyapi.io/v2/<your-alchemy-key>

{"jsonrpc": "2.0", "id": 67, "result": "Geth/v1.10.2-stable-97d11b01/linux-amd64/go1.16.3"}%
```
</details>

<details><summary><b>Infura</b></summary>

```console
❯ curl -X POST -H "Content-Type: application/json" \
    --data '{"jsonrpc":"2.0","method":"web3_clientVersion","params":[],"id":67}' \
    https://mainnet.infura.io/v3/<your-infura-ep>

{"jsonrpc":"2.0","id":67,"result":"Geth/v1.10.1-omnibus-2230b5e6/linux-amd64/go1.16"}%
```
</details>

<details><summary><b>Chainstack</b></summary>

> https://support.chainstack.com/hc/en-us/articles/900004755663-Connecting-your-Chainstack-Eth1-node-to-a-Prysm-Beacon-node
```console
❯ curl -X POST -H "Content-Type: application/json" \
    --data '{"jsonrpc":"2.0","method":"web3_clientVersion","params":[],"id":67}' \
    https://<username>:<password>@<your-chainstack-RPC-ep>

{"jsonrpc":"2.0","id":67,"result":"Geth/v1.10.1-stable-c2d2f4ed/linux-amd64/go1.16"}
```
</details>

## <a id="generate-eth2-keystore">Generate Eth2 keystore</a>

Download the latest `deposit-cli` tool from 
[**https://github.com/ethereum/eth2.0-deposit-cli/releases/**](https://github.com/ethereum/eth2.0-deposit-cli/releases/)

```console
❯ curl -LO https://github.com/ethereum/eth2.0-deposit-cli/releases/download/v1.2.0/eth2deposit-cli-256ea21-linux-amd64.tar.gz
❯ tar xvfz eth2deposit-cli-256ea21-linux-amd64.tar.gz
❯ cd eth2deposit-cli-256ea21-linux-amd64
```

Verify the `sha-256` signatures match:
```console
❯ curl -LO https://github.com/ethereum/eth2.0-deposit-cli/releases/download/v1.2.0/eth2deposit-cli-256ea21-linux-amd64.sha256
❯ cat eth2deposit-cli-256ea21-linux-amd64.sha256
825035b6d6c06c0c85a38f78e8bf3e9df93dfd16bf7b72753b6888ae8c4cb30a%
❯ sha256sum eth2deposit-cli-256ea21-linux-amd64.tar.gz
825035b6d6c06c0c85a38f78e8bf3e9df93dfd16bf7b72753b6888ae8c4cb30a  eth2deposit-cli-256ea21-linux-amd64.tar.gz
```
> The contents of the `.sha256` file should match the output of the `sha256sum` command.

#### Generate the Eth2 mnemonic and validator keystores:

> **Note:** For optimal security run the `deposit-cli` on a clean air-gapped machine.

<details><summary><b>Pyrmont (testnet)</b></summary>

```console
❯ ./deposit new-mnemonic --num_validators 2 --chain pyrmont
```
</details>

<details><summary><b>Prater (testnet)</b></summary>

```console
❯ ./deposit new-mnemonic --num_validators 1 --chain prater
```
</details>

<details><summary><b>Beacon (mainnet)</b></summary>

```console
❯ ./deposit new-mnemonic --num_validators 1 --chain mainnet
```
</details>

The keys will be saved in `./validator_keys` sorted by their index:

```console
❯ ls -la ./validator_keys
.r--r----- 1.4k bhagwan bhagwan 22 Apr 12:36 deposit_data-1619120219.json
.r--r-----  710 bhagwan bhagwan 22 Apr 12:36 keystore-m_12381_3600_0_0_0-1619120217.json
.r--r-----  710 bhagwan bhagwan 22 Apr 12:36 keystore-m_12381_3600_1_0_0-1619120218.json
```

> **DO NOT LOSE THE MNEMONIC:** the mnemonic is used to derive both the validator keys and the withdrawal keys, it can also be used to add validators or restore lost validator files. Read more about Eth2 keys [here](https://www.bloxstaking.com/knowledge-base/guides/validator/what-are-eth2-keys/)
>
> The password is used to encrypt the files but **it is not part of the mnemonic**, the password is later used to import the files into the Eth2 beacon validator client
>
> The `deposit_data-[timestamp].json` file contains the public keys for the validators and information about the deposit. This file will be used to complete the deposit process in the next step. Later on, this file will need to be transferred to the computer running [MetaMask](https://metamask.io/) browser extension in order to fund the validator contracts using the [Ethereum launchapd](https://launchpad.ethereum.org/)
>
> The `keystore-m-[idx]-[timestamp].json` files contain the password encrypted validator signing key. There is one `keystore-m-[idx]-[timestamp].json` per validator. These will later be imported into the Eth2 validator client.

## <a id="install-eth2-client">Install Eth2 client</a>

Download latest release of `lighthouse` from [**https://github.com/sigp/lighthouse/releases**](https://github.com/sigp/lighthouse/releases/)

If running on a cloud VPS, use the `portable` binary:

```console
# curl -LO https://github.com/sigp/lighthouse/releases/download/v1.3.0/lighthouse-v1.3.0-x86_64-unknown-linux-gnu-portable.tar.gz
# tar xvfz lighthouse-v1.3.0-x86_64-unknown-linux-gnu-portable.tar.gz
# sudp cp lighthouse /usr/local/bin/lighthouse
```

Verify `lighthouse` works and is in your `$PATH`:
```sh
❯ rehash
❯ lighthouse --version
Lighthouse v1.3.0-3a24ca5
BLS Library: blst-portable
Specs: mainnet (true), minimal (false), v0.12.3 (false)
```

### <a id="create-the-runit-beacon-node-service">Create the runit beacon node service</a>

Create the `lighthouse` user:

```console
# sudo useradd --no-create-home --shell /bin/false lighthouse
```

<details><summary><b>Pyrmont (testnet)</b></summary>

```console
# mkdir -p /var/lib/lighthouse/pyrmont/beacon
# chmod 700 /var/lib/lighthouse/pyrmont/beacon
# chown -R lighthouse:lighthouse /var/lib/lighthouse/pyrmont/beacon
# mkdir -p /var/log/lighthouse-pyrmont
# echo n3 > /var/log/lighthouse-pyrmont/config
# chown -R lighthouse:lighthouse /var/log/lighthouse-pyrmont
# mkdir /etc/sv/lighthouse-pyrmont
# cat <<EOF > /etc/sv/lighthouse-pyrmont/run
#!/bin/sh
exec chpst -u lighthouse:lighthouse /usr/local/bin/lighthouse beacon_node \
    --network pyrmont --datadir /var/lib/lighthouse/pyrmont --staking \
    --validator-monitor-auto --eth1-blocks-per-log-query 150 \
    --eth1-endpoints https://goerli.infura.io/v3/<your-infura-ep>,https://eth-goerli.alchemyapi.io/v2/<your-alchemy-key> \
    --metrics --port 9002 --http-port 7052 --metrics-port 7054 2>&1
EOF
# cat <<EOF > /etc/sv/lighthouse-pyrmont/log/run
#!/bin/sh
exec chpst -u lighthouse:lighthouse svlogd -tt /var/log/lighthouse-pyrmont
EOF
# chmod a+x /etc/sv/lighthouse-pyrmont/run
# chmod a+x /etc/sv/lighthouse-pyrmont/log/run
# ln -s /etv/sv/lighthouse-pyrmont /var/service
```
> **Notes:**
>
> - Modify the above with your own Infura/Alchemy endpoints.
> - To be able to use both the testnet and mainnet in parallel we use different ports using the `--port` flags, make note of the `--http-port` as we will use it in our validator service setup.

</details>

<details><summary><b>Prater (testnet)</b></summary>

```console
# mkdir -p /var/lib/lighthouse/prater/beacon
# chmod 700 /var/lib/lighthouse/prater/beacon
# chown -R lighthouse:lighthouse /var/lib/lighthouse/prater/beacon
# mkdir -p /var/log/lighthouse-prater
# echo n3 > /var/log/lighthouse-prater/config
# chown -R lighthouse:lighthouse /var/log/lighthouse-prater
# mkdir /etc/sv/lighthouse-prater
# cat <<EOF > /etc/sv/lighthouse-prater/run
#!/bin/sh
exec chpst -u lighthouse:lighthouse /usr/local/bin/lighthouse beacon_node \
    --network prater --datadir /var/lib/lighthouse/prater --staking \
    --validator-monitor-auto --eth1-blocks-per-log-query 150 \
    --eth1-endpoints https://goerli.infura.io/v3/<your-infura-ep>,https://eth-goerli.alchemyapi.io/v2/<your-alchemy-key> \
    --metrics --port 9001 --http-port 6052 --metrics-port 6054 2>&1
EOF
# cat <<EOF > /etc/sv/lighthouse-prater/log/run
#!/bin/sh
exec chpst -u lighthouse:lighthouse svlogd -tt /var/log/lighthouse-prater
EOF
# chmod a+x /etc/sv/lighthouse-prater/run
# chmod a+x /etc/sv/lighthouse-prater/log/run
# ln -s /etv/sv/lighthouse-prater /var/service
```
> **Notes:**
>
> - Modify the above with your own Infura/Alchemy endpoints.
> - To be able to use both the testnet and mainnet in parallel we use different ports using the `--port` flags, make note of the `--http-port` as we will use it in our validator service setup.
</details>

<details><summary><b>Beacon (mainnet)</b></summary>

```console
# mkdir -p /var/lib/lighthouse/mainnet/beacon
# chmod 700 /var/lib/lighthouse/mainnet/beacon
# chown -R lighthouse:lighthouse /var/lib/lighthouse/mainnet/beacon
# mkdir -p /var/log/lighthouse-mainnet
# echo n3 > /var/log/lighthouse-mainnet/config
# chown -R lighthouse:lighthouse /var/log/lighthouse-mainnet
# mkdir /etc/sv/lighthouse-mainnnet
# cat <<EOF > /etc/sv/lighthouse-mainnnet/run
exec chpst -u lighthouse:lighthouse /usr/local/bin/lighthouse beacon_node \
    --network mainnet --datadir /var/lib/lighthouse/mainnet --staking \
    --validator-monitor-auto --eth1-blocks-per-log-query 150 \
    --eth1-endpoints https://mainnet.infura.io/v3/<your-infura-ep>,https://eth-mainnet.alchemyapi.io/v2/<your-infura-key> \
    --metrics 2>&1
EOF
# cat <<EOF > /etc/sv/lighthouse-mainnnet/log/run
#!/bin/sh
exec chpst -u lighthouse:lighthouse svlogd -tt /var/log/lighthouse-mainnnet
EOF
# chmod a+x /etc/sv/lighthouse-mainnet/run
# chmod a+x /etc/sv/lighthouse-mainnet/log/run
# ln -s /etv/sv/lighthouse-mainnet /var/service
```
> **Notes:**
>
> - Modify the above with your own Infura/Alchemy endpoints.

</details>

> **Notes:**
>
> - If everything went as expected the `beacon_node` service should be started now, you can view it's status by running `sv status lighthouse-<network>` and inspecting the log at `/var/log/lighthouse-<network>/current`.
> - infura.io Eth1 reply is limited to 10k records, this may become an issue with large `eth_getLogs` requests (e.g. deposit contract logs). A workaround is to limit the per-request log limit with the `--eth1-blocks-per-log-query` flag. More info can be found in [RocketPool smartnode issue #20](https://github.com/rocket-pool/smartnode-install/issues/20) and [#29](https://github.com/rocket-pool/smartnode-install/pull/29).
> - Adding `--validator-monitor-auto` to the `beacon_node` client adds automatic monitoring for every `vaidator_client` using the endpoint, more info can be found in the [lighthouse book: Validator Monitoring section](https://lighthouse-book.sigmaprime.io/validator-monitoring.html).


### <a id="import-validator-keys">Import validator keys</a>

Copy the `validator_keys` folder (containing the `keystore-m…json` files) to the server running lighthouse and import the validator(s):

<details><summary><b>Pyrmont (testnet)</b></summary>

```console
# lighthouse --network pyrmont account validator import --datadir /var/lib/lighthouse/pyrmont \
    --directory <PathToValidatorKeys>
```
</details>

<details><summary><b>Prater (testnet)</b></summary>

```console
# lighthouse --network prater account validator import --datadir /var/lib/lighthouse/prater \
    --directory <PathToValidatorKeys>
```
</details>

<details><summary><b>Beacon (mainnet)</b></summary>

```console
# lighthouse --network mainnet account validator import --datadir /var/lib/lighthouse/mainnet \
    --directory <PathToValidatorKeys>
```
</details>

> **Notes:**
>
> - When running the above command `lighthouse` will request the password previously specified with the `deposit-cli` tool.
> - `lighthouse` will import the keys into the `validators` folder under the `--datadir path` (`/var/lib/lighthouse/<network>/validators`). Password will be prompted once for each validator.
> - Passwords are stored as **CLEAR TEXT** inside `.../validators/validator_definitions.yml`, to avoid that you can recreate the validator files from the mnemonic using the `lighthouse account validator recover` as detailed in [using lighthouse wallet](#using-lighthouse-wallet).

### <a id="create-the-runit-validator-client-service">Create the runit validator client service</a>

Create the `validator` user:
```console
# useradd --no-create-home --shell /bin/false validator
```

<details><summary><b>Pyrmont (testnet)</b></summary>

```console
# chown -R validator:validator /var/lib/lighthouse/pyrmont/validators
# mkdir -p /var/log/validator-pyrmont
# echo n3 > /var/log/validator-pyrmont/config
# chown -R validator:validator /var/log/validator-pyrmont
# cat <<EOF > /etc/sv/validator-pyrmont/run
#!/bin/sh
exec chpst -u validator:validator /usr/local/bin/lighthouse validator_client \
    --network pyrmont --datadir /var/lib/lighthouse/pyrmont \
    --beacon-nodes http://localhost:7052 --graffiti "<your-graffiti>" 2>&1
EOF
# cat <<EOF > /etc/sv/validator-pyrmont/log/run
#!/bin/sh
exec chpst -u validator:validator svlogd -tt /var/log/validator-pyrmont
EOF
# chmod a+x /etc/sv/validator-pyrmont/run
# chmod a+x /etc/sv/validator-pyrmont/log/run
# ln -s /etv/sv/validator-pyrmont /var/service
```
> **Note:** `--beacon-nodes` should specify the port we set using `--http-port` in our beacon node service earlier in the guide.

</details>

<details><summary><b>Prater (testnet)</b></summary>

```console
# chown -R validator:validator /var/lib/lighthouse/prater/validators
# mkdir -p /var/log/validator-prater
# echo n3 > /var/log/validator-prater/config
# chown -R validator:validator /var/log/validator-prater
# cat <<EOF > /etc/sv/validator-prater/run
#!/bin/sh
exec chpst -u validator:validator /usr/local/bin/lighthouse validator_client \
    --network prater --datadir /var/lib/lighthouse/prater \
    --beacon-nodes http://localhost:6052 --graffiti "<your-graffiti>" 2>&1
EOF
# cat <<EOF > /etc/sv/validator-prater/log/run
#!/bin/sh
exec chpst -u validator:validator svlogd -tt /var/log/validator-prater
EOF
# chmod a+x /etc/sv/validator-prater/run
# chmod a+x /etc/sv/validator-prater/log/run
# ln -s /etv/sv/validator-prater /var/service
```
> **Note:** `--beacon-nodes` should specify the port we set using `--http-port` in our beacon node service earlier in the guide.

</details>

<details><summary><b>Beacon (mainnet)</b></summary>

```console
# chown -R validator:validator /var/lib/lighthouse/mainnnet/validators
# mkdir -p /var/log/validator-mainnnet
# echo n3 > /var/log/validator-mainnnet/config
# chown -R validator:validator /var/log/validator-mainnnet
# cat <<EOF > /etc/sv/validator-mainnnet/run
#!/bin/sh
exec chpst -u validator:validator /usr/local/bin/lighthouse validator_client \
    --network mainnnet --datadir /var/lib/lighthouse/mainnnet \
    --graffiti "<your-graffiti>" 2>&1
EOF
# cat <<EOF > /etc/sv/validator-mainnnet/log/run
#!/bin/sh
exec chpst -u validator:validator svlogd -tt /var/log/validator-mainnnet
EOF
# chmod a+x /etc/sv/validator-mainnet/run
# chmod a+x /etc/sv/validator-mainnet/log/run
# ln -s /etv/sv/validator-mainnnet /var/service
```

</details>

> Replace `"<your-graffiti>"` with your own graffiti string. For security and privacy reasons avoid information that can uniquely identify you. E.g. --graffiti "Hello Eth2!". The graffiti will be entered into the beaconchain each time your validator proposes a block

## <a id="deposit-eth-into-the-validator">Deposit ETH into the validator</a>

Copy the `deposit_data-[timestamp].json` we generated using the `deposit-cli` utility to a computer with a browser and MetaMask extension funded with 32ETH per validator and use the Ethereum launchpad link below to complete the process:

<details><summary><b>Pyrmont (testnet)</b></summary>

> This step requires 32 gETH for this step, see [How to get Goerli ETH](#how-to-get-goerli-eth)

[**https://pyrmont.launchpad.ethereum.org/**](https://pyrmont.launchpad.ethereum.org)
</details>

<details><summary><b>Prater (testnet)</b></summary>

> This step requires 32 gETH for this step, see [How to get Goerli ETH](#how-to-get-goerli-eth)

[**https://prater.launchpad.ethereum.org/**](https://prater.launchpad.ethereum.org)
</details>

<details><summary><b>Beacon (mainnet)</b></summary>

[**https://launchpad.ethereum.org/**](https://launchpad.ethereum.org)
</details>

> **Notes:** After the funds are deposited it will take roughly 15-22 hours for our validators to become active, more on the process can be read [here#1](https://kb.beaconcha.in/ethereum-2.0-depositing) and [here#2](https://notes.ethereum.org/@vbuterin/rkhCgQteN?type=view#Depositing).
>
> To check the validator(s) deposit status you can search for the Eth1 sending wallet address in [**beaconcha.in**](https://beaconcha.in) which will result in all deposits made to the Eth2 validator smart contract (thus displaying all your validators)
>
> **Recommended:** read [**launchpad checklist section #3: After depositing**](https://launchpad.ethereum.org/en/checklist/#section-three)

## Appendix
## <a id="verifying-your-mnemonic">Verifying your mnemonic</a>

Since the mnemonic is used to derive both the withdrawal credentials (pubkey) and the validator public key, one can regenrate the `deposit_data-[timestamp].json` and the validator `keystore-m-[idx]-[timestamp].json` files using the `deposit-cli` with the `existing-mnemonic` option.

First step to validate your mnemonic would be to regenerate the first validator file:

```console
❯ ./deposit existing-mnemonic --validator_start_index 0 --num_validators 1 --chain <network>
```

> The `--validator_start_index` defines the validator index starting with index:0 for the first validator, index:0 for the first validator, index:1 for the second, etc.
>
> When using the same number of validators, the generated `deposit_data-[timestamp].json` **should be identical to the original file generated with `new_mnemonic`**, running `diff` on both files should return an empty string.
>
> The `pubkey` fields inside the `deposit_data-[timestamp].json` file contains the public key addresses of your validators on the beacon chain.
>
> Each `keystore-m-[idx]-[timestamp].json` also contains a matching `pubkey` field corresponding the validator index, however, the generated files will not match their originals as many of the parameters inside differ.

## <a id="adding-validators">Adding validators</a>

```console
❯ ./deposit existing-mnemonic --validator_start_index <index> --num_validators <count> --chain <network>
```

> The `--validator_start_index` should be +1 above the existing validator index starting with 0. For example, if we were to add 2 new validators to an existing 1 validator resulting in 3 validators (indexed:0,1,2) we would issue the command with `--validator_start_index 1 --num_validators 2`
>
> The process will generate one `keystore-m-[idx]-[timestamp].json` per validator and a single `deposit_data-[timestamp].json` containing all public keys of the new validators which we will use on the browser with MetaMask to fund our new validators, 
>
> During the process the `deposit-cli` will ask for a password, the password can differ from the original password as it's only used to encrypt the keystore validator data. The password will be required when we import the files into our validator service (i.e. lighthouse, prysm, etc).


The next steps will need to:

1. Copy the newly generated `validator_keys` folder to the server running `lighthouse`
2. Stop the `lighthouse validator_client` service
3. Import the new validator keystore(s)
4. Reset permissions on the imported files
5. Start the `lighthouse validator_client` service
6. Fund the new validators

For step **(1)**, I recommend using `scp` to copy the files between your servers, after the files are copied to the server, we execute steps **(2-5)**:

<details><summary><b>Pyrmont (testnet)</b></summary>

```console
❯ sudo su -
# sv stop validator-pyrmont
# lighthouse --network pyrmont account validator import --datadir /var/lib/lighthouse/pyrmont \
    --directory <PathToValidatorKeys>
# chown -R validator:validator /var/lib/lighthouse/pyrmont/validators
# sv start validator-pyrmont
```
</details>

<details><summary><b>Prater (testnet)</b></summary>

```console
❯ sudo su -
# sv stop validator-prater
# lighthouse --network prater account validator import --datadir /var/lib/lighthouse/prater \
    --directory <PathToValidatorKeys>
# chown -R validator:validator /var/lib/lighthouse/prater/validators
# sv start validator-prater
```
</details>

<details><summary><b>Beacon (mainnet)</b></summary>

```console
❯ sudo su -
# sv stop validator-mainnet
# lighthouse --network mainnet account validator import --datadir /var/lib/lighthouse/mainnet \
    --directory <PathToValidatorKeys>
# chown -R validator:validator /var/lib/lighthouse/mainnet/validators
# sv start validator-mainnet
```
</details>

Verify clean logs at `/var/log/validator-<network>/current`:
```console
INFO Lighthouse started                      version: Lighthouse/v1.3.0-3a24ca5
INFO Configured for network                  name: pyrmont
INFO Starting validator client               validator_dir: "/var/lib/lighthouse/pyrmont/validators", beacon_nodes: ["http://localhost:6052/"]
INFO HTTP metrics server is disabled
INFO Completed validator discovery           new_validators: 0
INFO Enabled validator                       voting_pubkey: <pubkey#0>
INFO Enabled validator                       voting_pubkey: <pubkey#1>
INFO Enabled validator                       voting_pubkey: <pubkey#2>
INFO Modified key_cache saved successfully
INFO Initialized validators                  enabled: 3, disabled: 0
INFO Connected to beacon node                endpoint: http://localhost:6052/, version: Lighthouse/v1.3.0-3a24ca5/x86_64-linux
INFO Initialized beacon node connections     available: 1, total: 1
INFO Loaded validator keypair store          voting_validators: 3
```

After everything is verified working, copy the newly generated `deposit_data-[timestamp].json` file and [**repeat the same ETH deposit steps**](#deposit-eth-into-the-validators).


## <a id="monitoring-your-validators">Monitoring your validators</a>

After depositing the funds using the launchpad, it will take between 15-22 hours to activate your validators, an easy way to view all your validators is to search for the Eth1 sending address in [**beaconcha.in**](https://beaconcha.in).

Once the validator status becomes `active` the validator will be assigned an index number. Once we have an index number, create an account with the beaconcha.in website and add monitoring for all possible scenarios for your validator (attestations missed, balance decreases, etc) which will then send an email alert when the conditions are met.

[**beaconcha.in**](https://beaconcha.in) also has an [iOS app](Ohttps://apps.apple.com/us/app/beaconchain-dashboard/id1541822121) which can send notifications to your mobile device.

[**beaconscan.com**](https://beaconscan.com) is another good option for monitoring your validators made by the folks behind Eth1's [**etherscan.io**](https://etherscan.io).

Use `tail` to actively monitor the lighthouse beacon node and validator:

```console
❯ tail -F /var/log/lighthouse-<network>/current
❯ tail -F /var/log/validator-<network>/current
```
> We use `-F` instead of `-f` to force `tail` to follow the log file upon rotation.

## <a id="validator-effectiveness">Validator effectiveness</a>

Monitor effectiveness is determined by different factors, read more about it below:

- [eth2 insights: validator effectiveness](https://bisontrails.co/eth2-insights-validator-effectiveness/)
- [Defining Attestation Effectiveness](https://www.attestant.io/posts/defining-attestation-effectiveness/)

## <a id="slashing-protection">Slashing protection</a>

Read the below:

- [Lighthouse book: slashing protection](https://lighthouse-book.sigmaprime.io/slashing-protection.html)
- [Prysmatic labs: Eth2 slashing prevention tips](https://medium.com/prysmatic-labs/eth2-slashing-prevention-tips-f6faa5025f50)
- [Blox: Understanding Eth2 slashing & preventative measures](https://www.bloxstaking.com/blog/ethereum-2-0/understanding-eth2-slashing-preventative-measures/)

## <a id="migrating-a-node">Migrating a node</a>

Follow all the steps of this guide on a second server, once the lighthouse beacon node is synchronized you can safely migrate to the new server.

> **In order to prevent [slashing](https://lighthouse-book.sigmaprime.io/slashing-protection.html) it is extremely important not to run duplicate validators**

Verify the validator service is shutdown and disabled on the old server before continuing any further:
```console
# stop the validator service
(old server) # sv stop validator-<network>

# disable the service upon boot
(old server) # rm /var/service/validator-<network>

# make sure the process is killed
(old server) # ps -ef | grep validator_client

# the log file should also indicate a shutdown
(old server) # tail /var/log/validator-<network>/current
```

[Import](#import-validator-keys) or [regenerate](#verifying-your-mnemonic) the validator keystores as explained [earlier in this guide](#generate-eth2-keystore). Alternatively copy the entire `/var/lib/lighthouse/<network>/validators/` folder to the new server.

> **If importing or regenerating the keystores it is important we also copy the slashing protection DB from `/var/lib/ligthouse/<network>/validators/slashing_protection.sqlite` to the new server**.

Verify all conditions are met:
- Lighthouse `beacon_node` is running and fully synchronized with `<network>` chain
- Validator keystores properly imported into `/var/lib/lighthouse/<network>/validators`
- Slashing protection DB exists in `/var/lib/lighthouse/<network>/validators/slashing_protection.sqlite`
- Proper permissions (`validator:validator`) are set for `/var/lib/lighthouse/<network>/validators`

And run the below on the new server:

```console
# enable the validator service at boot
# will also start the service immediately
(new server) # ln -s /etv/sv/validator-<network> /var/service/

# query the service status
(new server) # sv status validator-<network>

# inspect the service log for any erros
(new server) # tail -f /var/log/validator-<network>/current
```

> More information on how to migrate to lighthouse (from other clients) can be found in [Michael Sproul's "Why you should switch to lighthouse"](https://lighthouse.sigmaprime.io/switch-to-lighthouse.html)

## <a id="using-the-lighthouse-wallet">Using the lighthouse wallet</a>

**REMEMBER TO USE AN AIR GAPPED MACHINE EVERY TIME YOU ENTER YOUR MNEMONIC**

### <a id="recovery-using-the-lighthouse-wallet">Recovery using the lighthouse wallet</a>

Lighthouse has a built in wallet/key management functionality, while it's recommended to use the official `deposit-cli` tool to generate your keyset importing the keystores generated by the `deposit-cli` does have one disadvantage of having the validator keystore `keystore-m-[idx]-[timestamp].json` passwords stored in plain text inside the `.../validators/validator_definitions.yml` file.

Using the built-in wallet functionality we can avoid that by safely storing the validator keystore passwords in the `/var/lib/lighthouse/<network>/secrets` folder. This also verifies our mnemonic is correct by comparing the resulting validators pubkeys.

More details can be found in the [lighthouse book key management section](https://lighthouse-book.sigmaprime.io/key-recovery.html).

There are two ways to recover keys using `lighthouse` cli:
- `lighthouse account validator recover`: recover one or more EIP-2335 keystores from a mnemonic. These keys can be used directly in a validator client.
- `lighthouse account wallet recover`: recover an EIP-2386 wallet from a mnemonic.

I recommend using the `validator recover` option which skips wallet generation in `/var/lib/lighthouse/<network>/wallets` which givers potential attackers a larger attack vector.

<details><summary><b>Pyrmont (testnet)</b></summary>

```console
# lighthouse --network pyrmont --datadir /var/lib/lighthouse/pyrmont account validator recover --first-index 0 --count 1
# chown -R validator:validator /var/lib/lighthouse/pyrmont/validators
# chown -R validator:validator /var/lib/lighthouse/pyrmont/secrets
```
</details>
<details><summary><b>Prater (testnet)</b></summary>

```console
# lighthouse --network prater --datadir /var/lib/lighthouse/prater account validator recover --first-index 0 --count 1
# chown -R validator:validator /var/lib/lighthouse/prater/validators
# chown -R validator:validator /var/lib/lighthouse/prater/secrets
```
</details>

<details><summary><b>Beacon (mainnet)</b></summary>

```console
# lighthouse --network mainnet --datadir /var/lib/lighthouse/mainnet account validator recover --first-index 0 --count 1
# chown -R validator:validator /var/lib/lighthouse/mainnet/validators
# chown -R validator:validator /var/lib/lighthouse/mainnet/secrets
```
</details>

> `--first-index 0 --count 1` is equivalent to running the `deposit-cli` with `./deposit existing-mnemonic --validator_start_index 0 --num_validators 1`
>
> The command will output the generated indexes and pubkey addresses of the validators, they should match the addresses of your validators that can be found in the `deposit` cli `json` output files.

### <a id="add-validators-using-the-lighthouse-wallet">Add validators using the lighthouse wallet</a>

Follow the steps in [Recovery using the lighthouse wallet](#recovery-using-the-lighthouse-wallet) above and verify you have two files per validator (the keystore and the secret):
```
❯ tree ~/.lighthouse/<network>
pyrmont
├── secrets
│  └── <pubkey>
└── validators
   └── <pubkey>
      └── voting-keystore.json
```
Now we have 2 options:

#### First option (not recommended)

The first which is to simply copy the files to your lighthouse data directory and change the file ownership:
```console
# sv stop validator-<network>
# cp -R ~/.lighthouse/<network>/* /var/lib/lighthouse/<network>/
# chown -R validator:validator /var/lib/lighthosue/<network>/validators
# chown -R validator:validator /var/lib/lighthosue/<network>/secrets
```

The problem with this approach is that once we do that the service will refuse to start unless we use the `--init-slashing-protection` option, which is extremely dangerous and requires [waiting at least 2 epochs](https://lighthouse-book.sigmaprime.io/slashing-protection.html#troubleshooting) before starting the service again.

The reason for that is that although the new validator will be auto-detected and addded to `<data dir>/validators/validator_definitions.yml` the validator does not exist in the slashing protection database (`<data dir>/validators/slashing_protection.sqlite`) and the service will log the following error:

```console
CRIT Failed to start validator client        reason: One or more validators not found in slashing protection database.
Ensure you haven't misplaced your slashing protection database, or carefully consider running with --init-slashing-protection (see --help). Error: UnregisteredValidator(<pubkey>)
INFO Internal shutdown received              reason: Failed to start validator client
INFO Shutting down..
```

Per [lighthouse book troubleshooting](https://lighthouse-book.sigmaprime.io/slashing-protection.html#troubleshooting) to solve the issue, you should wait at least 1 epochs (~7m) before starting the service with `--init-slashing-protection`.

I recommend waiting at least 2 epochs and verifying at least 2 missed attestations in [beaconcha.in](https://beaconcha.in) before starting the service.

> **Note: It is extremnley important to remove the `--init-slashing-protection` from the service `run` file after starting the service, otherwise each time we start the service we will end up resetting the slashing protection DB**

#### Second option (recommended)

It is easy to see from the above there is lots of room for error in addition to losing the slashing protection DB. However, I was able to find a workaround using the `lighthouse validator import` function.

First we use the lighthouse import function with the generated validator files:
> Press enter at the password prompt, we are using `sercrets` instead
```console
# sv stop validator-<network>
# lighthouse --network <network> account validator import --datadir /var/lib/lighthouse/<network> \
    --directory ~/.lighthouse/<network>/validators
# chown -R validator:validator /var/lib/lighthosue/<network>/validators
```

At this point lighthouse has imported the validator and modified both `slashing_protection.sqlite` and `validator_definitions.yml`. However, if we try to start the validator client we will encounter the following error:

```console
The validator_definitions.yml file does not contain either of the following fields for "/var/lib/lighthouse/<network>/validators/<pubkey>/voting-keystore.json":

  - voting_keystore_password
  - voting_keystore_password_path

 You may exit and update validator_definitions.yml or enter a password. If you choose to enter a password now then this prompt will be raised next time the validator is started.

 Enter password (or press Ctrl+c to exit):
```

Solve this by copying the appropriate `secrets` file, edit the `validator_definitions.yml` and start the validator service:
```console
# cp ~/.lighthouse/<network>/secrets/<pubkey> /var/lib/lighthouse/<network>/secrets
# chown -R validator:validator /var/lib/lighthosue/<network>/secrets
# echo "  voting_keystore_password_path: /var/lib/lighthouse/<network>/secrets/<pubkey>" >> \
    /var/lib/lighthouse/<network>/validators/validator_definitions.yml
# sv start validator-<network>
```


## <a id="how-to-get-goerli-eth">How to get Goerli ETH</a>

If you're planning on running on a testnet (pyrmont / prater) you'll need to acquire testnet eth from the below options:

**(1)** Join [ethstaker discord](https://discord.gg/7z8wzehjrJ), navigate to the `#request-goerli-eth` channel and run `!goerliEth <your-eth1-address>`

**(2)** Join [Prysmatic labs discord](https://discord.gg/KSA7rPr), navigate to the `#request-goerli-eth` channel and run `!send <your-eth1-address>`

**(3)** Use the [Goerli authenticated faucet](https://faucet.goerli.mudit.blog/)

## <a id="enable-ntpd-on-void-linux">Enable `ntpd` on Void Linux</a>

```console
❯ sudo xbps-install -S ntp
❯ sudo ln -s /etv/sv/ntpd /var/service
```

Verify `ntpd` is working and has `leap_none` in the first line output:
```console
❯ ntpq -c rv
associd=0 status=0628 leap_none, sync_ntp, 2 events, no_sys_peer,
version="ntpd 4.2.8p15@1.3728-o Sat Mar  6 00:01:44 UTC 2021 (1)",
processor="x86_64", system="Linux/5.11.17_1", leap=00, stratum=3,
precision=-23, rootdelay=46.636, rootdisp=26.634, refid=74.6.168.73,
reftime=e43adff8.6127f1d8  Mon, May  3 2021 13:47:52.379,
clock=e43ae041.1116451b  Mon, May  3 2021 13:49:05.066, peer=39283, tc=9,
mintc=3, offset=+0.430596, frequency=+35.028, sys_jitter=1.536291,
clk_jitter=4.752, clk_wander=0.741
```
