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
            --neutrino.connect=btcd-mainnet.lightning.computer \
            --neutrino.connect=bb1.breez.technology \
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

6.  You will likely need to add these settings to lnd.conf, delete tls.cert and tls.key files, and restart lnd to regenerate them:

        tlsextraip=IPADDRESS
        externalip=IPADDRESS
        
        # Other settings are optional, refer to [these](https://github.com/alexbosworth/run-lnd) instructions for an example
        # [This](https://github.com/lightningnetwork/lnd/blob/master/sample-lnd.conf) is also an exhaustive exmaple
        
7.  I find that the easiest way to set up a gRPC connection to this node from other devices is to use lndconnect.

        # In a temporary location, download lndconnect and install
        wget https://github.com/LN-Zap/lndconnect/releases/download/v0.2.0/lndconnect-linux-386-v0.2.0.tar.gz
        sudo tar -xvf lndconnect-linux-386-v0.2.0.tar.gz --strip=1 -C /usr/local/bin
        # Display help to make sure it works
        lndconnect -h
        # Issue an lndconnect URI
        lndconnect --lnddir=/home/<username>/lnd-data/.lnd --host=<IPADDRESS> -j
        # If using EC2 following [these](https://github.com/alexbosworth/run-lnd) instructions:
        lndconnect --host=<EC2_IPADDRESS> -j
        # If you want a scannable QR code in your terminal, omit the -j parameter
        lndconnect --lnddir=/home/<username>/lnd-data/.lnd --host=<IPADDRESS>

8.  If still having trouble connecting, make sure firewall settings allow port 9735 & 10009.

9.  Use a library like [node-lnd-grpc](https://github.com/LN-Zap/node-lnd-grpc/) or [ln-service](https://github.com/alexbosworth/ln-service), connect with lndconnectUri or its cert and macaroon params and get started!

## Requesting a Bitrefill Thor channel

An easy way to set up a channel with inbound liquidity is to use [Bitrefill's Thor service](https://www.bitrefill.com/thor-lightning-network-channels/?hl=en).

In order to do so, you will need to enter the docker container created above and use the command line (bash) like so:

        sudo docker exec -it lnd-node bash

When opening a channel with the Bitrefill Thor service, you are given a long command ("LND Channel") in the website that looks like this:

        lncli connect <Bitrefill's LND Node Pubkey>@<Bitrefill's IP>:9735 >/dev/null 2>&1; lncli getinfo|grep '"identity_pubkey"'|sed -e 's/.*://;s/[^0-9a-f]//g'|tr -d '\n'| curl -G --data-urlencode remoteid@- "https://api.bitrefill.com/v1/thor?k1=d8826be4a667ddb43745c3107006d7a17fffb374e8cec2262309ba363d399569&private=0"
        
However, this may not work out of the box with our setup. This is due to the root directory being named `/lnd` rather than `/root`.

Therefore, we must run the command adding two critical options to each lncli call. (I have split the call into its two parts for visibility. You can execute them this way too)

        lncli --tlscertpath /lnd/.lnd/tls.cert --adminmacaroonpath /lnd/.lnd/data/chain/bitcoin/mainnet/admin.macaroon connect 030c3f19d742ca294a55c00376b3b355c3c90d61c6b6b39554dbc7ac19b141c14f@52.50.244.44:9735
        
        lncli --tlscertpath /lnd/.lnd/tls.cert --adminmacaroonpath /lnd/.lnd/data/chain/bitcoin/mainnet/admin.macaroon getinfo|grep '"identity_pubkey"'|sed -e 's/.*://;s/[^0-9a-f]//g'|tr -d '\n'| curl -G --data-urlencode remoteid@- "https://api.bitrefill.com/v1/thor?k1=d8826be4a667ddb43745c3107006d7a17fffb374e8cec2262309ba363d399569&private=0"

This should return a JSON string stating `{"status":"OK"}` if successful. I hope that works!

Exit the docker container by typing `exit`, or continue to fool around with lncli (remembering to add the --tlscerpath and --macaroonpath parameters!).

## Running your own btcd

Relying on altruistic third party btcd nodes for your neutrino node is not da wae. Check out Zap's repository [LN-Zap/docker-btcd](https://github.com/LN-Zap/docker-btcd) for a quick and simple way to run a local btcd node.

## Documentation

- Additional documentation in the [docs folder](docs).
