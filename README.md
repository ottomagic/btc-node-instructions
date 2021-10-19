# Running a bitcoin node

These instructions are made for an Ubuntu 20.04 server.

Reading material: https://en.bitcoinwiki.org/wiki/Running_bitcoind

---

## Step 1: Get a hosted VPS with plenty of disk space

For example: https://contabo.com/en/storage-vps/vps-700/?image=ubuntu.267&qty=1

### Additional security steps if you choose Contabo

Once you have access to the control panel do the following things:
  - Disable VNC access to the server
  - Change the initial login password for the control panel
  - Enable 2FA for the control panel

---

## Step 2: Basic server setup

### Create admin user

1. Login to the server as root
2. Create `admin` user and add it to the `sudo` group:
```
adduser admin
usermod -aG sudo admin
```
3. Now logout the `root` user session

From now on any administrative tasks will be done with the `admin` user account using the `sudo` command.
For security reasons we will disable direct login with the `root` account later in these instructions.

### Basic server security configuration

1. Open a new ssh session to the server as the `admin` user.
2. Make sure the software is up-to-date:
```
sudo apt update
sudo apt upgrade
```
3. Add firewall rules. We will allow incoming traffic only for SSH and Bitcoin.
```
sudo ufw allow ssh
sudo ufw allow 8333
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw enable
```

### Enable automatic security updates

1. Install `unattended-upgrades` package:
```
sudo apt install unattended-upgrades
```
2. Add the following lines in `/etc/apt/apt.conf.d/20auto-upgrades`:
```
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Download-Upgradeable-Packages "1";
APT::Periodic::AutocleanInterval "7";
APT::Periodic::Unattended-Upgrade "1";
```
Further reference:
https://ubuntu.com/server/docs/package-management (header "Automatic Updates")

### Add key based SSH authentication for the `admin` user

1. To generate a key pair for login, run the following command on the **client machine** (your own machine):
```
ssh-keygen
```
You should now have the key pair in `~/.ssh/id_rsa` and `~/.ssh/id_rsa.pub`.

2. Copy the public key to the server (of course replacing `SERVER_IP_HERE` with the correct IP address):
```
ssh-copy-id admin@SERVER_IP_HERE
```
3. **TEST THAT YOU CAN NOW SSH IN WITH `admin` USER WITHOUT A PASSWORD!**

### Disable password authentication for SSH

Key authentication is more secure than password authentication.
We will only allow login to the server with the key we just created in the previous step
and disable password authentication altogether.

1. Edit the file `/etc/ssh/sshd_config` and make sure `PasswordAuthentication` has the value `no` and
   the line is NOT commented out.
2. Restart the ssh service for the changes to take effect:
```
sudo systemctl restart ssh
```
3. Launch a new terminal and see if you can still ssh in as the `root` user.
   It should not be possible because we didn't define an authentication key for root.

### Disable root login

We want to disable root login altogether for security reasons.
All administration should be done via the `admin` user and the `sudo` command.
To disable the root account password, use the following command:
```
sudo passwd -l root
```

### Restrict SSH access to only the `admin` account.

1. Create a group called `sshlogin`:
```
sudo addgroup sshlogin
```
2. Add the user `admin` to the group `sshlogin`:
```
sudo usermod -aG sshlogin admin
```
3. Add the group name as the value associated with the `AllowGroups` variable located in the file
   `/etc/ssh/sshd_config`.
```
AllowGroups sshlogin
```
4. Also disable root SSH login in the same file:
```
PermitRootLogin no
```
5. After saving changes to the `sshd_config` file restart the SSH service:
```
sudo systemctl restart ssh
```
6. **DO NOT CLOSE YOUR EXISTING SSH SESSION BEFORE YOU HAVE TRIED AND SUCCEEDED OPENING A SEPARATE
   PARALLEL SSH SESSION WITH THE `admin` USER AFTER APPLYING THE CHANGES.**
If you close the existing session and aren't able to open a new session (because of some mistake)
you are effectively locked out from the server and need to start again from a fresh OS install.

---

## Step 3: Bitcoin daemon installation

### Setup a dedicated user account

1. Create `bitcoin` user for running the daemon on:
```
sudo adduser bitcoin
```
2. Add `admin` user to `bitcoin` group
```
sudo usermod -aG bitcoin admin
```
3. Limit home folder permissions
```
sudo chmod 700 /home/admin
sudo chmod 750 /home/bitcoin
```

### Option 1: Install bitcoind by downloading and verifying binaries

