# Network Mesh

## Nebula

TODO: general info, deployment topology, why and how


## Setting up Nebula Lighthouse

AWS EC2 `t2.nano`:

```bash
$ uname -srmo
Linux 6.14.0-1014-aws x86_64 GNU/Linux
```

```bash
$ cat /etc/os-release
PRETTY_NAME="Ubuntu 24.04.3 LTS"
NAME="Ubuntu"
VERSION_ID="24.04"
VERSION="24.04.3 LTS (Noble Numbat)"
VERSION_CODENAME=noble
ID=ubuntu
ID_LIKE=debian
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
UBUNTU_CODENAME=noble
LOGO=ubuntu-logo
```

Allocated storage:

```bash
$ df -hT / /data
Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/root      ext4   96G  8.3G   88G   9% /
/dev/xvdbb     ext4  196G   28K  186G   1% /data
```


Download the official archive and checksum file to `/tmp`:

```bash
curl -fL -o /tmp/nebula-linux-amd64.tar.gz https://github.com/slackhq/nebula/releases/download/v1.11.2/nebula-linux-amd64.tar.gz
curl -fL -o /tmp/nebula-SHASUM256.txt https://github.com/slackhq/nebula/releases/download/v1.11.2/SHASUM256.txt
```

```bash
-rw-rw-r-- 1 ubuntu ubuntu  18M Oct 17 18:22 nebula-linux-amd64.tar.gz
-rw-rw-r-- 1 ubuntu ubuntu 6.7K Oct 17 18:22 nebula-SHASUM256.txt
```

Verify the downloaded archive against the official checksum:

```bash
cd /tmp && grep ' nebula-linux-amd64.tar.gz$' nebula-SHASUM256.txt | sha256sum -c -
```

Expected result:

```bash
nebula-linux-amd64.tar.gz: OK
```

Extract the verified archive into its own temporary directory:

```bash
mkdir -p /tmp/nebula-v1.11.2
tar -xzf /tmp/nebula-linux-amd64.tar.gz -C /tmp/nebula-v1.11.2
```

Inspect the extracted files before installing them:

```bash
$ ls -la /tmp/nebula-v1.11.2
total 35824
drwxrwxr-x  2 ubuntu ubuntu     4096 Oct 17 18:26 .
drwxrwxrwt 13 root   root      20480 Oct 17 18:26 ..
-rwxr-xr-x  1 ubuntu ubuntu 26317909 Sep 22 13:02 nebula
-rwxr-xr-x  1 ubuntu ubuntu 10336960 Sep 22 13:02 nebula-cert
```


## Install the two Nebula binaries into `/usr/local/bin`:

```bash
sudo install -o root -g root -m 0755 /tmp/nebula-v1.11.2/nebula /tmp/nebula-v1.11.2/nebula-cert /usr/local/bin/
```

```bash
$ ls -ltrh /usr/local/bin/

.........

-rwxr-xr-x 1 root   root    26M Oct 17 18:31 nebula
-rwxr-xr-x 1 root   root   9.9M Oct 17 18:31 nebula-cert
```


Verify both installed binaries and their versions:

```bash
$ /usr/local/bin/nebula -version && /usr/local/bin/nebula-cert -version
Version: 1.11.2
Version: 1.11.2
```


We'll run Nebula as a dedicate user `nebula`, let's create it:

```bash
sudo useradd --system --user-group --no-create-home --home-dir /nonexistent --shell /usr/sbin/nologin nebula
```

```bash
$ getent passwd nebula
nebula:x:999:987::/nonexistent:/usr/sbin/nologin
```


Create the configuration directory so root controls its contents while the nebula group can read them:

```bash
sudo install -d -o root -g nebula -m 0750 /etc/nebula
```

## Configuration

Our chosen overlay subnet: `10.80.0.0/24` with the lighthouse overlay IP `10.80.0.1`.

Create a protected PKI working directory in `ubunbu`'s home:

```bash
install -d -m 0700 /home/ubuntu/nebula-pki
```

Generate an encrypted v2 CA:

```bash
cd /home/ubuntu/nebula-pki
nebula-cert ca -name "Nebula 10.80.0.0/24 CA" -duration 87600h -encrypt -argon-memory 65536 -argon-parallelism 2 -argon-iterations 3 -version 2
```

