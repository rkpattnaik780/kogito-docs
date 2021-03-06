# Kogito Data Index Service

This service aims for capturing and indexing data produced by one more Kogito runtime services.

Overall goals include:
* Focus on domain data ( Orders, Travel, etc ) 
* Flexible data structure
* Infinispan as first-class persistence service
* Distributable and cloud-ready
* Messaging based communication with Kogito runtime ( Kafka - Cloud Events )
* Powerful querying API using GraphQL

It is also important to note that this service is not intended to be used as permanent storage or audit log information. Focus is to make business domain data easily accessible for processes that are currently in progress.

![](https://github.com/kiegroup/kogito-runtimes/blob/master/docsimg/data-index-architecture.jpg)

## Technical Overview
From a technical perspective, the Data Index is a Quarkus application based on VertX and reactive messaging that exposes a GraphQL endpoint, allowing client applications to easily access business domain-specific data as well as technical detailed information about running process instance.

In its current version, it uses Kafka messaging to consume [CloudEvents](https://cloudevents.io) based messages from Kogito runtimes, process and index the information for later consumption via GraphQL queries.
These events contain information about units of work executed for a process. Visit [process-runtime-events](https://github.com/kiegroup/kogito-runtimes/wiki/Configuration#process-runtime-events) for more details about the payload of the messages.
This data is then parsed and pushed into different Infinispan caches.
These caches are structured as follows:
 - Domain cache: This cache is a generic cache, one per process id, where the process instance variables are pushed as the root content. This cache also includes
some process instance metadata, which allows correlating data between domain and process instances. Data is transferred as JSON format to Infinispan server as a concrete Java type is not available. 
 - Process instance cache: Each process instance is pushed here containing all information, not only metadata, that includes extra information such as nodes executed.
 - User Task instance cache: Each user task instance is pushed here containing all information, not only metadata, that includes extra information such as inputs and outputs.

Storage is provided by [Infinispan](https://infinispan.org/) which enables a cloud-ready and scalable persistence, as well as [Lucene](https://lucene.apache.org/) based indexing. Communication between the Index Service and Infinispan is handled via [Protocol Buffers](https://developers.google.com/protocol-buffers/).

Once the data is indexed and stored into the cache, the Data Index service inspects the process model in order to update the [GraphQL](https://graphql.org) schema, allowing a type-checked query system for consumer clients.

# Instructions for developers

Data Index service is a Quarkus based application that aims to consume [CloudEvents](https://cloudevents.io) based messages from Kogito runtimes, process and index the information for later consumption via GraphQL queries.
These events contain information about units of work executed for one or multiple processes. Visit [process-runtime-events](https://github.com/kiegroup/kogito-runtimes/wiki/Configuration#process-runtime-events) for more details about the payload of the messages.
This data is then parsed and pushed into different Infinispan caches.
These caches are structured as follows:
 - Domain cache: This cache process type specific cache, one per process id, where the process instance variables are pushed as the root content. This cache also includes
some process instance metadata, which allows correlating data between domain and process instances. Data is transferred as JSON format to Infinispan server as a concrete Java type is not available. 
 - Process instance cache: Each process instance is pushed here containing all information, not only metadata, that includes extra information such as nodes executed.

Storage is provided by [Infinispan](https://infinispan.org/) and integrated using `quarkus-infinispan-client`. Communication between the Index Service and Infinispan is handled via [Protocol Buffers](https://developers.google.com/protocol-buffers/) which requires creating
`.proto` files and marshallers to read and write bytes. For more information, visit the [Quarkus Infinispan Client Guide](https://quarkus.io/guides/infinispan-client-guide).

In order to enable indexing of custom process models, the Data Index Service consumes proto files generated for a given process, see [process-instance-variables](https://github.com/kiegroup/kogito-runtimes/wiki/Persistence#process-instance-variables).
That can be added by simply making the proto files available in a folder for the service to read once starting.
Once the Protocol Buffer model is parsed, a respective [GraphQL](https://graphql.org) type is generated to allow clients to query the information. Again, the process data is available
via the two caches, where users can query based on the technical aspects (Process Instances) or domain-specific (proto defined type).

The Data Index Service is also a [Vert.X](https://vertx.io) based application for more details, visit [using-vertx](https://quarkus.io/guides/using-vertx). An important aspect
provided by Vert.X is the native integration with [Java GraphQL](https://www.graphql-java.com/) provided by the [vertx-web-graphql](https://vertx.io/docs/vertx-web-graphql/java/).

Messaging integration is also provided by Quarkus using `quarkus-smallrye-reactive-messaging-kafka`. For more details and configuration details, visit [kafka-guide](https://quarkus.io/guides/kafka-guide) and [smallrye-reactive-messaging](https://smallrye.io/smallrye-reactive-messaging/).

## Getting started

You will need:
 - Infinispan server:
   Download and run https://downloads.jboss.org/infinispan/10.0.0.CR1/infinispan-server-10.0.0.CR1.zip.
 Service will be available on port 11222. This should match the setting `quarkus.infinispan-client.server-list=localhost:11222` on `application.properties`.
   To enable our simplified demo setup, go to /server/conf/infinispan.xml and remove the security domain from the endpoints definition:
   ```xml
   <endpoints socket-binding="default">
   ```
   You also need to add a basic template for the indexing cache:
   ```xml
   <local-cache-configuration name="kogito-template" statistics="true">
         <indexing index="ALL">
            <property name="default.directory_provider">local-heap</property>
         </indexing>
   </local-cache-configuration>
   ```
 - Kafka messaging server:
   The best way to get started is to use the Docker image that already contains all the necessary bits for running Kafka locally.
   For more details visit: https://hub.docker.com/r/spotify/kafka/ 
   ```
   docker run -p 2181:2181 -p 9092:9092 --env ADVERTISED_HOST=localhost --env ADVERTISED_PORT=9092 spotify/kafka
   ``` 
   For a comprehensive list of options for setting up the Data Index Kafka consumer please visit: [configuring-the-kafka-connector](https://quarkus.io/guides/kafka-guide#configuring-the-kafka-connector) and [Kafka consumer configuration](https://kafka.apache.org/documentation/#consumerconfigs)
   
### Running

As a Quarkus based application, running the service for development takes full benefit of the dev mode support with live code reloading.
For that simply run:
```
mvn clean compile quarkus:dev
```

Once the service is up and running, it will start consuming messages from the  Kafka topic named: `kogito-processinstances-events`.
For more details on how to enable a Kogito runtime to produce events, please visit [publishing-events](https://github.com/kiegroup/kogito-runtimes/wiki/Configuration#publishing-events).

When running on dev mode, the [GraphiQL](https://github.com/graphql/graphiql) UI is available, on `http://localhost:8180/`, which allows exploring and querying the available data model. Alternatively, it is also possible to use a GraphQL client API to communicate with the exposed endpoint at `http://localhost:8180/graphql`.

In there you can explore the current types available using the Docs section on the top right and execute queries on the model.
Some examples:

#### Querying the technical cache

##### Process instances
```graphql
{
  ProcessInstances {
    id
    processId
    state
    parentProcessInstanceId
    rootProcessId
    rootProcessInstanceId
    variables
    nodes {
      id
      name
      type
    }
  }
}
```  
##### User Task instances
```graphql
{
  UserTaskInstances {
    id
    name
    actualOwner
    description
    priority
    processId
    processInstanceId
  }
}
```  

##### Filtering 
The provided GraphQL schema also allows for further filtering of the results. A _where_ attribute is optional and allows multiple combinations. A few examples:

```graphql
{
  ProcessInstances(where: {state: {equal: ACTIVE}}) {
    id
    processId
    processName
    start
    state
    variables
  }
}
``` 

```graphql
{
  ProcessInstances(where: {id: {equal: "d43a56b6-fb11-4066-b689-d70386b9a375"}}) {
    id
    processId
    processName
    start
    state
    variables
  }
}
``` 

```graphql
{
  UserTaskInstances(where: {state: {equal: "Ready"}}) {
    id
    name
    actualOwner
    description
    priority
    processId
    processInstanceId
  }
}
``` 

Depending on the attribute type, some operators are available, for instance:

* String array argument:
  * contains : String
  * containsAll: Array of String
  * containsAny: Array of String
  * isNull: Boolean ( true| false )

* String argument
  * in: Array of String
  * like: String
  * isNull: Boolean ( true| false )
  * equal: String

* Id argument
  * in: Array of String
  * equal: String
  * isNull: Boolean ( true| false )

* Boolean argument
  * isNull: Boolean ( true| false )
  * equal: Boolean ( true| false )

* Numeric argument
  * in: Array of Integer
  * isNull: Boolean
  * equal: Integer
  * greaterThan: Integer
  * greaterThanEqual: Integer
  * lessThan: Integer
  * lessThanEqual: Integer
  * between: Numeric range: from: Integer to: Integer

* Date argument
  * isNull: Boolean ( true| false )
  * equal: Date Time
  * greaterThan: Date Time
  * greaterThanEqual: Date Time
  * lessThan: Date Time
  * lessThanEqual: Date Time
  * between: Date Range: from: Date Time to: Date Time

###### Combining AND and OR operators
By default, every attribute that is filtered on will be executed as an AND operation in query execution. This can be tweaked by combining filters with an AND or OR operator. Example:
```graphql
{
  ProcessInstances(where: {or: {state: {equal: ACTIVE}, rootProcessId: {isNull: false}}}) {
    id
    processId
    processName
    start
    end
    state
  }
}
``` 
```graphql
{
  ProcessInstances(where: {and: {processId: {equal: "travels"}, or: {state: {equal: ACTIVE}, rootProcessId: {isNull: false}}}}) {
    id
    processId
    processName
    start
    end
    state
  }
}
``` 

##### Sorting
Sorting of results is possible via _orderBy_ parameter, in there, some of the attributes from either ProcessInstances or UserTaskInstances can be used to sort the results. For each attribute available, it is necessary to also specify the direction os sorting if ASC or DESC.
Example:
```graphql
{
  ProcessInstances(where: {state: {equal: ACTIVE}}, orderBy: {start: ASC}) {
    id
    processId
    processName
    start
    end
    state
  }
}
``` 

##### Pagination
Pagination is also supported via a _pagination_ attribute, that allows specifying a limit and offset to the returned data set.
Example:
```graphql
{
  ProcessInstances(where: {state: {equal: ACTIVE}}, orderBy: {start: ASC}, pagination: {limit: 10, offset: 0}) {
    id
    processId
    processName
    start
    end
    state
  }
}
```
Multiple attributes can be applied to the _orderBy_ parameter, these will be applied to the database query in the order they are specified in the query filter.
 Example:
```graphql
{
  UserTaskInstances(where: {state: {equal: "Ready"}}, orderBy: {name: ASC, actualOwner: DESC}) {
    id
    name
    actualOwner
    description
    priority
    processId
    processInstanceId
  }
}
```

#### Querying the domain cache
Assuming a Travels model is deployed
```graphql
{
  Travels {
    visaApplication {
      duration
    }
    flight {
      flightNumber
      gate
    }
    hotel {
      name
      address {
        city
        country
      }
    }
    traveller {
      firstName
      lastName
      nationality
      email
    }
  }
}
``` 
##### Metadata
If needed, it is also possible to correlate domain cache with process and tasks. This is possible via the _metadata_ attribute. Which allows not only retrieving data but also filtering. This attribute is added to all root models deployed in the data index service. To query the metadata, simply select the appropriate attributes under _metadata_ attribute. For example:
```graphql
{
  Travels {
    flight {
      flightNumber
      arrival
      departure
    }
    metadata {
      lastUpdate
      userTasks {
        name
      }
      processInstances {
        processId
      }
    }
  }
}
``` 

##### Filtering
Filtering the domain-specific cache is also based on a type-based system. Allowing typed searches in different attributes, similarly to the parameters used for filtering Processes and Tasks. The attributes available for search depend on the model that is deployed. Below we can demonstrate some capabilities based on the Travel Agency domain.

* List all Travels for travellers that first name starts with `Cri`.
```graphql
{
  Travels(where: {traveller: {firstName: {like: "Cri*"}}}) {
    flight {
      flightNumber
      arrival
      departure
    }
    traveller {
      email
    }
  }
}
``` 
Please note that _like_ operator is case sensitive.

###### Filtering based on metadatada

* List the flight details related to a specific process instance.
```graphql
{
  Travels(where: {metadata: {processInstances: {id: {equal: "1aee8ab6-d943-4dfb-b6be-8ea8727fcdc5"}}}}) {
    flight {
      flightNumber
      arrival
      departure
    }
  }
}
``` 
* List the flight details related to a specific user task instance.
```graphql
{
  Travels(where: {metadata: {userTasks: {id: {equal: "de52e538-581f-42db-be65-09e8739471a6"}}}}) {
    flight {
      flightNumber
      arrival
      departure
    }
  }
}
``` 

##### Sorting
Similarly to sorting Processes and Tasks, sorting in the domain cache can be done in any of the attributes, including sub-types. The direction of the sort can be defined via its value. For example:
```graphql
{
  Travels(orderBy: {trip: {begin: ASC}}) {
    flight {
      flightNumber
      arrival
      departure
    }
  }
}
``` 
##### Pagination
Similarly to Processes and Tasks, pagination is defined via _limit_ and _offset_ parameters. For example:
```graphql
{
  Travels(where: {traveller: {firstName: {like: "Cri*"}}}, pagination: {offset: 0, limit: 10}) {
    flight {
      flightNumber
      arrival
      departure
    }
    traveller {
      email
    }
  }
}
``` 

#### Loading proto files from the file system
To bootstrap the service using a set of proto files from a folder, simply pass the following property `kogito.protobuf.folder` and any .proto file contained in the folder will automatically be loaded when the application starts.
You can also set up the service to reload/load any changes to files during the service execution by setting `kogito.protobuf.watch=true`.

```
mvn clean compile quarkus:dev -Dkogito.protobuf.folder=/home/git/kogito-runtimes/data-index/data-index-service/src/test/resources -Dkogito.protobuf.watch=true
```

### Building

To generate the final build artifact, sim run:

```
mvn clean install
```

The Maven build will generate a uber jar at `target/data-index-service-${version}-runner.jar`, that can be executed via:
```
java -jar target/data-index-service-8.0.0-SNAPSHOT-runner.jar
```
 
### Infinispan Indexing

Infinispan embeds a Lucene engine in order to provide indexing of models in the cache. To be able to hint the engine about which attributes should be indexed,
it needs to annotate the proto file attributes using Hibernate Search annotations  `@Indexed` and `@Field`.
For more details, visit [indexing_of_protobuf_encoded_entries](https://infinispan.org/docs/dev/user_guide/user_guide.html#indexing_of_protobuf_encoded_entries).  

Sample indexed model:
```
/* @Indexed */
message ProcessInstanceMeta {
    /* @Field(store = Store.YES) */
    optional string id = 1;
}
```

