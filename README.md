# Quicksilver Mainnet joining instructions

**Note: commit hashes and shasums will be added before launch**

For gentx instructions, follow the installation guide and follow the link below.

## Minimum hardware requirements

- 4 cores (max. clock speed possible)
- 16GB RAM
- 500GB of NVMe or SSD disk

## Software requirements

Quicksilver v1.0.0 will be released once a final audit report has been released.

### Install Quicksilver

Requires [Go version v1.19+](https://golang.org/doc/install).

  ```sh
  > git clone https://github.com/ingenuity-build/quicksilver && cd quicksilver
  > git fetch origin --tags
  > git checkout v1.0.0
  > make install
  ```

#### Verify installation

To verify if the installation was successful, execute the following command:

  ```sh
  > quicksilverd version --long
  ```

It will display the version of quicksilverd currently installed:

  ```sh
  name: quicksilverd
  server_name: quicksilverd
  version: 1.0.0
  commit: XXX
  build_tags: netgo,ledger
  go: go version go1.19 linux/amd64
  ```
## Genesis validators

**If the network is already running; continue with the steps below. Otherwise follow the instructions [here](GENTX.md) to create an gentx.**

## Create a validator

1. Init Chain and start your node

   ```sh
   > quicksilverd init <moniker-name> --chain-id=quicksilver-1
   ```

2. Create a local key pair
  **Note: we recommend _only_ using Ledger for mainnet! Key security is important!**

   ```sh
   > ## create a new key:
   > quicksilverd keys add <key-name>
   > ## or use a ledger:
   > quicksilverd key add <key-name> --ledger     
   > ## or import an old key:
   > quicksilverd keys show <key-name> -a
   ```

3. Download genesis
   Fetch `genesis.json` into `quicksilverd`'s `config` directory (default: ~/.quicksilverd)

   ```sh
   > curl -s https://raw.githubusercontent.com/ingenuity-build/mainnet/main/genesis/genesis.tar.gz > genesis.tar.gz
   > tar -C ~/.quicksilverd/config/ -xvf genesis.tar.gz
   ```

   **Genesis sha256**

   ```sh
    shasum -a 256 ~/.quicksilverd/config/genesis.json
    XXX  /home/<user>/.quicksilverd/config/genesis.json
   ```
4. Start your node and sync to the latest block

5. Create validator

   ```sh
   $ quicksilverd tx staking create-validator \
   --amount 50000000uqck \
   --commission-max-change-rate "0.1" \
   --commission-max-rate "0.20" \
   --commission-rate "0.1" \
   --min-self-delegation "1" \
   --details "a short description lives here" \
   --pubkey=$(quicksilverd tendermint show-validator) \
   --moniker <your_moniker> \
   --chain-id quicksilver-1 \
   --from <key-name>
   ```

## Cosmovisor

Optional, but highly recommended for upgrade automation.

Cosmovisor is process manager for Cosmos-SDK application binaries that enables node automation. It monitors the application's governance module for upgrade proposals and allows for automation of application binary downloads and replacement, resulting in near zero-downtime chain upgrades.

### Installation

#### 1. Install cosmovisor

Using go version 1.15 or later:

```
go install github.com/cosmos/cosmos-sdk/cosmovisor/cmd/cosmovisor@latest
```

or, specify the target version:

```
go install github.com/cosmos/cosmos-sdk/cosmovisor/cmd/cosmovisor@v1.0.0
```

Confirm installation with:

```
which cosmovisor
```

The output should be a path to the cosmovisor binary:

```
/home/<user>/go/bin/cosmovisor
```

#### 2. Add environment variables to shell

The following environment variables must be set:

1. `export DAEMON_NAME=quicksilverd`
2. `export DAEMON_HOME=$HOME/.quicksilverd`
3. `export DAEMON_DATA_BACKUP_DIR=$HOME/.quicksilverd/data_backup`

Ensure your environment setup is correctly configured to persist across sessions. Make use of the appropriate system environment configuration files, such as `.profile` to accomplish this.

#### 3. Directory structure

Cosmovisor expects the following directory structure in `$DAEMON_HOME/cosmovisor`:

```
.
├── current -> genesis or upgrades/<name>
├── genesis
│   └── bin
│       └── $DAEMON_NAME
└── upgrades
    └── <name>
        └── bin
            └── $DAEMON_NAME
```

Create the target directory structure with the following:

```
mkdir -p $DAEMON_HOME/cosmovisor/genesis/bin
mkdir -p $DAEMON_HOME/cosmovisor/upgrades
```

`current` is a symlink that will be created by `cosmovisor`.

### Set the genesis binary

Cosmovisor requires the genesis binary to be set. Do this by copying the quicksilverd binary to `$DAEMON_HOME/cosmovisor/genesis/bin/$DAEMON_NAME`.

```
# find quicksilverd binary
which quicksilverd

# copy binary to cosmovisor genesis using output from above command, e.g.
cp build/quicksilverd $DAEMON_HOME/cosmovisor/genesis/bin/$DAEMON_NAME
```

### Configure cosmovisor as a system service

Create the system service file:

```
sudo touch /etc/systemd/system/cosmovisor.service
```

Use an editor like `vim`, `micro` or `nano` and set the contents of the file according to your system configuration, for example:

```
[Unit]
Description=cosmovisor
After=network-online.target

[Service]
User=<your-user>
ExecStart=/home/<your-user>/go/bin/cosmovisor start
Restart=always
RestartSec=3
LimitNOFILE=4096
Environment="DAEMON_NAME=quicksilverd"
Environment="DAEMON_HOME=/home/<your-user>/.quicksilverd"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=false"
# Set buffer size to handle:
# https://github.com/cosmos/cosmos-sdk/pull/8590
Environment="DAEMON_LOG_BUFFER_SIZE=512"
Environment="DAEMON_RESTART_AFTER_UPGRADE=true"
Environment="DAEMON_POLL_INTERVAL=300ms"
Environment="DAEMON_DATA_BACKUP_DIR=${HOME}/.quicksilverd"
# Set to true if disk space is limited:
Environment="UNSAFE_SKIP_BACKUP=false"
Environment="DAEMON_PREUPGRADE_MAX_RETRIES=0"

[Install]
WantedBy=multi-user.target
```

**IMPORTANT**: If you have limited disk space please set `UNSAFE_SKIP_BACKUP=true`. This will avoid an upgrade failure due to insufficient disk space when the backup is created.

Enable and start the cosmovisor service:

```
sudo systemctl daemon-reload
sudo systemctl enable cosmovisor
sudo systemctl restart cosmovisor
```

Check that the service is running:

```
sudo systemctl status cosmovisor
```
