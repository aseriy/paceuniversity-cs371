# Operating System

## Goal
A Lenovo ThinkCentre M715q, booted from the bootstrap USB on campus Ethernet, installs Ubuntu 24.04.5 Server unattended onto its SSD and comes up reachable over SSH with the hostname `cs371-<last 6 hex digits of its Ethernet MAC>`.

## Prerequisites
- Campus Ethernet with DHCP and outbound HTTPS.
- The flashed `cs371-bootstrap.img` USB stick. Physical and firmware steps (USB boot order, Secure Boot) are covered on the [Hardware](hardware.md) page.

## Design decisions
- **The USB stick carries only a minimal embedded iPXE script** (`provisioning/usb.ipxe`) that chains to the GitHub-hosted `provisioning/bootstrap.ipxe`. Boot logic can be revised in this repository without reflashing any USB sticks.
- **Ubuntu autoinstall with a NoCloud seed served from this repository.** `bootstrap.ipxe` boots the Ubuntu 24.04.5 live-server installer over the network and points it at the raw GitHub URL of `provisioning/autoinstall/`, so the entire install is unattended and reproducible from this repository alone.
- **SSH key-only access.** The installer creates the `admin` user with a locked password and `allow-pw: false`; the only way in is the authorized SSH key.
- **MAC-derived hostname.** Each machine names itself `cs371-` plus the last 6 hex digits of its Ethernet MAC, guaranteeing unique, predictable hostnames with no per-machine configuration.

## Installation and configuration

Building and flashing the boot USB is covered on the [Hardware](hardware.md) page.

### Boot flow
```
USB → iPXE → bootstrap.ipxe (GitHub) → Ubuntu installer → Autoinstall
```

### Provisioning files
```
provisioning/
├── usb.ipxe
├── bootstrap.ipxe
└── autoinstall/
    ├── user-data
    └── meta-data
```

- `usb.ipxe` — embedded in the USB image. Obtains a DHCP lease and chains to the current `bootstrap.ipxe` on GitHub.
- `bootstrap.ipxe` — downloads the Ubuntu 24.04.5 netboot kernel and initrd from `https://releases.ubuntu.com/24.04.5`, boots the live-server ISO over HTTPS, and passes `autoinstall ds=nocloud-net` with the seed URL pointing at this repository's `provisioning/autoinstall/`. The `set semicolon:hex 3b` line embeds the literal `;` that the NoCloud datasource syntax requires in the kernel command line.
- `autoinstall/user-data` — the autoinstall configuration:
    - creates the `milkalmond` user with a locked password (`"!"`);
    - installs the SSH server with password login disabled and public-key authentication enabled;
    - installs Ubuntu directly to the SSD (`storage: layout: name: direct`);
    - installs `git` and `curl`;
    - in `late-commands`, derives the hostname from the default-route interface's MAC address and writes it into `/target/etc/hostname` and `/target/etc/hosts`;
    - creates `/target/etc/issue.d/ip.issue` so the current IPv4 address is displayed on the console login screen.
- `autoinstall/meta-data` — empty, but must exist for the NoCloud datasource to accept the seed.

## Verification

Our team completed a full provisioning test on the Lenovo Ryzen hardware.

1. The machine successfully booted from the USB, iPXE obtained a DHCP lease, and fetched `bootstrap.ipxe` from GitHub.
2. The installer downloaded the Ubuntu 24.04.5 kernel, initrd, and live-server ISO from `releases.ubuntu.com` and completed the autoinstall.
3. The machine rebooted successfully as `cs371-bc9b0c`, generated from the last six hexadecimal digits of the Ethernet MAC address `6c:4b:90:bc:9b:0c`.
4. Ubuntu 24.04.5 rejected the originally configured `admin` username because it is reserved by the system. Our team changed the autoinstall username to `milkalmond`, which allowed provisioning to complete successfully.
5. SSH public-key authentication to `milkalmond@cs371-bc9b0c` was successfully verified from a separate laptop.
6. Password-based SSH authentication was tested separately and correctly returned `Permission denied (publickey)`.
7. The console IP display was successfully tested using `/etc/issue.d/ip.issue`. After DHCP completed, the login screen displayed the machine's current IPv4 address, allowing the SSH destination to be identified without checking the DHCP server or router.

## Troubleshooting
Problems encountered and how we resolved them.
