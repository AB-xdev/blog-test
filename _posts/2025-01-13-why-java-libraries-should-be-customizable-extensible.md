---
layout: post
title: |
  Why (the code of) libraries should be customizable/extensible
categories:
  - Blog
tags:
  - libraries
---

I now chose to write a post about this topic
* because it has already cost a lot of time and money[^1]
* so that I can reference it the next time when it comes up again

[^1]: We are not talking about small amounts of money here - Having a team sit around and wait just for a specific update to be released costs a lot!

So after getting asked - like the 50th time - during java library development:
* Why are you using Interfaces, Builders, Factories?
* Why is all of your code using ``protected`` and not ``private``?
* Why are you only using ``final`` for ``static`` classes?
* Why are there so many configuration options?

The short answer: Too keep the library customizable/extensible!

![](../../../../assets/blog/why-java-libraries-should-be-customizable-extensible/Patrick_Wallet_Library.jpg)

### Usual arguments against this approach

A lot of people I encountered and talked about this said something like: 
* This will never be used in that way!
   * Well someone (me) is using it _that way_ so...
* You are using it wrong! Don't do that!
   * I'm using it the way I need to.<br/>If you can provide a better solution that works for me than please do so.

Also what is the problem with e.g. keeping code ``protected`` or removing the ``final`` from a method?
There are 0 drawbacks for the library and only benefits for the implementer.

#### The encapsulation topic

I also often heard that this is against the encapsulation principle.

The key fact of encapsulation is that the "implementation details" are hidden.[^2]

