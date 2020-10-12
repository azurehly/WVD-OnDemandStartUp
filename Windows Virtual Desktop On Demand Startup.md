# Windows Virtual Desktop On Demand Startup

Microsoft [Windows Virtual Desktop](https://docs.microsoft.com/en-us/azure/virtual-desktop/overview) is a desktop and app virtualization service that runs on the Azure cloud. It can offer a multi-session Windows 10 with scalability, or virtualized Microsoft 365 Apps for enterprise with unified management experience which brings existing Remote Desktop Services, Windows Server desktops and app to any computer. 

Optimizing resource consumption and saving cost is always a hot topic for lots of business. In Evan Yi's post [Save Azure WVD Cost with Logical App](https://blog.evanyi.net/2020/10/another-way-to-save-azure-wvd-cost-with.html), the solution can shut down WVDs with no user's active session by leveraging [Azure Logical App](https://docs.microsoft.com/en-us/azure/logic-apps/logic-apps-overview) and [Azure Log Analytics](https://www.dummies.com/programming/how-to-use-azure-log-analytics/). When users want to connect during the power off period without seeking help from IT nor allocating always-on WVD host, there is no out of box solution yet from Microsoft. However, this can be achieved by looking WVD connection attempt from Log Analytics query then trigger VM start up action via Azure Logic App.

## Scenario
During off-hours user attempt to connect to WVD VMs, the connection will failure and generate error logs: ‘cannot find any SessionHost available in specified pool’ which trigger the action of powering on the last WVD VM user interacted with. 

Here's a solution to that appears to work:
## Prerequisites

WVD Diagnostic logs shipping to Log Analytics

WVD workload Perf logs shipping to Log Analytics


## Define the scheduler of Logical App to run
Alternatively, the trigger can be a Webhook request or from Azure Monitor Alter. However, it is found that there is a noticeable delay when triggered from Alert. 

## Query WVD for session host availability
This is the sample code for querying user connection attempt (5 minutes ago) and the last WVD they connected. (We filter computer name with ‘wvd’, it can be changed based on resource group)

*KQL Query*
```
WVDErrors
| where TimeGenerated >= ago(5m)
| where Type == "WVDErrors"
| where Message contains "Could not find any SessionHost available in specified pool"
| summarize arg_max(TimeGenerated, *) by UserName
| extend CsvUsr = split(UserName, '@')
| extend User = tostring(CsvUsr[0])
| project User
| join (VMProcess
    | where Computer contains "wvd"
    | summarize arg_max(TimeGenerated, *) by UserName
    | project User=UserName, _ResourceId)
    on User
| extend CsvField = split(_ResourceId, '/')
| extend VMList = tostring(CsvField[8])
| extend ResourceGroup = tostring(CsvField[4])
| project VMList, ResourceGroup, User
```

## Logic App Action
Here is the work flow of Logic App, the sample deployment is scheduled to run every 5 minutes in weekend.

<img src="https://github.com/azurehly/WVD-OnDemandStartUp/blob/main/wvdondemand-flow.png"/>


## Click on button to deploy

<img src="https://aka.ms/deploytoazurebutton"/>
