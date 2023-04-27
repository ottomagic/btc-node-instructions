# Bitcoin daemon installation

## 1. Setup a dedicated user account

Create `bitcoin` user for running the daemon on:
```bash
sudo adduser bitcoin
```
Add `admin` user to `bitcoin` group
```bash
sudo usermod -aG bitcoin admin
```
Limit home folder permissions
```bash
sudo chmod 750 /home/bitcoin
```

## 2. Install bitcoind

### Option 1: Install bitcoind by building from source

Documentation: https://github.com/bitcoin/bitcoin/blob/master/doc/build-unix.md

Install git:
```bash
sudo apt install git
```

Clone the git repository
```bash
cd /home/admin
git clone https://github.com/bitcoin/bitcoin.git
```

Checkout the correct version tag. Replace `v24.0.1` with the desired tag.
```bash
cd bitcoin
git checkout tags/v24.0.1 -b tags/v24.0.1
```

Install Ubuntu build requirements
```bash
sudo apt-get install build-essential libtool autotools-dev automake pkg-config bsdmainutils python3
```

Install bitcoind dependencies
```bash
sudo apt-get install libevent-dev libboost-dev
```

Install ZMQ dependencies (provides ZMQ API):
```bash
sudo apt-get install libzmq3-dev
```

Build bitcoind without wallet and GUI
```bash
./autogen.sh
./configure --disable-wallet --without-gui
make
```

Install the program
```bash
sudo make install
```

### Option 2: Install bitcoind by downloading and verifying binaries

Official instructions: https://bitcoin.org/en/full-node#linux-instructions

1. Download & verify based on instructions.
2. Extract the files from the downloaded package to `/home/admin/`.
3. Install bitcoind binaries. Assuming bitcoin core version 0.21.1.
```bash
cd /home/admin
sudo install -m 0755 -o root -g root -t /usr/local/bin bitcoin-0.21.1/bin/*
```

4. Configure man pages
``` bash
sudo cp -r bitcoin-0.21.1/share/man/man1 /usr/local/share/man/
mandb
```

## 3. Configure bitcoind

Create a directory for the bitcoind configuration file and set the correct permissions:
```bash
sudo mkdir /etc/bitcoin
sudo chown bitcoin:bitcoin /etc/bitcoin
sudo chmod 750 /etc/bitcoin
```

Copy `bitcoin.conf` (under `config` folder of this repo) to the path `/etc/bitcoin/bitcoin.conf`

Make sure the file permissions are correct:
```bash
sudo chown bitcoin:bitcoin /etc/bitcoin/bitcoin.conf
sudo chmod 400 /etc/bitcoin/bitcoin.conf
```

## 4. Setup bitcoind as a system service

Current Ubuntu releases use `systemd` for managing services.

1. Copy `bitcoind.service` (under `systemd` folder of this repo) to the path `/lib/systemd/system/bitcoind.service`

2. Make sure the file permissions are correct:
```bash
sudo chown root:root /lib/systemd/system/bitcoind.service
sudo chmod 644 /lib/systemd/system/bitcoind.service
```

3. Run the following command so `systemd` loads the new config:
```bash
sudo systemctl daemon-reload
```

4. To test that everything works, run:
``` bash
sudo systemctl start bitcoind
```

5. After starting you can see the status by running:
```bash
sudo systemctl status bitcoind
```

6. You can have a look at the logs to see what `bitcoind` is up to:
```bash
sudo tail -n 50 /var/lib/bitcoind/debug.log
```

7. To enable for system startup, run:
```bash
sudo systemctl enable bitcoind
```

Further reference: https://github.com/bitcoin/bitcoin/blob/master/doc/init.md

## 5. Configure log file rotation

It's good to configure rotation for the `debug.log` file so the file size doesn't grow indefinitely.

1. Copy the file `bitcoind-debug` (under `logrotate` folder of this repo) to the path `/etc/logrotate.d/bitcoind-debug`
2. Make sure the file permissions are correct:
```bash
sudo chown root:root /etc/logrotate.d/bitcoind-debug
sudo chmod 644 /etc/logrotate.d/bitcoind-debug
```

3. Restart the `logrotate` service with
```bash
sudo systemctl restart logrotate
```

4. You can check the status for errors with
```bash
sudo systemctl status logrotate
```

## 6. Wait until the blockchain has been fully downloaded

This can take a few days.
You can follow the progress in the file `/var/lib/bitcoind/debug.log`.

## Tips & tricks

You can query your bitcoin node from the command line using `bitcoin-cli`.

For this use case it's best to assume the role of the `bitcoin` user. Type the following command:
```bash
su bitcoin
```
It will make you act as the `bitcoin` user. When you want to return to the `admin` user, just type the command `exit`.

The following command will give you an overview of the current blockchain information:
```bash
bitcoin-cli -datadir=/var/lib/bitcoind getblockchaininfo
```

Another useful command is `getconnectioncount` to see if the node is connecting to other nodes:
```bash
bitcoin-cli -datadir=/var/lib/bitcoind getconnectioncount
```
A full documentation of the RPC commands can be found here: https://developer.bitcoin.org/reference/rpc/

If you don't want to always add the `-datadir` argument there are two options:
1. Create a bash alias that includes the argument
2. Create the following symlink:
```bash
cd
ln -s /var/lib/bitcoind .bitcoin
```

If you are like me and don't remember the config and data directories by heart you can make the
following symbolic links to the bitcoin user's home directory so that you find the relevant
directories easily:
```bash
cd
ln -s /etc/bitcoin config
ln -s /var/lib/bitcoind data
```
