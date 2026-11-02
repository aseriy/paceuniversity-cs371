# Cloud Infrastructure
## Goal

Build a private, multi-node Docker Swarm cluster for the CS371 project using the existing Nebula VPN mesh.

The completed infrastructure should support:

1. Docker Engine installed and operational on each server.
2. `ryzen-0` configured as the Docker Swarm manager.
3. Additional Ryzen servers joined as worker nodes.
4. A private Docker image registry running on the manager.
5. Docker Swarm Visualizer running on the manager.
6. Deployment of a sample application across multiple nodes using a Docker Compose YAML file and `docker stack deploy`.

**Current status:** The manager is operational. Worker enrollment, registry, Visualizer, and multi-node deployment remain pending.

## Prerequisites

- Lenovo ThinkCentre Ryzen server (`ryzen-0`).
- Ubuntu 24.04.5 LTS.
- Docker Engine 29.1.3.
- Nebula VPN installed and configured.
- Nebula certificate assigned to the `servers` group.
- SSH access to the server.
- Administrative privileges through `sudo`.
- Additional servers with Docker and Nebula for future worker enrollment.

### Network information

| Component | Address |
|---|---|
| `ryzen-0` Nebula VPN | `10.80.0.10` |
| `ryzen-0` home LAN (observed) | `192.168.4.30` |
| Nebula lighthouse | `10.80.0.1` |
| MacBook Nebula VPN | `10.80.0.101` |

The home LAN address may change through DHCP. The Nebula address is used for Swarm communication.

## Design decisions

### Manager and worker architecture

`ryzen-0` is the initial Docker Swarm manager.

Additional Ryzen servers will join as worker nodes once their operating systems, Docker installations, Nebula certificates, and networking are ready.

A single-manager Swarm is sufficient for initial testing but does not provide manager high availability.

### Private networking

Docker Swarm communicates over the existing Nebula VPN mesh.

The manager advertises `10.80.0.10`, avoiding dependence on its home LAN address.

Swarm communication ports are restricted to Nebula nodes belonging to the `servers` group.

The Docker Swarm ports should not be exposed publicly through the home router.

### Planned services

The manager will host a Docker registry and Swarm Visualizer.

The project will reference the `aseriy/cockroachdb-sandbox` GitHub repository for deployment examples.

Registry storage, access controls, Visualizer deployment, and the test application have not yet been implemented.

## Installation and configuration

### 1. Verify Docker installation

On `ryzen-0`:

```bash
docker info --format 'Docker: {{.ServerVersion}} | Swarm: {{.Swarm.LocalNodeState}}'
```

Initial observed result:

```text
Docker: 29.1.3 | Swarm: inactive
```

Test container execution:

```bash
docker run --rm hello-world
```

Observed result:

```text
Hello from Docker!
```

This confirmed that Docker could contact its daemon, download an image, and execute a container.

### 2. Identify the Nebula network interface

```bash
ip -br addr
```

Observed Nebula interface:

```text
tun0 UNKNOWN 10.80.0.10/24
```

### 3. Configure Nebula firewall

File:

```text
/etc/nebula/config.yml
```

The following inbound rules were added while preserving the existing SSH and ICMP rules:

```yaml
    # Docker Swarm manager communication
    - port: 2377
      proto: tcp
      group: servers

    # Docker Swarm node communication
    - port: 7946
      proto: tcp
      group: servers

    - port: 7946
      proto: udp
      group: servers

    # Docker Swarm overlay networking
    - port: 4789
      proto: udp
      group: servers
```

Validate the configuration:

```bash
sudo nebula -config /etc/nebula/config.yml -test
```

The configuration passed validation.

The Nebula service was restarted after preparing a timed rollback mechanism. Remote SSH connectivity was restored, and the rollback timer was canceled.

The service logs confirmed that all four Swarm firewall rules were loaded.

Worker nodes will also require appropriate inbound Nebula rules for inter-node communication.

### 4. Initialize Docker Swarm

On `ryzen-0`:

```bash
docker swarm init --advertise-addr 10.80.0.10
```

Docker confirmed that the Swarm was initialized.

The worker join token must be treated as a secret. Do not commit it to GitHub or include it in this document.

### 5. Join worker nodes (pending)

After `ryzen-1` is ready, retrieve the worker join command on the manager:

```bash
docker swarm join-token worker
```

Execute the resulting join command on the worker without publishing its token.

Confirm that the worker can communicate with the manager through Nebula and that required firewall rules are in place.

Verify enrollment from the manager:

```bash
docker node ls
```

### 6. Configure private Docker registry (pending)

Deploy a private registry on the manager after determining the storage location, network access, and authentication or transport-security requirements.

Reference:

https://github.com/aseriy/cockroachdb-sandbox

### 7. Configure Swarm Visualizer (pending)

Deploy a visualization service to inspect Swarm nodes and scheduled containers.

Reference:

https://github.com/aseriy/cockroachdb-sandbox

### 8. Deploy sample application (pending)

Create a Compose-compatible YAML file defining a small application with Swarm-compatible service configuration.

Deploy it using:

```bash
docker stack deploy -c compose.yaml cs371-test
```

Verify service placement:

```bash
docker stack services cs371-test
docker service ls
docker service ps cs371-test_app
```

The final command assumes the example defines a service named `app`; update it to match the actual service name.

## Verification

### Docker Engine

```bash
docker run --rm hello-world
```

Result: Successful.

### Swarm manager

```bash
docker node ls
```

Observed output:

```text
HOSTNAME       STATUS   AVAILABILITY   MANAGER STATUS   ENGINE VERSION
cs371-bc9b0c   Ready    Active         Leader           29.1.3
```

### Advertised manager address

```bash
docker info --format 'Swarm: {{.Swarm.LocalNodeState}} | Manager IP: {{.Swarm.NodeAddr}} | Managers: {{.Swarm.Managers}} | Nodes: {{.Swarm.Nodes}}'
```

Observed result:

```text
Swarm: active | Manager IP: 10.80.0.10 | Managers: 1 | Nodes: 1
```

### Nebula firewall

The Nebula service logs confirmed the following inbound rules for the `servers` group:

| Port | Protocol | Purpose |
|---|---|---|
| 2377 | TCP | Swarm management |
| 7946 | TCP | Node communication |
| 7946 | UDP | Node communication |
| 4789 | UDP | Overlay network traffic |

Remote SSH access continued working after the Nebula restart.

Actual inter-node Swarm connectivity has not yet been tested.

## Troubleshooting

### Nebula firewall initially blocked Swarm ports

The original firewall allowed ICMP and SSH but not Docker Swarm traffic.

Resolution: Added the required Swarm rules for Nebula's `servers` group and validated the configuration.

### Missing TCP 2377 rule

Initial validation showed TCP/UDP 7946 and UDP 4789 but not TCP 2377.

Resolution: Added TCP 2377 and confirmed it appeared in the validation output.

### Remote Nebula restart risk

Restarting Nebula could interrupt SSH access to the home server.

Resolution: Created and validated a rollback configuration, scheduled a temporary systemd rollback timer, restarted Nebula, reconnected through SSH, and canceled the timer.

### Protected configuration directory

A wildcard-based file listing failed because the unprivileged shell could not expand the wildcard inside `/etc/nebula`.

Resolution:

```bash
sudo ls -lah /etc/nebula/
```

### Outstanding work

- Enroll `ryzen-1` and any additional workers.
- Verify worker-to-manager and worker-to-worker communication.
- Deploy and secure the private Docker registry.
- Deploy Docker Swarm Visualizer.
- Test multi-node service scheduling.
- Record deployment outputs and troubleshooting results.
