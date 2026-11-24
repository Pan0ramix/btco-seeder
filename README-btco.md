# BitcoinOil DNS seed

This is sipa's `bitcoin-seeder` set up for BitcoinOil (BTCO). It crawls the BitcoinOil network and answers DNS queries with a list of healthy nodes, so that new nodes can find peers.

Changes from upstream:

- Network magic, P2P ports (19712 mainnet, 29712 testnet) and bootstrap nodes of BitcoinOil.
- A node counts as good only if it is past block 157500 and has that block in its chain. Nodes left on 25.2.0, which stop at 157499, are never handed out.
- Merge-mined headers (AuxPoW) are accepted when checking that block.
- `x1000009` filter for nodes that offer MWEB data.

Tested on Debian 13 (Trixie) against BitcoinOil mainnet.

## Requirements

- A Debian 13 machine or VM with a public IPv4 address. IPv6 is not needed.
- UDP port 53 reachable from the internet. The seed only uses UDP.
- About 256 MB of RAM, very little disk and 10 to 50 MB of traffic a day.
- Two DNS records at your registrar, for example:

  ```
  seed.example.com      NS   ns-seed.example.com
  ns-seed.example.com   A    <public IPv4>
  ```

  With a dynamic IP, keep the `A` record current with dynamic DNS and give it a TTL of 300 seconds or less.

## Build

```bash
sudo apt update
sudo apt install -y build-essential libssl-dev libboost-dev git dnsutils
git clone <repository URL> btco-seeder
cd btco-seeder
make -j2
```

## Run as a service

The seed listens on port 5353 as an unprivileged user. The router forwards public UDP port 53 to it.

```bash
sudo useradd --system --home /var/lib/btco-seeder --create-home btco-seeder
sudo install -m 755 dnsseed /usr/local/bin/btco-dnsseed
```

Create `/etc/systemd/system/btco-seeder.service`:

```ini
[Unit]
Description=BitcoinOil DNS seed
After=network-online.target
Wants=network-online.target

[Service]
User=btco-seeder
WorkingDirectory=/var/lib/btco-seeder
ExecStart=/usr/local/bin/btco-dnsseed -h seed.example.com -n ns-seed.example.com -m hostmaster.example.com -p 5353 -t 16
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

`-m` is the contact address shown in the zone's SOA record, written with a dot in place of the `@`.

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now btco-seeder
```

On the router, forward **UDP** port 53 on the public IP to UDP port 5353 on this machine.

## Rate limit

The answers are small, so the seed is a poor tool for DNS amplification attacks, but a limit per source costs nothing:

```bash
sudo apt install -y nftables
sudo tee /etc/nftables.d/btco-seeder.nft >/dev/null <<'EOF'
table inet btco_seeder {
  set limited { type ipv4_addr; flags dynamic; timeout 1m; }
  chain input {
    type filter hook input priority 0;
    udp dport 5353 add @limited { ip saddr limit rate over 20/second burst 40 packets } drop
  }
}
EOF
sudo nft -f /etc/nftables.d/btco-seeder.nft
```

To keep the rule after a reboot, add `include "/etc/nftables.d/*.nft"` to `/etc/nftables.conf` and run `sudo systemctl enable nftables`.

## Check

On the machine, after about a minute:

```bash
dig +short @127.0.0.1 -p 5353 seed.example.com A
```

From anywhere else on the internet, once the router and the DNS records are in place:

```bash
dig +short seed.example.com A
```

Both should print a few IPv4 addresses of BitcoinOil nodes. `journalctl -u btco-seeder` shows the crawler status, and `/var/lib/btco-seeder/dnsseed.dump` lists every node it knows.
