# Pi-hole & Unbound DNS Docker Setup

This is a docker compose setup which starts a [Pi-hole](https://pi-hole.net/) and [nlnetlab's Unbound](https://nlnetlabs.nl/projects/unbound/about/) as upstream recursive DNS using official (or ready-to-use) images. The main idea here is to add security, [privacy](https://www.cloudflare.com/learning/dns/what-is-recursive-dns/) and have ad and malware protection, everything hosted locally.

If you want to learn more about why you want to have exactly this setup, read a [detailed explanation here](https://docs.pi-hole.net/guides/dns/unbound/).

## Prepare

### Understand the Setup

This setup works on a machine that does not itself already has DNS running (i.e. port 53 is already used). If you have a setup like that (e.g. running on a Synology NAS with a Directory Server), you would need a setup that creates a [Mac VLAN](https://docs.docker.com/network/macvlan/) so the container appears with a different IP. In this [case check out this example here](https://github.com/chriscrowe/docker-pihole-unbound/tree/main/two-container).

It is designed to have 2 containers running next to each other and do not aim to combine both programs in one. The idea is to minimize the work needed to adapt provided containerized versions of Pi-hole and Unbound, i.e. use the official images, therefore making it easier to upgrade each.

### Prerequisites

First you need a recent version of [Docker installed](https://docs.docker.com/get-docker/) which at least supports Docker compose v2.
Further you may want to have a [server or IoT device](https://docs.pi-hole.net/main/prerequisites/) where this stack can run on, since this should be reachable by every other client 24/7.
Finally, don't forget to change your [default DNS server to the server IPs address of your server](https://docs.pi-hole.net/main/post-install/).

### Configuration

The main configuration can be set in the `.env` file which overwrites the ENV variables in the `docker-compose.yml` - change it to your liking:

```properties
WEBPASSWORD= # set the password to use in the Web Admin UI
HOST_IP_V4_ADDRESS= # the IP of the host the Pi-hole runs on - defaults to localhost
TIMEZONE= # set your timezone (used to schedule cron jobs e.g.)
```

## Deploy Containers

[![asciicast](https://asciinema.org/a/581383.svg)](https://asciinema.org/a/581383)

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
#      - "/var/lib/docker/volumes/pihole/pihole:/etc/pihole/"
#      - "/var/lib/docker/volumes/pihole/dnsmasq.d:/etc/dnsmasq.d/"
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

### Define Local A-Records 

If you want to resolve certain domains locally you can set A-Records in `./unbound/conf/a-records.conf`. There are already examples, but to add a new record do:

```
# Example: Resolve all *.mysite.com addresses to the same ip of the main reverse proxy / router

local-zone: "mysite.com." redirect
local-data: "mysite.com. 86400 IN A 192.168.1.1"
```

Check here the [full documentation](https://unbound.docs.nlnetlabs.nl/_/downloads/en/latest/pdf/) or [tutorial](https://calomel.org/unbound_dns.html) to learn more.

### Unbound, Forwarders and Manual Configuration

Unbound is set as a recursive DNS, because all forwarders in `./unbound/conf/a-records.conf` are commented out. If you prefer to use cloudflare or any other public DNS as upstream instead of having the slight performance impact of directly asking the nameservers, then you can enable the respective server by removing the comment (but then using Unbound at all has little value.

If you want to fine-tune the Unbound configuration, you can add the file `./unbound/conf/unbound.conf` (see an [example here](https://github.com/MatthewVance/unbound-docker/blob/master/unbound.conf)) and Unbound will use it.

# Links

## Credits

* [nlnetlabs Unbound](https://nlnetlabs.nl/projects/unbound/about/) (BSD license)
* [MatthewVance's Unbound Docker Image](https://github.com/MatthewVance/unbound-docker) (MIT License)
* [Pi-hole](https://github.com/pi-hole/pi-hole) (European Union Public License)
* [Official Pi-hole Docker Image](https://github.com/pi-hole/docker-pi-hole) (unknown license)

## Similar Projects

* [chriscrowe's Mac Vlan Setup](https://github.com/chriscrowe/docker-pihole-unbound)
* [origamiofficial's One Container Solution](https://github.com/origamiofficial/docker-pihole-unbound)
* [JD10NN3's Solution](https://github.com/JD10NN3/docker-pihole-unbound)

## Further Information

* [Pi-hole Documentation](https://docs.pi-hole.net/)
* [Unbound Documentation](https://unbound.docs.nlnetlabs.nl/_/downloads/en/latest/pdf/)
* [Pi-hole + Unbound Details](https://docs.pi-hole.net/guides/dns/unbound/)
* [How to run docker-compose on remote host?](https://stackoverflow.com/questions/35433147/how-to-run-docker-compose-on-remote-host) 

# License

Copyright 2023 Patrick Favre-Bulle

Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with the License. You may obtain a copy of the License at

```
https://www.apache.org/licenses/LICENSE-2.0
```

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.
