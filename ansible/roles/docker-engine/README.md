# docker-engine

>Ansible role that installs Docker Engine and the Compose plugin on Ubuntu hosts, after this role is run, the host is ready to run containers and `docker-compose`.

<br>

## What it does

- Adds the official Docker APT repository with its GPG key (using `deb822_repository`).
- Installs `docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-buildx-plugin` and `docker-compose-plugin`.
- Adds the `vagrant` user to the `docker` group so that `sudo` is not required.
- Starts the Docker service and enables it at boot.

<br>

## Requirements

- This role uses `deb822_repository`, which requires ansible-core 2.15 or later.
- Network access on the target host to reach the Docker repository.

<br>

## Variables

| Variable | Default | Description |
|---|---|---|
| `docker_packages` | docker-ce, docker-ce-cli, containerd.io, docker-buildx-plugin, docker-compose-plugin | Packages to install |
| `docker_users` | [vagrant] | Users added to the docker group |
| `docker_repo_apt_url` | https://download.docker.com/linux/ubuntu | Docker APT repository |
| `docker_repo_gpg_key_url` | https://download.docker.com/linux/ubuntu/gpg | GPG key for the repository |
| `apt_default_cache_valid_time` | 3600 | `cache_valid_time` value passed to the apt module |

<br>

## Usage

```yaml
- hosts: all
  become: true
  roles:
    - docker-engine
```

<br>

## Notes

- The `sysops-app-poc` role declares this role as a dependency in `meta/main.yml`.
- The `docker` group is assigned with `append: yes`, so the user's existing groups are preserved.