Official instructions: https://bitcoin.org/en/full-node#linux-instructions

1. Download & verify based on instructions.
2. Extract the files from the downloaded package to `/home/admin/`.
3. Install bitcoind binaries. Assuming bitcoin core version 0.21.1.
```
cd /home/admin
sudo install -m 0755 -o root -g root -t /usr/local/bin bitcoin-0.21.1/bin/*
```
4. Configure man pages
```
sudo cp -r bitcoin-0.21.1/share/man/man1 /usr/local/share/man/
mandb
```

### Option 2: Install bitcoind by building from source

Documentation: https://github.com/bitcoin/bitcoin/blob/master/doc/build-unix.md

1. Clone the git repository
```
cd /home/admin
git clone https://github.com/bitcoin/bitcoin.git
```
2. Checkout the correct version tag. Replace `v22.0` with the desired tag.
```
cd bitcoin
git checkout tags/v22.0 -b tags/v22.0
```
3. Install Ubuntu build requirements
```
sudo apt-get install build-essential libtool autotools-dev automake pkg-config bsdmainutils python3
```
4. Install bitcoind dependencies
```
sudo apt-get install libevent-dev libboost-dev libboost-system-dev libboost-filesystem-dev libboost-test-dev
```
5. Build bitcoind
```
./autogen.sh
./configure --disable-wallet --without-gui
make
```
5. Install the program
```
sudo make install
```

### Configure bitcoind

1. Create a directory for the bitcoind configuration file:
```
sudo mkdir /etc/bitcoin
sudo chown bitcoin:bitcoin /etc/bitcoin
sudo chmod 710 /etc/bitcoin
```
2. Copy `bitcoin.conf` (under `config` folder of this repo) to the path `/etc/bitcoin/bitcoin.conf`
3. Make sure the file permissions are correct:
```
sudo chown bitcoin:bitcoin /etc/bitcoin/bitcoin.conf
sudo chmod 400 /etc/bitcoin/bitcoin.conf
```

### Setup bitcoind as a system service

Current Ubuntu releases use `systemd` for managing services.

1. Copy `bitcoind.service` (under `systemd` folder of this repo) to the path `/lib/systemd/system/bitcoind.service`
2. Make sure the file permissions are correct:
```
sudo chown root:root /lib/systemd/system/bitcoind.service
sudo chmod 644 /lib/systemd/system/bitcoind.service
```
3. Run the following command so `systemd` loads the new config:
```
sudo systemctl daemon-reload
```
4. To test that everything works, run:
```
sudo systemctl start bitcoind
```
5. After starting you can see the status by running:
```
sudo systemctl status bitcoind
```
6. You can have a look at the logs to see what `bitcoind` is up to:
```
sudo tail -n 50 /var/lib/bitcoind/debug.log
```
7. To enable for system startup, run:
```
sudo systemctl enable bitcoind
```
Further reference: https://github.com/bitcoin/bitcoin/blob/master/doc/init.md

### Configure log file rotation

It's good to configure rotation for the `debug.log` file so the file size doesn't grow indefinitely.

1. Copy the file `bitcoind-debug` (under `logrotate` folder of this repo) to the path `/etc/logrotate.d/bitcoind-debug`
2. Make sure the file permissions are correct:
```
sudo chown root:root /etc/logrotate.d/bitcoind-debug
sudo chmod 644 /etc/logrotate.d/bitcoind-debug
```
3. Restart the `logrotate` service with
```
sudo systemctl restart logrotate
```
4. You can check the status for errors with
```
sudo systemctl status logrotate
```

### Wait until the blockchain has been fully downloaded

This can take a few days.
You can follow the progress in the file `/var/lib/bitcoind/debug.log`.

### Tips & tricks

You can query your bitcoin node from the command line using `bitcoin-cli`.

For this use case it's best to assume the role of the `bitcoin` user. Type the following command:
```
su bitcoin
```
It will make you act as the `bitcoin` user. When you want to return to the `admin` user, just type the command `exit`.

The following command will give you an overview of the current blockchain information:
```
bitcoin-cli -datadir=/var/lib/bitcoind getblockchaininfo
```
Another useful command is `getconnectioncount` to see if the node is connecting to other nodes:
```
bitcoin-cli -datadir=/var/lib/bitcoind getconnectioncount
```
A full documentation of the RPC commands can be found here: https://developer.bitcoin.org/reference/rpc/