Then verify the certificates have been generated:

```bash
nebula-cert print -path /home/ubuntu/nebula-pki/ca.crt
```

The output should be something like:

```json
{
	"curve": "CURVE25519",
	"details": {
		"groups": null,
		"isCa": true,
		"issuer": "",
		"name": "Nebula 10.80.0.0/24 CA",
		"networks": null,
		"notAfter": "2036-10-14T19:42:29Z",
		"notBefore": "2026-10-17T19:42:29Z",
		"unsafeNetworks": null
	},
	"fingerprint": "9423b6a87f3f196ca61ed981423b1bef03b596ea8c733a8d95d7aec9b5df8cb5",
	"publicKey": "dd78e234cbb48a31b451cfc7567fc8d4778a60008b13cc9777e555e743093fa1",
	"signature": "607bd6447070a4ee8d7147863f5c6f82236747a0e15eb3f8497a9dcb03879860928d8a114820ba47419c1cb7832d98c77e112d9713467f8f4d75ac0df6607806",
	"version": 2
}
```


Next, issue the lighthouse certificate:

```bash
nebula-cert sign -ca-crt ca.crt -ca-key ca.key -name "lighthouse" -networks "10.80.0.1/24" -groups "lighthouse" -duration 8760h -version 2 -out-crt lighthouse.crt -out-key lighthouse.key
```

Verify the lighthouse certificate:

```bash
nebula-cert print -path lighthouse.crt
```

```json
{
	"curve": "CURVE25519",
	"details": {
		"groups": [
			"lighthouse"
		],
		"isCa": false,
		"issuer": "9423b6a87f3f196ca61ed981423b1bef03b596ea8c733a8d95d7aec9b5df8cb5",
		"name": "lighthouse",
		"networks": [
			"10.80.0.1/24"
		],
		"notAfter": "2027-10-17T19:47:39Z",
		"notBefore": "2026-10-17T19:47:39Z",
		"unsafeNetworks": null
	},
	"fingerprint": "ed873d21a84592d2a7c0a35789a5fc058270f1c68df05ab75d11697ed1c8b3f9",
	"publicKey": "832a9f2999019a7cc2dedf55e134ef5e50fa50e36b44d09a8cbde612fbd35c0a",
	"signature": "d25299d8bafa6df22275271b75ed73ecfe88a0852e3e2ceb304a88f3be11979d738cff78a66331208777e5a1c7c3cbe441bcfe15a94ee5668ac1af45119ac009",
	"version": 2
}
```

Install only the lighthouse’s required files into `/etc/nebula`. This does not copy the CA private key:

```bash
sudo install -o root -g nebula -m 0640 /home/ubuntu/nebula-pki/ca.crt /home/ubuntu/nebula-pki/lighthouse.crt /home/ubuntu/nebula-pki/lighthouse.key /etc/nebula/
```


The lighthouse host will provide both discover and relay services. Create the configuration file `/etc/nebula/config.yml`:

```yaml
pki:
  ca: /etc/nebula/ca.crt
  cert: /etc/nebula/lighthouse.crt
  key: /etc/nebula/lighthouse.key
  initiating_version: 2

static_host_map: {}

lighthouse:
  am_lighthouse: true
  serve_dns: true
  dns:
    host: 10.80.0.1
    port: 53
  hosts: []

relay:
  am_relay: true
  use_relays: false
  relays: []

listen:
  host: 0.0.0.0
  port: 4242

tun:
  disabled: false
  dev: nebula1

firewall:
  outbound: []
  inbound:
    - port: 53
      proto: udp
      group: any

logging:
  level: info
  format: text
```


Set correct permissions on the config file:

```bash
sudo chown root:nebula /etc/nebula/config.yml
sudo chmod 0640 /etc/nebula/config.yml
```

Validate this new config file:

