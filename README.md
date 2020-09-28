# lnd for Docker

Docker image that runs lnd in a container for easy deployment. Modified some parameters to run on mainnet.

The image contains the latest [lnd](https://github.com/lightningnetwork/lnd) daemon and [lndconnect](https://github.com/LN-Zap/lndconnect).

## Quick Start

1.  Create a `lnd-data` volume to persist the lnd data, should exit immediately. The `lnd-data` container will store the lnd data when the node container is recreated (software upgrade, reboot, etc):

        docker volume create --name=lnd-data
        docker run -v ~/lnd-data:/lnd --name=lnd-node -d \
            -p 9735:9735 \
            -p 10009:10009 \
            lnzap/lnd:latest \
            --bitcoin.active \
            --bitcoin.mainnet \
            --debuglevel=info \
            --bitcoin.node=neutrino \
            --neutrino.connect=mainnet1-btcd.zaphq.io \
            --neutrino.connect=mainnet2-btcd.zaphq.io \
            --neutrino.connect=btcd-mainnet.lightning.computer \
            --rpclisten=0.0.0.0:10009

2.  Verify that the container is running and lnd node is downloading the blockchain

        $ docker ps
        CONTAINER ID        IMAGE                         COMMAND             CREATED             STATUS              PORTS                                              NAMES
        d0e1076b2dca        lnzap/lnd:latest            "lnd_oneshot"       2 seconds ago       Up 1 seconds        0.0.0.0:9735->9735/tcp, 0.0.0.0:10009->10009/tcp   lnd-node

3.  You can then access the daemon's output thanks to the [docker logs command](https://docs.docker.com/reference/commandline/cli/#logs)

        docker logs -f lnd-node

4.  Install optional init scripts for upstart and systemd are in the `init` directory.

5.  You must set up a wallet in order to generate macaroons.

        docker exec -u lnd -it lnd-node lncli create


## Documentation

- Additional documentation in the [docs folder](docs).
