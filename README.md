# Multiple AIO instances
It is possible to run multiple instances of AIO on one server.

There are two ways to achieve this: The normal way is creating multiple VMs, installing AIO in [reverse proxy mode](./reverse-proxy.md) in each of them and having one reverse proxy in front of them that points to each VM (you also need to [use a different `TALK_PORT`](https://github.com/nextcloud/all-in-one#how-to-adjust-the-talk-port) for each of them). The second and more advanced way is creating multiple users on the server and using docker rootless for each of them in order to install multiple instances on the same server. 

## Run multiple AIO instances on the same server inside their own virtual machines
This guide will walk you through creating and configuring two Debian VMs (with "reverse proxy mode" Nextcloud AIO installed in each VM) behind one Caddy reverse proxy, all running on one physical host machine.

First, make sure your physical host machine has enough resources. A host machine with 8GB RAM and 100GB storage is sufficent for running two fairly minimal VMs, with 2GB RAM and 32GB storage allocated to each VM. This tutorial assumes you have these resources at the minimum. This is fine for just testing the setup, but you will probably want to allocate more resources to your VMs if you plan to use this for day-to-day use.

If your host machine has more than 8GB memory available and you plan to enable any of the optional containers (Nextcloud Office, Talk, Imaginary, etc.) in any of your instances, then you should definitely allocate more memory to the VM hosting that instance. Put simply, before turning on any extra features inside a particular AIO interface, make sure you've first allocated enough resources to the VM that the instance is running inside. If in doubt, the AIO interface iteself actually gives great recommendations regarding how much extra CPU and RAM to allocate.

For this guide, we'll assume that we have two domains where we would like to host Nextcloud AIO, `example1.com` and `example2.com`. Therefore, we create 2 VMs named `example1-com` and `example2-com`.

1. Create one virtual machine for each instance that you need first. I recommend fully configuring each VM one at a time by following steps 1 through 3 in order, and naming each VM the same as the domain name that will be used to access it. Assuming your physcial host is a barebones server without a desktop environment installed, the easiest way is to install QEMU/KVM + virt-install + virsh ([read more](https://wiki.debian.org/KVM)) and run this command (**on the physical host machine**):
    ```shell
    virt-install --virt-type kvm --name [YOUR-VM-NAME] --location http://deb.debian.org/debian/dists/bullseye/main/installer-amd64/ --os-variant debian11 --disk size=32 --memory 2048 --graphics none --console pty,target_type=serial --extra-args "console=ttyS0"
    ```

1. Running the above command will guide you through the CLI-based Debian installer. When asked, I recommend setting the hostname to the same value as the name you gave to your VM (for example, `example1-com`). When *tasksel* runs and asks you to install a desktop, uncheck the "debian graphical" and "GNOME" options, and check the "ssh server" option (so that you may easily login to configure it). Make sure "standard system utilities" is also checked. Most other installer options can remain default.
1. Once your VM reboots, it will ask you to login to it. Use "root", enter the password you selected, and then run the following (**on the VM**):
   ```shell
   apt install -y debian-keyring debian-archive-keyring apt-transport-https curl && curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg && curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | tee /etc/apt/sources.list.d/caddy-stable.list && apt update && apt install caddy && curl -fsSL https://get.docker.com | sh && docker run --init --sig-proxy=false --name nextcloud-aio-mastercontainer --restart always --publish 8080:8080 --env APACHE_PORT=11000 --env APACHE_IP_BINDING=0.0.0.0 --volume nextcloud_aio_mastercontainer:/mnt/docker-aio-config --volume /var/run/docker.sock:/var/run/docker.sock:ro nextcloud/all-in-one:latest
   ```
   This really long command will install the latest stable Caddy, docker, and Nextcloud AIO in reverse proxy mode! As with any other command, try your best to carefully read over it and understand it before running it.
1. Go ahead and run through steps 1-5 again in order to set up your second VM and Nextcloud AIO instance.
1. Almost done! All that's left is configuring Caddy. To do this, we need to find out the IP for each VM. Run (**on the physical host machine**):
    ```shell
    virsh net-dhcp-leases default
    ```
    This will show you the VMs you set up, and the IP address corresponding to each of them. Note down each IP and corresponding hostname.
    Finally, we will configure Caddy using the information proided by virsh. Open the default Caddyfile:
    ```shell
    nano /etc/caddy/Caddyfile
    ```
    Replace everything in this file with the following configuration:
    ```shell
    https://example-1.com:8443 {
        reverse_proxy https://[IP_ADDRESS_FOR_example1-com_VM]:8080 {
            transport http {
                tls_insecure_skip_verify
            }
        }
    }
    
    https://example-1.com:443 {
        reverse_proxy [IP_ADDRESS_FOR_example1-com_VM]:11000
    }
    
    
    https://example-2.com:8443 {
        reverse_proxy https://[IP_ADDRESS_FOR_example2-com_VM]:8080 {
            transport http {
                tls_insecure_skip_verify
            }
        }
    }
    
    https://example-2.com:443 {
        reverse_proxy [IP_ADDRESS_FOR_example2-com_VM]:11000
    }
    ```
    If you haven't already, edit the sample configuration and substitute in your domain names and IP addresses.

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