```bash
$ sudo /usr/local/bin/nebula -test -config /etc/nebula/config.yml
time=2026-10-18T18:10:06.913Z level=INFO msg="firewall:\n    inbound:\n        - group: any\n          port: 53\n          proto: udp\n    outbound: []\nlighthouse:\n    am_lighthouse: true\n    dns:\n        host: 10.80.0.1\n        port: 53\n    hosts: []\n    serve_dns: true\nlisten:\n    host: 0.0.0.0\n    port: 4242\nlogging:\n    format: text\n    level: info\npki:\n    ca: /etc/nebula/ca.crt\n    cert: /etc/nebula/lighthouse.crt\n    initiating_version: 2\n    key: /etc/nebula/lighthouse.key\nrelay:\n    am_relay: true\n    relays: []\n    use_relays: false\nstatic_host_map: {}\ntun:\n    dev: nebula1\n    disabled: false\n"
time=2026-10-18T18:10:06.918Z level=INFO msg="Firewall rule added" firewallRule="map[caName: caSha: cidr: direction:incoming endPort:53 groups:[any] host: localCidr: proto:17 startPort:53]"
time=2026-10-18T18:10:06.918Z level=INFO msg="Firewall started" firewallHashes=SHA:65bf52fb0778bf05e3b7bac9dfbed963481e710f2eeaf53d09cf1d59317bca14,FNV:715780406
time=2026-10-18T18:10:06.919Z level=INFO msg="Main HostMap created" preferredRanges=[]
time=2026-10-18T18:10:06.919Z level=INFO msg="punchy disabled"
```

Create the systemd service `/etc/systemd/system/nebula.service`:

```ini
[Unit]
Description=Nebula Lighthouse and Relay
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=nebula
Group=nebula
ExecStart=/usr/local/bin/nebula -config /etc/nebula/config.yml
Restart=on-failure
RestartSec=5
AmbientCapabilities=CAP_NET_ADMIN CAP_NET_BIND_SERVICE
CapabilityBoundingSet=CAP_NET_ADMIN CAP_NET_BIND_SERVICE
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

Reload systemd so it recognizes the new service:

```bash
sudo systemctl daemon-reload
```

Start Nebula and check whether it started successfully:

```bash
sudo systemctl start nebula
sudo systemctl status nebula --no-pager

● nebula.service - Nebula Lighthouse and Relay
     Loaded: loaded (/etc/systemd/system/nebula.service; disabled; preset: enabled)
     Active: active (running) since Sat 2026-10-17 20:10:20 UTC; 28s ago
   Main PID: 4336 (nebula)
      Tasks: 7 (limit: 517)
     Memory: 9.9M (peak: 10.1M)
        CPU: 53ms
     CGroup: /system.slice/nebula.service
             └─4336 /usr/local/bin/nebula -config /etc/nebula/config.yml

