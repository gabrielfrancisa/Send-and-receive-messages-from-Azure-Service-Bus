# Send-and-receive-messages-from-Azure-Service-Bus
This project, I create and configure Azure Service Bus resources, then build a .NET app to send and receive messages using the Azure.Messaging.ServiceBus SDK. I learnt how to provision a Service Bus namespace and queue, assign permissions, and interact with messages programmatically.


## Tasks performed in this exercise:
1. Create Azure Service Bus resources
2. Assign a role to your Microsoft Entra user name
3. Create a .NET console app to send and receive messages
4. Clean up resources


## Create Azure Event Hubs resources
1. In this section of the exercise, you create the needed resources in Azure with the Azure CLI.
2. In your browser, navigate to the Azure portal https://portal.azure.com, signing in with your Azure credentials if prompted.
3. Use the [>_] button to the right of the search bar at the top of the page to create a new Cloud Shell in the Azure portal, selecting a Bash environment. 
4. The cloud shell provides a command-line interface in a pane at the bottom of the Azure portal. If you are prompted to select a storage account to persist your files, select No storage account required, your subscription. 
5. and then select Apply.


## Step 2
1. Select Go to Classic version (this is required to use the code editor).
2. Create a resource group for the resources needed for this exercise. If you already have a resource group you want to use, proceed to the next step. 
3. Replace myResourceGroup with a name you want to use for the resource group. 
4. You can replace eastus with a region near you if needed.


## In this project, my Resource Group has already been created (myResourceGrouplod12121).

##Bash command 1
>>> az group create --name myResourceGroup --location eastus

i). Run the following commands to create the needed variables. 
ii). Replace myResourceGroup with the name you want. 
iii). If you changed the location in the previous step, make the same change in the location variable.

##Bash command 2
>>> resourceGroup=myResourceGrouplod55298919
>>> location=eastus
>>> namespaceName=svcbusns55298919

## Run the following command and record output.
>>> echo $namespaceName

## Create an Azure Service Bus namespace and queue
Create a Service Bus messaging namespace. 
The following command creates a namespace using the variable you created earlier. 
The operation takes a few minutes to complete.

>>> az servicebus namespace create \
    >>> --resource-group $resourceGroup \
    >>> --name $namespaceName \
    >>> --location $location

## Now that a namespace is created, you need to create a queue to hold the messages.
Run the following command to create a queue named myqueue.

>>> az servicebus queue create --resource-group $resourceGroup \
    >>> --namespace-name $namespaceName \
    >>> --name myqueue

## Assign a role to your Microsoft Entra user name
To allow your app to send and receive messages, assign your Microsoft Entra user to the Azure Service Bus Data Owner role at the Service Bus namespace level. 
This gives your user account permission to manage and access queues and topics using Azure RBAC. 
>> Perform the following steps in the cloud shell.

## Run the following command to retrieve the userPrincipalName from your account. This represents who the role will be assigned to.
>>> userPrincipal=$(az rest --method GET --url https://graph.microsoft.com/v1.0/me \
    >>> --headers 'Content-Type=application/json' \
    >>> --query userPrincipalName --output tsv)

## Run the following command to retrieve the resource ID of the Service Bus namespace. The resource ID sets the scope for the role assignment to a specific namespace.
>>> resourceID=$(az servicebus namespace show --name $namespaceName \
    >>> --resource-group $resourceGroup \
    >>> --query id --output tsv)

## Run the following command to create and assign the Azure Service Bus Data Owner role.
>>> az role assignment create --assignee $userPrincipal \
    >>> --role "Azure Service Bus Data Owner" \
    >>> --scope $resourceID


## Create a .NET console app to send and receive messages
# Now that the needed resources are deployed to Azure the next step is to set up the console application. 
# The following steps are performed in the cloud shell.


## Run the following commands to create a directory to contain the project and change into the project directory.
>>> mkdir svcbus
>>> cd svcbus

## Create the .NET console application.
>>> dotnet new console

## Run the following commands to add the Azure.Messaging.ServiceBus and Azure.Identity packages to the project.
>>> dotnet add package Azure.Messaging.ServiceBus
>>> dotnet add package Azure.Identity

## Add the starter code for the project
# Run the following command in the cloud shell to begin editing the application.


## Replace any existing contents with the following code. Be sure to review the comments in the code, and replace it with the Service Bus namespace you recorded earlier.

