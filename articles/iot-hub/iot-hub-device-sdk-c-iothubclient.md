---
title: Azure IoT device SDK for C - IoTHubClient | Microsoft Docs
description: How to use the IoTHubClient library in the Azure IoT device SDK for C to create device apps that communicate with an IoT hub.
author: kgremban
ms.service: iot-hub
services: iot-hub
ms.devlang: c
ms.topic: conceptual
ms.date: 08/29/2017
ms.author: kgremban
ms.custom: amqp
---

# Azure IoT device SDK for C – more about IoTHubClient

[Azure IoT device SDK for C](iot-hub-device-sdk-c-intro.md) is the first article in this series introducing the **Azure IoT device SDK for C**. That article explained that there are two architectural layers in SDK. At the base is the **IoTHubClient** library that directly manages communication with IoT Hub. There's also the **serializer** library that builds on top of that to provide serialization services. In this article, we'll provide additional detail on the **IoTHubClient** library.

[!INCLUDE [iot-hub-basic](../../includes/iot-hub-basic-partial.md)]

The previous article described how to use the **IoTHubClient** library to send events to IoT Hub and receive messages. This article extends that discussion by explaining how to more precisely manage *when* you send and receive data, introducing you to the **lower-level APIs**. We'll also explain how to attach properties to events (and retrieve them from messages) using the property handling features in the **IoTHubClient** library. Finally, we'll provide additional explanation of different ways to handle messages received from IoT Hub.

The article concludes by covering a couple of miscellaneous topics, including more about device credentials and how to change the behavior of the **IoTHubClient** through configuration options.

We'll use the **IoTHubClient** SDK samples to explain these topics. If you want to follow along, see the **iothub\_client\_sample\_http** and **iothub\_client\_sample\_amqp** applications that are included in the Azure IoT device SDK for C. Everything described in the following sections is demonstrated in these samples.