Oct 17 20:10:20 ip-172-31-32-174 systemd[1]: Started nebula.service - Nebula Lighthouse and Relay.
Oct 17 20:10:20 ip-172-31-32-174 nebula[4336]: time=2026-10-17T20:10:20.241Z level=INFO msg="Firewall started" firewallHashes=SHA:e3b0c44298fc1c149…:2166136261
Oct 17 20:10:20 ip-172-31-32-174 nebula[4336]: time=2026-10-17T20:10:20.246Z level=INFO msg=listening addr=0.0.0.0:4242
Oct 17 20:10:20 ip-172-31-32-174 nebula[4336]: time=2026-10-17T20:10:20.246Z level=INFO msg="Main HostMap created" preferredRanges=[]
Oct 17 20:10:20 ip-172-31-32-174 nebula[4336]: time=2026-10-17T20:10:20.246Z level=INFO msg="punchy disabled"
Oct 17 20:10:20 ip-172-31-32-174 nebula[4336]: time=2026-10-17T20:10:20.246Z level=INFO msg="Loaded send_recv_error config" sendRecvError=always
Oct 17 20:10:20 ip-172-31-32-174 nebula[4336]: time=2026-10-17T20:10:20.246Z level=INFO msg="Loaded accept_recv_error config" acceptRecvError=always
Oct 17 20:10:20 ip-172-31-32-174 nebula[4336]: time=2026-10-17T20:10:20.247Z level=INFO msg="Nebula interface is active" interface=nebula1 networks…rypto=false
Hint: Some lines were ellipsized, use -l to show in full.
```

Verify Nebula is listening on UDP port `4242`:

```bash
$ sudo ss -lunp | grep -E ':(53|4242)\b'
UNCONN 0      0               10.80.0.1:53        0.0.0.0:*    users:(("nebula",pid=10122,fd=9))
UNCONN 0      0              127.0.0.54:53        0.0.0.0:*    users:(("systemd-resolve",pid=329,fd=16))
UNCONN 0      0           127.0.0.53%lo:53        0.0.0.0:*    users:(("systemd-resolve",pid=329,fd=14))
UNCONN 0      0                 0.0.0.0:4242      0.0.0.0:*    users:(("nebula",pid=10122,fd=7))
```

Enable it at boot:

```bash
$ sudo systemctl enable nebula
Created symlink /etc/systemd/system/multi-user.target.wants/nebula.service → /etc/systemd/system/nebula.service.
```

Add this inbound rule to the EC2 instance’s security group:

| Type | Protocol | Port | Source |
|---|---|---:|---|
| Custom UDP | UDP | 4242 | `0.0.0.0/0` |


## Ryzen Severs

The lighthouse and relay are ready.
Next is issuing the six server certificates. Planned overlay addresses:


| Server | Nebula overlay IP |
|---|---|
| `ryzen-0` | `10.80.0.10/24` |
| `ryzen-1` | `10.80.0.11/24` |
| `ryzen-2` | `10.80.0.12/24` |
| `ryzen-3` | `10.80.0.13/24` |
| `ryzen-4` | `10.80.0.14/24` |
| `ryzen-5` | `10.80.0.15/24` |


Generate the first server certificate:

```bash
cd /home/ubuntu/nebula-pki
nebula-cert sign -ca-crt ca.crt -ca-key ca.key -name "ryzen-0" -networks "10.80.0.10/24" -groups "servers" -duration 8760h -version 2 -out-crt ryzen-0.crt -out-key ryzen-0.key
nebula-cert sign -ca-crt ca.crt -ca-key ca.key -name "ryzen-1" -networks "10.80.0.11/24" -groups "servers" -duration 8760h -version 2 -out-crt ryzen-1.crt -out-key ryzen-1.key
nebula-cert sign -ca-crt ca.crt -ca-key ca.key -name "ryzen-2" -networks "10.80.0.12/24" -groups "servers" -duration 8760h -version 2 -out-crt ryzen-2.crt -out-key ryzen-2.key
nebula-cert sign -ca-crt ca.crt -ca-key ca.key -name "ryzen-3" -networks "10.80.0.13/24" -groups "servers" -duration 8760h -version 2 -out-crt ryzen-3.crt -out-key ryzen-3.key
nebula-cert sign -ca-crt ca.crt -ca-key ca.key -name "ryzen-4" -networks "10.80.0.14/24" -groups "servers" -duration 8760h -version 2 -out-crt ryzen-4.crt -out-key ryzen-4.key
nebula-cert sign -ca-crt ca.crt -ca-key ca.key -name "ryzen-5" -networks "10.80.0.15/24" -groups "servers" -duration 8760h -version 2 -out-crt ryzen-5.crt -out-key ryzen-5.key
```

Verify that all six certificate/key pairs exist:

```bash
ls -l /home/ubuntu/nebula-pki/ryzen-{0..5}.{crt,key}
```

All six server certificate/key pairs exist with secure 0600 permissions.

```bash
-rw------- 1 ubuntu ubuntu 318 Oct 18 06:48 /home/ubuntu/nebula-pki/ryzen-0.crt
-rw------- 1 ubuntu ubuntu 127 Oct 18 06:48 /home/ubuntu/nebula-pki/ryzen-0.key
-rw------- 1 ubuntu ubuntu 318 Oct 18 07:13 /home/ubuntu/nebula-pki/ryzen-1.crt
-rw------- 1 ubuntu ubuntu 127 Oct 18 07:13 /home/ubuntu/nebula-pki/ryzen-1.key
-rw------- 1 ubuntu ubuntu 318 Oct 18 07:13 /home/ubuntu/nebula-pki/ryzen-2.crt
-rw------- 1 ubuntu ubuntu 127 Oct 18 07:13 /home/ubuntu/nebula-pki/ryzen-2.key
-rw------- 1 ubuntu ubuntu 318 Oct 18 07:13 /home/ubuntu/nebula-pki/ryzen-3.crt
-rw------- 1 ubuntu ubuntu 127 Oct 18 07:13 /home/ubuntu/nebula-pki/ryzen-3.key
-rw------- 1 ubuntu ubuntu 318 Oct 18 07:13 /home/ubuntu/nebula-pki/ryzen-4.crt
-rw------- 1 ubuntu ubuntu 127 Oct 18 07:13 /home/ubuntu/nebula-pki/ryzen-4.key
-rw------- 1 ubuntu ubuntu 318 Oct 18 07:13 /home/ubuntu/nebula-pki/ryzen-5.crt
-rw------- 1 ubuntu ubuntu 127 Oct 18 07:13 /home/ubuntu/nebula-pki/ryzen-5.key
```



## Generate Laptop Certificates

The laptop overlay IPs start at `10.80.0.100/24`.

| Owner ID | Nebula IP | Group |
|---|---|---|
| `jo14591n` | `10.80.0.100/24` | `laptops` |


```bash
nebula-cert sign -ca-crt ca.crt -ca-key ca.key -name "jo14591n" -networks "10.80.0.100/24" -groups "laptops" -duration 8760h -version 2 -out-crt jo14591n.crt -out-key jo14591n.key
```



# Shared client configuration

Replace `<LIGHTHOUSE_PUBLIC_IP>` with the EC2 Elastic IP.

## Ryzen server config

Use this as `/etc/nebula/config.yml`:

```yaml
pki:
  ca: /etc/nebula/ca.crt
  cert: /etc/nebula/host.crt
  key: /etc/nebula/host.key
  initiating_version: 2

