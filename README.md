# docker-compose-configs

## Synology

If running containers on a Synology NAS, create 2 tasks in `Control Panel` > `Services` > `Task Scheduler`

1. Disable NAT [script](https://gist.github.com/PedroLamas/db809a2b9112166da4a2dbf8e3a72ae9). This stops the Synology NAS NATing all traffic to the `docker0` interface IP desccribe in this [issue](https://github.com/pi-hole/docker-pi-hole/issues/135#issuecomment-614724181).
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