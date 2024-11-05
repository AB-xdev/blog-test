---
layout: post
title: |
  Supercruising your Testcontainer Test with XDEV's TCI Framework
categories:
  - Blog
tags:
  - opensource
  - testcontainers
---

<!-- # [Supercruising](https://en.wikipedia.org/wiki/Supercruise) your [Testcontainer](https://testcontainers.com/) Tests with XDEV's TCI Framework -->

We are proud to announce the release of our new Open Source project:
[Testcontainers Infrastructure (TCI) Framework](https://github.com/xdev-software/tci-base)

This framework builds on top of [Testcontainers](https://testcontainers.com/) and provides the following key features:

### 1. Improve customizability and parallelization

By using the factory pattern when creating container we make it easier for the developer to adjust the containers as needed.

  <table border=0>
  <tr>
  <td>
    <b>Without</b> TCI
  </td>
  <td>
    <b>With</b> TCI
  </td>
  </tr>
  <tr>
  <td markdown="1">
    
  ```java
  static final MySQLContainer MY_SQL_CONTAINER;

  static {
    MY_SQL_CONTAINER = new MySQLContainer();
    MY_SQL_CONTAINER.start();
  }
  ```

  </td>
  <td markdown="1">

  ```java
  static final DBTCIFactory DB_INFRA_FACTORY = 
    new DBTCIFactory();

  void startInfra() {
    this.dbInfra = DB_INFRA_FACTORY.getNew(...);
  }
  ```

  </td>
  </tr>
  </table>

For each "infrastructure" (abbreviated TCI) there is also a factory to create new ones. This factory can be easily configured and handles such things as Container creation, PreStarting and tracking the created TCI for the additional features described below.

Customizing containers/infrastructure looks like this in the TCI_FACTORY-Class
  ```java
  @Override
  public void start(final String containerName) {
    super.start(containerName);
    if(doMigrate) {
      this.migrateDatabase(BASELINE_FOR_TESTS);
    }
  }

  void migrateDatabase(String version) {
    // Migrate database with e.g. Flyway
  }
  ```

This means it is now possible to add additional non-container related code, like e.g. clients or common methods (e.g. ``createUser``) without needing to modify the container itself. 
This follows the [composition over inheritance](https://blogs.oracle.com/javamagazine/post/java-inheritance-composition) design principle.

By using factories the framework can also improve performance through handling parallelization and PreStartup.

### 2. Running tests as fast as possible

#### Why is this important in the first place?
Running test as fast as possible has multiple advantages:
* When run by a developer:<br/>Usually there is not much else you can do when running tests - except maybe getting some coffee. It's also possible to start another task but then you might lose focus on your original one and have to "rethink" back into the topic later.
* When run by a CI:
  * If you are paying for computing on demand (e.g. minute based billing for something like [Spot-Instances](https://aws.amazon.com/ec2/spot/)) running test faster (without enlarging the used machine) can save a lot of money due to lower rental times.
  * If you are paying for a fixed amount of computing running test faster means that there is more time available for other jobs to be executed on the CI. If the amount of saved time is high enough you can also think about scaling down the required computing power.
  * Faster test feedback: When e.g. requiring full integration test success before doing a release this can cut the time for shipping the release

The framework is explicitly designed for **parallelization** and provides multiple features to speed up tests:

#### 2.1. PreStarting mechanism

When running tests usually there are certain times when the available resources are barely utilized:
![Idea-Showcase](https://raw.githubusercontent.com/xdev-software/tci-base/develop/assets/PreStartingCauseIdea.png)

PreStarting uses a "cached" pool of infrastructure and tries to utilizes these idle times to fill/replenish this pool.
So that when new infrastructure is requested there is no need to wait for the creation of it and the already started infrastructure from this pool can be used - if it's available.

##### Additional performance

When implemented correctly this can have a huge performance difference as can be seen in our [performance comparison](https://github.com/xdev-software/tci-base/blob/develop/PERFORMANCE.md).

There is also a live example (using GitHub Actions), which yields the following results:

| Case | Parallelization | PreStarting enabled? | Time to run all test |
| --- | --- | --- | --- |
| A | - | ❌ | 8m 50s |
| B | - | ✔ | 5m 30s |
| C | 2 | ❌ | 6m |
| D | 2 | ✔ | 4m 50s |

So as we can see in the best case (D) there is a near 50% speed improvement when compared to running the tests conventional manor (A).

#### 2.2. Optimized Testcontainers Network
An optimized implementation of Testcontainers Network is used:
  <table>
  <tr>
  <td>
  Before
  </td>
  <td>
  After
  </td>
  </tr>
  <tr>
  <td markdown="1">
  
  ``NetworkImpl`` code from ``testcontainers 1.20``
  ```java
  @Override
  public synchronized String getId() {
      if (initialized.compareAndSet(false, true)) {
          boolean success = false;
          try {
              // Network is created when id is accessed
              // Takes a moment
              id = create();
              success = true;
          } finally {
              ...
          }
      }
      return id;
  }
  ```

  </td>
  <td markdown="1">
  
  ``LazyNetworkPool`` provides a pool of networks that are created in the background.<br/>
  No time is lost waiting for network creation.

  </td>
  </tr>
  </table>

  
#### 2.3. Container Leak detection
This detects if container that have been started by a test are also terminated. Prevents the test machine from running out of resources.<br/>
In the following example the Testcontainer is created, but never terminated.
  ```java
  @Test
  void test() {
    DummyTCI tci = DUMMY_FACTORY.getNew(...);
    ...
  }
  ```
  After running these tests with the framework the following error will show up in the logs:
  ```
  ERROR s.x.tci.leakdetection.TCILeakAgent - !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  ERROR s.x.tci.leakdetection.TCILeakAgent - ! PANIC: DETECTED CONTAINER INFRASTRUCTURE LEAK !
  ERROR s.x.tci.leakdetection.TCILeakAgent - !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  ERROR s.x.tci.leakdetection.TCILeakAgent - All test are finished but some infrastructure is still marked as in use:
  DummyTCIFactory leaked 1x [container-ids=[c1b6be852fac3bf65ac8f2739ab161d7f95bc4c62699c698ccc8b74da1be8a3d]]
  ```


### 3. Quality of Life

The framework also provides some minor enhancements:

#### 3.1. Human-readable names for containers
All started containers have a unique human-readable name, which makes identification easier when tracing or debugging
  <table>
  <tr>
  <td>
  Before
  </td>
  <td>
  After
  </td>
  </tr>
  <tr>
  <td markdown="1">
  
  ```
  docker stats

  NAME
  eager_rubin
  vigilant_archimedes
  practical_haibt
  ecstatic_sanderson
  serene_einstein
  great_saha
  agitated_dhawan
  strange_montalcini
  ```

  </td>
  <td markdown="1">
  
  ```
  docker stats

  NAME
  selenium-chrome-2-PS-1-...
  selenium-firefox-1-PS-1-...
  selenium-chrome-1-PS-1-...
  db-mariadb-1-PS-1-...
  oidc-2-PS-1-...
  selenium-firefox-2-PS-1-...
  recorder-selenium-chrome-1-PS-1-...
  recorder-selenium-firefox-1-PS-1-...
  ```

  </td>
  </tr>
  </table>

#### 3.2. Test run time statistics
A tracing mechanism that makes finding bottlenecks and similar problems easier<br/>
  _Example:_
  ```
  [main] [i.tracing.TCITracingAgent] === Test Tracing Info ===
  Duration: 2m 43.608s
  Tests: 20.656s / 15 / 5m 9.84s
  BrowserTCIFactory-firefox:
    bootNew - 1ms / 6 / 5ms
    connectToNetwork - 515ms / 5 / 2.575s
    getNew - 574ms / 5 / 2.87s
    infraStart(async) - 14.575s / 6 / 1m 27.448s
    postProcessNew - 54ms / 5 / 270ms
    warmUp - 2.448s / 1 / 2.448s
  ...
  ```

## Further reading

If you would like to try out the framework have a look at our GitHub Repo and check out the ["Usage" section](https://github.com/xdev-software/tci-base?tab=readme-ov-file#usage). There are also multiple demos available.

If you have any further questions or need help feel free to [open an issue](https://github.com/xdev-software/tci-base/issues/new/choose) or contact [our support](https://xdev.software/en/services/support).

Also don't forget to [star 🌟 the repo](https://github.com/xdev-software/tci-base).

