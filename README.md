# Eth Server

This project is inspired by [Eth Docker](https://eth-docker.net/) and they are complementary. Whereas Eth Docker focuses on installing the Ethereum clients, this project focuses on setting up the linux server itself, on which Eth Docker will run. Eth Server uses ansible to automate SSH hardening, VPN setup, chrony, monitoring etc. Namely

1. eth-server sets up your staking server
2. then eth-docker sets up your ethereum clients

So far, this project works for me and I feel like sharing it. My current motivation is to inspire others, but not necessary to extend it so that it meets everyone's custom needs and setup.

Eth Docker is a real utility software. This project here is mainly hacked together.

## Principles

1. keep it simple: it should be easy for most people to onboard to the project and tweak it to their needs.
2. it is opinionated: there could be a million of combination of OS and software to create a server for staking. As much as we want diversity in the staking ecosystem, this project only focuses on a simple and rather mainstream solution.
3. production ready: although simple, this ansible playbook should be reliable to setup a secured server, ready for staking.

## Scope

This project is meant for solo stakers with a single server.

Out of scope are:
* multi-tenant setup
* managing a fleet of N servers

## Why this project

I don't enjoy configuring my server. And I always forget what I've done and how. So I want to automate and document those mundane tasks. Moreover, if my server dies, I want to be able to recreate it within minutes, and in a deterministic and reproducible fashion. Ansible was designed for that !

## Architecture

This ansible playbook does the following things

1. it upgrades packages.
2. it installs Fail2Ban.
3. it installs UFW and sets it up, only allowing the SSH, execution and consensus client ports.
4. it hardens the SSH config.
5. it updates the chrony config to improve time synchronization. See reddit discussion [missing attestations, chrony and time sync drift](https://www.reddit.com/r/ethstaker/comments/17n3ffp/missing_attestations_chrony_and_time_sync_drift/).
6. it installs a [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/) that is used to SSH into the server from the public internet without having to expose the SSH port.
7. it updates the service `systemd-networkd-wait-online` to avoid blocking the server at boot time (at least in my case).
8. it downloads [eth-docker](https://eth-docker.net/) and uses it to install docker (making use of those great scripts).
9. it installs a poor man heartbeat script as a systemd daemon to send heartbeats to Grafana Cloud OnCall every 15 seconds.
10. it installs a [Grafana Agent](https://grafana.com/docs/agent/latest/) to monitoring the server (logs and metrics, including docker containers). It also takes care of setting up the [docker integration](https://grafana.com/docs/grafana-cloud/monitor-infrastructure/integrations/integration-reference/integration-docker/).

### Why Cloudflare Tunnel ?

It works. But there are other alternatives such a [Tailscale](https://tailscale.com/). The beauty of those tools is that they are very simple to install and work without exposing any ports to the public internet, thus greatly reducing the attack surface of the server compared to running a VPN on the server. On the other hand, you're trusting a 3rd party. It works for me.

### Why Grafana Agent ?

I want to keep my monitoring stack decoupled from docker (if docker fails for example), and I want to keep it super simple. I also want to keep my monitoring stack decoupled from eth-docker. Although eth-docker offers monitoring as part of its config, I prefer to reduce its scope to only managing my ethereum clients. Lastly, instead of running many components such prometheus, loki and other tools, the grafana agent ticks all the boxes for me.

### Why heartbeat script ?

Sometimes, I still miss attestations, and that's frustrating. So I've setup probes both ways:
* Grafana Cloud is pinging my clients with TCP probes every minutes (synthetic monitoring)
* My server is sending heartbeats to Grafana Cloud OnCall every 15s

It is useful to correlate network issues, or discarding them. Which points to the direction that my server is very reliable, but sometimes not so much my ISP... This is residential connection and the ISP doesn't offer the same guarantees as for businesses. Not much I can do. Nonetheless, still happy to see my heartbeat healthy.

## Prerequisites

You don't need most of the below to test the ansible script locally. On your laptop, Vagrant is used to spin up Ubuntu in a VM, that ansible uses to test the playbooks against. The below points are for the real staking server. For locally testing on your laptop, feel free to skip this section.

Laptop
* [ ] install [Vagrant](https://developer.hashicorp.com/vagrant/docs/installation)
* [ ] install [VirtualBox](https://www.virtualbox.org/) to be used by Vagrant
* [ ] install [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)
* [ ] install [Cloudflared client](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/downloads/)

Internet Box
* [ ] either get a static public IP for your server, or set up a dynamic name resolution (most box should offer that). My ISP doesn't offer static IP, but I don't have a dynamic name of the form `<my-sub-domain>.<my-isp-domain>.com`. That will be used to create TCP health checks in Grafana Cloud. If you don't plan on setting up health checks, then you don't need to setup a dynamic name resolution in your home router.
* [ ] setup port-forwarding for your execution (default `30303`) and consensus (default `9000`) client ports for both UDP and TCP. If you use Cloudflare Tunnel or Tailscale, you shouldn't need to expose any other ports: i.e. my SSH port 22 is NOT exposed.

Domain name
* [ ] buy a domain name that will be used to connect to your server from the public internet through the Cloudflare Tunnel. Use any registrar of your choice or cloudflare directly.

Cloudflare
* [ ] Create a cloudflare account
* [ ] Onboard your domain (or buy the domain directly from cloudflare)
* [ ] Create 2 Tunnels (one for prod, and one for local) in the cloudflare console and copy-paste the tokens into inventory files in `inventories` under the key `cloudflare.token`.

Grafana Cloud
* [ ] create an account (free tier)
* [ ] add your API key to the file `inventories/prod.yaml` at the key `grafana.grafana_cloud_api_key`.
* [ ] setup metrics endpoint and update the keys `grafana.metrics_username` and `grafana.prometheus_url` in config file `inventories/prod.yaml`.
* [ ] setup logs endpoint and update the keys `grafana.logs_username` and `grafana.loki_url` in config file `inventories/prod.yaml`.
* [ ] define a name for your server with the key `grafana.instance_name` in config file `inventories/prod.yaml`.
* [ ] create TCP probes (synthetic) on your EL and CL client ports, and using the dynamic name of your ISP internet box.

Server
* [ ] get your hardware.
* [ ] flash a USB stick with Ubuntu 22.04 LTS server (or the OS of your choice).
* [ ] get yourself a keyboard and screen and install the OS. Create strong passwords, for the BIOS also recommended. Saved them carefully (password manager ideally).
* [ ] install the OS.
* [ ] make sure you have correctly expanded your volumes and that the OS sees the full capacity of your disks.
* [ ] make sure you are using your SSH key when SSHing into the server: if you're using the username/password and run the ansible playbook, you will be locked out of the server, because password authentication is disabled.

Laptop
* [ ] create an SSH key pair. And upload the public key to the server.
* [ ] edit your ssh config `~.ssh/config` with the server host and the key to be used.

## Config

Configuration is stored in `inventories/`. Because it contains secrets, a low tech solution is simply to ignore those files in `.gitignore`. Copy the file `example.yaml` to `prod.yaml` and `local.yaml`:

* `inventories/local.yaml` is used with vagrant, to test things locally.
* `inventories/prod.yaml` is used with your server, the real thing.

if you prefer not to maintain your secrets in files (recommended), you can also pass the values with the flag `--extra-vars`. For example

```shell
ansible-playbook -i inventories/prod.yaml playbooks/main.yaml --extra-vars='{
    "cloudflare": {
        "secret_token": "<your_secret_token>"
    }
}'
```

it also works with env vars

```shell
CLOUDFLARE_TOKEN="<your_secret_token>"

ansible-playbook -i inventories/prod.yaml playbooks/main.yaml --extra-vars='{
    "cloudflare": {
        "secret_token": '"${CLOUDFLARE_TOKEN}"'
    }
}'
```

the flag `--extra-vars` will always overwrite variables defined elsewhere.

## Get started

You can test everything on your laptop (locally) thanks to Vagrant. From all the prerequisites above, you only need

* the grafana api key and usernames and config. It's actually cool to already see metrics and logs flowing into grafana cloud from the vagrant VM.
* the cloudflare token for the tunnel. You will also see the tunnel showing up as "UP" in the UI.

If you don't want the 2 points above, then comment out (or delete) that code in the ansible playbook.

Edit the local config file `inventories/local.yaml` accordingly to your setup. Good practices is to create distinct api key for local and prod configs.

clone the project on your laptop

```shell
git clone git@github.com:salanfe/eth-server.git
```

and add your api keys to the local config file `inventories/local.yaml`.

Then, start a virtual ubuntu server on your laptop with vagrant

```shell
vagrant up
```

once the virtual server is up and running, run the playbook with the local config

```shell
ansible-playbook -i inventories/local.yaml playbooks/main.yaml --diff
```

ssh into the virtual server and double-check the config, and see by yourself the changes applied by ansible

```shell
vagrant ssh
```

Feel free to edit the playbook `playbooks/main.yaml` as much as you want: e.g. you can remove grafana and cloudflare.

## Setup your server

Create new api keys for Cloudflare Tunnel and Grafana Cloud, that are dedicated to "prod", which is your real server for staking. Edit the file `inventories/prod.yaml` accordingly.

do a dry-run first

```shell
ansible-playbook -i inventories/prod.yaml playbooks/main.yaml --ask-become-pass --diff --check
```

carefully check the above. Then run the playbook

```shell
ansible-playbook -i inventories/prod.yaml playbooks/main.yaml --ask-become-pass --diff
```

if you want to dynamically pass secrets, you can use the flag `--extra-vars` (see config section above).

## Improvements

below are possible ideas for future improvements

* [ ] add Tailscale and/or a VPN server as alternatives to Cloudflare Tunnel. Having both Tailscale and Cloudflare Tunnel installed can offer a fail-safe alternative.
* [ ] fine tune the grafana agent config: scraping logs, relabeling, etc.
* [ ] get a definitive solution for the chrony config. Using Google NTP servers works so far, but there's probably a better solution. Additionally, the consensus protocol don't use leap smear time like the Google NTP servers [source](https://github.com/ethereum/consensus-specs/blob/36f0bb0ed62b463947fda97f42f8ddebc9565587/specs/phase0/fork-choice.md#fork-choice).
* [ ] add more tests and validation
* [ ] get more eyes on the ansible playbook
* [ ] terraform 3rd parties (grafana cloud, cloudflare). Although, those are less critical than the server itself, and doing things by hand is probably fine: as it's mainly a one-time setup. Nonetheless, having terraform could possibly speed up onboarding of new joiners (to be balanced with the extra complexity of introducing yet another tool).
* [ ] use Ansible to manage the eth-docker `.env` file. But there should be a reconciliation logic going both ways, making it probably more error prone than necessary. Nonetheless, if the server dies, having a copy of the `.env` file is also a good thing.
* [ ] better solution to manage secrets (e.g. I like working with Google Cloud... add a script to pull secret from Google Secret Manager). E.g. see below

```shell
GCP_PROJECT_ID="<ID of your GCP project>"
GCP_SECRET_NAME="<name of the secret>"

ansible-playbook -i inventories/prod.yaml playbooks/main.yaml --extra-vars='{
    "cloudflare": {
        "secret_token": '"$(gcloud secrets versions access latest --secret=${GCP_SECRET_NAME} --project=${GCP_PROJECT_ID})"'
    }
}'
```

## My Hardware

Currently running a node will the following hardware

* Case: Silverstone SST-SG05BB-Lite (Mini ITX, Mini DTX)
* Motherboard: Supermicro X12STL-IF ([reference](https://www.supermicro.com/en/products/motherboard/x12stl-if))
* SSD: Samsung 980 Pro with Heatsink (2000 GB, M.2 2280)
* RAM: Crucial DDR4 ECC UDIMM 2Rx8 3200 (1 x 32GB, 3200 MHz, DDR4-RAM, DIMM)
* CPU: Intel Core i3-10105F (LGA 1200, 3.70 GHz, 4 -Core)
* PSU: Cooler Master V Series SFX (750 W)

## My Clients

Running Besu and Teku to contribute with the minority clients.

## Credits and Inspiration

* https://github.com/eth-educators/eth-docker: no need to introduce that one.
* https://github.com/CryptoManufaktur-io/backend-ansible: the people being eth-docker also maintain an ansible repo for running nodes. Much more advanced (but also complex), this is worth giving a look.

## How to Contribute

Submit an issue or open a merge request.

## Buy me a coffee

Here's my ethereum address if you feel like buying me a coffee :coffee::heart: `0x48707A2D8cf862E14401e4DeDB94cD6b97bd67d7`
