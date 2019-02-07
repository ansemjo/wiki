# Docker

## Firewalling

By default, `docker` seems to start with `--iptables=true` everywhere. That means that the docker
daemon will insert its own iptables rules to enable inter-container communication and publish ports.
_However_ that means that published ports will be published **publicly** by default.

That means that a container started with `-p 8000:8000` will be open to the world on that port.
_Even if_ your firewalld configuration does not permit this port. This is because Docker completely
circumvents any firewall managers.

### Disable `iptables` tampering

To disable this behaviour add `--iptables=false` to the start arguments of docker. Either do that by
editing the systemd service, or set an `DOCKER_OPTS="..."` in `/etc/default/docker` if applicable.

    $ systemctl edit docker.service
    [Service]
    ExecStart=
    ExecStart=/usr/bin/dockerd -H fd:// --iptables=false

This also disables the forwarding rules however. Your containers will not be able to reach the
outside world anymore. To reenable the forwarding with `firewalld` use:

    $ firewall-cmd --add-masquerade --permanent

Or using raw `iptables` rules:

    -A FORWARD -i docker0 -o eth0 -j ACCEPT
    -A FORWARD -i eth0 -o docker0 -j ACCEPT
