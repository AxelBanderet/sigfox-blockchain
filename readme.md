# A Sigfox IoT Blockchain application with Microsoft Azure 

## Introduction
Sigfox 0G network allows to collect unprecedent amount of data from wide types of unreachable sources so far. However, it appears that this huge amount of information does not reveal a related worth value unless it has been processed in a proper way. Once correctly handled, it becomes so interesting to observe how this data can then bring an unprecedent amount of value. This is the power of IoT and this is also why Sigfox-based successfull value propositions are never only about collecting the data but more importantly about extracting and creating value from it.

On the other hand, Blockchain has emerged in the past decades as one of the most useful and democratized technology. So many applications in multitude of markets are now adopting it. As a recap, a Blockchain is a serie of blocks of information linked by cryptographic hashes which are distributed on a network of nodes. Each of them has a local copy of the blockchain and verify the new blocks by reaching an agreed consensus. Hence, the Blockchain becomes a public distributed ledger of information which is inherently immutable, transparent and secure. This is how it defines a new way of storing and exchanging the information and certainly explains why this technology has been and is still the pillar of cryptocurrencies. 

But what if we were storing IoT data in a Blockchain?  

Following this growth potential, several edge tehnologies have been developed on top of the Blockchain. Smart Contracts are one of those. The concept is basically to pre-define some logical rules and actions that will be triggered depending on the data processed within the Blockchain. Among others, it allows to use as an input the Blockchain data and to define modular, automatic, instantaneous and administrative-free actions based on it. 

We are now able to draw the goal of this article: 
Deploy a Blockchain based application fed by a Sigfox IoT device wich triggers Smart Contract related actions.   

## Architecture

The global flow overview is the following:
![Image](img/GlobalArchitectureOverview.png)

### 1. Writing to Microsoft Azure

![Image](img/WriteIntoAzure.png)

Sigfox devices send some data over Sigfox 0G network. Then the Sigfox Cloud pushes it through a callback up to Microsoft Azure IoT Hub.

### 2. Prepare the data

![Image](img/PrepareForBlockchainIngestion.png)

Then, a suite of Azure services are being sollicited. First we use a Function App to parse the data into a Temperature and Humidity decimal values. 

It is then collected in a Service Bus that is responsible of routing it up to a Logic App. 

This Logic App has a simple objective: formatting the previously mentionned data into something that can be ingested by Azure Workbench Blockchain

### 3. Publish into the Blockchain

![Image](img/PublishIntoBlockchain.png)

Two consumer services are listening for incoming messages into the previous service bus. The first one is a Database Consumer that will automatically push event informations into a simple SQL database.

The second one is a Data Ledger Technology Consumer responsible of forwarding the metadata for transactions to be written to the blockchain. Then the Transaction Builder & Signer assembles the blockchain transaction based on the related input data. Once assembled, the transaction is signed and delivered to Azure Blockchain Service through a specific router. Private keys are stored in Azure Key Vault.

### 4. Interact from WebApps

![Image](img/ReadFromWeb.png)

Azure Workbench Blockchain provides plug and play interaction tools such as a Client web app and a Smartphone app. They are connected to an Azure Active Directory for users and roles management. 

These web-services interact with a REST-based gatewayservice API. When writing to a blockchain, the API generates and delivers messages to an event broker. When data is requested by the API, queries are sent to the SQL database. This storage contains a replica of overall data and metadata that provides configuration and context information for the related smart contracts. Queries return the requested data from this "off-chain" replica in a format which has been specified by the smart contract.

## Demo

