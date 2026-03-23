<p align="center">
<img src="https://github.com/crowdsecurity/cs-cloudflare-worker-bouncer/raw/main/docs/assets/crowdsec_cloudfare.png" alt="CrowdSec" title="CrowdSec" width="280" height="300" />
</p>
<p align="center">
<img src="https://img.shields.io/badge/build-pass-green">
<img src="https://img.shields.io/badge/tests-pass-green">
</p>
<p align="center">
&#x1F4A0; <a href="https://hub.crowdsec.net">Hub</a>
&#128172; <a href="https://discourse.crowdsec.net">Discourse</a>
</p>

> **This is a fork of [crowdsecurity/cs-cloudflare-worker-bouncer](https://github.com/crowdsecurity/cs-cloudflare-worker-bouncer).**
> It adds configurable names for the Cloudflare resources the bouncer creates, allowing multiple bouncer instances to coexist on the same Cloudflare account without conflict. See [Running Multiple Instances](#running-multiple-instances-on-the-same-cloudflare-account) for details.

---

⚠️ Due to the heavy usage of KV and Workers quotas on Cloudflare free plan, the paid Cloudflare Worker Plan is recommended for this remediation component.

⚠️ Follow this guide to [test on free Plan](https://doc.crowdsec.net/u/bouncers/cloudflare-workers/#appendix-test-with-cloudflare-free-plan)

⚠️ Make sure to properly set up [The worker route fail mode](https://doc.crowdsec.net/u/bouncers/cloudflare-workers#setting-up-the-worker-route-fail-mode)

# CrowdSec Cloudflare Worker

A remediation component for Cloudflare.

## How does it work

This remediation component deploys Cloudflare Worker in front of a Cloudflare Zone/Website, which checks if incoming IP addresses are present in a KV store and takes necessary remedial actions. It also periodically updates the KV store with CrowdSec LAPI's decisions.

## Documentation

Configuration and usage follow the [official upstream documentation](https://docs.crowdsec.net/docs/next/bouncers/cloudflare-workers). The changes introduced by this fork are documented below.

---

## Running Multiple Instances on the Same Cloudflare Account

### The problem

The upstream bouncer hardcodes the names of every Cloudflare resource it creates. If you run two bouncer instances sharing the same Cloudflare account — for example, one protecting a VPS and another protecting a separate server — they will conflict:

| Resource | Upstream hardcoded name |
|---|---|
| Remediation worker script | `crowdsec-cloudflare-worker-bouncer` |
| Workers KV namespace | `CROWDSECCFBOUNCERNS` |
| D1 database (metrics) | `CROWDSECCFBOUNCERDB` |
| Decisions sync worker (autonomous mode) | `crowdsec-decisions-sync-worker` |

The second instance to start will overwrite the first instance's KV data, or fail entirely with an error like:

```
unable to cleanup existing workers: remove namespace: 'namespace has associated scripts: crowdsec-cloudflare-worker-bouncer' (10052)
```

### The fix

This fork exposes all four names as configurable fields under `cloudflare_config.worker`. All fields are optional — if omitted, they fall back to the same upstream defaults, so existing single-instance deployments require no config changes.

```yaml
cloudflare_config:
  worker:
    script_name: ""                   # default: crowdsec-cloudflare-worker-bouncer
    kv_namespace_name: ""             # default: CROWDSECCFBOUNCERNS
    d1_db_name: ""                    # default: CROWDSECCFBOUNCERDB
    decisions_sync_script_name: ""    # default: crowdsec-decisions-sync-worker (only used with -S)
```

### Example: two instances on the same account

**Instance 1** — uses all defaults, no config changes needed:

```yaml
cloudflare_config:
  worker: {}   # or omit the worker block entirely
```

**Instance 2** — all four names must differ from instance 1:

```yaml
cloudflare_config:
  worker:
    script_name: "crowdsec-bouncer-server2"
    kv_namespace_name: "CROWDSECCFBOUNCERNS2"
    d1_db_name: "CROWDSECCFBOUNCERDB2"
    decisions_sync_script_name: "crowdsec-decisions-sync-worker-2"
```

> **Note:** `decisions_sync_script_name` only matters if you use autonomous mode (`-S` flag). If you run in daemon mode, this field is never deployed and can be left at its default.

### Cleanup isolation

When you run `-d` (delete) on one instance, it only removes resources matching its own configured names. The other instance's resources are left untouched.

---

## Installation

Download the latest release for your architecture from the [Releases page](https://github.com/ruivza/cs-cloudflare-worker-bouncer/releases).

**Linux amd64 (most servers):**

```bash
mkdir ~/crowdsec-cf-bouncer && cd ~/crowdsec-cf-bouncer
wget https://github.com/ruivza/cs-cloudflare-worker-bouncer/releases/latest/download/crowdsec-cloudflare-worker-bouncer-linux-amd64.tgz
tar xzf crowdsec-cloudflare-worker-bouncer-linux-amd64.tgz
cd crowdsec-cloudflare-worker-bouncer-v*
sudo ./install.sh
```

**Linux arm64 (e.g. Oracle Cloud ARM, Raspberry Pi):**

```bash
mkdir ~/crowdsec-cf-bouncer && cd ~/crowdsec-cf-bouncer
wget https://github.com/ruivza/cs-cloudflare-worker-bouncer/releases/latest/download/crowdsec-cloudflare-worker-bouncer-linux-arm64.tgz
tar xzf crowdsec-cloudflare-worker-bouncer-linux-arm64.tgz
cd crowdsec-cloudflare-worker-bouncer-v*
sudo ./install.sh
```

`install.sh` installs the binary and places a default config at `/etc/crowdsec/bouncers/crowdsec-cloudflare-worker-bouncer.yaml`.

**Configure and start:**

```bash
sudo vim /etc/crowdsec/bouncers/crowdsec-cloudflare-worker-bouncer.yaml
sudo systemctl start crowdsec-cloudflare-worker-bouncer
sudo systemctl status crowdsec-cloudflare-worker-bouncer
```

For full configuration options, follow the [official upstream documentation](https://docs.crowdsec.net/docs/next/bouncers/cloudflare-workers).
