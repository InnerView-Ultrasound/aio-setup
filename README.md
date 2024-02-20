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

Inside VM, configure Nextcloud in reverse-proxy mode
