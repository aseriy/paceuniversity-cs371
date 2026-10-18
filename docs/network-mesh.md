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
  inbound: []

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
time=2026-10-17T20:06:20.563Z level=INFO msg="firewall:\n    inbound: []\n    outbound: []\nlighthouse:\n    am_lighthouse: true\n    hosts: []\nlisten:\n    host: 0.0.0.0\n    port: 4242\nlogging:\n    format: text\n    level: info\npki:\n    ca: /etc/nebula/ca.crt\n    cert: /etc/nebula/lighthouse.crt\n    initiating_version: 2\n    key: /etc/nebula/lighthouse.key\nrelay:\n    am_relay: true\n    relays: []\n    use_relays: false\nstatic_host_map: {}\ntun:\n    dev: nebula1\n    disabled: false\n"
time=2026-10-17T20:06:20.569Z level=INFO msg="Firewall started" firewallHashes=SHA:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855,FNV:2166136261
time=2026-10-17T20:06:20.569Z level=INFO msg="Main HostMap created" preferredRanges=[]
time=2026-10-17T20:06:20.570Z level=INFO msg="punchy disabled"
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
AmbientCapabilities=CAP_NET_ADMIN
CapabilityBoundingSet=CAP_NET_ADMIN
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
$ sudo ss -lunp | grep ':4242'
UNCONN 0      0                 0.0.0.0:4242      0.0.0.0:*    users:(("nebula",pid=4336,fd=7))
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

