# Multiple AIO instances
It is possible to run multiple instances of AIO on one server.

There are two ways to achieve this: The normal way is creating multiple VMs, installing AIO in [reverse proxy mode](./reverse-proxy.md) in each of them and having one reverse proxy in front of them that points to each VM (you also need to [use a different `TALK_PORT`](https://github.com/nextcloud/all-in-one#how-to-adjust-the-talk-port) for each of them). The second and more advanced way is creating multiple users on the server and using docker rootless for each of them in order to install multiple instances on the same server. 


## Run multiple AIO instances on the same server with docker rootless
1. Create as many linux users as you need first. The easiest way is to use `sudo adduser` and follow the setup for that. Make sure to create a strong unique password for each of them and write it down!
1. Log in as each of the users by opening a new SSH connection as the user and install docker rootless for each of them by following step 0-1 and 3-4 of the [docker rootless documentation](./docker-rootless.md) (you can skip step 2 in this case).
1. Then install AIO in reverse proxy mode by using the command that is described in step 2 and 3 of the [reverse proxy documentation](./reverse-proxy.md) but use a different `APACHE_PORT` and [`TALK_PORT`](https://github.com/nextcloud/all-in-one#how-to-adjust-the-talk-port) for each instance as otherwise it will bug out. Also make sure to adjust the docker socket and `WATCHTOWER_DOCKER_SOCKET_PATH` correctly for each of them by following step 6 of the [docker rootless documentation](./docker-rootless.md). Additionally, modify `--publish 8080:8080` to a different port for each container, e.g. `8081:8080` as otherwise it will not work.<br>
**⚠️ Please note:** If you want to adjust the `NEXTCLOUD_DATADIR`, make sure to apply the correct permissions to the chosen path as documented at the bottom of the [docker rootless documentation](./docker-rootless.md). Also for the built-in backup to work, the target path needs to have the correct permissions as documented there, too.
1. Now install your webserver of choice on the host system. It is recommended to use caddy for this as it is by far the easiest solution. You can do so by following https://caddyserver.com/docs/install#debian-ubuntu-raspbian or below. (It needs to be installed directly on the host or on a different server in the same network).
1. Next create your Caddyfile with multiple entries and domains for the different instances like described in step 1 of the [reverse proxy documentation](./reverse-proxy.md). Obviously each domain needs to point correctly to the chosen `APACHE_PORT` that you've configured before. Then start Caddy which should automatically get the needed certificates for you if your domains are configured correctly and ports 80 and 443 are forwarded to your server.
1. Now open each of the AIO interfaces by opening `https://ip.address.of.this.server:8080` or e.g. `https://ip.address.of.this.server:8081` or as chosen during step 3 of this documentation. 
1. Finally type in the domain that you've configured for each of the instances during step 5 of this documentation and you are done.
1. Please also do not forget to open/forward each chosen `TALK_PORT` UDP and TCP in your firewall/router as otherwise Talk will not work correctly!

Now everything should be set up correctly and you should have created multiple working instances of AIO on the same server!


## Run multiple AIO instances on the same server inside their own virtual machines
This guide will walk you through creating and configuring two Debian-based VMs (with "reverse proxy mode" Nextcloud AIO installed in each VM), behind one Caddy reverse proxy, all running on one host physical machine (like a laptop or desktop PC). It's highly recommend to follow the steps in order. Steps 1 through 4 will need to be repeated. Steps 5 through 8 only need to be completed once. All commands are expected to be run as root.

<details><summary><strong>PLEASE READ: A few expectations about your network</strong></summary>
This guide assumes that you have forwarded ports 443 and 8443 to your host physical machine via your router's configuration page, and either set up Dynamic DNS or obtained a static outbound IP address from your ISP. If this is not the case, or if you are brand-new to networking, you probably should not proceed with this guide, unless you are just using it for educational purposes. Proper network setup and security is critical when it comes to keeping your data safe. You may consider hosting using a VPS instead, or choosing one of <a href="https://nextcloud.com/providers/">Nextcloud's trusted providers.</a>
</details>

<details><summary><strong>A note for VPS users</strong></summary>
If you want to do this on a VPS, and your VPS is KVM-based and provides a static IP address, you can likely benefit from this guide too! Simply replace the words "host physical machine" with "VPS" and follow along. Good luck!
</details>

**Before starting:** Make sure your host physical machine has enough resources. A host machine with 8GB RAM and 100GB storage is sufficient for running two fairly minimal VMs, with 2GB RAM and 32GB storage allocated to each VM. This guide assumes you have these resources at the minimum. This is fine for just testing the setup, but you will probably want to allocate more resources to your VMs if you plan to use this for day-to-day use.
If your host machine has more than 8GB memory available, and you plan to enable any of the optional containers (Nextcloud Office, Talk, Imaginary, etc.) in any of your instances, then you should definitely allocate more memory to the VM hosting that instance. In other words, before turning on any extra features inside a particular AIO interface, make sure you've first allocated enough resources to the VM that the instance is running inside. If in doubt, the AIO interface itself gives great recommendations for extra CPU and RAM allocation.

**Additional prerequisites:** Your host physical machine needs to have virtualization enabled in it's UEFI/BIOS. It also needs a few tools installed in order to create VMs. Assuming your host machine is a bare-bones Ubuntu or Debian Linux server without a desktop environment installed, the easiest way to create VMs is to install *QEMU*, *virsh*, *virt-install*, and a few extra packages to support UEFI booting and network config ([more info](https://wiki.debian.org/KVM)). You only need to do this once. To do this, run one of these two commands (**on the host physical machine**):
```shell
# For host machines running Ubuntu Server:
apt install --no-install-recommends qemu-system libvirt-clients libvirt-daemon-system virtinst ovmf bridge-utils
```
```shell
# For host machines running Debian:
apt install --no-install-recommends qemu-system libvirt-clients libvirt-daemon-system virtinst ovmf bridge-utils dnsmasq-base
```

**Let's begin!** This guide assumes that you have two domains where you would like to host two individual AIO instances (one instance per domain). Let's call these domains `example1.com` and `example2.com`. Therefore, we'll create two VMs named `example1-com` and `example2-com` (These are the VM names we'll use below in step 1).

**Once you're ready, follow steps 1-4 below to set up your VMs. You will configure them one at a time.**

1. Choose a name for your VM. A good choice is to name each VM the same as the domain name that will be used to access it.
2. Choose the distribution you'd like to install within the VM. Here are a few options:
   <details><summary><strong>Ubuntu Server 22.04.4 LTS</strong></summary>
       <h4>Downloading the .iso image</h4>
       Ubuntu no longer hosts their "Legacy Ubuntu Server Installer" images, meaning we cannot pass a URL to virt-install to use as a location. Instead, you must first download an .iso image and pass the path for that image.
       <pre><code># Skip this part if you've already downloaded this image
   curl -o /tmp/ubuntu-22.04.4-live-server-amd64.iso https://releases.ubuntu.com/jammy/ubuntu-22.04.4-live-server-amd64.iso
   </code></pre>
       <em>Note: You may choose a different place to store the .iso file, but it needs to be somewhere accessable by QEMU. /tmp and /home should work fine, but choosing something like /root will cause virt-install to fail.</em>
       <h4>Creating the VM</h4>
       Now create the Ubuntu VM (Don't forget to replace [VM_NAME]):
       <pre><code>virt-install \
   --name [VM_NAME] \
   --virt-type kvm \
   --location /tmp/ubuntu-22.04.4-live-server-amd64.iso,kernel=casper/vmlinuz,initrd=casper/initrd \
   --os-variant ubuntujammy \
   --disk size=32 \
   --memory 2048 \
   --graphics none \
   --console pty,target_type=serial \
   --extra-args "console=ttyS0" \
   --autostart \
   --boot uefi
   </code></pre>
   </details>
   <details><summary><strong>Debian 11</strong></summary>
       <h4>Creating the VM</h4>
       Create the Debian VM (Don't forget to replace [VM_NAME]):
       <pre><code>virt-install \
   --name [VM_NAME] \
   --virt-type kvm \
   --location http://deb.debian.org/debian/dists/bullseye/main/installer-amd64/ \
   --os-variant debian11 \
   --disk size=32 \
   --memory 2048 \
   --graphics none \
   --console pty,target_type=serial \
   --extra-args "console=ttyS0" \
   --autostart \
   --boot uefi
   </code></pre>
   </details>
   <details><summary><strong>Debian 12</strong></summary>
       <h4>Creating the VM</h4>
       Create the Debian VM (Don't forget to replace [VM_NAME]):
       <pre><code># If the os-variant "debian12" is unavailable, try "debiantesting" instead
   virt-install \
   --name [VM_NAME] \
   --virt-type kvm \
   --location http://deb.debian.org/debian/dists/bookworm/main/installer-amd64/ \
   --os-variant debian12 \
   --disk size=32 \
   --memory 2048 \
   --graphics none \
   --console pty,target_type=serial \
   --extra-args "console=ttyS0" \
   --autostart \
   --boot uefi
   </code></pre>
   </details>
   For more options, or for help automating this process, check out <a href="https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/virtualization_deployment_and_administration_guide/sect-guest_virtual_machine_installation_overview-creating_guests_with_virt_install">this guide</a>.

<!-- TODO: Remove
run the following command (**on the host physical machine**). Don't forget to replace `[VM_NAME]`.
    ```shell
    virt-install --name [VM_NAME] --virt-type kvm --location http://deb.debian.org/debian/dists/bullseye/main/installer-amd64/ --os-variant debian11 --disk size=32 --memory 2048 --graphics none --console pty,target_type=serial --extra-args "console=ttyS0"
    ```
-->

3. Navigate through the text-based installer. Most options can remain as their default.
   #### For the Ubuntu Server installer
   When asked, you can set the computer's name to the same value as the name you gave to your VM (for example, `example1-com`). You can leave the HTTP proxy information blank. The installer will create a non-root user. Note down the password you use here! You can allow the partitioner to use the entire disk, this only uses the virtual disk that we defined above in step 1. You'll eventually be given the option to install additional software. Although "Nextcloud" is listed here, it is almost certainly not the option you want, since we are setting up Nextcloud AIO. [Read this](https://github.com/nextcloud/all-in-one?tab=readme-ov-file#how-does-it-work) if you don't understand the difference. You'll be asked about installing `SSH server`, this is optional (This lets you easily SSH into the VM in the future in case you have to perform any maintenance). Finally, disregard the `[FAILED] Failed unmounting /cdrom.` message, and press return.
   #### For the Debian installer
   When asked, you can set the hostname to the same value as the name you gave to your VM (for example, `example1-com`). You can leave the domain name and HTTP proxy information blank. Allow the installer to create both a root and a non-root user. Note down the password(s) you use here! You can allow the partitioner to use the entire disk, this only uses the virtual disk that we defined above in step 1. When *tasksel* (Software selection) runs and asks if you want to install additional software, use spacebar and your arrow keys to uncheck the `Debian desktop environment` and `GNOME` options, and check the `SSH server` option (This lets you easily SSH into the VM in the future in case you have to perform any maintenance). Make sure `standard system utilities` is also checked. Hit tab to select "Continue". Finally, disregard the warning about GRUB, allow it to install to your "primary drive" (again, it's only virtual, and this only applies to the VM- this will not affect the boot configuration of your host physical machine) and select `/dev/vda` for the bootable device.
5. Log in to your new VM. After it's finished installing, the VM will have rebooted and presented you with a login prompt. Use `root` as the username, and enter the password you chose during the installation process. Ubuntu restricts root account access, so you may need to login with the non-root user and run `sudo su -` to elevate your privileges. Then, run this command (**on the VM**):
   ```shell
   apt install -y curl && curl -fsSL https://get.docker.com | sh && docker run --init --sig-proxy=false --name nextcloud-aio-mastercontainer --restart always --publish 8080:8080 --env APACHE_PORT=11000 --env APACHE_IP_BINDING=0.0.0.0 --volume nextcloud_aio_mastercontainer:/mnt/docker-aio-config --volume /var/run/docker.sock:/var/run/docker.sock:ro nextcloud/all-in-one:latest
   ```
   This single command will install docker and AIO in reverse proxy mode! As with any other command, carefully read over it and try your best to understand it before running it. This may take a few minutes. When it's finished, you should see a success message, saying "Initial startup of Nextcloud All-in-One complete!". This concludes the setup needed on this particular VM.
   
   ---
6. Go ahead and run through steps 1-4 again in order to set up your second VM. When you're finished, proceed down to step 6. (Note: If you downloaded the Ubuntu .iso and no longer want it, you may delete it now.)
7. Almost done! All that's left is configuring our reverse proxy. To do this, we first need to install it. Run (**on the host physical machine**):
   ```shell
   apt update -y && apt install -y debian-keyring debian-archive-keyring apt-transport-https curl && curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg && curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | tee /etc/apt/sources.list.d/caddy-stable.list && apt update -y && apt install -y caddy
   ```
   This command will ensure that your system is up-to-date and install the latest stable version of Caddy via it's official binary source.
8. To configure Caddy, we need to know the IP address assigned to each VM. Run (**on the host physical machine**):
    ```shell
    virsh net-dhcp-leases default
    ```
    This will show you the VMs you set up, and the IP address corresponding to each of them. Note down each IP and corresponding hostname.
    Finally, we will configure Caddy using this information. Open the default Caddyfile with a text editor:
    ```shell
    nano /etc/caddy/Caddyfile
    ```
    Replace everything in this file with the following configuration. Don't forget to edit this sample configuration and substitute in your own domain names and IP addresses. `[DOMAIN_NAME_*]` should be a domain name like `example1.com`, and `[IP_ADDRESS_*]` should be a local IPv4 address like `192.168.122.225`.
    ```shell
    # Virtual machine #1 - "example1-com"
    https://[DOMAIN_NAME_1]:8443 {
        reverse_proxy https://[IP_ADDRESS_1]:8080 {
            transport http {
                tls_insecure_skip_verify
            }
        }
    }
    https://[DOMAIN_NAME_1]:443 {
        reverse_proxy [IP_ADDRESS_1]:11000
    }
    
    # Virtual machine #2 - "example2-com"
    https://[DOMAIN_NAME_2]:8443 {
        reverse_proxy https://[IP_ADDRESS_2]:8080 {
            transport http {
                tls_insecure_skip_verify
            }
        }
    }
    https://[DOMAIN_NAME_2]:443 {
        reverse_proxy [IP_ADDRESS_2]:11000
    }

    # (Add more configurations here if you set up more than two VMs!)
    ```
    After making this change, you'll need to restart Caddy:
   ```shell
   systemctl restart caddy
   ```
9. That's it! Now, all that's left is to set up your instances through the AIO interface as usual by visiting `https://example1.com:8443` and `https://example2.com:8443` in a browser. Once you're finished going through each setup, you can access your new instances simply through their domain names. You can host as many instances with as many domain names as you want this way, as long as you have enough system resources. Enjoy!
    
    <details><summary>A few extra tips for managing this setup!</summary>
        <ul>
            <li>You can SSH into a VM to perform maintenance using this command (<strong>on the host physical machine</strong>): <pre><code>ssh [NONROOT_USER]@[IP_ADDRESS] # By default openssh does not allow logins using the root account</code></pre></li>
            <li>If you mess up the configuration of a VM, you may wish to completely delete it and start fresh with a new one. <strong>THIS WILL DELETE ALL DATA ASSOCIATED WITH THE VM INCLUDING ANYTHING IN YOUR AIO DATADIR!</strong> To do this, run (<strong>on the host physical machine</strong>): <pre><code># BE EXTREMELY CAREFUL! You'll need to confirm this action (y/N)
   virsh destroy --domain [VM_NAME] \
   virsh undefine --nvram --domain [VM_NAME] && rm -rf /var/lib/libvirt/images/[VM_NAME].qcow2</code></pre></li>
            <li>Using Nextcloud Talk will require some extra configuration. Back when we set up our VMs, they were (by default) configured with NAT, meaning they are in their own subnet. My understanding is that the VMs must each instead be bridged, so that your router may directly "see" them (as if they were real, physical devices on your network), and each AIO instance inside each VM must be configured with a different Talk port (like 3478, 3479, etc.). To change the Talk port for a given instance, you must either have created it that way initially (back when you first configured the VM in step 3 above), or you can change it at any time by removing the mastercontainer and re-running the initial docker run command (but modifying the run command <a href="https://github.com/nextcloud/all-in-one?tab=readme-ov-file#how-to-adjust-the-talk-port">like so</a>). Then, the Talk port for EACH instance needs to be forwarded in your router's settings DIRECTLY to the VM hosting the instance (completely bypassing our host physical machine/reverse proxy). And finally, inside an admin-priveleged account (such as the default "admin" account) in each instance, you must visit <strong>https://[DOMAIN_NAME_*]/settings/admin/talk</strong> then find the STUN/TURN Settings, and from there set the proper values. If this is too complicated, it may be easier to use public STUN/TURN servers, but I have not tested any of this, rather I'm just sharing what I have found so far (more info available <a href="https://github.com/nextcloud/all-in-one/discussions/2517">here</a>). If you have figured this out or if any of this information is incorrect, please edit this section!</li>
            <li>If you want to shave off around 8-10 seconds of total boot time when you reboot your host physical machine, a simple trick is to lower the GRUB_TIMEOUT from the default five seconds to one second, on both the host physical machine and each of the VMs. You can also remove the delay, but it's generally safer to leave at least one second. (Always be extremely careful when editing GRUB config, especially on the host physical machine, as an incorrect configuration can prevent your device from booting!)</li>
        </ul>
    </details>