---
layout: post
title: |
  Optimized building with Vaadin Flow 24.5
categories:
  - Blog
tags:
  - opensource
  - vaadin
---

As of Vaadin Flow 24.5 you can take quite a few steps to 
* speed up your build
* de-bloat your final artifacts and
* remove some unnecessary dependencies to lower the amount of potential conflicts and attack vectors

## What can be removed / disabled?

### Servlet-Check

* Impact: small
* Feasibility: easy

This module has one job: To detect if [an outdated servlet version](https://github.com/vaadin/servlet-detector) is used.

This is only relevant when deploying to very old appservers and also only once.

Solution:
* Exclude the ``com.vaadin.servletdetector:throw-if-servlet3`` dependency

### Material Theme

* Impact: small
* Feasibility: easy

Vaadin only uses the Lumo theme by default, however the Material theme is still included.

Solution:
* Exclude the ``com.vaadin:vaadin-material-theme`` dependency

### React support

* Impact: small
* Feasibility: easy

Vaadin 24.5 includes support for React by default.

If you don't use React you can do the following:
* Exclude the ``com.vaadin:flow-react`` dependency
* Configure the ``vaadin-maven-plugin``
  * set ``reactEnable`` to ``false`` to prevent it from checking if React is present during each build

### Collaboration Engine

* Impact: small
* Feasibility: easy

Vaadin 24.5 also ships the [Collaboration Engine](https://github.com/vaadin/collaboration-kit) by default.

It's extremely unlikely that you need it:
* Exclude the ``com.vaadin:collaboration-engine`` dependency

### Vaadin Hilla

* Impact: big
* Feasibility: easy

If you don't use [Hilla](https://vaadin.com/hilla) there is no need to include it.

The following optimizations can be applied:
* Exclude the ``com.vaadin:hilla`` and ``com.vaadin:hilla-dev`` dependencies
* Configure the ``vaadin-maven-plugin`` 
  * set ``frontendHotdeploy`` to ``false`` to prevent it from checking if Hilla is present during each build
  * [ADVANCED] the plugin currently always calls the ``configure`` goal which is only required for Hilla. To disable this you have to modify the plugin.

### Vaadin Copilot

* Impact: medium - only during development
* Feasibility: easy

Vaadin Copilot is a "UI builder" that is active during development and requires some additional setup to work properly.
You also require a license to use it.

It brings a lot of dependencies and causes some performance impacts while developing.

The following optimizations can be applied:
* Exclude the ``com.vaadin:copilot`` dependency
* Disable the [component tracker](https://github.com/vaadin/flow/issues/18130) in development mode: ``vaadin.devmode.componentTracker.enabled=false`` (inside the ``application.properties``)

### Vaadin Copilot - UI Test Generation

* Impact: small - only during development
* Feasibility: easy

The Vaadin UI Generation test extension for Copilot requires an OpenAI Api Key.

If you don't use it:
* Exclude the ``com.vaadin:ui-tests`` dependency

### ph-css

* Impact: small
* Feasibility: medium

[``ph-css``](https://github.com/phax/ph-css) is a Java CSS parser that is used in 2 places inside Vaadin code:
1. Vaadin Copilot (during development) - when editing styles
2. ``StyleAttributeHandler`` - when a ``style``-attribute is used instead of the ``Style``-class
   * This is an extreme niche and likely never happens

It brings a few dependencies with [confusing class names](https://github.com/vaadin/flow/issues/18719), e.g.:
* ``CommonsAssert.assertEquals``
* ``@VisibleForTesting``
* ``javax.*``

Solution:
* Exclude the ``com.helger:ph-css`` dependency
* Overwrite/Replace ``com.vaadin.flow.dom.impl.StyleAttributeHandler`` with your own implementation. Notably the ``setAttribute``-method:
  ```java
  @Override
	public void setAttribute(final Element element, final String attributeValue)
	{
		throw new UnsupportedOperationException("Use the Style class instead of setting the attribute manually");
	}
  ```
  Due to this there should be no class loading problems during runtime

### AOT - reflections

* Impact: small
* Feasibility: medium
* Only relevant for Spring projects

The [``reflections``](https://github.com/ronmamo/reflections) library is deprecated and brings ``javax.*``.

It is used during Vaadin's Spring Boot's Ahead of Time (AOT) processing-integration. 

If you don't use AOT, you can remove it:
* Exclude the ``org.reflections:reflections`` dependency
* Overwrite/Replace ``com.vaadin.flow.spring.springnative.VaadinBeanFactoryInitializationAotProcessor`` with your own implementation. Notably the ``processAheadOfTime``-method:
  ```java
  @Override
	public BeanFactoryInitializationAotContribution processAheadOfTime(
		final ConfigurableListableBeanFactory beanFactory)
	{
		throw new UnsupportedOperationException();
	}
  ```
  Due to this there should be no class loading problems during runtime

### Vaadin Open

* Impact: small - only during development
* Feasibility: medium

[Vaadin Open](https://github.com/vaadin/open) is used to open URLs. 
Getting a lot of tabs automatically opened can be annoying if you restart your application quite often.

It also brings additional dependencies.

You can remove it like this:
* Remove ``vaadin.launch-browser=true`` (default value is false; inside the ``application.properties``)
* Exclude the ``com.vaadin:open`` dependency
* Overwrite/Replace ``com.vaadin.open.Open`` with your own implementation:
  ```java
  public class Open
  {
    public static boolean open(String target)
    {
      return false;
    }
    
    private Open()
    {
    }
  }
  ```
  Due to this there should be no class loading problems during runtime

### [PWA](https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps)

* Impact: huge
* Feasibility: easy

[Vaadin's PWA](https://vaadin.com/docs/latest/flow/configuration/pwa) implementation is quite complex and not very flexible:
* The PWA icon needs to be a ``png``. It can't be a ``svg`` or something else with better compression/scaling
* During startup the PWA icon is resized for every IOS device that exists (blame Apple for this)
  * This takes some time/resources
  * This requires access to some Java AWT classes
* Also a manifest has to be generated

If you don't need fancy PWA stuff like the offline page you can disable this and use a more simple PWA solution:
* Create a manifest file, e.g. ``manifest.webmanifest`` with the following contents:
  ```json
  {
    "name": "Application ",
    "short_name": "App",
    "display": "standalone",
    "background_color": "#f2f2f2",
    "theme_color": "#ffffff",
    "start_url": ".",
    "icons": [
      {
        "src": "icons/icon.svg",
        "sizes": "48x48 72x72 96x96 128x128 256x256 any",
        "type": "image/svg+xml",
        "purpose": "any"
      }
    ]
  }
  ``` 
* Add the icon - in this case ``icons/icon.svg``
* Add the following to your ``AppShellConfigurator``'s ``configurePage``-method:
  ```java
  settings.addLink(
    "manifest.webmanifest",
    Map.of(
      "rel", "manifest",
      // use-credentials is required otherwise no auth is sent and the request is discarded
      "crossorigin", "use-credentials"));
  settings.addLink("icon", "icons/icon.svg");
  ```


### Vaadin Maven plugin

* Impact: huge - only during build
* Feasibility: hard

Nearly all of the above changes can also be done to the Vaadin Maven plugin, otherwise the dependencies are still downloaded when the plugin is executed.

To achieve the changes you likely have to fork/overwrite parts of the plugin.

Here is a list of additional recommended changes:

#### gwt-elemental

* Impact: small
* Feasibility: easy

The plugin contains ``com.google.gwt:gwt-elemental`` but this [is not used anywhere inside the plugin](https://github.com/vaadin/flow/issues/20359).

Solution:
* Exclude the ``com.google.gwt:gwt-elemental`` dependency

#### polymer2lit

* Impact: small
* Feasibility: easy

The plugin contains a ``convert-polymer``-goal.

This goal will likely never be needed as polymer is deprecated since Vaadin 14.
Nearly everyone should have migrated by now.

Solution:
* Exclude the ``com.vaadin:flow-polymer2lit`` dependency

#### SBOM

* Impact: small
* Feasibility: easy

SBOM generation requires the ``maven-invoker``.

If you don't require Vaadin's SBOM generation you can remove it:
* Exclude the ``org.apache.maven.shared:maven-invoker`` dependency

#### Eclipse support

* Impact: small
* Feasibility: easy

<!-- Might contain traces of Eclipse bashing  --->

Because Eclipse is clearly the most supreme and best IDE in the universe, one has to introduce dedicated support for it into a Maven plugin.

If you use the command line or a more inferior IDE like IntelliJ IDEA - which has only 85% market share - you can consider removing it:
* Exclude the ``org.codehaus.plexus:plexus-build-api`` dependency

#### Disable Version checks

* Impact: medium
* Feasibility: medium

During every build/prepare-frontend execution it is checked if the correct versions of Node/NPM are installed.

Usually you don't need to run this check as 
* Vaadin brings its own Node and NPM if none is installed
* It's otherwise sufficient to run the check once during a Vaadin update or initial setup

The check can be disabled by:
* Setting the system property ``vaadin.ignoreVersionChecks`` to ``true`` when the plugin is run

#### Use a faster ``ClassFinder``

* Impact: huge
* Feasibility: hard

When scanning for annotations, classes, etc. Vaadin uses a ``ClassFinder``.

However the default solution is not very efficient:
* A new ``ClassFinder`` is created for every scan - no caching can be used
* The default ``ClassFinder`` searches everywhere - Not just in the required projects

This can be improved by:
* Using a cached ``ClassFinder``
* Only scanning the relevant locations - e.g. ignoring all ``.jar``-files

### Telemetry

* Impact: small
* Feasibility: medium

Vaadin sends a few usage statistics during development.

They can cause a unwanted side effects, e.g. conflicts with the Content Security Policy in the browser or otherwise unwanted network traffic.

#### Server side

Server-side telemetry can be disabled using:
* ``vaadin.devmode.usageStatistics.enabled=false`` (inside the ``application.properties``)

#### Client side

Client side telemetry is sent directly by the browser and has to be disabled there.

The easiest way to disable it is using a custom ``AppShellConfigurator``:
* Initially check if dev-mode is active (e.g. by searching for dev-mode classes)
* If this is the case inject a script during ``configurePage``:
  ```java
  settings.addInlineWithContents(
    TargetElement.HEAD,
    Inline.Position.PREPEND,
    """
      if(!localStorage.getItem('vaadin.statistics.optout')) {
        localStorage.setItem('vaadin.statistics.optout', 1);
        console.log('Forced opt-out of Vaadin telemetry');
      }""",
    Inline.Wrapping.JAVASCRIPT);
  ```

### Production

The above mentioned optimizations should also be applied when building for production

### Other

Here are some other considerations that might help you:
* Don't restart your server for every little change. Use a hot swapping tool like Spring Boot DevTools and configure them correctly
* Analyse your projects dependencies regularly
* Attach a profiler and look at the results. What took considerable time and does it makes sense?
* Build your UIs/Views in a way that they survive server restarts - e.g. by storing the current state in the Browser's URL - so that you can resume where you stopped
