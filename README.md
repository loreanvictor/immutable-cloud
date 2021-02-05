<div align="center">

<img src="immutable-cloud.svg" width="256"/>

# Immutable Cloud

(micro-)service architecture with resilience towards API changes

</div>

<br>

The API of any (micro-)service corresponds to its distinct set of functionalities. As these functionalities (inevitably) change overtime, so will its API.
Such a change would need to be propagated to all dependent code (code that consumes the API), which can be a resource-consuming and particularly error-prone process, often resulting in downtimes and runtime errors in production.

A particular solution is to keep multiple _versions_ of the service (with different APIs) available, as to ensure availability during the time all consumer code transitions from a previous version to the next. Maintaining multiple versions of some API can be demanding and complex, a complexity which multiplies by number of (micro-)services whose API is changing constanty.

_Immutable Cloud_ outlines a solution structure for this problem. It provides mechanisms to conveniently rolling out new versions of multiple APIs, detecting how many distinct versions need to be available at any time, which APIs can be safely deprecated at anytime, etc.

👉 The key words "**MUST**", "**MUST NOT**", "**REQUIRED**", "**SHALL**", "**SHALL NOT**", "**SHOULD**", "**SHOULD NOT**", "**RECOMMENDED**",  "**MAY**", and "**OPTIONAL**" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

<br>

# Overview

The core idea of _Immutable Cloud_ is to use [semantic versioning](https://semver.org) for the APIs. (Micro-)services are _immutable_ as in each specific
version of the API will be provided by exactly the same program (code), so each version, once released, can no longer be changed. _Version Registries_ **MUST**
be provided for each service, which outline available versions of its API.

Consumers **MUST** be available in the network to be queried for the exact version of APIs they consume. This availability is either direct (e.g. consumer is a service itself, running on the cluster) or by proxy (e.g. proxy for a web-app). Consumers **MUST** provide required version on each request, and **MUST** be able to handle redirects (to the URI of the proper version of their target API), with **OPTIONAL** caching of redirects for enhanced performance.

Updates to an API is broken into two atomic actions: _Release_, which is making a new version of the API available without affecting availability of other versions, and _Deprecation_, which is making an available version of the API no longer available. A service is said to be _stateless_ if it can have a _release_ without any _deprecation_, and _stateful_ otherwise. Generally speaking, a _release_ is always a safe action (no consumer will have the APIs it depends on not available), while _deprecation_ needs to be verified and checked across the network.

Detailed definitions of these concepts and description of these mechanisms are provided in following sections.

<br>

# Definitions

## Service

A _service_ is defined by:
- A unique [URI](https://tools.ietf.org/html/rfc3986), referred to as the _main URI_
- A [semantic version](https://semver.org), in form of `x.y.z`, referred to as the _latest version_
- A unique version registry URI, which is a mapping of all available versions (all semantic versions) to dinstinct URIs (for each version)

👉 All services **MUST** have a unique and consistent endpoint that returns the corresponding version registry URI. The exact protocol for this endpoint
is up to the implementation, but it should be consistent amongst ALL services. This is called the _registry endpoint_.

👉 Mapping of each version to its URI **MUST** be _immutable_, i.e. the URI for a specific version **MUST** always be the same.

👉 Each version of a service **MUST** be _immutable_, i.e. all requests to various endpoints of its URI **MUST** always have exactly the same results.

👉 At any given point in time, requests to endpoints of the _main URI_ (except for the registry endpoint) **MUST** be identical to requests
to corresponding URI of a particular version (outlined by the registry). i.e. sending requests to the _latest version_ should be equivalent to sending requests
to a particular version.

👉 The _main URI_ is exempt from immutability, as overtime it can refer to different versions of the API.

#### Example

An authentication service is identified by `https://auth.my-cloud`, with latest version being `2.0.1`. Its version registry
endpoint is at `https://auth.my-cloud/__api/version-registry`, which returns `https://versions.auth.my-cloud`. Fetching `https://versions.auth.my-cloud`
results in following mapping:

```json
{
  "2.0.1": "https://auth.my-could/2.3.1/",
  "1.4.7": "https://auth.my-cloud/1.4.7/"
}
```

<br>

## Consumer

A _consumer_ is defined by a unique URI, on which the consumer will return a list of APIs it consumes and versions it requires.

👉 APIs might have _out-of-cluster consumers_, for example mobile or web apps, or external costumers consuming APIs. For such consumers, proxies should exist within the network and with a unique URI which responds as described above. For example, you might have a JSON file available on the network as a proxy for your web-app, which outlines APIs it needs and versions it needs, and update this JSON file automatically via some CI pipeline whenever the web-app is being updated.

#### Example:

The proxy for the web-app, i.e. `https://web-app-proxy.my-cloud`, which returns the following JSON:

```json
{
  "https://auth.my-cloud": "2.1.9",
  "https://wallet.my-cloud": "1.0.5"
}
```
