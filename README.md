# Overview

This charm provides Keystone, the OpenStack identity service. Its target
platform is (ideally) Ubuntu LTS + OpenStack.

# Usage

The following interfaces are provided:

- nrpe-external-master: Used to generate Nagios checks.

- identity-service: OpenStack API endpoints request an entry in the
  Keystone service catalog + endpoint template catalog. When a relation is
  established, Keystone receives: `service_name`, `region`, `public_url`,
  `admin_url` and `internal_url`. It first checks that the requested service is
  listed as a supported service. This list should stay updated to support
  current OpenStack core services. If the service is supported, an entry in the
  service catalog is created, an endpoint template is created and an admin token
  is generated. The other end of the relation receives the token as well as
  info on which ports Keystone is listening on.

- keystone-service: This is currently only used by Horizon/dashboard
  as its interaction with Keystone is different from other OpenStack API
  services. That is, Horizon requests a Keystone role and token exists. During
  a relation, Horizon requests its configured default role and Keystone
  responds with a token and the auth + admin ports on which Keystone is
  listening.

- identity-admin: Charms use this relation to obtain the credentials
  for the admin user. This is intended for charms that automatically provision
  users, tenants, etc. or that otherwise automate using the OpenStack cluster
  deployment.

- identity-notifications: Used to broadcast messages to any services
  listening on the interface.

- identity-credentials: Charms use this relation to obtain keystone
  credentials without creating a service catalog entry. Set 'username'
  only on the relation and keystone will set defaults and return
  authentication details. Possible relation settings:
  - `username` Username to be created.
  - `project` Project (tenant) name to be created. Defaults to services
              project.
  - `requested_roles` Comma delimited list of roles to be created.
  - `requested_grants` Comma delimited list of roles to be granted.
                       Defaults to Admin role.
  - `domain` Keystone v3 domain the user will be created in. Defaults
             to the Default domain.

## Database

Keystone requires a database. The charm supports relation to a shared database
server through the `mysql-shared` interface. When a new data store is
configured, the charm ensures the minimum administrator credentials exist (as
configured in charm configuration)

## HA/Clustering

There are two mutually exclusive high availability options: using virtual
IP(s) or DNS. In both cases, a relationship to hacluster is required which
provides the corosync backend HA functionality.

To use virtual IP(s), the clustered nodes must be on the same subnet such that
the VIP is a valid IP on the subnet for one of the node's interfaces and each
node has an interface in said subnet. The VIP becomes a highly-available API
endpoint.

At a minimum, the config option `vip` must be set in order to use virtual IP
HA. If multiple networks are being used, a VIP should be provided for each
network, separated by spaces. Optionally, `vip_iface` or `vip_cidr` may be
specified.

To use DNS high availability, there are several prerequisites. However, DNS HA
does not require the clustered nodes to be on the same subnet.
Currently, the DNS HA feature is only available for MAAS 2.0 or greater
environments. MAAS 2.0 requires Juju 2.0 or greater. The clustered nodes must
have static or "reserved" IP addresses registered in MAAS. The DNS hostname(s)
must be pre-registered in MAAS before use with DNS HA.

At a minimum, the configuration option `dns-ha` must be set to true and at
least one of `os-public-hostname`, `os-internal-hostname` or
`os-admin-hostname` must be set in order to use DNS HA. One or more of the
above hostnames may be set.

The charm will throw an exception in the following circumstances:

- If neither `vip` nor `dns-ha` is set and the charm is related to hacluster

- If both `vip` and `dns-ha` are set as they are mutually exclusive

- If `dns-ha` is set and none of the `os-{admin,internal,public}-hostname`
  configuration options are set

## TLS/HTTPS

Support for TLS and HTTPS endpoints can be enabled through configuration
options.

To enable TLS and HTTPS endpoints with a certificate signed by your own
Certificate Authority, set the following configuration options:

- `ssl_ca`

- `ssl_cert`

- `ssl_key`

