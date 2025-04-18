---
title: "PXE Booting Talos with Matchbox"
---

I have recently been attempting (again) to create a test kubernetes cluster using 4 thin clients (Fujitsu Futro S920/S930) and the ones I got off eBay had the slight issue that they suddenly weren't able to boot off the local SSDs anymore. So, instead of doing the easy thing and using USB drives to boot instead, I decided to finally try to get PXE booting to work (the OS can still access the drives, oddly enough).

My network setup is as follows:

 - A Unifi Dream Machine acts as DHCP server and general network hub.
 - A Synology NAS to host the TFTP server and various docker containers.
 - A single S920 to act as the Kubernetes control plane.
 - 3 S930s to act as workers.

I will be using Talos to create the cluster and the official documentation mentioned a tool called Matchbox ([https://matchbox.psdn.io/](https://matchbox.psdn.io/)) for iPXE booting which renders iPXE configs. So now I need iPXE as well, which needs initial configuration because it should point to the Matchbox server to fetch the actual config. So the boot chain should be as follows:

  1. Device boots and asks the DHCP server for IP address, gateway and other options (including PXE booting options).
  2. DHCP server tells device to go to TFTP server on Synology NAS to fetch the iPXE boot file.
  3. Device loads iPXE boot file and asks the Matchbox server for the configuration.
  4. The Matchbox server returns the iPXE config and serves the files used to boot the devices.
  5. The device loads the OS and boots into the OS.

So first we need to create our own iPXE executable, which is loaded by the device and used to load the full OS. My main PC is running Windows 11, so I used Ubuntu WSL to build iPXE. First we need to install the build dependencies for iPXE which for Ubuntu 24.04 are as follows (info source: [https://ipxe.org/download](https://ipxe.org/download) and [https://github.com/ipxe/ipxe/issues/189#issuecomment-752045474](https://github.com/ipxe/ipxe/issues/189#issuecomment-752045474)):

```bash
sudo apt install gcc binutils make perl mtools liblzma-dev
```

Then we need to clone the source code from [https://github.com/ipxe/ipxe](https://github.com/ipxe/ipxe) (I use a dev drive so I cloned mine to D:\source\ipxe) and create a new file in the src folder (there might be better places, but this is easy to reference from the `make` command we will call later). I will be getting the iPXE executable to fetch the config from my NAS, so I will call the file nas-boot.ipxe and fill it out with the following content:

```bash
#!ipxe

dhcp
chain http://10.10.12.2:8080/boot.ipxe
```

Here the address is the iPXE endpoint of Matchbox, which will be hosted on my NAS (shown later).

I launched Ubuntu WSL, went to the src directory inside the repo we just cloned in WSL (`/mnt/d/source/ipxe/src`) and ran the following command:

```bash
make bin/undionly.kpxe EMBED=nas-boot.ipxe DEBUG=httpcore
```

While I waited for this to compile (it takes quite a while the first time), I went to my NAS and enabled the TFTP service. First I created a shared folder called `tftp`, then enabled the TFTP service by going to `Control Panel` -> `File Services` -> `Advanced` -> `TFTP`, checking the `Enable TFTP service` box, selecting the `tftp` folder I just created and then clicking `Apply`.

Once I had built iPXE, I went to the `D:\source\ipxe\src\bin` folder and copied the file `undionly.kpxe` to the `tftp` folder on the NAS.

Next step was to set up the Matchbox server which will deliver the configs and the OS files. I will be hosting mine using Docker. Synology thankfully has a nice manager for organising Docker projects called the `Container Manager`. First I created a folder for the files related to this Matchbox instance called `matchbox` and added two folders inside this, `varlib` and `etc`. Then I went to `Container Manager` -> `Projects` -> `Create`, selected a name for the project (just matchbox in my case), selected the `matchbox` folder I just created and selected `Create docker-compose.yml` for the source. I then entered the following config, skipped the creation of an entry in the Web Station, ensured that the project is started once it has been created and clicked `Done`. 

```yaml
version: "3.8"
services:
  matchbox:
    image: quay.io/poseidon/matchbox:v0.11.0
    container_name: matchbox
    volumes:
      - ./varlib:/var/lib/matchbox:Z
      - ./etc:/etc/matchbox:Z,ro
    command:
      - -address=0.0.0.0:8080
      - -log-level=debug
    restart: "no"
    ports:
      - "8080:8080"
      - "8081:8081"

```

For this next section I roughly followed the Talos docs ([https://www.talos.dev/v1.9/talos-guides/install/bare-metal-platforms/matchbox/](https://www.talos.dev/v1.9/talos-guides/install/bare-metal-platforms/matchbox/)), but found them rather confusing regarding the setup of Matchbox (there are some things which you really need to watch out for) so I am detailing my steps below.

First I created the talos config files for the servers using talosctl. This can be installed quite easily with winget using `winget install talosctl`. Then I ran `talosctl gen config homecluster https://10.10.12.125:6443` in a folder locally (it doesn't matter where, as long as your user can write to it) to generate the configs. Here, `homecluster` refers to the name of the Kubernetes cluster and `https://10.10.12.125:6443` refers to the Kubernetes endpoint of the control-plane node. I locked the IP addresses of my thin clients using the Unifi UI so they will always be given the same IP address when they ask for an IP address (essential in this case because the worker nodes always need to know which control-plane node to talk to).

I didn't need to change which drive would be used to install Talos because `/dev/sda` (the default) works well for my setup but your mileage may vary (insgtructions can be found on the Talos site, [https://www.talos.dev/v1.9/introduction/getting-started/#modifying-the-machine-configs](https://www.talos.dev/v1.9/introduction/getting-started/#modifying-the-machine-configs) contains some good instructions).

I also downloaded the latest Talos binaries from the latest Talos GitHub release ([https://github.com/siderolabs/talos/releases/tag/v1.9.5](https://github.com/siderolabs/talos/releases/tag/v1.9.5) in my case), we want `vmlinuz-amd64` and `initramfs-amd64.xz` (`amd64` is the architecture of the thin-clients). Then we want to remove the `-amd64` bits of the names.

Then I went back to folder for the Matchbox server (`matchbox`) and added the following file/folder structure to the `varlib` folder.

```
- assets
  - controlplane.yaml (generated by talosctl)
  - worker.yaml (generated by talosctl)
  - talosconfig (generated by talosctl)
  - initramfs.xz (downloaded from the GitHub release)
  - vmlinuz (downloaded from the GitHub release)
- groups
- profiles
```

Finally, we want to create the Matchbox groups and profiles which tell Matchbox what to render for each client. A group tells Matchbox which profile to deliver based on attributes of the device and the profile defines what iPXE config to render. Important to note here are the names of the files, it is important that the filename of the profile matches the profile name in the group (this tripped me up for a long time).

First, in the `groups` folder, we want to create two files, one for the control-plane node and one as a default (fallback) for the worker nodes.

_control-plane.json_

```json
{
  "id": "control-plane",
  "name": "control-plane",
  "profile": "control-plane", // This needs to match the filename without the filename suffix of the profile
  "selector": {
    "mac": "4c:52:62:08:a3:ce" // This is the MAC address of the control-plane thin client
  }
}
```

_default.json_
```json
{
  "name": "default",
  "profile": "default"
}
``` 

And now in the `profiles` folder we create the following two files:

_control-plane.json_
```json
{
  "id": "control-plane",
  "name": "control-plane",
  "boot": {
    "kernel": "/assets/vmlinuz",
    "initrd": ["/assets/initramfs.xz"],
    "args": [
      "initrd=initramfs.xz",
      "init_on_alloc=1",
      "slab_nomerge",
      "pti=on",
      "console=tty0",
      "printk.devkmsg=on",
      "talos.platform=metal",
      "talos.config=http://10.10.12.2:8080/assets/controlplane.yaml" // This IP address and port combo points to Matchbox
    ]
  }
}
```

_default.json_
```json
{
  "id": "default",
  "name": "default",
  "boot": {
    "kernel": "/assets/vmlinuz",
    "initrd": ["/assets/initramfs.xz"],
    "args": [
      "initrd=initramfs.xz",
      "init_on_alloc=1",
      "slab_nomerge",
      "pti=on",
      "console=tty0",
      "printk.devkmsg=on",
      "talos.platform=metal",
      "talos.config=http://10.10.12.2:8080/assets/worker.yaml"
    ]
  }
}
```

Finally, we need to set the correct DHCP options in Unifi to ensure that the devices start the boot chain. This is done by going to the Unifi network settings, selecting the network which connects NAS and thin clients, scrolling down to `DHCP Service Management`. Here we need to set the first field of the `Network Boot` option to the IP of our NAS (10.10.12.2) and the second to the iPXE filename which we compiled earlier `/undionly.kpxe`. Then we need to set the TFTP server also to our NAS IP `10.10.12.2` and then `Apply Changes`.

From now, if the thin clients are set to boot from the network, they should automatically pick up the correct config and boot.
