# RPC Node config for Equinix Metal

IMPORTANT: This guide is specifically for Equinix Machines from the Solana Reserve pool accessed throught he Solana Foudnation Server Program. https://solana.foundation/server-program


So you have your shiny new beast of a server. Let's make it a Shadow Operator RPC.

First things first - OS security updates
```
sudo apt update
sudo apt upgrade
sudo apt dist-upgrade
```
create user sol

```
adduser sol

usermod -aG sudo sol

su - sol

```
Partition hard drive for RPC
Partition NVME into 420gb (swap) and 3000gb (ledger)

adding new process using GPT partition with gdisk for larger filessytems. Make larger 3.5 (or 3.8) TB drive an ext4 via gdisk then partition using fdisk as normal. You have to delete the original GPT in order to select partition 1 with fdisk

Enter the "n" then hit enter
Etner the "1" then hit enter...and so on
```
sudo gdisk /dev/nvme0n1
n, 1, 2048, [max secor available], 8300, p, w
```
note the first step in the next section is deleting the partition we just created above
```
sudo fdisk /dev/nvme0n1
d, n, p, 1 or 2, default sector, +3000GB, n, p, 1 or 2, default sector, +420GB, w
```
Now make filessytems, directories, delete and make new swap, etc.
```
sudo fdisk -l 

sudo mkfs -t ext4 /dev/nvme0n1p1

sudo mkfs -t ext4 /dev/nvme0n1p2

sudo mkdir /mnt/

sudo mkdir /mnt/ramdrive

sudo mkdir /mt/

sudo mkdir /mt/ledger

sudo mkdir ~/log
```
Discover the swap directory, turn it off, make a new one and turn it on
```
sudo swapon --show

sudo swapoff /dev/sda2

sudo sed --in-place '/swap.img/d' /etc/fstab

sudo mount /dev/nvme0n1p2 /mnt/

sudo mount /dev/nvme0n1p1 /mt

sudo dd if=/dev/zero of=/mnt/swapfile bs=1M count=350k
```
it can take up to 5 minutes for the machine to make this size swapfile. sit tight.

next is setting permissions and adding the swapfile to fstab, then edit the swapiness to 30.
```
sudo chmod 600 /mnt/swapfile

sudo mkswap /mnt/swapfile

echo 'vm.swappiness=1' | sudo tee --append /etc/sysctl.conf > /dev/null

sudo sysctl -p

sudo swapon --all --verbose
```
capture nvme0n1p1 and nvme0n1p2 UUIDs to edit into /etc/fstab
```
lsblk -f
```
copy the section that looks like this and past it into a notepad (or VScode, etc) so that you can copy/past into fstab properly. We just need the UUID's so in the example below copy "5c24e241-239c-4aa5-baa6-fbb6fb44a847" and "87645b08-85c2-4fe2-9974-1bda4de317d9" and note which partition each belongs to (/mt and /mnt respectively)
```
nvme0n1
├─nvme0n1p1 ext4         5c24e241-239c-4aa5-baa6-fbb6fb44a847    2.8T     0% /mt
└─nvme0n1p2 ext4         87645b08-85c2-4fe2-9974-1bda4de317d9    9.5G    88% /mnt
```
These UUID above need to be edited into the fstab config below
```
sudo nano /etc/fstab
```
dump this into fstab below the current UUIDs. delete or hash out the old dwap UUID if needed. Leave the first UUIDs (OS related), just append these lines under whatever current UUIDs are listed as the ones already in the file are boot/OS related.
also make sure UUID is correct as they can change

Copy the below into a notepad along with the above "lsblk -f" information and then update the UUIDs to be the new ones from the current RPC. THen you paste this into the fstab file mentioned above
```
#GenesysGo RPC config
UUID=5c24e241-239c-4aa5-baa6-fbb6fb44a847 /mt  auto nosuid,nodev,nofail 0 0
UUID=87645b08-85c2-4fe2-9974-1bda4de317d9 /mnt  auto nosuid,nodev,nofail 0 0
#ramdrive and swap
#tmpfs /mnt/ramdrive tmpfs rw,size=50G 0 0
/mnt/swapfile none swap sw 0 0
```
save / exit
ctrl+s, ctrl+x

But Wait - what was that ramdrive and tmpfs stuff? Leave it for now. That is an performance enhancement option that will be covered in later documentation. In short, it's for running the solana accounts inside the memory of the server verssu on the hard drive. More on this later.

now edit permissions and make sure directories are made again (i think the order of ops is a little off here since the reformating of GPT using fdisk whipes the secondary directories of ledger & ramdrive while keeping the mountpoints for the nvme of /mnt and /mt. easy enough to just remake directories though)
```
sudo mkdir /mnt/ramdrive

sudo mkdir /mt/ledger

sudo mkdir /mt/ledger/validator-ledger

sudo ls -ld /mt/ledger

sudo chown sol

sol /mt/ledger

sudo chown sol:sol ~/log

sudo chown sol:sol /mt/ledger/validator-ledger

sudo mount --all --verbose
```