If you don't want to always add the `-datadir` argument there are two options:
1. Create a bash alias that includes the argument
2. Create the following symlink:
```
cd
ln -s /var/lib/bitcoind .bitcoin
```
If you are like me and don't remember the config and data directories by heart you can make the
following symbolic links to the bitcoin user's home directory so that you find the relevant
directories easily:
```
cd
ln -s /etc/bitcoin config
ln -s /var/lib/bitcoind data
```

---

## Step 4: (Optional) Install Tor

This will allow for more privacy by hiding your server's IP address from the peers it's connecting to.
The Tor network has some latency, so it's nice to do this part after the blockchain has been synced with the network.
After the initial download the latency is not a huge problem. However, note that peer nodes will know
that there's a bitcoin node running in your IP address if you run it without Tor.

### Install Tor

1. Install Tor with the following command:
```
sudo apt install tor
```
2. Add/uncomment these settings to `/etc/tor/torrc`
```
ControlPort 9051
CookieAuthentication 1
CookieAuthFileGroupReadable 1
```
3. Restart Tor to have the changes take effect:
```
sudo systemctl restart tor
```

### Configure bitcoind to use Tor

We are going to have our bitcoin node run both through clearnet (regular Internet) and Tor network.
This way it can work as a bridge better connecting bitcoin nodes in these two networks.
If you want the full privacy benefits from Tor then you should configure your node to
only use Tor network for its connections.

Reading material:
- https://github.com/bitcoin/bitcoin/blob/master/doc/tor.md
- https://en.bitcoin.it/wiki/Setting_up_a_Tor_hidden_service
- https://rossbennetts.com/2015/04/13/running-bitcoind-via-tor/

Check the tor user group. It's the group that the `/run/tor/control.authcookie` file belongs to.
```
sudo ls -l /run/tor/control.authcookie
```
On Ubuntu the group is called `debian-tor`. We need to add the `bitcoin` user to this group.
```
sudo usermod -aG debian-tor bitcoin
```
Uncomment (remove preceding `#`) the following two lines in the file `/etc/bitcoin/bitcoin.conf`:
```
debug=tor
proxy=127.0.0.1:9050
```
Restart bitcoind to have the changes take effect:
```
sudo systemctl restart bitcoind
```

---

## Step 5: (Optional) Install ElectrumX

ElectrumX is a good option if you want to run an Electrum server of your own.

Source code: https://github.com/spesmilo/electrumx/

Documentation: https://electrumx-spesmilo.readthedocs.io/en/latest/

### Setup a dedicated user account

1. Add a user account for ElectrumX:
```
sudo adduser electrumx
```
2. Add `admin` user to `electrumx` group
```
sudo usermod -aG electrumx admin
```
3. Limit home folder permissions
```
sudo chmod 750 /home/electrumx
```

### Install dependencies

1. Install git:
```
sudo apt update
sudo apt install git
```
2. Python3.8 is already installed by default on Ubuntu 20.04. Python version >= 3.7 is required for ElectrumX.
3. Install pip (Package installer for Python)
```
sudo apt install python3-pip
```
4. Install RocksDB
```
sudo apt install -y librocksdb-dev liblz4-dev build-essential libsnappy-dev zlib1g-dev libbz2-dev libgflags-dev
```

### Install ElectrumX

1. Take the role of user `electrumx` and go to its home directory:
```
su electrumx
cd
```
2. Clone the ElectrumX repository:
```
git clone https://github.com/spesmilo/electrumx.git
```
3. Checkout to the desired version tag:
```
cd electrumx
git checkout tags/1.16.0 -b tags/1.16.0
```
4. We don't need to install plyvel (LevelDB) because we are using RocksDB. Remove the dependency from the `setup.py` file.
```
sed -i "s/'plyvel',//" setup.py
```
5. We need `sudo` at the next step so get back to acting as the `admin` user:
```
exit
```
6. Run the following command in the `/home/electrumx/electrumx` directory to install ElectrumX:
```
sudo python3 -m pip install .[rocksdb,ujson]
```

### Prepare folders and config

1. Create a database directory for ElectrumX:
```
sudo mkdir -p /var/lib/electrumx/db
sudo chmod 750 -R /var/lib/electrumx
sudo chown -R electrumx:electrumx /var/lib/electrumx
```
2. Create a config directory for ElectrumX:
```
sudo mkdir /etc/electrumx
sudo chmod 750 /etc/electrumx
sudo chown electrumx:electrumx /etc/electrumx
```
3. Copy `electrumx.conf` (under `config` folder of this repo) to the path `/etc/electrumx/electrumx.conf`
4. Make sure the file permissions are good:
```
chmod 660 /etc/electrumx/electrumx.conf
sudo chown electrumx:electrumx /etc/electrumx
```
5. Depending on whether you want this to be a private ElectrumX instance (just in your own use) or open for public
you can uncomment one of these sets of lines in the `/etc/electrumx/electrumx.conf` file.

