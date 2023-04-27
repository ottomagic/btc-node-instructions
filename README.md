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
