
# Network Mesh

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