static_host_map:
  "10.80.0.1": ["<LIGHTHOUSE_PUBLIC_IP>:4242"]

lighthouse:
  am_lighthouse: false
  hosts:
    - "10.80.0.1"

relay:
  relays:
    - "10.80.0.1"
  am_relay: false
  use_relays: true

listen:
  host: 0.0.0.0
  port: 0

punchy:
  punch: true
  respond: true

tun:
  disabled: false

firewall:
  outbound:
    - port: any
      proto: any
      host: any

  inbound:
    - port: any
      proto: icmp
      host: any
    - port: 22
      proto: tcp
      group: laptops
    - port: any
      proto: any
      group: servers

logging:
  level: info
  format: text
```

This permits:

- ICMP testing from overlay members.
- SSH from certificates in the laptops group.
- All overlay traffic between servers.
- No other inbound overlay traffic.

## Ubuntu Ryzen server setup

Each server receives only:

```
ca.crt
ryzen-N.crt
ryzen-N.key
```

Rename its certificate and key to `host.crt` and `host.key` during installation.

### Install Nebula

```bash
cd /tmp
curl -fLO https://github.com/slackhq/nebula/releases/download/v1.11.2/nebula-linux-amd64.tar.gz
curl -fLO https://github.com/slackhq/nebula/releases/download/v1.11.2/SHASUM256.txt
grep ' nebula-linux-amd64.tar.gz$' SHASUM256.txt | sha256sum -c -
mkdir -p nebula-v1.11.2
tar -xzf nebula-linux-amd64.tar.gz -C nebula-v1.11.2
sudo install -o root -g root -m 0755 nebula-v1.11.2/nebula /usr/local/bin/nebula
```

The official project distributes Linux, macOS, and Windows binaries and also documents distribution packages. https://nebula.defined.net/docs/guides/quick-start/?utm_source=chatgpt.com

### Create the service account

```bash
sudo useradd --system --user-group --no-create-home --home-dir /nonexistent --shell /usr/sbin/nologin nebula
```

### Install credentials

For ryzen-0, for example:

```bash
sudo install -d -o root -g nebula -m 0750 /etc/nebula
sudo install -o root -g nebula -m 0640 ca.crt /etc/nebula/ca.crt
sudo install -o root -g nebula -m 0640 ryzen-0.crt /etc/nebula/host.crt
sudo install -o root -g nebula -m 0640 ryzen-0.key /etc/nebula/host.key
```

Use the matching certificate/key on each server.

### Install and validate the configuration

After creating `/etc/nebula/config.yml` from the server template:

```bash
sudo chown root:nebula /etc/nebula/config.yml
sudo chmod 0640 /etc/nebula/config.yml
sudo /usr/local/bin/nebula -test -config /etc/nebula/config.yml
```

### Create the systemd service

Create `/etc/systemd/system/nebula.service`:

```ini
[Unit]
Description=Nebula Overlay Client
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=nebula
Group=nebula
ExecStart=/usr/local/bin/nebula -config /etc/nebula/config.yml
Restart=on-failure
RestartSec=5
AmbientCapabilities=CAP_NET_ADMIN
CapabilityBoundingSet=CAP_NET_ADMIN
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```


`CAP_NET_ADMIN` lets the unprivileged service create and configure its TUN interface. https://nebula.defined.net/docs/guides/running-nebula-as-non-root/?utm_source=chatgpt.com

Start it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now nebula
sudo systemctl status nebula --no-pager
```