You can find the [**Azure IoT device SDK for C**](https://github.com/Azure/azure-iot-sdk-c) GitHub repository and view details of the API in the [C API reference](https://github.com/Azure/azure-iot-sdk-c/).

## The lower-level APIs

The previous article described the basic operation of the **IotHubClient** within the context of the **iothub\_client\_sample\_amqp** application. For example, it explained how to initialize the library using this code.

```C
IOTHUB_CLIENT_HANDLE iotHubClientHandle;
iotHubClientHandle = IoTHubClient_CreateFromConnectionString(connectionString, AMQP_Protocol);
```

It also described how to send events using this function call.

```C
IoTHubClient_SendEventAsync(iotHubClientHandle, message.messageHandle, SendConfirmationCallback, &message);
```

The article also described how to receive messages by registering a callback function.

```C
int receiveContext = 0;
IoTHubClient_SetMessageCallback(iotHubClientHandle, ReceiveMessageCallback, &receiveContext);
```

The article also showed how to free resources using code such as the following.

```C
IoTHubClient_Destroy(iotHubClientHandle);
```

There are companion functions for each of these APIs:

* IoTHubClient\_LL\_CreateFromConnectionString
* IoTHubClient\_LL\_SendEventAsync
* IoTHubClient\_LL\_SetMessageCallback
* IoTHubClient\_LL\_Destroy

These functions all include **LL** in the API name. Other the **LL** part of the name, the parameters of each of these functions are identical to their non-LL counterparts. However, the behavior of these functions is different in one important way.

When you call **IoTHubClient\_CreateFromConnectionString**, the underlying libraries create a new thread that runs in the background. This thread sends events to, and receives messages from, IoT Hub. No such thread is created when working with the **LL** APIs. The creation of the background thread is a convenience to the developer. You don't have to worry about explicitly sending events and receiving messages from IoT Hub -- it happens automatically in the background. In contrast, the **LL** APIs give you explicit control over communication with IoT Hub, if you need it.

To understand this concept better, let's look at an example:

When you call **IoTHubClient\_SendEventAsync**, what you're actually doing is putting the event in a buffer. The background thread created when you call **IoTHubClient\_CreateFromConnectionString** continually monitors this buffer and sends any data that it contains to IoT Hub. This happens in the background at the same time that the main thread is performing other work.

Similarly, when you register a callback function for messages using **IoTHubClient\_SetMessageCallback**, you're instructing the SDK to have the background thread invoke the callback function when a message is received, independent of the main thread.

The **LL** APIs don't create a background thread. Instead, a new API must be called to explicitly send and receive data from IoT Hub. This is demonstrated in the following example.

The **iothub\_client\_sample\_http** application that's included in the SDK demonstrates the lower-level APIs. In that sample, we send events to IoT Hub with code such as the following:

```C
EVENT_INSTANCE message;
sprintf_s(msgText, sizeof(msgText), "Message_%d_From_IoTHubClient_LL_Over_HTTP", i);
message.messageHandle = IoTHubMessage_CreateFromByteArray((const unsigned char*)msgText, strlen(msgText));

IoTHubClient_LL_SendEventAsync(iotHubClientHandle, message.messageHandle, SendConfirmationCallback, &message)
```

The first three lines create the message, and the last line sends the event. However, as mentioned previously, sending the event means that the data is simply placed in a buffer. Nothing is transmitted on the network when we call **IoTHubClient\_LL\_SendEventAsync**. In order to actually ingress the data to IoT Hub, you must call **IoTHubClient\_LL\_DoWork**, as in this example:

```C
while (1)
{
    IoTHubClient_LL_DoWork(iotHubClientHandle);
    ThreadAPI_Sleep(100);
}
```

This code (from the **iothub\_client\_sample\_http** application) repeatedly calls **IoTHubClient\_LL\_DoWork**. Each time **IoTHubClient\_LL\_DoWork** is called, it sends some events from the buffer to IoT Hub and it retrieves a queued message being sent to the device. The latter case means that if we registered a callback function for messages, then the callback is invoked (assuming any messages are queued up). We would have registered such a callback function with code such as the following:

```C
IoTHubClient_LL_SetMessageCallback(iotHubClientHandle, ReceiveMessageCallback, &receiveContext)
```

The reason that **IoTHubClient\_LL\_DoWork** is often called in a loop is that each time it's called, it sends *some* buffered events to IoT Hub and retrieves *the next* message queued up for the device. Each call isn't guaranteed to send all buffered events or to retrieve all queued messages. If you want to send all events in the buffer and then continue on with other processing you can replace this loop with code such as the following:

```C
IOTHUB_CLIENT_STATUS status;

while ((IoTHubClient_LL_GetSendStatus(iotHubClientHandle, &status) == IOTHUB_CLIENT_OK) && (status == IOTHUB_CLIENT_SEND_STATUS_BUSY))
{
    IoTHubClient_LL_DoWork(iotHubClientHandle);
    ThreadAPI_Sleep(100);
}
```

This code calls **IoTHubClient\_LL\_DoWork** until all events in the buffer have been sent to IoT Hub. Note this does not also imply that all queued messages have been received. Part of the reason for this is that checking for "all" messages isn't as deterministic an action. What happens if you retrieve "all" of the messages, but then another one is sent to the device immediately after? A better way to deal with that is with a programmed timeout. For example, the message callback function could reset a timer every time it's invoked. You can then write logic to continue processing if, for example, no messages have been received in the last *X* seconds.

When you're finished ingressing events and receiving messages, be sure to call the corresponding function to clean up resources.

```C
IoTHubClient_LL_Destroy(iotHubClientHandle);
```

Basically there's only one set of APIs to send and receive data with a background thread and another set of APIs that does the same thing without the background thread. A lot of developers may prefer the non-LL APIs, but the lower-level APIs are useful when the developer wants explicit control over network transmissions. For example, some devices collect data over time and only ingress events at specified intervals (for example, once an hour or once a day). The lower-level APIs give you the ability to explicitly control when you send and receive data from IoT Hub. Others will simply prefer the simplicity that the lower-level APIs provide. Everything happens on the main thread rather than some work happening in the background.

Whichever model you choose, be sure to be consistent in which APIs you use. If you start by calling **IoTHubClient\_LL\_CreateFromConnectionString**, be sure you only use the corresponding lower-level APIs for any follow-up work:

* IoTHubClient\_LL\_SendEventAsync
* IoTHubClient\_LL\_SetMessageCallback
* IoTHubClient\_LL\_Destroy
* IoTHubClient\_LL\_DoWork

The opposite is true as well. If you start with **IoTHubClient\_CreateFromConnectionString**, then use the non-LL APIs for any additional processing.

In the Azure IoT device SDK for C, see the **iothub\_client\_sample\_http** application for a complete example of the lower-level APIs. The **iothub\_client\_sample\_amqp** application can be referenced for a full example of the non-LL APIs.

## Property handling

So far when we've described sending data, we've been referring to the body of the message. For example, consider this code:

```C
EVENT_INSTANCE message;
sprintf_s(msgText, sizeof(msgText), "Hello World");
message.messageHandle = IoTHubMessage_CreateFromByteArray((const unsigned char*)msgText, strlen(msgText));
IoTHubClient_LL_SendEventAsync(iotHubClientHandle, message.messageHandle, SendConfirmationCallback, &message)
```

This example sends a message to IoT Hub with the text "Hello World." However, IoT Hub also allows properties to be attached to each message. Properties are name/value pairs that can be attached to the message. For example, we can modify the previous code to attach a property to the message:

```C
MAP_HANDLE propMap = IoTHubMessage_Properties(message.messageHandle);
sprintf_s(propText, sizeof(propText), "%d", i);
Map_AddOrUpdate(propMap, "SequenceNumber", propText);
```

We start by calling **IoTHubMessage\_Properties** and passing it the handle of our message. What we get back is a **MAP\_HANDLE** reference that enables us to start adding properties. The latter is accomplished by calling **Map\_AddOrUpdate**, which takes a reference to a MAP\_HANDLE, the property name, and the property value. With this API we can add as many properties as we like.

When the event is read from **Event Hubs**, the receiver can enumerate the properties and retrieve their corresponding values. For example, in .NET this would be accomplished by accessing the [Properties collection on the EventData object](/dotnet/api/microsoft.servicebus.messaging.eventdata).

In the previous example, we're attaching properties to an event that we send to IoT Hub. Properties can also be attached to messages received from IoT Hub. If we want to retrieve properties from a message, we can use code such as the following in our message callback function:

```C
static IOTHUBMESSAGE_DISPOSITION_RESULT ReceiveMessageCallback(IOTHUB_MESSAGE_HANDLE message, void* userContextCallback)
{
    . . .

    // Retrieve properties from the message
    MAP_HANDLE mapProperties = IoTHubMessage_Properties(message);
    if (mapProperties != NULL)
    {
        const char*const* keys;
        const char*const* values;
        size_t propertyCount = 0;
        if (Map_GetInternals(mapProperties, &keys, &values, &propertyCount) == MAP_OK)
        {
            if (propertyCount > 0)
            {
                printf("Message Properties:\r\n");
                for (size_t index = 0; index < propertyCount; index++)
                {
                    printf("\tKey: %s Value: %s\r\n", keys[index], values[index]);
                }
                printf("\r\n");
            }
        }
    }

    . . .
}
```

The call to **IoTHubMessage\_Properties** returns the **MAP\_HANDLE** reference. We then pass that reference to **Map\_GetInternals** to obtain a reference to an array of the name/value pairs (as well as a count of the properties). At that point it's a simple matter of enumerating the properties to get to the values we want.

You don't have to use properties in your application. However, if you need to set them on events or retrieve them from messages, the **IoTHubClient** library makes it easy.

## Message handling

As stated previously, when messages arrive from IoT Hub the **IoTHubClient** library responds by invoking a registered callback function. There is a return parameter of this function that deserves some additional explanation. Here's an excerpt of the callback function in the **iothub\_client\_sample\_http** sample application:

```C
static IOTHUBMESSAGE_DISPOSITION_RESULT ReceiveMessageCallback(IOTHUB_MESSAGE_HANDLE message, void* userContextCallback)
{
    . . .
    return IOTHUBMESSAGE_ACCEPTED;
}
```

Note that the return type is **IOTHUBMESSAGE\_DISPOSITION\_RESULT** and in this particular case we return **IOTHUBMESSAGE\_ACCEPTED**. There are other values we can return from this function that change how the **IoTHubClient** library reacts to the message callback. Here are the options.

* **IOTHUBMESSAGE\_ACCEPTED** – The message has been processed successfully. The **IoTHubClient** library will not invoke the callback function again with the same message.

* **IOTHUBMESSAGE\_REJECTED** – The message was not processed and there is no desire to do so in the future. The **IoTHubClient** library should not invoke the callback function again with the same message.

* **IOTHUBMESSAGE\_ABANDONED** – The message was not processed successfully, but the **IoTHubClient** library should invoke the callback function again with the same message.

For the first two return codes, the **IoTHubClient** library sends a message to IoT Hub indicating that the message should be deleted from the device queue and not delivered again. The net effect is the same (the message is deleted from the device queue), but whether the message was accepted or rejected is still recorded.  Recording this distinction is useful to senders of the message who can listen for feedback and find out if a device has accepted or rejected a particular message.

In the last case a message is also sent to IoT Hub, but it indicates that the message should be redelivered. Typically you'll abandon a message if you encounter some error but want to try to process the message again. In contrast, rejecting a message is appropriate when you encounter an unrecoverable error (or if you simply decide you don't want to process the message).

