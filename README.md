# SysOps Test — PoC Web Infrastructure

Proof of concept for a web application deployment lifecycle, provisioned with Ansible on a Vagrant-managed VM.

An HAProxy load balancer routes traffic by URL path:

- `/api` requests are served by EchoServer containers
- `/statics` requests are served by Nginx containers

Everything is deployed with a single `vagrant up`.

---

## Prerequisites

Only two tools are required on the host machine. **Ansible is not installed on the host** — it runs inside the VM (see [Design decisions](#design-decisions)).

| Tool | Minimum | Verified with |
|---|---|---|
| Vagrant | 2.4 | 2.4.9 |
| VirtualBox | 7.0 | 7.2.14 r174565 |
| Git | 2.x | 2.55.0 |

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
git clone <repository-url>
cd sysops-test
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

<!-- TODO: sticky routing and scaling checks once part 3 lands -->

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

<!-- TODO: remaining design decisions — shared Docker network, compose templating, balance uri -->

---

## Known limitations

<!-- TODO: to be completed -->