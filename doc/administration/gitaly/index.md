# Gitaly

[Gitaly](https://gitlab.com/gitlab-org/gitaly) is the service that
provides high-level RPC access to Git repositories. Without it, no other
components can read or write Git data.

GitLab components that access Git repositories (gitlab-rails,
gitlab-shell, gitlab-workhorse) act as clients to Gitaly. End users do
not have direct access to Gitaly.

## Configuring Gitaly

The Gitaly service itself is configured via a TOML configuration file.
This file is documented [in the gitaly
repository](https://gitlab.com/gitlab-org/gitaly/blob/master/doc/configuration/README.md).

To change a Gitaly setting in Omnibus you can use
`gitaly['my_setting']` in `/etc/gitlab/gitlab.rb`. Changes will be applied
when you run `gitlab-ctl reconfigure`.

```ruby
gitaly['prometheus_listen_addr'] = 'localhost:9236'
```

To change a Gitaly setting in installations from source you can edit
`/home/git/gitaly/config.toml`. Changes will be applied when you run
`service gitlab restart`.

```toml
prometheus_listen_addr = "localhost:9236"
```

## Client-side GRPC logs

Gitaly uses the [gRPC](https://grpc.io/) RPC framework. The Ruby gRPC
client has its own log file which may contain useful information when
you are seeing Gitaly errors. You can control the log level of the
gRPC client with the `GRPC_LOG_LEVEL` environment variable. The
default level is `WARN`.

## Running Gitaly on its own server

> This is an optional way to deploy Gitaly which can benefit GitLab
installations that are larger than a single machine. Most
installations will be better served with the default configuration
used by Omnibus and the GitLab source installation guide.

Starting with GitLab 11.4, Gitaly is a replacement for NFS except
when the [Elastic Search indexer](https://gitlab.com/gitlab-org/gitlab-elasticsearch-indexer)
is used.

Gitaly network traffic is unencrypted so you should use a firewall to
restrict access to your Gitaly server.

Below we describe how to configure a Gitaly server at address
`gitaly.internal:8075` with secret token `abc123secret`. We assume
your GitLab installation has two repository storages, `default` and
`storage1`.

### Client side token configuration

Start by configuring a token on the client side.

Omnibus installations:

```ruby
# /etc/gitlab/gitlab.rb
gitlab_rails['gitaly_token'] = 'abc123secret'
```

Source installations:

```yaml
# /home/git/gitlab/config/gitlab.yml
gitlab:
  gitaly:
    token: 'abc123secret'
```

You need to reconfigure (Omnibus) or restart (source) for these
changes to be picked up.

### Gitaly server configuration

Next, on the Gitaly server, we need to configure storage paths, enable
the network listener and configure the token.

Note: if you want to reduce the risk of downtime when you enable
authentication you can temporarily disable enforcement, see [the
documentation on configuring Gitaly
authentication](https://gitlab.com/gitlab-org/gitaly/blob/master/doc/configuration/README.md#authentication)
.

Gitaly must trigger some callbacks to GitLab via GitLab Shell. As a result,
the GitLab Shell secret must be the same between the other GitLab servers and
the Gitaly server. The easiest way to accomplish this is to copy `/etc/gitlab/gitlab-secrets.json`
from an existing GitLab server to the Gitaly server. Without this shared secret,
Git operations in GitLab will result in an API error.

> **NOTE:** In most or all cases the storage paths below end in `/repositories` which is
different than `path` in `git_data_dirs` of Omnibus installations. Check the
directory layout on your Gitaly server to be sure.

Omnibus installations:

```ruby
# /etc/gitlab/gitlab.rb

# Avoid running unnecessary services on the gitaly server
postgresql['enable'] = false
redis['enable'] = false
nginx['enable'] = false
prometheus['enable'] = false
unicorn['enable'] = false
sidekiq['enable'] = false
gitlab_workhorse['enable'] = false

# Prevent database connections during 'gitlab-ctl reconfigure'
gitlab_rails['rake_cache_clear'] = false
gitlab_rails['auto_migrate'] = false

# Configure the gitlab-shell API callback URL. Without this, `git push` will
# fail. This can be your 'front door' GitLab URL or an internal load
# balancer.
gitlab_rails['internal_api_url'] = 'https://gitlab.example.com'

# Make Gitaly accept connections on all network interfaces. You must use
# firewalls to restrict access to this address/port.
gitaly['listen_addr'] = "0.0.0.0:8075"
gitaly['auth_token'] = 'abc123secret'

gitaly['storage'] = [
  { 'name' => 'default', 'path' => '/mnt/gitlab/default/repositories' },
  { 'name' => 'storage1', 'path' => '/mnt/gitlab/storage1/repositories' },
]

# To use tls for gitaly you need to add
gitaly['tls_listen_addr'] = "0.0.0.0:9999"
gitaly['certificate_path'] = "path/to/cert.pem"
gitaly['key_path'] = "path/to/key.pem"
```

Source installations:

```toml
# /home/git/gitaly/config.toml
listen_addr = '0.0.0.0:8075'
tls_listen_addr = '0.0.0.0:9999'

[tls]
certificate_path = /path/to/cert.pem
key_path = /path/to/key.pem

[auth]
token = 'abc123secret'

[[storage]]
name = 'default'
path = '/mnt/gitlab/default/repositories'

[[storage]]
name = 'storage1'
path = '/mnt/gitlab/storage1/repositories'
```

Again, reconfigure (Omnibus) or restart (source).

### Converting clients to use the Gitaly server

Now as the final step update the client machines to switch from using
their local Gitaly service to the new Gitaly server you just
configured. This is a risky step because if there is any sort of
network, firewall, or name resolution problem preventing your GitLab
server from reaching the Gitaly server then all Gitaly requests will
fail.

We assume that your Gitaly server can be reached at
`gitaly.internal:8075` from your GitLab server, and that your GitLab
NFS shares are mounted at `/mnt/gitlab/default` and
`/mnt/gitlab/storage1` respectively.

Omnibus installations:

```ruby
# /etc/gitlab/gitlab.rb
git_data_dirs({
  'default' => { 'path' => '/mnt/gitlab/default', 'gitaly_address' => 'tcp://gitaly.internal:8075' },
  'storage1' => { 'path' => '/mnt/gitlab/storage1', 'gitaly_address' => 'tcp://gitaly.internal:8075' },
})

gitlab_rails['gitaly_token'] = 'abc123secret'
```

Source installations:

```yaml
# /home/git/gitlab/config/gitlab.yml
gitlab:
  repositories:
    storages:
      default:
        path: /mnt/gitlab/default/repositories
        gitaly_address: tcp://gitaly.internal:8075
      storage1:
        path: /mnt/gitlab/storage1/repositories
        gitaly_address: tcp://gitaly.internal:8075

  gitaly:
    token: 'abc123secret'
```

Now reconfigure (Omnibus) or restart (source). When you tail the
Gitaly logs on your Gitaly server (`sudo gitlab-ctl tail gitaly` or
`tail -f /home/git/gitlab/log/gitaly.log`) you should see requests
coming in. One sure way to trigger a Gitaly request is to clone a
repository from your GitLab server over HTTP.

## TLS support

Gitaly supports TLS credentials for GRPC authentication. To be able to communicate
with a gitaly instance that listens for secure connections you will need to use `tls://` url
scheme in the `gitaly_address` of the corresponding storage entry in the gitlab configuration.

The admin needs to bring their own certificate as we do not provide that automatically.
The certificate to be used needs to be installed on all gitaly nodes and on all client nodes that communicate with it following procedures described in [GitLab custom certificate configuration](https://docs.gitlab.com/omnibus/settings/ssl.html#install-custom-public-certificates)

### Example TLS configuration

### Omnibus installations:

#### On client nodes:

```ruby
# /etc/gitlab/gitlab.rb
git_data_dirs({
  'default' => { 'path' => '/mnt/gitlab/default', 'gitaly_address' => 'tls://gitaly.internal:9999' },
  'storage1' => { 'path' => '/mnt/gitlab/storage1', 'gitaly_address' => 'tls://gitaly.internal:9999' },
})

gitlab_rails['gitaly_token'] = 'abc123secret'
```

#### On gitaly server nodes:

```ruby
gitaly['tls_listen_addr'] = "0.0.0.0:9999"
gitaly['certificate_path'] = "path/to/cert.pem"
gitaly['key_path'] = "path/to/key.pem"
```

### Source installations:

#### On client nodes:

```yaml
# /home/git/gitlab/config/gitlab.yml
gitlab:
  repositories:
    storages:
      default:
        path: /mnt/gitlab/default/repositories
        gitaly_address: tls://gitaly.internal:9999
      storage1:
        path: /mnt/gitlab/storage1/repositories
        gitaly_address: tls://gitaly.internal:9999

  gitaly:
    token: 'abc123secret'
```

#### On gitaly server nodes:

```toml
# /home/git/gitaly/config.toml
tls_listen_addr = '0.0.0.0:9999'

[tls]
certificate_path = '/path/to/cert.pem'
key_path = '/path/to/key.pem'
```

## Disabling or enabling the Gitaly service in a cluster environment

If you are running Gitaly [as a remote
service](#running-gitaly-on-its-own-server) you may want to disable
the local Gitaly service that runs on your GitLab server by default.

> 'Disabling Gitaly' only makes sense when you run GitLab in a custom
cluster configuration, where different services run on different
machines. Disabling Gitaly on all machines in the cluster is not a
valid configuration.

If you are setting up a GitLab cluster where Gitaly does not need to
run on all machines, you can disable the Gitaly service in your
Omnibus installation, add the following line to `/etc/gitlab/gitlab.rb`:

```ruby
gitaly['enable'] = false
```

When you run `gitlab-ctl reconfigure` the Gitaly service will be
disabled.

To disable the Gitaly service in a GitLab cluster where you installed
GitLab from source, add the following to `/etc/default/gitlab` on the
machine where you want to disable Gitaly.

```shell
gitaly_enabled=false
```

When you run `service gitlab restart` Gitaly will be disabled on this
particular machine.

## Troubleshooting Gitaly in production

Since GitLab 11.6, Gitaly comes with a command-line tool called
`gitaly-debug` that can be run on a Gitaly server to aid in
troubleshooting. In GitLab 11.6 its only sub-command is
`simulate-http-clone` which allows you to measure the maximum possible
Git clone speed for a specific repository on the server.

For an up to date list of sub-commands see [the gitaly-debug
README](https://gitlab.com/gitlab-org/gitaly/blob/master/cmd/gitaly-debug/README.md).
