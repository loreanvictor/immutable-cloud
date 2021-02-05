<div align="center">

<img src="immutable-cloud.svg" width="256"/>

# Immutable Cloud

(micro-)service architecture with resilience towards API changes

</div>

<br>

## Preface

The API of any (micro-)service corresponds to its distinct set of functionalities. As these functionalities (inevitably) change overtime, so will its API.
Such a change would need to be propagated to all dependent code (code that consumes the API), which can be a resource-consuming and particularly error-prone process, often resulting in downtimes and runtime errors in production.

A particular solution is to keep multiple _versions_ of the service (with different APIs) available, as to ensure availability during the time all consumer code transitions from a previous version to the next. Maintaining multiple versions of some API can be resource-consuming, and the whole process can be error-prone on its own. _Immutable Cloud_ is a proposed solution for this problem. It focuses on determining exactly which versions of any particular API needs to be maintained within a cluster to avoid errors while minimizing maintenance efforts.

