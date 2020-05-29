#### ssh-users group ####

```bash
groupadd ssh-users
```

#### non-root login user ###

```bash
useradd -c "user to ssh and sudo" -m -d /home/igghibu -s /bin/bash -G sudo,ssh-users igghibu
```

#### set password and keys ####

**Set a password for <YOUR_SYSTEM_USER>, and enable your public ssh keys in the <YOUR_SYSTEM_USER>'s ~/.ssh/authorized_keys file, or you will lock yourself out.**

#### cardano-node user ####

```bash
useradd -c "user to run cardano node" -m -d /home/cardano-node -s /sbin/nologin cnode
passwd -d cnode
```

#### cardano-node home dir ####

```bash
root@htn-cardano-node:/home/cardano-node# tree /home/cardano-node/
/home/cardano-node/
├── cardano-node.env
├── config
│   ├── ff-config.json
│   ├── ff-genesis.json
│   └── ff-topology.json
├── db
├── logs
└── socket
```

#### install extra packages apt ####

```bash
apt update -y && apt install -y bc cbm ccze chrony curl dateutils fail2ban git git-man htop jq lshw manpages most net-tools ripgrep speedtest-cli sysstat tcptraceroute wget
```


#### cardano-node.env ####

```bash
CONFIG="./config/ff-config.json"
TOPOLOGY="./config/ff-topology.json"
DBPATH="./db/"
SOCKETPATH="./socket/core-node.socket"
HOSTADDR="0.0.0.0"
PORT="3000"
```

#### cardano-node.service ####

```bash
[Unit]
Description=Shelley Pioneer Pool
After=multi-user.target

[Service]
Type=simple
EnvironmentFile=/home/cardano-node/cardano-node.env
ExecStart=/usr/local/bin/cardano-node run --config $CONFIG --topology $TOPOLOGY --database-path $DBPATH --socket-path $SOCKETPATH --host-addr $HOSTADDR --port $PORT
KillSignal = SIGINT
RestartKillSignal = SIGINT
StandardOutput=journal
StandardError=journal
SyslogIdentifier=cardano-node

LimitNOFILE=32768

Restart=on-failure
RestartSec=15s
WorkingDirectory=~
User=cnode
Group=cnode

[Install]
WantedBy=multi-user.target
```
