# Setup electrumx testnet server with docker

## 1. Setup PACd node with docker

Used docker PACd image has `txindex=1` setting in PAC.conf,
which is need by electrumx server.

Create network to link with electrumx server.

```
docker network create PAC-testnet
```

Create volume to store PACd data and settings.

```
docker volume create PACd-data-testnet
```

Start PACd container.

```
docker run --restart=always -v PACd-data-testnet:/PAC \
    --name=PACd-node-testnet --net PAC-testnet -d \
    --env TESTNET=1 \
    -p 19999:19999 -p 127.0.0.1:19998:19998 zebralucky/PACd:v0.12.2
```

**Notes**:
 - port 19999 is published without bind to localhost and can be
 accessible from out world even with firewall setup:
 https://github.com/moby/moby/issues/22054

Copy or change RPC password. Random password generated
on first container startup.

```
docker exec -it PACd-node-testnet bash -l

# ... login to container

cat .PACcore/PAC.conf | grep rpcpassword
```

See log of PACd.

```
docker logs PACd-node-testnet
```

## 2. Setup electrumx server with docker

Create volume to store elextrumx server data and settings.

```
docker volume create electrumx-PAC-data-testnet
```

Start elextrumx container.

```
docker run --restart=always -v electrumx-PAC-data-testnet:/data \
    --name electrumx-PAC-testnet --net PAC-testnet -d \
    -p 51001:51001 -p 51002:51002 zebralucky/electrumx-PAC:testnet
```

Change DAEMON_URL `rpcpasswd` to password from PACd and creaate SSL cert.

**Notes**:
 - DAEMON_URL as each URL can not contain some symbols.
 - ports 51001, 51002 is published without bind to localhost and can be
 accessible from out world even with firewall setup:
 https://github.com/moby/moby/issues/22054

```
docker exec -it electrumx-PAC-testnet bash -l

# ... login to container

cd /data/

# Edit and save env/DAEMON_URL
nano env/DAEMON_URL

# Create SSL self signed certificate

openssl genrsa -des3 -passout pass:x -out server.pass.key 2048 && \
openssl rsa -passin pass:x -in server.pass.key -out server.key && \
rm server.pass.key && \
openssl req -new -key server.key -out server.csr

openssl x509 -req -days 730 -in server.csr -signkey server.key \
  -out server.crt && rm server.csr


exit
# ... logout from container

# Restart electrumx container to switch on new RPC password

docker restart electrumx-PAC-testnet
```

See log of electrumx server.

```
docker exec -it electrumx-PAC-testnet bash -l

# ... login to container

tail /data/log/current

# or less /data/log/current
```

Wait some time, when electrumx sync with PACd and
starts listen on client ports. It can be seen on `/data/log/current`.
