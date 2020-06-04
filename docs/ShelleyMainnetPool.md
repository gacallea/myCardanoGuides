# Shelley Main-net Pool #

## Prepare Your System ##

The guides assumes that the system will be managed with ```root```. Don't worry, to ```ssh``` and ```sudo```, there will be a dedicated **non-root user**. To run the pool, yet another *service user*, with neither a shell nor privileges. So, if you are wondering if the pool will run as ```root```, the answer is **no way.** Systemd will take care of running the pool as the *service user*. A *service user* without a shell or a password, means less surface attack for an hacker trying to exploit *testing quality* software.

### ssh-users group ###

Firstly, you need to create a group for a finer ssh login control and for an added layer of security. If you want to understand how this works and improves security, it is simple: it adds a restriction that will allow ```ssh``` connections from users who are in the ```ssh-users``` group **only**. Later in the guide, you'll also find a ```sshd_config``` file with more enhancements and restrictions.

```bash
groupadd ssh-users
```

Make sure that the ```ssh-users``` group was successfully created:

```bash
grep "ssh-users" /etc/group
```

It should return something like the following (the ```gid``` will likely be different for your system):

```bash
ssh-users:x:998
```

### non-root login user ###

This is **your main non-root user** that you will be using to ```ssh``` into the server and to ```sudo``` to ```root```, and to manage your system. Make sure to replace ```<YOUR_SYSTEM_USER>``` with your user name of choice. **If you already have such user** that you actively ```ssh``` and ```sudo``` with, you can skip creating one, but make sure you add ```<YOUR_EXISTING_USER>``` to ```sudo``` and ```ssh-users``` groups.

For a new user run:

```bash
useradd -c "user to ssh and sudo" -m -d /home/<YOUR_SYSTEM_USER> -s /bin/bash -G sudo,ssh-users <YOUR_SYSTEM_USER>
```

For an existing user run:

```bash
usermod -aG sudo,ssh-users <YOUR_EXISTING_USER>
```

Double-check that your user (either ```<YOUR_SYSTEM_USER>``` or ```<YOUR_EXISTING_USER>```) is in both the ```sudo``` and ```ssh-users``` groups. **This step is important, don't skip it**. Later will be setting up ```sshd``` to only allow ```ssh``` from this group only. **You risk of locking yourself out**.

```bash
groups <YOUR_SYSTEM_USER>
```

It should show the following:

```bash
<YOUR_SYSTEM_USER> : <YOUR_SYSTEM_USER> sudo ssh-users
```

#### set password and keys ####

**Set a password for ```<YOUR_SYSTEM_USER>```, and enable your public ssh keys in the ```<YOUR_SYSTEM_USER>```'s ```~/.ssh/authorized_keys``` file, or you will lock yourself out.**

### non-root service user ###

Running a service exposed to the Internet, with a user who has a shell it is not a wise choice, to use an euphemism. This is why you are creating a dedicated user to run the service. This is also standard practice for services in Linux. Think of ```nginx```, for example. It has both a user and a group, directories, configurations, and some permissions; but it doesn't need neither a shell nor password. Because exposing a shell to the outside world is a security risk. This reduces the attack surface on the server.

```bash
useradd -c "user to run cardano node" -m -d /home/cardano-node -s /sbin/nologin cnode
passwd -d cnode
```

## Install Cardano Node Software ##

### compile and install cardano-node ###

