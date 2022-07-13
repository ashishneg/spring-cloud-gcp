## Spring Cloud GCP Core

Each Spring Cloud GCP module uses `GcpProjectIdProvider` and
`CredentialsProvider` to get the GCP project ID and access credentials.

Spring Cloud GCP provides a Spring Boot starter to auto-configure the
core components.

Maven coordinates, using [Spring Cloud GCP
BOM](getting-started.xml#bill-of-materials):

``` xml
<dependency>
    <groupId>com.google.cloud</groupId>
    <artifactId>spring-cloud-gcp-starter</artifactId>
</dependency>
```

Gradle coordinates:

    dependencies {
        implementation("com.google.cloud:spring-cloud-gcp-starter")
    }

### Configuration

The following options may be configured with Spring Cloud core.

|                                 |                                                 |          |               |
| ------------------------------- | ----------------------------------------------- | -------- | ------------- |
| Name                            | Description                                     | Required | Default value |
| `spring.cloud.gcp.core.enabled` | Enables or disables GCP core auto configuration | No       | `true`        |

### Project ID

`GcpProjectIdProvider` is a functional interface that returns a GCP
project ID string.

``` java
public interface GcpProjectIdProvider {
    String getProjectId();
}
```

The Spring Cloud GCP starter auto-configures a `GcpProjectIdProvider`.
If a `spring.cloud.gcp.project-id` property is specified, the provided
`GcpProjectIdProvider` returns that property value.

``` java
spring.cloud.gcp.project-id=my-gcp-project-id
```

Otherwise, the project ID is discovered based on an [ordered list of
rules](https://googlecloudplatform.github.io/google-cloud-java/google-cloud-clients/apidocs/com/google/cloud/ServiceOptions.html#getDefaultProjectId--):

1.  The project ID specified by the `GOOGLE_CLOUD_PROJECT` environment
    variable

2.  The Google App Engine project ID

3.  The project ID specified in the JSON credentials file pointed by the
    `GOOGLE_APPLICATION_CREDENTIALS` environment variable

4.  The Google Cloud SDK project ID

5.  The Google Compute Engine project ID, from the Google Compute Engine
    Metadata Server

### Credentials

`CredentialsProvider` is a functional interface that returns the
credentials to authenticate and authorize calls to Google Cloud Client
Libraries.

``` java
public interface CredentialsProvider {
  Credentials getCredentials() throws IOException;
}
```

The Spring Cloud GCP starter auto-configures a `CredentialsProvider`. It
uses the `spring.cloud.gcp.credentials.location` property to locate the
OAuth2 private key of a Google service account. Keep in mind this
property is a Spring Resource, so the credentials file can be obtained
from a number of [different
locations](https://docs.spring.io/spring/docs/current/spring-framework-reference/html/resources.html#resources-implementations)
such as the file system, classpath, URL, etc. The next example specifies
the credentials location property in the file system.

    spring.cloud.gcp.credentials.location=file:/usr/local/key.json

Alternatively, you can set the credentials by directly specifying the
`spring.cloud.gcp.credentials.encoded-key` property. The value should be
the base64-encoded account private key in JSON format.

If that credentials aren’t specified through properties, the starter
tries to discover credentials from a [number of
places](https://github.com/GoogleCloudPlatform/google-cloud-java#authentication):

1.  Credentials file pointed to by the `GOOGLE_APPLICATION_CREDENTIALS`
    environment variable

2.  Credentials provided by the Google Cloud SDK `gcloud auth
    application-default login` command

3.  Google App Engine built-in credentials

4.  Google Cloud Shell built-in credentials

5.  Google Compute Engine built-in credentials

If your app is running on Google App Engine or Google Compute Engine, in
most cases, you should omit the `spring.cloud.gcp.credentials.location`
property and, instead, let the Spring Cloud GCP Starter get the correct
credentials for those environments. On App Engine Standard, the [App
Identity service account
credentials](https://cloud.google.com/appengine/docs/standard/java/appidentity/)
are used, on App Engine Flexible, the [Flexible service account
credential](https://cloud.google.com/appengine/docs/flexible/java/service-account)
are used and on Google Compute Engine, the [Compute Engine Default
Service
Account](https://cloud.google.com/compute/docs/access/create-enable-service-accounts-for-instances#using_the_compute_engine_default_service_account)
is used.

#### Scopes

By default, the credentials provided by the Spring Cloud GCP Starter
contain scopes for every service supported by Spring Cloud GCP.

|                      |                                                                                                 |
| -------------------- | ----------------------------------------------------------------------------------------------- |
| Service              | Scope                                                                                           |
| Spanner              | <https://www.googleapis.com/auth/spanner.admin>, <https://www.googleapis.com/auth/spanner.data> |
| Datastore            | <https://www.googleapis.com/auth/datastore>                                                     |
| Pub/Sub              | <https://www.googleapis.com/auth/pubsub>                                                        |
| Storage (Read Only)  | <https://www.googleapis.com/auth/devstorage.read_only>                                          |
| Storage (Read/Write) | <https://www.googleapis.com/auth/devstorage.read_write>                                         |
| Runtime Config       | <https://www.googleapis.com/auth/cloudruntimeconfig>                                            |
| Trace (Append)       | <https://www.googleapis.com/auth/trace.append>                                                  |
| Cloud Platform       | <https://www.googleapis.com/auth/cloud-platform>                                                |
| Vision               | <https://www.googleapis.com/auth/cloud-vision>                                                  |

The Spring Cloud GCP starter allows you to configure a custom scope list
for the provided credentials. To do that, specify a comma-delimited list
of [Google OAuth2
scopes](https://developers.google.com/identity/protocols/googlescopes)
in the `spring.cloud.gcp.credentials.scopes` property.

`spring.cloud.gcp.credentials.scopes` is a comma-delimited list of
[Google OAuth2
scopes](https://developers.google.com/identity/protocols/googlescopes)
for Google Cloud Platform services that the credentials returned by the
provided `CredentialsProvider` support.

    spring.cloud.gcp.credentials.scopes=https://www.googleapis.com/auth/pubsub,https://www.googleapis.com/auth/sqlservice.admin

You can also use `DEFAULT_SCOPES` placeholder as a scope to represent
the starters default scopes, and append the additional scopes you need
to add.

    spring.cloud.gcp.credentials.scopes=DEFAULT_SCOPES,https://www.googleapis.com/auth/cloud-vision

### Environment

`GcpEnvironmentProvider` is a functional interface, auto-configured by
the Spring Cloud GCP starter, that returns a `GcpEnvironment` enum. The
provider can help determine programmatically in which GCP environment
(App Engine Flexible, App Engine Standard, Kubernetes Engine or Compute
Engine) the application is deployed.

``` java
public interface GcpEnvironmentProvider {
    GcpEnvironment getCurrentEnvironment();
}
```

### Customizing bean scope

Spring Cloud GCP starters autoconfigure all necessary beans in the
default singleton scope. If you need a particular bean or set of beans
to be recreated dynamically (for example, to rotate credentials), there
are two options:

1.  Annotate custom beans of the necessary types with `@RefreshScope`.
    This makes the most sense if your application is already redefining
    those beans.

2.  Override the scope for autoconfigured beans by listing them in the
    Spring Cloud property `spring.cloud.refresh.extra-refreshable`.
    
    For example, the beans involved in Cloud Pub/Sub subscription could
    be marked as refreshable as follows:

<!-- end list -->

    spring.cloud.refresh.extra-refreshable=com.google.cloud.spring.pubsub.support.SubscriberFactory,\
      com.google.cloud.spring.pubsub.core.subscriber.PubSubSubscriberTemplate

<div class="note">

`SmartLifecycle` beans, such as Spring Integration adapters, do not
currently support `@RefreshScope`. If your application refreshes any
beans used by such `SmartLifecycle` objects, it may also have to restart
the beans manually when `RefreshScopeRefreshedEvent` is detected, such
as in the Cloud Pub/Sub example below:

``` java
@Autowired
private PubSubInboundChannelAdapter pubSubAdapter;

@EventListener(RefreshScopeRefreshedEvent.class)
public void onRefreshScope(RefreshScopeRefreshedEvent event) {
  this.pubSubAdapter.stop();
  this.pubSubAdapter.start();
}
```

</div>

### Spring Initializr

This starter is available from [Spring
Initializr](https://start.spring.io/) through the `GCP Support` entry.