# Artela-node....
# Minimum Hardware Requirements
  4x CPUs; the faster clock speed the better
  8GB RAM
  100GB of storage (SSD or NVME)

# Recommended Hardware Requirements
  8x CPUs; the faster clock speed the better
  16GB RAM
  1TB of storage (SSD or NVME)

# Dependencies Installation

# Install dependencies for building from source
sudo apt update
sudo apt install -y curl git jq lz4 build-essential

# Install Go
sudo rm -rf /usr/local/go
curl -L https://go.dev/dl/go1.21.6.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> $HOME/.bash_profile
source .bash_profile

# Node Installation

# note : please Put your node name instead of moniker

# Clone project repository
cd && rm -rf artela
git clone https://github.com/artela-network/artela
cd artela
git checkout v0.4.7-rc6

# Build binary
make install

# Set node CLI configuration
artelad config chain-id artela_11822-1
artelad config keyring-backend test
artelad config node tcp://localhost:27857

# Initialize the node
artelad init "moniker" --chain-id artela_11822-1

# Download genesis and addrbook files
curl -L https://snapshots-testnet.nodejumper.io/artela-testnet/genesis.json > $HOME/.artelad/config/genesis.json
curl -L https://snapshots-testnet.nodejumper.io/artela-testnet/addrbook.json > $HOME/.artelad/config/addrbook.json

# Set seeds
sed -i -e 's|^seeds *=.*|seeds = "211536ab1414b5b9a2a759694902ea619b29c8b1@47.251.14.47:26656,d89e10d917f6f7472125aa4c060c05afa78a9d65@47.251.32.165:26656,bec6934fcddbac139bdecce19f81510cb5e02949@47.254.24.106:26656,32d0e4aec8d8a8e33273337e1821f2fe2309539a@47.88.58.36:26656,1bf5b73f1771ea84f9974b9f0015186f1daa4266@47.251.14.47:26656"|' $HOME/.artelad/config/config.toml

# Set minimum gas price
sed -i -e 's|^minimum-gas-prices *=.*|minimum-gas-prices = "20000000000uart"|' $HOME/.artelad/config/app.toml

# Set pruning
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "17"|' \
  $HOME/.artelad/config/app.toml

# Change ports
sed -i -e "s%:1317%:27817%; s%:8080%:27880%; s%:9090%:27890%; s%:9091%:27891%; s%:8545%:27845%; s%:8546%:27846%; s%:6065%:27865%" $HOME/.artelad/config/app.toml
sed -i -e "s%:26658%:27858%; s%:26657%:27857%; s%:6060%:27860%; s%:26656%:27856%; s%:26660%:27861%" $HOME/.artelad/config/config.toml

# Download latest chain data snapshot
curl "https://snapshots-testnet.nodejumper.io/artela-testnet/artela-testnet_latest.tar.lz4" | lz4 -dc - | tar -xf - -C "$HOME/.artelad"

# Create a service
sudo tee /etc/systemd/system/artelad.service > /dev/null << EOF
[Unit]
Description=Artela node service
After=network-online.target
[Service]
User=$USER
ExecStart=$(which artelad) start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable artelad.service

# Start the service and check the logs
sudo systemctl start artelad.service
sudo journalctl -u artelad.service -f --no-hostname -o cat


# check your node block 
artelad status 2>&1 | jq .SyncInfo

# Get Latest Height
artelad status 2>&1 | jq -r '.SyncInfo.latest_block_height // .sync_info.latest_block_height'

# Create a new wallet and write down 24 words and your wallet address
artelad keys add wallet

# now you need faucet 
1. join to the discord  https://discord.com/invite/artela
2. you need Eip 55 address to get faucet
   artelad query bank balances $ARTELA_WALLET_ADDRESS
3.request <YOUR_EVM_ADDRESS>  
4.Voting the private key of Volt to import to Metamask, just type the following command
  artelad keys unsafe-export-eth-key wallet

# Create New Validator
# node : change your moniker and detail 

artelad tx staking create-validator \
--amount=1000000uart \
--pubkey=$(artelad tendermint show-validator) \
--moniker="moniker" \
--identity="" \
--details="I will go ahead " \
--chain-id=artela_11822-1 \
--commission-rate=0.10 \
--commission-max-rate=0.20 \
--commission-max-change-rate=0.01 \
--min-self-delegation=1 \
--from=wallet \
--gas-prices=20000000000uart \
--gas-adjustment=1.5 \
--gas=auto \
-y 

# Edit Existing Validator
artelad tx staking edit-validator \
--moniker="moniker" \
--identity="" \
--details="I will go ahead " \
--chain-id=artela_11822-1 \
--commission-rate=0.1 \
--from=wallet \
--gas-prices=20000000000uart \
--gas-adjustment=1.5 \
--gas=auto \
-y 

# List All Active Validators
artelad q staking validators -oj --limit=3000 | jq '.validators[] | select(.status=="BOND_STATUS_BONDED")' | jq -r '(.tokens|tonumber/pow(10; 6)|floor|tostring) + " \t " + .description.moniker' | sort -gr | nl 

# List All Inactive Validators
artelad q staking validators -oj --limit=3000 | jq '.validators[] | select(.status=="BOND_STATUS_UNBONDED" or .status=="BOND_STATUS_UNBONDING")' | jq -r '(.tokens|tonumber/pow(10; 6)|floor|tostring) + " \t " + .description.moniker' | sort -gr | nl 

# View Validator Details
artelad q staking validator $(artelad keys show wallet --bech val -a)

# Delegate to yourself
artelad tx staking delegate $(artelad keys show wallet --bech val -a) 1000000000000000000uart --from wallet --chain-id artela_11822-1 --gas-adjustment 1.5 --gas auto --gas-prices 0.025uart -y

# List All Proposals
artelad query gov proposals

# node : change proposle id and you vote ( yes - no - NO_WITH_VETO - ABSTAIN )
artelad tx gov vote 1 yes --from wallet --chain-id artela_11822-1 --gas-adjustment 1.5 --gas auto --gas-prices 0.025uart -y

# Be sure to fill out the form below
 https://atkty6pceir.typeform.com/to/o4359Rsd

 #  thank you 12

