# Buid Log-based integration with SAP Data Intelligence

Log-based integration is getting popular for integrating different data systems. By capturing the changes in a database and continually apply the same changes to derived data systems, the data in the derived systems is match with the data in the database as long as the message ordering of changed data stream is preserved. The derived data systems are consumers of the changed data stream, as illustrated in below figure.<br><br>
![](images/illustration.png)

> The derived data systems are downstream consumers, in practice, they might be indexes, caches, analytics systems, etc. 

In this toturial, we want to explore how to achieve data integration between soure database and various other derived data systems using a log-based approach in SAP Data Intelligence.

In order to build a log-based intergtation approach, we need two critical compotents:
- A CDC tool for capturing database changes.
- A message borker for persisting messages stream while preserving the message ordering.

SAP Data Intelligence provides hundreds of out-of-box operators for user to choose and combine them into a powerful data pipline to implement data driven application. Now Let's see how SAP Data Intelligence's built-in operators can satisfy above two requirements.

### CDC
Change data capture(CDC) is the process of observing all data changes written to a database and extracting them in a form in which they can be replicated to other systems. 

SAP Data Intelligence 3.0 introduced the table Replicator operator which allows capturing delta changes for different databases. Besides capturing the delta, this operator also handles applying the changes to a target table, allowing replication of tables in an efficient and secure way, through an in-built recovery mechanism.

Table Replicator operator effectively implements the concept of change data capture(CDC) using an approach of trigger-based replication. For more detail, see https://help.sap.com/viewer/97fce0b6d93e490fadec7e7021e9016e/Cloud/en-US/79fcadb91f584f868a6662111b92f6e7.html.

### Kafka
Apache Kafka is a message broker which provides a total ordering of the messages inside each partition. A partition lives on a physical node and persists the messages it receives.

SAP Data Intelligence has built-in Kafka producer/consumer operators.

## Getting started
Now we have our tools ready, let's see how to implement a concret integration scenario.

For the soure database table, its table schema is illustrated as below.<br><br>
![](images/hanaSourceSchema.png)

The table records are illustrated below.<br><br>
![](images/HanaSourceTable.png)

As you can see, the table initially contains 6 records.
> For simplicity, we only use factitious data for demonstration purpose.

Our approach involves two steps which play different roles.
1. Initial loading source table data.
2. Continuously capture source table changes(Delta extraction).

We will create separate graphs for these two steps.

## Initial loading [(Graph source code)](https://github.com/Andyyh2005/log-based-integration-with-DI/blob/master/src/vrep/vflow/graphs/CDC_InitialLoading_test/graph.json)
The following figure illustrates the initial loading graph:<br><br>
![](images/GraphInitialLoading.png)

## Delta extraction[(Graph source code)](https://github.com/Andyyh2005/log-based-integration-with-DI/blob/master/src/vrep/vflow/graphs/CDC_InitialLoading_test/graph.json)
The following figure illustrates the initial loading graph:<br><br>
![](images/GraphInitialLoading.png)