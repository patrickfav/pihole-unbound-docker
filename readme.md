# DNS Pi-hole & Unbound Stack

This is a docker compose setup which starts a [Pi-hole](https://pi-hole.net/) and [nlnetlab's Unbound](https://nlnetlabs.nl/projects/unbound/about/) as upstream recursive DNS. The main idea here is to add security, [privacy](https://www.cloudflare.com/learning/dns/what-is-recursive-dns/) and have ad and malware protection, everything locally hosted.

If you want to learn more about why you want to have exactly this setup, read a [detailed explanation here](https://docs.pi-hole.net/guides/dns/unbound/).

## Prepare

### Prerequisites

First you need a recent version of [Docker installed](https://docs.docker.com/get-docker/) which at least supports Docker compose v2.
Further you may want to have a s[erver or IoT device](https://docs.pi-hole.net/main/prerequisites/) where this stack can run on, since this should be reachable by every other client 24/7.
Finally, don't forget to change your [default DNS server to the server IPs address of your server](https://docs.pi-hole.net/main/post-install/).

### Configuration

The main configuration can be set in the `.env` file which overwrites the ENV variables in the `docker-compose.yml` - change it to your liking:

```properties
WEBPASSWORD= # set the password to use in the Web Admin UI
HOST_IP_V4_ADDRESS= # the IP of the host the Pi-hole runs on - defaults to localhost
TIMEZONE= # set your timezone (used to schedule cron jobs e.g.)
```
## Deploy

Start the stack with going to the root of the repo and do:

```bash
 docker compose up --build -d --remove-orphans
```

To stop everything:

```bash
 docker compose down
```

Pro-Tip, if you want to directly deploy to a remote you can do

```bash
 docker compose -H "ssh://your-remote-host" up --build -d --remove-orphans
```

## Use Web UI

If you didn't change anything and start this on your local machine you can access the Pi-hole web ui with

```
http://localhost:8080/admin/
```

The default password is `changeMeNow`.

## Test Setup

To test if Pi-Hole with unbound is working correctly you can use the test domain `unboundpiholetestdomain.org` I set up in Unbound.
In your terminal (you might [need to install `nslookup`](https://www.tecmint.com/install-dig-and-nslookup-in-linux/)) do:

```
nslookup unboundpiholetestdomain.org localhost
```
This command will use localhost as DNS, if you are running it on a different machine, use the appropriate IP.

This should return the IP `192.168.123.123`:
```
Server:         localhost
Address:        ::1#53

Name:   unboundpiholetestdomain.org
Address: 192.168.123.123
```

if setup correctly it should also work without forcing DNS

```
nslookup unboundpiholetestdomain.org
```

## Advanced

### Persistence after Restart

By default, Pi-hole will forget everything after a restart of the docker container. To change that you need to set
a docker volume to show Pi-hole where to save the configuration. You need to map `/etc/Pi-hole/` and `/etc/dnsmasq.d/` to
a directory on the server. [Read here if you want to learn more about volumes](https://stackoverflow.com/questions/68647242/define-volumes-in-docker-compose-yaml).

There is an example in the `docker-compose.yml`:

```yaml
services:
  Pi-hole:
    container_name: ...
# RECOMMENDED: Uncomment and adapt if you want to persist Pi-hole configurations after restart
#    volumes:
#      - "/var/lib/docker/volumes/Pi-hole/Pi-hole:/etc/Pi-hole/"
#      - "/var/lib/docker/volumes/Pi-hole/dnsmasq.d:/etc/dnsmasq.d/"
```

### Pi-hole Configurations

In the `docker-compose.yml` you can add or change the Pi-hole Docker standard configuration variables in

```yaml
services:
  Pi-hole:
    container_name: ...
    environment:
      # here
```
Check out [possible configurations here](https://github.com/pi-hole/docker-pi-hole).

Additionally, you can change various settings in your Pi-hole instance (e.g. the used ad-list) through the web ui. I won't
get into detail here apart from recommending `https://v.firebog.net/hosts/lists.php` as a good default starting list.


### Upgrade Base Images

In the `docker-compose.yml` change the used Pi-hole version by changing

```yaml
services:
  Pi-hole:
    container_name: ...
    image: Pi-hole/Pi-hole:2023.03.1 # <- update image version here, see: https://github.com/pi-hole/docker-pi-hole/releases
    hostname: ...
```

and Unbound by changing the `FROM` in `./unbound/Dockerfile` 

```dockerfile
# Update the version here, I use the docker build from https://github.com/MatthewVance/unbound-docker
FROM mvance/unbound:1.17.1
...
```

## Define Local A-Records 

If you want to resolve certain domains locally you can set A-Records in `./unbound/conf/a-records.conf`. There are already examples, but to add a new record do:

```
# Example: Resolve all *.mysite.com addresses to the same ip of the main reverse proxy / router

local-zone: "mysite.com." redirect
local-data: "mysite.com. 86400 IN A 192.168.1.1"
```

Check here the [full documentation](https://unbound.docs.nlnetlabs.nl/_/downloads/en/latest/pdf/) or [tutorial](https://calomel.org/unbound_dns.html) to learn more.

# Links

* [Pi-hole Documentation](https://docs.pi-hole.net/)
* [Pi-hole Docker](https://github.com/pi-hole/docker-pi-hole)
* [Unbound Docker](https://github.com/MatthewVance/unbound-docker)
* [Unbound Documentation](https://unbound.docs.nlnetlabs.nl/_/downloads/en/latest/pdf/)
* [Pi-hole + Unbound Details](https://docs.pi-hole.net/guides/dns/unbound/)
* [How to run docker-compose on remote host?](https://stackoverflow.com/questions/35433147/how-to-run-docker-compose-on-remote-host) 
