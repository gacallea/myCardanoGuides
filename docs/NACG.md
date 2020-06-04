# Not Another Cardano Guide #

## Register Your Pool ##

Before you can proceed with the installation and configuration, it is necessary to register your pool ticker with the [Cardano Foundation Registry](https://github.com/cardano-foundation/incentivized-testnet-stakepool-registry). Follow [IOHK](https://github.com/input-output-hk)'s [**guide**](https://github.com/input-output-hk/shelley-testnet/blob/master/docs/stake_pool_operator_how_to.md) and use their [QA Team](https://github.com/input-output-hk/jormungandr-qa) [**scripts**](https://github.com/input-output-hk/jormungandr-qa/tree/master/scripts), to do that. The resulting ```node-secret.yaml``` and a registered pool ticker are requirements to proceed with the installation of the cluster. **Make sure to backup the generated files and keys when the IOHK guide instructs you to do so. They are vital to run a pool and to get your rewards, in due time**.

Mind that **you need to have enough tADA** - meaning ADA that were **in your wallet before the November 2019 snapshot** - otherwise you won't be able to proceed with the pool registration, and with this guide.

Come back after you have successfully completed **all** the necessary steps, and have registered your pool with the [Cardano Foundation Registry](https://github.com/cardano-foundation/incentivized-testnet-stakepool-registry). Once your pool is available on Daedalus and Yoroi (**testnet versions**), you can discard the IOHK installation to proceed with this guide and set up your cluster.

Should you need help at any stage of your pool operator journey, join the '[Cardano Shelley Testnet & StakePool Best Practice Workgroup](https://t.me/CardanoStakePoolWorkgroup)' group on Telegram; it is packed with knowledge, and great and helpful people.

**IMPORTANT: if you have already successfully created and registered your pool, you don't need to do it all over again. Just have the ```node-secret.yaml``` file handy for later.**

Should you have a ```node-secret.json```, convert it to ```node-secret.yaml``` with this tool: [https://www.json2yaml.com/](https://www.json2yaml.com/)

## Prepare Your System ##

The guides assumes that the system will be managed with ```root```. Don't worry, to ```ssh``` and ```sudo```, there will be a dedicated **non-root user**. To run the pool, yet another *service user*, with neither a shell nor privileges. So, if you are wondering if the pool will run as ```root```, the answer is **no way.** Systemd will take care of running the pool as the *service user*. A *service user* without a shell or a password, means less surface attack for an hacker trying to exploit *testing quality* software. **Let's get started.**

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

This is **your main user** that you will be using to ```ssh``` into the server and to ```sudo``` to ```root```, to manage your system. Make sure to replace ```<YOUR_SYSTEM_USER>``` with your user name of choice.

**If you already have such user** that you actively ```ssh``` and ```sudo``` with, you can skip creating one, but make sure you add it to ```sudo``` and ```ssh-users``` groups.

For a **new** user run:

```bash
useradd -c "user to ssh and sudo" -m -d /home/<YOUR_SYSTEM_USER> -s /bin/bash -G sudo,ssh-users <YOUR_SYSTEM_USER>
```

For an **existing** user run:

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

**Set a password for your login user, and enable your public ssh keys in the users' "```~/.ssh/authorized_keys```" file, or you will lock yourself out.**

### non-root service user ###

Running a service exposed to the Internet, with a user who has a shell it is not a wise choice, to use an euphemism. This is why you are creating a dedicated user to run the service. This is also standard practice for services in Linux. Think of ```nginx```, for example. It has both a user and a group, directories, configurations, and some permissions; but it doesn't need neither a shell nor password. Because exposing a shell to the outside world is a security risk. This reduces the attack surface on the server.

If you want to customize this user, Make sure to replace ```poolrun``` with your user name of choice, and take a note of it. You will be needing this username later in the guide when you will configure ```systemd``` and the scripts. Note: ```poolrun``` is a safe default, should you want to use it as is.

```bash
useradd -c "user to run the pool" -m -d /home/poolrun -s /sbin/nologin poolrun
```

```bash
passwd -d poolrun
```

### install extra packages ###

This guide assumes that you are familiar with compilation, and that you know why and when compilation is necessary or useful, and that you are capable of compiling. Therefore, during this guide you **won't** be compiling ```jormungandr``` or ```jcli```. If you reckon that compiling will give you more, knock yourself out. If you compile, is advisable to do it on a dedicated environment, or cross-compile, and transfer the binaries to the pool server.

#### install from apt ####

Some of the installed tools are used in my scripts, some others serve system administration purposes:

- ```bc``` is used for calculations in my scripts
- ```cbm``` is a nice real-time bandwidth monitor for the terminal
- ```chrony``` is used for better time sync
- ```ccze``` is for coloring commands output
- ```dateutils``` is used for date related calculations in my scripts
- ```fail2ban``` to keep script kiddies at bay
- ```htop``` is a must have ```top``` on steroids
- ```jq``` if you want to send your stats to [PoolTool.io](https://pooltool.io/health)
- ```ripgrep``` is used in my scripts
- ```speedtest-cli``` in case you need a good speed test for your server
- ```musl``` is a [C library](https://wiki.musl-libc.org/functional-differences-from-glibc.html), in case you want to run the [musl](https://musl.libc.org/) version of ```jormungandr```

```bash
apt-get update && apt-get install -y bc cbm ccze chrony curl dateutils fail2ban htop jq lshw manpages most net-tools ripgrep speedtest-cli sysstat tcptraceroute vnstat wget musl
```

#### install jormungandr and jcli ####

You should stick [to the latest stable release](https://github.com/input-output-hk/jormungandr/releases), unless it introduces regressions. The following works for the current release for a ```x86_64``` architecture (PC/Mac - Intel/AMD Server) and [GNU](https://www.gnu.org/) ```glibc```.

```bash
curl -sLOJ https://github.com/input-output-hk/jormungandr/releases/download/v0.8.19/jormungandr-v0.8.19-x86_64-unknown-linux-gnu-generic.tar.gz
```

```bash
tar xzvf jormungandr-v0.8.19-x86_64-unknown-linux-gnu-generic.tar.gz
```

```bash
mv jormungandr jcli /usr/local/bin/
```

```bash
chmod +x /usr/local/bin/jormungandr /usr/local/bin/jcli
```

```bash
chown -R root\: /usr/local/bin/
```

#### install tcpping ####

This is going to be the only alien piece of software, besides pool software, that you will be installing from a source that is not from official Ubuntu repositories. It is used in my scripts.

```bash
curl -s http://www.vdberg.org/~richard/tcpping -o /usr/local/bin/tcpping
```

```bash
chmod +x /usr/local/bin/tcpping
```

```bash
chown -R root\: /usr/local/bin/
```

## Configure Your System ##

To configure your system, you'll be using configuration files and scripts that are either provided by me or linked to other great guides. Always remember to **adapt them to your system**,  where it's needed.

### configure the firewall ###

A controversial note, first: believe it or not, if your server is only running ```sshd``` and ```jormungandr``` a firewall is not really necessary. Both services need an open port, the ```jcli``` REST API runs locally, and there's not other running service to externally attack. You could skip the firewall configuration, change the ```sshd``` port and install ```fail2ban```; that would be good enough. However, setting up a firewall is something this guide will help you do. This guide will help you configure a firewall with ```ufw```.

To be on the safe side, let's disable the ```ufw``` service first:

```bash
ufw disable
```

This should not be necessary, as it is usually the default settings. To be on the safe side, let's set defaults for incoming/outgoing ports:

```bash
ufw default deny incoming
ufw default allow outgoing
```

Next, you should decide which ports to use for ```jormungandr``` and ```sshd```. This guide will use ```3001``` and ```5269```, respectively, as an example, and to provide a default settings that you may want to keep. If unsure, stick with the guide's defaults. Let's limit the ```sshd``` ports, both the default ```22``` and the guide's ```5269```. You'll remove the default port once you configure ```sshd_config```:

```bash
ufw limit 22
ufw limit 5269
```

Let's open the ```jormungandr``` port to which the pool will be connected to the ITN network:

```bash
ufw allow 3001
```

Now that all the necessary rules have been set, let's enable the firewall and double check the rules:

```bash
ufw enable
ufw status verbose
```

### configure sshd ###

You'll be enabling some additional restrictions, and disabling some features that are enabled by default. Like tunneling and forwarding. [Read why](https://www.ssh.com/ssh/tunneling#ssh-tunneling-in-the-corporate-risk-portfolio) it is bad to leave SSH tunneling on. Some guides suggest to tunnel into your remote server for monitoring purposes. This is bad practice, and a security risk. Make sure you have the following configured in ```/etc/ssh/sshd_config```; everything else can be commented out.

```bash
Port 5269
Protocol 2

LoginGraceTime 2m
PermitRootLogin no
StrictModes yes
MaxAuthTries 6
MaxSessions 10

PubkeyAuthentication yes
IgnoreRhosts yes
PasswordAuthentication no
PermitEmptyPasswords no
ChallengeResponseAuthentication no
UsePAM yes

X11Forwarding no
PrintMotd no

ClientAliveInterval 300
ClientAliveCountMax 2

AllowTcpForwarding no
AllowStreamLocalForwarding no
GatewayPorts no
PermitTunnel no

AllowGroups ssh-users

AcceptEnv LANG LC_*

Subsystem       sftp    /usr/lib/openssh/sftp-server
```

Restart the ```sshd``` server, and ```ssh``` into the server from another terminal to test the new configuration..

```bash
systemctl restart sshd.service
```

Make **absolutely sure** you can ```ssh``` into your server with the newly configured port ```5269```, delete the ```ssh``` (**port ```22```**) rule, and reload the ```ufw``` service.

To delete a rule from ```ufw```, first list the rules numbered:

```bash
ufw status numbered
```

Then proceed to delete the corresponding rule (**replace 1 with the actual number you get from the above numbered list!!!**):

```bash
ufw delete 1
```

```bash
ufw reload
```

### check fail2ban ###

While ```fail2ban``` doesn't offer perfect security - [*security is a process, not a product*](https://www.schneier.com/essays/archives/2000/04/the_process_of_secur.html) - it serves its purpose. The default ```fail2ban``` configuration is generally good enough, to check that the service is active and a ssh jail configured, run:

```bash
fail2ban-client status
```

It should return:

```text
Status
|- Number of jail:      1
`- Jail list:   sshd
```

### configure chrony ###

This is the first of three files configurations that are borrowed from other great guides. There's no need to reinvent the wheel here, so I'm pointing you to [LovelyPool](https://github.com/lovelypool/)'s [chronysettings](https://github.com/lovelypool/cardano_stuff/blob/master/chronysettings.md) guide instead, but still provide the configuration for your convenience. Make sure to **read** Lovelypool's **Chrony Settings** guide, to understand it fully, and to know why to use ```chrony```.

Place this in ```/etc/chrony/chrony.conf```:

```bash
pool time.google.com       iburst minpoll 1 maxpoll 2 maxsources 3

# This directive specify the location of the file containing ID/key pairs for
# NTP authentication.
keyfile /etc/chrony/chrony.keys

# This directive specify the file into which chronyd will store the rate
# information.
driftfile /var/lib/chrony/chrony.drift

# Uncomment the following line to turn logging on.
#log tracking measurements statistics

# Log files location.
logdir /var/log/chrony

# Stop bad estimates upsetting machine clock.
maxupdateskew 5.0

# This directive enables kernel synchronisation (every 11 minutes) of the
# real-time clock. Note that it can’t be used along with the 'rtcfile' directive.
rtcsync

# Step the system clock instead of slewing it if the adjustment is larger than
# one second, but only in the first three clock updates.
makestep 0.1 -1

# Get TAI-UTC offset and leap seconds from the system tz database.
leapsectz right/UTC

# Serve time even if not synchronized to a time source.
local stratum 10
```

Restart Chrony:

```tet
systemctl restart chronyd.service
```

### configure limits ###

These are the other two, and last, files that I borrowed from other great guides. This time I borrowed from [Ilap](https://github.com/ilap/)'s [guide](https://gist.github.com/ilap/54027fe9af0513c2701dc556221198b2). For convenience, I do provide the configuration for these too. Again, **read** his reasoning [here](https://gist.github.com/ilap/54027fe9af0513c2701dc556221198b2), and check often for his updates.

Place these at the bottom of your ```/etc/security/limits.conf```:

```bash
root soft nofile 32768
poolrun soft nofile 32768
poolrun hard nofile 1048577
```

Place these at the bottom of your ```/etc/sysctl.conf```:

```bash
fs.file-max = 10000000
fs.nr_open = 10000000

net.core.netdev_max_backlog = 100000
net.core.somaxconn = 100000
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.ip_local_port_range = 1024 65535
net.ipv4.ip_nonlocal_bind = 1
net.ipv4.tcp_fin_timeout = 10
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_max_orphans = 262144
net.ipv4.tcp_max_syn_backlog = 100000
net.ipv4.tcp_max_tw_buckets = 262144
net.ipv4.tcp_mem = 786432 1697152 1945728
net.ipv4.tcp_reordering = 3
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_sack = 0
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_syn_retries = 5
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_wmem = 4096 16384 16777216

net.netfilter.nf_conntrack_max = 10485760
net.netfilter.nf_conntrack_tcp_timeout_fin_wait = 30
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 15

vm.swappiness = 10
```

Load your newly configured variables:

```bash
sysctl -p /etc/sysctl.conf
```

### configure systemd ###

It is time to manage ```jormungandr``` as you would manage any other service on your server: with ```root``` and ```systemd```. Place the following in ```/etc/systemd/system/jormungandr.service```, and **make sure to change** ```poolrun``` and ```3101``` to match your system:

```bash
[Unit]
Description=Shelley Staking Pool
After=multi-user.target

[Service]
Type=simple
ExecStart=/usr/local/bin/jormungandr --config node-config.yaml --secret node-secret.yaml --genesis-block-hash 8e4d2a343f3dcf9330ad9035b3e8d168e6728904262f2c434a4f8f934ec7b676
ExecStop=/usr/local/bin/jcli rest v0 shutdown get -h http://127.0.0.1:3101/api
StandardOutput=journal
StandardError=journal
SyslogIdentifier=jormungandr

LimitNOFILE=32768

Restart=on-failure
RestartSec=5s
WorkingDirectory=~
User=poolrun
Group=poolrun

[Install]
WantedBy=multi-user.target
```

Let's unpack the ```unit``` file:

1. it runs ```jormungandr``` as ```poolrun```.
2. it looks ```node-*.yaml``` in  ```poolrun``` home directory
3. it provides for  ```systemctl``` start, stop and restart.
4. it restarts ```jormungandr``` on failures
5. it logs to ```journal```.
6. it sets the limits accordingly.

Reload ```systemd``` to read the new ```unit``` file, and enable it.

```bash
systemctl daemon-reload
systemctl enable jormungandr.service
```

Whenever you need to ```start```, ```stop```, and ```restart``` your node, do it with:

```bash
systemctl start jormungandr.service
```

```bash
systemctl stop jormungandr.service
```

```bash
systemctl restart jormungandr.service
```

### configure logging ###

Now ```jormungandr``` is a system managed service, it's time to configure system level logging with ```rsyslog``` and ```logrotate```.

Place the following in ```/etc/rsyslog.d/90-jormungandr.conf```:

```bash
if $programname == 'jormungandr' then /var/log/jormungandr.log
& stop
```

Place the following in ```/etc/logrotate.d/jormungandr```:

```bash
/var/log/jormungandr.log {
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
journalctl -f -u jormungandr.service
```

### configure node ###

Now that you have configured your server, hosting your pool, you may consider using my ```node-config.yaml```. It was refined every single day until my node run smoothly (for a testing stage software like ```jormungandr``` is as of this writing). This step is **completely optional**, feel free to skip it and trust your own experience and configuration.

By "*running smoothly*", I mean that the node bootstraps relatively quickly; that the "peerAvailableCnt:peerQuarantinedCnt" peers count ratio is reasonable; that the node has a decent amount of established connections, that the node has a great uptime, and a good sync to the network. It is not final by any means, and performance **it varies from node to node**.

For reference only, my node has the following specs:

| Resource | Specs                               |
| -------- | ----------------------------------- |
| CPU      | Intel  Xeon W3520 (4c/8t - 2,66GHz) |
| RAM      | 16GB DDR3 ECC 1333 MHz              |
| SSD      | SoftRaid 2x2TB                      |
| Network  | 100Mpbs                             |
| Traffic  | Unlimited                           |

**IMPORTANT: place  the ```node-config.yaml``` and ```node-secret.yaml``` in your ```/home/poolrun/``` directory:**

```bash
/home/poolrun/node-config.yaml
/home/poolrun/node-secret.yaml
```

Should you decide to use it, place the following in ```/home/poolrun/node-config.yaml```. The only adjustment you should take care of, **besides changing the variables to match your system**, is to change the ```trusted_peers``` order to place the nearest to you at the top of the list.

```bash
---
log:
  - output: stderr
    format: "plain"
    level: "info"
p2p:
  listen_address: "/ip4/0.0.0.0/tcp/3001"
  public_address: "/ip4/<YOUR_NODE_PUBLIC_IP>/tcp/3001"
  topics_of_interest:
    blocks: high
    messages: high
  max_connections: 256
  max_inbound_connections: 128
  max_unreachable_nodes_to_connect_per_event: 32
  max_bootstrap_attempts: 3
  gossip_interval: 4s
  policy:
    quarantine_duration: 15m
  trusted_peers:
    - address: "/ip4/13.56.0.226/tcp/3000"
      id: 7ddf203c86a012e8863ef19d96aabba23d2445c492d86267
    - address: "/ip4/52.28.91.178/tcp/3000"
      id: 23b3ca09c644fe8098f64c24d75d9f79c8e058642e63a28c
    - address: "/ip4/3.125.75.156/tcp/3000"
      id: 22fb117f9f72f38b21bca5c0f069766c0d4327925d967791
    - address: "/ip4/13.112.181.42/tcp/3000"
      id: 52762c49a84699d43c96fdfe6de18079fb2512077d6aa5bc
    - address: "/ip4/13.114.196.228/tcp/3000"
      id: 7e1020c2e2107a849a8353876d047085f475c9bc646e42e9
    - address: "/ip4/52.8.15.52/tcp/3000"
      id: 18bf81a75e5b15a49b843a66f61602e14d4261fb5595b5f5
    - address: "/ip4/52.9.132.248/tcp/3000"
      id: 671a9e7a5c739532668511bea823f0f5c5557c99b813456c
    - address: "/ip4/3.125.183.71/tcp/3000"
      id: 9d15a9e2f1336c7acda8ced34e929f697dc24ea0910c3e67
    - address: "/ip4/3.125.31.84/tcp/3000"
      id: 8f9ff09765684199b351d520defac463b1282a63d3cc99ca
    - address: "/ip4/18.184.35.137/tcp/3000"
      id: 06aa98b0ab6589f464d08911717115ef354161f0dc727858
    - address: "/ip4/18.182.115.51/tcp/3000"
      id: 8529e334a39a5b6033b698be2040b1089d8f67e0102e2575
    - address: "/ip4/3.115.154.161/tcp/3000"
      id: 35bead7d45b3b8bda5e74aa12126d871069e7617b7f4fe62
    - address: "/ip4/18.177.78.96/tcp/3000"
      id: fc89bff08ec4e054b4f03106f5312834abdf2fcb444610e9
    - address: "/ip4/52.9.77.197/tcp/3000"
      id: fcdf302895236d012635052725a0cdfc2e8ee394a1935b63
    - address: "/ip4/54.183.149.167/tcp/3000"
      id: df02383863ae5e14fea5d51a092585da34e689a73f704613
    - address: "/ip4/3.124.116.145/tcp/3000"
      id: 99cb10f53185fbef110472d45a36082905ee12df8a049b74
rest:
  listen: 127.0.0.1:3101
storage: "/home/poolrun/"
mempool:
    pool_max_entries: 10000
    log_max_entries: 100000
leadership:
    logs_capacity: 4096
http_fetch_block0_service:
- "https://github.com/input-output-hk/jormungandr-block0/raw/master/data/"
skip_bootstrap: false
bootstrap_from_trusted_peers: false
```

Restart ```jormungandr``` to use the new configuration:

```bash
systemctl restart jormungandr.service
```

## Monitor Your System ##

Monitoring is indispensable for any given service. Monitoring, in layman's terms, *provides and automates real-time services health history*, and alerts you whenever something goes wrong. So that you don't have to be constantly connected to your server. Mind that monitoring doesn't substitute server administration. It is only a tool that informs the administrator about the health of the server and its services, and that lets one know it's time to connect and act upon alerts. It is still the administrator responsibility to fix any issue, and to keep the server healthy. Moreover, the history aspect that monitoring provides, can help you look into the big picture to understand what went wrong, even when you were not connected or sound asleep.

In this section you will configure and automate Prometheus and Grafana *to be there for you*. In a future revision of this guide, you will configure alerting to send you messages or emails, to let you know about issues when they happen. With alerting, if you don't receive any alerts, "*no news is good news*". If you do receive them, it's time connect and troubleshoot the issues.

**IMPORTANT:** this guide assumes a remote server with a working domain. [DNS](https://en.wikipedia.org/wiki/Domain_Name_System) needs to be configured to access you monitoring, remotely.

### Install Monitoring ###

You need to install [Prometheus](https://prometheus.io/) from the official Ubuntu repository, and [Grafana](https://grafana.com/) from the [repository they provide](https://grafana.com/docs/grafana/latest/installation/Ubuntu/). You'll also install and configure [Nginx](https://www.nginx.com/) and [EFF](https://eff.org/)'s [Certbot](https://certbot.eff.org/) to automate the process of creating and managing [Let's Encrypt](https://letsencrypt.org/) certificates. Nginx is used to [reverse-proxy](https://en.wikipedia.org/wiki/Reverse_proxy) to Grafana, and the [SSL encryption](https://www.ssl.com/faqs/faq-what-is-ssl/) is necessary to securely connect to and peruse your remote monitoring, [while keeping safe from prying eyes](https://www.youtube.com/watch?v=-enHfpHMBo4). To install the needed software, use the following commands:

```bash
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
```

```bash
echo "deb https://packages.grafana.com/oss/deb stable main" > /etc/apt/sources.list.d/grafana.list
```

```bash
apt-get update && apt-get install -y prometheus prometheus-alertmanager prometheus-node-exporter nginx certbot python3-pip python3-certbot-nginx grafana
```

### Prometheus ###

At the fundamental level, Prometheus collects data and generates metrics. The resulting metrics can be fed into a tool, like Grafana, to depict the big picture for you. Learn more about Prometheus [here](https://prometheus.io/docs/introduction/overview/). Prometheus works with [exporters](https://prometheus.io/docs/instrumenting/exporters/) to collect data from several services. You'll be using two: the official [Node Exporter](https://github.com/prometheus/node_exporter) (installed above), and the Jormungandr Exporter.

#### Jormungandr Exporter ####

This Python script, based on [the official IOHK one](https://github.com/input-output-hk/jormungandr-nix/blob/master/nixos/jormungandr-monitor/monitor.py), improves to use the latest ```peerConnectedCnt```, and will collect a number of Jormungandr metrics using ```jcli``` and the REST API. Let's install the necessary Python dependencies:

```bash
pip3 install prometheus_client python-dateutil systemd-python ipython
```

To get the script in place, use the the following ```curl```, and **remember to change** ```<REST_API_PORT>``` to your REST API port in the ```jormungandr-monitor.py``` **or it will not work correctly**.

```bash
curl -sLJ https://raw.githubusercontent.com/gacallea/cardanoRelatedStuff/master/monitoring/jormungandr-monitor.py -o /etc/prometheus/jormungandr-monitor.py
```

. ```jormungandr-monitor.py``` will be managed with ```systemd``` as well. Let's create the necessary unit file called ```/etc/systemd/system/jormungandr-monitor.service```

```bash
[Unit]
Description="Jormungandr Monitoring Script"
After=multi-user.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 /etc/prometheus/jormungandr-monitor.py

StandardOutput=journal
StandardError=journal
SyslogIdentifier=jormungandr-monitor

Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

Reload ```systemd``` to read the new ```unit```:

```bash
systemctl daemon-reload
```

#### Prometheus Configuration ####

Prometheus configuration is pretty straight-forward. Edit ```/etc/prometheus/prometheus.yml``` and use the following settings:

```bash
# NACG Config for Prometheus.

global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.

  # Attach these labels to any time series or alerts when communicating with
  # external systems (federation, remote storage, Alertmanager).
  external_labels:
      monitor: 'Jormungandr Monitor'

# A scrape configuration containing exactly one endpoint to scrape:
scrape_configs:
  - job_name: 'prometheus'
    scrape_interval: 5s
    scrape_timeout: 5s
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']

  - job_name: 'jormungandr'
    static_configs:
      - targets: ['localhost:9101']
```

That's all its needed for Prometheus to be collecting data from the Node and Jormungandr exporters, and to generate metrics that will be later fed to Grafana. Make sure to enable the services and start them as well:

```bash
systemctl enable prometheus.service
```

```bash
systemctl enable prometheus-node-exporter.service
```

```bash
systemctl enable jormungandr-monitor.service
```

```bash
systemctl restart prometheus.service
```

```bash
systemctl restart prometheus-node-exporter.service
```

```bash
systemctl restart jormungandr-monitor.service
```

#### Prometheus Alerting ####

Alerting with Prometheus will be implemented in a future revision of this guide. Follow [insaladaPool](https://twitter.com/insaladaPool) for future updates.

### Grafana ###

Grafana is a graphing software that provides dashboards that give you a complete view of what's going on with your server and with your pool. It can show backward history, as long there is data to graph on. This is you main point of entry to monitoring. Grafana can talk to a number of data sources that *feed data to it*. You are using and have configured Prometheus to be our source of data, now it's time to bring it all together.

#### Grafana Configuration ####

Grafana by default runs on port ```3000```. If ```jormungandr``` is running on port ```3000``` as well, you need to change either one. I suggest you use port ```3000``` for Grafana, and set ```jormungandr``` port to ```3001``` and its ```<REST_API_PORT>``` set to ```3101```. This way it can be easy to mnemonically associate ```jormungandr``` and its instance.

The Grafana configuration has pretty solid default values. You only need to change a handful to successfully configure it. Here, ```grafana.example.com``` is used as an example, you can use whatever suits you to replace the ```grafana.``` bit (for instance ```monitoring.```). However, **the domain has to match your actual domain**. Here's what needs changing (in which blocks):

##### [server] #####

```bash
domain = grafana.example.com
root_url = https://grafana.example.com
```

##### [users] #####

```bash
allow_sign_up = false
```

##### [auth.anonymous] #####

```bash
enabled = false
```

Restart Grafana to reload the new configuration:

```bash
systemctl restart grafana-server.service
```

#### DNS Configuration #####

Now that you have chosen your preferred monitoring [URL](https://en.wikipedia.org/wiki/URL), you need to configure it on your [DNS](https://en.wikipedia.org/wiki/Domain_Name_System) with an [A record](https://en.wikipedia.org/wiki/List_of_DNS_record_types) that points to your server. Follow your DNS provider's guide to do so. For example, if your server [IP address](https://en.wikipedia.org/wiki/IP_address) is ```1.2.3.4```, and your URL is ```grafana.example.com```, you will need an A record similar to the following:

```text
Type: A, Host: grafana, Value: 1.2.3.4
```

Here's Namecheap guide as a pointer: [How do I set up host records for a domain?](https://www.namecheap.com/support/knowledgebase/article.aspx/434/2237/how-do-i-set-up-host-records-for-a-domain)

**Make sure that your A record is working correctly and that is has propagated**, before proceeding with the rest of the guide. The following should return your server IP address, if it doesn't your record hasn't propagated yet.

```bash
dig grafana.example.com +short a
```

#### Firewall Configuration ####

In order for the next steps to work, and to be able to remotely connect to your monitoring, the firewall need to be open for both port ```80``` and ```443```. Issue these commands to do so:

```bash
ufw allow 80
ufw allow 443
```

```bash
ufw reload
```

To verify that everything has worked, issue the following:

```bash
ufw status verbose
```

#### Nginx Reverse Proxy ####

[Nginx](https://www.nginx.com/) is a modern and reliable web server, in this case you'll be using it to [reverse-proxy](https://en.wikipedia.org/wiki/Reverse_proxy) to Grafana. The main reason to be using a web server, rather than the Grafana built-in, is to take advantage of the [SSL encryption](https://www.ssl.com/faqs/faq-what-is-ssl/) to securely connect to and peruse your remote monitoring, [while keeping safe from prying eyes](https://www.youtube.com/watch?v=-enHfpHMBo4).

Configuring it is very easy, and it involves three simple steps:

1. disabling the default server.
2. configuring the reverse proxy.
3. implementing SSL.

To disable the default Nginx server, simply remove it from ```sites-enabled```:

```bash
rm -f /etc/nginx/sites-enabled/default
```

To configure the reverse-proxy, add this configuration to ```/etc/nginx/sites-available/grafana```

```bash
server {
  listen 80;
  server_name grafana.example.com;
  return 301 https://grafana.example.com$request_uri;
}

server {
  listen 443;
  server_name grafana.example.com;

  location / {
    rewrite /(.*) /$1 break;
    proxy_pass http://localhost:3000/;
    proxy_redirect off;
    proxy_set_header Host $host;
  }
}
```

Enable it:

```bash
cd /etc/nginx/sites-enabled/
ln -sf ../sites-available/grafana
systemctl restart nginx.service
```

#### Let's Encrypt ####

[Let's Encrypt is](https://letsencrypt.org/about/) an amazing initiative that aims to bring SSL to everyone, for free. Bigger companies and institutions may still need an higher grade of SSL, but Let's Encrypt is perfect for small time sites, like blogs, small indie websites, and your pool. It is also very smart and easy to configure, with [EFF](https://eff.org/)'s [Certbot](https://certbot.eff.org/). Certbot automates the process of creating, managing, and keeping [Let's Encrypt](https://letsencrypt.org/) certificates up to date for you. All it takes is this simple command:

```bash
certbot --nginx -d grafana.example.com
```

If you need a more detailed guide to help you with ```certbot```, Digital Ocean has [a great guide](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-Ubuntu-10) that can help.

When you get to this point:

```bash
Please choose whether or not to redirect HTTP traffic to HTTPS, removing HTTP access.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
1: No redirect - Make no further changes to the webserver configuration.
2: Redirect - Make all requests redirect to secure HTTPS access. Choose this for
new sites, or if you're confident your site works on HTTPS. You can undo this
change by editing your web server's configuration.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Select the appropriate number [1-2] then [enter] (press 'c' to cancel):
```

Choose option ```1```, since your ```/etc/nginx/sites-available/grafana``` already includes a proper redirection. If you accidentally chose option ```2```, remove these lines from your configuration:

```bash
if ($host = grafana.example.com) {
  return 301 https://$host$request_uri;
} # managed by Certbot
```

Upon the successful Certbot run, your Nginx configuration will be changed to use SSL. Do not touch it, as it is used and managed by Certbot at this point. Certbot will also set and run a ```systemd-timer``` to renew the SSL certificate when necessary.

Restart Nginx with:

```bash
systemctl restart nginx.service
```

#### Installing Dashboards ####

It is now time to connect to Grafana by browsing to ```https://grafana.example.com```. Login with the default credentials (user: ```admin```, password: ```admin```). Upon a successful login, you will be asked to create a new password for the admin user.

It is time to install two dashboards. No need to reinvent the wheel here, please follow the official documentation if you need guidance: [Grafana Export/Import](https://grafana.com/docs/grafana/latest/reference/export_import/). Install the following dashboards to make use of Node Exporter and Jormungandr Exporter:

- Node Exporter Dashboard: [https://grafana.com/grafana/dashboards/1860](https://grafana.com/grafana/dashboards/1860)
- Jormungandr Dashboard: [gacallea/cardanoRelatedStuff/master/monitoring/jormungandr-monitor.json](https://raw.githubusercontent.com/gacallea/cardanoRelatedStuff/master/monitoring/jormungandr-monitor.json)

## What's Next ##

Congratulations!!! If you made it this far, you are running a leader candidate node for your pool, and you know its state and have an history thanks to monitoring. This is only the beginning, though. Running a successful pool takes more than having a good uptime. The pool needs to participate in the network, and crunch blocks. To do so, it needs delegations, **a lot of them**, and to be scheduled to participate into the blocks generation, and win them too.

### Helping Hands ###

One thing I can help you with, is to provide you with tools that will help you manage your server and your node.

```jor_wrapper``` and ```node_helpers``` are a set of ```bash``` scripts to help pool operators manage their nodes. These spun off [Chris G ```.bash_profile```](https://github.com/Chris-Graffagnino/Jormungandr-for-Newbs/blob/master/config/.bash_profile). I have *ported them to bash (scripts)*, improved some of the commands, adapted others to the ```NACG``` guide setup, and implemented brand new features and scripts.

Head over to the [**scripts page**](SCRIPTS.md) to learn about ```jor_wrapper``` and the ```node_helpers```. In there, you will also find suggested server management commands and tools, examples, teaser screenshots, and more resources. Follow [**insaladaPool**](https://twitter.com/insaladaPool)  on Twitter for future updates.

### Operator Resources ###

There a number of useful community created tools, guides, [scripts](SCRIPTS.md), and sites, that can be very helpful for a pool operator. Hereby you find a constantly updated collection of what can be useful to a pool operator. If you are aware of more useful pool operators tools, please be kind and suggest them in an [issue](https://github.com/gacallea/cardanoRelatedStuff/issues) on Github, for inclusion.

#### Pool Tool ####

One very useful site, is [**PoolTool**](https://pooltool.io/) by [papacarp](https://twitter.com/mikefullman). It has all sort of network and pools statistics, and offers a number of useful tools for pool operators. One of them is about one's own pool health and status. Create an account and register your pool, to keep others informed about the state of your pool. Here's [mine](https://pooltool.io/pool/93756c507946c4d33d582a2182e6776918233fd622193d4875e96dd5795a348c) as an example.

#### Stake Pool Bootstrap ####

A **must have community resource** for who's just starting their pool operator journey, **where we all help each others grow**, is [The Cardano Stake Pool Bootstrap Initiative](https://www.adafrog.io/bootstrap.php). It is a [Telegram group](https://t.me/StakePoolBootstrapChannel), where it is possible to participate if you follow some simple rules, where to stake with each others in turn, **to give small pools a chance**.

Anyone can join the party, as long as their pool meets these simple requirements to be eligible:

1. Delegate to other pools in the list.
2. Have a ticker (registered on CF GitHub).
3. Have less than 5M ADA already delegated to the pool.

Join us, and make sure to read the pinned message for all of the nitty gritty details.

#### Organic Design ####

Organic Design has a great deal of useful information on [Cardano](https://organicdesign.nz/Cardano), [Staking Pool FAQ](https://organicdesign.nz/Cardano_staking_pool_FAQ), and [Cardano terminology](https://organicdesign.nz/Cardano#Staking_in_Cardano). Familiarize with these, and your pool operator journey will improve a lot.

#### ADA Stat ####

[ADA Stat](https://adastat.net/en) offers a minimalistic and elegant approach to stats for the explorer and pools. It is accurate and a pleasure to use and look at. Here's INSL as an example: [Insalada Stake Pool](https://adastat.net/en/pool/93756c507946c4d33d582a2182e6776918233fd622193d4875e96dd5795a348c) statistics.

#### ADAtainement ####

[**Adatainement**](https://www.adatainment.com/) is one of the oldest community driven, informative site about Cardano. It also offers graphs, and statistics. Moreover it offers mobile apps, a calculator and a number of other useful tools. Be sure to check them out.

#### Adapools ####

In the same fashion to the above-mentioned Pool Tool, [**Adapools**](https://adapools.org/) offers a number of useful network statistics and pool operators' tools. You can find tools that check against the explorer to understand [if you are forked](https://adapools.org/amiforked); info on [what peers are currently best](https://adapools.org/peers) for your bootstrap, [blocks statistics](https://adapools.org/blocks), and more.

#### Pegasus App ####

[Pegasus App](https://pegasuspool.info/mobile) is a mobile application that provides statistics like the above sites, for pools and the explorer, and it can be with you at all times.

### Telegram ###

Last but not least, should you need help at any stage of your pool operator journey, join the '[Cardano Shelley Testnet & StakePool Best Practice Workgroup](https://t.me/CardanoStakePoolWorkgroup)' group on Telegram; it is packed with knowledge, and great and helpful people.

Insalada Stake Pool also has a [Telegram chat](https://t.me/insaladaPool), should you want to follow us and ask anything about INSL :)