In any case, be aware of the different return codes so that you can elicit the behavior you want from the **IoTHubClient** library.

## Alternate device credentials

As explained previously, the first thing to do when working with the **IoTHubClient** library is to obtain a **IOTHUB\_CLIENT\_HANDLE** with a call such as the following:

```C
IOTHUB_CLIENT_HANDLE iotHubClientHandle;
iotHubClientHandle = IoTHubClient_CreateFromConnectionString(connectionString, AMQP_Protocol);
```

The arguments to **IoTHubClient\_CreateFromConnectionString** are the device connection string and a parameter that indicates the protocol we use to communicate with IoT Hub. The device connection string has a format that appears as follows:

```C
HostName=IOTHUBNAME.IOTHUBSUFFIX;DeviceId=DEVICEID;SharedAccessKey=SHAREDACCESSKEY
```

There are four pieces of information in this string: IoT Hub name, IoT Hub suffix, device ID, and shared access key. You obtain the fully qualified domain name (FQDN) of an IoT hub when you create your IoT hub instance in the Azure portal — this gives you the IoT hub name (the first part of the FQDN) and the IoT hub suffix (the rest of the FQDN). You get the device ID and the shared access key when you register your device with IoT Hub (as described in the [previous article](iot-hub-device-sdk-c-intro.md)).