**Public server configuration**
```
PEER_DISCOVERY=on
PEER_ANNOUNCE=on
REPORT_SERVICES=ssl://:50002
```
**Private server configuration**
```
PEER_DISCOVERY=self
PEER_ANNOUNCE=
REPORT_SERVICES=
```

### Create RPC auth for ElectrumX to access bitcoind

1. Switch to the bitcoin user and go to its home directory:
```
su bitcoin
cd
```
2. Run the following command in the home directory to clone the bitcoin repository.
   We are going to utilize a script from the repository.
```
git clone https://github.com/bitcoin/bitcoin.git
```
3. Go into the repository directory and checkout the latest version tag:
```
cd bitcoin
git checkout tags/v0.21.1 -b tags/v0.21.1
```
4. Run the python script `share/rpcauth/rpcauth.py` to create RPC auth credentials for ElectrumX:
```
cd share/rpcauth/
python3 rpcauth.py electrumx
```
After running the script you need `sudo` privileges again so run the following command to return acting as `admin` user:
```
exit
```
The `rpcauth.py` script provided two outputs:
- **rpcauth**: Add the provided `rpcauth=...` line to `/etc/bitcoin/bitcoin.conf`
- **password**: Edit `/etc/electrumx/electrumx.conf` and replace `REPLACE_THIS_WITH_GENERATED_PASSWORD` with the provided password on the following line:
```
DAEMON_URL=http://electrumx:REPLACE_THIS_WITH_GENERATED_PASSWORD@127.0.0.1:8332/
```
5. To take the changes in `bitcoin.conf` into effect:
```
sudo systemctl restart bitcoind
```

### Create a self-signed SSL certificate

1. Assume the role of `electrumx` user:
```
su electrumx
```
2. Go to the directory where we keep ElectrumX config:
```
cd /etc/electrumx
```
3. Create a key and sign a certificate with the key. You can go with default values and leave the password empty.
```
openssl genrsa -out server.key 2048
openssl req -new -key server.key -out server.csr
openssl x509 -req -days 1825 -in server.csr -signkey server.key -out server.crt
```
4. The SSL certificate files are sensitive. Make sure the permissions are very strict:
```
chmod 400 /etc/electrumx/server.*
```
5. Return to the `admin` user:
```
exit
```

### Configure firewall

Open the port `50002` on the firewall:
```
sudo ufw allow 50002
```

### Setup ElectrumX as a systemd service

1. Copy `electrumx.service` (under `systemd` folder of this repo) to the path `/lib/systemd/system/electrumx.service`.
2. Make sure the file permissions are correct:
```
sudo chown root:root /lib/systemd/system/electrumx.service
sudo chmod 644 /lib/systemd/system/electrumx.service
```
3. Run the following command so that `systemd` loads the new config:
```
sudo systemctl daemon-reload
```
4. To test that it works, run:
```
sudo systemctl start electrumx
```
5. After starting you can see the status by running:
```
sudo systemctl status electrumx
```
6. You can use `journalctl` to check the log output:
```
sudo journalctl -u electrumx -f
```
7. To enable for system startup, run:
```
sudo systemctl enable electrumx
```

### Wait until ElectrumX indexes the blockchain

ElectrumX will index the blockchain using data from the local `bitcoind` service. This can take a few days.
You can follow the progress through logs using the `journalctl` tool as shown above.
ElectrumX will not serve any requests from outside until the indexing is finished.

When indexing is finished you can comment out (by adding a hash `#` in front) the following line in `/etc/electrumx/electrumx.conf` and just go with
the default value:
```
#CACHE_MB=2000
```
After that restart the service:
```
sudo systemctl restart electrumx
```

### Further reading material:

- https://freedomnode.com/blog/how-to-install-an-electrum-server-using-full-bitcoin-node-and-electrumx/
- https://junsun.net/wordpress/2021/01/setup-bitcoin-electrum-lightning-network-on-raspberry-pi-4/
- https://github.com/lukechilds/docker-electrumx
- https://github.com/bauerj/electrumx-installer/tree/master/distributions
