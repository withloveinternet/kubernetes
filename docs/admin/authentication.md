<!-- BEGIN MUNGE: UNVERSIONED_WARNING -->


<!-- END MUNGE: UNVERSIONED_WARNING -->

# Authentication Plugins

Kubernetes uses client certificates, tokens, or http basic auth to authenticate users for API calls.

**Client certificate authentication** is enabled by passing the `--client_ca_file=SOMEFILE`
option to apiserver. The referenced file must contain one or more certificates authorities
to use to validate client certificates presented to the apiserver. If a client certificate
is presented and verified, the common name of the subject is used as the user name for the
request.

**Token authentication** is enabled by passing the `--token_auth_file=SOMEFILE` option
to apiserver.  Currently, tokens last indefinitely, and the token list cannot
be changed without restarting apiserver.  We plan in the future for tokens to
be short-lived, and to be generated as needed rather than stored in a file.

The token file format is implemented in `plugin/pkg/auth/authenticator/token/tokenfile/...`
and is a csv file with 3 columns: token, user name, user uid.

When using token authentication from an http client the apiserver expects an `Authorization`
header with a value of `Bearer SOMETOKEN`.

**Basic authentication** is enabled by passing the `--basic_auth_file=SOMEFILE`
option to apiserver. Currently, the basic auth credentials last indefinitely,
and the password cannot be changed without restarting apiserver. Note that basic
authentication is currently supported for convenience while we finish making the
more secure modes described above easier to use.

The basic auth file format is implemented in `plugin/pkg/auth/authenticator/password/passwordfile/...`
and is a csv file with 3 columns: password, user name, user id.

When using basic authentication from an http client, the apiserver expects an `Authorization` header
with a value of `Basic BASE64ENCODEDUSER:PASSWORD`.

## Plugin Development

We plan for the Kubernetes API server to issue tokens
after the user has been (re)authenticated by a *bedrock* authentication
provider external to Kubernetes.  We plan to make it easy to develop modules
that interface between Kubernetes and a bedrock authentication provider (e.g.
github.com, google.com, enterprise directory, kerberos, etc.)


<!-- BEGIN MUNGE: IS_VERSIONED -->
<!-- TAG IS_VERSIONED -->
<!-- END MUNGE: IS_VERSIONED -->


<!-- BEGIN MUNGE: GENERATED_ANALYTICS -->
[![Analytics](https://kubernetes-site.appspot.com/UA-36037335-10/GitHub/docs/admin/authentication.md?pixel)]()
<!-- END MUNGE: GENERATED_ANALYTICS -->
