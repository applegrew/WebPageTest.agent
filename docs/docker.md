# Docker Linux Headless Agent

The `Dockerfile` has multi-stage definition:
* **production**: Default stage, produce a image without debug features;
* **debug**: When running the produced image the wptagent script will wait for a debugger to.attach

## Setup in Azure

Choose Ubuntu 22.04 CCOE image.

These images come with CCOE network rules which prevent SSH and HTTP from being accessed outside office cloud VPN network (test reveals that SSH does not work over cloud VPN). So we need to create lower order rules from Networking tab in Azure to override those rules however, it seems there is some automation which deletes those rules after around 30mins. Also the image has iptables rules to restrict inbound HTTP (which is relevant only for WPT server, no change needed here for WPT agent).

Goto: http://wpt-server.centralindia.cloudapp.azure.com/install/ to see if WPT server setup is correct.

![Install check](Install%20Check.png)

## Build the Image

Arguments can be passed at build time:
* **TIMEZONE**: to set the timezone inside the container. Default `UTC.`

To build the production container with UTC timezone (recommended)
```bash
docker build --tag wptagent .
```

changing the timezone at build time
```bash
docker build --build-arg TIMEZONE=EST .
```

To build the debug container
```bash
docker build --target debug --tag wptagent-debug .
```

## Prerequisites to use traffic shaping in docker
**Experimental**: Running the agent with traffic shaping is experimental. It might
have influence on the host system network. Running multiple agents on the
same host might result in incorrect traffic shaping.

For traffic shaping to work correctly, you need to load the ifb module on the **host**:
```bash
    sudo modprobe ifb numifbs=1
```

Also, the container needs `NET_ADMIN` capabilities, so run the container with 
`--cap-add=NET_ADMIN`.

To disable traffic-shaping, pass environment variable at docker un `SHAPER="none"`.

## Run the container
To run the agent, simply specify a few environment variables with docker:

- `SERVER_URL`: will be passed as `--server` (note: it must end with '/work/')
- `LOCATION`: will be passed as `--location`
- `KEY`: will be passed as `--key`
- `NAME`: will be passed as `--name` (optional)
- `SHAPER`: will be passed as `--shaper` (optional)
- `EXTRA_ARGS`: extra command-line options that will be passed through verbatim (optional)

Build the image first (from project root), load ifb and start it the container.

A typical run :
```bash
    sudo modprobe ifb numifbs=1
    docker build --tag wptagent .
    docker run -d \
      -e SERVER_URL="http://my-wpt-server.org/work/" \
      -e LOCATION="docker-location" \
      -e NAME="Docker Test" \
      --cap-add=NET_ADMIN \
      --init \
      wptagent
```

Recommended run (uses default Netem traffic shaper):
```bash
sudo docker run -d \
      -e SERVER_URL="http://wpt-server.centralindia.cloudapp.azure.com/work/" \
      -e LOCATION="india-central-1" \
      -e NAME="India Central 1" \
      -e KEY="<location_key from wpt server /var/www/webpagetest/www/settings/settings.ini>"\
      --cap-add=NET_ADMIN \
      --init \
      wptagent
```

The above assumes the following settings in locations.ini on WPT server.
```ini
[locations]
1=india-central
default=india-central

; These are the top-level locations that are listed in the location dropdown
; Each one points to one or more browser configurations

[india-central]
1=india-central-1
label=India Central

; Tese are the browser-specific configurations that match the configurations
; defined in the top-level locations.  Each one of these MUST match the location
; name configured on the test agent (urlblast.ini or wptdriver.ini)

[india-central-1]
browser=Chrome,Firefox
label="India Central Loc 1"
```

Additional parameters can be also passed as additional commands. 
A typical run in debug mode, note that we need to expose the port as `50000`:
```bash
sudo modprobe ifb numifbs=1
docker run -d \
    -e SERVER_URL=http://127.0.0.1:80/work/ \
    -e LOCATION=Test \
    --init \
    --cap-add=NET_ADMIN \
    -p 50000:50000 \
    wptagent-debug
    --key 123456789
```

## Container Disk Space Fills Up Quickly

If you see disk space within the container filling up rapidly and you notice
core dump files in the /wptagent folder, try adding `--shm-size=1g` to your Docker run
command. This can help resolve an issue with shared memory and headless Chrome in Docker.

### Helpful docker commands

List running containers:
```bash
sudo docker ps
```

Access shell in running container:
```bash
sudo docker exec -it <container_name> bash
```

Check if node version is 18 and not 12:
```bash
sudo docker exec <container_name> node -v
```

Restart container:
```bash
sudo docker restart <container_name>
```

Check logs:
```bash
sudo docker logs <container_name>
```
