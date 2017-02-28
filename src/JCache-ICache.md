

## Hazelcast JCache Extension - ICache

Hazelcast provides extension methods to Cache API through the interface `com.hazelcast.cache.ICache`.

It has two sets of extensions:

* Asynchronous version of all cache operations. See [Async Operations](#icache-async-methoods).
* Cache operations with custom `ExpiryPolicy` parameter to apply on that specific operation. See [Custom ExpiryPolicy](#defining-a-custom-expirypolicy).

### Scoping to Join Clusters

As mentioned before, you can scope a `CacheManager` in the case of a client to connect to multiple clusters. In the case of an embedded member, you can scope a `CacheManager` to join different clusters at the same time. This process is called scoping. To apply scoping, request
a `CacheManager` by passing a `java.net.URI` instance to `CachingProvider::getCacheManager`. The `java.net.URI` instance must point to either a Hazelcast configuration or to the name of a named
`com.hazelcast.core.HazelcastInstance` instance.

<br></br>
![image](images/NoteSmall.jpg) ***NOTE:*** *Multiple requests for the same `java.net.URI` result in returning a `CacheManager`
instance that shares the same `HazelcastInstance` as the `CacheManager` returned by the previous call.*
<br></br>

#### Summary of use cases and scoping options

|Use case|Scoping solution|
|--------|----------------|
|Given an existing `HazelcastInstance`, create a `CacheManager` that uses this `HazelcastInstance`.| * [Bind to existing Hazelcast Instance object](#binding-to-an-existing-hazelcast-instance-object) using `Properties`<br> * [Bind to instance by name](#binding-to-a-named-instance) by using the instance name as `URI` as or via `Properties`|
|Given an existing configuration, create a `CacheManager` that starts a new `HazelcastInstance` with this configuration or locates an existing one that was already started with this configuration.| * [Apply configuration scope](#applying-configuration-scope) by using config location as `URI` or via `Properties`| 


#### Applying Configuration Scope

To connect or join different clusters, apply a configuration scope to the `CacheManager`. If the same `URI` is
used to request a `CacheManager` that was created previously, those `CacheManager`s share the same underlying `HazelcastInstance`.

To apply configuration scope you can do either one of the following:
 - pass the path to the configuration file using the location property
`HazelcastCachingProvider#HAZELCAST_CONFIG_LOCATION` (which resolves to `hazelcast.config.location`) as a mapping inside a
`java.util.Properties` instance to the `CachingProvider#getCacheManager(uri, classLoader, properties)` call.
 - use directly the configuration path as the `CacheManager`'s `URI`.

If both `HazelcastCachingProvider#HAZELCAST_CONFIG_LOCATION` property is set and the `CacheManager` `URI` resolves to a valid config file location, then the property value will be used to obtain the configuration for the `HazelcastInstance` the first time a `CacheManager` is created for the given `URI`.
 
Here is an example of using Configuration Scope.

```java
CachingProvider cachingProvider = Caching.getCachingProvider();

// Create Properties instance pointing to a Hazelcast config file
Properties properties = new Properties();
// "scope-hazelcast.xml" resides in package com.domain.config
properties.setProperty( HazelcastCachingProvider.HAZELCAST_CONFIG_LOCATION,
    "classpath:com/domain/config/scoped-hazelcast.xml" );

URI cacheManagerName = new URI( "my-cache-manager" );
CacheManager cacheManager = cachingProvider
    .getCacheManager( cacheManagerName, null, properties );
```

Here is an example using `HazelcastCachingProvider::propertiesByLocation` helper method.

```java
CachingProvider cachingProvider = Caching.getCachingProvider();

// Create Properties instance pointing to a Hazelcast config file in root package
String configFile = "classpath:scoped-hazelcast.xml";
Properties properties = HazelcastCachingProvider
    .propertiesByLocation( configFile );

URI cacheManagerName = new URI( "my-cache-manager" );
CacheManager cacheManager = cachingProvider
    .getCacheManager( cacheManagerName, null, properties );
```

The retrieved `CacheManager` is scoped to use the `HazelcastInstance` that was just created and was configured using the given XML
configuration file.

Available protocols for config file URL include `classpath` to point to a classpath location, `file` to point to a filesystem
location, `http` and `https` for remote web locations. In addition, everything that does not specify a protocol is recognized
as a placeholder that can be configured using a system property.

```java
String configFile = "my-placeholder";
Properties properties = HazelcastCachingProvider
    .propertiesByLocation( configFile );
```

You can set this on the command line.

```plain
-Dmy-placeholder=classpath:my-configs/scoped-hazelcast.xml
```

You should consider the following rules about the Hazelcast instance name when you specify the configuration file location using `HazelcastCachingProvider#HAZELCAST_CONFIG_LOCATION` (which resolves to `hazelcast.config.location`):

* If you also specified the `HazelcastCachingProvider#HAZELCAST_INSTANCE_NAME` (which resolves to `hazelcast.instance.name`) property, this property is used as the instance name even though you configured the instance name in the configuration file.
* If you do not specify `HazelcastCachingProvider#HAZELCAST_INSTANCE_NAME` but you configure the instance name in the configuration file using the element `<instance-name>`, this element's value will be used as the instance name.
* If you do not specify an instance name via property or in the configuration file, the URL of the configuration file location is used as the instance name.

<br></br>
![image](images/NoteSmall.jpg) ***NOTE:*** *No check is performed to prevent creating multiple `CacheManager`s with the same cluster
configuration on different configuration files. If the same cluster is referred from different configuration files, multiple
cluster members or clients are created.*

![image](images/NoteSmall.jpg) ***NOTE:*** *The configuration file location will not be a part of the resulting identity of the
`CacheManager`. An attempt to create a `CacheManager` with a different set of properties but an already used name will result in
undefined behavior.*
<br></br>

#### Binding to a Named Instance

You can bind `CacheManager` to an existing and named `HazelcastInstance` instance. If the `instanceName` is specified in `com.hazelcast.config.Config`, it can be used directly by passing it to `CachingProvider` implementation. Otherwise (`instanceName` not set or instance is a client instance) you must get the instance name from the `HazelcastInstance` instance via the `String getName()` method to pass the `CachingProvider` implementation. Please note that `instanceName` is not configurable for the client side `HazelcastInstance` instance and is auto-generated by using group name (if it is specified). In general, `String getName()` method over `HazelcastInstance` is safer and the preferable way to get the name of the instance. Multiple `CacheManager`s created using an equal `java.net.URI` will share the same `HazelcastInstance`.

A named scope is applied nearly the same way as the configuration scope: pass the instance name using:
 - either property `HazelcastCachingProvider#HAZELCAST_INSTANCE_NAME` (which resolves to `hazelcast.instance.name`) as a mapping inside a `java.util.Properties` instance to the `CachingProvider#getCacheManager(uri, classLoader, properties)` call.
 - or use the instance name when specifying the `CacheManager`'s `URI`.

If a valid instance name is provided both as property and as `URI`, then the property value takes precedence and is used to resolve the `HazelcastInstance` the first time a `CacheManager` is created for the given `URI`.

Here is an example of Named Instance Scope with specified name.

```java
Config config = new Config();
config.setInstanceName( "my-named-hazelcast-instance" );
// Create a named HazelcastInstance
Hazelcast.newHazelcastInstance( config );

CachingProvider cachingProvider = Caching.getCachingProvider();

// Create Properties instance pointing to a named HazelcastInstance
Properties properties = new Properties();
properties.setProperty( HazelcastCachingProvider.HAZELCAST_INSTANCE_NAME,
     "my-named-hazelcast-instance" );

URI cacheManagerName = new URI( "my-cache-manager" );
CacheManager cacheManager = cachingProvider
    .getCacheManager( cacheManagerName, null, properties );
```

Here is an example of Named Instance Scope with specified name passed as `URI` of the `CacheManager`.

```java
Config config = new Config();
config.setInstanceName( "my-named-hazelcast-instance" );
// Create a named HazelcastInstance
Hazelcast.newHazelcastInstance( config );

CachingProvider cachingProvider = Caching.getCachingProvider();
URI cacheManagerName = new URI( "my-named-hazelcast-instance" );
CacheManager cacheManager = cachingProvider
    .getCacheManager( cacheManagerName, null);
```

Here is an example of Named Instance Scope with auto-generated name.

```java
Config config = new Config();
// Create a auto-generated named HazelcastInstance
HazelcastInstance instance = Hazelcast.newHazelcastInstance( config );
String instanceName = instance.getName();

CachingProvider cachingProvider = Caching.getCachingProvider();

// Create Properties instance pointing to a named HazelcastInstance
Properties properties = new Properties();
properties.setProperty( HazelcastCachingProvider.HAZELCAST_INSTANCE_NAME, 
     instanceName );

URI cacheManagerName = new URI( "my-cache-manager" );
CacheManager cacheManager = cachingProvider
    .getCacheManager( cacheManagerName, null, properties );
```

Here is an example of Named Instance Scope with auto-generated name on client instance.

```java
ClientConfig clientConfig = new ClientConfig();
ClientNetworkConfig networkConfig = clientConfig.getNetworkConfig();
networkConfig.addAddress("127.0.0.1", "127.0.0.2");

// Create a client side HazelcastInstance
HazelcastInstance instance = HazelcastClient.newHazelcastClient( clientConfig );
String instanceName = instance.getName();

CachingProvider cachingProvider = Caching.getCachingProvider();

// Create Properties instance pointing to a named HazelcastInstance
Properties properties = new Properties();
properties.setProperty( HazelcastCachingProvider.HAZELCAST_INSTANCE_NAME, 
     instanceName );

URI cacheManagerName = new URI( "my-cache-manager" );
CacheManager cacheManager = cachingProvider
    .getCacheManager( cacheManagerName, null, properties );
```

Here is an example using `HazelcastCachingProvider::propertiesByInstanceName` method.

```java
Config config = new Config();
config.setInstanceName( "my-named-hazelcast-instance" );
// Create a named HazelcastInstance
Hazelcast.newHazelcastInstance( config );

CachingProvider cachingProvider = Caching.getCachingProvider();

// Create Properties instance pointing to a named HazelcastInstance
Properties properties = HazelcastCachingProvider
    .propertiesByInstanceName( "my-named-hazelcast-instance" );

URI cacheManagerName = new URI( "my-cache-manager" );
CacheManager cacheManager = cachingProvider
    .getCacheManager( cacheManagerName, null, properties );
```

<br></br>
![image](images/NoteSmall.jpg) ***NOTE:*** *The `instanceName` will not be a part of the resulting identity of the `CacheManager`.
An attempt to create a `CacheManager` with a different set of properties but an already used name will result in undefined behavior.*
<br></br>

#### Binding to an existing Hazelcast Instance object

When an existing `HazelcastInstance` object is available, it can be passed to the `CacheManager` by setting property `HazelcastCachingProvider#HAZELCAST_INSTANCE_ITSELF`:

```java
// Create a member HazelcastInstance
HazelcastInstance instance = Hazelcast.newHazelcastInstance();

Properties properties = new Properties();
properties.setProperty( HazelcastCachingProvider.HAZELCAST_INSTANCE_ITSELF, 
     instance );

CachingProvider cachingProvider = Caching.getCachingProvider();
// cacheManager initialized for uri will be bound to instance
CacheManager cacheManager = cachingProvider.getCacheManager(uri, classLoader, properties);
```

### Namespacing

The `java.net.URI`s that don't use the above-mentioned Hazelcast-specific schemes are recognized as namespacing. Those
`CacheManager`s share the same underlying default `HazelcastInstance` created (or set) by the `CachingProvider`, but they cache with the
same names and different namespaces on the `CacheManager` level, and therefore they won't share the same data. This is useful where multiple
applications might share the same Hazelcast JCache implementation, e.g., on application or OSGi servers, but are developed by
independent teams. To prevent interfering on caches using the same name, every application can use its own namespace when
retrieving the `CacheManager`.

Here is an example of using namespacing.

```java
CachingProvider cachingProvider = Caching.getCachingProvider();

URI nsApp1 = new URI( "application-1" );
CacheManager cacheManagerApp1 = cachingProvider.getCacheManager( nsApp1, null );

URI nsApp2 = new URI( "application-2" );
CacheManager cacheManagerApp2 = cachingProvider.getCacheManager( nsApp2, null );
```

That way both applications share the same `HazelcastInstance` instance but not the same caches.