Example bundle usage:

    keystone:
      charm: cs:keystone
      num_units: 1
      options:
        ssl_ca:   include-base64://path-to-PEM-formatted-ca-data
        ssl_cert: include-base64://path-to-PEM-formatted-certificate-data
        ssl_key:  include-base64://path-to-PEM-formatted-key-data

> **Note**: If your certificate is signed by a Certificate Authority present in
  the CA Certificate Store in operating systems used in your deployment, you do
  not need to provide the `ssl_ca` configuration option.

> **Note**: The `include-base64` bundle keyword tells Juju to source a file and
  Base64 encode it before storing it as a configuration option value. The path
  can be absolute or relative to the location of the bundle file.

## Spaces

This charm supports the use of Juju Network Spaces, allowing the charm to be
bound to network space configurations managed directly by Juju. This is only
supported with Juju 2.0 and above.

API endpoints can be bound to distinct network spaces supporting the network
separation of public, internal and admin endpoints.

Access to the underlying MySQL instance can also be bound to a specific space
using the shared-db relation.

To use this feature, use the --bind option when deploying the charm:

    juju deploy keystone --bind \
       "public=public-space \
        internal=internal-space \
        admin=admin-space \
        shared-db=internal-space"

Alternatively, these can also be provided as part of a Juju native bundle
configuration:

```yaml
    keystone:
      charm: cs:xenial/keystone
      num_units: 1
      bindings:
        public: public-space
        admin: admin-space
        internal: internal-space
        shared-db: internal-space
```

NOTE: Spaces must be configured in the underlying provider prior to attempting
to use them.

NOTE: Existing deployments using `os\-\*-network` configuration options will
continue to function; these options are preferred over any network space
binding provided if set.

## Policy Overrides

Policy overrides is an **advanced** feature that allows an operator to override
the default policy of an OpenStack service. The policies that the service
supports, the defaults it implements in its code, and the defaults that a charm
may include should all be clearly understood before proceeding.

> **Caution**: It is possible to break the system (for tenants and other
  services) if policies are incorrectly applied to the service.

Policy statements are placed in a YAML file. This file (or files) is then (ZIP)
compressed into a single file and used as an application resource. The override
is then enabled via a Boolean charm option.

Here are the essential commands (filenames are arbitrary):

    zip overrides.zip override-file.yaml
    juju attach-resource keystone policyd-override=overrides.zip
    juju config keystone use-policyd-override=true

See appendix [Policy Overrides][cdg-appendix-n] in the [OpenStack Charms
Deployment Guide][cdg] for a thorough treatment of this feature.

## Security Compliance config option "password-security-compliance"

The `password-security-compliance` configuration option sets the
`[security_compliance]` section of Keystone's configuration file.

The configuration option is a YAML dictionary, that is one level deep, with the
following keys (and value formats).

```yaml
lockout_failure_attempts: <int>
lockout_duration: <int>
disable_user_account_days_inactive: <int>
change_password_upon_first_use: <boolean>
password_expires_days: <int>
password_regex: <string>
password_regex_description: <string>
unique_last_password_count: <int>
minimum_password_age: <int>
```

It can be set by placing the keys and values in a file and then using the Juju
command:

    juju config keystone --file path/to/config.yaml

Note that, in this, case the `config.yaml` file requires the YAML key
`password-security-compliance:` with the desired config keys and values on the
following lines, indented for a dictionary.

> **Note**: Please ensure that the page [Security compliance and PCI-DSS][SCPD]
  is consulted before setting these option.

The charm will protect service accounts (accounts requested by other units that
are in the service domain) against being forced to change their password.
Operators should also ensure that any other accounts are protected as per the
above referenced note.

If the config value cannot be parsed as YAML and/or the options are not
able to be parsed as their indicated types then the charm will enter a blocked
state until the config value is changed.

## Token Support

As the keystone charm supports multiple releases of the OpenStack software, it
also supports two keystone token systems: UUID and Fernet. The capabilities are:

- pre 'ocata': UUID tokens only.
- ocata and pike: UUID or Fernet tokens, configured via the 'token-provider'
  configuration parameter.
- rocky and later: Fernet tokens only.