For this demo, we will be using a ready-to-use Smart Contract provided by Microsoft. It is linked to an "IoT Refrigerated Transportation use-case". I would strongly advise to read the documentation available [here](https://github.com/Azure-Samples/blockchain/blob/master/blockchain-workbench/application-and-smart-contract-samples/refrigerated-transportation/readme.md) before going further. Here is the summary overview: 

**«** *The refrigerated transportation smart contract covers a provenance scenario with IoT monitoring. You can think of it as a supply chain transport scenario where certain compliance rules must be met throughout the duration of the transportation process. The initiating counterparty specifies the humidity and temperature range the measurement must fall in to be compliant. At any point, if the device takes a temperature or humidity measurement that is out of range, the contract state will be updated to indicate that it is out of compliance.* **»** 

### Interfacing Azure with the Sigfox Cloud

The first step is to configure the Sigfox backend to push your device data up to an Azure IoT Hub.

It is about configuring a Sigfox Backend Azure IoT Hub callback to push the data generated by the Sens'it up to an Azure IoT Hub 
instance.
A great tutorial regarding this Sigfox data ingestion in Azure is available [here](https://medium.com/@nicolas.farolfi_48489/how-to-use-sigfox-with-microsoft-azure-c6ab6e1d1708).
All credit to [Nicolas Farolfi](https://medium.com/@nicolas.farolfi_48489) for this tutorial. 

### Blockchain set-up

#### 1. Deploy Azure Workbench Blockchain

Go to [Azure Workbench Blockchain > Deploy](https://docs.microsoft.com/en-gb/azure/blockchain/workbench/deploy) and follow the documentation.

#### 2. Active Directory configuration: User management

The external ID of the User table is used to hold the id of the device.

Add a user to Azure Blockchain Workbench that will represent your device using this [documentation](https://docs.microsoft.com/en-gb/azure/blockchain/workbench/manage-users).

Identify the device ID for your device that will be sent with the telemetry message.

In the query window, enter and execute the following SQL

Update [User] Set External Id = ‘ < your device id here >’ where EmailAddress = ‘< insert email address here >’

After creating a user to representing my Sens'it, I end up with this Active Directory user list:

![Image](img/ActiveDirectoryUsers.png)


### Data formatting for Blockhain

#### 1. Service Bus

Create a standard Service Bus and a Queue within this bus.

#### 2. Function App: Parsing the raw data

In this second step, we first need to configure a Function App to parse the data from hexadecimal caracters into understandable data such as Temperature and Humidity decimal values. 

Note that once done, instead of pushing the data into an Event Hub we will chose instead to output it in a Service Bus.
Also, in the interest of simplification, we will need to "Round" those decimal parsed values.

Create a *SensitV3Parser* Function App

Here below is an example of a parser written in Javascript. 

```javascript
module.exports = function (context, IoTHubMessages) {
    context.log(`JavaScript eventhub trigger function called for message array: ${IoTHubMessages}`);
    
    IoTHubMessages.forEach(message => {
        var payload,
            temperature,
            humidity,
            battery;

        payload = message.data;

        //temperture----------------------------------------
        // Calculate from payload
        var pay = payload.slice(3,6);
        var convertOfPayload = parseInt(pay,16);
        var mask = parseInt("001111111111",2);
        var x = convertOfPayload & mask;
        temperature = (x - 200) / 8;


        //humidity--------------------------------------------
        // Calculate from payload
        pay = payload.slice(6,8);
        convertOfPayload = parseInt(pay,16);
        var humidity = convertOfPayload / 2;

        //battery--------------------------------------------
        //Calculate from payload
        pay = payload.slice(0,2);
        convertOfPayload = parseInt(pay,16);
        mask = parseInt("11111000",2);
        x = convertOfPayload & mask;
        var battery = (x * 0.05) + 2.7;

        // create a well-formed object, to use in time series
        var obj = new Object();
        // date manipulation
        var today = new Date();

        obj.deviceId = message.device;
        obj.time = today.toISOString();
        obj.temperature = Math.round(temperature);
        obj.humidity = Math.round(humidity);
        obj.battery = battery;

        context.log(`Processed message: ${message}`);
        context.bindings.outputSbMsg = [];
        context.bindings.outputSbMsg.push(obj);

        context.log(JSON.stringify(context.bindings.outputSbMsg));
    });

    context.done();
};
```
From the integration tab, it is required to configure both inputs and outputs of your parser. Chose the queue previously created as an output of your FunctionApp

![Image](img/FunctionAppConfig.png)

The next step is about being able to deliver the previous parsed data up to [Azure Blockchain Workbench](https://azure.microsoft.com/en-gb/features/blockchain-workbench/). 

A great tutorial released by Microsoft explains a way of doing so. It is available [here](https://github.com/Azure-Samples/blockchain/blob/master/blockchain-workbench/iot-integration-samples/ConfigureIoTDemo.md).
However, as before it needs to be adapted. 

#### 3. Deploy the stored procedures

Prior to adding the newly-created message onto the Workbench Service Bus, the Logic App first makes a SQL call to the GetContractInstanceInfoForDeviceId Stored Procedure in the Workbench db.

This stored procedure takes a DeviceID as an input and returns the data for the specific contract instance that the device has been added to in Workbench with the "Device" role. (Each device can only be mapped to a single instance with this role.)

Once this message has been placed onto the Azure Blockchain Workbench Service Bus, it is picked up by Workbench and the appropriate actions(s) are taken to execute the request on the blockchain.

To load these stored procedures, go to the SQL database that has been deployed along with your Azure Workbench Blockchain instance, select the *Query Editor (preview)* and log in.

Download the sql procedure file [here](https://github.com/Azure-Samples/blockchain/blob/master/blockchain-workbench/iot-integration-samples/SQL/IoTSprocs.sql) and save it locally.

Finally you need to load it in Azure by selecting *Open query* and run it.

#### 5. Logic App

It is composed of 9 steps:

![Image](img/LogicAppSteps.png)

* Step 1

The logic app will scan the queue for new incoming messages every second. 

![Image](img/LogicAppConfigStep1.png)

* Step 2

Select **Parse JSON** action. The content needs to be converted from Base64 to String. Click in the content field, select Expression and enter the following: ```json(base64ToString(triggerBody()?['ContentData']))```

For the schema property, we need to enter the one corresponding to the Sens'it in Temperature & Humidity mode:

```json
{
    "properties": {
        "battery": {
            "type": "number"
        },
        "deviceId": {
            "type": "string"
        },
        "humidity": {
            "type": "number"
        },
        "temperature": {
            "type": "number"
        },
        "time": {
            "type": "string"
        }
    },
    "type": "object"
}
```

![Image](img/LogicAppConfigStep2.png)

* Step 3

Select **SQL Stored Procedure**. Configure the connection to the corresponding Azure Workbench SQL Database and then chose the stored procedure named *GetContractInfoForDeviceID*.

Using Dynamic Properties, select *deviceId*

![Image](img/LogicAppConfigStep3.png)

* Step 4

Select **Parse JSON** action. Click in the *Content field* and then select the *ResultSets* item for content. The schema is here below:

```json
{
    "properties": {
        "Table1": {
            "items": {
                "properties": {
                    "ConnectionId": {
                        "type": "number"
                    },
                    "ContractCodeBlobStorageUrl": {
                        "type": "string"
                    },
                    "ContractId": {
                        "type": "number"
                    },
                    "ContractLedgerIdentifier": {
                        "type": "string"
                    },
                    "IngestTelemetry_ContractPersonaID": {},
                    "IngestTelemetry_ContractWorkflowFunctionID": {
                        "type": "number"
                    },
                    "IngestTelemetry_Humidity_WorkflowFunctionParameterID": {
                        "type": "number"
                    },
                    "IngestTelemetry_Temperature_WorkflowFunctionParameterID": {
                        "type": "number"
                    },
                    "IngestTelemetry_Timestamp_WorkflowFunctionParameterID": {
                        "type": "number"
                    },
                    "UserChainIdentifier": {
                        "type": "string"
                    },
                    "WorkflowFunctionId": {
                        "type": "number"
                    },
                    "WorkflowFunctionName": {
                        "type": "string"
                    },
                    "WorkflowName": {
                        "type": "string"
                    }
                },
                "required": [
                    "ContractId",
                    "WorkflowFunctionId",
                    "ConnectionId",
                    "ContractLedgerIdentifier",
                    "ContractCodeBlobStorageUrl",
                    "UserChainIdentifier",
                    "WorkflowFunctionName",
                    "WorkflowName",
                    "IngestTelemetry_ContractWorkflowFunctionID",
                    "IngestTelemetry_ContractPersonaID",
                    "IngestTelemetry_Humidity_WorkflowFunctionParameterID",
                    "IngestTelemetry_Temperature_WorkflowFunctionParameterID",
                    "IngestTelemetry_Timestamp_WorkflowFunctionParameterID"
                ],
                "type": "object"
            },
            "type": "array"
        }
    },
    "type": "object"
}
```

![Image](img/LogicAppConfigStep4.png)

* Step 5

Select **Initialize variable** action. This new variable will be used to specify a unique id for the request. This can be useful for monitoring purposes later on.

Set the Name property to *RequestId*.

Set the Type property to *String*.

In the Value property, use the Dynamic Content window, select Expression and enter: ```guid()```

![Image](img/LogicAppConfigStep5.png)

* Step 6

Select **Initialize variable** action. This new variable will be used to define the Unix Epoch time to include as the timestamp for every new message.

Set the Name property to *TicksNow*.

Set the Type property to *Integer*.

Click on the Value property field and then in the Dynamic Content window enter the following in the Expression tab: ```ticks(utcNow())``` 

![Image](img/LogicAppConfigStep6.png)

* Step 7

Select **Initialize variable** action. This variable will be used for defining the Unix Epoch time as well.

Set the Name property to *TicksTo1970*.

Set the Type property to *Integer*.

Click on the Value property field and then in the Dynamic Content window enter the following in the Expression tab: ```ticks('1970-01-01')``` 

![Image](img/LogicAppConfigStep7.png)

* Step 8

Select **Initialize variable** action.

Set the Name property to *Timestamp*.

Set the Type property to *Integer*.

Click on the Value property field and then in the Dynamic Content window enter the following in the Expression tab: ```div(sub(variables('TicksNow'),variables('TicksTo1970')),10000000)``` 

![Image](img/LogicAppConfigStep8.png)

* Step 9

Select **For each** action.

In the field called "Select an output from previous steps property", select *Table1* from Dynamic Content list. 

Then configure the action to send a message into your Worbench Blockchain associated input service bus. Configure the connection relatively:
Set the "ueue/Topic name" to *ingressQueue*.

Set the "SessionId" to *RequestId*.

Then the Content property can be filled with the template below:

```
{
    "requestId": "@{variables('RequestId')}",
    "userChainIdentifier": "@{items('For_each')?['UserChainIdentifier']}",
    "contractLedgerIdentifier": "@{items('For_each')?['ContractLedgerIdentifier']}",
    "workflowFunctionName": "IngestTelemetry",
    "Parameters": [
        {
            "name": "humidity",
            "value": "@{body('Parse_JSON')?['humidity']}"
        },
        {
            "name": "temperature",
            "value": "@{body('Parse_JSON')?['temperature']}"
        },
        {
            "name": "timestamp",
            "value": ""
        }
    ],
    "connectionId": @{items('For_each')?['ConnectionId']},
    "messageSchemaVersion": "1.0.0",
    "messageName": "CreateContractActionRequest"
}
``` 

![Image](img/LogicAppConfigStep9.png)


Sources:

https://github.com/Azure-Samples/blockchain/blob/master/blockchain-workbench/iot-integration-samples/ConfigureIoTDemo.md

https://github.com/Azure-Samples/blockchain