```csharp
using Azure.Messaging.ServiceBus;
using Azure.Identity;
using System.Timers;

// Replace <YOUR-NAMESPACE> with your Service Bus namespace
string svcbusNameSpace = "<YOUR-NAMESPACE>.servicebus.windows.net";
string queueName = "myQueue";

// ADD CODE TO CREATE A SERVICE BUS CLIENT
// Create a DefaultAzureCredentialOptions object to configure the DefaultAzureCredential
DefaultAzureCredentialOptions options = new()
{
    ExcludeEnvironmentCredential = true,
    ExcludeManagedIdentityCredential = true
};

// Create a Service Bus client using the namespace and DefaultAzureCredential
// The DefaultAzureCredential will use the Azure CLI credentials, so ensure you are logged in
ServiceBusClient client = new(svcbusNameSpace, new DefaultAzureCredential(options));

// ADD CODE TO SEND MESSAGES TO THE QUEUE
// Create a sender for the specified queue
ServiceBusSender sender = client.CreateSender(queueName);

// create a batch 
using ServiceBusMessageBatch messageBatch = await sender.CreateMessageBatchAsync();

// number of messages to be sent to the queue
const int numOfMessages = 3;

for (int i = 1; i <= numOfMessages; i++)
{
    // try adding a message to the batch
    if (!messageBatch.TryAddMessage(new ServiceBusMessage($"Message {i}")))
    {
        // if it is too large for the batch
        throw new Exception($"The message {i} is too large to fit in the batch.");
    }
}

try
{
    // Use the producer client to send the batch of messages to the Service Bus queue
    await sender.SendMessagesAsync(messageBatch);
    Console.WriteLine($"A batch of {numOfMessages} messages has been published to the queue.");
}
finally
{
    // Calling DisposeAsync on client types is required to ensure that network
    // resources and other unmanaged objects are properly cleaned up.
    await sender.DisposeAsync();
}

Console.WriteLine("Press any key to continue");
Console.ReadKey();


// ADD CODE TO PROCESS MESSAGES FROM THE QUEUE
// Create a processor that we can use to process the messages in the queue
ServiceBusProcessor processor = client.CreateProcessor(queueName, new ServiceBusProcessorOptions());

// Idle timeout in milliseconds, the idle timer will stop the processor if there are no more 
// messages in the queue to process
const int idleTimeoutMs = 3000;
System.Timers.Timer idleTimer = new(idleTimeoutMs);
idleTimer.Elapsed += async (s, e) =>
{
    Console.WriteLine($"No messages received for {idleTimeoutMs / 1000} seconds. Stopping processor...");
    await processor.StopProcessingAsync();
};

try
{
    // add handler to process messages
    processor.ProcessMessageAsync += MessageHandler;

    // add handler to process any errors
    processor.ProcessErrorAsync += ErrorHandler;

    // start processing 
    idleTimer.Start();
    await processor.StartProcessingAsync();

    Console.WriteLine($"Processor started. Will stop after {idleTimeoutMs / 1000} seconds of inactivity.");
    // Wait for the processor to stop
    while (processor.IsProcessing)
    {
        await Task.Delay(500);
    }
    idleTimer.Stop();
    Console.WriteLine("Stopped receiving messages");
}
finally
{
    // Dispose processor after use
    await processor.DisposeAsync();
}

// handle received messages
async Task MessageHandler(ProcessMessageEventArgs args)
{
    string body = args.Message.Body.ToString();
    Console.WriteLine($"Received: {body}");

    // Reset the idle timer on each message
    idleTimer.Stop();
    idleTimer.Start();

    // complete the message. message is deleted from the queue. 
    await args.CompleteMessageAsync(args.Message);
}

// handle any errors when receiving messages
Task ErrorHandler(ProcessErrorEventArgs args)
{
    Console.WriteLine(args.Exception.ToString());
    return Task.CompletedTask;
}

// Dispose client after use
await client.DisposeAsync();
```

## Sign into Azure and run the app
# In the cloud shell command-line pane, enter the following command to sign into Azure.
>>> az login
You must sign into Azure - even though the cloud shell session is already authenticated.


## Run the following command to start the console app. 
# The app will pause a various stages and prompt you to press a key to continue. 
# This allows you to view the messages in the Azure portal.

>>> dotnet run

# In the Azure portal, navigate to the Service Bus namespace you created.
# Select myqueue at the bottom of the Overview window.
# Select Service Bus Explorer in the left navigation pane.

# Select Peek from start, and the three messages should appear after a few seconds.
1. In the cloud shell, press any key to continue, and the application will process the three messages.
2. Return to the portal after the application has completed processing the messages. Select Peek from start again and notice there are no messages in the queue.

## Clean up resources
1. Now that you have finished the project, you should delete the cloud resources you created to avoid unnecessary resource usage.
2. In your browser, navigate to the Azure portal https://portal.azure.com, signing in with your Azure credentials if prompted.
3. Navigate to the resource group you created and view the contents of the resources used in this exercise.
4. On the toolbar, select Delete resource group.
5. Enter the resource group name and confirm that you want to delete it.


Congratulations, end of my project.
