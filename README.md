<div align="center">

<img src="immutable-cloud.svg" width="256"/>

# Immutable Cloud

micro-service architecture for synchronizing changing APIs

</div>

<br>

## Preface

This is a draft. Definitions are loose, rules are loose, etc.

<br>

## Definitions

- Cluster: a set of running services (perhaps across multiple machines) that are connected together via some network (internal network, the internet, etc).
- Server: some code providing an API. has a unique URI.
  - each API can have multiple running versions, which are served on different URIs. still the service as a whole is known by one singular URI.
  - instances providing the same version of the API **MUST** behave exactly the same, i.e. each version is _immutable_.
  - servers **MUST** redirect requests to the URI of the appropriate version.
  - servers **MUST** provide a version registry URI (which is unique per Server).
- Client: some code consuming some API.
  - clients **MUST** be accessible on the cluster. edge clients are to be proxied by some in-network accessible stand-in.
  - clients **MUST** to be able to manage redirects (ideally cache redirects)
  - clients **MUST** provide required version information on requests they make to Servers.
  - clients **MUST** provide a list of Servers they depend on, alongside versions they depend on, when queried by other nodes in the cluster.
- Version Registry: provides a list of available versions of the service, mapped to the unique URI of each version.
