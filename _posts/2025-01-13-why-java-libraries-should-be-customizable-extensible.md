---
layout: post
title: |
  Why (Java) libraries code should be customizable/extensible
categories:
  - Blog
tags:
  - libraries
---

So after getting asked - like the 50th time - during java library development:
* Why are you using interfaces, Builders, Factories?
* Why is all of your code using ``protected`` and not ``private``?
* Why are you only using ``final`` for ``static`` classes?

I now chose to write a post so that I can simply reference it next time the same question comes up.

The short answer: Too keep the library customizable/extensible!

![](../../../../assets/blog/why-java-libraries-should-be-customizable-extensible/Patrick_Wallet_Library.jpg)

### Encapsulation

But now you are yourself likely asking: _Why? Isn't this against encapsulation?_

Let's do a quick web-search what "encapsulation" means:
* Essentially, encapsulation prevents external code from being concerned with the internal workings of an object. <sup><a href="https://en.wikipedia.org/wiki/Encapsulation_(computer_programming)">Source</a></sup>
* It allows implementation details to be hidden while exposing a public interface for interaction. <sup><a href="https://www.geeksforgeeks.org/encapsulation-in-java/">Source</a></sup>

The key facts here are that the "implementation details" are hidden.

This can easily be achieved by using something like a Factory/Builder-pattern or by simply using ``protected``.<br/>
That way the implementation is still "hidden" but it can be easily modified by the developer who uses the library, if required.

### Examples

To further emphasize my point here's an excerpt of problems that I encountered with existing libraries in the past:

| Reference | Problems | Outcome<sup>1</sup> |
| --- | --- | --- |
| [TestContainers Selenium](https://github.com/xdev-software/testcontainers-selenium) | [Fields and methods are private](https://github.com/testcontainers/testcontainers-java/blob/2707f3143d3cfa8351f727bfd5752c1155818bd6/modules/selenium/src/main/java/org/testcontainers/containers/BrowserWebDriverContainer.java) <br/>Video recorder implementation can't be replaced<br/>Can't access constants for Ports, Passwords, etc. | Hard-fork |
| [TestContainers Image-Builder](https://github.com/xdev-software/testcontainers-advanced-imagebuilder) | [Fields and methods are private](https://github.com/testcontainers/testcontainers-java/blob/2707f3143d3cfa8351f727bfd5752c1155818bd6/core/src/main/java/org/testcontainers/images/builder/ImageFromDockerfile.java) | Hard-fork |
| [spring-security-advanced-authentication-ui](https://github.com/xdev-software/spring-security-advanced-authentication-ui) | Field and methods are private.<br/>Filter can't be properly replaced in framework.<br/>Forced use of reflection to read/copy "internal" framework data. | Overlay
| [Spring Boot OidcUserService](https://github.com/spring-projects/spring-security/issues/14898) | Method couldn't be overridden.<br/>After update method was customizable, however [original code is still private and can't be reused](https://github.com/spring-projects/spring-security/blob/b63e8f50a5e90a47b5dac28d2c2d952d8de11973/oauth2/oauth2-client/src/main/java/org/springframework/security/oauth2/client/oidc/userinfo/OidcUserService.java#L148-L177).<br/> | Waited for update/<br/>Copied code |
| [Quarkus - OIDC Back-channel Logout](https://github.com/quarkusio/quarkus/issues/42990) | Serious malfunction, rendering functionality completely unusable.<br/>Everything is private and not overwriteable. Only possible way to patch it are bytecode modifications or waiting for update. | PR/Code supplied.<br/>Implementing project delayed for ~1 week ($), while waiting for update and then performing it. |
| [prometheus-metrics Protobuf is not optional](https://github.com/prometheus/client_java/issues/1173) | Bloated library with obsolete/experimental protocol.<br/>Used protobuf version was flagged by vulnerability scanner.<br/>[Not designed in a customizable way](https://github.com/prometheus/client_java/blob/v1.3.1/prometheus-metrics-exposition-formats/src/main/java/io/prometheus/metrics/expositionformats/ExpositionFormats.java). | Overlay/Update provided |
| [Flyway-Core Slim](https://github.com/xdev-software/flyway-core-slim) | [Private methods](https://github.com/flyway/flyway/blob/ba8b11c0272c744786e52049b0391710253ea7d2/flyway-core/src/main/java/org/flywaydb/core/internal/plugin/PluginRegister.java#L85-L104) don't allow for filtering out [unused things](https://github.com/flyway/flyway/issues/3893). | Overlay |

<sup>1</sup> = Common outcomes explained:
* _Hard-fork_: Code was copied over and modified - may break when a new version is released
* _Overlay_: Code runs "on top" of existing code. May also replace existing code.