**IoTHubClient\_CreateFromConnectionString** gives you one way to initialize the library. If you prefer, you can create a new **IOTHUB\_CLIENT\_HANDLE** by using these individual parameters rather than the device connection string. This is achieved with the following code:

```C
IOTHUB_CLIENT_CONFIG iotHubClientConfig;
iotHubClientConfig.iotHubName = "";
iotHubClientConfig.deviceId = "";
iotHubClientConfig.deviceKey = "";
iotHubClientConfig.iotHubSuffix = "";
iotHubClientConfig.protocol = HTTP_Protocol;
IOTHUB_CLIENT_HANDLE iotHubClientHandle = IoTHubClient_LL_Create(&iotHubClientConfig);
```

This accomplishes the same thing as **IoTHubClient\_CreateFromConnectionString**.

It may seem obvious that you would want to use **IoTHubClient\_CreateFromConnectionString** rather than this more verbose method of initialization. Keep in mind, however, that when you register a device in IoT Hub what you get is a device ID and device key (not a connection string). The *device explorer* SDK tool introduced in the [previous article](iot-hub-device-sdk-c-intro.md) uses libraries in the **Azure IoT service SDK** to create the device connection string from the device ID, device key, and IoT Hub host name. So calling **IoTHubClient\_LL\_Create** may be preferable because it saves you the step of generating a connection string. Use whichever method is convenient.

## Configuration options

So far everything described about the way the **IoTHubClient** library works reflects its default behavior. However, there are a few options that you can set to change how the library works. This is accomplished by leveraging the **IoTHubClient\_LL\_SetOption** API. Consider this example:

```C
unsigned int timeout = 30000;
IoTHubClient_LL_SetOption(iotHubClientHandle, "timeout", &timeout);
```

There are a couple of options that are commonly used:

* **SetBatching** (bool) – If **true**, then data sent to IoT Hub is sent in batches. If **false**, then messages are sent individually. The default is **false**. You can use the batching option with AMQP / AMQP-WS, as well as adding system properties on D2C messages, is supported.

* **Timeout** (unsigned int) – This value is represented in milliseconds. If sending an HTTPS request or receiving a response takes longer than this time, then the connection times out.

The batching option is important. By default, the library ingresses events individually (a single event is whatever you pass to **IoTHubClient\_LL\_SendEventAsync**). If the batching option is **true**, the library collects as many events as it can from the buffer (up to the maximum message size that IoT Hub will accept).  The event batch is sent to IoT Hub in a single HTTPS call (the individual events are bundled into a JSON array). Enabling batching typically results in big performance gains since you're reducing network round-trips. It also significantly reduces bandwidth since you are sending one set of HTTPS headers with an event batch rather than a set of headers for each individual event. Unless you have a specific reason to do otherwise, typically you'll want to enable batching.

## Next steps

This article describes in detail the behavior of the **IoTHubClient** library found in the **Azure IoT device SDK for C**. With this information, you should have a good understanding of the capabilities of the **IoTHubClient** library. 

To learn more about developing for IoT Hub, see the [Azure IoT SDKs](iot-hub-devguide-sdks.md).

To further explore the capabilities of IoT Hub, see [Deploying AI to edge devices with Azure IoT Edge](../iot-edge/quickstart-linux.md).