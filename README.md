# ansible-elasticsearch

Ansible role for 6.x/5.x Elasticsearch. Currently this works on Debian and RedHat based Linux systems. Tested platforms are:

* Ubuntu 14.04
* Ubuntu 16.04
* Debian 8
* CentOS 7

The latest Elasticsearch versions of 6.x and 5.x are actively tested. **Only Ansible versions > 2.4.3.0 are supported, as this is currently the only version tested.**

**THIS ROLE IS FOR 6.x, 5.x. FOR 2.x SUPPORT PLEASE USE THE 2.x BRANCH.**

See [AGENTS.md](AGENTS.md) for the role's task layout and how the pieces fit together.

##### Dependency
This role uses the `json_query` filter, which [requires jmespath](https://github.com/ansible/ansible/issues/24319) on the local machine.

## Usage

Create your Ansible playbook with your own tasks, and include the role elasticsearch. You will have to have this repository accessible within the context of the playbook.

```sh
ansible-galaxy install elastic.elasticsearch
```

Then create your playbook yaml adding the role elasticsearch. By default, the user is only required to specify a unique `es_instance_name` per role application. This should be unique per node.
The application of the elasticsearch role results in the installation of a node on a host.

The simplest configuration therefore consists of:

```yaml
- name: Simple Example
  hosts: localhost
  roles:
    - role: elastic.elasticsearch
      es_instance_name: "node1"
```

The above installs a single node 'node1' on the host 'localhost'.

This role also uses [Ansible tags](http://docs.ansible.com/ansible/playbooks_tags.html). Run your playbook with the `--list-tasks` flag for more information.

## Testing

This role uses [Test Kitchen](https://kitchen.ci/) for local testing; there is no CI workflow configured in this fork. See [AGENTS.md](AGENTS.md#commands) for the exact `kitchen`/`make` commands, required Ruby/Docker setup, and how to select a suite or Elasticsearch version.

If you want to test X-Pack features with a license, export `ES_XPACK_LICENSE_FILE` first:
```sh
export ES_XPACK_LICENSE_FILE="$(pwd)/license.json"
```

### Basic Elasticsearch Configuration

All Elasticsearch configuration parameters are supported. This is achieved using a configuration map parameter `es_config`, which is serialized into the `elasticsearch.yml` file.
Using a map means the playbook doesn't need to change to reflect new, deprecated, or plugin configuration parameters.

In addition to `es_config`, several other parameters support additional functions, e.g. script installation. These are documented below and in the role's `defaults/main.yml`.

The following applies configuration parameters to an Elasticsearch instance:

```yaml
- name: Elasticsearch with custom configuration
  hosts: localhost
  roles:
    - role: elastic.elasticsearch
  vars:
    es_instance_name: "node1"
    es_data_dirs:
      - "/opt/elasticsearch/data"
    es_log_dir: "/opt/elasticsearch/logs"
    es_config:
      node.name: "node1"
      cluster.name: "custom-cluster"
      discovery.zen.ping.unicast.hosts: "localhost:9301"
      http.port: 9201
      transport.tcp.port: 9301
      node.data: false
      node.master: true
      bootstrap.memory_lock: true
    es_scripts: false
    es_templates: false
    es_version_lock: false
    es_heap_size: 1g
    es_api_port: 9201
```

To form a cluster successfully, configure these `es_config` keys:

* `http.port` - the HTTP port for the node
* `transport.tcp.port` - the transport port for the node
* `discovery.zen.ping.unicast.hosts` - the unicast discovery list, as `"<host>:<port>,<host>:<port>"` (typically the cluster's dedicated masters)
* `network.host` - sets both `network.bind_host` and `network.publish_host` to the same value, controlling which host different network components bind to and which host the node publishes to the cluster.

See the [Elasticsearch network settings guide](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-network.html) for default binding behavior and available options. The role does not enforce these settings — set them appropriately, and list and deploy master nodes first where possible.

A more complex example:

```yaml
- name: Elasticsearch with custom configuration
  hosts: localhost
  roles:
    - role: elastic.elasticsearch
  vars:
    es_instance_name: "node1"
    es_data_dirs:
      - "/opt/elasticsearch/data"
    es_log_dir: "/opt/elasticsearch/logs"
    es_config:
      node.name: "node1"
      cluster.name: "custom-cluster"
      discovery.zen.ping.unicast.hosts: "localhost:9301"
      http.port: 9201
      transport.tcp.port: 9301
      node.data: false
      node.master: true
      bootstrap.memory_lock: true
    es_scripts: false
    es_templates: false
    es_version_lock: false
    es_heap_size: 1g
    es_start_service: false
    es_plugins_reinstall: false
    es_api_port: 9201
    es_plugins:
      - plugin: ingest-geoip
        proxy_host: proxy.example.com
        proxy_port: 8080
```

#### Important Note

**The role uses `es_api_host` and `es_api_port` to communicate with the node for actions only achievable via HTTP, e.g. installing templates and checking that the node is active. These default to `localhost` and `9200`. If the node binds to a different host or port, change these.**

### Multi Node Server Installations

Applying the role results in the installation of one node per application. Specifying the role multiple times for a host installs multiple nodes on that host.

An example two-server deployment follows. The first server holds the master and is declared first — not mandatory, but recommended in any multi-node cluster. The second server hosts two data nodes.

**Note the structure below for the data nodes. More succinct structures exist that let the same role apply to a host multiple times, but this structure has proven the most reliable with respect to var behavior — it's the tested approach.**

```yaml
- hosts: master_nodes
  roles:
    - role: elastic.elasticsearch
  vars:
    es_instance_name: "node1"
    es_heap_size: "1g"
    es_config:
      cluster.name: "test-cluster"
      discovery.zen.ping.unicast.hosts: "elastic02:9300"
      http.port: 9200
      transport.tcp.port: 9300
      node.data: false
      node.master: true
      bootstrap.memory_lock: false
    es_scripts: false
    es_templates: false
    es_version_lock: false
    ansible_user: ansible
    es_plugins:
     - plugin: ingest-geoip

- hosts: data_nodes
  roles:
    - role: elastic.elasticsearch
  vars:
    es_instance_name: "node1"
    es_data_dirs:
      - "/opt/elasticsearch"
    es_config:
      discovery.zen.ping.unicast.hosts: "elastic02:9300"
      http.port: 9200
      transport.tcp.port: 9300
      node.data: true
      node.master: false
      bootstrap.memory_lock: false
      cluster.name: "test-cluster"
    es_scripts: false
    es_templates: false
    es_version_lock: false
    ansible_user: ansible
    es_api_port: 9200
    es_plugins:
      - plugin: ingest-geoip

- hosts: data_nodes
  roles:
    - role: elastic.elasticsearch
  vars:
    es_instance_name: "node2"
    es_api_port: 9201
    es_config:
      discovery.zen.ping.unicast.hosts: "elastic02:9300"
      http.port: 9201
      transport.tcp.port: 9301
      node.data: true
      node.master: false
      bootstrap.memory_lock: false
      cluster.name: "test-cluster"
    es_scripts: false
    es_templates: false
    es_version_lock: false
    es_api_port: 9201
    ansible_user: ansible
    es_plugins:
      - plugin: ingest-geoip
```

Parameters can also be assigned to hosts using the inventory file if desired.

Make sure your hosts are defined in your `inventory` file with the appropriate `ansible_ssh_host`, `ansible_ssh_user`, and `ansible_ssh_private_key_file` values.

Then run it:

```sh
ansible-playbook -i hosts ./your-playbook.yml
```

### Installing X-Pack Features

X-Pack features, such as Security, are supported. This feature is currently experimental.

`es_xpack_features` defaults to enabling all features: `["alerting","monitoring","graph","security","ml"]`.

These parameters configure X-Pack:

* `es_message_auth_file` - System Key field for message authentication. Place this file in the `files` directory.
* `es_xpack_custom_url` - URL to download X-Pack from, for installations where the elastic.co repo isn't accessible, e.g. `es_xpack_custom_url: "https://artifacts.elastic.co/downloads/packs/x-pack/x-pack-5.5.1.zip"`.
* `es_role_mapping` - role mappings file, described in the [role mapping guide](https://www.elastic.co/guide/en/x-pack/current/mapping-roles.html):

```yaml
es_role_mapping:
  power_user:
    - "cn=admins,dc=example,dc=com"
  user:
    - "cn=users,dc=example,dc=com"
    - "cn=admins,dc=example,dc=com"
```

* `es_users` - users declared as YAML under two sub-keys, `native` and `file`, which determine the realm the user is created under. e.g.:

```yaml
es_users:
  native:
    kibana4_server:
      password: changeMe
      roles:
        - kibana4_server
  file:
    es_admin:
      password: changeMe
      roles:
        - admin
    testUser:
      password: changeMeAlso!
      roles:
        - power_user
        - user
```

* `es_roles` - Elasticsearch roles declared as YAML under `native` and `file` sub-keys, determining whether the role is created via a file or an HTTP (native) call. List roles with their permissions under each key, using the [file-realm format](https://www.elastic.co/guide/en/x-pack/current/file-realm.html), e.g.:

```yaml
es_roles:
  file:
    admin:
      cluster:
        - all
      indices:
        - names: '*'
          privileges:
            - all
    power_user:
      cluster:
        - monitor
      indices:
        - names: '*'
          privileges:
            - all
    user:
      indices:
        - names: '*'
          privileges:
            - read
    kibana4_server:
      cluster:
          - monitor
      indices:
        - names: '.kibana'
          privileges:
            - all
  native:
    logstash:
      cluster:
        - manage_index_templates
      indices:
        - names: 'logstash-*'
          privileges:
            - write
            - delete
            - create_index
```

* `es_xpack_license` - X-Pack license, as a JSON blob. Set it directly (optionally protected by Ansible Vault) or load it from a file on the control machine via a lookup:

```yaml
es_xpack_license: "{{ lookup('file', playbook_dir + '/files/' + es_cluster_name + '/license.json') }}"
```

X-Pack configuration parameters can be added to `elasticsearch.yml` using the normal `es_config` parameter.

For a full example, see [`test/integration/xpack-upgrade.yml`](test/integration/xpack-upgrade.yml).

#### Important Note for Native Realm Configuration

Configuring native users and roles requires the role to call the Elasticsearch API. With Security installed, this needs two parameters:

* `es_api_basic_auth_username` - admin username
* `es_api_basic_auth_password` - admin password

Use a user declared in the file-based realm with admin permissions, or the default `elastic` superuser (default password `changeme`).

### Additional Configuration

Beyond `es_config`, these parameters customize the Java/Elasticsearch versions and role behavior:

* `es_enable_xpack` - default `true`. Set `false` to install the OSS release of Elasticsearch instead.
* `es_major_version` - must match `es_version`: `"5.x"` for versions >= 5.0 and < 6.0, `"6.x"` for versions >= 6.0.
* `es_version` (e.g. `"6.3.0"`).
* `es_api_host` - hostname used for HTTP actions such as installing templates. Defaults to `localhost`.
* `es_api_port` - port used for HTTP actions such as installing templates. Defaults to `9200`. **Change this if the HTTP port isn't 9200.**
* `es_api_basic_auth_username` - Elasticsearch username for admin actions, used if Security is enabled. Ensure this user is an admin.
* `es_api_basic_auth_password` - password for `es_api_basic_auth_username`.
* `es_start_service` - `true` (default) or `false`.
* `es_plugins_reinstall` - `true` or `false` (default).
* `es_plugins` - an array of plugin definitions, e.g.:
  ```yaml
    es_plugins:
      - plugin: ingest-geoip
  ```
* `es_path_repo` - whitelist for allowing local backup repositories.
* `es_action_auto_create_index` - controls auto index creation; use this syntax for a specific index list (else `true`/`false`):
  `es_action_auto_create_index: '[".watches", ".triggered_watches", ".watcher-history-*"]'`
* `es_allow_downgrades` - for development purposes only. `true` or `false` (default).
* `es_java_install` - `true` (default) or `false`. If `false`, Java isn't installed.
* `update_java` - updates Java to the latest version. `true` or `false` (default).
* `es_max_map_count` - maximum number of VMAs (Virtual Memory Areas) a process can own. Defaults to `262144`.
* `es_max_open_files` - maximum file descriptor number this process can open. Defaults to `65536`.
* `es_max_threads` - maximum number of threads the process can start. Defaults to `2048` for Elasticsearch versions below 6.0, `8192` from 6.0 up — so with this role's own default `es_version` (6.3.1), the effective default is `8192`.
* `es_debian_startup_timeout` - how long Debian-family SysV init scripts wait for the service to start, in seconds. Defaults to `10`.

Earlier examples show installing plugins via `es_plugins`. Officially supported plugins need no version or source delimiter — the plugin script determines the appropriate version for the target Elasticsearch version. For community plugins, include the full URL. Don't use this approach for the X-Pack plugin; see X-Pack above.

If installing Monitoring or Alerting, also specify the license plugin. Security configuration currently has limited support, with more planned for later versions.

To configure X-Pack to send mail, add this configuration to the role. When `require_auth` is true, also provide the user and password; otherwise, remove those keys:
```yaml
    es_mail_config:
        account: <functional name>
        profile: standard
        from: <from address>
        require_auth: <true or false>
        host: <mail domain>
        port: <port number>
        user: <e-mail address> --optional
        pass: <password> --optional
```

* `es_user` - defaults to `elasticsearch`.
* `es_group` - defaults to `elasticsearch`.
* `es_user_id` - undefined by default.
* `es_group_id` - undefined by default.

Both `es_user_id` and `es_group_id` must be set for the user and group IDs to be set.

By default, each node on a host installs to unique pid, plugin, work, data, and log directories, named using the instance and host name, beneath these defaults:

* `es_pid_dir` - defaults to `/var/run/elasticsearch`.
* `es_data_dirs` - defaults to `/var/lib/elasticsearch`. Can be a list or comma-separated string, e.g. `["/opt/elasticsearch/data-1","/opt/elasticsearch/data-2"]` or `"/opt/elasticsearch/data-1,/opt/elasticsearch/data-2"`.
* `es_log_dir` - defaults to `/var/log/elasticsearch`.
* `es_restart_on_change` - defaults to `true`. If `false`, changes don't restart Elasticsearch.
* `es_plugins_reinstall` - defaults to `false`. If `true`, removes all currently installed plugins, then reinstalls the listed ones.

This role ships sample scripts and templates under [`files/scripts/`](files/scripts) and [`files/templates/`](files/templates). These variables feed Ansible's [`with_fileglob`](http://docs.ansible.com/ansible/playbooks_loops.html#id4) loop — use an absolute path when setting the globs.
* `es_scripts_fileglob` - defaults to `<role>/files/scripts/`.
* `es_templates_fileglob` - defaults to `<role>/files/templates/`.

### Proxy

To define a proxy globally, set:

* `es_proxy_host` - global proxy host
* `es_proxy_port` - global proxy port

To define a proxy for a single plugin's installation:

```yaml
  es_plugins:
    - plugin: ingest-geoip
      proxy_host: proxy.example.com
      proxy_port: 8080
```

> For plugin installation, `proxy_host`/`proxy_port` take precedence when defined, falling back to the global proxy settings otherwise. The same values are used for both HTTP and HTTPS proxy settings.

## Notes

* The role assumes the user/group exists on the server. The Elasticsearch packages create the default `elasticsearch` user; if you need a different one, ensure it exists first.
* The role relies on each host's `inventory_hostname` to keep its directories unique.
* Changing an instance's `es_instance_name` installs a new component; the previous one remains.
* The role aims to be idempotent: running it repeatedly with no config changes should leave server state unchanged. Changed configuration gets applied, and Elasticsearch restarted where required.
* Systemd is used for Ubuntu >= 15, Debian >= 8, and CentOS >= 7; other versions use init scripts.
* Running X-Pack tests requires a license file with Security enabled — a trial license works. Set `ES_XPACK_LICENSE_FILE` to its full path before running tests.

## IMPORTANT NOTES RE PLUGIN MANAGEMENT

* Changing the ES version removes all plugins; those listed in the playbook are then reinstalled. ES 6.x requires this behavior.
* If no plugins are listed for a node, the role removes all currently installed plugins.
* The role auto-detects differences between installed and listed plugins, installing missing ones and removing unlisted ones. To force a reinstall, set `es_plugins_reinstall: true` — this removes all currently installed plugins, then installs the listed ones.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for the PR process and pre-PR checklist.

## Questions on Usage

We welcome questions on how to use the role. To keep the GitHub issues list focused on actual issues, please raise usage questions at the [Elasticsearch discussion forum](https://discuss.elastic.co/c/elasticsearch), which the maintainers monitor.