[^2]: See also: [Wikipedia](https://en.wikipedia.org/wiki/Encapsulation_(computer_programming)), [geeksforgeeks.org](https://www.geeksforgeeks.org/encapsulation-in-java)

This can easily be achieved by using something like a Factory/Builder-pattern or by simply using ``protected``.<br/>
That way the implementation is still "hidden" - as only public methods can be accessed from the outside - but it can still be modified by the implementer using overrides.


#### Illustration using a code example

Consider the following code provided by a library:
```java
final class DatabaseContainer {
  private static final int SQL_PORT = 1234;

  public void start() {
    // launch container and wait until isStarted is true
  }

  private boolean isStarted() {
    return container.isStarted();
  }
}
```

Now you need to modify the ``isStarted`` method to additionally check if the port is reachable.

This is currently not possible because:
```java
final class DatabaseContainer { // ⚡ class is final → can't be extended
  private static final int SQL_PORT = 1234; // ⚡ private → can't be accessed in derived class 

  private boolean isStarted() // ⚡ private → can't be extended
}
```

To implement the check the following must be done:
```java
// -- Library --
public class DatabaseContainer {
  protected static final int SQL_PORT = 1234;

  public void start() {
    // launch container and wait until isStarted is true
  }

  protected boolean isStarted() {
    return container.isStarted();
  }
}

// -- Implementing application --
class DatabaseContainerWithPortCheck extends DatabaseContainer {
  @Override
  protected boolean isStarted() {
    return super.isStarted() && canPortBeReached(SQL_PORT);
  }
}

class MyApp {
  public static void run(String[] args) {
    new DatabaseContainerWithPortCheck().start(); 
    // Encapsulation preserved:
    // isStarted can't be accessed since it's not public
  }
}
```


#### Further examples

<details><summary>To further emphasize my point here's an excerpt of problems that I encountered with existing libraries in the past (click to expand)</summary>

<table>
  <tr>
    <th>Reference</th>
    <th>Problems</th>
    <th>Outcome <sup>Explanation below</sup></th>
  </tr>
  <tr>
    <td><a href="https://github.com/xdev-software/testcontainers-selenium">TestContainers Selenium</a></td>
    <td><a href="https://github.com/testcontainers/testcontainers-java/blob/2707f3143d3cfa8351f727bfd5752c1155818bd6/modules/selenium/src/main/java/org/testcontainers/containers/BrowserWebDriverContainer.java">Fields and methods are private</a><br/>Video recorder implementation can't be replaced<br/>Can't access constants for Ports, Passwords, etc.</td>
    <td>Hard-fork</td>
  </tr>
  <tr>
    <td><a href="https://github.com/xdev-software/testcontainers-advanced-imagebuilder">TestContainers Image-Builder</a></td>
    <td><a href="https://github.com/testcontainers/testcontainers-java/blob/2707f3143d3cfa8351f727bfd5752c1155818bd6/core/src/main/java/org/testcontainers/images/builder/ImageFromDockerfile.java">Fields and methods are private</a></td>
    <td>Hard-fork</td>
  </tr>
  <tr>
    <td><a href="https://github.com/xdev-software/spring-security-advanced-authentication-ui">spring-security-advanced-authentication-ui</a></td>
    <td>Field and methods are private.<br/>Filter can't be properly replaced in framework.<br/>Forced use of reflection to read/copy "internal" framework data.</td>
    <td>Overlay</td>
  </tr>
  <tr>
    <td><a href="https://github.com/spring-projects/spring-security/issues/14898">Spring Boot OidcUserService</a></td>
    <td>Method couldn't be overridden.<br/>After update method was customizable, however <a href="https://github.com/spring-projects/spring-security/blob/b63e8f50a5e90a47b5dac28d2c2d952d8de11973/oauth2/oauth2-client/src/main/java/org/springframework/security/oauth2/client/oidc/userinfo/OidcUserService.java#L148-L177">original code is still private and can't be reused</a></td>
    <td>Waited for update/<br/>Copied code</td>
  </tr>
  <tr>
    <td><a href="https://github.com/quarkusio/quarkus/issues/42990">Quarkus - OIDC Back-channel Logout</a></td>
    <td>Serious malfunction, rendering functionality completely unusable.<br/>Everything is private and not overwriteable. Only possible way to patch it are bytecode modifications or waiting for update.</td>
    <td>PR/Fix submitted to Quarkus.<br/>Implementing project delayed for ~1 week ($) while waiting for update and then performing it.</td>
  </tr>
  <tr>
    <td><a href="https://github.com/prometheus/client_java/issues/1173">prometheus-metrics Protobuf is not optional</a></td>
    <td>Bloated library with obsolete/experimental protocol.<br/>Used protobuf version was flagged by vulnerability scanner.<br/><a href="https://github.com/prometheus/client_java/blob/v1.3.1/prometheus-metrics-exposition-formats/src/main/java/io/prometheus/metrics/expositionformats/ExpositionFormats.java">Not designed in a customizable way</a></td>
    <td>Overlay/Update provided</td>
  </tr>
  <tr>
    <td><a href="https://hibernate.atlassian.net/browse/HHH-18873">Hibernate Annotation Processor - Entity Indexing</a></td>
    <td><a href="https://hibernate.atlassian.net/browse/HHH-18162?focusedCommentId=117262">No mention anywhere in changelogs.<br/>No option/flag to disable this; active by default.</a><a href="https://hibernate.atlassian.net/browse/HHH-18162?focusedCommentId=117274">Functionality is useless for some cases</a> and <a href="https://hibernate.atlassian.net/browse/HHH-18863">performance problems</a>.<br/>No way to overwrite the code since fields and methods are private.</td>
    <td>Update provides option to disable</td>
  </tr>
  <tr>
    <td><a href="https://github.com/xdev-software/flyway-core-slim">Flyway-Core Slim</a></td>
    <td><a href="https://github.com/flyway/flyway/blob/ba8b11c0272c744786e52049b0391710253ea7d2/flyway-core/src/main/java/org/flywaydb/core/internal/plugin/PluginRegister.java#L85-L104">Private methods</a> don't allow filtering out <a href="(https://github.com/flyway/flyway/issues/3893">unused things</a>.</td>
    <td>Overlay</td>
  </tr>
</table>

Common outcomes explained:
<ul>
  <li><i>Hard-fork</i><br/>Code was copied over and modified - may break when a new version is released.</li>
  <li><i>Overlay</i><br/>Code runs "on top" of existing code. May also replace existing code.</li>
</ul>

</details>

### Caveats

Implementer needs to be aware that - when the library is updated - their overwritten code may also require changes.<br/>
A good solution to detect possible incompatibilities are automated tests.

<!-- Section for footnotes -->
---
