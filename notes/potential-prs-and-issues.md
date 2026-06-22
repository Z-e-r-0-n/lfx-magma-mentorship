# Potential PRs and Issues Found During Magma Docker Deployment

## Current Status

A local Magma Docker lab deployment was completed using:

* Ubuntu 20.04 VM on KVM/libvirt
* Docker Engine and Docker Compose v2
* Local Orc8r Docker deployment
* Docker-based AGW deployment
* Local Orc8r REST API
* Tenant, LTE network, tier, gateway, and AGW registration flow

Final proof showed:

* AGW Docker services running
* `magmad` check-in successful
* `control_proxy` successfully forwarding streamer/state calls
* Gateway status visible through Orc8r API
* Gateway hardware ID reported by Orc8r

## Problems Found

### 1. Orc8r Docker build/run order was unclear

Running `./run.py` before `./build.py` caused build-context errors involving missing paths like `gomod`, `src`, and `configs`.

Observed resolution:

```bash
cd orc8r/cloud/docker
./build.py
./run.py
```

Potential PR:

* Documentation PR clarifying that `./build.py` must be run before `./run.py`.
* Add a troubleshooting note for missing Docker build context errors.

Priority: High
Difficulty: Low
Best first PR: Yes

---

### 2. Fluentd Docker build failed due Ruby gem dependency drift

The Fluentd image used an older Ruby version, while newer Ruby gems required a newer Ruby runtime.

Observed error class:

```text
multi_json requires Ruby >= 3.2
current Ruby was 2.7.x
```

Workaround used:

```dockerfile
RUN gem install multi_json -v 1.15.0 --no-document
RUN gem install excon -v 1.2.5 --no-document
```

Potential PR:

* Pin compatible gem versions in Fluentd Dockerfiles.
* Or update the Fluentd base image if appropriate.
* Or document the workaround if the Docker setup is not actively maintained.

Priority: Medium
Difficulty: Medium
Best first PR: Maybe, but needs maintainer confirmation

---

### 3. AGW Docker installer assumes specific interface names

The AGW installer expected cloud-style interface names such as `ens5` and `ens6`.

In the KVM/libvirt VM, interfaces were:

```text
enp1s0
enp7s0
```

Workaround used:

* Patched installer to also rename:

```text
enp1s0 -> eth0
enp7s0 -> eth1
```

Potential PR:

* Improve installer interface detection.
* Or document how to adapt interface names for non-cloud KVM/libvirt VMs.

Priority: Medium
Difficulty: Medium
Best first PR: Documentation version yes; code version later

---

### 4. AGW installer uses legacy `docker-compose`

The AGW install flow/script expected:

```bash
docker-compose
```

But the VM used modern Docker Compose v2:

```bash
docker compose
```

Also, the AGW upgrade script attempted broad container operations that could affect unrelated containers, which is risky when Orc8r and AGW run on the same lab VM.

Potential PR:

* Update scripts/docs to support Docker Compose v2.
* Avoid stopping unrelated Docker containers.
* Document that Orc8r and AGW should ideally be separated if using scripts that manage Docker globally.

Priority: High
Difficulty: Medium
Best first PR: Documentation first, script fix later

---

### 5. Minimal `control_proxy.yml` was insufficient

A minimal `control_proxy.yml` caused AGW services to fail because required keys were missing.

Missing/required keys included:

```text
gateway_cert
gateway_key
local_port
proxy_cloud_connections
nghttpx_config_location
sentry_url
sentry_sample_rate
```

Workaround used:

* Created a fuller `control_proxy.yml` in:

```text
/etc/magma/control_proxy.yml
/var/opt/magma/configs/control_proxy.yml
```

Potential PR:

* Add a complete Docker AGW `control_proxy.yml` example.
* Document required fields for local Docker Orc8r + Docker AGW setup.

Priority: High
Difficulty: Low
Best first PR: Yes

---

### 6. AGW containers needed hostname mappings

AGW containers using host networking still needed explicit hostname mappings for local Orc8r service names.

Hostnames involved:

