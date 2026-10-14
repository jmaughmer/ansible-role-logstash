# ansible-role-logstash

An Ansible role that installs and configures Logstash 9.x on RHEL, AlmaLinux and Rocky Linux 9 and 10.

What it does:

- Creates multiple pipelines from a variable list and writes them to `pipelines.yml`.
- Sets up disk-backed persisted queues and dead letter queues, globally or per pipeline.
- Tunes performance: workers, batch size and delay, JVM heap, buffer type, open-file limits and process priority.
- Validates each changed pipeline with `logstash --config.test_and_exit` before it's written.
- Hot-reloads changed pipelines, and checks that each reload succeeded.
- Restarts and upgrades hosts in batches of limited size.
- Keeps secrets used in pipelines in the keystore, manages plugins, and installs TLS files from the `ssl_certificates` host variable.
- Works over high-latency, low-bandwidth links such as satellite. See [Slow and unreliable networks](#slow-and-unreliable-networks).

## Requirements

- **Control node:** ansible-core 2.15 or later, plus the `ansible.posix` collection:

  ```bash
  ansible-galaxy collection install -r requirements.yml
  ```

- **Target hosts:** RHEL, AlmaLinux or Rocky Linux 9 or 10 (see [Supported platforms](#supported-platforms)), with `openssl` installed. On STIG-hardened hosts see [Hardened hosts](#hardened-hosts-fips-selinux-fapolicyd) — fapolicyd has to allow Ansible before anything runs. They need a package source: Elastic's repository (`artifacts.elastic.co`) or a mirror, or a Red Hat Satellite or Katello server. See [Package source](#package-source).
- **Plugins:** hosts that install plugins from rubygems need access to it, or a proxy (`logstash_plugin_install_env`), or offline plugin packs.

## Supported platforms

| Distribution | `ansible_facts['distribution']` | Major versions |
|---|---|---|
| Red Hat Enterprise Linux | `RedHat` | 9, 10 |
| AlmaLinux | `AlmaLinux` | 9, 10 |
| Rocky Linux | `Rocky` | 9, 10 |

- **Same tasks everywhere.** The role has no distribution-specific tasks, and Logstash comes from Elastic's RPM packages on all of them.
- **Other distributions.** Preflight stops the run on any other distribution or major version. To try another Enterprise Linux 9 or 10 rebuild, such as CentOS Stream (`CentOS`) or Oracle Linux (`OracleLinux`), add its distribution fact to `logstash_supported_distributions`. Those aren't tested.
- **Satellite or Katello.** `logstash_repo_source: satellite` needs `subscription-manager` and a host registered to the server. RHEL hosts can use Red Hat Satellite or Katello. AlmaLinux and Rocky Linux hosts are normally registered to Katello; check your Satellite version's supported host operating systems before using it for them.
- **Elastic's support.** See [Elastic's support matrix](https://www.elastic.co/support/matrix) for the operating systems Elastic supports for your Logstash release.

## Installation

Add the role to your project's `requirements.yml`:

```yaml
roles:
  - name: logstash
    src: https://github.com/jmaughmer/ansible-role-logstash.git
    scm: git
    version: main          # or a release tag

collections:
  - name: ansible.posix
    version: ">=1.5.0"
```

```bash
ansible-galaxy install -r requirements.yml
```

## Package source

`logstash_repo_source` selects where the `logstash` package comes from:

| Value | Package source |
|---|---|
| `elastic` (default) | Elastic's public repository, or a mirror set with `logstash_repo_baseurl` and `logstash_repo_gpgkey`. The role writes a disabled repository file and enables it only for its own dnf tasks, so a routine `dnf update` never upgrades Logstash. |
| `satellite` | A repository synced into Red Hat Satellite or Katello, delivered to registered hosts through subscription-manager. See [Red Hat Satellite or Katello](#red-hat-satellite-or-katello). |
| `none` | Repositories you manage yourself. The role installs from whatever is enabled on the host. |

**Version check.** Before installing or upgrading, the role checks that `logstash_version` is available from the package source, so a missing version fails before anything changes. You can turn this off with `logstash_package_version_check`.

**Migrating.** `logstash_repo_source` replaces `logstash_manage_repo`; use `logstash_repo_source: none` where you had `false`. Preflight fails if the old variable is still set.

### Red Hat Satellite or Katello

One Satellite repository of Elastic's packages serves all three roles: Elasticsearch, Logstash and Kibana.

**On Satellite:**
1. Create a product, for example `Elastic`.
2. Create a content credential from Elastic's GPG key (`https://artifacts.elastic.co/GPG-KEY-elasticsearch`), so hosts verify package signatures.
3. In that product, create a yum repository with the upstream URL `https://artifacts.elastic.co/packages/9.x/yum` and the content credential, then sync it.
4. Add the repository to the content view the hosts use, then publish and promote it to their lifecycle environment.
5. Make sure the hosts' activation key gives them access to the product.

**On a registered host,** find the repository ID. It is normally `<organization label>_<product label>_<repository label>`:

```bash
subscription-manager repos --list | grep -i -A4 elastic
```

**Role variables:**

```yaml
logstash_repo_source: satellite
logstash_satellite_repo_id: ACME_Elastic_Elastic_9_x

# Optional
logstash_satellite_repo_state: disabled      # unmanaged (default), enabled or disabled
logstash_satellite_repo_overrides:           # applied with subscription-manager repo-override
  timeout: "600"
  retries: "20"
logstash_satellite_gpgkey: ""                # extra key to import (URL or path on the host)
```

**What the role does in `satellite` mode:**
- **No repository file.** subscription-manager owns `redhat.repo`. The role reads it on the host, without contacting Satellite, and passes the repository to dnf with `enablerepo`. Installs and upgrades work whether the repository is enabled or disabled.
- **Checks that `subscription-manager` is installed**, and fails with a clear message if it isn't.
- **Removes its own Elastic repository file** if one is left over from `elastic` mode, so packages don't come from two sources.
- **Checks the repository exists.** If it doesn't, the role runs `subscription-manager refresh` once (`logstash_satellite_refresh`), then fails with the related repository IDs it can see.
- **Enabled state** (`logstash_satellite_repo_state`):
  - `unmanaged` leaves it to Satellite.
  - `enabled` or `disabled` sets it with `subscription-manager repos`.
  - `disabled` keeps routine `dnf update` from upgrading Logstash, like `elastic` mode does.
- **Content overrides** (`logstash_satellite_repo_overrides`) are applied with `subscription-manager repo-override` when a value differs. Use them for dnf options such as `timeout`, `retries` and `minrate`, because `redhat.repo` can't be edited directly. `enabled`, `baseurl` and `name` aren't allowed.
- **GPG keys.** dnf imports the key assigned to the repository in Satellite. Set `logstash_satellite_gpgkey` to import a different key, from a URL or a file on the host.

Enabled-state changes and content overrides made with `subscription-manager` are stored on the Satellite server, as part of that host's settings.

## Quick start

[`examples/`](examples/) deploys two Logstash hosts that receive Beats and syslog traffic over TLS and send events to Elasticsearch:

```
examples/
├── ansible.cfg                      # SSH settings for slow links, roles/collections paths
├── requirements.yml                 # this role + ansible.posix
├── playbook.yml
├── templates/logstash/              # pipeline templates referenced from logstash_pipelines
└── inventory/
    ├── hosts.yml                    # logstash hosts (+ elasticsearch hosts for the output)
    ├── group_vars/all/network.yml   # retries/timeouts for satellite links
    ├── group_vars/logstash/
    │   ├── main.yml                 # pipelines, queues, tuning, keystore, plugins
    │   └── vault.yml                # secrets (encrypt with ansible-vault)
    └── host_vars/
        └── ls-N.example.com.yml     # heap, ssl_certificates
```

```bash
cd examples
ansible-galaxy install -r requirements.yml
# edit inventory/, then:
ansible-vault encrypt inventory/group_vars/logstash/vault.yml
ansible-playbook playbook.yml --ask-vault-pass
```

To run the examples against your working copy instead of the published role, link it into place instead of installing it:

```bash
mkdir -p roles && ln -s ../.. roles/logstash
```

**The example pipelines.**
- `beats` receives Beats on port 5044, over TLS.
- `syslog` receives syslog on port 5514, over TLS.
- Both hand events to `es-output`, which writes to the hosts in the `elasticsearch` inventory group. It has a 16 GB persisted queue and a dead letter queue, so events buffer on disk while Elasticsearch is unavailable.

**Before Logstash can write,** create a `logstash_internal` user in Elasticsearch with a role that can write `logs-*` data streams. The [ansible-role-elasticsearch](https://github.com/jmaughmer/ansible-role-elasticsearch) example defines a matching `logstash_writer` role. Then put the user's password in `vault_logstash_es_password`.

Useful tags: `logstash_pipelines`, `logstash_config`, `logstash_queues`, `logstash_keystore`, `logstash_plugins`, `logstash_tls`.

## Pipelines

```yaml
logstash_pipelines:
  - id: beats
    workers: 4                 # pipeline.workers
    batch_size: 500            # pipeline.batch.size
    batch_delay: 50            # pipeline.batch.delay
    ordered: auto              # pipeline.ordered
    ecs_compatibility: v8      # pipeline.ecs_compatibility
    queue:
      type: persisted
      max_bytes: 8gb
      page_capacity: 64mb
      checkpoint_writes: 1024
      drain: true
      path: /data/logstash/queue          # optional; Logstash appends /<id>
    dead_letter_queue:
      enable: true
      max_bytes: 1gb
      storage_policy: drop_older
    settings:                             # any other pipelines.yml keys, verbatim
      pipeline.plugin_classloaders: true
    templates:
      - logstash/beats.conf.j2            # from the role's or the playbook's templates/
    config: |                             # inline; appended after templates
      filter { mutate { add_tag => ["beats"] } }

  - id: external
    path_config: "/opt/pipelines/external/*.conf"   # files managed elsewhere

  - id: old-pipeline
    state: absent
```

- **One file per pipeline.** Each managed pipeline becomes one file, `logstash_pipeline_dir/<id>.conf`: the rendered `templates`, followed by `config`. Templates are rendered with the host's variables, so they can build things like Elasticsearch host lists from inventory.
- **Validation.** When `logstash_pipeline_validate` is true, a changed file is first checked by `logstash --config.test_and_exit`.
  - The check runs as the `logstash` user with a throwaway settings and data directory, so the running instance isn't affected.
  - It has access to the keystore and `logstash_environment`, so `${VAR}` references resolve.
  - An invalid pipeline fails the task and is never written.
  - Each changed pipeline costs one short JVM start.
- **Removal.** `.conf` files in `logstash_pipeline_dir` that aren't listed are removed when `logstash_pipelines_purge` is true.
- **Order of changes.** New files are written first, then `pipelines.yml`, then old files are removed, so automatic reload never sees a missing file.
- **Unknown keys.** Keys that aren't recognised at the pipeline, queue or dead-letter-queue level fail preflight, which catches typos such as `worker:` or `maxbytes:`.

### Applying pipeline changes

| `logstash_pipeline_change_action` | Behaviour on running hosts |
|---|---|
| `reload` (default) | Logstash reloads only the changed pipelines. With automatic reload that happens on its own; otherwise the role sends `SIGHUP`. The role then waits until the host runs exactly the managed pipelines and each changed pipeline's reload counter has moved. It fails with Logstash's error message if the latest reload failed; Logstash keeps running the previous version. |
| `restart` | Changes are applied with a rolling restart. Requires `logstash_config_reload_automatic: false`. |

With `reload`, pipeline changes reach every host at about the same time, and unchanged pipelines keep running. Use `restart` to roll pipeline changes out one batch at a time.

## Persisted queues and dead letter queues

`logstash_queue` and `logstash_dead_letter_queue` set the defaults in `logstash.yml`. A pipeline's `queue` or `dead_letter_queue` settings are merged over them.

- **Directories.** The role creates the base directories, owned by `logstash` with mode `0750`. Logstash creates a subdirectory per pipeline.
- **Size check.** `queue.page_capacity` must not be larger than `queue.max_bytes`.
- **Disk capacity check** (`logstash_queue_disk_check`). For each persisted-queue pipeline, the role takes the effective `max_bytes` minus what that queue already uses (measured with `du`). It groups the results by filesystem and fails if any filesystem doesn't have that much free space. This catches oversized queues before a pipeline fails to start.
- **Draining.** `drain: true` makes a stop wait until the queue is empty. Size `logstash_stop_retries` × `logstash_stop_delay` to allow for that, or set `logstash_systemd_timeout_stop_sec`.

Tuning notes:

- **Page cache.** Persisted queues use memory-mapped pages. Keep `logstash_heap_size` well under physical RAM; preflight warns above 50%.
- **Checkpoints.** Raising `checkpoint_writes` (for example to 4096) gives more throughput, but more events may be replayed after a crash.
- **Pipeline-to-pipeline.** Let input pipelines hand events to a dedicated output pipeline with a large persisted queue. The Elasticsearch connection then buffers on disk during an outage without slowing inputs (see the example group_vars).
- **Workers and batches.** `workers` defaults to the number of CPU cores. Larger `batch_size` values suit bulk outputs such as Elasticsearch; heap use grows with workers × batch size.
- **Process priority.** The package runs Logstash with `Nice=19`. Set `logstash_systemd_nice: 0` on dedicated hosts.

## Restarts

**Changes that need a restart:**

- `logstash.yml`
- JVM options
- the systemd drop-in or `/etc/sysconfig/logstash`
- keystore entries
- plugins
- TLS files
- pipelines, when `logstash_pipeline_change_action: restart`

**How the restart works.** Running hosts that need a restart, or have a version upgrade pending, are handled from the first host in the play. They go in batches of `logstash_restart_batch_size`, and each batch:

1. Stops gracefully, then waits for the service to be inactive.
2. If an upgrade is pending, upgrades the package and brings plugins up to date.
3. Starts, then waits until the managed pipelines are running.
4. Pauses for `logstash_restart_pause_seconds` so Beats load balancing can rebalance.

If a batch fails, the play stops and no further hosts are restarted.

**New or stopped hosts** are simply started, all in parallel.

Don't use `serial` on the play; preflight checks for it. Keep `any_errors_fatal: true`.

## Keystore

```yaml
logstash_keystore_password: "{{ vault_logstash_keystore_password }}"   # optional
logstash_keystore_entries:
  ES_LOGSTASH_PASSWORD: "{{ vault_logstash_es_password }}"            # use as ${ES_LOGSTASH_PASSWORD}
```

- **Detecting changes.** `logstash-keystore` can't display stored values. The role instead keeps a salted SHA-256 of each managed value in `/etc/logstash/.ansible-keystore-state.json` (root-only, mode `0600`), and replaces an entry only when its value changes.
- **Password.** The keystore password is written to `/etc/sysconfig/logstash` as `LOGSTASH_KEYSTORE_PASS`. Set it before the keystore is first created; changing the password of an existing keystore isn't automated.
- **API password.** Setting `logstash_api_auth_basic_username` enables basic auth on the Logstash API. The password is stored in the keystore as `LOGSTASH_API_PASSWORD`.

## TLS: `ssl_certificates`

Certificates are supplied per host, with the private key vaulted:

```yaml
# host_vars/ls-1.example.com.yml
ssl_certificates:
  - name: "ls-1.example.com"
    certificate:
      content: |
        -----BEGIN CERTIFICATE-----
        ...
    key:
      content: !vault |
        $ANSIBLE_VAULT;1.1;AES256
        ...
      # passphrase: !vault | ...             # only for encrypted PEM keys
```

`logstash_ssl_certificate_names` selects which entries to install. The example uses the inventory hostname. Each selected entry is installed into `logstash_ssl_dir` as:

- `<name>.crt`
- `<name>.key`, the key as supplied
- `<name>.pkcs8.key`, an unencrypted PKCS#8 copy for inputs such as `beats`, which require that format

`logstash_ssl_ca_certificate` is written to `ca.crt`, for example for the Elasticsearch output. The role checks that each key matches its certificate.

### Certificates issued by ansible-role-elastic-tls

[ansible-role-elastic-tls](https://github.com/jmaughmer/ansible-role-elastic-tls) can issue these certificates instead, from a private CA it creates and keeps, or from
Let's Encrypt. Run it first in the same play:

```yaml
  roles:
    - role: elastic_tls
    - role: logstash
```

It publishes what it issued as `elastic_tls_ssl_certificates`, which this role reads **after**
`ssl_certificates`. A manual entry with the same name always wins, so hosts can be moved across one
at a time. Match the name this role looks for: ``logstash_ssl_certificate_names``.

Entries in either list can point at a file already on the host instead of carrying its content:

```yaml
ssl_certificates:
  - name: ls-1.example.com
    certificate: {path: /etc/pki/ansible-tls/certs/ls-1.example.com.crt}
    key: {path: /etc/pki/ansible-tls/private/ls-1.example.com.key}
    ca: {path: /etc/pki/ansible-tls/certs/ca.crt}
```

``logstash_ssl_ca_certificate_path`` does the same for the CA when it is not part of the entry. Inline content wins when an
entry carries both.

Note that Let's Encrypt issues server certificates only. That is enough for Logstash inputs that only serve clients, but a private CA is simpler when beats also authenticate with certificates.

## Plugins

```yaml
logstash_plugins:
  - logstash-filter-tld
  - name: logstash-filter-translate
    version: 3.4.2                                   # reinstalled if a different version is installed
  - name: logstash-output-custom
    source: file:///opt/packs/logstash-output-custom-1.0.0.zip   # offline pack
  - name: logstash-input-unused
    state: absent
```

Set `logstash_plugin_install_env` for a proxy, for example `HTTPS_PROXY`. After an upgrade, unpinned plugins are updated with `logstash-plugin update`.

## Slow and unreliable networks

The role is built to work over high-latency, low-bandwidth links such as satellite. It doesn't rely on SSH pipelining.

**Connection settings.** Put these in your project's `ansible.cfg`; [`examples/ansible.cfg`](examples/ansible.cfg) has them.
- A 60 s connect timeout.
- Keepalives every 30 s, and a connection is only given up after 10 missed replies.
- Compression, and control connections that persist for 30 minutes.
- `retries = 10`, so Ansible reconnects when SSH drops mid-task.
- `gather_timeout = 120`.

**What the role does.**
- **Downloads.** Package and plugin downloads run as async jobs. They keep running on the host if the SSH session drops, and Ansible polls for the result. In `--check` mode they run as normal tasks, because Ansible rejects async tasks in check mode.
  - Failed downloads are retried.
  - The dnf repository gets a longer per-request timeout and more retries: `logstash_repo_timeout` and `logstash_repo_retries` for the Elastic repository, or `logstash_satellite_repo_overrides` for a Satellite repository.
  - dnf waits for a lock held by an interrupted earlier attempt.
- **Safe retries.** Plugin installs first check whether the plugin is already installed, so a retry after a lost response is harmless. It still counts as a change, so the restart still happens.

The defaults suit ordinary networks. For satellite links, use the values in [`examples/inventory/group_vars/all/network.yml`](examples/inventory/group_vars/all/network.yml):

| Variables | Value |
|---|---|
| `logstash_task_retries`, `logstash_task_delay` | 8 retries, 60 s apart |
| `logstash_download_async_timeout` | 4 hours per download attempt |
| `logstash_api_timeout` | 180 s per request |
| `logstash_api_retries`/`_delay`, `logstash_stop_retries`/`_delay` | 180 × 20 s |

A retried task can take up to retries × (delay + attempt time) before it gives up. If many hosts share one link, lower `forks`.

## Firewall

Logstash has no fixed port: what has to be open depends on the inputs your pipelines declare, so the
role opens nothing until you list it. `logstash_firewall_ports` is that list:

```yaml
logstash_firewall_ports:
  - 514/udp          # syslog from network gear
  - 514/tcp
  - 5044/tcp         # beats
  - 5000-5010/tcp    # a range, for a set of tcp inputs
logstash_firewall_services: [syslog]      # or named firewalld services
```

| Variable | Default | |
|---|---|---|
| `logstash_manage_firewall` | `true` | Set to `false` to manage the firewall yourself |
| `logstash_firewall_zone` | `public` | Zone the rules go in |
| `logstash_firewall_ports` | `[]` | Ports the configured inputs listen on |
| `logstash_firewall_services` | `[]` | Named firewalld services |

Rules are added permanently and immediately. Preflight rejects entries that are not `port/proto` or
`range/proto`, so a typo fails the run instead of quietly leaving an input unreachable. The API port
is deliberately not opened: it binds to localhost. If firewalld is not running the role reports the
ports that still have to be reachable.

**Ports below 1024.** Logstash runs as the `logstash` user, which cannot bind privileged ports such
as syslog's 514. Grant the capability through the systemd drop-in:

```yaml
logstash_systemd_capabilities: [CAP_NET_BIND_SERVICE]
```

The role sets `AmbientCapabilities` and `CapabilityBoundingSet` to that list, and restarts Logstash
when it changes. Preflight fails if `logstash_firewall_ports` lists a port below 1024 without
`CAP_NET_BIND_SERVICE`, since the input would otherwise fail when its pipeline starts.

When a virtual address fronts these hosts, list the same ports in the keepalived role as well, so
the rules are right even when this role is run with `--tags`.

## Hardened hosts (FIPS, SELinux, fapolicyd)

The role is meant to run on hosts with the DISA STIG applied. What that takes:

### fapolicyd

**Ansible itself has to be allowed first.** fapolicyd blocks whatever it does not trust, and that
includes the Python module Ansible copies to the host for *every* task. Nothing runs until that is
allowed, so add a rule once, before using these roles:

```
# /etc/fapolicyd/rules.d/30-ansible.rules
allow perm=open exe=/usr/bin/python3.9 : dir=/root/.ansible/tmp/
```

Load it with `fagenrules --load`. The file has to sort before the rules that deny untrusted
interpreted files, and the interpreter and directory have to match the host's Python and the user you
become (`ansible_remote_tmp`). The same rule covers the `repo_lookup.py` helper these roles run in
Satellite mode. Denials are audited, so `ausearch -m fanotify -ts recent` shows what was blocked.

**What Logstash runs that rpm did not install:** plugins added with `logstash-plugin`, and the
pipeline validation helper this role writes to `/usr/local/libexec`. Without trust entries fapolicyd
blocks both — the helper visibly, since `logstash_pipeline_validate` then fails on every run.

| Variable | Default | Purpose |
|---|---|---|
| `logstash_fapolicyd_trust` | `auto` | `auto` acts only when fapolicyd is running; `true` always; `false` never |
| `logstash_fapolicyd_trust_paths` | helper, plus the plugin directory when plugins are managed | Paths to trust |
| `logstash_fapolicyd_trust_file` | `ansible-logstash` | File under `/etc/fapolicyd/trust.d/` |

The helper is written and trusted before any pipeline is validated. The plugin directory holds
thousands of files, so refreshing entries after a plugin change takes a while; a fapolicyd rule for
that directory is the lighter alternative, in which case set `logstash_fapolicyd_trust_paths` to the
helper alone. A rolling upgrade rewrites the entries too, on each host after its plugins are updated
and before Logstash starts again.

The entries go in a file of their own under `/etc/fapolicyd/trust.d/`, which needs fapolicyd 1.1 or
newer (RHEL 9 and 10 both ship that). Set `logstash_fapolicyd_trust: false` on anything older and manage
trust yourself.

### FIPS mode

Logstash has no FIPS mode: it runs JRuby on the JDK it ships with, which does not inherit the system
policy, and Elastic does not validate it. A host in FIPS mode runs Logstash unchanged.

What matters in practice:

- **Private keys must not be in the traditional encrypted PEM format.** A key starting with
  `Proc-Type: 4,ENCRYPTED` derives its key with MD5 and cannot be read in FIPS mode — including by
  the `openssl pkcs8` conversion this role runs for the `beats` input. Preflight fails with that
  message when the host reports FIPS. Convert it once with `openssl pkcs8 -topk8 -in old.key -out new.key`,
  or let [ansible-role-elastic-tls](https://github.com/jmaughmer/ansible-role-elastic-tls) issue the
  keys, which are PKCS#8 already.
- **TLS settings live in the pipelines,** so protocol and cipher choices for inputs and outputs are
  yours to set in the plugin configuration (`ssl_supported_protocols`, `ssl_cipher_suites`).
- **Importing Elastic's GPG key** can fail under a strict crypto policy; use an internal mirror or a
  Satellite repository.

### SELinux

The Elastic packages ship no SELinux policy, so Logstash runs in `unconfined_service_t` and works
under a targeted policy in enforcing mode. Directories moved out of their packaged locations —
`logstash_path_data`, `logstash_path_logs`, and any `path.queue` or `path.dead_letter_queue` outside
the data directory — come out as `default_t`. With `logstash_selinux_manage: auto` the role adds an
`fcontext` rule and runs `restorecon` for those, and leaves the packaged ones alone.

- **Ansible needs `python3-libselinux`** on the hosts, or the file and copy modules cannot set
  contexts. It is part of a normal RHEL install.
- **`semanage` comes from `policycoreutils-python-utils`,** which is required only when the role
  manages contexts.

## Notes and caveats

- **Check mode** is not supported for first-time installs, because pipeline validation and the plugin and keystore CLIs need the Logstash binaries. It works on hosts where Logstash is already installed.

## Variables

[`defaults/main.yml`](defaults/main.yml) documents every variable.

## Role layout

```
├── defaults/main.yml
├── vars/main.yml              # setting-name maps, internal values
├── meta/main.yml
├── files/repo_lookup.py        # reads repository definitions, including redhat.repo
├── tasks/
│   ├── main.yml
│   ├── facts.yml / preflight.yml
│   ├── repository.yml              # Elastic repository file, or Satellite repository checks
│   ├── package_version_check.yml   # requested version is available before install/upgrade
│   ├── install.yml
│   ├── system.yml             # systemd drop-in, /etc/sysconfig/logstash, firewalld
│   ├── queues.yml             # queue directories, disk capacity check
│   ├── certificates.yml       # certs, PKCS#8 conversion
│   ├── keystore.yml
│   ├── plugins.yml / plugins_sync.yml
│   ├── configure.yml          # logstash.yml, jvm.options
│   ├── pipelines.yml          # validated pipeline files, pipelines.yml, purge
│   ├── service.yml            # start, reload, decide who needs restart
│   ├── wait_pipelines.yml     # readiness and reload verification
│   └── rolling_restart.yml / restart_batch.yml
├── templates/
└── examples/
```

## Related roles

- [ansible-role-elasticsearch](https://github.com/jmaughmer/ansible-role-elasticsearch): the Elasticsearch cluster Logstash writes to.
- [ansible-role-kibana](https://github.com/jmaughmer/ansible-role-kibana)
- [ansible-role-elastic-tls](https://github.com/jmaughmer/ansible-role-elastic-tls): issues the certificates these roles install, from a private CA or Let's Encrypt.
- [ansible-role-keepalived](https://github.com/jmaughmer/ansible-role-keepalived): a virtual address for syslog senders that can only be pointed at one destination.

All three use the same `ssl_certificates` host variable format.

## License

MIT. See [LICENSE](LICENSE).