At the minute, the only way to install the ```cardano-node``` and the ```cardano-cli``` software components necessary to run a stake pool on Shelley is to compile them. Compilation can be troublesome and lead to issues. To avoid any complications and always get solid results and the right binaries, I provide a Docker build environment to build against the latest version/tag as in [the official documentation to build a node](https://github.com/input-output-hk/cardano-tutorials/blob/master/node-setup/build.md).

**IMPORTANT: you must run these steps to compile ```cardano-node``` and ```cardano-cli``` on your local computer, and copy them over to your servers. This is to avoid useless load and installing compilation software on your servers.**

#### install docker ####

Install and run Docker Desktop for your OS from [the official site](https://www.docker.com/products/docker-desktop).

#### clone the build repo ####

```bash
git clone https://github.com/gacallea/cardanoNodeBuilder.git
```

#### compile the software ####

```bash
cd cardanoNodeBuilder
docker-compose up --build -d
```

#### copy the binaries to your host ####

```bash
docker container cp build_builder_1:/usr/local/bin cardano-bins/
```

#### scp the binaries to your nodes ####

**NOTE: Make sure you point to your actual servers.**

```bash
scp cardano-bins/bin/cardano-node node-server:/usr/local/bin/
scp cardano-bins/bin/cardano-cli node-server:/usr/local/bin/
```

```bash
scp cardano-bins/bin/cardano-node relay-server:/usr/local/bin/
scp cardano-bins/bin/cardano-cli relay-server:/usr/local/bin/
```

#### cardano-relay home dir ####

```bash
root@htn-cardano-relay:~# tree /home/cardano-node/
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

#### cardano-relay environment ####

```bash
CONFIG="./config/ff-config.json"
TOPOLOGY="./config/ff-topology.json"
DBPATH="./db/"
SOCKETPATH="./socket/core-node.socket"
HOSTADDR="0.0.0.0"
PORT="3000"
```

#### cardano-relay.service ####

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

#### cardano-node home dir ####

```bash
root@htn-cardano-node:~# tree /home/cardano-node/
/home/cardano-node/
├── cardano-node.env
├── config
│   ├── ff-config.json
│   ├── ff-genesis.json
│   └── ff-topology.json
├── db
├── keys
│   ├── kes.skey
│   ├── opcert
│   └── vrf.skey
├── logs
└── socket
```

#### cardano-node environment ####

```bash
CONFIG="./config/ff-config.json"
TOPOLOGY="./config/ff-topology.json"
DBPATH="./db/"
SOCKETPATH="./socket/core-node.socket"
HOSTADDR="0.0.0.0"
PORT="3000"
KES_SK="./keys/kes.skey"
VRF_SK="./keys/vrf.skey"
OPCERT="./keys/opcert"
```

#### cardano-node.service ####

```bash
[Unit]
Description=Shelley Pioneer Pool
After=multi-user.target

[Service]
Type=simple
EnvironmentFile=/home/cardano-node/cardano-node.env
ExecStart=/usr/local/bin/cardano-node run --config $CONFIG --topology $TOPOLOGY --database-path $DBPATH --socket-path $SOCKETPATH --host-addr $HOSTADDR --port $PORT  --shelley-kes-key $KES_SK --shelley-vrf-key $VRF_SK --shelley-operational-certificate $OPCERT
KillSignal = SIGINT
RestartKillSignal = SIGINT
StandardOutput=journal
StandardError=journal
SyslogIdentifier=cardano-node

LimitNOFILE=65536

Restart=on-failure
RestartSec=15s
WorkingDirectory=~
User=cnode
Group=cnode

[Install]
WantedBy=multi-user.target
```

### configure logging ###

Now ```cardano-node``` is a system managed service, it's time to configure system level logging with ```rsyslog``` and ```logrotate```.

Place the following in ```/etc/rsyslog.d/90-cardano-node.conf```:

```bash
if $programname == 'cardano-node' then /var/log/cardano-node.log
& stop
```

Place the following in ```/etc/logrotate.d/cardano-node```:

```bash
/var/log/cardano-node.log {
    daily
    rotate 30
    copytruncate
    compress
    delaycompress
    notifempty
    missingok
}
```

Restart the logging services:

```bash
systemctl restart rsyslog.service
systemctl restart logrotate.service
```

Now you can check your logs as for any other service with:

```bash
journalctl -f -u cardano-node.service
```
