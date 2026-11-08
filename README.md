# sysops-test

> SysOps test PoC app by Carlos Guerrero

Deploys a mockup infrastructure for a web application, deployed with Ansible on a single Ubuntu VM with Vagrant. 
The stack consists of an HAProxy load balancer that routes traffic to two backends based on the request path: echoserver for `/api` and nginx for `/statics`.

## Repository structure

```
.
├── ansible/
│   ├── ansible.cfg
│   ├── playbooks/apps/sysops-poc.yaml    # poc-app deployment playbook (vagrant up calls this playbook by default)
│   └── roles/
│       ├── docker-engine/        # Docker engine role, installs Docker Engine and the Compose plugin
│       └── sysops-app-poc/       # Deploys the app stack with docker-compose
├── infra/
│   └── Vagrantfile               # Ubuntu Jammy64 box, featuring Ansible provisioning
└── README.md
```

## Stack

| Service | Image | Role |
|---|---|---|
| lb | haproxy:3.0 | Load balancer, published on port 80 |
| api | registry.k8s.io/echoserver:1.8 | Simulates the API behind `/api` |
| statics | nginx:latest | Serves `/statics`, answering `Response from Server N` |

## How traffic is routed

- `/api/...` goes to the echoserver backend, balanced round-robin.
- `/statics/...` goes to the nginx backend, balanced by URI hash so the same path always hits the same instance.
- Any other path falls back to the statics backend.

Each nginx instance answers with its own id (`0`, `1`, ...). HAProxy sets the id in the `X-Server-ID` header and nginx reads it with a `map` block, see [sysops-app-poc-role-docs](ansible/roles/sysops-app-poc/README.md) for a detailed end-to-end explanation.

## Prerequisites

- Vagrant with the VirtualBox provider.
- Ansible installed locally, with the `community.docker` collection:

  ```
  ansible-galaxy collection install community.docker
  ```

  The Vagrant provisioner runs Ansible from your host.

## Quick start

```
vagrant up
```

The `Vagrantfile` provisions an `ubuntu/jammy64` box with Ansible, installs Docker and the Compose plugin, and deploys the stack. The app is reachable on `http://192.168.56.11` through the host-only network, and on `http://localhost:8080` through the forwarded port, which uses NAT and does not depend on host-only routing. If the host-only network is not reachable from your machine (VPN or interface conflict), use `http://localhost:8080`.

## Scaling

The number of instances is driven by two variables:

| Variable | Default | Description |
|---|---|---|
| `api_replicas` | 1 | Number of echoserver instances |
| `statics_replicas` | 1 | Number of nginx instances |

Set them in the `Vagrantfile` (`ansible.extra_vars`) and re-provision, or run the playbook directly:

```
ansible-playbook ansible/playbooks/apps/sysops-poc.yaml -e api_replicas=3 -e statics_replicas=3
```

## Verifying

```
# via host-only network
curl http://192.168.56.11/api
curl http://192.168.56.11/statics

# via forwarded port
curl http://localhost:8080/api
curl http://localhost:8080/statics
```

- `/api` returns the echoserver output. Repeated requests show round-robin across instances (the `host` field changes).
- `/statics` returns `Response from Server N`. Repeated requests to the same path return the same `N`; different paths may land on different instances.

## Featured roles docs

- [ansible/roles/docker-engine/README.md](ansible/roles/docker-engine/README.md)
- [ansible/roles/sysops-app-poc/README.md](ansible/roles/sysops-app-poc/README.md)

## Resources I found useful while doing this assessment

- [Ansible deb822_repository module](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/deb822_repository_module.html)
- [AnsibleDockerInstallation](https://github.com/kayvansol/AnsibleDockerInstallation)
- [HAProxy Docker image](https://hub.docker.com/_/haproxy/)
- [Docker Compose deploy specification](https://docs.docker.com/reference/compose-file/deploy/)
- [HAProxy configuration manual](https://docs.haproxy.org/3.0/configuration.html)
- [HAProxy DNS resolution tutorial](https://www.haproxy.com/documentation/haproxy-configuration-tutorials/proxying-essentials/dns-resolution)
- [HAProxy load balancing for static content (consistent hashing)](https://www.haproxy.com/blog/haproxys-load-balancing-algorithm-for-static-content-delivery-with-varnish)
- [How to access docker-compose created replicas in HAProxy config](https://stackoverflow.com/questions/68967624/how-to-access-docker-compose-created-replicas-in-haproxy-config)