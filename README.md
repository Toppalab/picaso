Dependencies Installation

**Install dependencies for building from source**
```
sudo apt update
sudo apt install -y curl git jq lz4 build-essential
```

**Install Go**
```
sudo rm -rf /usr/local/go
curl -L https://go.dev/dl/go1.22.7.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> $HOME/.profile
source .profile
```

**Clone project repository**
```
cd && rm -rf composable-cosmos
git clone https://github.com/ComposableFi/composable-cosmos
cd composable-cosmos
git checkout v6.6.4
```

**Build binary**
```
make install
```

**Prepare cosmovisor directories**
```
mkdir -p $HOME/.banksy/cosmovisor/genesis/bin
ln -s $HOME/.banksy/cosmovisor/genesis $HOME/.banksy/cosmovisor/current -f
```

**Copy binary to cosmovisor directory**
```
cp $(which picad) $HOME/.banksy/cosmovisor/genesis/bin
```

**Set node CLI configuration**
```
picad config chain-id centauri-1
picad config keyring-backend file
picad config node tcp://localhost:22257
```

**Initialize the node**
```
picad init "Your Node Name" --chain-id centauri-1
```

**Download genesis and addrbook files**
```
curl -L https://snapshots.nodejumper.io/picasso/genesis.json > $HOME/.banksy/config/genesis.json
curl -L https://snapshots.nodejumper.io/picasso/addrbook.json > $HOME/.banksy/config/addrbook.json
```

**Set seeds**
```
sed -i -e 's|^seeds *=.*|seeds = "ebc272824924ea1a27ea3183dd0b9ba713494f83@composable-mainnet-seed.autostake.com:26976,20e1000e88125698264454a884812746c2eb4807@seeds.lavenderfive.com:22256,a3910d1bf22b4dacf66979d6ea75fd134aee00db@seed.composable.validatus.com:2000,8542cd7e6bf9d260fef543bc49e59be5a3fa9074@seed.publicnode.com:26656,aa6398f9644e98fa3d04f7dbdd7740c995eb0530@composable.seed.stavr.tech:20306,ebc272824924ea1a27ea3183dd0b9ba713494f83@composable-mainnet-peer.autostake.com:26976,63559b939442512ed82d2ded46d02ab1021ea29a@95.214.55.138:53656,7082a715395427a519e611ed1454b0965fd95ef5@138.201.21.197:37656,715af1847e1c785510d4cb94ac29f2bd7d0ddf91@65.108.206.74:36656,c6eefdcc5cbe41dd457183c7c3bd7311ddf97638@composable.peer.stakevillage.net:16156"|' $HOME/.banksy/config/config.toml
```

**Set minimum gas price**
```
sed -i -e 's|^minimum-gas-prices *=.*|minimum-gas-prices = "0ppica"|' $HOME/.banksy/config/app.toml
```

**Set pruning**
```
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "17"|' \
  $HOME/.banksy/config/app.toml
```

**Enable prometheus**
```
sed -i -e 's|^prometheus *=.*|prometheus = true|' $HOME/.banksy/config/config.toml
```

**Change ports**
```
sed -i -e "s%:1317%:22217%; s%:8080%:22280%; s%:9090%:22290%; s%:9091%:22291%; s%:8545%:22245%; s%:8546%:22246%; s%:6065%:22265%" $HOME/.banksy/config/app.toml
sed -i -e "s%:26658%:22258%; s%:26657%:22257%; s%:6060%:22260%; s%:26656%:22256%; s%:26660%:22261%" $HOME/.banksy/config/config.toml
```

**Download latest chain data snapshot**
```
curl "https://snapshots.nodejumper.io/picasso/picasso_latest.tar.lz4" | lz4 -dc - | tar -xf - -C "$HOME/.banksy"
```

**Install Cosmovisor**
```
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.6.0
```

**Create a service**
```
sudo tee /etc/systemd/system/picasso.service > /dev/null << EOF
[Unit]
Description=Picasso node service
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.banksy
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.banksy"
Environment="DAEMON_NAME=picad"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=true"
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable picasso.service
```

**Start the service and check the logs**
```
sudo systemctl start picasso.service
sudo journalctl -u picasso.service -f --no-hostname -o cat
Secure Server Setup (Optional)
```

**generate ssh keys, if you don't have them already, DO IT ON YOUR LOCAL MACHINE**
```
ssh-keygen -t rsa
```

**save the output, we'll use it later on instead of YOUR_PUBLIC_SSH_KEY**
```
cat ~/.ssh/id_rsa.pub
```

**upgrade system packages**
```
sudo apt update
sudo apt upgrade -y
```

**add new admin user**
```
sudo adduser admin --disabled-password -q
```

**upload public ssh key, replace YOUR_PUBLIC_SSH_KEY with the key above**
```
mkdir /home/admin/.ssh
echo "YOUR_PUBLIC_SSH_KEY" >> /home/admin/.ssh/authorized_keys
sudo chown admin: /home/admin/.ssh
sudo chown admin: /home/admin/.ssh/authorized_keys

echo "admin ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers
```

**disable root login, disable password authentication, use ssh keys only**
```
sudo sed -i 's|^PermitRootLogin .*|PermitRootLogin no|' /etc/ssh/sshd_config
sudo sed -i 's|^ChallengeResponseAuthentication .*|ChallengeResponseAuthentication no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PasswordAuthentication .*|PasswordAuthentication no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PermitEmptyPasswords .*|PermitEmptyPasswords no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PubkeyAuthentication .*|PubkeyAuthentication yes|' /etc/ssh/sshd_config

sudo systemctl restart sshd
```

**install fail2ban**
```
sudo apt install -y fail2ban
```

**install and configure firewall**
```
sudo apt install -y ufw
sudo ufw default allow outgoing
sudo ufw default deny incoming
sudo ufw allow ssh
sudo ufw allow 9100
sudo ufw allow 26656
```

# make sure you expose ALL necessary ports, only after that enable firewall
sudo ufw enable

# make terminal colorful
sudo su - admin
source <(curl -s https://raw.githubusercontent.com/nodejumper-org/cosmos-scripts/master/utils/enable_colorful_bash.sh)

# update servername, if needed, replace YOUR_SERVERNAME with wanted server name
sudo hostnamectl set-hostname YOUR_SERVERNAME

# now you can logout (exit) and login again using ssh admin@YOUR_SERVER_IP
