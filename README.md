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
