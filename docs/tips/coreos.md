# CoreOS

Various tricks for the super slim container OS.

## QEMU Guest Agent

The guest agent `qemu-ga` is required for the host to discover the virtual machine's network setup,
specifically it's IP. You can start the guest agent in an Alpine container:

```sh
docker run -d \
  -v /dev:/dev \
  --privileged \
  --net host \
  alpine ash -c 'apk add qemu-guest-agent && exec qemu-ga -v'
```
