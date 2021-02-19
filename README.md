<div align="center">

<img src="immutable-cloud.svg" width="256"/>

# Immutable Cloud

(micro-)service architecture with resilience towards API changes

</div>

<br>

# TLDR

This is an architectural guide for maintaining multiple versions of multiple [(micro-)services](#service) and keeping track of [consumers](#consumer) who depend on different versions of these services, while ensuring [safety](#operation-safety) of various operations (release, deprecation, rollbacks) automatically.

- 👉 Use [semantic versioning](https://semver.org/) for each service.
- 👉 Once deployed, never redeploy a particular version of a service (hence, _immutable cloud_).
- 👉 Maintain a [version registry](#service) for each service, outlining available versions and their URIs.
- 👉 Consumers must also be present in-cluster and outline the versions of the services they need.
  - For [out-of-cluster consumers](#consumer) (e.g. mobile apps), use in-cluster proxies.
- 👉 Maintain a [cluster topology](#cluster-topology), outlining which services are used by which consumers.
  - This can also be calculated on-demand by querying all consumers (or their proxies).
- 👉 Whenever possible, [release](#release) (a new version) and [deprecate](#deprecation) (an older version) independently.
  - Safety of each of these operations can be (automatically) checked.
  - In cases [where this is not possible](#stateful-and-stateless-services), still [analyze operation safety](#stateful-services) by looking at operations independently.
- 👉 In [version matching](#consumer-version-matching), implicitly treat patch numbers more flexibly.
  - This allows for patch rollbacks to be treated differently than minor / major rollbacks (see below).

<br>

# Preface

The API of any (micro-)service corresponds to its distinct set of functionalities. As these functionalities (inevitably) change overtime, so will its API.
Such a change would need to be propagated to all dependent code (code that consumes the API), which can be a resource-consuming and particularly error-prone process, often resulting in downtimes and runtime errors in production.

A particular solution is to keep multiple _versions_ of the service (with different APIs) available, as to ensure availability during the time all consumer code transitions from a previous version to the next. Maintaining multiple versions of some API can be demanding and complex, a complexity which multiplies by number of (micro-)services whose API is changing constanty.

_Immutable Cloud_ outlines a solution structure for this problem. It provides mechanisms for conveniently rolling out new versions of multiple APIs, detecting how many distinct versions need to be available at any time, which APIs can be safely deprecated at anytime, etc.

👉 The key words "**MUST**", "**MUST NOT**", "**REQUIRED**", "**SHALL**", "**SHALL NOT**", "**SHOULD**", "**SHOULD NOT**", "**RECOMMENDED**",  "**MAY**", and "**OPTIONAL**" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

<br>

# Contents

- [Overview](#overview)
- [Definitions](#definitions)
  - [Service](#service)
    - [Stateful and Stateless Services](#stateful-and-stateless-services)
  - [Consumer](#consumer)
  - [Cluster](#cluster)
- [Versioning](#versioning)
  - [Consumer Version Matching](#consumer-version-matching)
- [Updating Services](#updating-services)
  - [Release](#release)
  - [Deprecation](#deprecation)
  - [Rollbacks](#rollbacks)
- [Stateful Services](#stateful-services)
- [TODOs](#todos)

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

👉 Mapping of each version to its URI **MUST** be _immutable_, i.e. the URI for a specific version **MUST** always be the same. This only applies to entries that do exist in the mapping, as entries can be omitted or even re-included in the mapping at any time (signaling changes in availability of different versions).
The mapping **MUST** include only API versions that are available (i.e. the service is running). Versions not listed in the mapping are assumed to be unavailable.

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

### Stateful and Stateless Services

Adjectives _stateful_ and _stateless_ are used in this document to indicate possibility of having multiple instances of a service,
with potentially differing API versions, available. This mostly co-incides with the traditional definition of a stateless service. However, a service
which by the standard definition might be deemed stateful, might be considered _stateless_ under the definition provided here, and vice-versa.

- If a service can be available with multiple differing API versions, it is called _stateless_.
- If a service can NOT be available with multiple differing API versions, it is called _stateful_.

#### Example

A database is (most probably) a stateful service. The schema of the data IS the API of this service, and regardless of clustering, the data
can NOT have multiple schemas, which means varying API versions can NOT be available at the same time.

<br>

## Consumer

A _consumer_ is defined by a unique URI, on which the consumer will return a list of APIs it consumes and versions it requires.

👉 APIs might have _out-of-cluster consumers_, for example mobile or web apps, or external costumers consuming APIs. For such consumers, proxies should exist within the network and with a unique URI which responds as described above. For example, you might have a JSON file available on the network as a proxy for your web-app, which outlines APIs it needs and versions it needs, and update this JSON file automatically via some CI pipeline whenever the web-app is being updated.

👉 In this document, _consumer_ refers to the code utilizing an API if it is in-cluster, or its in-cluster proxy if it is out of the cluster. The term
_actual consumer_ is used in reference to the API consuming program, regardless of whether it is inside the cluster or not.

#### Example:

The proxy for the web-app, i.e. `https://web-app-proxy.my-cloud`, which returns the following JSON:

```json
{
  "https://auth.my-cloud": "2.1.9",
  "https://wallet.my-cloud": "1.0.5"
}
```

In this case, each instance of the web app (which is running on some browser) is an _actual consumer_, all proxied by one _consumer_ at
aforementioned URI.

<br>

## Cluster

The _cluster_ is the collection of all services and their consumers (or consumer proxies). Specifically, it consists of:

- A set of URIs for services
- A set of URIs for consumers (or consumer proxies)

#### Example:

```json
{
  "services": [
    "https://auth.my-cloud",
    "https://wallet.my-could",
    "https://monitoring.my-cloud"
  ],
  "consumers": [
    "https://web-app-proxy.my-cloud",
    "https://admin-panel.my-cloud",
    "https://android-proxy.my-cloud",
    "https://ios-proxy.my-cloud",
    "https://monitoring.my-cloud"
  ]
}
```

### Cluster Topology

The _topology_ of a cluster is a mapping of services and their consumers. The exact format is not constrained, but the most basic example
each service is identified by its main URI, and it is mapped to an array of URIs of consumers.

Knowing the cluster, the topology can be computed on demand (by querying all consumers for their dependencies). It can also be
accumulatively updated through deployments or kept up to date at regular intervals, etc.

#### Example:

```json
{
  "https://auth.my-cloud": [
    "https://web-app-proxy.my-cloud",
    "https://android-proxy.my-cloud",
    "https://ios-proxy.my-cloud",
    "https://wallet.my-could",
    "https://monitoring.my-cloud"
  ],
  "https://wallet.my-could": [
    "https://web-app-proxy.my-cloud",
    "https://android-proxy.my-cloud",
    "https://ios-proxy.my-cloud",
    "https://monitoring.my-cloud"
  ],
  "https://monitoring.my-cloud": [
    "https://admin-panel.my-cloud"
  ]
}
```

```js
export async function getClusterTopology() {
  const cluster = await getCluster()
  const topology = {}

  await Promise.all(cluster.consumers.map(
    consumer => getConsumer(consumer).then(dependencies => {
      Object.keys(dependencies).forEach(
        service => topology[service] = (topology[service] || []).concat(consumer)
      )
    })
  ))
  
  return topology
}
```

<br>

## Operation Safety

An operation (changes to available programs on the cluster) is called _SAFE_ if no new issues (undesired behaviors) are introduced
to the actual consumers after the operation, and is called _UNSAFE_ otherwise.

<br>

# Versioning

Versioning of service APIs **MUST** strictly follow [semantic versioning](https://semver.org), with MAJOR starting from `1`.
In particular:

1. Any actual consumer using version `A.B.C` of a service can also (without any change) use version `A.B.x` for all `x`
2. Any actual consumer using version `A.B.C` of a service can also (without any change) use version `A.x.y` for all `y` and all `x >= B`

Additionally, we will assume that no actual consumer using version `A.B.C` of a service can switch to `x.y.z` for `x != A` and any `y` and `z`. In other words,
a difference in MAJOR version number **SHOULD** correspond to a non-backwards-compatible change in the API.

Another **RECOMMENDED** criteria is having consumers specifying the earliest possible version of an API that they truly need. For example,
if an actual consumer can use version `A.B.C` of a service, its corresponding consumer **SHOULD NOT** specify version `A.D.E` as its dependency where
either `D > B` or `D == B && E > C`. Violating this criteria might result in wrongfully assuming some safe _deprecations_ unsafe (see below).

<br>

## Consumer Version Matching

Actual consumers **MUST** know which version of an API they expect to utilize. An expected version, `X.Y.Z`, is said to match a service version (or API version),
`A.B.C`, IF AND ONLY IF the following hold:

- `A == X`
- `B >= Y`

```js
export function match(target, expected) {
  const [tsplit, esplit] = [target.split('.'), expected.split('.')]
  return (
    tsplit[0] === esplit[0]                         // --> major should be the same
    && parseInt(tsplit[1]) >= parseInt(esplit[1])   // --> minor should be greater or equal
  )
}
```

A _matching available version_ of a service, for given expected version, is a version with an entry on the version registry who matches expected version as well.

```js
export function matchingAvailableVersions(expected, registry) {
  return Object.keys(registry).filter(version => match(version, expected))
}
```

The _latest matching available version_ of a service, for a given expected version, is a _matching available version_ for the expected version `X.Y.Z`, where for any other _matching available version_ `X.W.T`, either one of the following hold:

- `W < Y`
- `W == Y && T < Z`

```js
function greater(a, b) {
  const [asplit, bsplit] = [a.split('.'), b.split('.')]
  return (
    (parseInt(asplit[0]) > parseInt(bsplit[0]))
    || (parseInt(asplit[1]) > parseInt(bsplit[1]))
    || (parseInt(asplit[2]) > parseInt(bsplit[2]))
  )
}

export function latestMatchingAvailableVersion(expected, registry) {
  let res

  Object.keys(registry).forEach(version => {
    if (match(version, expected)) {
      if (!res || greater(version, res)) {
        res = version
      }
    }
  });

  return res
}
```

Each actual consumer **MUST** include its expected API version on each request. It is **RECOMMENDED** to be included in request header, and on a unique key (e.g. `Expect` or a custom header `Expected-Version`). In case the request is being made to the _main URI_ of the service, then:

👉 If the _latest version_ matches expected version, the service responds itself.

👉 If the _latest version_ DOES NOT match expected version, then:
- The service **MUST** fetch registry information from version registry (**MAY** be cached for increased performance).
- The service then **MUST** return a redirect to the requested endpoint on _latest matching available verison_.
- If there is no _matching available version_ in the registry, the service **MAY** attempt to respond to the request itself,
    or it **MAY** return a proper error (e.g. `417`)

In case the request is being made to the URI of a specific version, the service **MAY** attempt to redirect to _latest matching available version_
(in which case it **MUST** satisfy the same criteria outlined above), it **MAY** try to respond to the request itself, or it **MAY**
return a proper error. In case of returning error, it is **RECOMMENDED** to have a consistent behavior between all versions, which includes _latest version_.

Actual consumers **MUST** be able to handle redirects, and **MAY** cache them for increased performance.

<br>

# Updating Services

Since services **MUST** be immutable, any update to a service **MUST** correspond to a change in version, even if the API is exactly the same.
Subsequently, we assume updating a service and its API are the same thing (bug-fixes and internal minor changes **MUST** result in a new API version
with an increased PATCH number).

Updates are conducted with two distinct and atomic actions: _Release_ and _Deprecation_.

<br>

## Release

A _release_ makes a new version of a service available, without affecting availability of other versions. A release **MUST** result in a new entry in the version registry, mapping the version of the released API to its corresponding URI. A release **MAY** also change the availability and behavior of endpoints
on _main URI_ of the service, in which case it has effectively changed _latest version_ of the service to the version mirroring behavior on main URI.

👉 A _release_ (without any deprecation) is ALWAYS SAFE, since existing consumers won't get affected.

```js
export async function release(service, version, url) {
  const registry = await getRegistry(service)
  registry[version] = url
  await saveRegistry(service, registry)
}
```

<br>

## Deprecation

A _deprecation_ makes a previously released version unavailable. After deprecation, the deprecated version **MUST NOT** have an entry in URI mapping provided by version registry. Additionally, if the deprecated version was the latest version, the behavior of the API on main URI **MUST** also change to mirror that of another, still available version.

Lets call the version being deprecated the _target version_. Then:

👉 The deprecation is UNSAFE when by excluding target version from version registry, there would not be any _matching available version_ for some consumer.
This means the deprecation will result in the consumer invoking non-existing endpoints or existing endpoints with wrong parameters / response expectation.

👉 The deprecation _might be_ UNSAFE when target version is _latest matching available version_ for some consumer.
This means the deprecation might re-introduce some bugs that were to be fixed between target version and _latest matching available version_ for the consumer calculated by excluding target version from registry.

👉 The deprecation is SAFE when target version is NOT _latest matching available version_ for any consumer.

```js
export async function deprecate(service, version) {
  const registry = await getRegistry(service)
  delete registry[version]
  await saveRegistry(service, registry)
}
```
```js
export async function checkDeprecate(service, version) {
  const registry = await getRegistry(service)
  const lmav = latestMatchingAvailableVersion(version, registry)

  if (lmav !== version) {
    //
    // this is not the latest matching available version
    // for any consumer, safe deprecation
    //
    return 'SAFE'
  } else {
    delete registry[version]

    const topology = await getClusterTopology()
    const consumers = await Promise.all(
      topology[service].map(consumer => getConsumer(consumer))
    )

    let warn = false
    for (let consumer of consumers) {
      if (!latestMatchingAvailableVersion(consumer[service], registry)) {
        //
        // some consumer would be left without a matching
        // available version.
        //
        return 'UNSAFE'
      } else if (match(version, consumer[service])) {
        //
        // version was lmav for itself, which means in its own major it is
        // the latest matching. if it matches any consumer expectation, it means
        // it was lmav for the consumer as well, and deprecating it is a rollback with
        // potential for re-introducing bugs.
        //
        warn = true
      }
    }

    return warn ? 'WARN' : 'SAFE'
  }
}
```

<br>

## Rollbacks

A rollback is a deprecation where target version is _latest matching available version_ for some consumers. As such, it falls under one of the first two criteria outlined for deprecations (either some consumers will be left without any _matching available version_ after the deprecation or not), and as such safety of rollbacks can be assessed as outlined in the previous section.

<br>

# Stateful Services

Stateful services, by definition, could not be updated without a deprecation, as they can only have one available version at any given time. Updates to such services can be viewed as simultaenous release and deprecations, and subsequently safety of such updates can be assessed by analyzing safety of the deprecation, as described above.

This in particular means patches and minor updates are SAFE to perform on stateful services, rollbacks might be UNSAFE and major updates are definitely UNSAFE and will result in downtime until all dependent consumers are updated correspondingly. Subsequently it is highly **RECOMMENDED** to avoid rollbacks and major updates to stateful services. IF major updates are inevitable, it is highly **RECOMMENDED** to avoid having direct dependencies between stateful services and consumers that cannot be updated in a timely fashion.

<br>

# TODOs

This document requires further work in these areas:

- [ ] Figures, for better visualizing processes and concepts.
- [ ] Pre-release versions: as of now, present document provides no guidance on how to treat [pre-release versions](https://semver.org/#spec-item-9), in particular, whether they should be considered for calculating _matching available version_ of a service, and if so, how should they be considered for calculating _latest matching available version_s.
- [ ] Deprecation warnings: it might be useful to incorporate mechanisms for outlining potential timeline for deprecation of a version of an API, which might be useful for dev tools that can then warn maintainers of a consumer.
- [ ] Patch version fixes: right now consumers cannot require a certain patch version or upwards. Fixating on patch versions might get too restrictive,
which is not optimal for API versioning, but also in some cases consumers might need to ensure some fix is on the API version they use.
