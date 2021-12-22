##Environmental Review – Sandbox/Dev/UAT – FFE
#####APIM Policies – Reviewed APIM Policies, several changes recommended:
1. Use a backing Logic App when connecting to the Event Grid for better orchestration of events
2. Use a backing Logic App when also processing anything related to Azure Service Bus
3. For the Storage API, consider using a Function App with a Docker Container as opposed to having logic built into the APIM Policy Directly
4. Use the built in Azure Caching capabilities when possible for caching of auth tokens
#####Logic Apps – Use more Logic Apps where possible 
1. Low code environment with excellent telemetry and orchestration (start/stop/retry/success and fail built in)
2. Use Logic Apps when processing JSON messages primarily
3. Avoid Logic Apps when processing large data sets, database inline queries (non-stored procedure calls) and many to one relationships.
#####Azure Data Factory – Use more ADF where possible
1. Processing large amounts of database retrieval type data, such as Oracle/SQL, etc
2. Processing and transforming subsets of data, CSV files, XML, JSON to database or storage
3. Loading data from multiple systems in a single ADF pipeline 
#####Azure Monitor


#####Vnet/Subnet – Mostly Information Gathering
*[10/12/2021 8:15 AM]* Sean Joseph
I reviewed the Vnet/Subnet setup. Did not see any issues, but I realized I really don't have the context as to what you are trying to achieve.
Is it just to spin up some VM's or are there plans to expose a public endpoint and allow internet traffic eventually? As I did not see any public IP addresses assigned.

*[10/12/2021 8:19 AM]* Sean Joseph
Actually I take that back, I did not see the bastion-FFE-Dev

*[10/12/2021 8:22 AM]* Sean Joseph
With that said, I see the vnet's applied to the Linux VM's. Is this the only use case, any others?

*[10/12/2021 8:24 AM]* Sean Joseph
Also I don't have access to some of the setup items, like On-Prem-Net-Connect-vnet used for peering

*[10/12/2021 8:30 AM]* Sean Joseph
Lastly was most of this setup for connecting on premise databases, like Oracle/Maximo? Or are there other network resources that were needed? I don't see any usage of on-premise data gateways setup in Azure which can do a lot of the data gateway connectivity. Again without knowing the reasons for this setup, my feedback is limited at the moment.
1. I'm not much aware of the network setup in IMS Development subscription, but from what I can recollect/understand, we don't want to expose any of the Azure resources over the public internet. Those resources (app services, event grids, function apps etc.) need to be accessible only over the ABS network and some of those need to be accessible from the PTC cloud instance. 
2. *On-Prem-Net-Connect-vnet* access permissions - I don't think any of us have access to such resources. Those are configured by the security team and not accessible to anyone. We can talk to Arunava and request access if you need to dive deeper into its details. 
3. We don't connect to any of the on-prem databases as per my knowledge. We only connect to the on-prem systems - IBM Maximo, GEMS etc through REST APIs. 
4. I believe the long term goal is to setup an Application Gateway perhaps, to allow traffic from the public internet to the APIM so that the services in the APIM can be consumed by the external clients/customers who would like to subscribe to those services.

