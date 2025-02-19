---
title: Custom Code Quality Rules
seo-title: Custom Code Quality Rules
description: Follow this page to learn about the custom code quality rules executed by Cloud Manager.
seo-description: Follow this page to learn about the custom code quality rules executed by Adobe Experience Manager Cloud Manager.
uuid: a7feb465-1982-46be-9e57-e67b59849579
contentOwner: jsyal
products: SG_EXPERIENCEMANAGER/CLOUDMANAGER
topic-tags: using
discoiquuid: d2338c74-3278-49e6-a186-6ef62362509f
feature: Code Quality Rules
exl-id: 7d118225-5826-434e-8869-01ee186e0754
---
# Custom Code Quality Rules {#custom-code-quality-rules}

>[!NOTE]
>To learn about Custom Code Quality Rules for Cloud Manager in AEM as a Cloud Service, see [here](https://experienceleague.adobe.com/docs/experience-manager-cloud-service/implementing/using-cloud-manager/test-results/custom-code-quality-rules.html?lang=en#using-cloud-manager).

This page describes the custom code quality rules executed by Cloud Manager created based on best practices from AEM Engineering.

>[!NOTE]
>The code samples provided here are only for illustrative purposes. See [Concepts](https://docs.sonarqube.org/7.4/user-guide/concepts/) to learn about SonarQube concepts and quality rules.

## SonarQube Rules {#sonarqube-rules}

The following section highlights the SonarQube Rules:

### Do not use potentially dangerous functions {#do-not-use-potentially-dangerous-functions}

**Key**: CQRules:CWE-676

**Type**: Vulnerability

**Severity**: Major

**Since**: Version 2018.4.0

The methods ***Thread.stop()*** and ***Thread.interrupt()*** can produce hard-to-reproduce issues and, in some cases, security vulnerabilities. Their usage should be tightly monitored and validated. In general, message passing is a safer way to accomplish similar goals.

#### Non-Compliant Code {#non-compliant-code}

```java
public class DontDoThis implements Runnable {
  private Thread thread;
 
  public void start() {
    thread = new Thread(this);
    thread.start();
  }
 
  public void stop() {
    thread.stop();  // UNSAFE!
  }
 
  public void run() {
    while (true) {
        somethingWhichTakesAWhileToDo();
    }
  }
}
```

#### Compliant Code {#compliant-code}

```java
public class DoThis implements Runnable {
  private Thread thread;
  private boolean keepGoing = true;
 
  public void start() {
    thread = new Thread(this);
    thread.start();
  }
 
  public void stop() {
    keepGoing = false;
  }
 
  public void run() {
    while (this.keepGoing) {
        somethingWhichTakesAWhileToDo();
    }
  }
}
```

### Do not use format strings which may be externally controlled {#do-not-use-format-strings-which-may-be-externally-controlled}

**Key**: CQRules:CWE-134

**Type**: Vulnerability

**Severity**: Major

**Since**: Version 2018.4.0

Using a format string from an external source (such a request parameter or user-generated content) can produce can expose an application to denial of service attacks. There are circumstances where a format string may be externally controlled, but is only allowed from trusted sources.

#### Non-Compliant Code {#non-compliant-code-1}

```java
protected void doPost(SlingHttpServletRequest request, SlingHttpServletResponse response) {
  String messageFormat = request.getParameter("messageFormat");
  request.getResource().getValueMap().put("some property", String.format(messageFormat, "some text"));
  response.sendStatus(HttpServletResponse.SC_OK);
}
```

### HTTP requests should always have socket and connect timeouts {#http-requests-should-always-have-socket-and-connect-timeouts}

**Key**: CQRules:ConnectionTimeoutMechanism

**Type**: Bug

**Severity**: Critical

**Since**: Version 2018.6.0

When executing HTTP requests from inside an AEM application, it is critical to ensure that proper timeouts are configured in order to avoid unnecessary thread consumption. Unfortunately, the default behavior of both Java's default HTTP Client (java.net.HttpUrlConnection) and the commonly used Apache HTTP Components client is to never timeout, so timeouts must be explicitly set. Further, as a best practice, these timeouts should be no more than 60 seconds.

#### Non-Compliant Code {#non-compliant-code-2}

```java
@Reference
private HttpClientBuilderFactory httpClientBuilderFactory;
 
public void dontDoThis() {
  HttpClientBuilder builder = httpClientBuilderFactory.newBuilder();
  HttpClient httpClient = builder.build();

  // do something with the client
}

public void dontDoThisEither() {
  URL url = new URL("http://www.google.com");
  URLConnection urlConnection = url.openConnection();
 
  BufferedReader in = new BufferedReader(new InputStreamReader(
    urlConnection.getInputStream()));
 
  String inputLine;
  while ((inputLine = in.readLine()) != null) {
    logger.info(inputLine);
  }
 
  in.close();
}
```

#### Compliant Code {#compliant-code-1}

```java
@Reference
private HttpClientBuilderFactory httpClientBuilderFactory;
 
public void doThis() {
  HttpClientBuilder builder = httpClientBuilderFactory.newBuilder();
  RequestConfig requestConfig = RequestConfig.custom()
    .setConnectTimeout(5000)
    .setSocketTimeout(5000)
    .build();
  builder.setDefaultRequestConfig(requestConfig);
 
  HttpClient httpClient = builder.build();
   
  // do something with the client
}

public void orDoThis() {
  URL url = new URL("http://www.google.com");
  URLConnection urlConnection = url.openConnection();
  urlConnection.setConnectTimeout(5000);
  urlConnection.setReadTimeout(5000);
 
  BufferedReader in = new BufferedReader(new InputStreamReader(
    urlConnection.getInputStream()));
 
  String inputLine;
  while ((inputLine = in.readLine()) != null) {
    logger.info(inputLine);
  }
 
  in.close();
}
```

### Product APIs annotated with @ProviderType should not be implemented or extended by customers {#product-apis-annotated-with-providertype-should-not-be-implemented-or-extended-by-customers}

**Key**: CQBP-84, CQBP-84-dependencies

**Type**: Bug

**Severity**: Critical

**Since**: Version 2018.7.0

The AEM API contains Java interfaces and classes which are only meant to be used, but not implemented, by custom code. For example, the interface *com.day.cq.wcm.api.Page* is designed to be implemented by ***AEM only***.

When new methods are added to these interfaces, those additional methods do not impact existing code which uses these interfaces and, as a result, the addition of new methods to these interfaces are considered to be backwards-compatible. However, if custom code ***implements*** one of these interfaces, that custom code has introduced a backwards-compatibility risk for the customer.

Interfaces (and classes) which are only intended to be implemented by AEM are annotated with *org.osgi.annotation.versioning.ProviderType* (or, in some cases, a similar legacy annotation *aQute.bnd.annotation.ProviderType*). This rule identifies the cases where such an interface is implemented (or a class is extended) by custom code.

#### Non-Compliant Code {#non-compliant-code-3}

```java
import com.day.cq.wcm.api.Page;

public class DontDoThis implements Page {
// implementation here
}
```

### ResourceResolver objects should always be closed {#resourceresolver-objects-should-always-be-closed}

**Key**: CQRules:CQBP-72

**Type**: Code Smell

**Severity**: Major

**Since**: Version 2018.4.0

ResourceResolver objects obtained from the ResourceResolverFactory consume system resources. Although there are measures in place to reclaim these resources when a ResourceResolver is no longer in use, it is more efficient to explicitly close any opened ResourceResolver objects by calling the close() method.

One relatively common misconception is that ResourceResolver objects created using an existing JCR Session should not be explicitly closed or that doing so will close the underlying JCR Session. This is not the case - regardless of how a ResourceResolver is opened, it should be closed when no longer used. Since ResourceResolver implements the Closeable interface, it is also possible to use the try-with-resources syntax instead of explicitly invoking close().

#### Non-Compliant Code {#non-compliant-code-4}

```java
public void dontDoThis(Session session) throws Exception {
  ResourceResolver resolver = factory.getResourceResolver(Collections.singletonMap("user.jcr.session", (Object)session));
  // do some stuff with the resolver
}
```

#### Compliant Code {#compliant-code-2}

```java
public void doThis(Session session) throws Exception {
  ResourceResolver resolver = null;
  try {
    resolver = factory.getResourceResolver(Collections.singletonMap("user.jcr.session", (Object)session));
    // do something with the resolver
  } finally {
    if (resolver != null) {
      resolver.close();
    }
  }
}

public void orDoThis(Session session) throws Exception {
  try (ResourceResolver resolver = factory.getResourceResolver(Collections.singletonMap("user.jcr.session", (Object) session))){
    // do something with the resolver
  }
}
```

### Do not use Sling servlet paths to register servlets {#do-not-use-sling-servlet-paths-to-register-servlets}

**Key**: CQRules:CQBP-75

**Type**: Code Smell

**Severity**: Major

**Since**: Version 2018.4.0

As described in the [Sling documentation](http://sling.apache.org/documentation/the-sling-engine/servlets.html), bindings servlets by paths is discouraged. Path-bound servlets cannot use standard JCR access controls and, as a result, require additional security rigor. Rather than using path-bound servlets, it is recommended to create nodes in the repository and register servlets by resource type.

#### Non-Compliant Code {#non-compliant-code-5}

```java
@Component(property = {
  "sling.servlet.paths=/apps/myco/endpoint"
})
public class DontDoThis extends SlingAllMethodsServlet {
 // implementation
}
```

### Caught Exceptions should be logged or thrown, but not both {#caught-exceptions-should-be-logged-or-thrown-but-not-both}

**Key**: CQRules:CQBP-44---CatchAndEitherLogOrThrow

**Type**: Code Smell

**Severity**: Minor

**Since**: Version 2018.4.0

In general, an exception should be logged exactly one time. Logging exceptions multiple times can cause confusion as it is unclear how many times an exception occurred. The most common pattern which leads to this is logging and throwing a caught exception.

#### Non-Compliant Code {#non-compliant-code-6}

```java
public void dontDoThis() throws Exception {
  try {
    someOperation();
  } catch (Exception e) {
    logger.error("something went wrong", e);
    throw e;
  }
}
```

#### Compliant Code {#compliant-code-3}

```java
public void doThis() {
  try {
    someOperation();
  } catch (Exception e) {
    logger.error("something went wrong", e);
  }
}

public void orDoThis() throws MyCustomException {
  try {
    someOperation();
  } catch (Exception e) {
    throw new MyCustomException(e);
  }
}
```

### Avoid having a log statement immediately followed by a throw statement {#avoid-having-a-log-statement-immediately-followed-by-a-throw-statement}

**Key**: CQRules:CQBP-44---ConsecutivelyLogAndThrow

**Type**: Code Smell

**Severity**: Minor

**Since**: Version 2018.4.0

Another common pattern to avoid is to log a message and then immediately throw an exception. This generally indicates that the exception message will end up duplicated in log files.

#### Non-Compliant Code {#non-compliant-code-7}

```java
public void dontDoThis() throws Exception {
  logger.error("something went wrong");
  throw new RuntimeException("something went wrong");
}
```

#### Compliant Code {#compliant-code-4}

```java
public void doThis() throws Exception {
  throw new RuntimeException("something went wrong");
}
```

### Avoid logging at INFO when handling GET or HEAD requests {#avoid-logging-at-info-when-handling-get-or-head-requests}

**Key**: CQRules:CQBP-44---LogInfoInGetOrHeadRequests

**Type**: Code Smell

**Severity**: Minor

In general, the INFO log level should be used to demarcate important actions and, by default, AEM is configured to log at the INFO level or above. GET and HEAD methods should only ever be read-only operations and thus do not constitute important actions. Logging at the INFO level in response to GET or HEAD requests is likely to create significant log noise thereby making it harder to identify useful information in log files. Logging when handling GET or HEAD requests should be either at the WARN or ERROR levels when something has gone wrong or at the DEBUG or TRACE levels if deeper troubleshooting information would be helpful.

>[!CAUTION]
>
>This does not apply to access.log-type logging for each requests.

#### Non-Compliant Code {#non-compliant-code-8}

```java
public void doGet() throws Exception {
  logger.info("handling a request from the user");
}
```

#### Compliant Code {#compliant-code-5}

```java
public void doGet() throws Exception {
  logger.debug("handling a request from the user.");
}
```

### Do not use Exception.getMessage() as the first parameter of a logging statement {#do-not-use-exception-getmessage-as-the-first-parameter-of-a-logging-statement}

**Key**: CQRules:CQBP-44---ExceptionGetMessageIsFirstLogParam

**Type**: Code Smell

**Severity**: Minor

**Since**: Version 2018.4.0

As a best practice, log messages should provide contextual information about where in the application an exception has occurred. While context can also be determined through the use of stack traces, in general the log message is going to be easier to read and understand. As a result, when logging an exception, it is a bad practice to use the exception's message as the log message - the exception message will contain what went wrong whereas the log message should be used to tell a log reader what the application was doing when the exception happened. The exception message will still be logged; by specifying your own message the logs will just be easier to understand.

#### Non-Compliant Code {#non-compliant-code-9}

```java
public void dontDoThis() {
  try {
    someMethodThrowingAnException();
  } catch (Exception e) {
    logger.error(e.getMessage(), e);
  }
}
```

#### Compliant Code {#compliant-code-6}

```java
public void doThis() {
  try {
    someMethodThrowingAnException();
  } catch (Exception e) {
    logger.error("Unable to do something", e);
  }
}
```

### Logging in catch blocks should be at the WARN or ERROR level {#logging-in-catch-blocks-should-be-at-the-warn-or-error-level}

**Key**: CQRules:CQBP-44---WrongLogLevelInCatchBlock

**Type**: Code Smell

**Severity**: Minor

**Since**: Version 2018.4.0

As the name suggests, Java exceptions should always be used in *exceptional* circumstances. As a result, when an exception is caught, it is important to ensure that log messages are logged at the appropriate level - either WARN or ERROR. This ensures that those messages appear correctly in the logs.

#### Non-Compliant Code {#non-compliant-code-10}

```java
public void dontDoThis() {
  try {
    someMethodThrowingAnException();
  } catch (Exception e) {
    logger.debug(e.getMessage(), e);
  }
}
```

#### Compliant Code {#compliant-code-7}

```java
public void doThis() {
  try {
    someMethodThrowingAnException();
  } catch (Exception e) {
    logger.error("Unable to do something", e);
  }
}
```

### Do not print stack traces to the console {#do-not-print-stack-traces-to-the-console}

**Key**: CQRules:CQBP-44---ExceptionPrintStackTrace

**Type**: Code Smell

**Severity**: Minor

**Since**: Version 2018.4.0

As mentioned, context is critical when understanding log messages. Using Exception.printStackTrace() causes **only** the stack trace to be output to the standard error stream thereby losing all context. Further, in a multi-threaded application like AEM if multiple exceptions are printed using this method in parallel, their stack traces may overlap which produces significant confusion. Exceptions should be logged through the logging framework only.

#### Non-Compliant Code {#non-compliant-code-11}

```java
public void dontDoThis() {
  try {
    someMethodThrowingAnException();
  } catch (Exception e) {
    e.printStackTrace();
  }
}
```

#### Compliant Code {#compliant-code-8}

```java
public void doThis() {
  try {
    someMethodThrowingAnException();
  } catch (Exception e) {
    logger.error("Unable to do something", e);
  }
}
```

### Do not output to Standard Output or Standard Error {#do-not-output-to-standard-output-or-standard-error}

**Key**: CQRules:CQBP-44—LogLevelConsolePrinters

**Type**: Code Smell

**Severity**: Minor

**Since**: Version 2018.4.0

Logging in AEM should always be done through the logging framework (SLF4J). Outputting directly to the standard output or standard error streams loses the structural and contextual information provided by the logging framework and may, in some cases, cause performance issues.

#### Non-Compliant Code {#non-compliant-code-12}

```java
public void dontDoThis() {
  try {
    someMethodThrowingAnException();
  } catch (Exception e) {
    System.err.println("Unable to do something");
  }
}
```

#### Compliant Code {#compliant-code-9}

```java
public void doThis() {
  try {
    someMethodThrowingAnException();
  } catch (Exception e) {
    logger.error("Unable to do something", e);
  }
}
```

### Avoid Hardcoded /apps and /libs Paths {#avoid-hardcoded-apps-and-libs-paths}

**Key**: CQRules:CQBP-71

**Type**: Code Smell

**Severity**: Minor

**Since**: Version 2018.4.0

In general, paths which start with /libs and /apps should not be hardcoded as the paths they refer to are most commonly stored as paths relative to the Sling search path (which is set to /libs,/apps by default). Using the absolute path may introduce subtle defects which would only appear later in the project lifecycle.

#### Non-Compliant Code {#non-compliant-code-13}

```java
public boolean dontDoThis(Resource resource) {
  return resource.isResourceType("/libs/foundation/components/text");
}
```

#### Compliant Code {#compliant-code-10}

```java
public void doThis(Resource resource) {
  return resource.isResourceType("foundation/components/text");
}
```

### Sling Scheduler Should Not Be Used {#sonarqube-sling-scheduler}

**Key**: CQRules:AMSCORE-554

**Type**: Code Smell/Cloud Service Compatibility

**Severity**: Minor

**Since**: Version 2020.5.0

The Sling Scheduler must not be used for tasks that require a guaranteed execution. Sling Scheduled Jobs guarantee execution and better suited for both clustered and non-clustered environments. 

Refer to [Apache Sling Eventing and Job Handling](https://sling.apache.org/documentation/bundles/apache-sling-eventing-and-job-handling.html) to learn more about how Sling Jobs are handled in a clustered environments.

### AEM Deprecated APIs Should Not Be Used {#sonarqube-aem-deprecated}

**Key**: AMSCORE-553

**Type**: Code Smell/Cloud Service Compatibility

**Severity**: Minor

**Since**: Version 2020.5.0

The AEM API surface is under constant revision to identify APIs for which usage is discouraged and thus considered deprecated. 

In many cases, these APIs are deprecated using the standard Java *@Deprecated* annotation and, as such, as identified by `squid:CallToDeprecatedMethod`. 

However, there are cases where an an API is deprecated in the context of AEM but may not be deprecated in other contexts. This rule identifies this second class.

## OakPAL Content Rules {#oakpal-rules}

Please find below the OakPAL checks executed by Cloud Manager.

>[!NOTE]
>
>OakPAL is a framework developed by an AEM Partner (and winner of 2019 AEM Rockstar North America) which validates content packages using a standalone Oak repository.

### Customer Packages Should Not Create or Modify Nodes Under /libs {#oakpal-customer-package}

**Key**: BannedPaths

**Type**: Bug

**Severity**: Blocker

**Since**: Version 2019.6.0

It has been a long-standing best practice that the /libs content tree in the AEM content repository should be considered read-only by customers. Modifying nodes and properties under */libs* creates significant risk for major and minor updates. Modifications to */libs* should only be made by Adobe through official channels.

### Packages Should Not Contain Duplicate OSGi Configurations {#oakpal-package-osgi}

**Key**: DuplicateOsgiConfigurations

**Type**: Bug

**Severity**: Major

**Since**: Version 2019.6.0

A common problem that occurs on complex projects is where the same OSGi component is configured multiple times. This creates an ambiguity as to which configuration will be operable. This rule is "runmode-aware" in that it will only identify issues where the same component is configured multiple times in the same runmode (or combination of runmodes).

This rule will produce issues where the same configuration, at the same path, is defined in multiple packages, including cases where the same package is duplicated in the overall list of built packages. For example, if the build produces packages named `com.myco:com.myco.ui.apps` and `com.myco:com.myco.all` where `com.myco:com.myco.all` embeds `com.myco:com.myco.ui.apps`, then all of the configurations within `com.myco:com.myco.ui.apps` will be reported as duplicates. This is generally a case of not following the [Content Package Structure Guidelines](https://experienceleague.adobe.com/docs/experience-manager-cloud-service/implementing/developing/aem-project-content-package-structure.html#cloud-manager-target); in this specific example, the package `com.myco:com.myco.ui.apps` is missing the `<cloudManagerTarget>none</cloudManagerTarget>` property.

#### Non Compliant Code {#non-compliant-code-osgi}

```

+ apps
  + projectA
    + config
      + com.day.cq.commons.impl.ExternalizerImpl
  + projectB
    + config
      + com.day.cq.commons.impl.ExternalizerImpl
```

#### Compliant Code {#compliant-code-osgi}

```

+ apps
  + shared-config
    + config
      + com.day.cq.commons.impl.ExternalizerImpl
```

### Config and Install Folders Should Only Contain OSGi Nodes {#oakpal-config-install}

**Key**: ConfigAndInstallShouldOnlyContainOsgiNodes

**Type**: Bug

**Severity**: Major

**Since**: Version 2019.6.0

For security reasons, paths containing */config/ and /install/* are only readable by administrative users in AEM and should be used only for OSGi configuration and OSGi bundles. Placing other types of content under paths which contain these segments results in application behavior which unintentionally varies between administrative and non-administrative users.

A common problem is use of nodes named `config` within component dialogs or when specifying the rich text editor configuration for inline editing. To resolve this the offending node should be renamed to a compliant name. For the rich text editor configuration make use of the `configPath` property on the `cq:inplaceEditing` node to specify the new location.

#### Non Compliant Code {#non-compliant-code-config-install}

```

+ cq:editConfig [cq:EditConfig]
  + cq:inplaceEditing [cq:InplaceEditConfig]
    + config [nt:unstructured]
      + rtePlugins [nt:unstructured]
```

#### Compliant Code {#compliant-code-config-install}

```

+ cq:editConfig [cq:EditConfig]
  + cq:inplaceEditing [cq:InplaceEditConfig]
    ./configPath = inplaceEditingConfig (String)
    + inplaceEditingConfig [nt:unstructured]
      + rtePlugins [nt:unstructured]
```

### Packages Should Not Overlap {#oakpal-no-overlap}

**Key**: PackageOverlaps

**Type**: Bug

**Severity**: Major

**Since**: Version 2019.6.0

Similar to the *Packages Should Not Contain Duplicate OSGi Configurations* this is a common problem on complex projects where the same node path is written to by multiple separate content packages. While using content package dependencies can be used to ensure a consistent result, it is better to avoid overlaps entirely. 

### Default Authoring Mode Should Not Be Classic UI {#oakpal-default-authoring}

**Key**: ClassicUIAuthoringMode

**Type**: Code Smell/Cloud Service Compatibility

**Severity**: Minor

**Since**: Version 2020.5.0

The OSGi configuration `com.day.cq.wcm.core.impl.AuthoringUIModeServiceImpl` defines the default authoring mode within AEM. As Classic UI has been deprecated since AEM 6.4, an issue will now be raised when the default authoring mode is configured to Classic UI.

### Components With Dialogs Should Have Touch UI Dialogs {#oakpal-components-dialogs}

**Key**: ComponentWithOnlyClassicUIDialog

**Type**: Code Smell/Cloud Service Compatibility

**Severity**: Minor

**Since**: Version 2020.5.0

AEM Components which have a Classic UI dialog should always have a corresponding Touch UI dialog both to provide an optimal authoring experience and to be compatible with the Cloud Service deployment model, where Classic UI is not supported. This rule verifies the following scenarios:

* A component with a Classic UI dialog (that is, a dialog child node) must have a corresponding Touch UI dialog (that is, a `cq:dialog` child node).
* A component with a Classic UI design dialog (i.e. a design_dialog node) must have a corresponding Touch UI design dialog (that is, a `cq:design_dialog` child node).
* A component with both a Classic UI dialog and a Classic UI design dialog must have both a corresponding Touch UI dialog and a corresponding Touch UI design dialog.

The AEM Modernization Tools documentation provides documentation and tooling for how to convert components from Classic UI to Touch UI. Refer  to [The AEM Modernization Tools](https://opensource.adobe.com/aem-modernize-tools/pages/tools.html) for more details.

### Packages Should Not Mix Mutable and Immutable Content {#oakpal-packages-immutable}

**Key**: ImmutableMutableMixedPackage

**Type**: Code Smell/Cloud Service Compatibility

**Severity**: Minor

**Since**: Version 2020.5.0

In order to be compatible with the Cloud Service deployment model, individual content packages must contain either content for the immutable areas of the repository (that is, `/apps and /libs, although /libs` should not be modified by customer code and will cause a separate violation) or the mutable area (that is, everything else), but not both. For example, a package which includes both `/apps/myco/components/text and /etc/clientlibs/myco` is not compatible with Cloud Service and will cause an issue to be reported.

Refer to [AEM Project Structure](https://docs.adobe.com/content/help/en/experience-manager-cloud-service/implementing/developing/aem-project-content-package-structure.html) for more details.

### Reverse Replication Agents Should Not Be Used {#oakpal-reverse-replication}

**Key**: ReverseReplication

**Type**: Code Smell/Cloud Service Compatibility

**Severity**: Minor

**Since**: Version 2020.5.0

Support for Reverse Replication is not available in Cloud Service deployments, as described in [Release Notes: Removal of Replication Agents](https://docs.adobe.com/content/help/en/experience-manager-cloud-service/release-notes/aem-cloud-changes.html#replication-agents).

Customers using reverse replication should contact Adobe for alternative solutions.

### OakPAL - Resources Contained in Proxy-Enabled Client Libraries Should Be in a folder named resources {#oakpal-resources-proxy}

**Key**: ClientlibProxyResource

**Type**: Bug

**Severity**: Minor

**Since**: Version 2021.2.0

AEM client libraries may contain static resources like images and fonts. As described in [Using Preprocessors](https://experienceleague.adobe.com/docs/experience-manager-65/developing/introduction/clientlibs.html?lang=en#using-preprocessors), when using proxied client libraries these static resources must  be contained in a child folder named resources in order to be effectively referenced on the publish instances.

#### Non Compliant Code {#non-compliant-proxy-enabled}

```

+ apps
  + projectA
    + clientlib
      - allowProxy=true
      + images
        + myimage.jpg
```

#### Compliant Code {#compliant-proxy-enabled}

```

+ apps
  + projectA
    + clientlib
      - allowProxy=true
      + resources
        + myimage.jpg
```

### OakPAL - Usage of Cloud Service Incompatible Workflow Processes {#oakpal-usage-cloud-service}

**Key**: CloudServiceIncompatibleWorkflowProcess

**Type**: Code Smell

**Severity**: Blocker

**Since**: Version 2021.2.0

With the move to Asset micro-services for asset processing on AEM Cloud Service, several workflow processes which were used in on-premise and AMS versions of AEM have become either unsupported or unnecessary. The migration tool at [aem-cloud-migration](https://github.com/adobe/aem-cloud-migration) can be used to update workflow models during AEM Cloud Service migration.

### OakPAL - Usage of Static Templates is Discouraged in Favor of Editable Templates {#oakpal-static-template}

**Key**: StaticTemplateUsage

**Type**: Code Smell

**Severity**: Minor

**Since**: Version 2021.2.0

While the use of static templates has historically been very common in AEM projects, editable templates are highly recommended as they provide the most flexibility and support additional features not present in static templates. More information can be found on [Page Templates - Editable](https://experienceleague.adobe.com/docs/experience-manager-65/developing/platform/templates/page-templates-editable.html?lang=en). Migration from static to editable templates can be largely automated using the [AEM Modernization Tools](https://opensource.adobe.com/aem-modernize-tools/).

### OakPAL - Usage of Legacy Foundation Components is Discouraged {#oakpal-usage-legacy}

**Key**: LegacyFoundationComponentUsage

**Type**: Code Smell

**Severity**: Minor

**Since**: Version 2021.2.0

The legacy foundation components (i.e. components under `/libs/foundation`) have been deprecated for several AEM releases in favor of the WCM Core Components. Usage of the legacy foundation components as the basis for custom components – whether by overlay or inheritance – is discouraged and should be converted to the corresponding core component. This conversion can be facilitated by the [AEM Modernization Tools](https://opensource.adobe.com/aem-modernize-tools/).

### OakPAL - Only Supported Runmode Names and Ordering Should Be Used {#oakpal-supported-runmodes}

**Key**: SupportedRunmode

**Type**: Code Smell

**Severity**: Minor

**Since**: Version 2021.2.0

AEM Cloud Service enforces a strict naming policy for runmode names and a strict ordering for those runmodes. The list of supported runmodes can be found on [Runmodes](https://experienceleague.adobe.com/docs/experience-manager-cloud-service/implementing/deploying/overview.html?lang=en#runmodes) and any deviation from this will be identified as an issue.

### OakPAL - Custom Search Index Definition Nodes Must Be Direct Children of /oak:index {#oakpal-custom-search}

**Key**: OakIndexLocation

**Type**: Code Smell

**Severity**: Minor

**Since**: Version 2021.2.0

AEM Cloud Service requires that custom search index definitions (i.e. nodes of type oak:QueryIndexDefinition) be direct child nodes of `/oak:index`. Indexes in other locations must be moved to be compatible with AEM Cloud Service. More information on search indexes can be found on [Content Search and Indexing](https://experienceleague.adobe.com/docs/experience-manager-cloud-service/operations/indexing.html?lang=en).

### OakPAL - Custom Search Index Definition Nodes Must Have a compatVersion of 2 {#oakpal-custom-search-compatVersion}

**Key**: IndexCompatVersion

**Type**: Code Smell

**Severity**: Minor

**Since**: Version 2021.2.0

AEM Cloud Service requires that custom search index definitions (i.e. nodes of type oak:QueryIndexDefinition) must have the compatVersion property set to 2. Any other value is not supported by AEM Cloud Service. More information on search indexes can be found on [Content Search and Indexing](https://experienceleague.adobe.com/docs/experience-manager-cloud-service/operations/indexing.html?lang=en).

### OakPAL - Descendent Nodes of Custom Search Index Definition Nodes Must Be Of Type nt:unstructured {#oakpal-descendent-nodes}

**Key**: IndexDescendantNodeType

**Type**: Code Smell

**Severity**: Minor

**Since**: Version 2021.2.0

Hard to troubleshoot issues can occur when occur when a custom search index definition node has unordered child nodes. To avoid these, it is recommended that all descendent nodes of an `oak:QueryIndexDefinition` node be of type nt:unstructured.

### OakPAL - Custom Search Index Definition Nodes Must Contain a Child Node Named indexRules that Has Children {#oakpal-custom-search-index}

**Key**: IndexRulesNode

**Type**: Code Smell

**Severity**: Minor

**Since**: Version 2021.2.0

A properly defined custom search index definition node must contain a child node named indexRules which, in turn must have at least one child. More information can be found on [Oak Documentation](https://jackrabbit.apache.org/oak/docs/query/lucene.html).

### OakPAL - Custom Search Index Definition Nodes Must Follow Naming Conventions {#oakpal-custom-search-definitions}

**Key**: IndexName

**Type**: Code Smell

**Severity**: Minor

**Since**: Version 2021.2.0

AEM Cloud Service requires that custom search index definitions (that is, nodes of type `oak:QueryIndexDefinition`) must be named following a specific pattern described on [Content Search and Indexing](https://experienceleague.adobe.com/docs/experience-manager-cloud-service/operations/indexing.html?lang=en#how-to-use).

### OakPAL - Custom Search Index Definition Nodes Must Use the Index Type lucene  {#oakpal-index-type-lucene}

**Key**: IndexType

**Type**: Code Smell

**Severity**: Minor

**Since**: Version 2021.2.0

AEM Cloud Service requires that custom search index definitions (i.e. nodes of type oak:QueryIndexDefinition) have a type property with the value set to **lucene**. Indexing using legacy index types must be updated before migration to AEM Cloud Service. See [Content Search and Indexing](https://experienceleague.adobe.com/docs/experience-manager-cloud-service/operations/indexing.html?lang=en#how-to-use) for more information.

### OakPAL - Custom Search Index Definition Nodes Must Not Contain a Property Named seed {#oakpal-property-name-seed}

**Key**: IndexSeedProperty

**Type**: Code Smell

**Severity**: Minor

**Since**: Version 2021.2.0

AEM Cloud Service prohibits custom search index definitions (that is, nodes of type `oak:QueryIndexDefinition`) from containing a property named seed. Indexing using this property must be updated before migration to AEM Cloud Service. See [Content Search and Indexing](https://experienceleague.adobe.com/docs/experience-manager-cloud-service/operations/indexing.html?lang=en#how-to-use) for more information.

### OakPAL - Custom Search Index Definition Nodes Must Not Contain a Property Named reindex {#oakpal-reindex-property}

**Key**: IndexReindexProperty

**Type**: Code Smell

**Severity**: Minor

**Since**: Version 2021.2.0

AEM Cloud Service prohibits custom search index definitions (that is, nodes of type `oak:QueryIndexDefinition`) from containing a property named reindex. Indexing using this property must be updated before migration to AEM Cloud Service. See [Content Search and Indexing](https://experienceleague.adobe.com/docs/experience-manager-cloud-service/operations/indexing.html?lang=en#how-to-use) for more information.

## Dispatcher Optimization Tool {#dispatcher-optimization-tool-rules}

The following section highlights the DOT checks executed by Cloud Manager:

* [DOT - Parsing Violation - Dispatcher Configuration Unexpected Tokens](https://github.com/adobe/aem-dispatcher-optimizer-tool/blob/main/docs/Rules.md#dot---parsing-violation---dispatcher-configuration-unexpected-tokens)

* [DOT - Parsing Violation - Dispatcher Configuration Unmatched Quote](https://github.com/adobe/aem-dispatcher-optimizer-tool/blob/main/docs/Rules.md#dot---parsing-violation---dispatcher-configuration-unmatched-quote)

* [DOT - Parsing Violation - Dispatcher Configuration Missing Brace](https://github.com/adobe/aem-dispatcher-optimizer-tool/blob/main/docs/Rules.md#dot---parsing-violation---dispatcher-configuration-missing-brace)

* [DOT - Parsing Violation - Dispatcher Configuration Extra Brace](https://github.com/adobe/aem-dispatcher-optimizer-tool/blob/main/docs/Rules.md#dot---parsing-violation---dispatcher-configuration-extra-brace)

* [DOT - Parsing Violation - Dispatcher Configuration Missing Mandatory Property](https://github.com/adobe/aem-dispatcher-optimizer-tool/blob/main/docs/Rules.md#dot---parsing-violation---dispatcher-configuration-missing-mandatory-property)

* [DOT - Parsing Violation - Dispatcher Configuration Deprecated Property](https://github.com/adobe/aem-dispatcher-optimizer-tool/blob/main/docs/Rules.md#dot---parsing-violation---dispatcher-configuration-deprecated-property)

* [DOT - Parsing Violation - Dispatcher Configuration Not Found](https://github.com/adobe/aem-dispatcher-optimizer-tool/blob/main/docs/Rules.md#dot---parsing-violation---dispatcher-configuration-not-found)

* [DOT - Parsing Violation - Httpd Configuration Include file not found](https://github.com/adobe/aem-dispatcher-optimizer-tool/blob/main/docs/Rules.md#dot---parsing-violation---httpd-configuration-include-file-not-found)

* [DOT - Parsing Violation - Dispatcher Configuration General](https://github.com/adobe/aem-dispatcher-optimizer-tool/blob/main/docs/Rules.md#dot---parsing-violation---dispatcher-configuration-general)

* [DOT - The Dispatcher publish farm cache should have serveStaleOnError enabled](https://github.com/adobe/aem-dispatcher-optimizer-tool/blob/main/docs/Rules.md#dot---the-dispatcher-publish-farm-cache-should-have-servestaleonerror-enabled)

* [DOT - The Dispatcher publish farm filters should contain the default deny rules from the 6.x.x version of the AEM archetype](https://github.com/adobe/aem-dispatcher-optimizer-tool/blob/main/docs/Rules.md#dot---the-dispatcher-publish-farm-filters-should-contain-the-default-deny-rules-from-the-6xx-version-of-the-aem-archetype)

* [DOT - The Dispatcher publish farm cache statfileslevel property should be >= 2](https://github.com/adobe/aem-dispatcher-optimizer-tool/blob/main/docs/Rules.md#dot---the-dispatcher-publish-farm-cache-statfileslevel-property-should-be--2)

* [DOT - The Dispatcher publish farm gracePeriod property should be >= 2](https://github.com/adobe/aem-dispatcher-optimizer-tool/blob/main/docs/Rules.md#dot---the-dispatcher-publish-farm-graceperiod-property-should-be--2)

* [DOT - Each Dispatcher farm should have a unique name](https://github.com/adobe/aem-dispatcher-optimizer-tool/blob/main/docs/Rules.md#dot---each-dispatcher-farm-should-have-a-unique-name)

* [DOT - The Dispatcher publish farm cache should have its ignoreUrlParams rules configured in an allow list manner](https://github.com/adobe/aem-dispatcher-optimizer-tool/blob/main/docs/Rules.md#dot---the-dispatcher-publish-farm-cache-should-have-its-ignoreurlparams-rules-configured-in-an-allow-list-manner)

* [DOT - The Dispatcher publish farm filters should specify the allowed Sling selectors in an allow list manner](https://github.com/adobe/aem-dispatcher-optimizer-tool/blob/main/docs/Rules.md#dot---the-dispatcher-publish-farm-filters-should-specify-the-allowed-sling-selectors-in-an-allow-list-manner)

* [DOT - The Dispatcher publish farm filters should specify the allowed Sling suffix patterns in an allow list manner](https://github.com/adobe/aem-dispatcher-optimizer-tool/blob/main/docs/Rules.md#dot---the-dispatcher-publish-farm-filters-should-specify-the-allowed-sling-suffix-patterns-in-an-allow-list-manner)

* [DOT - The 'Require all granted' directive should not be used in a VirtualHost Directory section with a root directory-path](https://github.com/adobe/aem-dispatcher-optimizer-tool/blob/main/docs/Rules.md#dot---the-require-all-granted-directive-should-not-be-used-in-a-virtualhost-directory-section-with-a-root-directory-path)
