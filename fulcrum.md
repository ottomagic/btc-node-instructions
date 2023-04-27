# Fulcrum installation

Fulcrum is a performant electrum server alternative.

Code repository: https://github.com/cculianu/Fulcrum

Performance: https://www.sparrowwallet.com/docs/server-performance.html

## 1. Setup a dedicated user account

Create `fulcrum` user for running the daemon on:
```bash
sudo adduser fulcrum
```
Add `admin` user to `fulcrum` group
```bash
sudo usermod -aG fulcrum admin
```
Limit home folder permissions
```bash
sudo chmod 750 /home/fulcrum
```

## 2. Clone the git repository

As `admin` user clone the git repository in `/home/admin` directory.
```
git clone https://github.com/cculianu/Fulcrum.git
cd Fulcrum
```

Checkout the correct version tag:
```
git checkout tags/v1.9.1 -b tags/v1.9.1
```


## 3. Install build dependencies

```
sudo apt update
sudo apt install -y libqt5core5a libqt5network5 qtbase5-dev qt5-qmake libbz2-dev zlib1g-dev
```

Install optional/recommended dependencies (zmq libraries):
```
sudo apt install libzmq3-dev
```


## 4. Build

To generate the Makefile:
```
qmake
```

Replace 6 here with the number of cores on your machine:
```
make -j6
```

After a successful build install the files to the correct places:
```
sudo make install
```

## 5. Create RPC auth for Fulcrum to access bitcoind

1. Switch to folder where the bitcoind repository is located:
```bash
cd /home/admin/bitcoin
```

2. Run the python script `share/rpcauth/rpcauth.py` to create RPC auth credentials for Fulcrum:
```bash
cd share/rpcauth/
python3 rpcauth.py fulcrum
```

3. The `rpcauth.py` script provided two outputs:
- **rpcauth**: Add the provided `rpcauth=...` line to `/etc/bitcoin/bitcoin.conf`
- **password**: Take note of the password. It will be added to `/home/fulcrum/fulcrum.conf` in a later step.

4. To take the changes in `bitcoin.conf` into effect run this command:
```bash
sudo systemctl restart bitcoind
```

## 6. Create a self-signed SSL certificate

1. Assume the role of `fulcrum` user:
```bash
su fulcrum
```

2. Create a directory for SSL certs and fix permissions:
```bash
mkdir /home/fulcrum/ssl-certs
chmod 700 /home/fulcrum/ssl-certs
```

3. Create a key and sign a certificate with the key. You can go with default values and leave the password empty.
```bash
cd /home/fulcrum/ssl-certs
openssl genrsa -out server.key 2048
openssl req -new -key server.key -out server.csr
openssl x509 -req -days 1825 -in server.csr -signkey server.key -out server.crt
```

4. The SSL certificate files are sensitive. Make sure the permissions are very strict:
```bash
chmod 400 /home/fulcrum/ssl-certs/server.*
```

5. Return to the `admin` user:
```bash
exit
```


## 7. Configure fulcrum

1. Create a data directory and set the correct owner:
```bash
sudo mkdir -m 750 /home/fulcrum/database
sudo chown fulcrum:fulcrum /home/fulcrum/database
```

2. Copy the example config file as a template:
```bash
sudo cp /usr/local/share/doc/Fulcrum/fulcrum-example-config.conf /home/fulcrum/fulcrum.conf
```

Make sure the file permissions are correct:
```bash
sudo chown fulcrum:fulcrum /home/fulcrum/fulcrum.conf
sudo chmod 400 /home/fulcrum/fulcrum.conf
```

Edit the following values in the config file:
```
datadir = /home/fulcrum/database
rpcuser = fulcrum
rpcpassword = REPLACE_THIS_WITH_GENERATED_PASSWORD
tcp = disabled
ssl = 0.0.0.0:50002
cert = /home/fulcrum/ssl-certs/server.crt
key = /home/fulcrum/ssl-certs/server.key
tls-disallow-deprecated = true
peering = false
pidfile = /run/fulcrum/fulcrum.pid
```

## 8. Configure firewall

Open the port `50002` on the firewall:
```bash
sudo ufw allow 50002
```


## 9. Setup fulcrum as a system service

Current Ubuntu releases use `systemd` for managing services.

1. Copy `fulcrum.service` (under `systemd` folder of this repo) to the path `/lib/systemd/system/fulcrum.service`

2. Make sure the file permissions are correct:
```bash
sudo chown root:root /lib/systemd/system/fulcrum.service
sudo chmod 644 /lib/systemd/system/fulcrum.service
```

3. Run the following command so `systemd` loads the new config:
```bash
sudo systemctl daemon-reload
```

4. To test that everything works, run:
```bash
sudo systemctl start fulcrum
```

5. After starting, you can see the status by running:
```bash
sudo systemctl status fulcrum
```

6. You can use `journalctl` to check the log output:
```bash
sudo journalctl -u fulcrum -f
```

7. To enable for system startup, run:
```bash
sudo systemctl enable fulcrum
```

## 10. Wait for synchronization to finish

Once sync is finished remove the following parameter from the `/lib/systemd/system/fulcrum.service` service config file:
```
--fast-sync 10000 \
```

Reload the new config:
```bash
sudo systemctl daemon-reload
```

Restart the service:
```bash
sudo systemctl restart fulcrum
```
