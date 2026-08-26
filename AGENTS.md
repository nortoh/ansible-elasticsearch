# ansible-elasticsearch

Ansible role that installs and configures Elasticsearch 5.x/6.x nodes on Debian- and RedHat-family Linux hosts, including X-Pack security, plugins, and templates.

## Architecture in a paragraph

The role exists so a playbook can stand up one or more Elasticsearch nodes per host with a single `roles:` entry, instead of hand-rolling package install, config templating, and plugin/template/X-Pack management per platform. `tasks/main.yml` is the entry point: it loads OS-specific vars (`vars/Debian.yml` / `vars/RedHat.yml`), resolves compatibility and instance-scoped parameters, then includes task files for Java, package install, `elasticsearch.yml` config rendering, scripts, plugins, and X-Pack in a fixed order, each tagged so a caller can run a subset with `--tags`; some (Java, scripts, plugins, snapshot release) are also gated by a `when`, but install, config, and X-Pack always run. X-Pack file-realm security (users, roles, role mappings) is written to disk from `tasks/xpack/elasticsearch-xpack.yml` before the service starts. Only native-realm user/role management and license activation — the parts that call the Elasticsearch HTTP API — run later in `tasks/main.yml`, after the service is confirmed started and, for the native realm, after a fixed 15-second settle. `filter_plugins/custom.py` supplies Jinja filters (`extract_role_users`, `remove_reserved`, …) used by the security templates under `templates/security/`. The role declares no Galaxy dependencies (`meta/main.yml`); it does require `jmespath` on the control node for a `json_query` filter used in the parameter/compatibility tasks. Multiple nodes on one host are supported by applying the role multiple times with distinct `es_instance_name` values, each getting its own pid/data/log directories.

## File map

```
defaults/
  main.yml                 # role-wide default vars (es_version, es_config, dirs, xpack feature list)
files/
  logging/                 # log4j2 template fragment shipped as-is
  scripts/                 # sample custom scoring script for with_fileglob install
  system_key               # binary X-Pack system key fixture used by tests
  templates/                # sample query template fixture used by tests
filter_plugins/
  custom.py                 # Jinja filters consumed by the security/role-mapping templates
handlers/
  main.yml                 # systemd reload + service restart, gated on es_restart_on_change
tasks/
  main.yml                 # entry point: orchestrates every other task file via when/tags
  compatibility-variables.yml
  elasticsearch-config.yml           # renders elasticsearch.yml, jvm.options, log4j2.properties
  elasticsearch-Debian.yml           # apt-based install path
  elasticsearch-Debian-version-lock.yml
  elasticsearch-optional-user.yml    # creates es_user/es_group when requested
  elasticsearch-parameters.yml       # resolves per-instance dirs/ports from es_instance_name
  elasticsearch-plugins.yml          # diffs installed vs. declared plugins, installs/removes
  elasticsearch-RedHat.yml           # yum-based install path
  elasticsearch-RedHat-version-lock.yml
  elasticsearch-scripts.yml          # pushes files/scripts/* via with_fileglob
  elasticsearch-template.yml         # pushes index templates over the ES HTTP API
  elasticsearch.yml                  # dispatches to the Debian/RedHat install task
  java.yml                           # installs/updates the JVM
  snapshot-release.yml               # switches package source to a snapshot build
  xpack/
    elasticsearch-xpack.yml          # enable/disable X-Pack features, always runs
    elasticsearch-xpack-install.yml
    security/
      elasticsearch-security.yml         # keystore bootstrap, file-realm dispatch, role-mapping/auth-key file rendering
      elasticsearch-security-file.yml    # renders roles.yml/users_roles to disk, manages file-realm users
      elasticsearch-security-native.yml  # creates native-realm users/roles via the ES API (included directly from tasks/main.yml)
      elasticsearch-xpack-activation.yml # applies the X-Pack license over the API
templates/
  elasticsearch.j2, elasticsearch.repo, elasticsearch.yml.j2, jvm.options.j2, log4j2.properties.j2
  init/{debian,redhat}/elasticsearch.j2   # SysV init scripts for pre-systemd platforms
  security/                               # role/role-mapping/users_roles templates
  systemd/elasticsearch.j2                # systemd unit for systemd-based platforms
test/
  matrix.yml                # kitchen suite/platform matrix reference
  integration/
    <suite>.yml              # one Ansible playbook per Kitchen suite (oss, xpack, multi, …)
    <suite>/serverspec/       # serverspec assertions run after each suite converges
    helpers/serverspec/       # shared serverspec matchers used across suites
vars/
  main.yml                  # cross-platform constants (paths, xpack feature/user lists)
  Debian.yml, RedHat.yml     # per-OS-family package/service names and paths
.kitchen.yml                 # Test Kitchen driver/provisioner/platform/suite config
Makefile                     # thin wrapper around `bundle exec kitchen`
```

## Commands

No CI workflow, `ansible-lint`, or `yamllint` config exists in this repo. Testing is local-only, via [Test Kitchen](https://kitchen.ci/) with the `kitchen-ansible` and `kitchen-docker` drivers, wrapped by the `Makefile`. Requires Ruby, Bundler, Docker, and Make on the machine running the tests.

```sh
bundle install                       # install Ruby test dependencies (make setup)
bundle exec kitchen list             # list available suite/platform combinations
bundle exec kitchen converge <PATTERN>   # converge a suite, e.g. `make converge PATTERN=oss-centos-7`
bundle exec kitchen verify <PATTERN>     # run serverspec assertions against a converged suite
bundle exec kitchen test <PATTERN> --destroy=always   # converge, verify, and tear down
bundle exec kitchen destroy          # tear down all running instances (make destroy-all)
```

The default `PATTERN` (`xpack-ubuntu-1604`) and `VERSION` (`6.x`) come from the `Makefile`; override either as `make converge PATTERN=oss-centos-7 VERSION=5.x`. X-Pack suites need `ES_XPACK_LICENSE_FILE` exported to a trial license before converging.

## Conventions

- Every task file included from `tasks/main.yml` is tagged (`java`, `install`, `config`, `scripts`, `plugins`, `xpack`) — add new functionality as its own included file rather than growing `main.yml` inline. Only some includes (`java`, `scripts`, `plugins`, the snapshot release) are also gated by a `when:`; `install`, `config`, and `xpack` always run — `xpack` deliberately, per the in-file comment, so features can be removed as well as added. Don't assume a `when:` skip flag exists for install/config/xpack.
- X-Pack file-realm security (users, roles) is written to disk before the service starts. Only the HTTP-API-driven pieces — native realm, license activation, templates — run after the service has started, and native realm additionally waits a fixed 15-second settle (`tasks/main.yml`). Don't reorder those ahead of the service start.
- OS differences are isolated to `vars/Debian.yml` / `vars/RedHat.yml` and the matching `tasks/elasticsearch-{Debian,RedHat}.yml` pair; add a new platform by extending that pair, not by branching inline in shared tasks.
- Custom Jinja filters live in `filter_plugins/custom.py`, not inline `{{ }}` gymnastics in templates — extend that module for new template logic.
- Each Kitchen suite in `.kitchen.yml` pairs an integration playbook (`test/integration/<suite>.yml`) with serverspec assertions (`test/integration/<suite>/serverspec/`); a behavior change needs both updated together.

## See also

- [README.md](README.md) — usage examples and the full `es_*` variable reference
- [CONTRIBUTING.md](CONTRIBUTING.md) — PR process and pre-PR checklist
