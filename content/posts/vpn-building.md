---
author: "King Consk"
title: "Building a VPN Service"
date: "2022-03-17"
description: "Create your own VPN service using Headscale."
tags: ["VPN", "Headscale", "Tailscale"]
categories: ["howto"]
draft: false
ShowToc: true
disableHLJS: false
---

I've been exploring using Headscale to build my own VPN service.

<!--more-->

### Overview
There is a misconception that using a VPN will hide your activity on the Internet. Whilst this is partially true, the concern is around what information the VPN provider is collecting and how they [might be storing/using that information](https://www.techradar.com/news/this-top-vpn-provider-may-have-leaked-millions-of-user-details).

It's simple enough to purchase a virtual private server from somewhere and run OpenVPN or Wireguard to host a single VPN server. But what if you wanted to mimic the ease of use that purchasing a third-party VPN provides? What if you wanted to be able to exit in many different countries around the world?

[Tailscale](https://tailscale.com/blog/how-tailscale-works/) is an interesting product that provides a key management layer to [Wireguard](https://www.wireguard.com/) along with some NAT traversal to allow hosts behind CGNAT or other firewall restrictions to connect to one another. The control component is closed-source and while it has a free tier, you’re still placing trust in a third party. The client component on the other hand is open source. Enter [Headscale]( https://github.com/juanfont/headscale), an open source project to mimic the functionality of Tailscale server component. 

One of the features of Tailscale is to allow a node on a Tailnet to advertise as an exit point. Nodes can advertise routes and access control policies can restrict which nodes can access what resources. Let's try and advertise a default route `0.0.0.0/0` with an allow all policy. 

### Architecture
We'll need the following components:
- [Headscale](https://github.com/juanfont/headscale/blob/main/docs/running-headscale-linux.md) server
- [DERP](https://tailscale.com/kb/1118/custom-derp-servers/#why-run-your-own-derp-server) server (Designated Encrypted Relay for Packets) - While not strictly necessary. Let's build one anyway.
- Exit points running [Tailscale](https://tailscale.com/download/linux)

Here's a diagram of what's going to be built. 
![vpn](/images/vpn.jpg)

### Installation
Let's start by installing some applications on our Headscale server. We'll use Nginx to act as a reverse proxy so that we can host both the Headscale and DERP server components on a single server. 

```bash
apt update && apt upgrade -y
apt install nginx certbot python3-certbot-nginx

# Install the Tailscale client
curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/focal.gpg | apt-key add -
curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/focal.list | tee /etc/apt/sources.list.d/tailscale.list
apt-get update
apt-get install tailscale

# Install Headscale
wget --output-document=/usr/local/bin/headscale https://github.com/juanfont/headscale/releases/download/v0.14.0/headscale_0.14.0_linux_amd64
chmod +x /usr/local/bin/headscale

mkdir -p /etc/headscale
mkdir -p /var/lib/headscale

touch /var/lib/headscale/db.sqlite
useradd headscale
chown -R headscale:headscale /var/lib/headscale/
```
### Configuration
Now we need to configure the Headscale server. I won't go into much detail on what I'm doing here, you can have a look at the [project documentation](https://github.com/juanfont/headscale/blob/main/docs/running-headscale-linux.md) if you need more information. 

```bash
cat << EOF >> /etc/headscale/config.yaml
---
server_url: http://127.0.0.1:8080
listen_addr: 0.0.0.0:8080
grpc_listen_addr: 0.0.0.0:50443
grpc_allow_insecure: false
private_key_path: /var/lib/headscale/private.key
ip_prefixes:
  - fd7a:115c:a1e0::/48
  - 100.64.0.0/10
derp:
  urls:
    - https://controlplane.tailscale.com/derpmap/default
  paths: []
  auto_update_enabled: true
  update_frequency: 24h
disable_check_updates: false
ephemeral_node_inactivity_timeout: 30m
db_type: sqlite3
db_path: /var/lib/headscale/db.sqlite
acme_url: https://acme-v02.api.letsencrypt.org/directory
acme_email: ""
tls_letsencrypt_hostname: ""
tls_client_auth_mode: relaxed
tls_letsencrypt_cache_dir: /etc/letsencrypt/live/headscale.example.com/
tls_letsencrypt_challenge_type: HTTP-01
tls_letsencrypt_listen: ":http"
tls_cert_path: ""
tls_key_path: ""
log_level: info
acl_policy_path: "/etc/headscale/acl.hujson"
dns_config:
  nameservers:
    - 1.1.1.1
  domains: []
  magic_dns: true
  base_domain: example.com
unix_socket: /var/run/headscale/headscale.sock
unix_socket_permission: "0770"
EOF
```

We can use systemd to manage Headscale as a daemon. 

```bash
cat << EOF >> /etc/systemd/system/headscale.service
[Unit]
Description=headscale controller
After=syslog.target
After=network.target

[Service]
Type=simple
User=headscale
Group=headscale
ExecStart=/usr/local/bin/headscale serve
Restart=always
RestartSec=5

# Optional security enhancements
NoNewPrivileges=yes
PrivateTmp=yes
ProtectSystem=strict
ProtectHome=yes
ReadWritePaths=/var/lib/headscale /var/run/headscale
AmbientCapabilities=CAP_NET_BIND_SERVICE
RuntimeDirectory=headscale

[Install]
WantedBy=multi-user.target
EOF
```
An ACL can be set up for who has access to what. We can treat namespaces as a user account. The admin account would be able to use SSH on all of the nodes, and the user account will access anything from any server with the tag `prod-exit`, as well as be able to browse anywhere using HTTP/S and DNS. 

```bash
cat << EOF >> /etc/headscale/acl.hujson
{
  "groups": {
    "group:admin": ["admin1"],
    "group:user": ["user1"]
  },
  "tagOwners": {
    "tag:prod-controller": ["group:admin"],
    "tag:prod-exit": ["group:admin", "group:user"]
  },
  "acls": [
    {
      "action": "accept",
      "users": ["group:admin"],
      "ports": [
        "tag:prod-controller:22",
        "tag:prod-exit:22"
        ]
    },
    {
      "action": "accept",
      "users": ["group:user"],
      "ports": [
        "tag:prod-exit:*",
        "*:80,443,53"
      ]
    }
  ],
  "derpMap": {
    "OmitDefaultRegions": true,
    "Regions": { "900": {
      "RegionID": 900,
      "RegionCode": "myderp",
      "Nodes": [{
          "Name": "1",
          "RegionID": 900,
          "HostName": "derp.example.com"
      }]
    }}
  }
}
EOF
```

### DERP Installation & Configuration

To install our DERP server, we need to use GOLANG to build it.

```bash
wget https://go.dev/dl/go1.17.7.linux-amd64.tar.gz
rm -rf /usr/local/go && tar -C /usr/local -xzf go1.17.7.linux-amd64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin' | tee /etc/profile.d/go.sh
go install tailscale.com/cmd/derper@main
mv go/bin/derper /usr/local/bin/
```

Again we can use systemd to run it as a daemon using a shell script. 

```bash
cat << EOF >> /etc/systemd/system/derper.service
[Unit]
Description=derper
After=syslog.target
After=network.target

[Service]
Type=forking
ExecStart=/usr/local/bin/derper_start.sh
ExecStop=/usr/local/bin/derper_stop.sh

[Install]
WantedBy=multi-user.target
EOF

cat << EOF >> /usr/local/bin/derper_start.sh
#!/bin/bash
# Start the derper service
nohup /usr/local/bin/derper -a :8081 -http-port -1 -stun --verify-clients &
echo $! > /var/run/derper.pid
EOF
chmod +x /usr/local/bin/derper_start.sh

cat << EOF >> /usr/local/bin/derper_stop.sh
#!/bin/bash 
# Stop the derper service
kill `cat /var/run/derper.pid`
rm -rf /var/run/derper.pid
EOF
chmod +x /usr/local/bin/derper_stop.sh
```

### NGINX Configuration
Now we can set up NGINX to act as a reverse proxy to our DERP and Headscale daemons. 

```bash
cat << EOF >> /etc/nginx/sites-available/headscale.example.com.conf
server {
    server_name headscale.example.com;

    client_body_timeout 5m;
    client_header_timeout 5m;

    access_log            /var/log/nginx/headscale.example.com.access.log;
    error_log            /var/log/nginx/headscale.example.com.error.log info;

    # reverse proxy
    location / {
         proxy_pass http://127.0.0.1:8080;  # headscale listen_addr
         proxy_read_timeout 6m;
         proxy_ignore_client_abort off;
         proxy_request_buffering off;
         proxy_buffering off;
         proxy_no_cache "always";
         proxy_set_header Host $host;
         proxy_set_header X-Real-IP $remote_addr;
         proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
EOF

cat << EOF >> /etc/nginx/sites-available/derp.example.com.conf
server {
    server_name derp.example.com;

    client_body_timeout 5m;
    client_header_timeout 5m;

    access_log            /var/log/nginx/derp.example.com.access.log;
    error_log            /var/log/nginx/derp.example.com.error.log info;

    # reverse proxy
    location / {
         proxy_pass http://127.0.0.1:8081;  # derp listen_addr
         proxy_read_timeout 6m;
         proxy_ignore_client_abort off;
         proxy_request_buffering off;
         proxy_buffering off;
         proxy_no_cache "always";
         proxy_set_header Host $host;
         proxy_set_header X-Real-IP $remote_addr;
         proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
EOF

ln -s /etc/nginx/sites-available/headscale.example.com.conf /etc/nginx/sites-enabled/headscale.example.com.conf
ln -s /etc/nginx/sites-available/derp.example.com.conf /etc/nginx/sites-enabled/derp.example.com.conf
```

Certbot makes it easy to register with Let's Encrypt and generate a TLS certificate for both our DERP and Headscale domains. 

```bash
certbot
# Follow prompts to issue certs and enable https redirection.  
```

### Run and configure Headscale
After a `daemon-reload` we can load and enable the services we've set up. 

```bash
systemctl daemon-reload
systemctl enable nginx --now
systemctl enable headscale --now
systemctl enable derper --now
```

Our namespaces we referenced earlier in our ACL need to be created. 

```bash
headscale namespaces create admin1
headscale namespaces create user1
```

Generate a preauth key for each our namespaces to use to authenticate when we register clients in the Tailnet. By default these will expire after 1 hour.

```bash
headscale --namespace admin1 preauthkeys create --reusable --expiration 24h
#AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA

headscale --namespace user1 preauthkeys create --reusable --expiration 24h
#BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB
```

Finally we can start adding some nodes. We'll start with the Headscale server itself as it will need to be a member in order to verify the DERP clients. 

```bash
tailscale up --login-server https://headscale.example.com --authkey AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA --advertise-tags 'tag:prod-controller'
```

### Adding exit nodes
Once we've got a working Headscale server we can build our exit nodes and add them to the Tailnet. We'll need to install Tailscale similar to what we did previously. IP forwarding also needs to be enabled for the host to act as a router and forward traffic. 

```bash
echo 'net.ipv4.ip_forward = 1' | tee -a /etc/sysctl.conf
echo 'net.ipv6.conf.all.forwarding = 1' | tee -a /etc/sysctl.conf
sysctl -p /etc/sysctl.conf

tailscale up --login-server https://headscale.example.com --authkey AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA --advertise-tags 'tag:prod-exit' --advertise-routes '0.0.0.0/0,::/0' --advertise-exit-node=true
```

The node then needs to be authorised to be able to advertise itself as an exit as well as advertise its' routes. We first identify the node's ID number.

```bash
headscale nodes list

headscale routes enable --identifier 2 --route '0.0.0.0/0,::/0' 
```

### Client Setup
Now that we've got an exit node, we can configure a client connect to the Tailnet. After installing the [Windows package](https://tailscale.com/download/windows), we set two registry keys to use our custom Headscale server.

```
HKLM:\SOFTWARE\Tailscale IPN\UnattendedMode=always
HKLM:\SOFTWARE\Tailscale IPN\LoginURL=https://headscale.example.com
```

After loading up tailscale, it will prompt to login which will then load a webpage. Because we're using Headscale behind a proxy the URL will be incorrect and set to the localhost `127.0.0.1:8080`. Change the first part of this to your headscale URL and you'll get instructions on how to enrol the client. This will include a generated client key.   

```bash
headscale -n user1 nodes register --key CCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCC
```

Back in the windows client if we select our exit node, we can then start routing all traffic through it. Confirm that you're showing a different IP and country from your client machine.

```bash
curl ifconfig.io
curl ifconfig.io/country_code
```

If this were a Linux client, we'd simply enrol the client in the Tailnet similar to before then re-run the Tailscale client and use the `--exit-node` option.

```bash
tailscale status
tailscale up --exit-node=<exit node IP>
```

### Conclusion

Adding more exits is now trivial as we simply just create an auth key, enable forwarding and enrol the node in the Tailnet. We don't even need to have a node that's exposed to the Internet as long as it can reach the Headscale server. In my example I used Azure with peered vnet between different regions. I then just used the Headscale server as a bastion host to configure the exit server. However if you built a custom image or used an cloud init script you could build the exit node anywhere. 

```yaml
#cloud-config
runcmd:
  - 'echo "net.ipv4.ip_forward = 1" | tee -a /etc/sysctl.conf'
  - 'echo "net.ipv6.conf.all.forwarding = 1" | tee -a /etc/sysctl.conf'
  - 'sysctl -p /etc/sysctl.conf'
  - 'curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/focal.gpg | apt-key add -'
  - 'curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/focal.list | tee /etc/apt/sources.list.d/tailscale.list'
  - 'apt-get update'
  - 'apt-get install tailscale -y'
  - 'tailscale up --login-server https://headscale.example.com --authkey AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA --advertise-tags "tag:prod-exit" --advertise-routes "0.0.0.0/0,::/0" --advertise-exit-node=true'
```

Further work could be done to integrate Headscale with an authentication provider such as LDAP along with some fine tuned ACLs. 
