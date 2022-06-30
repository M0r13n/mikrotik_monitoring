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

## Prepare Raspi

You need Ubuntu Server for ARM 64 bit in order to use this setup. You may also use Raspian, but then you are limited to 32bit ARM executables. This would mean, that you need to compile the `mikrotik-exporter` by hand, because there are no pre built 32-bit Docker images.

You need to execute the following steps on the target machine itself (e.g. Raspberry Pi).

Install Python and pip:

`sudo apt install python3-dev python3 python3-pip -y`

Install Docker + Docker-compose (reboot required)

```bash
curl -sSL https://get.docker.com | sh
sudo usermod -aG docker ubuntu
sudo pip3 install docker-compose
sudo systemctl enable docker
sudo reboot
```

Build the mktxp Docker image

```bash
# Get the mktxp repository
git clone https://github.com/akpw/mktxp.git

# Go into the newly downloaded repo
cd mktxp

# Build the docker image
docker build . -t mktxp
```

Now get this repo and install all services:

```bash
# Clone this repo
git clone https://github.com/M0r13n/mikrotik_monitoring.git


# Go into the cloned directory
cd mikrotik_monitoring

# Let docker-compose do it's job
docker-compose up -d
```

You may need to adjust the following configuration files and add your own credentials for your router:

- `mktxp/mktxp.conf`


Done. You should now be able to open the Grafana dashboard on Port 3000 of your Raspberry Pi.

## Multiple Nodes

It is possible to monitor multiple (Mikrotik) devices. Just change add as many devices to `mktxp/mktxp.conf` as you want. 

## HTTPS

It is also possible to access the Grafana Dashboard over HTTPS.
Depending on your security requirements and/or threat model it might be a good idea to enable HTTPS.

Generate a self signed certificate for your domain:

`sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout ./nginx/nginx-selfsigned.key -out ./nginx/nginx-selfsigned.crt`

This command generates two files:
- the private key: `./nginx/nginx-selfsigned.key`
- the certificate file `./nginx/nginx-selfsigned.crt`

Both files need to be mapped as a volume to `/etc/nginx/ssl/`.

Then you also need to adjust the `docker-compose.yml` file:
- comment the line `./nginx/nginx.conf:/etc/nginx/conf.d/default.conf`
- and uncomment the four lines below

Finally you need to adjust the `nginx/nginx.conf.https` and adjust the `server_name` to your domain.
