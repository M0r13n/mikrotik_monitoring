# Monitor your Mikrotik router with Prometheus and Grafana

Over the past years I replaced all my networking gear with Mikrotik devices. I absolutely love WinBox for management, but sometimes I miss some fancy charts and graphs. Luckily, RouterOS comes with a REST-API that can be used to query arbitrary data from the device. After some research I found some useful tools and blog posts that solved similar problems. Namely:

- [A blog post by Devin Smith that first got me interested](https://blog.devinsmith.co.za/home-internet-grafana-lockdown/)
- [A somewhat useable Grafana Dashboard](https://grafana.com/grafana/dashboards/10950)
- [A Prometheus exporter for Mikrotik devices written in Python](https://github.com/akpw/mktxp).
- [A Prometheus exporter for Mikrotik devices written in Go](https://github.com/nshttpd/mikrotik-exporter)

## Setup

- Router running RouterOS 7.x.x
- Raspberry Pi 4 with 2 gb RAM (other PIs may also work, but I wanted ARM 64 bit)

## Demo pictures

![General system stats](https://github.com/M0r13n/mikrotik_monitoring/blob/main/doc/pic1.png)
![Wifi stats](https://github.com/M0r13n/mikrotik_monitoring/blob/main/doc/pic2.png)

## Installation

## Mikrotik Router
At first you need to prepare your router. 

Create a group on the device that has API and read-only access:

`/user group add name=prometheus policy=api,read,winbox,test`

Create a user that is part of the group:

`/user add name=prometheus group=prometheus password=TOP_SECRET`


## Prepare Synology NAS

Firstly, activate SNMPv3 in your Synology NAS und set a username and password (md5).

In order to get stats from a Synology NAS into Prometheus an exporter is needed.

-> https://github.com/prometheus/snmp_exporter

This exporter comes with a pre-build docker image.

There is a pre-build `snmp.yml`, where you only need to update the username and password in the bottom `auth` section.

<details>
<summary>Optional: How to generate `snmp.yml`</summary>

The generator file can be found in `./synology/generator.yml`.

Create the `snmp.yml` exporter config:

1. Change `auth.username` and `auth.password` to match the Synology NAS
2. Make a temporary directory: `mkdir tmp && cd tmp`
3. Get the SNMP exporter repo: `git clone git@github.com:prometheus/snmp_exporter.git && cd snmp_exporter`
4. Copy the generator file: `cp ../../synology/generator.yml ./generator && cd generator` (yes override)
5. Prepare the MIB files: `make mibs`
6. Generate the `snmp.yml` file using Docker:
```bash
docker build -t snmp-generator .
sudo docker run -ti \
  -v "${PWD}:/opt/" \
  snmp-generator generate 
```
7. If everything went well, a config file has been written to `./snmp.yml`
8. Copy this file into the Synology folder (next to the generator.yml): `cp ./snmp.yml ../../../synology/`

</details>


## Prepare Raspi
You need Ubuntu Server for ARM 64 bit in order to use this setup. You may also use Raspian, but then you are limited to 32bit ARM executables. This would mean, that you need to compile the `mikrotik-exporter` by hand, because there are no predefined 32-bit Docker images.

Install Python and pip:

`sudo apt install python3-dev python3 python3-pip -y`

Install Docker + Docker-compose

```bash
curl -sSL https://get.docker.com | sh
sudo usermod -aG docker ubuntu
sudo pip3 install docker-compose
sudo systemctl enable docker
```

Build the mktxp Docker image

```bash
# Get the mktxp repository
git clone https://github.com/akpw/mktxp.git

# Go into the newly downloaded repo
cd mktxp

# Build the docker image
docker build -t mktxp
```

Now get this repo and install all services:

```bash
# Clone this repo
git clone https://github.com/M0r13n/mikrotik_monitoring.git


# Go into the cloned directory
cd mikrotik_monitoring

# Let docker-compose do it's job
sudo docker-compose up -d
```

You may need to adjust the following configuration files and add your own credentials for your router:

- `mktxp/mktxp.conf`


Done. You should now be able to open the Grafana dashboard on Port 3000 of your Raspberry Pi.
