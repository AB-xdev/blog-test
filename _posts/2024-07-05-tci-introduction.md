---
title: |
  [Supercruising](https://en.wikipedia.org/wiki/Supercruise) your [Testcontainer](https://testcontainers.com/) Test with XDEV's TCI Framework
author: Blex
categories:
  - Blog
tags:
  - opensource
  - testcontainers
---

<!-- # [Supercruising](https://en.wikipedia.org/wiki/Supercruise) your [Testcontainer](https://testcontainers.com/) Test with XDEV's TCI Framework -->

XDEV is glad to announce the release of a new Open Source project:
[Testcontainers Infrastructure (TCI) Framework](https://github.com/xdev-software/tci-base)

This framework builds on top of [Testcontainers](https://testcontainers.com/) and provides the following key principles:

### Defined way / abstraction to writing test infrastructure

This way there is a predefined "way" how to use and write infrastructure - which allows for cleaner code that can easier be understood and written.

It also ensures that the infrastructure
* is modular
* is extensible and can easily be customized
* can be utilized for parallelization

The framework achieves this by encapsulating the Testcontainer which makes it possible to add additional non-container related code, like e.g. clients or common methods (e.g. ``createUser``) without needing to modify the container.

This follows the ["You should favor composition over inheritance"](https://blogs.oracle.com/javamagazine/post/java-inheritance-composition) design principle.

For each infrastructure there is also a factory to create new ones. This factory can be easily configured and handles such things as Container creation, PreStarting and tracking the created TCI for the additional features described below.

### Options to run the tests as fast as possible

Running test as fast as possible has multiple advantages:
* When run by a developer:<br/>Usually there is not much else you can do when running tests - except maybe getting some coffee. It's also possible to start another task but then you might lose focus on your original one and have to "rethink" back into the topic later.
* When run by a CI:
  * If you are paying for computing on demand (e.g. minute based billing for something like [Spot-Instances](https://aws.amazon.com/ec2/spot/)) running test faster (without enlarging the used machine) can save a lot of money due to lower rental times.
  * If you are paying for a fixed amount of computing running test faster means that there is more time available for other jobs to be executed on the CI. If the saved time is high enough you can also think about scaling down the required computing power.
  * Faster test feedback: When e.g. requiring full integration test success before doing a release this can cut the time for release shipment

The framework uses a "PreStarting mechanism" to achieve this:

#### What is PreStarting? 

When running tests usually there are certain times when the available resources are barely utilized:
![Idea-Showcase](https://raw.githubusercontent.com/xdev-software/tci-base/develop/assets/PreStartingCauseIdea.png)

PreStarting uses a "cached" pool of infrastructure and tries to utilizes these idle times to fill/replenish this pool.
So that when new infrastructure is requested there is no need to wait for the creation of it and use the already started infrastructure from this pool - if it's available.

#### Additional performance

When implemented correctly this can have a huge performance difference as can be seen in our [performance comparison](https://github.com/xdev-software/tci-base/blob/develop/PERFORMANCE.md).

There is also a live example (using GitHub Actions), which yields the following results:

| Case | Parallelization | PreStarting enabled? | Time to run all test |
| --- | --- | --- | --- |
| A | - | ❌ | 8m 50s |
| B | - | ✔ | 5m 30s |
| C | 2 | ❌ | 6m |
| D | 2 | ✔ | 4m 50s |

So as we can see in the best case (D) there is a near 50% speed improvement when compared to running the tests conventional manor (A).

### "Quality of Life"

Next to the already listed aspects above there are also some other minor enhancement included in the framework:
* All started containers have a unique human-readable name, which makes identification easier when tracing or debugging
* An optimized implementation of Testcontainers Network
* "Container Leak detection" - detects if container that have been started by a test are also terminated. Prevents your machine running out of resources
* A Tracing mechanism that makes finding bottlenecks and similar problems easier

## Further reading

If you would like to try out the framework have a look at our GitHub Repo and check out the ["Usage" section](https://github.com/xdev-software/tci-base?tab=readme-ov-file#usage). There are also multiple demos available.

If you have any further questions or need help feel free to [open an issue](https://github.com/xdev-software/tci-base/issues/new/choose) or contact [our support](https://xdev.software/en/services/support).

Also don't forget to [star 🌟 the repo](https://github.com/xdev-software/tci-base).

