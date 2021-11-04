!!! important

    - An average pool operator may not require this component at all. Please verify if it is required for your use as mentioned [here](../build.md#components)
    - Ensure that you have setup [DBSync](../Build/dbsync.md) and that it is in sync atleast to Mary fork before you proceed. *IF* you're participating in Koios services, ensure that you're using [latest dbsync release](https://github.com/input-output-hk/cardano-db-sync/releases/latest)

### What is gRest

gRest is an open source implementation of a `query layer built over dbsync using PostgREST and HAProxy`. The package is built as part of [Koios](https://www.koios.rest) team's efforts to unite community individual stream of work together and give back a more aligned structure to query dbsync and adopt standardisation to queries utilising open-source tooling as well as collaboration. In addition to these, there are also accessibility features to deploy rules for failover, do healthchecks, set up priorities, have ability to prevent DDoS attacks, provide timeouts, report tips for analysis over a longer period, etc - which can prove to be really useful when performing any analysis for instances.

### Components

1. [PostgREST](https://postgrest.org/en/latest):  
    An RPC JSON interface for any PostgreSQL database (in our case, database served via `cardano-db-sync`) to provide a RESTful Web Service. The endpoints of PostgREST in itself are essentially the table/functions defined in elected schema via grest config file. You can read more about advanced query syntax using PostgREST API [here](https://postgrest.org/en/latest/api.html), but we will provide a simpler view using examples towards the end of the page. It is an easy alternative - with almost no overhead as it directly serves the underlying database as an API, as compared to `Cardano GraphQL` component (which may often have lags). Some of the other advantages of PostgREST over graphql based projects are also performance, being stateless, 0 overhead, support for JWT / native Postgres DB authentication against the Rest Interface as well.

2. [HAProxy](http://cbonte.github.io/haproxy-dconv/2.4/configuration.html):  
    An easy gateway proxy that automatically provides failover/basic DDoS protection, specify rules management for load balancing, setup multiple frontend/backends, provide easy means to have TLS enabled for public facing instances, etc. You may alter the settings for proxy layer as per your SecOps preferences. This component is optional (eg: if you prefer to expose your PostgREST server itself, you can do so using similar steps below).

### Setup gRest services {: id="setup"}

To start with you'd want to ensure your current shell session has access to Postgres credentials, continuing from examples from the above mentioned [Sample Postgres deployment guide](../Appendix/postgres.md).

``` bash
cd $CNODE_HOME/priv
PGPASSFILE=$CNODE_HOME/priv/.pgpass
psql cexplorer
```

Ensure that you can connect to your Postgres DB fine using above (quit from psql once validated using `\q`). As part of `prereqs.sh` execution, you'd find setup-grest.sh file made available in `${CNODE_HOME}/scripts` folder, which will help you automate installation of PostgREST, HAProxy as well as brings in latest queries/functions provided via Koios to your instances.

!!! warning "Warning"
    As of now, gRest services are in alpha stage - while can be utilised, please remember there may be breaking changes and every collaborator is expected to work with the team to keep their instances up-to-date using alpha branch.

Familiarise with the usage options for the setup script , the syntax can be viewed as below:

``` bash
cd "${CNODE_HOME}"/scripts
./setup-grest.sh -h
#
# Usage: setup-grest.sh [-f] [-i [p][r][m][c][d]] [-u] [-b <branch>]
# 
# Install and setup haproxy, PostgREST, polling services and create systemd services for haproxy, postgREST and dbsync
# 
# -f    Force overwrite of all files including normally saved user config sections
# -i    Set-up Components individually. If this option is not specified, components will only be installed if found missing (eg: -i prcd)
#     p    Install/Update PostgREST binaries by downloading latest release from github.
#     r    (Re-)Install Reverse Proxy Monitoring Layer (haproxy) binaries and config
#     m    Install/Update Monitoring agent scripts
#     c    Overwrite haproxy, postgREST configs
#     d    Overwrite systemd definitions
# -u    Skip update check for setup script itself
# -q    Run all DB Queries to update on postgres (includes creating grest schema, and re-creating views/genesis table/functions/triggers and setting up cron jobs)
# -b    Use alternate branch of scripts to download - only recommended for testing/development (Default: master)
#
```

To run the setup with typical options, you may want to use:
``` bash
./setup-grest.sh -f -q
```

Similarly - if instead, you'd like to re-install all components as well as force overwrite all configs and queries, you may run:
``` bash
./setup-grest.sh -f -i prmcd -q
```

Another example could be to preserve your config, but only update queries using an alternate branch (eg: `alpha`). To do so, you may run:
``` bash
./setup-grest.sh -q -b alpha
```

Please ensure to follow the on-screen instructions, if any (for example restarting deployed services, or updating configs to specify correct target postgres URLs/enable TLS/add peers etc in `${CNODE_HOME}/priv/grest.conf` and `${CNODE_HOME}/files/haproxy.cfg`).

The default ports used will make haproxy instance available at port 8053 or 8453 if TLS is enabled (you might want to enable firewall rule to open this port to services you would like to access). If you want to prevent unauthenticated access to grest schema, uncomment the jwt-secret and specify a custom `secret-token`.

### Enable TLS on HAProxy {: id="tls"}

In order to enable SSL on your haproxy, all you need to do is edit the file `${CNODE_HOME}/files/haproxy.cfg` and update the *frontend app* section to disable normal bind and enable ssl bind. Note that the server.pem referred in example below should contain certificate chain as well as the private key. If you're not familiar with how to configure TLS OR would not like to buy one, you can find tips on how to create a TLS certificate  for free via LetsEncrypt using tutorials [here](https://letsencrypt.org/getting-started/). Once you do have a TLS Certificate generated, you need to chain the private key and full chain cert together in a file - `/etc/ssl/server.pem` in example below:

```
frontend app
  bind 0.0.0.0:8053
  http-request replace-value Host (.*):8053 :8453
  redirect scheme https code 301 if !{ ssl_fc }
  
frontend app-secured
  bind :8453 ssl crt /etc/ssl/server.pem no-sslv3
  http-request track-sc0 src table flood_lmt_rate
  http-request deny deny_status 429 if { sc_http_req_rate(0) gt 100 }
  default_backend grest_core
```
Restart haproxy service for changes to take effect.

### Performance Tuning Considerations

While the defaults on your system should not cause you any trouble, note that haproxy relies on ephemeral ports available on your system to be able to redirect frontend requests to backend servers. The four important configuration settings in your `/etc/sysctl.conf` files would be:

```
net.ipv4.ip_local_port_range="1024 65534"
net.core.somaxconn=65534
net.ipv4.tcp_rmem=4096 16060 64060
net.ipv4.tcp_wmem=4096 16384 262144
```
Again, defaults should be fine for minimal usage, you do not need to tinker with above unless you expect a very high amount of load on your frontend. We intentionally do not set these for you as system tuning activities should be done very carefully, understanding what you're modifying and their impacts - these would be part of your SysOps + SecOps practices.

### Validation

With the setup, you also have a `checkstatus.sh` script, which will query the Postgres DB instance via haproxy (coming through postgREST), and only show an instance up if the latest block in your DB instance is within 180 seconds.

!!! warning "Important"
    While currently the HAProxy config only checks for tip, there will be test cases added for validating each endpoint in future. If you'd like to participate in joining to the elastic cluster via Koios, please raise a PR request by editing topology files in [this folder](https://github.com/cardano-community/guild-operators/tree/alpha/files/grest) to do so!!

If you were using `guild` network, you could do a couple of very basic sanity checks as per below:

1. To query active stake for pool `pool1z2ry6kxywgvdxv26g06mdywynvs7jj3uemnxv273mr5esukljsr` in epoch `122`, we can execute the below:
``` bash
curl -d _pool_bech32=pool1z2ry6kxywgvdxv26g06mdywynvs7jj3uemnxv273mr5esukljsr -d _epoch_no=122 -s http://localhost:8053/rpc/pool_active_stake
## {"active_stake_sum" : 19409732875}
```

2. To check latest owner key(s) for a given pool `pool1z2ry6kxywgvdxv26g06mdywynvs7jj3uemnxv273mr5esukljsr`, you can execute the below:
``` bash
curl -d _pool_bech32=pool1z2ry6kxywgvdxv26g06mdywynvs7jj3uemnxv273mr5esukljsr -s http://localhost:8050/rpc/pool_owners
## [{"owner" : "stake_test1upx5p04dn3t6dvhfh27744su35vvasgaaq565jdxwlxfq5sdjwksw"}, {"owner" : "stake_test1uqak99cgtrtpean8wqwp7d9taaqkt9gkkxga05m5azcg27chnzfry"}]
```

You may want to explore what all endpoints come out of the box, and test them out, to do so - refer to [API documentation](https://api.koios.rest) for OpenAPI3 documentation. Each endpoint has a pre-filled example for mainnet and connects by default to primary Koios endpoint, allowing you to test endpoints and if needed - grab the `curl` commands to start testing yourself against your local or remote instances.