firewall / ssh
```
sudo snap install ufw

sudo ufw enable

sudo ufw allow ssh
```
dump this entire command block
```
sudo ufw allow 80;sudo ufw allow 80/udp;sudo ufw allow 80/tcp;sudo ufw allow 53;sudo ufw allow 53/tcp;sudo ufw allow 53/udp;sudo ufw allow 8899;sudo ufw allow 8899/tcp;sudo ufw allow 8900/tcp;sudo ufw allow 8900/udp;sudo ufw allow 8901/tcp;sudo ufw allow 8901/udp;sudo ufw allow 9900/udp;sudo ufw allow 9900/tcp;sudo ufw allow 9900;sudo ufw allow 8899/udp;sudo ufw allow 8900;sudo ufw allow 8000:8020/tcp;sudo ufw allow 8000:8020/udp
```
# reboot. test mounts, swaps, ramdrive, etc. Reboot should mount to both /nvme0n1p1 and p2 on both /mt and /mt. 


# Install Solana CLI! Don't forget to check version.
these are three seperate commands below:
```
sh -c "$(curl -sSfL https://release.solana.com/v1.7.14/install)"
export PATH="/home/sol/.local/share/solana/install/active_release/bin:$PATH"
solana-gossip spy --entrypoint entrypoint.mainnet-beta.solana.com:8001
```
if the machine is gossiping withour any errors it can be spun up on the mainnet to start reading the chain.

exit gosspy with ctrl + c

create keys.
IF A a validator always set a password for the wallet-keypair key. log it on 1password. make sure the team knows you are storing keys for a validator and get guidance). you must copy seed phrases.
IF RPC then these are throw away keys. no funds are moved and if corrupt or lost just make new ones. You do not need to set a password for the keys. no need to copy seed phrases. You do not need a wallet-keypair if just RPC. Skip to making validator-keypair
```
solana-keygen new -o ~/validator-keypair.json
solana config set --keypair ~/validator-keypair.json
solana-keygen new -o ~/vote-account-keypair.json
```

# making system services (sol.service and systuner.service) and the startup script.

solana-validator shell script
```
sudo vim ~/start-validator.sh
```
dump this into start-validator.sh:
THIS IS VERSION 2 ( updated 07/15/2021)
```
#!/bin/bash
exec solana-validator \
--entrypoint entrypoint.mainnet-beta.solana.com:8001 \
--entrypoint entrypoint2.mainnet-beta.solana.com:8001 \
--entrypoint entrypoint3.mainnet-beta.solana.com:8001 \
--entrypoint entrypoint4.mainnet-beta.solana.com:8001 \
--entrypoint entrypoint5.mainnet-beta.solana.com:8001 \
--trusted-validator 7cVfgArCheMR6Cs4t6vz5rfnqd56vZq4ndaBrY5xkxXy \
--trusted-validator DDnAqxJVFo2GVTujibHt5cjevHMSE9bo8HJaydHoshdp \
--trusted-validator Ninja1spj6n9t5hVYgF3PdnYz2PLnkt7rvaw3firmjs \
--trusted-validator wWf94sVnaXHzBYrePsRUyesq6ofndocfBH6EmzdgKMS \
--trusted-validator 7Np41oeYqPefeNQEHSv1UDhYrehxin3NStELsSKCT4K2 \
--trusted-validator GdnSyH3YtwcxFvQrVVJMm1JhTS4QVX7MFsX56uJLUfiZ \
--trusted-validator DE1bawNcRJB9rVm3buyMVfr8mBEoyyu73NBovf2oXJsJ \
--trusted-validator CakcnaRDHka2gXyfbEd2d3xsvkJkqsLw2akB3zsN1D2S \
--rpc-port 8899 \
--dynamic-port-range 8002-8020 \
--no-port-check \
--gossip-port 8001 \
--no-untrusted-rpc \
--no-voting \
--private-rpc \
--rpc-send-retry-ms 10 \
--enable-cpi-and-log-storage \
--enable-rpc-transaction-history \
--account-index program-id \
--account-index spl-token-owner \
--account-index spl-token-mint \
--rpc-bigtable-timeout 600 \
--enable-rpc-bigtable-ledger-storage \
--no-duplicate-instance-check \
--wal-recovery-mode skip_any_corrupted_record \
--identity ~/validator-keypair.json \
--vote-account ~/vote-account-keypair.json \
--log ~/log/solana-validator.log \
--accounts /mnt/ramdrive/solana-accounts \
--ledger /mt/ledger/validator-ledger \
--limit-ledger-size 200000000 \
--rpc-max-multiple-accounts 10000000 \
--rpc-pubsub-max-in-buffer-capacity 1000000 \
--rpc-pubsub-max-out-buffer-capacity 10000000000 \
--rpc-pubsub-max-active-subscriptions 100000 \
--rpc-pubsub-max-connections 1000 \
--rpc-threads 48 \
```
make sure you left adjust all the flags (vim likes to paste things tabbed over and in purple you need to delet the tabbing and bring everything to the left)
save / exit (:wq)
make executable
```
sudo chmod +x ~/start-validator.sh
sudo chown sol:sol start-validator.sh
```
create system service - sol.service (run on boot, auto-restart when sys fail) 
```
sudo vim /etc/systemd/system/sol.service
```
dump this into file:
```
[Unit]
Description=Solana Validator
After=network.target
Wants=systuner.service
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=on-failure
RestartSec=1
LimitNOFILE=1000000
LogRateLimitIntervalSec=0
User=sol
Environment=PATH=/bin:/usr/bin:/home/sol/.local/share/solana/install/active_release/bin
Environment=SOLANA_METRICS_CONFIG=host=https://metrics.solana.com:8086,db=mainnet-beta,u=mainnet-beta_write,p=password
ExecStart=/home/sol/start-validator.sh

[Install]
WantedBy=multi-user.target
```
save/exit (:wq)

