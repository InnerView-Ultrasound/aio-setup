# Multiple AIO instances
It is possible to run multiple instances of AIO on one server.

There are two ways to achieve this: The normal way is creating multiple VMs, installing AIO in [reverse proxy mode](./reverse-proxy.md) in each of them and having one reverse proxy in front of them that points to each VM (you also need to [use a different `TALK_PORT`](https://github.com/nextcloud/all-in-one#how-to-adjust-the-talk-port) for each of them). The second and more advanced way is creating multiple users on the server and using docker rootless for each of them in order to install multiple instances on the same server. 

## Run multiple AIO instances on the same server inside their own virtual machines
This guide will walk you through creating and configuring two Debian VMs (with "reverse proxy mode" Nextcloud AIO installed in each VM), behind one Caddy reverse proxy, all running on one physical host machine (like a laptop or desktop PC). It's highly recommend to follow the steps in order. Steps 1 through 3 will need to be repeated. Steps 4 through 7 only need to be completed once.

<details><summary><strong>PLEASE READ: A few expectations about your network</strong></summary>
<strong>This guide assumes that you have <a href="https://github.com/nextcloud/all-in-one?tab=readme-ov-file#which-ports-are-mandatory-to-be-open-in-your-firewallrouter">already forwarded all necessary ports</a> via your router's configuration page, and either set up Dynamic DNS or obtained a static outbound IP address from your ISP. If this is not the case, or if you are brand-new to networking, you probably should not proceed with this guide, unless you are just using it for educational purposes. Proper network setup and security is critical when it comes to keeping your data safe. You may consider hosting using a VPS instead, or choosing one of <a href="https://nextcloud.com/providers/">Nextcloud's trusted providers.</a></strong>
</details>

<details><summary><strong>A note for VPS users</strong></summary>
If you want to do this on a VPS, and your VPS is KVM-based and provides a static IP address, you can likely benefit from this guide too! Simply replace the words "physical host machine" with "VPS" and follow along. Good luck!
</details>

**Before starting:** Make sure your physical host machine has enough resources. A host machine with 8GB RAM and 100GB storage is sufficent for running two fairly minimal VMs, with 2GB RAM and 32GB storage allocated to each VM. This guide assumes you have these resources at the minimum. This is fine for just testing the setup, but you will probably want to allocate more resources to your VMs if you plan to use this for day-to-day use.
If your host machine has more than 8GB memory available, and you plan to enable any of the optional containers (Nextcloud Office, Talk, Imaginary, etc.) in any of your instances, then you should definitely allocate more memory to the VM hosting that instance. In other words, before turning on any extra features inside a particular AIO interface, make sure you've first allocated enough resources to the VM that the instance is running inside. If in doubt, the AIO interface itself gives great recommendations for extra CPU and RAM allocation.

**Additional prerequisites:** Your physical host machine needs to have virtualization enabled in it's UEFI/BIOS. It also needs a few tools installed in order to create VMs. Assuming your host machine is a barebones Linux server without a desktop environment installed, the easiest way to create VMs is to install *QEMU*, *virsh*, and *virt-install* ([more info](https://wiki.debian.org/KVM)). You only need to do this once. To do this on Debian, run this command (**on the physical host machine**):
```shell
apt install --no-install-recommends qemu-system libvirt-clients libvirt-daemon-system virtinst
```

**Let's begin!** This guide assumes that you have two domains where you would like to host two individual AIO instances. Let's call these domains `example1.com` and `example2.com`. Therefore, we'll create two VMs named `example1-com` and `example2-com` (These are the VM names we'll use below in step 1).

**Once you're ready, follow steps 1-3 below to set up your VMs.**

1. Choose a name for your VM. A good choice is to name each VM the same as the domain name that will be used to access it. Once you've selected a name, run the following command (**on the physical host machine**). Don't forget to replace `[VM_NAME]`.
    ```shell
    virt-install --name [VM_NAME] --virt-type kvm --location http://deb.debian.org/debian/dists/bullseye/main/installer-amd64/ --os-variant debian11 --disk size=32 --memory 2048 --graphics none --console pty,target_type=serial --extra-args "console=ttyS0"
    ```

1. Navigate through the text-based Debian installer. Most options can remain as their default. When asked, you can set the hostname to the same value as the name you gave to your VM (for example, `example1-com`). You can leave the domain name and HTTP proxy information blank. Allow the installer to create both a root and a non-root user. Note down the password(s) you use here! You can allow the partitioner to use the entire disk, this uses the virtual disk that we defined above in step 1. When *tasksel* (Software selection) runs and asks if you want to install additional software, use spacebar and your arrow keys to uncheck the `Debian desktop environment` and `GNOME` options, and check the `SSH server` option (Lets you easily login again in the future in case you have to perform any maintenance). Make sure `standard system utilities` is also checked. Hit tab to select "Continue". Finally, disregard the warning about GRUB, allow it to install to your "primary drive" (again, virtual- this will not prevent your physical host machine from booting) and select `/dev/vda` for the bootable device.
1. Log in to your new VM. When it's finished installing, your new VM will reboot and present you with a login prompt. Use "root" as the username, and enter the password you chose during the installation process. Then, run this command (**on the VM**):
   ```shell
   apt install -y curl && curl -fsSL https://get.docker.com | sh && docker run --init --sig-proxy=false --name nextcloud-aio-mastercontainer --restart always --publish 8080:8080 --env APACHE_PORT=11000 --env APACHE_IP_BINDING=0.0.0.0 --volume nextcloud_aio_mastercontainer:/mnt/docker-aio-config --volume /var/run/docker.sock:/var/run/docker.sock:ro nextcloud/all-in-one:latest
   ```
   This command will install docker and AIO in reverse proxy mode! As with any other command, try your best to carefully read over it and understand it before running it. This may take a minute or two. When it's finished, you should see a success message, saying "Initial startup of Nextcloud All-in-One complete!". This concludes the setup needed on this particular VM.
   
   ---
1. Go ahead and run through steps 1-3 again in order to set up your second VM. When you're finished, proceed down to step 5.
1. Almost done! All that's left is configuring our reverse proxy. To do this, we first need to install it. Run (**on the physical host machine**):
   ```shell
   apt update -y && apt install -y debian-keyring debian-archive-keyring apt-transport-https curl && curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg && curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | tee /etc/apt/sources.list.d/caddy-stable.list && apt update -y && apt install -y caddy
   ```
   This command will ensure that your system is up-to-date and install the latest stable version of Caddy via it's official binary source.
1. To configure Caddy, we need to know the IP address assigned to each VM. Run (**on the physical host machine**):
    ```shell
    virsh net-dhcp-leases default
    ```
    This will show you the VMs you set up, and the IP address corresponding to each of them. Note down each IP and corresponding hostname.
    Finally, we will configure Caddy using the information proided by virsh. Open the default Caddyfile with a text editor:
    ```shell
    nano /etc/caddy/Caddyfile
    ```
    Replace everything in this file with the following configuration. Don't forget to edit the sample configuration and substitute in your own domain names and IP addresses. `[DOMAIN_NAME_*]` should be a domain name like `example1.com`, and `[IP_ADDRESS_*]` should be a local IPv4 address like `192.168.1.225`.
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
    ```
    After making this change, you'll need to restart caddy:
   ```shell
   systemctl restart caddy
   ```
1. That's it! Now, all that's left is to set up your instances through the AIO interface as usual by visiting `https://example1.com:8443` and `https://example2.com:8443` in a browser. Once you're finished going through each setup, you can access your new instances simply through their domain names. You can host as many instances with as many domain names as you want this way, as long as you have enough system resources. Enjoy!
    <!--
    <details><summary>A few extra tips!</summary>
    <ul>
        <li>Use `virsh autostart --domain [VM_NAME]` to set a VM to start automatically as soon as the physical host machine finishes booting. Highly recommended if you want to achieve minimal downtime.</li>
        <li>Use `virsh autostart --domain [VM_NAME]` to set a VM to start automatically as soon as the physical host machine finishes booting. Highly recommended if you want to achieve minimal downtime.</li>
    </ul>
    </details>
   -->

## Run multiple AIO instances on the same server with docker rootless
1. Create as many linux users as you need first. The easiest way is to use `sudo adduser` and follow the setup for that. Make sure to create a strong unique password for each of them and write it down!
1. Log in as each of the users by opening a new SSH connection as the user and install docker rootless for each of them by following step 0-1 and 3-4 of the [docker rootless documentation](./docker-rootless.md) (you can skip step 2 in this case).
1. Then install AIO in reverse proxy mode by using the command that is descriebed in step 2 and 3 of the [reverse proxy documentation](./reverse-proxy.md) but use a different `APACHE_PORT` and [`TALK_PORT`](https://github.com/nextcloud/all-in-one#how-to-adjust-the-talk-port) for each instance as otherwise it will bug out. Also make sure to adjust the docker socket and `WATCHTOWER_DOCKER_SOCKET_PATH` correctly for each of them by following step 6 of the [docker rootless documentation](./docker-rootless.md). Additionally, modify `--publish 8080:8080` to a different port for each container, e.g. `8081:8080` as otherwise it will not work.<br>
**⚠️ Please note:** If you want to adjust the `NEXTCLOUD_DATADIR`, make sure to apply the correct permissions to the chosen path as documented at the bottom of the [docker rootless documentation](./docker-rootless.md). Also for the built-in backup to work, the target path needs to have the correct permissions as documented there, too.
1. Now install your webserver of choice on the host system. It is recommended to use caddy for this as it is by far the easiest solution. You can do so by following https://caddyserver.com/docs/install#debian-ubuntu-raspbian or below. (It needs to be installed directly on the host or on a different server in the same network).
1. Next create your Caddyfile with multiple entries and domains for the different instances like described in step 1 of the [reverse proxy documentation](./reverse-proxy.md). Obviously each domain needs to point correctly to the chosen `APACHE_PORT` that you've configured before. Then start Caddy which should automatically get the needed certificates for you if your domains are configured correctly and ports 80 and 443 are forwarded to your server.
1. Now open each of the AIO interfaces by opening `https://ip.address.of.this.server:8080` or e.g. `https://ip.address.of.this.server:8081` or as chosen during step 3 of this documentation. 
1. Finally type in the domain that you've configured for each of the instances during step 5 of this documentation and you are done.
1. Please also do not forget to open/forward each chosen `TALK_PORT` UPD and TCP in your firewall/router as otherwise Talk will not work correctly!

Now everything should be set up correctly and you should have created multiple working instances of AIO on the same server!
