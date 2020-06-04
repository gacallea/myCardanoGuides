# Guides For Cardano Pool Ops

Hereby collected, my humble contributions to the Cardano ecosystem. A set of detailed guides, with custom scripts, to help you install, configure, manage, automate, and monitor a  Cardano pool.

## The Guide ##

This guide is written with experienced users in mind. Things like creating a GitHub account, creating and using a pair of ssh keys, are a given. If you think you need help with those - there's nothing wrong with it - you should refer to Chris's [guide for newbs](https://github.com/Chris-Graffagnino/Jormungandr-for-Newbs/blob/master/docs/jormungandr_node_setup_guide.md).

This guide won't reinvent the wheel either. Its focus are the system and the node itself, and it will point you to [IOHK](https://iohk.io/)'s, when it's time to create, fund, and register your pool. IOHK [**guide**](https://github.com/input-output-hk/shelley-testnet/blob/master/docs/stake_pool_operator_how_to.md) and [**scripts**](https://github.com/input-output-hk/jormungandr-qa/tree/master/scripts) are all you need, and they are **official**.

The only note worth adding, before you venture into configuring a server and creating a pool, is that **you need to have enough tADA** - meaning ADA coins that were **in your wallet before the November 2019** snapshot - to register your pool. Otherwise, you won't be able to proceed with the pool registration.

**IMPORTANT**: this guide helps you configure and run a single pool with a single leader candidate node. If you are planning to run a multi-nodes cluster, this guide assumes that you know what you are doing. Having said that, a guide to satisfy a cluster configuration, complete with scripts to manage it, is coming soon. Stay tuned.

### License ###

This guide is licensed under the terms of a Creative Commons [**CC BY-NC-SA 4.0**](https://creativecommons.org/licenses/by-nc-sa/4.0/). If you are not familiar with Creative Commons licenses, here's a bit [I wrote about them](https://gacallea.info/posts/a-primer-on-linux-open-source-and-copyleft-hackers-included/#creative-commons) that clarify what Creative Commons licenses are about.

### System ###

This guide will be focusing on the latest **Debian stable**, and assumes you already have your server installed with a vanilla Debian 10 (Buster), **that you can already ```ssh``` into on port ```22```**. This guide will stick to official repositories, with the exception for ```jormungandr```, ```jcli```, and ```tcpping```. This guide will stick with best practices for stability (e.g: [backports](https://backports.debian.org/) vs [unstable](https://wiki.debian.org/DontBreakDebian)).

Lastly, this guide assumes that you are familiar with Linux, its shell, its commands and have some experience in managing a server. This guide will promote system administration over shortcuts and aliases. For example, it will favor and configure ```systemd``` over aliases to manage the node. It will favor using ```pidof jormungandr``` instead of configuring an alias. In this guide you'll be also configuring system wide logging, and using ```journalctl``` over typing "*logs*". You get the idea.

### Updates ###

Updates are implemented only after I'll have done so on my pool, and tested it. I will add more sections to the guide as needed. The same goes for any useful feature that could help pool owners run and manage the server and the pool. Follow [insaladaPool](https://twitter.com/insaladaPool) for future updates.

### Contributions ###

**If you have comments, issues, changes, and suggestions, please [file an issue](https://github.com/gacallea/cardanoRelatedStuff/issues) on Github**. Any insight is valuable and will be considered for integration and improvements. If these resources help you in any way, consider [buying me a beer](https://seiza.com/blockchain/address/Ae2tdPwUPEZHwvuNhu7qGeBcZBTQAwL2SUA49T6CubbQzoxgxyffYJ8VvcW).

## The Guide ##

This guide builds upon, extends, and supersedes the [Not Another Cardano Guide](NACG.md). ```NACG``` focused on a single Jormungandr node. ```ITN1 Cluster``` helps you install, configure, automate and manage a three nodes cluster pool **on the same server**.

Depending on your server resources, you may want to run less (or more) nodes. Three nodes are used here, because it's what [INSL](https://insalada.io) runs, and have been tested in production. Better still, you could run each node on its own dedicated VPS instance, and scale up to as much as you can afford. That would require modifications to this guide and to the scripts. You are free to experiment. As much as I would love to expand the guide to multiple VPS or Docker, I can't afford to test those scenarios.

Having said that, running a *three-Jormungandr-nodes cluster* **does not mean** that you'd have three leaders running at all times. That would cause a number of issues to the network. Since you want to avoid [forks](https://pooltool.io/health), a single leader candidate node will be active at any given time. Each node starts as leader, and it's demoted on some conditions. This *multi-leader strategy* is used by many operators and, as of now, it's the best way to run multi-nodes. At least until [*the passive strategy*](https://github.com/input-output-hk/jormungandr/issues/1551) is possible.

Lastly, both this guide and installing, configuring, and managing a cluster requires a certain degree of Linux skills. And familiarity with its shell, and its commands. And that you have, at least, some experience in managing a a Linux distro.

### System ###

This guide focuses on the latest **Debian stable**, and assumes you already have a server installed with **a vanilla Debian 10 (Buster), that you can already ```ssh``` into on port ```22```**. Only official repositories are used, with the exception for ```jormungandr```, ```jcli```, and a few other tools. This guide adheres to best practices as well, e.g: [backports](https://backports.debian.org/) vs [unstable](https://wiki.debian.org/DontBreakDebian).

### Server ###

[Insalada Stake Pool](https://insalada.io) runs on a hosted bare metal with the following specs, and handles three nodes, configured as illustrated in this guide, quite solidly. You could use the specs as a reference, to choose the number of nodes you may want to run.

| Resource | Specs                               |
| -------- | ----------------------------------- |
| CPU      | Intel  Xeon W3520 (4c/8t - 2,66GHz) |
| RAM      | 16GB DDR3 ECC 1333 MHz              |
| SSD      | SoftRaid 2x2TB                      |
| Network  | 100Mpbs                             |
| Traffic  | Unlimited                           |

### Assumptions ###

It would be impossible to cover every single scenario that a user could come up with, therefore this guide makes some assumptions during the process of installation and in the provided set of scripts. Examples and default values are provided, both in the guide and in the configuration files. It would still be possible for you to change the values of course. However, if unsure, sticking with the provided defaults is a safe choice. Should you decide to make changes, they should adhere to the same underlying logic of the default values. Otherwise the installation could fail or be troublesome.

### Updates ###

Updates are implemented only after they have been tested on my pool. Sections are added to the guide as needed. The same goes for useful features in the pool management scripts. Follow [insaladaPool](https://twitter.com/insaladaPool) for updates.

### Contributions ###

**If you have comments, issues, changes, and suggestions, please [file an issue](https://github.com/gacallea/itn1_cluster/issues) on Github**. Any insight is valuable and will be considered for integration and improvements. If these resources help you in any way, consider [buying me a beer](https://seiza.com/blockchain/address/Ae2tdPwUPEZHwvuNhu7qGeBcZBTQAwL2SUA49T6CubbQzoxgxyffYJ8VvcW).

### License ###

This guide is licensed under the terms of a Creative Commons [**CC BY-NC-SA 4.0**](https://creativecommons.org/licenses/by-nc-sa/4.0/). If you are not familiar with Creative Commons licenses, here's a bit [I wrote about them](https://gacallea.info/posts/a-primer-on-linux-open-source-and-copyleft-hackers-included/#creative-commons) that clarify what Creative Commons licenses are about.