Fernet tokens were added to OpenStack to solve the problem of keystone being
required to persist tokens to a common database (cluster) like UUID tokens,
and solve the problem of size for PKI or PKIZ tokens.

For further information, please see [Fernet - Frequently Asked
Questions](https://docs.openstack.org/keystone/pike/admin/identity-fernet-token-faq.html).

### Theory of Operation

Fernet keys are used to generate tokens; they are generated by keystone
and have an expiration date. The key repository is a directory, and each
key is an integer number, with the highest number being the primary key. Key
'0' is the staged key, that will be the next primary. Other keys are secondary
keys.

New tokens are only ever generated from the primary key, whilst the secondary
keys are used to validate existing tokens. The staging key is not used to
generate tokens but can be used to validate tokens as the staging key might be
the new primary key on the master due to a rotation and the keys have not yet
been synchronised across all the units.

Fernet keys need to be rotated at periodic intervals, and the keys need to be
synchronised to each of the other keystone units. Keys should only be rotated
on the master keystone unit and must be synchronised *before* they are rotated
again. *Over rotation* occurs if a unit rotates its keys such that there is
no suitable decoding key on another unit that can decode a token that has been
generated on the master. This happens if two key rotations are done on the
master before a synchronisation has been successfully performed. This should
be avoided. Over rotations can also cause validation keys to be removed
*before* a token's expiration which would result in failed validations.

There are 3 parts to the **Key Rotation Strategy**:

1. The rotation frequency
2. The token lifespan
3. The number of active keys

There needs to be at least 3 keys as a minimum. The actual number of keys is
determined by the *token lifespan* and the *rotation frequency*. The
*max_active_keys* must be one greater than the *token lifespan* / *rotation
frequency*

To quote from the [FAQ](https://docs.openstack.org/keystone/queens/admin/identity-fernet-token-faq.html):

        The number of max_active_keys for a deployment can be determined by
        dividing the token lifetime, in hours, by the frequency of rotation in
        hours and adding two. Better illustrated as:

### Configuring Key Lifetime

In the keystone charm, the _rotation frequency_ is calculated
automatically from the `token-expiration` and the `fernet-max-active-keys`
configuration parameters. For example, with an expiration of 24 hours and
6 active keys, the rotation frequency is calculated as:

```python
token_expiration = 24   # actually 3600, as it's in seconds
max_active_keys = 6
rotation_frequency = token_expiration / (max_active_keys - 2)
```

Thus, the `fernet-max-active-keys` can never be less than 3 (which is
enforced in the charm), which would make the rotation frequency the *same*
as the token expiration time.

NOTE: To increase the rotation frequency, _either_ increase
`fernet-max-active-keys` or reduce `token-expiration`, and, to decrease
rotation frequency, do the opposite.

NOTE: If the configuration parameters are used to significantly reduce the
key lifetime, then it is possible to over-rotate the verification keys
such that services will hold tokens that cannot be verified but haven't
yet expired. This should be avoided by only making small changes and
verifying that current tokens will still be able to be verified. In
particular, `fernet-max-active-keys` affects the rotation time.

### Upgrades

When an older keystone charm is upgraded to this version, NO change will
occur to the token system. That is, an ocata system will continue to use
UUID tokens. In order to change the token system to Fernet, change the
`token-provider` configuration item to `fernet`. This will switch the
token system over. There may be a small outage in the _control plane_,
but the running instances will be unaffected.

# Bugs

Please report bugs on [Launchpad][lp-bugs-charm-keystone].

For general charm questions refer to the OpenStack [Charm Guide][cg].

<!-- LINKS -->

[cg]: https://docs.openstack.org/charm-guide
[cdg]: https://docs.openstack.org/project-deploy-guide/charm-deployment-guide
[cdg-appendix-n]: https://docs.openstack.org/project-deploy-guide/charm-deployment-guide/latest/app-policy-overrides.html
[lp-bugs-charm-keystone]: https://bugs.launchpad.net/charm-keystone/+filebug
[SCPD]: https://docs.openstack.org/keystone/latest/admin/configuration.html#security-compliance-and-pci-dss
