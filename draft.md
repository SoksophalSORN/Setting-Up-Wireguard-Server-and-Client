This is a detailed description of how I set up my own VPN server to access my servers.

## Scenario
---
You have a server with public IP and another database server with private IP. 

This is a common scenario of AWS EC2 and RDS service (free tier as in this case) where EC2 is used as a database bastion to avoid paying for public IP address fee when using RDS with public IP. Public IP on EC2 is not being charged unless Elastic IP service is being used.

Let’s call our server EC2 and our database server RDS.

Having a publicly accessible MySQL server is fatal vulnerability as hackers can brute force their way in, so you come up with a clever idea — VPN. With VPN, only you and several selected people can access the database, and you do not have to worry about hackers brute forcing the database and the EC2 as you use secure communication protocols such as SSH with Public Key Auth, WireGuard, etc. Moreover, since you decided that you want to access the RDS via the EC2, configuring VPN has become possible.

Your EC2 has a **network interface (eth0)** with both a private IP (in the VPC subnet) and a public IP (for internet access).

Since RDS is in the same VPC subnet (or a peered subnet), EC2 can reach RDS via its **private IP** as long as the **security groups and subnet routing** allow it.

You want to build a VPN with network 10.10.0.0/24 to connect to a virtual private network with subnet 172.16.0.0/24. Network 172.16.0.0/24 is where your EC2 and RDS are connected. ^a00796

## Prerequisite
---

- Client (your computer) has already been configured to be able to SSH into the EC2.
	
- Some knowledge of how asymmetric encryption works.
	
- Both client and EC2 are Linux system (I have no plan to configure this on Windows and MacOS).
	
- On server, WireGuard-tool is installed from the host’s package manager, and a WireGuard package with all the necessary tools to host a WireGuard VPN server.
	
- RDS is accessible via EC2
	
- An EC2 instance with a **public IP address**

## Part 1 — Configuring VPN
---
### 1. Key Pair Generation

Similar to SSH, WireGuard uses private-public key pair for authentication.

On both client and EC2, you need to generate their own public and private key pairs then exchange public keys with each other so that EC2 can encrypt its message before sending to the client, and the client is able to decrypt it, and vice versa.

So, you would issue this command to generate public-private key pair for WireGuard:

```bash
$ wg genkey | tee privatekey | wg pubkey > publickey
```

This command means:

- `wg genkey` — generate a private key and print it out on the screen
	
- `| tee privatekey` — take the output of the previous command (piping), save it in a file named `privatekey`, and then print it out again
	
- `| wg pubkey > publickey` — piping the last command output, use it to generate a WireGuard public key. Then, store (redirect) it inside a file named `publickey`.

You do this for both client and EC2. To avoid confusion, you would name the private-public key pair on the client as `client_privatekey` and `client_publickey`. You would name your files as `server_privatekey` and `server_publickey` as well on the EC2.

### 2. Public Key Sharing

There are 2 ways

- The hard way: use `scp` command to transfer public key back and forth between the client and EC2, or
	
- The ez way: `cat` (print out) the content of the public key files and copy it. This option is recommended as most of us are SHH-ing into the server during when we are trying to configure Wireguard VPN for the client and the EC2.

> [!NOTE]
> NEVER share your private keys.

### 3. Wireguard Server Configuration 

First, you need traverse to `/etc/wireguard/` with root privilege (putting `sudo` at the start of a command). `cd /etc/wireguard/` in short. If you do not have the folder, it is likely that you have not installed WireGuard yet.

Then, you create a file named `wg0.conf`. (I use Vim btw)

Your `wg0.conf` would need the following:

```WireGuard
[Interface]
PrivateKey = <content_of_server_privatekey>
Address = 10.10.0.1/24
ListenPort = 51820

[Peer]
PublicKey = <content_of_client_publickey>
AllowedIPs = 10.10.0.10/32
```

