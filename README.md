# SysOps Test — PoC Web Infrastructure

Proof of concept for a web application deployment lifecycle, provisioned with Ansible on a Vagrant-managed VM.

An HAProxy load balancer routes traffic by URL path:

- `/api` requests are served by EchoServer containers
- `/statics` requests are served by Nginx containers

Everything is deployed with a single `vagrant up`.

---

## Prerequisites

Only three tools are required on the host machine. **Ansible is not installed on the host** — it runs inside the VM (see [Design decisions](#design-decisions)).

| Tool | Minimum | Verified with |
|---|---|---|
| Vagrant | 2.4 | 2.4.9 |
| VirtualBox | 7.0 | 7.2.14 r174565 |
| Git | 2.x | 2.55.0 |

After installing VirtualBox, a system reboot is needed.
Roughly 4 GB of free RAM and 5 GB of disk space are needed for the guest VM and container images.

### Verified environment

The following stack was validated end to end before development started:

**Host**

- Windows 11, VirtualBox running with hardware virtualization enabled (nested paging, KVM paravirtualization)
- Base box `ubuntu/jammy64` v20241002.0.0 pre-fetched into the local Vagrant cache
- VM boot, SSH connectivity and synced folder mount (`/vagrant`) confirmed

**Guest (provisioned automatically)**

- Ubuntu 22.04 LTS (kernel 5.15)
- `ansible-core` 2.17.14, installed inside the VM by the `ansible_local` provisioner via pip
- `community.docker` collection 5.2.2, installed from `requirements.yml`

---

## Quick start

```bash
git clone https://github.com/zorb0l1in0/Web_Infra_Automation
cd Web_Infra_Automation
vagrant up
```

That is the whole setup. The provisioner installs Docker Engine and Compose v2 inside the VM, then deploys the three services.

---

## Verification

All checks run from the host against the forwarded port.

```bash
# /api is routed to the EchoServer backend
curl -s http://localhost:8080/api | grep "real path"

# /statics is routed to the Nginx backend
curl -s http://localhost:8080/statics

# prefix match: any path under /statics reaches the same backend
curl -s http://localhost:8080/statics/foo/bar

# unmatched paths are rejected — there is no default_backend
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080/other
```

Expected output:

| Request | Response |
|---|---|
| `/api` | `real path=/api` (EchoServer echoes the request) |
| `/statics` | `Response from Server 0` |
| `/statics/foo/bar` | `Response from Server 0` |
| `/other` | `503` |

On Windows PowerShell, use `curl.exe` — the bare `curl` is an alias for `Invoke-WebRequest` and does not accept these flags.

### Load balancing behaviour

The two backends deliberately use different algorithms, so they behave differently under repeated requests.

```bash
# Round robin across the EchoServer replicas — the hostname rotates
for i in $(seq 6); do curl -s localhost:8080/api | grep ^Hostname; done

# Sticky routing — the same path always lands on the same Nginx replica
for i in $(seq 6); do curl -s localhost:8080/statics/foo; done

# Different paths hash to different replicas
for p in alpha beta gamma delta epsilon; do
  echo -n "$p -> "; curl -s localhost:8080/statics/$p
done
```

Observed with `echo_replicas: 3` and `nginx_replicas: 3`:

- `/api` cycles through all three EchoServer containers in order and repeats.
- `/statics/foo` returns `Response from Server 1` on every request.
- The five sample paths spread across all three replicas (`2, 3, 1, 2, 3`), and the mapping is stable across runs.

### Scaling

Replica counts live in `group_vars/all.yml`. Change either value and re-provision — no template or task edits required.

```bash
# edit nginx_replicas, then:
vagrant provision
vagrant ssh -c "docker ps --format '{{.Names}}'"
```

Scaling down removes the surplus containers rather than leaving them orphaned.

---

## Repository layout

```
.
├── Vagrantfile                VM definition and provisioner configuration
├── playbook.yml               Entry point: ordered list of roles
├── requirements.yml           Ansible collections installed at provisioning time
├── group_vars/
│   └── all.yml                Shared variables (paths, network name, replica counts)
└── roles/
    ├── docker/                Docker Engine, Compose v2, shared network
    ├── echo/                  EchoServer stack — serves /api
    ├── statics/               Nginx stack — serves /statics
    └── haproxy/               Load balancer — the only published port
```

The playbook holds no tasks of its own: it only declares which roles run and in which order. `haproxy` runs last so its health checks find the backends already up. Each role is self-contained, owning its tasks, templates and handlers.

Compose files are not checked in as static files. Each role ships a `docker-compose.yml.j2` template that Ansible renders into `/opt/poc/<service>/` inside the VM, which is what makes the replica counts configurable.

---

## Guest access

The VM forwards guest port 80 to host port 8080, so the load balancer is reachable at `http://localhost:8080` from the host. Application containers are not published to the host at all — HAProxy is the only entry point.

```bash
vagrant ssh          # shell into the VM
vagrant provision    # re-run the playbook without recreating the VM
vagrant destroy -f   # tear everything down
```

---

## Design decisions

### Ansible runs inside the VM (`ansible_local`)

Ansible does not support Windows as a control node, so the standard `ansible` provisioner would restrict this project to POSIX hosts. The `ansible_local` provisioner installs Ansible inside the guest and executes the playbook there, reading it from the `/vagrant` synced folder.

The benefit is that the host only needs Vagrant and VirtualBox: the repository behaves identically on Windows, macOS and Linux, with no control-node setup step.

The trade-off is that the SSH-based, agentless connection model normally used in production is not exercised here. For a single-node PoC the portability gain outweighs that.

`ansible-core` is pinned and installed via pip rather than taken from the Ubuntu repositories, because the distribution package on Jammy lags behind the version required by recent `community.docker` releases.

### Three Compose projects on one shared Docker network

The test asks for three separate Compose deployments, so the network they share is created once by the `docker` role and referenced as `external: true` by all three. Without this, each project would create its own isolated network and HAProxy could never reach the backends.

Sharing a network means containers resolve each other by name through Docker's embedded DNS: HAProxy points at `echo-1:8080` and `nginx-1:80` with no hardcoded IPs and no dependency on start-up order.

### HAProxy is the only published port

Application containers publish nothing to the host. The single `80:80` mapping on HAProxy, forwarded to host port 8080 by Vagrant, is the only way in. Backends are reachable exclusively from inside the Docker network, which keeps the exposed surface to one process.

### Configuration changes trigger a restart

Both the Nginx config and `haproxy.cfg` are bind-mounted read-only. Neither process re-reads its configuration on its own, so the roles notify a handler that restarts the stack whenever a rendered template changes. Without it, editing a template and re-running the playbook would silently have no effect.

### Path affinity via URI hashing, not session state

The `/statics` backend uses `balance uri` with `hash-type consistent`. HAProxy hashes the request URI and maps it deterministically onto a server, so a given path always reaches the same replica — without cookies, session tables or any shared state between requests.

`hash-type consistent` changes how that hash becomes a choice. The default map-based mode divides the hash by the number of servers, so adding or removing one reshuffles nearly every path-to-server assignment. Consistent hashing remaps only the minimum necessary fraction, which in production means a scale-up does not invalidate every replica's local cache at once.

The `/api` backend stays on `roundrobin`. EchoServer is stateless and the brief asks for no affinity there, so even distribution is the better fit. The two algorithms coexist because the two workloads have different requirements, not because one is generally better.

### Replica counts are variables, not duplicated code

Both stacks render their Compose file from a single Jinja2 template that loops over a replica count. Nginx additionally renders one config file per instance, injecting the loop index as the server id, so each replica reports which one answered. Going from one replica to five is a one-line change in `group_vars/all.yml`.

---

## Known limitations

This is a proof of concept and the following are deliberate simplifications, not oversights.

- **`nginx:latest` is not pinned.** The brief specifies that tag; production would pin a digest for reproducibility.
- **No TLS.** The load balancer terminates plain HTTP only.
- **Health checks are TCP-only.** `check` verifies the port accepts connections, not that the application is healthy. `option httpchk` would be the production choice.
- **HAProxy resolves backend names once, at startup.** There is no `resolvers` section, so a container recreated with a different IP would not be picked up until HAProxy restarts. In this project every topology change also rewrites `haproxy.cfg` and triggers the restart handler, so the gap never opens — but that is a property of the workflow, not a guarantee of the design.
- **Single host, no high availability.** HAProxy is itself a single point of failure.
- **No resource limits, log aggregation or secret management.**
- **Scale-down is not graceful.** Surplus containers are removed without draining in-flight connections.