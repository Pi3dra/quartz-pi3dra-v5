# Why Home‑Lab at All?

#WIP

I decided to start **home labbing** for a mix of practical and privacy reasons.

On the practical side, I want to learn more about **networking, servers, virtualization, and system administration** by actually running things myself instead of just reading about them. There’s no better teacher than breaking your own setup and fixing it at 2 a.m.

On the privacy side, we’re living in a slightly dystopian era where *everyone and their mum* is tracking your data.
where it lives, how you use it, and how long it’s kept. I want more control over my data and media, even if that control comes with responsibility and pain.

This journey was heavily inspired by **Louis Rossmann**, an independent electronics technician, YouTuber, and consumer‑rights activist.

* [Louis Rossmann on YouTube](https://www.youtube.com/channel/UCl2mFZoRqjw_ELax4Yisf6w)
* [FUTO – Introduction to a Self‑Managed Life](https://wiki.futo.org/index.php/Introduction_to_a_Self_Managed_Life:_a_13_hour_%26_28_minute_presentation_by_FUTO_software)

While his guides are excellent, they often assume **multiple machines and a comfortable budget**. I don’t have that, so this lab is very much a *single box, budget constrained, learning first* experiment.

---

## Software Stack

The software I plan to use (some decisions still pending):

* **Proxmox VE** – Hypervisor
* **TrueNAS** – Network‑attached storage
* **Immich** – Photo gallery and image backup (Google Photos replacement)
* **Document collaboration suite** – Google Docs replacement (TBD)
* **pfSense** – Router and firewall, mainly to provide VPN access without port forwarding


## Hardware

Everything runs on a single repurposed machine:

* HP Z230 SFF workstation
* 4 × 500 GB 2.5" HDDs (This was really cheap, and i don’t need a lot of storage… yet)
* 1 × 250 GB SSD (boot / fast storage)
* NVIDIA GT 1030 GPU (mostly unnecessary, but I already had it)
* 16GB of RAM (this is very tight, and will be upgraded soon-ish)

Yes, everything on one box is a terrible idea from a redundancy perspective, but this is a **learning lab**, not production infrastructure.

---

## The Good

* Perfect for experimenting and learning
* Full control over software, versions, and configuration
* No third‑party restrictions
* Long‑term cost savings
* Better privacy
* *Potentially* better security, though I’m not arrogant enough to claim I can host services better than Google (and probably never will)

---

## The Bad & the Ugly

* **Single point of failure**
  this machine dies → storage, services, and routing die with it. This is both inconvenient and a security concern.

* **Power & cooling**
  Even with 2.5" drives, running services for family and friends increases power usage and heat.

* **Networking complexity**
  The workstation has only one physical NIC. I’ll either need:
  * an additional network adapter, or
  * VLAN black magic

* **Maintenance & updates**
  OS updates, Proxmox upgrades, security patches, backups all on me.


## Disclaimer

I take **no responsibility** if you follow this path and break your network, lose your data, or anger your household (I already did).

Self‑hosting is not for the faint of heart, and this is my **first ever home lab**. Everything here is educational and most importantly **experimental**.

These notes are mostly a reminder for **future me**, in case I mess things up so badly that even backups can’t save me.

# Local‑First, Then Secure Remote Access

At the time of writing, I am running everything locally.
Meanwhile I haven't setted up VPN acces yet, so before exposing *anything* to the outside world, my approach will be:

1. **Run everything locally first**
   * Services are tested only on the LAN
   * No public exposure
   * No port forwarding
   * Break things safely

2. **Once stable, add VPN access**
   * Remote access only through a VPN
   * No services directly exposed to the internet
   * No port forwarding to individual services

### Why VPN Instead of Port Forwarding?

Port forwarding:

* Increases attack surface
* Requires securing each exposed service individually
* Is easy to misconfigure

A VPN:

* Exposes **one hardened entry point**
* Gives remote devices the same access as if they were on the LAN
* Simplifies firewall rules
* Scales better as more services are added

pfSense will be used to provide **OpenVPN or WireGuard**, allowing secure access from anywhere without publicly exposing internal services.


# Installing Proxmox

Installing Proxmox is as straightforward as installing any other OS:

1. Download the ISO from the [Proxmox website](https://www.proxmox.com/en/)
2. Flash it to a USB stick (BalenaEtcher, Ventoy, `dd`, etc.)
3. Boot the machine and follow the installer

## Fixing a Bad IP Choice

If, like me, you picked a static IP that was already in use:

```bash
nano /etc/network/interfaces
systemctl restart networking
```

## Updates & Repositories

* Disable **enterprise repositories**
* Add the **no‑subscription repository**
* Skip Ceph (overkill for a single‑node lab)

Navigate to **updates** and click on:
* Refresh
* Upgrade

## Notifications 

Navigate to:

`Datacenter → Notifications`

I use Gmail via SMTP:
* Server: `smtp.gmail.com`
* Encryption: StartTLS

Create an **App Password**:
[https://myaccount.google.com/apppasswords](https://myaccount.google.com/apppasswords)

* From address = Gmail address
* Recipient: `root@pam`

Test the setup, then configure **Notification Matchers** to send everything to email and disable `mail-to-root`.

## PCI Passthrough

Documentation:
[https://pve.proxmox.com/wiki/PCI_Passthrough](https://pve.proxmox.com/wiki/PCI_Passthrough)

Used for passing GPUs or NICs directly to VMs.

When using PCI passthrough:

* VM → `System → Machine` → set to `q35`


# Backups & TrueNAS

I installed TrueNAS **before** any serious backup configuration.

Since everything lives on one machine, backups mainly protect against:

* User error
* Broken updates
* Misbehaving VMs

They do **not** protect against:

* Hardware failure
* Fire
* Theft

It is also interesting to know the 3‑2‑1 Backup Rule
* 3 total copies of data
* 2 local copies on different devices
* 1 offsite copy

I don't think I will completely respect this, mainly for budget reasons, but it is something to keep in mind and aim for.

## Installing TrueNAS

I followed this guide:
[https://www.youtube.com/watch?v=M3pKprTdNqQ](https://www.youtube.com/watch?v=M3pKprTdNqQ)


IMPORTANT: Set DNS under `Network → Global Configuration`, to properly have network access.


## Intended Use

TrueNAS will be used **only as storage**, not for running services.

Planned datasets:

* Documents (Google Drive‑like)
* Photos / media
* Backups

## Notifications

* Add your email under `Credentials → Users`
* Configure alerts under `Settings → Email`
* I used **Google OAuth**, but SMTP also works


# Installing Immich

Proxmox helper script:
[https://community-scripts.github.io/ProxmoxVE/scripts?id=immich&category=Media+%26+Streaming](https://community-scripts.github.io/ProxmoxVE/scripts?id=immich&category=Media+%26+Streaming)
Immich will act as my Google Photos replacement and image backup solution.

Currently I am triying to learn on how to resize the memory size used by Immich, in case I understimate my needs!

---

## TODO / Future Work
* Document permissions, shares, and groups in TrueNAS
- Explain SMART monitoring for hard drives with TrueNAS
* Decide on Google Docs replacement
* Proper offsite or local backups?
* VLAN design? and VPN setup (I need to buy a PCIe Network Interface Card)
