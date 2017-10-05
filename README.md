# Event Emitter

Event emitter is a Keycloak module based on EventListener service provider interface (SPI).
During the lifecycle of Keycloak, Event and AdminEvent are created when specific actions are created.
The aim of this module is to send those Events and AdminEvents to another server in a serialized format.


## Installation
Event emitter module is expected to be installed as a module in a specific layer.

```Bash
#Create layer in keycloak setup
install -d -v -m755 /opt/keycloak/modules/system/layers/eventemitter -o keycloak -g keycloak

#Setup the module directory
install -d -v -m755 /opt/keycloak/modules/system/layers/eventemitter/io/cloudtrust/keycloak/main/ -o keycloak -g keycloak

#Install jar
install -v -m0755 -o keycloak -g keycloak -D target/event-emitter-1.0.Final.jar /opt/keycloak/modules/system/layers/eventemitter/io/cloudtrust/keycloak/main/

#Install module file
install -v -m0755 -o keycloak -g keycloak -D src/main/resources/module.xml /opt/keycloak/modules/system/layers/eventemitter/io/cloudtrust/keycloak/main/

```

module.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<module xmlns="urn:jboss:module:1.3" name="io.cloudtrust.keycloak.module.eventemitter">
    <resources>
        <resource-root path="event-emitter-1.0-Final.jar"/>
    </resources>
    <dependencies>
        <module name="org.keycloak.keycloak-core"/>
        <module name="org.keycloak.keycloak-server-spi"/>
        <module name="org.keycloak.keycloak-server-spi-private"/>
        <module name="org.jboss.logging"/>
        <module name="org.apache.httpcomponents"/>
        <module name="com.fasterxml.jackson.core.jackson-databind"/>
        <module name="com.fasterxml.jackson.core.jackson-core"/>
        <module name="org.apache.commons.collections4"/>
        <module name="com.google.guava"/>
    </dependencies>
</module>
```

As far as possible, existing dependencies are used by this module but some of them are new ones that need to be added.
Download JAR dependency of commons-collections4 and Guava with the version specified in the pom.xml.

Dependencies to add:
* commons-collections4
* guava

```Bash
#Create the module directory for collections4
install -d -v -m755 /opt/keycloak/modules/system/layers/eventemitter/org/apache/commons/collections4/main -o keycloak -g keycloak

#Create the module directory for Guava
install -d -v -m755 /opt/keycloak/modules/system/layers/eventemitter/com/google/guava/main -o keycloak -g keycloak

#Install jar
install -v -m0755 -o keycloak -g keycloak -D commons-collections4-4.1.jar /opt/keycloak/modules/system/layers/eventemitter/org/apache/commons/collections4/main
install -v -m0755 -o keycloak -g keycloak -D guava-23.1-jre.jar /opt/keycloak/modules/system/layers/eventemitter/com/google/guava/main


#Install module file
install -v -m0755 -o keycloak -g keycloak -D module.xml /opt/keycloak/modules/system/layers/eventemitter/org/apache/commons/collections4/main
install -v -m0755 -o keycloak -g keycloak -D module.xml /opt/keycloak/modules/system/layers/eventemitter/com/google/guava/main

```

module.xml for Collections4
```xml
<?xml version="1.0" encoding="UTF-8"?>
<module xmlns="urn:jboss:module:1.3" name="org.apache.commons.collections4">
    <properties>
        <property name="jboss.api" value="private"/>
    </properties>

    <resources>
        <resource-root path="commons-collections4-4.1.jar"/>
    </resources>

    <dependencies>
    </dependencies>
</module>
```


module.xml for Guava
```xml
<?xml version="1.0" encoding="UTF-8"?>
<module xmlns="urn:jboss:module:1.3" name="com.google.guava">
    <properties>
        <property name="jboss.api" value="private"/>
    </properties>

    <resources>
        <resource-root path="guava-23.1-jre.jar"/>
    </resources>

    <dependencies>
    </dependencies>
</module>
```

Enable the newly created layer, edit __layers.conf__:
```Bash
layers=keycloak,eventemitter
```



## Enable & Configure

In __standalone.xml__, add the new module and configure it

```xml
<web-context>auth</web-context>
<providers>
    <provider>module:com.quest.keycloak-wsfed</provider>
    ...
</providers>
...
<theme>
    <modules>
            <module>
                    com.quest.keycloak-wsfed
            </module>
    </modules>
    ...
</theme>
...
```

Configuration parameters:


Note that if any parameter is invalid or missing, keycloak fails to start with a error message in the log about the cause.

After file edition, restart keycloak

Finally, to make the event emitter functional we hate to register it via the admin console.
In Manage - Events, go to Config tab and add event-emitter among the Event listeners.

Note that configuration parameters can be seen in Server Info, tab Providers.


## Development tips
### Compilation
To produce the JAR of this module just use maven in a standard way:
```Bash
mvn package
```

### Flatbuffers

Flatbuffers schema is located under src/main/flatbuffers/flatbuffers/event.fbs.

Compilation of the schema
```Bash
$FLATC_HOME/flatc --java events.fbs
```
Generated classes must be located in src/main/java/flatbuffers/events

*Quick note for flatc installation*
```Bashde 
$ git clone https://github.com/google/flatbuffers.git
$ cd flatbuffers
$ cmake -G "Unix Makefiles"
$ make
$ ./flattests # this is quick, and should print "ALL TESTS PASSED"
```
(Source: https://rwinslow.com/posts/how-to-install-flatbuffers/)

To use flatbuffers, some classes are needed. No package are available to manage those dependencies, thus classes must be copy paste directly in the project.


### Idempotence
A unique id is added to the serialized Events and AdminEvents in order to uniquely identify each of them and thus ensure the storage unicity on the target server.
The unique ID generation is ensured by Snowflake ID generation which ensure unicity of ID among multiple keycloak nodes and datacenters.


### Logging
Logging level usage:
* DEBUG : verbose information for debug/development purpose.
* INFO : Informative message about current lifecycle of the module.
* ERROR : Fatal error. Recoverable errors are logged at INFO level.


### Buffer
If the target server is not available, the Events and AdminEvents are stored in a Queue.
This queue has a configurable limited capacity. When the queue is full, the oldest event is dropped to store  the new one.
For each new events or adminEvents, the event-emitter will try to send all the events stored in the buffer.
Events remains in the buffer until they are sucessfully received by the target or dropped to make space for new ones.


### Concurrency 
Factory is application-scoped while provider is request-scoped (hence single-threaded).
Different threads can use multiple providers concurrently.
Provider doesn't need to be thread-safe but factory should be, that's why the Queue used to store the events is concurrency-safe.
(Mailing list keycloak-dev, answer from Marek Posolda <mposolda@redhat.com>)