**Feedback:**
From the statements gathered, it seems to be that the virtual networks and usage of Azure are more of a private cloud that encapsulates all endpoints to on prem traffic. There are many use cases to allow public traffic to resources so I would need to understand why this is not being allowed. Using Azure to build a custom private cloud is not a best practice. Instead, it would be good to use something like Azure Stack:
Azure Stack | Microsoft Azure (https://azure.microsoft.com/en-us/overview/azure-stack/#industries)

Node.js based Function Apps –
1. The only comment to the usage of Node.js is that there is some loss of ORM mapping capabilities that are natural via EF Core in .net (to downstream databases such as Oracle, SQL). In the current environment since most calls are to Restful API’s this may be sufficient, but future use cases may need to make usage of ORM and I would suggest to make the Function Apps, .net/C# based for those use cases. EF Core maps all database objects to POCO classes in C# and is easily portable.
Azure DevOps –
1. Have not currently reviewed, still under consideration.

Managed Identity vs Service Principal:
1. For the secured resources, Managed Identities have been selected. A brief write up is included here - Demystifying Service Principals - Managed Identities - Azure DevOps Blog (https://devblogs.microsoft.com/devops/demystifying-service-principals-managed-identities/)
2. From a management standpoint, Managed Identities incur some overhead in the fact that individual apps/resources need to be configured, but has advantages over SP’s (service principals) in the fact that configuration details need not be stored in each individual app.
3. In the current environment, Managed Identifies should be satisfactory for the use cases provided.

Azure B2C – Not currently reviewed

Azure Monitor, Log Analytics, and Application Insights

Out of the box Azure Monitor App features –
https://docs.microsoft.com/en-us/azure/azure-monitor/monitor-reference

![Azure Monitor](azure_monitor.jpg)

*For custom telemetry or logging, applications (even outside of Azure, can utilize the Azure Monitor Data Collector API)*

You can use the HTTP Data Collector API to send log data to a Log Analytics workspace in Azure Monitor from any client that can call a REST API. The client might be a runbook in Azure Automation that collects management data from Azure or another cloud, or it might be an alternative management system that uses Azure Monitor to consolidate and analyze log data.

All data in the Log Analytics workspace is stored as a record with a particular record type. You format your data to send to the HTTP Data Collector API as multiple records in JavaScript Object Notation (JSON). When you submit the data, an individual record is created in the repository for each record in the request payload.

![Azure Monitor](azure_monitor2.jpg)

https://docs.microsoft.com/en-us/azure/api-management/api-management-log-to-eventhub-sample#a-policy-to-send-applicationhttp-messages
___________________________

#Use Event Hubs for heavily trafficked API’s in APIM 

###Why send to an Azure Event Hub? 
It is a reasonable to ask, why create a policy that is specific to Azure Event Hubs? There are many different places where I might want to log my requests. Why not just send the requests directly to the final destination? That is an option. However, when making logging requests from an API management service, it is necessary to consider how logging messages impact the performance of the API. Gradual increases in load can be handled by increasing available instances of system components or by taking advantage of geo-replication. However, short spikes in traffic can cause requests to be delayed if requests to logging infrastructure start to slow under load. 

The Azure Event Hubs is designed to ingress huge volumes of data, with capacity for dealing with a far higher number of events than the number of HTTP requests most APIs process. The Event Hub acts as a kind of sophisticated buffer between your API management service and the infrastructure that stores and processes the messages. This ensures that your API performance will not suffer due to the logging infrastructure.

Once the data has been passed to an Event Hub, it is persisted and will wait for Event Hub consumers to process it. The Event Hub does not care how it is processed, it just cares about making sure the message will be successfully delivered. 

Event Hubs has the ability to stream events to multiple consumer groups. This allows events to be processed by different systems. This enables supporting many integration scenarios without putting addition delays on the processing of the API request within the API Management service as only one event needs to be generated. 

==**We will utilize Log Analytics to capture data from Logic App runs where appropriate (Logic App must turn on setting to allow Log Analytics to capture data):**==
1. Real Time Logic Apps that are dependent on multiple work flows or streams such as Event Grid events.
2. Long running Logic Apps (more than 1-3 minutes).
3. Logic Apps that make usage of Function Apps or APIM api's.

**For detailed tracking (including tracking data across multiple Logic App workflows):**
Utilize clientTrackingId: If not provided, Azure automatically generates this ID and correlates events across a logic app run, including any nested workflows that are called from the logic app. You can manually specify this ID in a trigger by passing a x-ms-client-tracking-id header with your custom ID value in the trigger request. You can use a request trigger, HTTP trigger, or webhook trigger.

**For correlation of output of an action:**
Utilize trackedProperties: To track inputs or outputs in diagnostics data, you can add a trackedProperties section to an action either by using the Logic App Designer or directly in your logic app's JSON definition. Tracked properties can track only a single action's inputs and outputs, but you can use the correlation properties of events to correlate across actions in a run. To track more than one property, one or more properties, add the trackedProperties section and the properties that you want to the action definition.

*Custom Query Creation using Kusto:*
https://docs.microsoft.com/en-us/azure/logic-apps/create-monitoring-tracking-queries
https://docs.microsoft.com/en-us/azure/data-explorer/kusto/query/ 

==**We will utilize Application Insights for Function App monitoring where appropriate:**==
Azure Functions offers built-in integration with Application Insights, which is available through the ILogger Interface. Below is the list of currently supported features, of which we will make use of the automatic collection capabilities.


![Azure Monitor](azure_monitor3.jpg)

https://docs.microsoft.com/en-us/azure/azure-functions/functions-monitoring?tabs=cmd#enable-application-insights-integration

For a function app to send data to Application Insights, it needs to know the instrumentation key of an Application Insights resource. The key must be in an app setting named *APPINSIGHTS_INSTRUMENTATIONKEY*.

When you create your function app in the Azure portal, from the command line by using Azure Functions Core Tools, or by using Visual Studio Code, Application Insights integration is enabled by default. The Application Insights resource has the same name as your function app, and it's created either in the same region or in the nearest region.
https://docs.microsoft.com/en-us/azure/azure-functions/functions-monitoring?tabs=cmd#query-telemetry-data
Application Insights Analytics gives you access to all telemetry data in the form of tables in a database. Analytics provides a query language for extracting, manipulating, and visualizing the data. 

==**We will utilize the Azure Monitor capabilities for API Management:**==
https://docs.microsoft.com/en-us/azure/api-management/api-management-howto-use-azure-monitor

<ins>Metrics</ins>
API Management emits metrics every minute, giving you near real-time visibility into the state and health of your APIs via two of the most used metrics:

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**Capacity**: helps you make decisions about upgrading/downgrading your APIM services. The metric is emitted per minute and reflects the gateway capacity at the time of reporting.
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**Requests**: helps you to analyze API traffic going through your APIM services.

<ins>Activity Logs</ins>
Activity logs provide insight into the operations that were performed on your API Management services. Using activity logs, you can determine the "what, who, and when" for any write operations (PUT, POST, DELETE) taken on your API Management services.

<ins>Resource Logs</ins>
Resource logs provide rich information about operations and errors that are important for auditing as well as troubleshooting purposes. 

##Application Insights Logging from APIM Performance implications and log sampling

<span style="color:red">Warning</span>
Logging all events may have serious performance implications, depending on incoming requests rate. 

Based on internal load tests, enabling this feature caused a 40%-50% reduction in throughput when request rate exceeded 1,000 requests per second. Application Insights is designed to use statistical analysis for assessing application performances. It is not
intended to be an audit system and is not suited for logging each individual request for high-volume APIs. 

You can manipulate the number of requests being logged by adjusting the **Sampling** setting (see the preceding steps). A value of 100% means all requests are logged, while 0% reflects no logging. **Sampling** helps to reduce volume of telemetry, effectively preventing significant performance degradation, while still carrying the benefits of logging. 

Skipping logging of headers and body of requests and responses will also have positive impact on alleviating performance issues. 

Consolidate App Insights and Log Analytics Workspaces in Environment: 

Multiple App Insights creations due to default settings in app service and function apps. Make sure the team uses ONE standard App Insights instance. 
![Azure Monitor](azure_monitor4.jpg)

Consolidate into one consistent Log Analytics workspace.
![Azure Monitor](azure_monitor5.jpg)