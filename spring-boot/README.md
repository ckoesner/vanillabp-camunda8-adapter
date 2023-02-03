![Draft](../readme/vanillabp-headline.png)
# Camunda 8 adapter

This is an adapter which implements the binding of the Blueprint SPI to [Camunda 8](https://docs.camunda.io/) in order to run business processes.

Camunda 8 is a BPMN engine that represents a closed system similar to a database. The engine itself uses a cloud native storage concept to enable horizontal scaling. By using the remote API BPMN can be deployed and processes started. In order to execute BPMN tasks a client has to subscribe and will be pushed if work is available. Next to the BPMN engine other optional components are required to handle user tasks or to operate the system. Those components do not access the engine's data but rather use exported data stored in Elastic Search. As it's nature suggests it fits good to mid- and high-scaled use-cases.

This adapter is aware of all the details needed to keep in mind on using Camunda 8.

To run Camunda 8 on your local computer for development purposes consider to use [docker-compose](https://github.com/camunda/camunda-platform#using-docker-compose).

## Usage

Just add this dependency to your project, no additional dependencies from Camunda needed:

```xml
<dependency>
  <groupId>org.camunda.community.vanillabp</groupId>
  <artifactId>camunda8-spring-boot-adapter</artifactId>
  <version>1.0.0</version>
</dependency>
```

If you want a certain version of Zeebe client then you have to replace the transitive dependencies like this:

```xml
<dependency>
  <groupId>io.camunda</groupId>
  <artifactId>spring-zeebe-starter</artifactId>
  <version>8.1.14</version>
</dependency>
<dependency>
  <groupId>org.camunda.community.vanillabp</groupId>
  <artifactId>camunda7-spring-boot-adapter</artifactId>
  <version>1.0.0</version>
  <exclusions>
    <exclusion>
      <groupId>io.camunda</groupId>
      <artifactId>spring-zeebe-starter</artifactId>
    </exclusion>
  </exclusions>
</dependency>
```

*Hint:* This adapter is compatible with the configuration of the regular Zeebe Spring Boot auto-configuration. However, some additional configuration is described in the upcoming sections.

## Features

### Worker ID

When using asynchronious task processing one has to define a worker id. There is no default value to avoid bringing anything unwanted into production. On using [VanillaBP's SpringApplication](https://github.com/vanillabp/spring-boot-support#spring-boot-support) instead of `org.springframework.boot.SpringApplication` [additional support](https://github.com/vanillabp/spring-boot-support#worker-id) is available.

### Module aware deployment

To avoid interdependencies between the implementation of different use-cases packed into a single microservice the concept of [workflow modules](https://github.com/vanillabp/spring-boot-support#workflow-modules) is introduced. This adapter builds a Camunda deployment for each workflow module found in the classpath.

Since Camunda 8 does not provide BPMS meta-data (e.g. historical deployments) the adapter stores necessary meta-data in separate DB tables to ensure correct operation.

### SPI Binding validation

On starting the application BPMNs of all workflow modules will be wired to the SPI. This includes

1. BPMN files which are part of the current workflow module bundle (e.g. classpath:processes/*.bpmn)
1. BPMN files deployed as part of previous versions of the workflow module
   1. Older versions of BPMN files which are part of the current workflow module bundle
   1. BPMN files which are not part of the current workflow module bundle any more
   
This ensures that correct wiring of all process definitions according to the SPI is done.

#### Multi-instance

Since Camunda 8 is a remote engine the workflow is processed in a different runtime environment. Due to this fact the Blueprint adapter cannot do the entire binding of multi-instance context information under the hood. In the BPMN the multi-instance aspects like the input element, the element's index and the total number of elements have to be defined according to a certain naming convention:

1. The input element: The ID of the BPMN element (e.g. `Activity_RequestRide`)
1. The current element's index: The ID of the BPMN element plus `_index` (e.g. `Activity_RequestRide_index`)
1. The total number of elements: The ID of the BPMN element plus `_total` (e.g. `Activity_RequestRide_total`)

Last two one needs to be defined as input mappings where the index always points to `=loopCounter` and the total number of elements can be determinated using an expression like `=count(nearbyDrivers)` where "nearbyDrivers" is the collection used as a basis for multi-instance iteration.

This naming convention might look a little bit strange but is necessary to hide the BPMS in the business code. Unfortunately, since Camunda 8 is a remote engine, BPMNs get a little bit verbose as information needed to process the BPMN cannot be passed on-the-fly/at runtime and have to be provided upfront.

#### Message correlation IDs

On using receive tasks Camunda 8 requires us to define correlation IDs. If your process does not have any special correlation strategy then use the expression `=id` and use the two-parameter method `ProcessService#correlateMessage` to correlate incoming messages.

In case your model has more than one receive task active you have to define unique correlation IDs for each receive task of that message name to enable the BPMS to find the right receive task to correlate to. This might happen for multi-instance receive tasks or receive tasks within a multi-instance embedded sub-process. In that case use the workflow id in combination with the multi-instance element as a correlation id: `=id+"+"+RequestRideOffer` (where "RequestRideOffer" is the name of the multi-instance element).

### Transaction behavior

Since Camunda 8 is an external system to your services one has to deal with eventual consistency in conjunction with transactions. This adapter uses the recommended pattern to report a task as completed and roll back the local transaction in case of errors. Possible errors are:

1. Camunda 8 is not reachable / cannot process the confirmation of a completed task
1. The task to be completed was cancelled meanwhile e.g. due to boundary events

If there is an exception in your business code and you have to roll back the transaction then Camunda's task retry-mechanism should retry as configured. Additionally, the `TaskException` is used for expected business errors handled by BPMN error boundary events which must not cause a rollback. To achieve both one should mark the service bean like this:

```java
@Service
@WorkflowService(workflowAggregateClass = Ride.class)
@Transactional(noRollbackFor = TaskException.class)
public class TaxiRide {
```
