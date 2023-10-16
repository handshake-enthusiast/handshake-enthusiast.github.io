# How to setup a listening Handshake node on Google Cloud for free

One of the easiest ways to make a substantial contribution to Handshake is to setup a full node that allows inbound connections. Reliable full nodes with inbound connection slots are the backbone of a healthy p2p network. We are going to setup such a node and call it "a listening node" because it listens to inbound connections. Our node will be able to support other full nodes running [`hsd`](https://github.com/handshake-org/hsd) like [Bob wallet](https://bobwallet.io) and light clients running [`hnsd`](https://github.com/handshake-org/hnsd) like [Fintertip](https://impervious.com/fingertip) or `hsd --spv` like Bob wallet in SPV mode.

Below you can find a complete guide. Before you begin please consider starting a countdown to share in comments how much time it took you to setup a listening Handshake node on Google Cloud.

Plan:  
0. [Register on Google Cloud](#register-on-google-cloud)  
1. [Create a project](#create-a-project)  
2. [Create an instance](#create-an-instance)  
3. [Connect to your instance](#connect-to-your-instance)  
4. [Configure the instance](#configure-the-instance)  
5. [Install hsd](#install-hsd)  
6. [Run hsd](#run-hsd)  
7. [Configure hsd](#configure-hsd)  
8. [Open ports](#open-ports)  
9. [Run hsd as a listening node](#run-hsd-as-a-listening-node)  
10. [Make hsd start on system boot](#make-hsd-start-on-system-boot)  
11. [Monitor progress](#monitor-progress)  
12. [Monitor uptime](#monitor-uptime)  
13. [Closing notes](#closing-notes)  

## Register on Google Cloud  

To register on Google Cloud you need to register on Google first. Then go to [https://cloud.google.com/](https://cloud.google.com/) and signup for the free trial.

**90-day, &#36;300!**  

This is a generous gift from Google Cloud. As [the documentation page](https://cloud.google.com/free/docs/free-cloud-features#free-trial) mentions:  
>The Free Trial provides you with free Cloud Billing credits to pay for resources used while you learn about Google Cloud.

There are some restrictions like  
>For example, you may not use Google Cloud services to mine cryptocurrency during your Free Trial.  

However, they are not relevant to our use case.

## Create a project

Go to [https://console.cloud.google.com/projectcreate](https://console.cloud.google.com/projectcreate), specify a project name like "Handshake listening nodes" and click "Create". After the project is created select it from the notifications dropdown on the right or from the projects selector dropdown on the left at the top. You should appear on the dashboard page for your project with a URL like [https://console.cloud.google.com/home/dashboard?project=handshake-listening-nodes](https://console.cloud.google.com/home/dashboard?project=handshake-listening-nodes).

## Create an instance

Now go to [https://console.cloud.google.com/compute/instancesAdd](https://console.cloud.google.com/compute/instancesAdd) and enable Compute Engine API for your project. Once it's enabled you should be redirected to an instance creation page. There are several options to choose from, more on that below.

---

### Name

While naming is considered to be a hard problem in computer science this point is actually the most trivial among others. Just name it as you wish, e.g. "handshake-listening-node" or even better append a region like "handshake-listening-node-us-west1".

### Region and zone  

This one is a bit trickier as there are many options to choose from. A good idea would be to choose the closest region to you with Low CO<sub>2</sub>. [A regions map](https://cloud.google.com/about/locations#regions), [a regions list](https://cloud.google.com/compute/docs/regions-zones#available) and [region picker](https://cloud.withgoogle.com/region-picker/) may come handy.

### Machine type

I'd say a general-purpose machine is a good candidate for this job. The E2 series seem to be giving the best price and start from low power machines. Choose the machine type based on how much of your Free Trial you want to spend (remember, you got &#36;300 to spend within 90 days!), how many machines you want to launch and whether you want to maintain it after your Free Trial is over. <del>Even the smallest e2-micro machine with 1 GB RAM can handle hundreds of inbound connections.</del> **UPD: I do not recommend this option.** Fortunately, you will be able to change the machine type later.  

Here are my results:  
- <del>0.25-2 vCPU (1 shared core), 1 GB memory</del> I do not recomment this instance type. Even during the synchronisation process it goes down regularly.  
- 0.5-2 vCPU (1 shared core), 2 GB memory  
- 1-2 vCPU (1 shared core), 4 GB memory  
- e2-standard-2 (2 vCPU, 8 GB memory)  
- e2-standard-4 (4 vCPU, 16 GB memory)  
- e2-standard-8 (8 vCPU, 32 GB memory)  

### Boot disk size

By default it's 10 GiB. This size is quite obviously not enough to store the blockchain. As of today at the time of publishing (2023-10-16) at height 194552 the whole database takes more than 60 GiB:  

- `tree/` – 35 GiB  
- `blocks/` – 22 GiB  
- `chain/` – 4.3 GiB  

So considering the database grows over time set a higher value. It depends on how long you want to run your node without thinking about the disk space. It might be a good idea to add at least 20 GiB to the current blockchain size, so you don't worry about the nearest future. Remember this disk is going to contain everything including OS. You can easily increase the disk size later when necessary. I set it's size to 120 GiB and a reminder to revisit this in the upcoming months.

### Boot disk type

I haven't really done any measurements on how the disk type influences the node's performance. After reading about [disk types](https://cloud.google.com/compute/docs/disks#disk-types) and their [pricing](https://cloud.google.com/compute/disks-image-pricing#disk) I made a simple conclusion that a balanced persistent disk should deliver good price/value in this case. It's SSD which is supposedly better than HDD. A standard persistent disk is noticeably cheaper, but it's HDD. A performance (SSD) persistent disk is noticeably more expensive, but likely provide a tiny benefit over a balanced persistent disk.

Overall, in my view disk space is quite expensive, but again – remember, you got &#36;300 to spend!

### Boot disk image

For this tutorial I use the latest Debian OS available. At the time of writing it's Debian GNU/Linux 11 (bullseye).

### SSH keys

As the instance creation page mentions:  
>By default, when you connect to a VM using this console or gcloud, your SSH keys are generated automatically. [Learn more](https://cloud.google.com/compute/docs/instances/ssh)

If you want to use your usual console to access the instance it would be better to add your own SSH key. To do so expand Advanced options -> Security -> MANAGE ACCESS and click "Add item" right below "Add manually generated SSH keys".

### Network settings

It's important to configure firewall to allow inbound connections to mainnet [default ports](https://hsd-dev.org/guides/config.html#default-ports) 12038 and 44806. In order to do so you need to add corresponding firewall rules to your VPC network.  

For SSH I set to allow traffic from a single IP address in order to avoid any kind of brute-force attacks.  

---

After you configured the new instance settings you are ready to proceed with it's creation. Click "Create".

## SSH into the instance

Now you have created an instance and should be redirected to [https://console.cloud.google.com/compute/instances](https://console.cloud.google.com/compute/instances?project=handshake-listening-nodes). To access the instance you can click SSH in the Connect column or expand the dropdown to the right and see the full options list you are given. If you added your SSH key then you can use your own terminal to ssh into the instance. To see a username to ssh into the instance you can click on the instance in the instances table, scroll down to the SSH keys section and find your key. Use that username together with the external IP.

## Configure the instance

### Configure your locale

If you see a warning like `-bash: warning: setlocale: LC_ALL: cannot change locale (en_US.UTF-8)` when you ssh into the instance it can be fixed with the following command:  

```
sudo apt install locales
sudo locale-gen
sudo dpkg-reconfigure locales
```

And choose your preferred locale like `en_US.UTF-8`.

### Update packages

At the moment of writing `Processing triggers for man-db (2.9.4-2) ...` takes an enormous amount of time during the upgrade. This may significantly speed up the upgrade process:

```
sudo apt --purge autoremove man-db
```

Update packages:  

```
sudo apt update --allow-releaseinfo-change
sudo apt full-upgrade
```

This can take a while.

Reboot before starting the next section:  

```
sudo reboot
```

## Install hsd

Now we are ready to setup [https://github.com/handshake-org/hsd](https://github.com/handshake-org/hsd) which is "Handshake Daemon & Full Node" as stated in it's description.

[The installation doc](https://github.com/handshake-org/hsd/blob/master/docs/install.md) says that Node.js is required, so we first should install it:  

```
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
sudo apt install nodejs
node -v
```

Mine is v18.12.1 which is [supported](https://github.com/handshake-org/hsd/blob/master/.github/workflows/build.yml#L57).

To be able to clone a git repository while building hsd from source git first need to be installed:  

```
sudo apt install git
```

It installs an outdated version, but we don't really care for our needs.

Now after above preparations we are ready to clone the repo and build `hsd` from source.  

Clone the repo:

```
git clone --depth 1 --branch latest https://github.com/handshake-org/hsd.git
```

It is generally a good idea to verify the hsd code downloaded from Github. You need to import public PGP keys from [https://github.com/handshake-org/hsd/security/policy](https://github.com/handshake-org/hsd/security/policy):  

```
gpg --keyserver hkps://keyserver.ubuntu.com --recv-keys "B4B1F62DBAC084E333F3A04A8962AB9DE6666BBD"
gpg --keyserver hkps://keyserver.ubuntu.com --recv-keys "E61773CD6E01040E2F1BD78CE7E2984B6289C93A"
gpg --keyserver hkps://keyserver.ubuntu.com --recv-keys "D2B2828BD29374D5E9BA3E52CCE677B05CC0FE23"
```

Now you can enter the downloaded repo and verify it's a good signature from one of the project maintainers:  

```
cd hsd
git log --show-signature
```

or  

```
git verify-tag latest
```

Before building `hsd` you need to run the following command:  

```
sudo apt install build-essential
```

Otherwise the build process will result into an error `npm ERR! gyp ERR! stack Error: not found: make`.

**Note this does not verify dependencies**, they are downloaded from the npm. Now, after you verified the signature, you are ready to install the hsd dependencies:  

```
npm install --omit=dev
```

The build may take some time. After the successful installation you can check your `hsd` version.  

```
./bin/hsd --version
```

## Run hsd

Basically, you are ready to run your full node:  

```
~/hsd/bin/hsd --no-wallet
```

You will see it immediately starts to sync the whole blockchain. It's not a listening node yet. Further configuration is required.

## Configure hsd

While your node is being synchronized block by block we should proceed and configure it to be a listening node. Ssh into the instance in a separate tab.

As mentioned on [https://hsd-dev.org/guides/config.html](https://hsd-dev.org/guides/config.html):
>By default, the mainnet hsd config files will reside in `~/.hsd/hsd.conf` (node) and `~/.hsd/hsw.conf` (wallet).  

Let's edit the node configuration file to make it a public listening node that accepts inbound connections from other nodes and SPV clients:  

```
nano ~/.hsd/hsd.conf
```

Paste the following content:  

```
listen: true
max-inbound: 100
public-host: <IP address>
bip37: true
agent: enthusiast
```

Where:  

- `listen: true` is necessary to accept incoming connections.  
- `max-inbound` depends on [the machine type](#machine-type) you setup.  
- `<IP address>` is the actual IP address of the instance.  
- `bip37: true` is necessary to serve light clients like Bob Wallet in SPV mode. [`hnsd`](https://github.com/handshake-org/hnsd) can still connect to full nodes that do not have `bip37` enabled. 
- `agent: enthusiast` is the only optional setting. You can simply omit it.

You can read more about these settings on [https://hsd-dev.org/guides/config.html](https://hsd-dev.org/guides/config.html).  

As you probably previously noticed we passed `--no-wallet` as command line argument for `hsd`. This configuration setting wouldn't take any effect if placed in `hsd.conf`. As stated on [https://hsd-dev.org/guides/config.html#/etc/systemd/system/](https://hsd-dev.org/guides/config.html#/etc/systemd/system/):  

>The following configuration settings are only available for the command line when hsd is launched. They WILL NOT be read from a `hsd.conf` file or pulled from the shell environment.  
>
>- `--no-wallet`: Launches hsd without a wallet plugin, allowing `hs-wallet` to be run in a separate process.  

## Open ports

(This is already covered by the Network settings configuration during the instance creation. However, I'm leaving this section as an alternative firewall configuration in case your cloud provider permits all incoming internet traffic. Feel free to skip to the next step and run hsd as a listening node.)  

In order to allow others to connect to your node you need to open port 12038 and 44806. We'll open them using [`ufw`](https://en.wikipedia.org/wiki/Uncomplicated_Firewall):  

```
sudo apt install ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow 12038
sudo ufw allow 44806
sudo ufw enable
```

This should result into the following firewall settings:

```
$ sudo ufw status verbose
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere                  
12038                      ALLOW IN    Anywhere                  
44806                      ALLOW IN    Anywhere                  
22/tcp (v6)                ALLOW IN    Anywhere (v6)             
12038 (v6)                 ALLOW IN    Anywhere (v6)             
44806 (v6)                 ALLOW IN    Anywhere (v6)   
```

If you have a static IP address (e.g. from your home network or personal VPN) you can limit IP addresses that can connect via SSH port to your static one:  

```
sudo ufw allow from YOUR.STATIC.IP.ADDRESS to any port 22 proto tcp
sudo ufw delete allow ssh
sudo ufw status verbose
```

## Run hsd as a listening node

Your node should be running in a different tab without the configuration specified in `~/.hsd/hsd.conf` yet. You can verify that fact by running `~/hsd/bin/hsd-cli info`. In the command output you should be able to find:  

```
"public": {
  "listen": false,
  "host": null,
  "port": null,
  "brontidePort": null
}
```

Now let's stop it using `~/hsd/bin/hsd-cli rpc stop` and run it again in the first tab using the same command as before:  

```
~/hsd/bin/hsd --no-wallet
```

Let's check the output of `~/hsd/bin/hsd-cli info` again:  

```
"public": {
  "listen": true,
  "host": "<IP address>",
  "port": 12038,
  "brontidePort": 44806
}
```

## Make hsd start on system boot

We'll use `systemd` to run our `hsd` script on boot up.

Run `sudo nano /etc/systemd/system/hsd.service`, copy-paste and save the following file content:  

```
[Unit]
Description=Handshake daemon and full node
After=network.target

[Service]
Type=simple
User=USERNAME
Group=USERNAME
ExecStart=/home/USERNAME/hsd/bin/hsd --no-wallet
ExecStop=/home/USERNAME/hsd/bin/hsd-cli rpc stop

[Install]
WantedBy=default.target
```

Replace `USERNAME` with actual username you used to ssh into the instance.

Refresh the systemd configuration files:  

```
sudo systemctl daemon-reload
```

And enable the service to start automatically on system boot:  

```
sudo systemctl enable hsd.service
``` 

Check the service status:  

```
sudo systemctl status hsd.service
```

Stop `hsd` running in foreground in the first tab:  

```
~/hsd/bin/hsd-cli rpc stop
```

You can conveniently start and stop the service now:  

```
sudo systemctl start hsd.service
sudo systemctl stop hsd.service
```

Try to reboot the instance with `sudo reboot`. After it's up and running again you can verify the hsd node is up and running as well. Ssh into the instance and run `~/hsd/bin/hsd-cli info` or check the hsd service status `sudo systemctl status hsd.service`. 

## Monitor progress

`~/hsd/bin/hsd-cli info | grep progress` should show you which percentage of the blockchain you added to your node's blockchain. Once it's synchronized you'll see `"progress": 1,`. You can set a reminder for yourself to check the progress in a day. I suppose the progress mostly depends on your machine type and other listening nodes resources. My seed node synchronised in less than 7 hours! This is a fantastic result in my view. I believe that's because of the latest powerful tech stack I'm using.  

## Monitor uptime

In the meantime let's make sure we are immediately notified when there are any issues with the node. (Hopefully, there won't be any.)

Let's setup some free uptime monitoring. You can search for "cron monitoring" and choose a SaaS solution to use based on your needs and preferences. After a bit of googling and pricing pages comparison I chose [cronitor.io](https://cronitor.io/pricing). It's free plan allows 5 monitors seemingly without frequency limitations and what was the key selling point for me which I was particularly looking for – it allows to receive Telegram notifications. It's up to you which service to choose, the approach will be the same. Likely you can even use Google Cloud Monitoring, I haven't explored it yet.  

May be later I'll cover how to setup monitoring on cronitor.io in details in a separate article, but here I'll just cover the basics:  

1. Configure alerts/notifications – I added Telegram notifications besides Email notifications. You may want to setup other notifications like Slack, Teams, Webhook, SMS, WhatsApp and so on.  
2. Create a Heartbeat – I named it exactly like an instance, setup a heartbeat.schedule to be run every minute and made Grace Period to be one minute as well.  
3. Add a crontab entry:  

```  
crontab -e
* * * * * systemctl is-active --quiet hsd && curl https://cronitor.link/p/abcdef...123456/aBcDeF
```  

4. Check it's working by stopping and starting the daemon while waiting for corresponding notifications 

```
crontab -l
sudo systemctl stop hsd
sudo systemctl status hsd
sudo systemctl start hsd
```  

Coincidentally, the monitor proved it's usefulness the next day after I set it up for the first time. I woke up and saw several up and down notifications during a short period of time until the node went completely down. When I looked at `sudo systemctl status hsd` there was a log entry `hsd.service: A process of this unit has been killed by the OOM killer.`. This happened because it was the smallest possible machine on Google Cloud with just 1 GB of RAM and it was still synchronising it's yet incomplete blockchain.

There is a nice Live Status Badge available:  
![Handshake seed node Mumbai](https://cronitor.io/badges/6vhCDn/production/2bHaU9W85BCvBmbhzL_pf_2YzwY.svg)  

As well as a [Status Page](https://empty-bonus-92feac03.cronitorstatus.com).

## Closing notes

Hooray! Now you are running a listening node on AWS and making Handshake stronger. 🤝

Alternatively you can follow a guide [How to setup a listening Handshake node on AWS](/how-to-setup-listening-handshake-node-on-aws).

I'd like to express my gratitude to Erik Cederwall and [Open Systems](https://opensystems.dev). [Setting up the seed node on AWS](/how-to-setup-listening-handshake-node-on-aws) was partially [sponsored](https://github.com/opensystm/handshake-micro-grants/issues/5) by Handshake Micro Grants. Also I'd like to thank [@befranz](https://github.com/befranz) who did and described [similar work](https://www.reddit.com/r/handshake/comments/poo5iw/each_handshake_node_depends_on_other_listening/) a couple years ago. I learnt a lot from it. Also thanks to [@pinheadmz](https://github.com/pinheadmz), [@rithvikvibhu](https://github.com/rithvikvibhu) and many other Handshake Enthusiasts who helped me to learn from their countless contributions. You are awesome!

Would be great to hear from you in the comments section below. I'm curious where you are from, how you resolve this website, which cloud provider and tech stack you are using.

---

_© 2023 Handshake Enthusiast_

_All rights reserved. You may share the link to this article, but the content itself may not be reproduced or distributed without my express permission. I retain all copyright. I intend to republish this article in the future under the Creative Commons Attribution 4.0 International License (CC BY 4.0), which will permit sharing and adaptation with appropriate credit._
