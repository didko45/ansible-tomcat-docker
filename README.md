# ansible-tomcat-docker

Ansible project that provisions RHEL-compatible hosts, deploys a containerized Tomcat application behind an Apache HTTP Server load balancer, and performs zero-downtime rolling updates with automatic rollback.

The application image comes from [tomcat-docker](https://github.com/didko45/tomcat-docker); the lab VMs come from [devops-lab](https://github.com/didko45/devops-lab).

## Architecture

```
                 control (192.168.56.10)
   ┌───────────────────────────────────────────────┐
   │ Ansible │ registry :5000 │ httpd :80 balancer │
   └──────────────────┬─────────────────┬──────────┘
            pull image│                 │ round-robin
          ┌───────────┴──────┐  ┌───────┴──────────┐
          │ app1 :8080       │  │ app2 :8080       │
          │ tomcat-app       │  │ tomcat-app       │
          └──────────────────┘  └──────────────────┘
```

## Layout

```
├── ansible.cfg
├── collections/requirements.yml   # pinned collections for ansible-core 2.14
├── inventories/lab/
│   ├── hosts.yml                  # groups: app, lb
│   └── group_vars/app.yml         # registry and the released app version
├── roles/
│   ├── common/                    # packages, chrony, motd, unique SSH host keys
│   ├── docker/                    # Docker CE, log rotation, live-restore
│   ├── tomcat_app/                # app container with memory limit and health gate
│   └── httpd_lb/                  # Apache balancer generated from inventory
├── tasks/lb_member.yml            # drain/enable a host via balancer-manager
├── site.yml                       # full provisioning
└── deploy.yml                     # rolling deploy with rollback
```

## Usage

```bash
ansible-galaxy collection install -r collections/requirements.yml
ansible-playbook site.yml      # provision registry, app servers and load balancer

# Release: set tomcat_app_version in inventories/lab/group_vars/app.yml, commit, then
ansible-playbook deploy.yml
```

The version in Git is the version that runs: `site.yml` and `deploy.yml` both read it from `group_vars`.

## Rolling deploy

`deploy.yml` updates one host at a time (`serial: 1`, `max_fail_percentage: 0`):

1. Skip the host if it already runs the target version.
2. Record the running version for rollback.
3. Drain the host in the load balancer and let in-flight requests finish.
4. Deploy the new version and wait for `/health` to return 200.
5. On failure: redeploy the previous version, return the host to the balancer and stop the rollout, so the remaining hosts are never touched.
6. On success: return the host to the balancer and continue with the next one.

## Verified results

| Scenario | Result |
| --- | --- |
| Rollout 0.3 → 0.4 with a request every 0.5 s | Every request during the rollout returned HTTP 200 |
| Broken 0.5 (health check returns 404) | app1 rolled back to 0.4 automatically, app2 untouched, nearly 200 requests during the incident, all HTTP 200 |
| Second run of `site.yml` / `deploy.yml` | `changed=0` on all hosts |

## Notable details

| Topic | Detail |
| --- | --- |
| Handlers | Handlers run at the end of a play, so the `docker` role calls `meta: flush_handlers` before later roles pull images; `force_handlers = True` keeps restarts from being lost on failure |
| Cloned VMs | Hosts built from one box share SSH host keys; `common` regenerates them once (`creates:` marker) and `site.yml` re-syncs `known_hosts` on the control node |
| SELinux | `httpd_can_network_connect` is enabled so httpd can proxy to the backends |
| Container updates | `image_name_mismatch: recreate` makes a new tag always recreate the container, even when the image ID is unchanged |
| Collections | Pinned to versions that support ansible-core 2.14 shipped with Rocky Linux 9 |
| Registry | Plain-HTTP registry is for the lab only; production would use TLS |
| Code quality | Passes `ansible-lint` with the `production` profile |
