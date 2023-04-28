# Tor installation

This will allow for more privacy by hiding your server's IP address from the peers it's connecting to.
The Tor network has some latency, so it's nice to do this part after the blockchain has been synced with the network.
After the initial download the latency is not a huge problem. However, note that peer nodes will know
that there's a bitcoin node running in your IP address if you run it without Tor.

## 1. Install Tor

1. Install Tor with the following command:
```bash
sudo apt install tor
```

2. Add/uncomment these settings to `/etc/tor/torrc`
```
ControlPort 9051
CookieAuthentication 1
CookieAuthFileGroupReadable 1
```

3. Restart Tor to have the changes take effect:
```bash
sudo systemctl restart tor
```

## 2. Configure bitcoind to use Tor

We are going to have our bitcoin node run both through clearnet (regular Internet) and Tor network.
This way it can work as a bridge better connecting bitcoin nodes in these two networks.
If you want the full privacy benefits from Tor then you should configure your node to
only use Tor network for its connections.

Reading material:
- https://github.com/bitcoin/bitcoin/blob/master/doc/tor.md
- https://en.bitcoin.it/wiki/Setting_up_a_Tor_hidden_service
- https://rossbennetts.com/2015/04/13/running-bitcoind-via-tor/

Check the tor user group. It's the group that the `/run/tor/control.authcookie` file belongs to.
```bash
sudo ls -l /run/tor/control.authcookie
```

On Ubuntu the group is called `debian-tor`. We need to add the `bitcoin` user to this group.
```bash
sudo usermod -aG debian-tor bitcoin
```

Enable/uncomment the following settings in the file `/etc/bitcoin/bitcoin.conf`:
```dotenv
listenonion=1
debug=tor
onion=127.0.0.1:9050
```

Restart bitcoind to have the changes take effect:
```bash
sudo systemctl restart bitcoind
```
