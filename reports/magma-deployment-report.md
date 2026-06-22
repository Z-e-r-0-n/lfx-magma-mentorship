# Magma Docker Deployment Report
## 1. Objective

This report documents my process of deploying Magma using Docker inside a virtual machine as part of the LFX Magma mentorship task. The goal of this phase was to set up a working local Magma lab environment, bring up Orc8r and AGW services, register the AGW with Orc8r, and collect proof logs for the deployment.

The deployment was done as a hands-on learning exercise to understand Magma's Docker-based setup, service interactions, gateway registration flow, and common setup/debugging issues.

## 2. Environment
### 2.1 Host and VM Environment
Host machine: mediaserver3
Host OS: Ubuntu 26.04 LTS
VM software: KVM/QEMU with libvirt
VM name: magma-ubuntu20
VM OS: Ubuntu 20.04.6 LTS
VM CPU allocation: 6 vCPUs
VM RAM: 16 GB
VM disk: 120 GB qcow2
VM root filesystem: approximately 117 GB
Network mode: libvirt default NAT network
VM management IP: 192.168.122.146
Second VM interface IP: 192.168.122.139
### 2.2 Local Network Layout

The deployment used the following layout:

Laptop: Evangelion
  |
  | LAN
  v
Server / VM host: mediaserver3
  |
  | libvirt NAT
  v
VM: magma-ubuntu20
  |
  | Docker
  v
Local Orc8r + Docker-based AGW

The laptop was used for notes, reports, and repository tracking. The VM host ran the Ubuntu VM. The VM itself ran Docker, Orc8r, and AGW.

### 2.3 Docker and Magma Versions
Docker version: Docker version 28.1.1, build 4eba377
Docker Compose version: Docker Compose version v2.35.1
Magma repository: cloned from the Magma GitHub repository
Main local repo branch checked: master
AGW Docker installer/scripts used Magma v1.8
AGW Docker image version: 1.8.0
## 3. VM Preparation

A clean Ubuntu 20.04 VM was created on the mediaserver3 host using KVM/QEMU and libvirt. The VM was created from the Ubuntu 20.04 cloud image and configured using cloud-init.

Tools used:

qemu-system-x86
libvirt
virt-install
cloud-localds
Ubuntu 20.04 focal cloud image

The VM was allocated 6 vCPUs, 16 GB RAM, and a 120 GB qcow2 disk.

A second network interface was added to the VM because the Docker AGW setup expects two interfaces. Initially, the VM interfaces appeared as enp1s0 and enp7s0. After the AGW installer and reboot, these became eth0 and eth1.

Final interface state:

eth0: 192.168.122.146
eth1: 192.168.122.139
## 4. Docker Installation

Docker Engine and Docker Compose v2 were installed inside the Ubuntu 20.04 VM using Docker's official Ubuntu apt repository.

Verification commands used:

docker --version
docker compose version
docker run hello-world

The installation was successful, and hello-world ran correctly.

## 5. Magma Repository Setup

The Magma repository was cloned inside the VM:

git clone https://github.com/magma/magma.git
cd magma
git status

The repository was clean after cloning. The local checked branch was master.

Important directories used during deployment:

~/magma/orc8r/cloud/docker
/var/opt/magma/docker
/etc/magma
/var/opt/magma/configs
/var/opt/magma/certs
## 6. Orc8r Docker Deployment

The local Orc8r Docker deployment was done from:

~/magma/orc8r/cloud/docker

The correct deployment order was:

./build.py
./run.py

Running ./run.py before ./build.py caused Docker build context errors because required generated build context directories were not present yet.

After building and running Orc8r, the core Orc8r containers came up, including:

orc8r_controller_1
orc8r_nginx_1
orc8r_postgres_1
orc8r_maria_1
elasticsearch
fluentd

Orc8r certificates were generated inside the nginx/controller containers. The root CA and admin operator certificates were copied out for API access.

The local hostnames were mapped to 127.0.0.1, including:

controller.magma.test
bootstrapper-controller.magma.test
streamer-controller.magma.test
subscriberdb-controller.magma.test
metricsd-controller.magma.test
state-controller.magma.test
fluentd.magma.test
dispatcher-controller.magma.test

The Orc8r API was verified using the admin operator certificate:

curl -sk \
  --cert ~/orc8r-admin-certs/admin_operator.pem \
  --key ~/orc8r-admin-certs/admin_operator.key.pem \
  https://controller.magma.test:9443/

The API returned:

"hello"

This confirmed that the Orc8r REST API was reachable with certificate authentication.

## 7. AGW Docker Deployment

The Docker-based AGW deployment used the AGW Docker installer flow, but some manual fixes were required.

The AGW Docker compose directory was:

/var/opt/magma/docker

AGW services were started using:

sudo docker compose -f docker-compose.yaml up -d

Important AGW services included:

magmad
control_proxy
oai_mme
mobilityd
pipelined
subscriberdb
sessiond
state
directoryd
eventd
enodebd
sctpd
td-agent-bit

After fixes, the AGW Docker services were running and mostly healthy.

## 8. Gateway Registration Flow

The following objects were created through the Orc8r API:

Network: my-network
Tier: default
Gateway: my-gateway
Tenant ID: 1

The gateway hardware ID was:

b9fa217d-355d-49a7-b3e5-bc9747f87efa

