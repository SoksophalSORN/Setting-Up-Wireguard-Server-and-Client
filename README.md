## Scenario

You have one server with a **public IP** (EC2) and another with a **private IP** (RDS).

This is a common setup in AWS:

- **EC2** is used as a bastion host with a public IP.
    
- **RDS** is kept private in the VPC, reducing attack surface.
    
- AWS only charges for Elastic IPs, not regular (AWS's dynamically allocated) public IPs on EC2.

> Having a publicly accessible database is a critical vulnerability. Attackers can brute-force their way in. Using a VPN restricts access to you (and selected peers), while keeping traffic secure via SSH, WireGuard, and other encrypted protocols.

Since EC2 and RDS are in the same VPC (or peered subnets), EC2 can reach RDS over **private IP**. By placing a VPN on EC2, you can securely reach RDS through EC2.

We’ll create a VPN with subnet **10.10.0.0/24**, which connects into your AWS VPC subnet **172.16.0.0/24**, where EC2 and RDS live.

---

## Prerequisites

- You can SSH from your local machine into EC2.
    
- Basic understanding of asymmetric encryption.
    
- Both client and EC2 run Linux.
    
- WireGuard installed on EC2 (`wireguard-tools` + required packages).
    
- RDS is reachable from EC2 over its private IP.
    
- An EC2 instance with a **public IP address**.

---

## Part 1 — Configuring the VPN

### 1. Key Pair Generation

WireGuard uses public/private key pairs, similar to SSH.

On both client and EC2, generate keys:

```
wg genkey | tee privatekey | wg pubkey > publickey
```

- `wg genkey` → creates a private key.
    
- `tee privatekey` → saves it to a file while displaying it.
    
- `wg pubkey > publickey` → generates the public key from the private key.

Recommended filenames:

- Client → `client_privatekey`, `client_publickey`
    
- Server → `server_privatekey`, `server_publickey`

### 2. Public Key Exchange

Two options:

- Use `scp` to transfer keys.
    
- Or (easier) just `cat` the key and copy/paste.

⚠️ **Never share private keys.**

### 3. Server Configuration

Go to `/etc/wireguard/` on EC2, and create `wg0.conf`:

```ini
[Interface]
PrivateKey = <server_privatekey>
Address = 10.10.0.1/24
ListenPort = 51820

[Peer]
PublicKey = <client_publickey>
AllowedIPs = 10.10.0.10/32
```

Notes:

- `AllowedIPs` on the server must be `/32` (per client). `/24` breaks when multiple clients connect.
    
- `ListenPort` defaults to `51820/UDP` but can be changed.

Start WireGuard:

```
sudo systemctl start wg-quick@wg0.service
```

Enable on boot:

```
sudo systemctl enable wg-quick@wg0.service
```

If EC2 has a firewall, allow **UDP 51820**.

### 4. Client Configuration

On your client, create `/etc/wireguard/wg0.conf`:

```ini
[Interface]
PrivateKey = <client_privatekey>
Address = 10.10.0.10/24

[Peer]
PublicKey = <server_publickey>
Endpoint = <EC2_public_IP>:51820
AllowedIPs = 10.10.0.0/24, 172.16.0.0/24
PersistentKeepAlive = 25
```
>If you configured without PersistentKeepAlive, and the server cannot ping the client despite that the client can ping the server, consider configure it with the PersistentKeepAlive attribute.

Start the client the same way:

```
sudo systemctl start wg-quick@wg0.service
```

#### With GUI client

- Add empty tunnel → paste private key.
    
- Set `Address` and `[Peer]` as above.

### 5. Connectivity Test

- From client:

```
ping 10.10.0.1
```

- From EC2:

```
ping 10.10.0.10
```

If successful, continue to **Part 2**.

---

## Part 2 — Configuring IP Forwarding

We need VPN clients (`10.10.0.0/24`) to reach AWS resources (`172.16.0.0/24`).

> ❓ Why not just give VPN clients IPs inside `172.16.0.0/24`?  
> Because AWS manages this subnet. Assigning VPN clients here risks **IP conflicts**. A dedicated VPN subnet avoids this.

### 1. Add Forwarding Rules

In `/etc/wireguard/wg0.conf`, under `[Interface]`, add:

```
PostUp   = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o <interface> -j ACCEPT; iptables -t nat -A POSTROUTING -o <interface> -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o <interface> -j ACCEPT; iptables -t nat -D POSTROUTING -o <interface> -j MASQUERADE
```

Replace `<interface>` with EC2’s VPC interface (e.g., `eth0`).  
Find it with:

```
ip a
```

Restart WireGuard:

```
sudo systemctl restart wg-quick@wg0.service
```

### 2. Enable IP Forwarding

#### Temporary (until reboot):

```
sudo sysctl -w net.ipv4.ip_forward=1
```

#### Permanent:

Edit `/etc/sysctl.conf`:

```ini
net.ipv4.ip_forward = 1
```

Apply:

```
sudo sysctl -p
```

### 3. Connectivity Test

Instead of `ping`, use `nmap`.

Scan RDS host:

```
nmap -sV -p <port> -Pn <RDS_IP>
```

If unsure of port:

```
nmap -sV -p- -Pn <RDS_IP>
```

⚠️ Full scans are slow and may be blocked by AWS Security Groups. Prefer scanning the known DB port (e.g., `3306` for MySQL).

If the output shows the port open and service detected → forwarding works.

---

## Troubleshooting

### 1. Keys Mismatch

Check live config:

```
sudo wg show
```

Verify that server and client public keys match what’s configured.

### 2. AllowedIPs Issues

- Server must list client with `/32`.
    
- Client must list VPN subnet (`10.10.0.0/24`).
    
- CIDR notations must match.
    
- Ports must match (`51820/UDP` default).
    

### 3. Firewall

Scenarios:

- Cloud firewall only (common).
    
- Cloud + OS firewall.
    

At minimum, allow **UDP 51820** inbound on EC2.  
Best practice: restrict source to your own IP range instead of `0.0.0.0/0`.

If also using OS firewall, add matching rules there.

---

> **Acknowledgment**  
> This guide uses [WireGuard®](https://www.wireguard.com/), an open-source VPN protocol and software developed by Jason A. Donenfeld and contributors, distributed under the GPLv2 license.