## Laptop Configuration

A laptop receives:

```
ca.crt
<slack_id>.crt
<slack_id>.key
```

Use this client policy:

```yaml
lighthouse:
  am_lighthouse: false
  hosts:
    - "10.80.0.1"

relay:
  relays:
    - "10.80.0.1"
  am_relay: false
  use_relays: true

listen:
  host: 0.0.0.0
  port: 0

punchy:
  punch: true
  respond: true

tun:
  disabled: false

firewall:
  outbound:
    - port: any
      proto: any
      host: any

  inbound:
    - port: any
      proto: icmp
      group: servers

logging:
  level: info
  format: text
```

Replace the three uppercase file placeholders with platform-specific absolute paths below.


## MacOS laptop

### Install

```bash
brew install nebula
nebula -version
```

The official repository documents Homebrew installation. https://github.com/slackhq/nebula?utm_source=chatgpt.com

The version must be at least `1.10` because these are `v2` certificates.

### Install credentials

```bash
sudo install -d -o root -g wheel -m 0700 /etc/nebula
sudo install -o root -g wheel -m 0600 ca.crt /etc/nebula/ca.crt
sudo install -o root -g wheel -m 0600 <slack_id>.crt /etc/nebula/host.crt
sudo install -o root -g wheel -m 0600 <slack_id>.key /etc/nebula/host.key
```

Use these paths in the configuration:

```yaml
pki:
  ca: /etc/nebula/ca.crt
  cert: /etc/nebula/host.crt
  key: /etc/nebula/host.key
```

Save the complete laptop configuration as `/etc/nebula/config.yml`, then:

```bash
sudo chmod 0600 /etc/nebula/config.yml
sudo nebula -test -config /etc/nebula/config.yml
sudo nebula -service install -config /etc/nebula/config.yml
sudo nebula -service start
```


## Windows laptop

### Install files

Download the official `nebula-windows-amd64.zip` for `v1.11.2` and extract nebula.exe to:

```
C:\Program Files\Nebula\nebula.exe
```

Place the credentials in:

```
C:\ProgramData\Nebula\ca.crt
C:\ProgramData\Nebula\host.crt
C:\ProgramData\Nebula\host.key
```

Save the configuration as:

```
C:\ProgramData\Nebula\config.yml
```

Use absolute Windows paths:

```yaml
pki:
  ca: "C:\\ProgramData\\Nebula\\ca.crt"
  cert: "C:\\ProgramData\\Nebula\\host.crt"
  key: "C:\\ProgramData\\Nebula\\host.key"
  initiating_version: 2
```

Absolute paths are important because the Windows service does not use the interactive shell’s working directory. https://github.com/slackhq/nebula/issues/1140?utm_source=chatgpt.com

### Validate and install the service

Run PowerShell as Administrator:

```powershell
& 'C:\Program Files\Nebula\nebula.exe' -test -config 'C:\ProgramData\Nebula\config.yml'
& 'C:\Program Files\Nebula\nebula.exe' -service install -config 'C:\ProgramData\Nebula\config.yml'
& 'C:\Program Files\Nebula\nebula.exe' -service start
Get-Service Nebula
```

Nebula’s Windows service supports `-service install` and `-service start`. https://github.com/slackhq/nebula/discussions/1089?utm_source=chatgpt.com


