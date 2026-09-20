# Hardware

## Goal
A bare Lenovo ThinkCentre M715q, plugged into the university campus network and booted from the bootstrap USB stick, reaches the point where remote-config provisioning takes over. The unattended install itself is covered on the [Operating System](operating-system.md) page.

## Prerequisites
- A campus network drop (RJ45).
- From university IT: DHCP and outbound HTTPS on that drop.
- A USB stick.
- A Linux machine to build the iPXE USB image.

## Design decisions
- **The process is codified and repeatable.** Nothing in it is machine-specific — hostnames, credentials, and configuration all derive from the remote config in this repository — so it scales to any number of boxes.
- **Commodity small-form-factor hardware.** The Lenovo ThinkCentre M715q is inexpensive, quiet, and fits campus desk space.
- **The USB stick is the only physical bootstrap medium.** It carries a minimal embedded script that chains to this repository; see the [Operating System](operating-system.md) page for why the boot logic lives in the repository rather than on the stick.

## The hardware
Lenovo ThinkCentre M715q.

Specifications (CPU, RAM, SSD, NIC) — to be researched and documented.

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

### Connecting to the campus network
Plug the machine into a campus network drop via RJ45. The drop must provide DHCP and allow outbound HTTPS.

What to request from IT, and how — to be researched and documented.

### Booting from the USB
Insert the USB stick and boot the machine from it (boot order or one-time boot menu). Disable Secure Boot if necessary.

Document the exact M715q firmware steps (boot menu key, boot order settings, Secure Boot) — to be researched and documented after the first boot test.

## Verification
The first boot test on the Lenovo hardware is still pending. Expected results:

1. The machine powers on and boots from the USB stick.
2. iPXE obtains a DHCP lease and fetches `bootstrap.ipxe` from GitHub.
3. The Ubuntu installer starts — from here the [Operating System](operating-system.md) page takes over.

## Troubleshooting
Problems encountered and how we resolved them.
