---
layout: "docs"
page_title: "Active Directory - Secrets Engines"
sidebar_current: "docs-secrets-active-directory"
description: |-
  The Active Directory secrets engine for Vault generates passwords dynamically based on
  roles.
---

# Active Directory Secrets Engine

The Active Directory (AD) secrets engine rotates AD passwords dynamically,
and is designed for a high-load environment where many instances may be accessing
a shared password simultaneously. With a simple set up and a simple creds API,
it doesn't require instances to be manually registered in advance to gain access. 
As long as access has been granted to the creds path via a method like 
[AppRole](https://www.vaultproject.io/api/auth/approle/index.html), they're available.

Passwords are lazily rotated based on preset TTLs and can have a length configured to meet 
your needs.

## A Note on Escaping

**It is up to the administrator** to provide properly escaped DNs. This
includes the user DN, bind DN for search, and so on.

The only DN escaping performed by this method is on usernames given at login
time when they are inserted into the final bind DN, and uses escaping rules
defined in RFC 4514.

Additionally, Active Directory has escaping rules that differ slightly from the
RFC; in particular it requires escaping of '#' regardless of position in the DN
(the RFC only requires it to be escaped when it is the first character), and
'=', which the RFC indicates can be escaped with a backslash, but does not
contain in its set of required escapes. If you are using Active Directory and
these appear in your usernames, please ensure that they are escaped, in
addition to being properly escaped in your configured DNs.

For reference, see [RFC 4514](https://www.ietf.org/rfc/rfc4514.txt) and this
[TechNet post on characters to escape in Active
Directory](http://social.technet.microsoft.com/wiki/contents/articles/5312.active-directory-characters-to-escape.aspx).

## Quick Setup

Most secrets engines must be configured in advance before they can perform their
functions. These steps are usually completed by an operator or configuration
management tool.

1. Enable the Active Directory secrets engine:

    ```text
    $ vault secrets enable ad
    Success! Enabled the ad secrets engine at: ad/
    ```

    By default, the secrets engine will mount at the name of the engine. To
    enable the secrets engine at a different path, use the `-path` argument.

2. Configure the credentials that Vault uses to communicate with Active Directory 
to generate passwords:

    ```text
    $ vault write ad/config \
        username=$USERNAME \
        password=$PASSWORD \
        urls=ldap://138.91.247.105 \
        dn='dc=example,dc=com'
    ```

    The `$USERNAME` and `$PASSWORD` given must be of a high enough access level that
    they can be used for modifying passwords. Typically, this will be a domain admin.

3. Configure a role that maps a name in Vault to an account in Active Directory.
When applications request passwords, password rotation settings will be managed by
this role.

    ```text
    $ vault write ad/roles/my-application \
        service_account_name="my-application@example.com"
    ```
    
4. Grant "my-application" access to its creds at `ad/creds/my-application` using an 
auth method like [AppRole](https://www.vaultproject.io/api/auth/approle/index.html).

## Configuration

### Connection parameters

* `urls` (string, required) - The LDAP server to connect to. Examples: `ldap://ldap.myorg.com`, `ldaps://ldap.myorg.com:636`. This can also be a comma-delineated list of URLs, e.g. `ldap://ldap.myorg.com,ldaps://ldap.myorg.com:636`, in which case the servers will be tried in-order if there are errors during the connection process.
* `starttls` (bool, optional) - Defaults to true. If true, issues a `StartTLS` command after establishing an unencrypted connection.
* `insecure_tls` - (bool, optional) - Defaults to false. If true, skips LDAP server SSL certificate verification - insecure, use with caution!
* `certificate` - (string, optional) - CA certificate to use when verifying LDAP server certificate, must be x509 PEM encoded.
* `tls_min_version` - (string, optional) - Defaults to `tls12`. Designates the minimum TLS version to use when communicating. Example: `tls12`
* `tls_max_version` - (string, optional) - Defaults to `tls12`. Designates the maximum TLS version to use when communicating. Example: `tls10`

### Binding parameters

* `dn` (string, required) - Distinguished name of object to bind when performing user and group search. Example: `cn=vault,ou=Users,dc=example,dc=com`
* `username` (string, required) - Username to use along with `dn` of sufficient privilege to modify passwords.
* `password` (string, required) - Password to use along with `dn`.

### Password rotation parameters

* `ttl` (string, optional) - The default password time-to-live in seconds. Once the ttl has passed, a password will be rotated the next time it's requested. Defaults to the number of seconds in 32 days.
* `max_ttl` (string, optional) - The maximum password time-to-live in seconds. No role will be allowed to set a custom ttl greater than the `max_ttl`. Defaults to the number of seconds in 32 days.
* `password_length` (string, optional) - The desired password length. Defaults to 64. Minimum is 14. Note: to meet complexity requirements, all passwords begin with "?@09AZ".

## Roles

### Parameters

* `service_account_name` (string, required) - The name of a pre-existing service account in Active Directory that maps to this role.
* `ttl` (string, optional) - The password time-to-live in seconds. Defaults to the configuration `ttl` if not provided.

## FAQ

### What if someone directly rotates an Active Directory password that Vault is managing?

If an administrator at your company rotates a password that Vault is managing, the next time an application asks _Vault_ 
for that password, Vault won't know it. 

To maintain that application's up-time, Vault will need to return to a state of knowing the password. Vault will generate 
a new password, update it, and return it to the application(s) asking for it.

Thus, we wouldn't recommend that administrators directly rotate the passwords for accounts that Vault is managing. This
may lead to behavior the administrator wouldn't expect, like finding very quickly afterwards that their new password
has already been changed. 

The password `ttl` on a role can be updated at any time to ensure that the responsibility of updating passwords can be 
left to Vault, rather than requiring manual administrator updates.

### How does this feature work with Managed Service Accounts?

Managed Service Accounts are a feature where, in some situations, Active Directory can be set up to rotate passwords for you.
Vault can be used alongside Managed Service Accounts, but on separate accounts.

If Vault were set up to rotate a Managed Service Account's password, there would effectively be _two_ entities rotating
passwords for that account. This would create a strange situation where the password my be rotated by Active Directory,
and then Vault would also be forced to rotate the password in order to know and return it again.

We're not aware of any use case for this setup and would advise against it. Please use one _or_ the other as best fits
your needs.

### How does this feature work with Group Managed Service Accounts?

Group Managed Service Accounts are a successor to Managed Service Accounts. Please see their discussion above.

### Why does Vault return the last password in addition to the current one?

Active Directory promises _eventual consistency_, which means that new passwords may not be propagated to all instances
immediately. To deal with this, Vault returns the current password with the last password if it's known. That way, if a new
password isn't fully operational, the last password can also be used.