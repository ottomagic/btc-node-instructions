# ElectrumX installation

ElectrumX is a good option if you want to run an Electrum server of your own.

Source code: https://github.com/spesmilo/electrumx/

Documentation: https://electrumx-spesmilo.readthedocs.io/en/latest/

## 1. Setup a dedicated user account

1. Add a user account for ElectrumX:
```bash
sudo adduser electrumx
```

2. Add `admin` user to `electrumx` group
```bash
sudo usermod -aG electrumx admin
```

3. Limit home folder permissions
```bash
sudo chmod 750 /home/electrumx
```

## 2. Install dependencies

1. Install git if it's not already installed:
```bash
sudo apt update
sudo apt install git
```

2. Python3.8 is already installed by default on Ubuntu 20.04. Python version >= 3.7 is required for ElectrumX.
3. Install pip (Package installer for Python)
```bash
sudo apt install python3-pip
```

4. Install RocksDB
```bash
sudo apt install -y librocksdb-dev liblz4-dev build-essential libsnappy-dev zlib1g-dev libbz2-dev libgflags-dev
```

## 3. Install ElectrumX

1. Take the role of user `electrumx` and go to its home directory:
```bash
su electrumx
cd
```

2. Clone the ElectrumX repository:
```bash
git clone https://github.com/spesmilo/electrumx.git
```

3. Checkout to the desired version tag:
```bash
cd electrumx
git checkout tags/1.16.0 -b tags/1.16.0
```

4. We don't need to install plyvel (LevelDB) because we are using RocksDB. Remove the dependency from the `setup.py` file.
```bash
sed -i "s/'plyvel',//" setup.py
```

5. We need `sudo` at the next step so get back to acting as the `admin` user:
```bash
exit
```

6. Run the following command in the `/home/electrumx/electrumx` directory to install ElectrumX:
```bash
sudo python3 -m pip install .[rocksdb,ujson]
```

## 4. Prepare folders and config

1. Create a database directory for ElectrumX:
```bash
sudo mkdir -p /var/lib/electrumx/db
sudo chmod 750 -R /var/lib/electrumx
sudo chown -R electrumx:electrumx /var/lib/electrumx
```

2. Create a config directory for ElectrumX:
```bash
sudo mkdir /etc/electrumx
sudo chmod 750 /etc/electrumx
sudo chown electrumx:electrumx /etc/electrumx
```

3. Copy `electrumx.conf` (under `config` folder of this repo) to the path `/etc/electrumx/electrumx.conf`
4. Make sure the file permissions are good:
```bash
chmod 660 /etc/electrumx/electrumx.conf
sudo chown electrumx:electrumx /etc/electrumx
```

5. Depending on whether you want this to be a private ElectrumX instance (just in your own use) or open for public
   you can uncomment one of these sets of lines in the `/etc/electrumx/electrumx.conf` file.

**Public server configuration**
```dotenv
PEER_DISCOVERY=on
PEER_ANNOUNCE=on
REPORT_SERVICES=ssl://:50002
```

**Private server configuration**
```dotenv
PEER_DISCOVERY=self
PEER_ANNOUNCE=
REPORT_SERVICES=
```

## 5. Create RPC auth for ElectrumX to access bitcoind

1. Switch to the bitcoin user and go to its home directory:
```bash
su bitcoin
cd
```

2. Run the following command in the home directory to clone the bitcoin repository.
   We are going to utilize a script from the repository.
```bash
git clone https://github.com/bitcoin/bitcoin.git
```

3. Go into the repository directory and checkout the latest version tag:
```bash
cd bitcoin
git checkout tags/v0.21.1 -b tags/v0.21.1
```

4. Run the python script `share/rpcauth/rpcauth.py` to create RPC auth credentials for ElectrumX:
```bash
cd share/rpcauth/
python3 rpcauth.py electrumx
```

5. After running the script you need `sudo` privileges again so run the following command to return acting as `admin` user:
```bash
exit
```

6. The `rpcauth.py` script provided two outputs:
- **rpcauth**: Add the provided `rpcauth=...` line to `/etc/bitcoin/bitcoin.conf`
- **password**: Edit `/etc/electrumx/electrumx.conf` and replace `REPLACE_THIS_WITH_GENERATED_PASSWORD` with the provided password on the following line:
```dotenv
DAEMON_URL=http://electrumx:REPLACE_THIS_WITH_GENERATED_PASSWORD@127.0.0.1:8332/
```

7. To take the changes in `bitcoin.conf` into effect run this command:
```bash
sudo systemctl restart bitcoind
```

## 6. Create a self-signed SSL certificate

1. Assume the role of `electrumx` user:
```bash
su electrumx
```

2. Go to the directory where we keep ElectrumX config:
```bash
cd /etc/electrumx
```

3. Create a key and sign a certificate with the key. You can go with default values and leave the password empty.
```bash
openssl genrsa -out server.key 2048
openssl req -new -key server.key -out server.csr
openssl x509 -req -days 1825 -in server.csr -signkey server.key -out server.crt
```

4. The SSL certificate files are sensitive. Make sure the permissions are very strict:
```bash
chmod 400 /etc/electrumx/server.*
```

5. Return to the `admin` user:
```bash
exit
```

## 7. Configure firewall

Open the port `50002` on the firewall:
```bash
sudo ufw allow 50002
```

## 8. Setup ElectrumX as a systemd service

1. Copy `electrumx.service` (under `systemd` folder of this repo) to the path `/lib/systemd/system/electrumx.service`.
2. Make sure the file permissions are correct:
```bash
sudo chown root:root /lib/systemd/system/electrumx.service
sudo chmod 644 /lib/systemd/system/electrumx.service
```

3. Run the following command so that `systemd` loads the new config:
```bash
sudo systemctl daemon-reload
```

4. To test that it works, run:
```bash
sudo systemctl start electrumx
```

5. After starting, you can see the status by running:
```bash
sudo systemctl status electrumx
```

6. You can use `journalctl` to check the log output:
```bash
sudo journalctl -u electrumx -f
```

7. To enable for system startup, run:
```bash
sudo systemctl enable electrumx
```

## 9. Wait until ElectrumX indexes the blockchain

ElectrumX will index the blockchain using data from the local `bitcoind` service. This can take a few days.
You can follow the progress through logs using the `journalctl` tool as shown above.
ElectrumX will not serve any requests from outside until the indexing is finished.

When indexing is finished you can comment out (by adding a hash `#` in front) the following line in `/etc/electrumx/electrumx.conf` and just go with
the default value:
```dotenv
#CACHE_MB=2000
```

After that restart the service:
```bash
sudo systemctl restart electrumx
```

## Compact history script

In case you run into the following error in the logs:
```
struct.error: 'H' format requires 0 <= number <= 65535
```

Use the following script as `electrumx` user to fix the issue:
``` bash
set -o allexport; source /etc/electrumx/electrumx.conf; set +o allexport; ./electrumx_compact_history
```

Source: https://github.com/kyuupichan/electrumx/issues/185


## Further reading material:

- https://freedomnode.com/blog/how-to-install-an-electrum-server-using-full-bitcoin-node-and-electrumx/
- https://junsun.net/wordpress/2021/01/setup-bitcoin-electrum-lightning-network-on-raspberry-pi-4/
- https://github.com/lukechilds/docker-electrumx
- https://github.com/bauerj/electrumx-installer/tree/master/distributions
