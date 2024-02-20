# Multiple AIO instances
It is possible to run multiple instances of AIO on one server.

There are two ways to achieve this: The normal way is creating multiple VMs, installing AIO in [reverse proxy mode](./reverse-proxy.md) in each of them and having one reverse proxy in front of them that points to each VM (you also need to [use a different `TALK_PORT`](https://github.com/nextcloud/all-in-one#how-to-adjust-the-talk-port) for each of them). The second and more advanced way is creating multiple users on the server and using docker rootless for each of them in order to install multiple instances on the same server. 

## Run multiple AIO instances on the same server inside their own virtual machines
1. Make sure your host machine has enough resources. 8GB ram and 100GB storage on the host machine is sufficent for running two instances with 2GB ram and 32GB storage each. This tutorial assumes you have these resources at the minimum.
2. Create one virtual machine for each instance that you need first. I recommend fully configuring them one at a time, and naming each VM the same as the domain that will be used to access it. Assuming you are on a server without a desktop environment installed, the easiest way is to install QEMU/KVM + virt-install + virsh ([help](https://wiki.debian.org/KVM)) and run this command:
```
virt-install --virt-type kvm --name [YOUR-VM-NAME] --location http://deb.debian.org/debian/dists/bullseye/main/installer-amd64/ --os-variant debian11 --disk size=32 --memory 2048 --graphics none --console pty,target_type=serial --extra-args "console=ttyS0"
```
For this example, we will assume that we have two domains where we would like to host Nextcloud AIO, *example1.com* and *example2.com*. Therefore, we create 2 VMs named *example1-com* and *example2-com*.
3. Running the above command will guide you through the Debian installer. I recommend setting the hostname to the same value as the name you gave to your VM. When *tasksel* runs and asks you to install a desktop, uncheck the "debian graphical" and "GNOME" options, and check the "ssh server" option (so that you may easily login to configure it). Make sure "standard system utilities" is also checked. Most other options can remain default.
4. 

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












# aio-setup
Settings for multiple AIO instances + multiple domains

Caddy file on host machine, not inside VM:

```
https://[DOMAIN_1]:8443 {
    reverse_proxy https://[VM_1_IP_ADDRESS]:8080 {
        transport http {
            tls_insecure_skip_verify
        }
    }
}
```
ALSO ADD:
```
https://[DOMAIN_1]:443 {
    reverse_proxy [VM_1_IP_ADDRESS]:11000
}
```

Both of those should be in caddy config on the host machine.

You can use the default nextcloud aio reverse proxy command but don't bother configuring caddy the way it tells you to.

Inside the VM, use
```shell
ip a
```
to fetch the IP address that the host will use in it's caddy config.

Inside VM, configure Nextcloud in reverse-proxy mode
