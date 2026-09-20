# Operating System

## Goal
A Lenovo ThinkCentre M715q, booted from the bootstrap USB on campus Ethernet, installs Ubuntu 24.04.5 Server unattended onto its SSD and comes up reachable over SSH with the hostname `cs371-<last 6 hex digits of its Ethernet MAC>`.

## Prerequisites
- Campus Ethernet with DHCP and outbound HTTPS.
- The flashed `cs371-bootstrap.img` USB stick. Physical and firmware steps (USB boot order, Secure Boot) are covered on the [Hardware](hardware.md) page.
- A Linux machine to build the iPXE USB image.

## Design decisions
- **The USB stick carries only a minimal embedded iPXE script** (`provisioning/usb.ipxe`) that chains to the GitHub-hosted `provisioning/bootstrap.ipxe`. Boot logic can be revised in this repository without reflashing any USB sticks.
- **Ubuntu autoinstall with a NoCloud seed served from this repository.** `bootstrap.ipxe` boots the Ubuntu 24.04.5 live-server installer over the network and points it at the raw GitHub URL of `provisioning/autoinstall/`, so the entire install is unattended and reproducible from this repository alone.
- **SSH key-only access.** The installer creates the `admin` user with a locked password and `allow-pw: false`; the only way in is the authorized SSH key.
- **MAC-derived hostname.** Each machine names itself `cs371-` plus the last 6 hex digits of its Ethernet MAC, guaranteeing unique, predictable hostnames with no per-machine configuration.

## Installation and configuration

### Building the boot USB
On a Linux machine, install the build dependencies and build iPXE with `usb.ipxe` embedded:

```bash
sudo apt install syslinux-common mtools syslinux
git clone https://github.com/ipxe/ipxe.git
cd ipxe/src
make bin-x86_64-efi/ipxe.usb EMBED=<path to paceuniversity-cs371>/provisioning/usb.ipxe
```

Rename the resulting `bin-x86_64-efi/ipxe.usb` to `cs371-bootstrap.img` for easier tool compatibility, then flash it to a USB stick:

- Windows: Rufus or balenaEtcher
- Linux: `dd`

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
    - creates the `admin` user with a locked password (`"!"`);
    - installs the SSH server with password login disabled and the authorized key from the `authorized-keys` list (currently the `ssh-ed25519 YOUR_PUBLIC_KEY` placeholder);
    - installs Ubuntu directly to the SSD (`storage: layout: name: direct`);
    - installs `git` and `curl`;
    - in `late-commands`, derives the hostname from the default-route interface's MAC address and writes it into `/target/etc/hostname` and `/target/etc/hosts`.
- `autoinstall/meta-data` — empty, but must exist for the NoCloud datasource to accept the seed.

## Verification
The first full boot test on the Lenovo hardware is still pending. Expected results:

1. The machine boots from the USB, iPXE obtains DHCP, and fetches `bootstrap.ipxe` from GitHub.
2. The installer downloads the kernel, initrd, and ISO from `releases.ubuntu.com` and runs the autoinstall to completion with no prompts.
3. The machine reboots as `cs371-<last 6 hex digits of its MAC>`.
4. SSH as `admin` with the authorized key succeeds; password login is refused.

## Troubleshooting
Problems encountered and how we resolved them.
