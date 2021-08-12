# powerdns-auth-proxy

**Please don't use this fork. Use [the original one](https://github.com/catalyst/powerdns-auth-proxy) instead.**

This proxy implements a subset of the PowerDNS API with more flexible authentication and per-zone access control.

More information about the PowerDNS API is available at https://doc.powerdns.com/md/httpapi/api_spec/.

Future versions of this software will also support per-RRset ACLs.

## Configuration

The proxy expects to read a `proxy.ini` file which defines ACLs, for instance:

````
[pdns]
api-key = 7128ae9eb680a14390ee22a988a9d01a
api-url = http://127.0.0.1:8081/api/v1/servers/localhost
override-soa_edit_api = INCEPTION-INCREMENT
override-nameservers = ns1.example.com. ns2.example.com. ns3.example.com. ns4.example.com.
override-kind = Master

# This user will be able to create a zone called "example.org." if it doesn't already exist.
[user:demo-example-org]
key = dd70d1b0eccd79a0cf5d79ddf6672dce
allow-suffix-creation = example.org.

# The `account` in PowerDNS backend will be set to this value
# If not specified, the user will not be allowed to create zones
# or update their metadata, unless trusted-meta is set to "True"
#
# This has _no_ bearing on whether the user is allowed to access
# a zone or not. Use `allowed-zones` and `allowed-zone-fn`.
account = demo-example-org

# Alternatively, setting trusted-meta to "True" will allow the user
# to create domains or modify their metadata without having the
# `account` tag overridden.
#
# The user will still be subject to zone and rrset restrictions.
trusted-meta = True

# Static list of allowed zones, space-separated
allowed-zones = example.org. another.zone.com.

# Will be used if allowed-zones does not contain the zone
#
# For structure of `zone`, refer to:
# https://doc.powerdns.com/md/httpapi/api_spec/#zones
#
# Be careful - This is eval()'d!
allowed-zone-fn = lambda zone: do_stuff_and_return_a_boolean(zone)

# If set to "True", then this user can delete zones they have access to
allow-zone-deletion = True

# If not specified, the user can manipulate any record in the allowed zones
#
# For structure of `rrset`, refer to:
# https://doc.powerdns.com/md/httpapi/api_spec/#zones
#
# Be careful - This is eval()'d!
allowed-rrset-fn = lambda zone, rrset: False

````

This specifies the API key and URL used to connect to the PowerDNS backend, as well as allowing for certain zone metadata items which will be overridden during zone creation and metadata updates.

By default, users cannot do anything. You can specify `allowed-zones` and/or `allowed-zone-fn`

If `account` is set, the user is able to create zones and update their metadata and `account` will be overridden to that value to prevent people from moving zones around between accounts they don't control. What you override will likely depend on how much access users need in your environment.

Keys should be generated using something like `dd if=/dev/urandom bs=1 count=16 | xxd -ps` to ensure they have sufficient entropy.

## Installation

We run the application with `waitress` launched by `supervisord`, though any of the standard methods for deploying Flask 1.x applications should work. We recommend using a virtual env to install the dependencies of the application.

An example `supervisord` configuration would be:

````
[program:dnsapi]
command=/opt/dnsapi/venv/bin/waitress-serve --listen=127.0.0.1:8000 --call 'powerdns_auth_proxy:create_app'
directory=/opt/dnsapi/powerdns-auth-proxy
user=dnsapi
autostart=true
autorestart=true
````

## Running tests

### Setup

* Ubuntu / Debian

  The officially supported system for testing is Debian or Ubuntu. To start, install the the official PowerDNS 4.1.x upstream packages: `pdns-server` and `pdns-backend-sqlite3`.

* Arch

  The tests can also be run on Arch Linux. Install the `powerdns` package.

Additonally a few python modules need to be installed to run the tests (mainly `pytest`): `pip install -r requirements-dev.txt`. You can then run tests by running `pytest -v` inside the source directory.

## Authenticating

Either use HTTP Basic authentication or encode your username and key in the `X-API-Key` HTTP header with a `:` character separating the parts (e.g. add an `X-API-Key: username:key` header).

## Client compatibility

This proxy has been tested with:

* Terraform [PowerDNS provider](https://www.terraform.io/docs/providers/powerdns/index.html)
* Traefik [ACME pdns plugin](https://traefik.io/)
