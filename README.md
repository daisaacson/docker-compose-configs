# docker-compose-configs

## Raspberry PI

Drive considerations:
- Don't use SDCard, too slow
- Don't use USB thumb drives, better but still too slow and small
- I'm using USB M.2 SSD.
    - My USB devices was buggy with `UAS`, but works perfectly with it disabled<sup>[1](https://forums.raspberrypi.com/viewtopic.php?t=245931) [2](https://leo.leung.xyz/wiki/How_to_disable_USB_Attached_Storage_(UAS))</sup>.
    
        ```bash
        lsusb
        ```

        ```plainttext
        Bus 002 Device 002: ID 152d:0580 JMicron Technology Corp. / JMicron USA Technology Corp. USB 3.1 Storage Device
        Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
        Bus 001 Device 002: ID 2109:3431 VIA Labs, Inc. Hub
        Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
        ```

    - Add the storage device `ID` a quirks in a `/etc/modprobe.d/disable-uas.conf` file

        ```plaintext
        options usb-storage quirks=152d:0580:u
        ```

## Synology

If running containers on a Synology NAS, create 2 tasks in `Control Panel` > `Services` > `Task Scheduler`

1. Disable NAT. This stops the Synology NAS NATing all traffic to the `docker0` interface IP<sup>[3](https://gist.github.com/PedroLamas/db809a2b9112166da4a2dbf8e3a72ae9) [4](https://github.com/pi-hole/docker-pi-hole/issues/135#issuecomment-614724181)</sup>.
    1. Task: `Docker Disable NAT`
    1. User: `root`
    1. Event: `Boot-up`
    1. User-defined script:

        ```bash
        #!/bin/bash
        currentAttempt=0
        totalAttempts=10
        delay=15

        while [ $currentAttempt -lt $totalAttempts ]
        do
            currentAttempt=$(( $currentAttempt + 1 ))

            echo "Attempt $currentAttempt of $totalAttempts..."

            result=$(iptables-save)

            if [[ $result =~ "-A DOCKER -i docker0 -j RETURN" ]]; then
                echo "Docker rules found! Modifying..."

                iptables -t nat -A PREROUTING -m addrtype --dst-type LOCAL -j DOCKER
                iptables -t nat -A PREROUTING -m addrtype --dst-type LOCAL ! --dst 127.0.0.0/8 -j DOCKER

                echo "Done!"

                break
           fi

            echo "Docker rules not found! Sleeping for $delay seconds..."
            
            sleep $delay
        done
        ```

1. Prune Images to at boot to prevent a build up of unused images.
    1. Task: `Docker Prune Images`
    1. User: `root`
    1. Event: `Boot-up`
    1. User-defined script:

        ```bash
        #!/bin/bash
        docker image prune --force --all
        ```