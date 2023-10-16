# How to setup a listening Handshake node on AWS

One of the easiest ways to make a substantial contribution to Handshake is to setup a full node that allows inbound connections. Reliable full nodes with inbound connection slots are the backbone of a healthy p2p network. We are going to setup such a node and call it "a listening node" because it listens to inbound connections. Our node will be able to support other full nodes running [`hsd`](https://github.com/handshake-org/hsd) like [Bob wallet](https://bobwallet.io) and light clients running [`hnsd`](https://github.com/handshake-org/hnsd) like [Fintertip](https://impervious.com/fingertip) or `hsd --spv` like Bob wallet in SPV mode.

Below you can find a complete guide which I compiled while setting up a Handshake seed node. Before you begin please consider starting a countdown to share in comments how much time it took you to setup a listening Handshake node on Amazon Web Services.

Plan:  
0. [Register on AWS](#register-on-aws)  
1. [Choose a region](#choose-a-region)  
2. [Create an EC2 instance](#create-an-ec2-instance)  
3. [Connect to your instance](#connect-to-your-instance)  
4. [Install hsd](#install-hsd)  
5. [Run hsd](#run-hsd)  
6. [Configure hsd](#configure-hsd)  
7. [Open ports](#open-ports)  
8. [Run hsd as a listening node](#run-hsd-as-a-listening-node)  
9. [Make hsd start on system boot](#make-hsd-start-on-system-boot)  
10. [Monitor progress](#monitor-progress)  
11. [Monitor uptime](#monitor-uptime)  
12. [Closing notes](#closing-notes)  

## Register on AWS

Obviously, you need an account on Amazon Web Services to go ahead. Open the [Create Account](https://aws.amazon.com/resources/create-account/) page and complete the sign up form. Next, you need to choose a region.

## Choose a region

While you may already know which exact region you prefer there are a few things to consider. First, you can visit [https://shakenodes](https://shakenodes) to have an idea which regions are already well covered. Second, AWS prices and availability noticeably vary over different AWS regions. I'd recommend to review [Amazon EC2 On-Demand Pricing](https://aws.amazon.com/ec2/pricing/on-demand/) as well as [EC2 Instance Savings Plans](https://aws.amazon.com/savingsplans/compute-pricing/).

## Create an EC2 instance  

Select a chosen region in the dropdown in the upper right corner. You'll be able to come back later and select another region if you change your mind. In my case the region is Asia Pacific (Mumbai) `ap-south-1`. Open EC2 and click "Launch instance". On this page you need to configure your instance which you are going to launch.

### Name

While naming is considered to be a hard problem in computer science this point is actually the most trivial among others. Just name it as you wish, e.g. "Handshake listening node". In my case it's a seed node, so I named it "Handshake seed node".

### Application and OS Images (Amazon Machine Image)

I kept the standard Amazon Linux which comes by default, but with 64-bit (Arm) architecture due to my instance type.

### Instance type

Amazon EC2 provides a wide selection of instance types optimized to fit different use cases. I suppose a general purpose instance is a good candidate to serve as a listening Handshake node. Choosing a proper EC2 instance type requires some consideration. Learning about the [EC2 instance types](https://aws.amazon.com/ec2/instance-types/) and reviewing the pricing pages mentioned above should help. I chose the latest generation of general purpose instances which is [M7g](https://aws.amazon.com/ec2/instance-types/m7g/) at the moment of writing. As advertised it provides "the best price performance in Amazon EC2 for general purpose workloads". I do not recomment to choose an instance type with less than 2 GiB RAM. To make sure my seed node can serve thousands of clients I chose `m7g.xlarge` which has 4 vCPU and 16 GiB RAM.

### Key pair (login)

To connect to the instance via SSH you can either create new key pair or use your existing SSH key. I chose to use my default SSH key for simplicity.  

To upload your existing SSH key:  

- Open the AWS Management Console in another tab. Make sure you are within the same region where you are setting up an instance.  
- Navigate to the EC2 Dashboard.
- In the "Network & Security" section, click on "Key Pairs".
- Click on the "Import key pair" button in the "Actions" dropdown in the upper right corner.
- Enter key pair name. I named it "My default SSH key" for clarity.
- Copy and paste the contents of your public key (`.ssh/id_XYZ.pub`) into the "Public key content" box.
- Click on the "Import key pair" button.

Go back to the instance creation page. Reload the key pairs list and choose your imported key pair.

### Network settings

For SSH I set to allow traffic from a single IP address in order to avoid any kind of brute-force attacks. You can keep the defaults ("Anywhere" as a Source type) if you don't have a static IP address.  

It's important to configure firewall to allow inbound connections to mainnet [default ports](https://hsd-dev.org/guides/config.html#default-ports) 12038 and 44806. In order to do so you need to create two more Inbound Security Group Rules with "Custom TCP" as a Type and "Anywhere" as a Source type.  

### Storage

After learning EBS [pricing](https://aws.amazon.com/ebs/pricing/) and [volume types](https://aws.amazon.com/ebs/volume-types/) I sticked with the default choice of not encrypted General Purpose SSD (`gp3`). At the moment of writing it's the latest generation of general-purpose SSD-based EBS volumes.

By default it's 8 GiB. This size is quite obviously not enough to store the blockchain. As of today at the time of publishing (2023-10-16) at height 194552 the whole database takes more than 60 GiB:  

- `tree/` – 35 GiB  
- `blocks/` – 22 GiB  
- `chain/` – 4.3 GiB  

So considering the database grows over time set a higher value. It depends on how long you want to run your node without thinking about the disk space. It might be a good idea to add at least 20 GiB to the current blockchain size, so you don't worry about the nearest future. Remember this disk is going to contain everything including OS. You can easily increase the disk size later when necessary. I set it's size to 120 GiB and a reminder to revisit this in the upcoming months.

### Advanced details

I left them as is, i.e. without any changes.

---

After you configured the new instance settings you are ready to proceed with it's creation. Click "Launch instance".

## Connect to your instance

After you successfully initiated launch of instance you should see a button "Connect to instance". If you click on it you can find several options how to connect to the instance. I added my default SSH key and I prefer to use Terminal to ssh into the instance. Public IP address is 3.110.49.183, so to ssh into the instance I simply use the following command:  

```
ssh ec2-user@3.110.49.183
```

## Install hsd

Now we are ready to setup [https://github.com/handshake-org/hsd](https://github.com/handshake-org/hsd) which is "Handshake Daemon & Full Node" as stated in it's description.

[The hsd installation doc](https://github.com/handshake-org/hsd/blob/master/docs/install.md) says that Node.js is required, so we first should [install](https://docs.aws.amazon.com/sdk-for-javascript/v2/developer-guide/setting-up-node-on-ec2-instance.html) it:  

```
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.5/install.sh | bash
. ~/.nvm/nvm.sh
nvm install --lts
node -e "console.log('Running Node.js ' + process.version)"
```

Mine is v18.18.1 which is [supported](https://github.com/handshake-org/hsd/blob/master/.github/workflows/build.yml#L57).

To be able to clone a git repository while building hsd from source git first need to be installed:  

```
sudo dnf install git -y
git --version
```

It installs an outdated version, but we don't really care for our needs.

Now after above preparations we are ready to clone the repo and build `hsd` from source.  

Clone the repo:

```
git clone --depth 1 --branch latest https://github.com/handshake-org/hsd.git
```

It is generally a good idea to verify the hsd code downloaded from Github. You need to import public PGP keys from [https://github.com/handshake-org/hsd/security/policy](https://github.com/handshake-org/hsd/security/policy):  

```
sudo dnf swap gnupg2-minimal gnupg2-full -y
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
sudo dnf groupinstall "Development Tools" -y
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
User=ec2-user
Group=ec2-user
Environment="NVM_DIR=/home/ec2-user/.nvm"
ExecStart=/bin/bash -c 'source $NVM_DIR/nvm.sh && /home/ec2-user/hsd/bin/hsd --no-wallet'
ExecStop=/bin/bash -c 'source $NVM_DIR/nvm.sh && /home/ec2-user/hsd/bin/hsd-cli rpc stop'

[Install]
WantedBy=default.target
```

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

Let's setup some free uptime monitoring. You can search for "cron monitoring" and choose a SaaS solution to use based on your needs and preferences. After a bit of googling and pricing pages comparison I chose [cronitor.io](https://cronitor.io/pricing). It's free plan allows 5 monitors seemingly without frequency limitations and what was the key selling point for me which I was particularly looking for – it allows to receive Telegram notifications. It's up to you which service to choose, the approach will be the same. Likely you can even use Amazon CloudWatch, I haven't explored it yet.  

May be later I'll cover how to setup monitoring on cronitor.io in details in a separate article, but here I'll just cover the basics:  

- Configure alerts/notifications – I added Telegram notifications besides Email notifications. You may want to setup other notifications like Slack, Teams, Webhook, SMS, WhatsApp and so on.  
- Create a Heartbeat – I setup a heartbeat.schedule to be run every minute and made Grace Period to be one minute as well.  
- Setup cron  

```
sudo dnf install cronie -y
sudo systemctl enable crond
sudo systemctl start crond
sudo systemctl status crond
```
 
- Add a crontab entry:  

```  
crontab -e
```  

Replace the template cronitor.link URL with a unique URL you were given:  

```
* * * * * systemctl is-active --quiet hsd && curl https://cronitor.link/p/abcdef...123456/aBcDeF
```

Check it's correctly added:  

```
crontab -l
```

- Check it's working by stopping and starting the daemon while waiting for corresponding notifications 

```
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

Alternatively you can follow a guide [How to setup a listening Handshake node on Google Cloud for free](/how-to-setup-listening-handshake-node-on-google-cloud).

Setting up the seed node was partially [sponsored](https://github.com/opensystm/handshake-micro-grants/issues/5) by Handshake Micro Grants. Thanks Erik Cederwall and [Open Systems](https://opensystems.dev). Without their grant I'm not sure whether setting up this seed node and publishing this blog post would ever happen.

Also I'd like to thank [@befranz](https://github.com/befranz) who did and described [similar work](https://www.reddit.com/r/handshake/comments/poo5iw/each_handshake_node_depends_on_other_listening/) a couple years ago. I learnt a lot from it. Also thanks to [@pinheadmz](https://github.com/pinheadmz), [@rithvikvibhu](https://github.com/rithvikvibhu) and many other Handshake Enthusiasts who helped me to learn from their countless contributions. You are awesome!

Would be great to hear from you in the comments section below. I'm curious where you are from, how you resolve this website, which cloud provider and tech stack you are using.

---

_© 2023 Handshake Enthusiast_

_All rights reserved. You may share the link to this article, but the content itself may not be reproduced or distributed without my express permission. I retain all copyright. I intend to republish this article in the future under the Creative Commons Attribution 4.0 International License (CC BY 4.0), which will permit sharing and adaptation with appropriate credit._

<script src="https://giscus.app/client.js"
        data-repo="handshake-enthusiast/handshake-enthusiast.github.io"
        data-repo-id="R_kgDOKhK0Tg"
        data-category="General"
        data-category-id="DIC_kwDOKhK0Ts4CaL2Y"
        data-mapping="title"
        data-strict="0"
        data-reactions-enabled="1"
        data-emit-metadata="0"
        data-input-position="top"
        data-theme="preferred_color_scheme"
        data-lang="en"
        data-loading="lazy"
        crossorigin="anonymous"
        async>
</script>
