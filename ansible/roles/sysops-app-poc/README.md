# sysops-app-poc

Ansible role that deploys the SysOps app PoC on a Linux host.

>The role renders the configuration files, deploys the containers with Docker Compose, and verifies the endpoints respond with HTTP 200.

<br>

## Components and Network

The docker services deployed by this stack are the following:
- An external-facing HAProxy service acting as a load balancer.
- An "api" (echoserver), the HAProxy service will route all /api requests to this service.
- A "statics" endpoint (nginx), HAProxy will route all /statics requests to this service.

Two networks are defined in Compose:

- `external` (bridge): carries the published port 80, used only by the LB.
- `internal` (`internal: true`): backends only, no outbound internet access. The LB and both backends are attached to it.

<br>

## Request flow

1. A client sends an HTTP request to the host on port 80. Docker publishes this port to the `lb` container, where HAProxy listens (`bind *:80`).

2. The `frontend loadbalancer` evaluates two ACLs against the request path:

   | ACL | Condition | Backend |
   |---|---|---|
   | `api` | `path_beg /api` | `api_backend` |
   | `statics` | `path_beg /statics` | `statics_backend` |

   `use_backend` rules are evaluated in order, so `/api/...` goes to the API and `/statics/...` to the statics. Any other path falls through to `default_backend statics_backend`.

3. `api_backend` balances with `roundrobin`: requests are distributed one after another across `api-0 … api-(N-1)`, where `N = api_replicas`.

4. `statics_backend` balances with `balance uri` + `hash-type consistent`: the request path is hashed, so a given path is always routed to the same nginx instance.

<br>

## How the server id is assigned

The `statics` requirement states that each nginx must answer with its own id (`0`, `1`, `2`, …). The mechanism is:

```
HAProxy (statics_backend)
  balance uri + hash-type consistent   → same path → same server (e.g. statics-1)
  http-send-name-header X-Server-ID    → forwards the header "X-Server-ID: statics-1"

nginx (the statics-1 container)
  map $http_x_server_id $server_id     → extracts "1" from "statics-1"
  return 200 "Response from Server 1"
```

In detail:

1. On HAProxy config, `server-template statics- 0-(N-1)` generates servers named `statics-0`, `statics-1`, … `statics-(N-1)`...
2. `http-send-name-header X-Server-ID` makes HAProxy add a `X-Server-ID` header to the request it forwards, containing the name of the server that will handle it (for example `statics-1`).
3. On the nginx side, a `map` translates that header into a numeric id:

   ```nginx
   map $http_x_server_id $server_id {
       default "unknown";
       ~^statics-(?<server_num>\d+)$ $server_num;
   }
   ```

4. The `location /statics` block returns `Response from Server $server_id`.

A single nginx replica (`statics_replicas: 1`) therefore answers `Response from Server 0`, matching the base requirement.

### Weaknesses

Since the server id on the nginx service comes from the header added by HAProxy, nginx does not know its own identity. A request sent directly to the nginx container (bypassing HAProxy) does not carry the `X-Server-ID` header and returns `Response from Server unknown`.

A possible solution to this, considered and discarded, was to loop the `statics` service in `docker-compose.yaml.j2` with Jinja and inject each container's id through an environment variable, so the id would be persistent. That approach was rejected because using replicas keeps the Compose definition cleaner and is considered better practice, even with the weakness above. In that alternative, the id would not be assigned by HAProxy but directly by Ansible based on the number of replicas needed.

<br>

## HAProxy configs featured

- **Path-based routing:** ACLs on `path_beg` split traffic between `/api` and `/statics`.
- **Round-robin** for the API backend.
- **URI hashing with consistent hashing** for the statics backend, guaranteeing that a given path is always served by the same instance while minimising reshuffling on scale changes.
- **Dynamic server discovery:** `server-template` combined with the Docker embedded DNS resolver (`127.0.0.11:53`) discovers the container IPs at runtime, so the number of replicas is driven purely by the variables.
- **Health checks**: every generated server carries `check`, so HAProxy only routes to healthy containers.

<br>

## Variables

| Variable | Default | Description |
|---|---|---|
| `api_replicas` | 1 | Number of echoserver instances |
| `statics_replicas` | 1 | Number of nginx instances |
| `app_name` | sysops-poc | Docker Compose project name |
| `app_user` | vagrant | Owner of the files generated on the host |
| `app_dir_path` | /home/vagrant/sysops-poc | Directory where configs are rendered |
| `haproxy_config_file_src_path` | …/haproxy/haproxy.cfg | Host path of the HAProxy config |
| `nginx_config_file_src_path` | …/nginx/default.conf | Host path of the nginx config |

<br>

## Files

| File | Purpose |
|---|---|
| `tasks/main.yml` | Creates directories, renders configs, deploys Compose, runs health checks |
| `handlers/main.yml` | Restarts the `lb` and `statics` services when their config changes |
| `templates/docker-compose.yaml.j2` | The three services, replica counts and networks |
| `templates/haproxy.cfg.j2` | Frontend, ACLs, backends and `server-template` |
| `files/nginx.conf` | nginx config with the `map` that reads `X-Server-ID` |
| `defaults/main.yml` | Role defaults |
| `meta/main.yml` | Dependency on `docker-engine` |

The HAProxy and nginx configuration files are explained line by line in `haproxy-docs.md` and `nginx-docs.md` at the repository root.

<br>

## Deployment

```bash
ansible-playbook playbooks/apps/sysops-poc.yaml -e api_replicas=2 -e statics_replicas=3
```

Or, with the bundled Vagrant box:

```bash
vagrant up

# or
vagrant provision
```

The `Vagrantfile` invokes the playbook and passes the replica counts.

<br>

## Verification

```bash
# via the host-only network
curl http://192.168.56.11/api
curl http://192.168.56.11/statics

# or via the forwarded port configured in the Vagrantfile
curl http://localhost:8080/api
curl http://localhost:8080/statics
```

- `/api`: the echoserver returns the request details. Repeated requests show round-robin across the instances (the `host` field changes).
- `/statics`: returns `Response from Server N`. Repeated requests to the same path return the same `N` (URI hashing); different paths may land on different instances.

<br>

## Requirements

- `docker-engine` role (declared as a dependency).
- `community.docker` collection for `docker_compose_v2`.