```text
controller.magma.test
bootstrapper-controller.magma.test
streamer-controller.magma.test
subscriberdb-controller.magma.test
metricsd-controller.magma.test
state-controller.magma.test
fluentd.magma.test
dispatcher-controller.magma.test
```

Workaround used:

* Added host mappings / compose `extra_hosts` entries so AGW containers could resolve local Orc8r service names.

Potential PR:

* Document required `/etc/hosts` or compose hostname setup for single-VM local Docker deployments.
* Add `dispatcher-controller.magma.test` to local Docker hostname notes.

Priority: Medium
Difficulty: Low
Best first PR: Yes

---

### 7. Gateway registration required tenant and tenant control_proxy setup

Gateway registration initially failed because tenant-level `control_proxy` was not configured.

Observed failure:

```text
could not get control-proxy from tenant with network ID my-network
```

Then another issue was found:

```text
/magma/v1/tenants/my-network/control_proxy
```

does not work because `tenant_id` must be numeric.

Correct flow:

```text
1. Create network: my-network
2. Create tier: default
3. Create gateway: my-gateway
4. Create tenant ID 1
5. Attach my-network to tenant 1
6. Set /magma/v1/tenants/1/control_proxy
7. Register AGW
```

Potential PR:

* Improve gateway registration docs.
* Explicitly state that tenant ID is numeric.
* Add the tenant/control_proxy prerequisite before `register.py`.

Priority: Very High
Difficulty: Low
Best first PR: Strong candidate

---

### 8. Gateway create endpoint required tier

Gateway creation failed without a tier.

Observed error:

```text
tier in body is required
```

Resolution:

* Create tier first:

```text
default
```

* Then create gateway with:

```json
"tier": "default"
```

Potential PR:

* Add gateway creation API example including tier.
* Add note that tier must exist before gateway creation.

Priority: Medium
Difficulty: Low
Best first PR: Yes

---

### 9. Network creation required DNS config

Network creation failed until `dns` was included.

Observed error:

```text
dns in body is required
```

Resolution:

```json
"dns": {
  "enable_caching": false,
  "local_ttl": 60,
  "records": []
}
```

Potential PR:

* Add complete minimal network creation JSON example.
* Mention required `dns` field.

Priority: Medium
Difficulty: Low
Best first PR: Yes

---

## Best First PR Candidates

### Candidate 1: Gateway registration docs improvement

Why:

* Directly based on real deployment pain.
* Low-risk documentation change.
* Useful for new contributors.
* Connects strongly to mentorship deliverables.

Include:

* Create tenant before registration.
* Tenant ID is numeric.
* Tenant must include the network ID.
* Set `/magma/v1/tenants/<tenant_id>/control_proxy`.
* Then run `register.py`.

Recommended as first PR.

---

### Candidate 2: Orc8r Docker build/run order docs

Why:

* Very simple.
* Easy to explain.
* Prevents common beginner deployment failure.

Include:

```bash
./build.py
./run.py
```

and mention that `run.py` before `build.py` may fail due missing build context.

Good first PR.

---

### Candidate 3: Full local Docker deployment troubleshooting page

Why:

* Combines several issues into one useful doc.
* Could be accepted as a new troubleshooting section.

Include:

* Docker Compose v2 note
* Fluentd gem issue
* Hostname mappings
* control_proxy full config
* tenant/control_proxy registration prerequisite
* gateway tier requirement
* DNS field requirement

Useful, but maybe too large for first PR.

---

## Recommended PR Order

1. Small docs PR: gateway registration tenant/control_proxy prerequisite
2. Small docs PR: Orc8r Docker build order
3. Small docs PR: minimal API examples for network/tier/gateway
4. Larger troubleshooting doc for local Docker Orc8r + AGW
5. Script/code PRs only after discussing with maintainers

## Notes for Deployment Report

Mention these as “issues encountered and resolutions,” not necessarily all as PRs.

Important proof files:

```text
logs/agw-compose-ps-after-registration.txt
logs/magmad-after-registration.log
logs/control-proxy-after-registration.log
logs/gateway-status-after-registration.json
```
