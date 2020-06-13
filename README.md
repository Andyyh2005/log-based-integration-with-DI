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

Table Replicator operator effectively implements the CDC using an approach of trigger-based replication. For more detail, see https://help.sap.com/viewer/97fce0b6d93e490fadec7e7021e9016e/Cloud/en-US/79fcadb91f584f868a6662111b92f6e7.html.

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

We would first do an initial load to load all the 6 records to a file in Data Intelligence local file system. Then Continuously capture the subsequent table changes into a Kafka topic, and further apply the changes to the downstream operators.

Our approach involves two sequential tasks playing different roles.
1. Initial loading source table data.
2. Continuously capture source table changes(Delta extraction).

We will create separate graphs for these two tasks.

## Initial loading [(Graph source code)](https://github.com/Andyyh2005/log-based-integration-with-DI/blob/master/src/vrep/vflow/graphs/CDC_InitialLoading_test/graph.json)
The following figure illustrates the initial loading graph:<br><br>
![](images/GraphInitialLoading.png)

The configuration of the table replicator operator is illustrated as below.<br><br>
![](images/ConfigTableReplicatorInitiaLoading.png)
Some of the important cofiguration parameters are marked in red box.
> Note that the **deltaGrapMode** is set to Manual. This ensures the graph would finish its execution once the intial loading completed. Otherwise, the graph would run indefinitely to track further delta changes.

Now run the graph and verify the target file has been generated.<br><br>
![](images/FileInitialLoading.png)

Also, open the file and verify the table content has been replicated to the target file successfully.<br><br>
![](images/FileInitialLoadingContent.png)

Now we see that our initial loading task has been finished successfully. let's turn to the Delta extraction task.

## Delta extraction[(Graph source code)](https://github.com/Andyyh2005/log-based-integration-with-DI/blob/master/src/vrep/vflow/graphs/CDC_InitialLoading_test/graph.json)
The following figure illustrates the Delta extraction graph:<br><br>
![](images/GraphDeltaExtraction.png)

Let's take an overview of the dataflow sequence of this grpah.
- The constant generator operator will trigger the Table Replicator to begin the CDC delta tracking once the graph start running.
- The Table Replicator operator(labeled as "CDC (delta tracking)") will replicated the database changes to a tagret file. 
- The JS operator(labeled as "Remove path prefix") will remove the '/vrep' prefix from the target file path. The prefix was added by Table Replicator operator which will prevent the downstream Read File operator from finding the file if we do not remove it.
- The Read File operator(labeled as "Read Delta File")will read the target file content and send its content to downstream JS operator.
- The JS operator(labeled as "Parse & send changes") will parse the received file content and send the parsed change message into Kafka.
- The downstream Kakfa Producer operator will receieve the incoming messages and publish them into the specified topic on the Kafka cluster.
- The Kafka Consumers subscribe the specified topic and consume the messages from the Kafka cluster.
- The Connected Wiretap opertor(labeled as "Change consumer1" and "Change consumer2") acts as the derived data system applying the change messages.

Now let's take a look at the configuraion of some operators in this graph.

### Table Replicator
Its configuration is illustrated as below.<br><br>
![](images/ConfigTableReplicatorDeltaTracking.png)

Some of the important cofiguration parameters are marked in red box.
> Note that the **deltaGrapMode** is set to "Polling Interval", and the **maxPollingInterval** is set to "60". This ensures the graph would run indefinitely to track delta changes, and Table Replicator will polling the delta changes within one minute.

### The "Remove path prefix" JS operator 
The script code of this operator is shown below
```
$.setPortCallback("input",onInput);

function onInput(ctx,s) {
    var msg = {};

    msg.Attributes = {};
    for (var key in s.Attributes) {
        msg.Attributes[key] = s.Attributes[key];
    }
    
    msg.Body = s.Body.replace(/^\/vrep/, '');
    $.output(msg);
}
```
It simply remove the "/vrep" prefix from the receieved message body(The body contains the target file path)