make system tuner service - systuner.service
```
sudo vim /etc/systemd/system/systuner.service
```
dump this into file:
```
[Unit]
Description=Solana System Tuner
After=network.target
[Service]
Type=simple
Restart=on-failure
RestartSec=1
LogRateLimitIntervalSec=0
ExecStart=/home/sol/.local/share/solana/install/active_release/bin/solana-sys-tuner --user sol
[Install]
WantedBy=multi-user.target
```
reload the system services
```
sudo systemctl daemon-reload
```
log rotation for ~/log/solana-validator.log
```
sudo vim /etc/logrotate.d/solana
```
dump this into file:
```
/home/sol/log/solana-validator.log {
  su sol sol
  daily
  rotate 7
  missingok
  postrotate
    systemctl kill -s USR1 sol.service
  endscript
}
```
reset log rotate
```
sudo systemctl restart logrotate
```
start up and test

```
sudo systemctl enable --now systuner.service
sudo systemctl status systuner.service
sudo systemctl enable --now sol.service
sudo systemctl status sol.service
```
or this (prefer the above the option to use bash is just for debugging)
```
bash ~/start-validator.sh
```
tail log to make sure it's fethcing snapshot and working
```
sudo tail -f ~/log/solana-validator.log
```


# Experimental Performance Tuning
UPDATE: this tuning is now running across over half the machines and can be implemented

CPU to performance mode (careful with this)
```
sudo apt-get install cpufrequtils
echo 'GOVERNOR="performance"' | sudo tee /etc/default/cpufrequtils
sudo systemctl disable ondemand
```
modifications to sysctl.conf
```
sudo vim /etc/sysctl.conf
```
edit into bottom of file

```
# other tunings
# sysctl_optimisations:
vm.max_map_count=1000000
kernel.nmi_watchdog=0
# Minimal preemption granularity for CPU-bound tasks:
# (default: 1 msec#  (1 + ilog(ncpus)), units: nanoseconds)
kernel.sched_min_granularity_ns=10000000
# SCHED_OTHER wake-up granularity.
# (default: 1 msec#  (1 + ilog(ncpus)), units: nanoseconds)
kernel.sched_wakeup_granularity_ns=15000000 
vm.swappiness=30
kernel.hung_task_timeout_secs=600
# this means that virtual memory statistics is gathered less often but is a reasonable trade off for lower latency
vm.stat_interval=10
vm.dirty_ratio=40
vm.dirty_background_ratio=10
vm.dirty_expire_centisecs=36000
vm.dirty_writeback_centisecs=3000
vm.dirtytime_expire_seconds=43200
kernel.timer_migration=0
# A suggested value for pid_max is 1024 * <# of cpu cores/threads in system>
kernel.pid_max=49152
net.ipv4.tcp_fastopen=3
# From solana systuner
# Reference: https://medium.com/@CameronSparr/increase-os-udp-buffers-to-improve-performance-51d167bb1360
net.core.rmem_max=134217728
net.core.rmem_default=134217728
net.core.wmem_max=134217728
net.core.wmem_default=134217728
```

# revisiting setting to start-validator.sh (bottom section)

remove big table

```

--account-index-exclude-key kinXdEcpDQeHPEuQnqmUgtYykqKGVFq6CeVX5iAHJq6 \


```
```
sudo mkdir /mt/solana-accounts; sudo chown sol:sol /mt/solana-accounts

```