---
title: Ingest data from OpenTelemetry to Azure Data Explorer
description: Learn how to use Azure Data Explorer as an OpenTelemetry sink.
ms.date: 10/03/2022
ms.topic: how-to
ms.reviewer: ramacg
---

# Ingest data from OpenTelemetry to Azure Data Explorer

[OpenTelemetry](https://opentelemetry.io/docs/concepts/what-is-opentelemetry/) (OTel) is an open framework for application observability. The instrumentation is hosted by the Cloud Native Computing Foundation (CNCF), which provides standard interfaces for observability data, including [metrics](https://opentelemetry.io/docs/concepts/observability-primer/#reliability--metrics), [logs](https://opentelemetry.io/docs/concepts/observability-primer/#logs), and [traces](https://opentelemetry.io/docs/concepts/observability-primer/#distributed-traces).

The OpenTelemetry exporter supports ingestion of data from many receivers into Azure Data Explorer. In this article, you'll learn how to configure the OTel collector to ingest into Azure Data Explorer, you'll set up the collector to ingest sample data, and then you'll take a quick look at the data that has been ingested.

> [!NOTE]
> The configuration settings are summarized in the [readme documentation](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/exporter/azuredataexplorerexporter/README.md).

## Prerequisites

* An Azure subscription. Create a [free Azure account](https://azure.microsoft.com/free/)
* A cluster and a database: [Quickstart: Create an Azure Data Explorer cluster and database](create-cluster-database-portal.md)

## Supported authentication methods

Azure Active Directory (Azure AD) applications with app keys are supported. To create and register an app in Azure AD, see [Register an application](/azure/active-directory/develop/quickstart-register-app#register-an-application). For information on service principals, see [Application and service principal objects in Azure AD](/azure/active-directory/develop/app-objects-and-service-principals).

> [!NOTE]
> The designated principal must have *Data Ingestor* role-based authorization. For more information, see [role-based authorization](kusto/management/access-control/role-based-authorization.md).

## Create tables

1. Browse to [Azure Data Explorer web UI](https://dataexplorer.azure.com/). 
1. Select **Query** from the left menu. 
1. Expand the target cluster in the left pane.
1. Select the database to give your queries the correct context.

1. Run the following commands are to create tables and schema mapping for the incoming data:

    ```kusto
    .create-merge table <Logs-Table-Name> (Timestamp:datetime, ObservedTimestamp:datetime, TraceId:string, SpanId:string, SeverityText:string, SeverityNumber:int, Body:string, ResourceAttributes:dynamic, LogsAttributes:dynamic) 
    
    .create-merge table <Metrics-Table-Name> (Timestamp:datetime, MetricName:string, MetricType:string, MetricUnit:string, MetricDescription:string, MetricValue:real, Host:string, ResourceAttributes:dynamic,MetricAttributes:dynamic) 
    
    .create-merge table <Traces-Table-Name> (TraceId:string, SpanId:string, ParentId:string, SpanName:string, SpanStatus:string, SpanKind:string, StartTime:datetime, EndTime:datetime, ResourceAttributes:dynamic, TraceAttributes:dynamic, Events:dynamic, Links:dynamic) 
    ```

## Set up streaming or batching ingestion

Azure Data Explorer has two main types of ingestion: batching and streaming. For more information, see [batching vs streaming ingestion](ingest-data-overview.md#batching-vs-streaming-ingestion). The *streaming* method is called *managed* in the OTel collector. Streaming ingestion may be a good choice for you if you need the logs and traces are to be available in near real time. However, streaming ingestion uses more resources than batched ingestion. The OTeL framework itself batches data, which should be considered when choosing which method to use for ingestion.

> [!NOTE]
> [Streaming ingestion](ingest-data-streaming.md) must be enabled on Azure Data Explorer cluster to enable the `managed` option.
> You can check if streaming is enabled using the [.show database streaming ingestion policy](kusto/management/show-database-streaming-ingestion-policy-command.md) command.

Run the following command for each of the three tables to enable streaming ingestion:

```kusto
.alter table <Table-Name> policy streamingingestion enable
```

## Configure the Azure Data Explorer collector

In order to ingest your OpenTelemetry data into Azure Data Explorer, you need [deploy and run](https://opentelemetry.io/docs/collector/deployment/) the OpenTelemetry distribution with the following Azure Data Explorer exporter configuration.

1. Configure the OTel collector using the following fields:

    |Field | Description | Suggested setting|
    |---|---|---|
    | Exporters| Type of exporter | Azure Data Explorer | 
    |  cluster_uri |   Kusto cluster URI where the database and tables will be created |  https:// &lt;cluster>.kusto.windows.net |
    | application_id |  Client ID|  &lt;application id> |
    | application_key| Client secret |  &lt;application key> |
    | tenant_id | Tenant |  &lt;application tenant>|
    | db_name | Database that will receive the logs | oteldb, or other database you have already created
    | metrics_table_name | The target table in the database db_name that stores exported metric data. | OTELMetrics
    | logs_table_name | The target table in the database db_name that stores exported logs data. | OTELLogs
    | traces_table_name | The target table in the database db_name that stores exported traces data. | OTELTraces
    | ingestion_type | Type of ingestion: managed (streaming) or batched | managed
    | otelmetrics_mapping | Optional parameter. Default table mapping is defined during table creation based on OTeL [metrics attributes] (https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/azuredataexplorerexporter#metrics). The default mapping can be changed using this parameter. | &lt;json metrics_table_name mapping>
    | otellogs_mapping | Optional parameter. Default table mapping is defined during table creation based on OTeL [logs attributes] (https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/azuredataexplorerexporter#logs). The default mapping can be changed using this parameter. | &lt;json logs_table_name mapping>
    | oteltraces_mapping | Optional parameter. Default table mapping is defined during table creation based on OTeL [trace attributes] (https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/azuredataexplorerexporter#traces). The default mapping can be changed using this parameter. |&lt;json traces_table_name mapping>
    | logLevel |  | info
    | extensions | Services: extension components to enable | [pprof, zpages, health_check]
    | traces | Services: traces components to enable   |  receivers: [otlp] <br> processors: [batch] <br> exporters: [azuredataexplorer]
    | metrics | Services: metrics components to enable | receivers: [otlp] <br> processors: [batch] <br> exporters: [logging, azuredataexplorer]
    | logs | Services: logs components to enable | receivers: [otlp] <br> processors: [batch] <br> exporters: [ azuredataexplorer]

1. Use the "--config" flag to run the OpenTelemetry collector.

The following is an example configuration for the OTel collector:

```yaml
exporters:
  azuredataexplorer:
    cluster_uri: "https://<cluster>.kusto.windows.net"
    application_id: "<application id>"
    application_key: "<application key>"
    tenant_id: "<application tenant>"
    db_name: "oteldb"
    metrics_table_name: "OTELMetrics"
    logs_table_name: "OTELLogs"
    traces_table_name: "OTELTraces"
    ingestion_type : "managed"
    otelmetrics_mapping : "<json metrics_table_name mapping>"
    otellogs_mapping  : "<json logs_table_name mapping>"
    oteltraces_mapping  : "<json traces_table_name mapping>"
  logging:
    logLevel: info
service:
  extensions: [pprof, zpages, health_check]
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [azuredataexplorer]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [logging, azuredataexplorer]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [ azuredataexplorer]
```

## Collect data with a sample application

Now that the collector is configured, you need to send data to be ingested. In this example. you'll use the sample [spring pet clinic](https://github.com/spring-projects/spring-petclinic) application with the java OTeL collector agent.

1. Download the collector agent here: [Open telemetry collector agent](https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases).

1. To enable open telemetry for the sample application, set the following environment variables in a command line:

    ```command_line
    $env:OTEL_SERVICE_NAME="pet-clinic-service"
    $env:OTEL_TRACES_EXPORTER="otlp"
    $env:OTEL_LOGS_EXPORTER="otlp "                   
    $env:OTEL_EXPORTER_OTLP_ENDPOINT="http://<open-telemetry-collector-host>:4317"
    ```
    
    Alternatively, you can use the following commands in bash shell:

    ```bash
    export OTEL_SERVICE_NAME=pet-clinic-service 
    export OTEL_TRACES_EXPORTER=otlp 
    export OTEL_LOGS_EXPORTER=otlp  
    export OTEL_EXPORTER_OTLP_ENDPOINT=http://<open-telemetry-collector-host>:4317 
    ```

    The open-telemetry-collector-host references the host where ADX OTEL exporter is configured and running.

1. Run the sample spring-boot application with the following command line arguments:
    
    ```
    java -javaagent:./opentelemetry-javaagent.jar -jar spring-petclinic-<version>-SNAPSHOT.jar    
    ```

## Query incoming data

Once the sample app has run, your data has been ingested into the defined tables in Azure Data Explorer. These tables were created in a database that was defined in the OTel collector configuration, as *oteldb*. The tables you've created were defined in the OTel collector configuration. In this example, you've created three tables: *OTELMetrics*, *OTELLogs*, and *OTELTraces*. In this section, you'll query each table separately to get a small selection of the available data.

1. Browse to [Azure Data Explorer web UI](https://dataexplorer.azure.com/). 
1. Select **Query** from the left menu. 
1. Expand the target cluster in the left pane.
1. Select the **oteldb** database to give your queries the correct context.
1. Copy/paste the following queries sequentially, to see an arbitrary number of rows from each table:

    * Metrics
        ```kusto
        OTELMetrics
        |take 2
        ```
    
        You should get results that are similar, but not exactly the same, as the following:
    
        |Timestamp           |MetricName                 |MetricType|MetricUnit|MetricDescription                                                  |MetricValue|Host           |MetricAttributes                                                                                                                                                                                                                   |ResourceAttributes                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
        |--------------------|---------------------------|----------|----------|-------------------------------------------------------------------|-----------|---------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
        |2022-07-01T12:55:33Z|http.server.active_requests|Sum       |requests  |The number of concurrent HTTP requests that are currently in-flight|0          |DESKTOP-SFS7RUQ|{"http.flavor":"1.1", "http.host":"localhost:8080", "scope.name":"io.opentelemetry.tomcat-7.0", "scope.version":"1.14.0-alpha", "http.method":"GET", "http.scheme":"http"}                                                              |{"host.name":"DESKTOP-SFS7RUQ", "process.command_line":"C:\\Program Files\\Java\\jdk-18.0.1.1;bin;java.exe -javaagent:./opentelemetry-javaagent.jar", "process.runtime.description":"Oracle Corporation Java HotSpot(TM) 64-Bit Server VM 18.0.1.1+2-6", "process.runtime.version":"18.0.1.1+2-6", "telemetry.sdk.language":"java", "host.arch":"amd64", "process.runtime.name":"Java(TM) SE Runtime Environment", "telemetry.auto.version":"1.14.0", "telemetry.sdk.name":"opentelemetry", "os.type":"windows", "os.description":"Windows 11 10.0", "process.executable.path":"C:\\Program Files\\Java\\jdk-18.0.1.1;bin;java.exe", "process.pid":5980, "service.name":"my-service", "telemetry.sdk.version":"1.14.0"}|
        |2022-07-01T12:55:33Z|http.server.duration_sum   |Histogram |ms        |The duration of the inbound HTTP request(Sum total of samples)     |114.9881   |DESKTOP-SFS7RUQ|{"http.flavor":"1.1", "http.host":"localhost:8080", "scope.name":"io.opentelemetry.tomcat-7.0", "scope.version":"1.14.0-alpha", "http.method":"GET", "http.scheme":"http", "http.route":"/owners/find", "http.status_code":200}           |{"host.name":"DESKTOP-SFS7RUQ", "process.command_line":"C:\\Program Files\\Java\\jdk-18.0.1.1;bin;java.exe -javaagent:./opentelemetry-javaagent.jar", "process.runtime.description":"Oracle Corporation Java HotSpot(TM) 64-Bit Server VM 18.0.1.1+2-6", "process.runtime.version":"18.0.1.1+2-6", "telemetry.sdk.language":"java", "host.arch":"amd64", "process.runtime.name":"Java(TM) SE Runtime Environment", "telemetry.auto.version":"1.14.0", "telemetry.sdk.name":"opentelemetry", "os.type":"windows", "os.description":"Windows 11 10.0", "process.executable.path":"C:\\Program Files\\Java\\jdk-18.0.1.1;bin;java.exe", "process.pid":5980, "service.name":"my-service", "telemetry.sdk.version":"1.14.0"}|

    * Logs

        ```kusto
        OTELLogs
        |take 2
        ```

        You should get results that are similar, but not exactly the same, as the following:
 
        |Timestamp           |TraceId|SpanId|SeverityText|SeverityNumber|Body                                                                                                                                                                                                                                                                      |ResourceAttributes                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |LogsAttributes                                                             |
          |--------------------|-------|------|------------|--------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------|
          |2022-07-01T13:00:39Z|       |      |INFO        |9             |Starting PetClinicApplication v2.7.0-SNAPSHOT using Java 18.0.1.1 on DESKTOP-SFS7RUQ with PID 37280 (C:\Users\adxuser\Documents\Repos\spring-petclinic\target\spring-petclinic-2.7.0-SNAPSHOT.jar started by adxuser in C:\Users\adxuser\Documents\Repos\spring-petclinic)|{"host.name":"DESKTOP-SFS7RUQ", "process.executable.path":"C:\\Program Files\\Java\\jdk-18.0.1.1;bin;java.exe", "process.pid":37280, "process.runtime.description":"Oracle Corporation Java HotSpot(TM) 64-Bit Server VM 18.0.1.1+2-6", "process.runtime.name":"Java(TM) SE Runtime Environment", "telemetry.sdk.name":"opentelemetry", "os.type":"windows", "process.runtime.version":"18.0.1.1+2-6", "telemetry.sdk.language":"java", "telemetry.sdk.version":"1.14.0", "process.command_line":"C:\\Program Files\\Java\\jdk-18.0.1.1;bin;java.exe -javaagent:./opentelemetry-javaagent.jar", "os.description":"Windows 11 10.0", "service.name":"my-service", "telemetry.auto.version":"1.14.0", "host.arch":"amd64"}|{"scope.name":"org.springframework.samples.petclinic.PetClinicApplication"}|
          |2022-07-01T13:00:39Z|       |      |INFO        |9             |No active profile set, falling back to 1 default profile: "default"                                                                                                                                                                                                       |{"host.name":"DESKTOP-SFS7RUQ", "process.executable.path":"C:\\Program Files\\Java\\jdk-18.0.1.1;bin;java.exe", "process.pid":37280, "process.runtime.description":"Oracle Corporation Java HotSpot(TM) 64-Bit Server VM 18.0.1.1+2-6", "process.runtime.name":"Java(TM) SE Runtime Environment", "telemetry.sdk.name":"opentelemetry", "os.type":"windows", "process.runtime.version":"18.0.1.1+2-6", "telemetry.sdk.language":"java", "telemetry.sdk.version":"1.14.0", "process.command_line":"C:\\Program Files\\Java\\jdk-18.0.1.1;bin;java.exe -javaagent:./opentelemetry-javaagent.jar", "os.description":"Windows 11 10.0", "service.name":"my-service", "telemetry.auto.version":"1.14.0", "host.arch":"amd64"}|{"scope.name":"org.springframework.samples.petclinic.PetClinicApplication"}|


    * Traces

        ```kusto
        OTELTraces
        |take 2
        ```
    
        You should get results that are similar, but not exactly the same, as the following:


       |TraceId             |SpanId          |ParentId|SpanName|SpanStatus|SpanKind                                                                                                                                                                                                                                                                  |StartTime                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |EndTime                                                                    |ResourceAttributes                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |TraceAttributes                                                                                                                                                                                                                                                                 |Events|Links|
      |--------------------|----------------|--------|--------|----------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------|-----|
      |573c0e4e002a9f7281f6d63eafe4ef87|dab70d0ba8902c5e|        |87d003d6-02c1-4f3d-8972-683243c35642|STATUS_CODE_UNSET|SPAN_KIND_CLIENT                                                                                                                                                                                                                                                          |2022-07-01T13:17:59Z                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |2022-07-01T13:17:59Z                                                       |{"telemetry.auto.version":"1.14.0", "os.description":"Windows 11 10.0", "process.executable.path":"C:\\Program Files\\Java\\jdk-18.0.1.1;bin;java.exe", "process.runtime.description":"Oracle Corporation Java HotSpot(TM) 64-Bit Server VM 18.0.1.1+2-6", "service.name":"my-service", "process.runtime.name":"Java(TM) SE Runtime Environment", "telemetry.sdk.language":"java", "telemetry.sdk.name":"opentelemetry", "host.arch":"amd64", "host.name":"DESKTOP-SFS7RUQ", "process.pid":34316, "process.runtime.version":"18.0.1.1+2-6", "os.type":"windows", "process.command_line":"C:\\Program Files\\Java\\jdk-18.0.1.1;bin;java.exe -javaagent:./opentelemetry-javaagent.jar", "telemetry.sdk.version":"1.14.0"}|{"db.user":"sa", "thread.id":1, "db.name":"87d003d6-02c1-4f3d-8972-683243c35642", "thread.name":"main", "db.system":"h2", "scope.name":"io.opentelemetry.jdbc", "scope.version":"1.14.0-alpha", "db.connection_string":"h2:mem:", "db.statement":"DROP TABLE vet_specialties IF EXISTS"}|[]    |[]   |
      |84a9a8c4009d91476da02dfa40746c13|3cd4c0e91717969a|        |87d003d6-02c1-4f3d-8972-683243c35642|STATUS_CODE_UNSET|SPAN_KIND_CLIENT                                                                                                                                                                                                                                                          |2022-07-01T13:17:59Z                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |2022-07-01T13:17:59Z                                                       |{"telemetry.auto.version":"1.14.0", "os.description":"Windows 11 10.0", "process.executable.path":"C:\\Program Files\\Java\\jdk-18.0.1.1;bin;java.exe", "process.runtime.description":"Oracle Corporation Java HotSpot(TM) 64-Bit Server VM 18.0.1.1+2-6", "service.name":"my-service", "process.runtime.name":"Java(TM) SE Runtime Environment", "telemetry.sdk.language":"java", "telemetry.sdk.name":"opentelemetry", "host.arch":"amd64", "host.name":"DESKTOP-SFS7RUQ", "process.pid":34316, "process.runtime.version":"18.0.1.1+2-6", "os.type":"windows", "process.command_line":"C:\\Program Files\\Java\\jdk-18.0.1.1;bin;java.exe -javaagent:./opentelemetry-javaagent.jar", "telemetry.sdk.version":"1.14.0"}|{"db.user":"sa", "thread.id":1, "db.name":"87d003d6-02c1-4f3d-8972-683243c35642", "thread.name":"main", "db.system":"h2", "scope.name":"io.opentelemetry.jdbc", "scope.version":"1.14.0-alpha", "db.connection_string":"h2:mem:", "db.statement":"DROP TABLE vets IF EXISTS"}           |[]    |[]   |


## Further data processing

Using update policies, the collected data can further be processed as per application need. For more information, see [Update policy overview](kusto/management/updatepolicy.md).

1. The following example exports histogram metrics to a histo-specific table with buckets and aggregates. Run the following command in the query pane of the Azure Data Explorer web UI:
    
    ```kusto
    .create table HistoBucketData (Timestamp: datetime, MetricName: string , MetricType: string , Value: double, LE: double, Host: string , ResourceAttributes: dynamic, MetricAttributes: dynamic )
    
    .create function 
    with ( docstring = "Histo bucket processing function", folder = "UpdatePolicyFunctions") ExtractHistoColumns()
    {
        OTELMetrics
        | where MetricType == 'Histogram' and MetricName has "_bucket"
        | extend f=parse_json(MetricAttributes)
        | extend le=todouble(f.le)
        | extend M_name=replace_string(MetricName, '_bucket','')
        | project Timestamp, MetricName=M_name, MetricType, MetricValue, LE=le, Host, ResourceAttributes, MetricAttributes
    }
    
    .alter table HistoBucketData policy update 
    @'[{ "IsEnabled": true, "Source": "OTELMetrics","Query": "ExtractHistoColumns()", "IsTransactional": false, "PropagateIngestionProperties": false}]'
    ```

1. The following commands create a table that only contains count and sum values of Histogram metric type and attaches an update policy. Run the following command in the query pane of the Azure Data Explorer web UI:

    ```kusto
     .create table HistoData (Timestamp: datetime, MetricName: string , MetricType: string , Count: double, Sum: double, Host: string , ResourceAttributes: dynamic, MetricAttributes: dynamic)
    
     .create function 
    with ( docstring = "Histo sum count processing function", folder = "UpdatePolicyFunctions") ExtractHistoCountColumns()
    {
       OTELMetrics
        | where MetricType =='Histogram'
        | where MetricName has "_count"
        | extend Count=MetricValue
        | extend M_name=replace_string(MetricName, '_bucket','')
        | join kind=inner (OTELMetrics
        | where MetricType =='Histogram'
        | where MetricName has "_sum"
        | project Sum = MetricValue , Timestamp)
     on Timestamp | project Timestamp, MetricName=M_name, MetricType, Count, Sum, Host, ResourceAttributes, MetricAttributes
    }
    
    .alter table HistoData policy update 
    @'[{ "IsEnabled": true, "Source": "RawMetricsData","Query": "ExtractHistoCountColumns()", "IsTransactional": false, "PropagateInge
    ```


## Next steps

* [KQL quick reference](kql-quick-reference.md)
* [Kusto Query Language (KQL) overview](kusto/query/index.md)