## Connectivity test

After one Ryzen server and one laptop are running:

```bash
ping 10.80.0.10
ssh <server-user>@10.80.0.10
ssh <server-user>@ryzen-0
```


---

## Ryzen and MacBook Client Setup


## Goal

Establish secure remote connectivity between the CS371 Ryzen servers and student laptops using Nebula VPN.

The network mesh allows students to administer their Linux servers from different networks without exposing SSH directly to the public internet.

## Prerequisites

- Lenovo ThinkCentre M715q running Ubuntu 24.04.5 LTS
- MacBook running macOS
- Internet connectivity on both devices
- Nebula VPN v1.11.2 installed on both devices
- Nebula certificates and private keys provided for each device
- Access to the CS371 Nebula lighthouse
- OpenSSH server configured on Ubuntu
- SSH public-key authentication

## Design decisions

We selected Nebula VPN to establish encrypted connectivity between devices across different networks.

The network uses the following Nebula IP addresses:

| Device | Nebula IP | Role |
|---|---|---|
| Lighthouse | 10.80.0.1 | Peer discovery |
| Ryzen (`ryzen-0`) | 10.80.0.10 | Linux server |
| MacBook (`bayron`) | 10.80.0.101 | Remote administration |

The Ryzen runs Nebula as a systemd service, while the MacBook currently runs Nebula manually.

SSH access is restricted through Nebula firewall rules.

## Installation and configuration

### 1. Install Nebula

Install Nebula v1.11.2 on the Ryzen and MacBook.

Place the appropriate certificate, private key, and CA certificate in `/etc/nebula/` on each device.

Never commit private keys or certificates containing sensitive information to GitHub.

### 2. Configure Nebula

Both devices use `/etc/nebula/config.yml`.

The configuration includes the lighthouse, device certificates, VPN interface, and firewall rules.

The Ryzen's inbound firewall configuration includes:

```yaml
firewall:
  outbound:
    - port: any
      proto: any
      host: any

  inbound:
    - port: any
      proto: icmp
      host: any

    - port: 22
      proto: tcp
      host: bayron

    - port: 22
      proto: tcp
      group: servers
```

The `host: bayron` rule matches the MacBook's Nebula certificate identity.

### 3. Start Nebula on Ubuntu

```bash
sudo systemctl enable nebula
sudo systemctl restart nebula
sudo systemctl status nebula
```

### 4. Start Nebula on macOS

```bash
sudo nebula -config /etc/nebula/config.yml
```

Keep this process running while using the VPN.

## Verification

### Test Nebula connectivity

From the MacBook:

```bash
ping -c 4 10.80.0.10
```

Expected result: Successful ICMP replies.

### Test SSH connectivity

```bash
nc -vz -w 5 10.80.0.10 22
```

Expected result:

```text
Connection to 10.80.0.10 port 22 [tcp/ssh] succeeded!
```

### Connect remotely

```bash
ssh -i ~/.ssh/id_ed25519 milkalmond@10.80.0.10
```

Expected result:

```text
milkalmond@cs371-bc9b0c:~$
```

Remote SSH connectivity was successfully verified from a separate internet connection outside the home network.

## Troubleshooting

### SSH connection timed out

Initially, Nebula ICMP ping succeeded, but SSH connections to port 22 timed out.

We enabled debug logging and inspected the Ryzen's Nebula logs:

```bash
sudo journalctl -u nebula --since "10 minutes ago" --no-pager
```

The logs reported:

```text
dropping inbound packet
reason="no matching rule in firewall table"
```

The original firewall configuration used:

```yaml
groups:
  - laptops
  - servers
```

This required the connecting peer to match both groups.

We corrected the configuration by using the MacBook's certificate identity:

```yaml
- port: 22
  proto: tcp
  host: bayron
```

After restarting Nebula, TCP port 22 became reachable and SSH authentication succeeded.

### SSH key authentication

The SSH private key must be available on the client device initiating the connection.

For this setup, the private key is stored on the MacBook at:

```text
~/.ssh/id_ed25519
```

### Security considerations

Private keys must not be uploaded to the repository. Any previously exposed Nebula private keys should be rotated before production or sensitive use.