### The "Parse & send changes JS operator 
The script code of this operator is shown below.
```
$.setPortCallback("input",onInput);

function isByteArray(data) {
    switch (Object.prototype.toString.call(data)) {
        case "[object Int8Array]":
        case "[object Uint8Array]":
            return true;
        case "[object Array]":
        case "[object GoArray]":
            return data.length > 0 && typeof data[0] === 'number';
    }
    return false;
}

function onInput(ctx,s) {
    var inbody = s.Body;
    var inattributes = s.Attributes;
    
    var msg = {};
    msg.Attributes = {};
    for (var key in inattributes) {
        msg.Attributes[key] = inattributes[key];
    }

    // convert the body into string if it is bytes
    if (isByteArray(inbody)) {
        inbody = String.fromCharCode.apply(null, inbody);
    }
    
    var lines = inbody.split(/\r\n/);
    
    if (typeof inbody === 'string') {
        // if the body is a string (e.g., a plain text or json string),
        msg.Attributes["js.action"] = "parseFile";
        
        var readOffset = 1;
        var dataCols = lines[0].split(',');
        var o_inter = {};
        var fields = [];

        lines.slice(readOffset).forEach(function(line) {
            if(line.length !== 0){
                fields = line.split(',')
                dataCols.forEach(function(c, i) {
                    o_inter[c] = fields[i];
                    
                });
                ++readOffset;
                msg.Body = o_inter;
                $.output(msg);
           }
        });
    }
    else {
        // if the body is an object (e.g., a json object),
        // forward the body and indicate it in attribute js.action
        msg.Body = inbody;
        msg.Attributes["js.action"] = "noop";
        $.output(msg);
    }
}

```
It simply parse the target file contnet line by line. And each line represents a change message and sent to the downstream operator.

### Kafka Producer
Its configuration is illustrated as below.<br><br>
![](images/ConfigKafkaProducer.png)

Some of the important cofiguration parameters are marked in red box.

### Kafka Consumer1
Its configuration is illustrated as below.<br><br>
![](images/ConfigKafkaConsumer1.png)

Some of the important cofiguration parameters are marked in red box.

### Kafka Consumer2
Its configuration is illustrated as below.<br><br>
![](images/ConfigKafkaConsumer2.png)

Some of the important cofiguration parameters are marked in red box.

> Note that the "Group ID" configuration are different for the two consumers. This is important since we want to achieve fan-out messaging. That is, a single Kafka partition consumed by multiple consumers, each maintaining its own message offset.

The graph is ready. Now We can start run the graph by clicking the run button. 

Next we will start to write to source table through insert, update and delete operations and observe how the changes will be replicated into the derived systems.

### Insert
Let's insert one record into the source table by running the below SQL statement.
```
INSERT INTO "TRIALEDITION1"."HANA_SOURCE" VALUES (7, 3, 40)
```

Go to Data Intelligence Local file system workspaces, check and verify the target file has been generated, as below figure illustrated.<br><br>
![](images/FileDeltaInsert.png)

Open that file and verify the inserted record has been replicated to the target file successfully, as below figure illustrated.<br><br>
![](images/FileIDeltaInsertContent.png)

Finally, we open the Change consumer wiretap UI to check its output, we can see both wiretap output the same thing, as below figure illustrated.<br><br>
![](images/changeConsumer1Insert.png)

#### Update
Let's update one record in the source table by running the below SQL statement.

```
UPDATE "TRIALEDITION1"."HANA_SOURCE" SET TEMPERATURE = TEMPERATURE+ 1 WHERE ID = 1
```

Go to Data Intelligence Local file system workspaces, check and verify the target file has been generated, as below figure illustrated.<br><br>
![](images/FileDeltaUpdate.png)

Open that file and verify the change messages of the updateed record has been replicated to the target file successfully, as below figure illustrated.<br><br>
![](images/FileIDeltaUpdateContent.png)

Finally, we open the Change consumer wiretap UI to check its output, we can see both wiretap output the same thing, as below figure illustrated.<br><br>
![](images/changeConsumer1Update.png)

### Delete
Let's delete one record in the source table by running the below SQL statement.

```
DELETE FROM "TRIALEDITION1"."HANA_SOURCE" WHERE  ID = 7; 
```
Go to Data Intelligence Local file system workspaces, check and verify the target file has been generated, as below figure illustrated.<br><br>
![](images/FileDeltaDelete.png)

Open that file and verify the change messages for the deleted record has been replicated to the target file successfully, as below figure illustrated.<br><br>
![](images/FileIDeltaDeleteContent.png)

Finally, we open the Change consumer wiretap UI to check its output, we can see both wiretap output the same thing, as below figure illustrated.<br><br>
![](images/changeConsumer1Delete.png.png)