The gateway registration flow required a tenant-level control_proxy to exist. The tenant was created with the network attached:

{
  "id": 1,
  "name": "local-tenant",
  "networks": ["my-network"]
}

The tenant control proxy was then configured at:

/magma/v1/tenants/1/control_proxy

After the tenant and control proxy were configured, AGW registration/check-in succeeded.

## 9. Issues Encountered and Resolutions
### 9.1 Orc8r Docker build/run order

Issue:

Running ./run.py before ./build.py caused Docker build context errors involving missing paths such as gomod, src, and configs.

Resolution:

Run:

./build.py
./run.py

in that order.

Potential improvement:

Add a documentation note clarifying this order.

### 9.2 Fluentd Docker build failed due Ruby gem dependency drift

Issue:

The Fluentd Docker build failed because newer Ruby gems required a newer Ruby runtime than the image provided.

Observed error class:

multi_json requires Ruby >= 3.2
current Ruby was 2.7.x

Resolution:

Compatible gem versions were pinned in the Fluentd Dockerfiles:

RUN gem install multi_json -v 1.15.0 --no-document
RUN gem install excon -v 1.2.5 --no-document

Potential improvement:

Pin compatible gem versions or update the Fluentd image.

### 9.3 AGW installer assumed specific interface names

Issue:

The AGW installer expected interface names such as ens5 and ens6, but the KVM/libvirt VM used:

enp1s0
enp7s0

Resolution:

The installer was patched locally to also rename:

enp1s0 -> eth0
enp7s0 -> eth1

Potential improvement:

Improve interface detection or document how to adapt interface names in non-cloud VM environments.

### 9.4 AGW installer used legacy docker-compose

Issue:

The AGW installer and upgrade script expected the legacy docker-compose command, while the VM had Docker Compose v2 as docker compose.

There was also risk in the upgrade script because it attempted broad Docker container operations that could affect unrelated containers, which is risky when Orc8r and AGW are running on the same lab VM.

Resolution:

The AGW services were managed manually using:

sudo docker compose -f docker-compose.yaml up -d

Potential improvement:

Update scripts or documentation for Docker Compose v2 and avoid stopping unrelated containers.

### 9.5 Minimal control_proxy.yml was insufficient

Issue:

A minimal control_proxy.yml caused AGW services to fail because required keys were missing.

Missing keys included:

gateway_cert
gateway_key
local_port
proxy_cloud_connections
nghttpx_config_location
sentry_url
sentry_sample_rate

Resolution:

A complete control proxy configuration was created in:

/etc/magma/control_proxy.yml
/var/opt/magma/configs/control_proxy.yml

Potential improvement:

Provide a complete local Docker AGW control_proxy.yml example in the docs.

### 9.6 AGW containers needed hostname mappings

Issue:

Even with host networking, AGW containers needed explicit hostname mappings for local Orc8r service names.

Resolution:

Hostname mappings were added for services such as:

controller.magma.test
bootstrapper-controller.magma.test
streamer-controller.magma.test
subscriberdb-controller.magma.test
metricsd-controller.magma.test
state-controller.magma.test
fluentd.magma.test
dispatcher-controller.magma.test

Potential improvement:

Document required hostname mappings for single-VM local Docker deployments.

### 9.7 Gateway registration required tenant and tenant control_proxy setup

Issue:

Gateway registration initially failed because no tenant-level control proxy was configured.

Observed failure:

could not get control-proxy from tenant with network ID my-network

Another confusion was that tenant_id must be numeric. The endpoint below does not work:

/magma/v1/tenants/my-network/control_proxy

Correct endpoint:

/magma/v1/tenants/1/control_proxy

Resolution:

Created tenant ID 1, attached my-network to it, and set the tenant control proxy before completing AGW registration.

Potential improvement:

Update gateway registration docs to explicitly mention numeric tenant ID and tenant control_proxy prerequisite.

### 9.8 Gateway creation required an upgrade tier

Issue:

Gateway creation failed without a tier.

Observed error:

tier in body is required

Resolution:

Created a default tier first, then created the gateway using:

"tier": "default"

Potential improvement:

Add a gateway creation API example that includes the required tier.

### 9.9 Network creation required DNS config

Issue:

Network creation failed without a dns field.

Observed error:

dns in body is required

Resolution:

Included a minimal DNS config:

"dns": {
  "enable_caching": false,
  "local_ttl": 60,
  "records": []
}

Potential improvement:

Add a complete minimal network creation JSON example to the docs.

## 10. Final Verification

After registration and service restart, AGW service status was captured with:

sudo docker compose -f docker-compose.yaml ps

The AGW containers were running, and most services were marked healthy.

The magmad logs showed successful check-in:

Checkin Successful! Successfully sent states to the cloud!

The control_proxy logs showed successful HTTP/2 forwarding to Orc8r services after registration, including 200 responses for streamer and state service calls.

The gateway status endpoint returned HTTP 200:

curl -sk \
  --cert ~/orc8r-admin-certs/admin_operator.pem \
  --key ~/orc8r-admin-certs/admin_operator.key.pem \
  https://controller.magma.test:9443/magma/v1/networks/my-network/gateways/my-gateway/status

The response included the gateway hardware ID, machine information, network interfaces, routing table, and system status.