> [!NOTE]
> For `[Peer]`’s allowedIPs, it should be in `/32` subnet which means that traffic is allowed from that one specific IP address, not a subnet. I tried using `/24` at first. It works when you have only 1 peer/client, but the VPN stops working when you have 2 and more peers using `/24`.

`ListenPort` variable indicates which port WireGuard should be listening to. 51820 is the default port of WireGuard. Fill free to change it.

For the reason of why these IP addresses are chosen, please refer to [[WireGuard Server and Client Set up - draft#^a00796|this]].

Next, you would fire up WireGuard by using `systemctl`. You do so by issuing the following command:
```bash
$ systemctl start wg-quick.target
# systemctl start wg-quick@<wireguard_server_config_filename>.service
$ systemctl start wg-quick@wg0.service
```

^26d3dc

These commands ensure that WireGuard service is up and is using the configuration configured in `/etc/wireguard/wg0.conf`.

If you want the service to be persistent even after restart, replace `start` with `enable`.

> [!NOTE]
> If your EC2 has a firewall, enable port `51820/UDP` for the VPN connection. On the client, if you also have firewall, please make sure that 

### 4. Wireguard Client Configuration

#### Without UI

Similar to Wireguard server configuration, you need to traverse to `/etc/wireguard/` with root privilege. Then, you create file named `wg0.conf` with the following configuration:

```WireGuard
[Interface]
PrivateKey = <content_of_client_privatekey>
Address = 10.10.0.10/24

[Peer]
AllowedIPs = 10.10.0.0/24
PublicKey = <content_of_server_publickey>
Endpoint = <EC2_public_IP>:51820
# Uncomment the below if your client is always on. Measured in seconds
# PersistentKeepAlive = 25
```

The address under the interface must match the configured address inside the EC2, and the content of client private key must be able to be used to derive for the public key the same as the one used inside the EC2.

The value of `AllowedIPs` means that any packet with destination inside the subnet of `10.10.0.0` will be forwarded through the interface `10.10.0.10/24` to the server with the public IP address and port.

To start the WireGuard client, issue the exact [[WireGuard Server and Client Set up - draft#^26d3dc|same command]] as you did on the EC2.

#### With UI

1. Open your WireGuard client
	
2. Click on `Add Tunnel`
	
3. Click on `Add empty tunnel`
	
4. Change the value of `PrivateKey` to the content of `client_privatekey`
	
5. Copy, paste, and adjust the value of the below script:
	
```WireGuard
Address = 10.10.0.10/24

[Peer]
AllowedIPs = 10.10.0.0/24
PublicKey = <content_of_server_publickey>
Endpoint = <EC2_public_IP>:51820
# Uncomment the below if your client is always on. Measured in seconds
# PersistentKeepAlive = 25
```
	
6. Click `save` and done.

### 5. Connectivity Testing

From the WireGuard client, `ping` the WireGuard server. In this case, try:

```bash
ping 10.10.0.1
```

Then, on the EC2, `ping` the WireGuard client. In this case, try:

```bash
ping 10.10.0.10
```

If both can successfully `ping` each other, you are ready for the [[WireGuard Server and Client Set up - draft#Part 2 — Configuring IP Forwarding|next step]].

If you cannot, however, please refer to [[WireGuard Server and Client Set up - draft#Troubleshooting|this section]] immediately before moving on to the next step.

## Part 2 — Configuring IP Forwarding
---
This part revolves around how to make the client able to communicate with with another subnet. In this case, we are trying to make network `10.10.0.0/24` able to communicate with network `172.16.0.0/24`.

> [!NOTE]
> You might wonder why not put VPN clients directly into `172.16.0.0/24`.
> 
> The reason is that this subnet is already managed by AWS for your VPC. If you assign VPN clients addresses inside it, you risk **IP conflicts** with EC2, RDS, or other resources.
> 
> By using a separate VPN subnet (`10.10.0.0/24`) and then **forwarding traffic**, you keep things isolated and safe.

### 1. Configuring IP Forwarding Rule

On the EC2, you add the following 2 lines to the existing `wg0.conf`, just above `[Peer]` and below the `[Interface]` section:

```
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o <interface_name> -j ACCEPT; iptables -t nat -A POSTROUTING -o <interface_name> -j MASQUERADE

PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o <interface_name> -j ACCEPT; iptables -t nat -D POSTROUTING -o <interface_name> -j MASQUERADE
```

Explanation

- **`PostUp`**  
    Commands executed after the WireGuard interface (`wg0`) comes up.
	
	- `iptables -A FORWARD -i %i -j ACCEPT`
	    - `%i` = the interface name (`wg0` automatically).
			
		- Accepts all traffic coming _in_ from the VPN interface.
		
	- `iptables -A FORWARD -o <interface_name> -j ACCEPT`
		- Accepts all traffic going _out_ via the physical NIC (`<interface_name>`, replace with your real WAN/LAN interface).
		
	- `iptables -t nat -A POSTROUTING -o <interface_name> -j MASQUERADE`
		- NAT masquerade: rewrites source IPs of VPN clients to appear as the server’s `<interface_name>` IP.
			
		- This is what allows VPN clients to access the internet (or any network behind `<interface_name>`) without needing special routes.
	    
- **`PostDown`**  
    Commands executed when the WireGuard interface goes down, to clean up the rules.
    They’re the same rules but with `-D` instead of `-A`, so they **remove** those rules when the VPN interface shuts down.

To get the interface name, use the following command:

```bash
ip a
```

Choose the name of the interface which has the IP address inside the same network as your RDS. In this case, it should be inside `172.16.0.0/24` subnet.

After saving the configuration, restart WireGuard to load the newly added configuration by:

```bash
$ systemctl restart wg-quick@wg0.service
```

Replace `wg0` with your actual WireGuard config filename.

### 2. Enable IP Forwarding

Without enabling this, even if you configured the rules, EC2 won’t be able to connect you to the `172.16.0.0/24` network.

To do so, you have 2 options:

- Enable IP forwarding for only this session
	
- Enable IP forwarding for all sessions

In this context, a session is a duration between the last boot/startup and the next shutdown/restart.

#### Enable IP forwarding for only this session

With root privilege, you issue the following command:

```bash
$ sysctl -w net.ipv4.ip_forward=1
```

This command allows the system to forward IP based on the written rules. However, it lasts only for this session, and IP forwarding will be disabled in the next session.

#### Enable IP forwarding for all sessions

With root privilege, traverse to `/etc/`, then create a file called `sysctl.conf`, and add the following configuration to the file:

```sysctl
net.ipv4.ip_forward = 1
```

To make `sysctl` uses the newly added configuration, use the following command (with `sudo`)

```bash
$ sysctl -p
```

This command allows `sysctl` reads configuration from `/etc/sysctl.conf` (by default). After restart, the configuration will still persist since `sysctl` will read `sysctl.conf` on start up.

### Connectivity Testing

We won’t be using `ping` to test. However, we will use `nmap`.

For your information, `nmap` is a popular network tool for gathering information about a host. 

Assuming you have the IP address of your RDS, replace it with `<host_IP>` in the below command. Also, replace `<port>` with the actual port that your database is listening to:

```bash
nmap -sV -p <port> -Pn <host_IP>
```

However, if you don’t know what port number that your database is listening to, you might need the below command:

```bash
nmap -sV -p- -Pn <host_IP>
```

A full 65,536 port scan can be **slow** and may even be blocked by AWS Security Groups. In practice, it may take much longer or return filtered results.

If you know the database port (e.g., `3306` for MySQL), scan that specific port instead for faster and more reliable results.

If the output is something like this:

```bash
Nmap scan report for <host_IP>
Host is up (0.0065s latency).

PORT     STATE SERVICE VERSION
<port>/tcp open  mysql   MySQL <version>

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 0.65 seconds
```

Your IP forwarding is successful. Your journey ends here.

If you are not, however, please refer to [[WireGuard Server and Client Set up - draft#Troubleshooting|this section]] immediately.

## Troubleshooting
---
### Cannot `ping` the WireGuard server and vice versa

#### 1. Check your public keys

Instead checking the configuration file first, use the below command with root privilege:

```bash
$ wg show
```

Run this command on both WireGuard server and client.

On server, it shows something like this:

```Server
interface: wg0
  public key: <server_publickey>
  private key: (hidden)
  listening port: 51820

peer: <client_publickey>
  endpoint: <client_public_IP>:51784
  allowed ips: 10.10.0.10/32
```

On client, it shows something like this:

```Client
interface: wg0
  public key: <client_publickey>
  private key: (hidden)
  listening port: <random_high_port>

peer: <server_publickey>
  endpoint: <server_public_IP>:51820
  allowed ips: 10.10.0.0/24, 172.16.0.0/24
```

Check for 3 things:

- `server_public_key`
	The value must be the same on both client and server.
	
- `client_public_key`
	The value must be the same on both client and server.
	

If you see differences, you should reconfigure the public and private key across the server and client.

If there is no problem with this, or you have reconfigure the keys, and the problem still persists, go the next step.

#### 2. Check your `AllowedIPs`

In your WireGuard server config file, you should see something like this:

```Server
[Interface]
PrivateKey = <server_private_key>
Address = 10.10.0.1/24
ListenPort = 51820

[Peer]
PublicKey = <client_public_key>
AllowedIPs = 10.10.0.10/32
```

In your WireGuard client config file, you should see something like this

```Client
[Interface]
PrivateKey = <client_private_key>
Address = 10.10.0.10/24

[Peer]
AllowedIPs = 10.10.0.0/24
PublicKey = <server_public_key>
Endpoint = <server_public_IP>:51820
```

Make sure that:

- CIDR notation (`/24, /32, ...`) inside `[Interface]` block is consistent across both server and client.
	
- `AllowedIPs` inside the server configuration must be `/32`.
	
- `AllowedIPs` inside the client configuration file must contain the network address of the VPN network (`10.10.0.0/24` in this context)
	
- `<server_public_IP>` inside client configuration file must be the correct IP of your WireGuard server.
	
- Port number in the server config file and the client config file must be consistent.

If you have checked all of the above, and there is nothing wrong, or you have reconfigured according to the guide, and the problem still persists, please proceed to the next step.

#### 3. Allow the configured port on the Firewall

There are 3 scenarios:

- I. You have 2 firewalls — cloud provider firewall and OS firewall
	- a. Your cloud provider firewall implements the rule, and your OS firewall allows everything (common).
	
	- b. Your cloud provider firewall implements the rule, and your OS firewall also implements the rules.
	
- II. You have 1 firewall — Cloud provider firewall (common)

##### Scenario I.a and II

Go to your cloud provider firewall. There should be actually 2 firewalls since we are using 2 instances — an EC2 and an RDS instance. Choose the EC2 instance’s firewall since it is the WireGuard VPN server.

If you see no port number for your WireGuard listening port, add an `allow` rule to the firewall which has the following rule:

- `allow` rule
	
- Custom UDP (WireGuard uses UDP)
	
- the port number of the WireGuard server listening port\
	
- For convenience, you can allow the VPN port from **anywhere (0.0.0.0/0)**.
	**Best practice:** restrict this to your own IP ranges if possible, to reduce exposure. For example, allow only your ISP’s IP range or your office’s network.
	
- Put some description for this rule as what this rule for. It is a good practice.

If you still can’t `ping` the VPN server, check if your client is connected to the internet. AND if you can’t still `ping`, I have no more solution.

##### Scenario I.b

In this scenario, you will need to configure both cloud provider firewall and the OS firewall.

Please see [[WireGuard Server and Client Set up - draft#Scenario I.a and II|Scenario I.a and II]] for the firewall rules configuration

Then, inside the server, you must try to add an `allow` rule to your firewall client for the WireGuard port. 

I wouldn’t go in-dept about how you configure it due to the fact that there are different firewall clients out there.

Just remember that you need to allow UDP port with the number of the WireGuard listening port.
