---
title: Understand the Azure IoT Hub identity registry
description: Developer guide - description of the IoT Hub identity registry and how to use it to manage your devices. Includes information about the import and export of device identities in bulk.
author: kgremban
ms.author: kgremban
ms.service: iot-hub
services: iot-hub
ms.topic: conceptual
ms.date: 06/29/2021
ms.custom: [amqp, mqtt, 'Role: Cloud Development', 'Role: IoT Device', ignite-2022]
---

# Understand the identity registry in your IoT hub

Every IoT hub has an identity registry that stores information about the devices and modules permitted to connect to the IoT hub. Before a device or module can connect to an IoT hub, there must be an entry for that device or module in the IoT hub's identity registry. A device or module must also authenticate with the IoT hub based on credentials stored in the identity registry.

The device or module ID stored in the identity registry is case-sensitive.

At a high level, the identity registry is a REST-capable collection of device or module identity resources. When you add an entry in the identity registry, IoT Hub creates a set of per-device resources such as the queue that contains in-flight cloud-to-device messages.

Use the identity registry when you need to:

* Provision devices or modules that connect to your IoT hub.
* Control per-device/per-module access to your hub's device or module-facing endpoints.

## Identity registry operations

The IoT Hub identity registry exposes the following operations:

* Create device or module identity
* Update device or module identity
* Retrieve device or module identity by ID
* Delete device or module identity
* List up to 1000 identities
* Export device identities to Azure blob storage
* Import device identities from Azure blob storage

All these operations can use optimistic concurrency, as specified in [RFC7232](https://tools.ietf.org/html/rfc7232).

> [!IMPORTANT]
> The only way to retrieve all identities in an IoT hub's identity registry is to use the [Export](iot-hub-devguide-identity-registry.md#import-and-export-device-identities) functionality.

An IoT Hub identity registry:

* Does not contain any application metadata.

> [!IMPORTANT]
> Only use the identity registry for device management and provisioning operations. High throughput operations at run time should not depend on performing operations in the identity registry. For example, checking the connection state of a device before sending a command is not a supported pattern. Make sure to check the [throttling rates](iot-hub-devguide-quotas-throttling.md) for the identity registry, and the [device heartbeat](iot-hub-devguide-identity-registry.md#device-heartbeat) pattern.

> [!NOTE]
> It can take a few seconds for a device or module identity to be available for retrieval after creation. Please retry `get` operation of device or module identities in case of failures.

## Disable devices

You can disable devices by updating the **status** property of an identity in the identity registry. Typically, you use this property in two scenarios:

* During a provisioning orchestration process. For more information, see [Device Provisioning](iot-hub-devguide-identity-registry.md#device-provisioning).

* If, for any reason, you think a device is compromised or has become unauthorized.

This feature is not available for modules.

## Import and export device identities

Use asynchronous operations on the [IoT Hub resource provider endpoint](iot-hub-devguide-endpoints.md) to export device identities in bulk from an IoT hub's identity registry. Exports are long-running jobs that use a customer-supplied blob container to save device identity data read from the identity registry.

Use asynchronous operations on the [IoT Hub resource provider endpoint](iot-hub-devguide-endpoints.md) to import device identities in bulk to an IoT hub's identity registry. Imports are long-running jobs that use data in a customer-supplied blob container to write device identity data into the identity registry.

For more information about the import and export APIs, see [IoT Hub resource provider REST APIs](/rest/api/iothub/iothubresource). To learn more about running import and export jobs, see [Bulk management of IoT Hub device identities](iot-hub-bulk-identity-mgmt.md).

Device identities can also be exported and imported from an IoT Hub via the Service API via either the [REST API](/rest/api/iothub/service/jobs/createimportexportjob) or one of the IoT Hub [Service SDKs](./iot-hub-devguide-sdks.md#azure-iot-hub-service-sdks).

## Device provisioning

The device data that a given IoT solution stores depends on the specific requirements of that solution. But, as a minimum, a solution must store device identities and authentication keys. Azure IoT Hub includes an identity registry that can store values for each device such as IDs, authentication keys, and status codes. A solution can use other Azure services such as Table storage, Blob storage, or Azure Cosmos DB to store any additional device data.

*Device provisioning* is the process of adding the initial device data to the stores in your solution. To enable a new device to connect to your hub, you must add a device ID and keys to the IoT Hub identity registry. As part of the provisioning process, you might need to initialize device-specific data in other solution stores. You can also use the Azure IoT Hub Device Provisioning Service to enable zero-touch, just-in-time provisioning to one or more IoT hubs without requiring human intervention. To learn more, see the [provisioning service documentation](/azure/iot-dps).

## Device heartbeat

The IoT Hub identity registry contains a field called **connectionState**. Only use the **connectionState** field during development and debugging. IoT solutions should not query the field at run time. For example, do not query the **connectionState** field to check if a device is connected before you send a cloud-to-device message or an SMS. We recommend subscribing to the [**device disconnected** event](iot-hub-event-grid.md#event-types) on Event Grid to get alerts and monitor the device connection state. Use this [tutorial](iot-hub-how-to-order-connection-state-events.md) to learn how to integrate Device Connected and Device Disconnected events from IoT Hub in your IoT solution.

If your IoT solution needs to know if a device is connected, you can implement the *heartbeat pattern*.
In the heartbeat pattern, the device sends device-to-cloud messages at least once every fixed amount of time (for example, at least once every hour). Therefore, even if a device does not have any data to send, it still sends an empty device-to-cloud message (usually with a property that identifies it as a heartbeat). On the service side, the solution maintains a map with the last heartbeat received for each device. If the solution does not receive a heartbeat message within the expected time from the device, it assumes that there is a problem with the device.

A more complex implementation could include the information from [Azure Monitor](../azure-monitor/index.yml) and [Azure Resource Health](../service-health/resource-health-overview.md) to identify devices that are trying to connect or communicate but failing. To learn more about using these services with IoT Hub, see [Monitor IoT Hub](monitor-iot-hub.md) and [Check IoT Hub resource health](iot-hub-azure-service-health-integration.md#check-iot-hub-health-with-azure-resource-health). For more specific information about using Azure Monitor or Event Grid to monitor device connectivity, see [Monitor, diagnose, and troubleshoot device connectivity](iot-hub-troubleshoot-connectivity.md). When you implement the heartbeat pattern, make sure to check [IoT Hub Quotas and Throttles](iot-hub-devguide-quotas-throttling.md).

> [!NOTE]
> If an IoT solution uses the connection state solely to determine whether to send cloud-to-device messages, and messages are not broadcast to large sets of devices, consider using the simpler *short expiry time* pattern. This pattern achieves the same result as maintaining a device connection state registry using the heartbeat pattern, while being more efficient. If you request message acknowledgements, IoT Hub can notify you about which devices are able to receive messages and which are not.

## Device and module lifecycle notifications

IoT Hub can notify your IoT solution when a device identity is created or deleted by sending lifecycle notifications. To do so, your IoT solution needs to create a route and to set the Data Source equal to *DeviceLifecycleEvents*. By default, no lifecycle notifications are sent, that is, no such routes pre-exist. By creating a route with Data Source equal to *DeviceLifecycleEvents*, lifecycle events will be sent for both device identities and module identities; however, the message contents will differ depending on whether the events are generated for module identities or device identities.  It should be noted that for IoT Edge modules, the module identity creation flow is different than for other modules, as a result for IoT Edge modules the create notification is only sent if the corresponding IoT Edge Device for the updated IoT Edge module identity is running. For all other modules, lifecycle notifications are sent whenever the module identity is updated on the IoT Hub side.  To learn more about the properties and body returned in the notification message, see  [Non-telemetry event schemas](iot-hub-non-telemetry-event-schema.md).

## Device identity properties

Device identities are represented as JSON documents with the following properties:

| Property | Options | Description |
| --- | --- | --- |
| deviceId |required, read-only on updates |A case-sensitive string (up to 128 characters long) of ASCII 7-bit alphanumeric characters plus certain special characters: `- . + % _ # * ? ! ( ) , : = @ $ '`. |
| generationId |required, read-only |An IoT hub-generated, case-sensitive string up to 128 characters long. This value is used to distinguish devices with the same **deviceId**, when they have been deleted and re-created. |
| etag |required, read-only |A string representing a weak ETag for the device identity, as per [RFC7232](https://tools.ietf.org/html/rfc7232). |
| authentication |optional |A composite object containing authentication information and security materials. For more information, see [Authentication Mechanism](/rest/api/iothub/service/devices/get-identity#authenticationmechanism) in the REST API documentation. |
| capabilities | optional | The set of capabilities of the device. For example, whether the device is an edge device or not. For more information, see [Device Capabilities](/rest/api/iothub/service/devices/get-identity#devicecapabilities) in the REST API documentation. |
| deviceScope | optional | The scope of the device. In edge devices, auto generated and immutable. Deprecated in non-edge devices. However, in child (leaf) devices, set this property to the same value as the **parentScopes** property (the **deviceScope** of the parent device) for backward compatibility with previous versions of the API. For more information, see [IoT Edge as a gateway: Parent and child relationships](../iot-edge/iot-edge-as-gateway.md#parent-and-child-relationships).|
| parentScopes | optional | The scope of a child device's direct parent (the value of the **deviceScope** property of the parent device). In edge devices, the value is empty if the device has no parent. In non-edge devices, the property is not present if the device has no parent. For more information, see [IoT Edge as a gateway: Parent and child relationships](../iot-edge/iot-edge-as-gateway.md#parent-and-child-relationships). |
| status |required |An access indicator. Can be **Enabled** or **Disabled**. If **Enabled**, the device is allowed to connect. If **Disabled**, this device cannot access any device-facing endpoint. |
| statusReason |optional |A 128 character-long string that stores the reason for the device identity status. All UTF-8 characters are allowed. |
| statusUpdateTime |read-only |A temporal indicator, showing the date and time of the last status update. |
| connectionState |read-only |A field indicating connection status: either **Connected** or **Disconnected**. This field represents the IoT Hub view of the device connection status. **Important**: This field should be used only for development/debugging purposes. The connection state is updated only for devices using MQTT or AMQP. Also, it is based on protocol-level pings (MQTT pings, or AMQP pings), and it can have a maximum delay of only 5 minutes. For these reasons, there can be false positives, such as devices reported as connected but that are disconnected. |
| connectionStateUpdatedTime |read-only |A temporal indicator, showing the date and last time the connection state was updated. |
| lastActivityTime |read-only |A temporal indicator, showing the date and last time the device connected, received, or sent a message. This property is eventually consistent, but could be delayed up to 5 to 10 minutes. For this reason, it shouldn't be used in production scenarios. |

> [!NOTE]
> Connection state can only represent the IoT Hub view of the status of the connection. Updates to this state may be delayed, depending on network conditions and configurations.

> [!NOTE]
> Currently the device SDKs do not support using  the `+` and `#` characters in the **deviceId**.

## Module identity properties

Module identities are represented as JSON documents with the following properties:

| Property | Options | Description |
| --- | --- | --- |
| deviceId |required, read-only on updates |A case-sensitive string (up to 128 characters long) of ASCII 7-bit alphanumeric characters plus certain special characters: `- . + % _ # * ? ! ( ) , : = @ $ '`. |
| moduleId |required, read-only on updates |A case-sensitive string (up to 128 characters long) of ASCII 7-bit alphanumeric characters plus certain special characters: `- . + % _ # * ? ! ( ) , : = @ $ '`. |
| generationId |required, read-only |An IoT hub-generated, case-sensitive string up to 128 characters long. This value is used to distinguish devices with the same **deviceId**, when they have been deleted and re-created. |
| etag |required, read-only |A string representing a weak ETag for the device identity, as per [RFC7232](https://tools.ietf.org/html/rfc7232). |
| authentication |optional |A composite object containing authentication information and security materials. For more information, see [Authentication Mechanism](/rest/api/iothub/service/modules/get-identity#authenticationmechanism) in the REST API documentation. |
| managedBy | optional | Identifies who manages this module. For instance, this value is "Iot Edge" if the edge runtime owns this module. |
| cloudToDeviceMessageCount | read-only | The number of cloud-to-module messages currently queued to be sent to the module. |
| connectionState |read-only |A field indicating connection status: either **Connected** or **Disconnected**. This field represents the IoT Hub view of the device connection status. **Important**: This field should be used only for development/debugging purposes. The connection state is updated only for devices using MQTT or AMQP. Also, it is based on protocol-level pings (MQTT pings, or AMQP pings), and it can have a maximum delay of only 5 minutes. For these reasons, there can be false positives, such as devices reported as connected but that are disconnected. |
| connectionStateUpdatedTime |read-only |A temporal indicator, showing the date and last time the connection state was updated. |
| lastActivityTime |read-only |A temporal indicator, showing the date and last time the device connected, received, or sent a message. |

> [!NOTE]
> Currently the device SDKs do not support using  the `+` and `#` characters in the **deviceId** and **moduleId**.

## Additional reference material

Other reference topics in the IoT Hub developer guide include:

* [IoT Hub endpoints](iot-hub-devguide-endpoints.md) describes the various endpoints that each IoT hub exposes for run-time and management operations.

* [Throttling and quotas](iot-hub-devguide-quotas-throttling.md) describes the quotas and throttling behaviors that apply to the IoT Hub service.

* [Azure IoT device and service SDKs](iot-hub-devguide-sdks.md) lists the various language SDKs you can use when you develop both device and service apps that interact with IoT Hub.

* [IoT Hub query language](iot-hub-devguide-query-language.md) describes the query language you can use to retrieve information from IoT Hub about your device twins and jobs.

* [IoT Hub MQTT support](iot-hub-mqtt-support.md) provides more information about IoT Hub support for the MQTT protocol.

## Next steps

Now that you have learned how to use the IoT Hub identity registry, you may be interested in the following IoT Hub developer guide topics:

* [Control access to IoT Hub](iot-hub-devguide-security.md)

* [Use device twins to synchronize state and configurations](iot-hub-devguide-device-twins.md)

* [Invoke a direct method on a device](iot-hub-devguide-direct-methods.md)

* [Schedule jobs on multiple devices](iot-hub-devguide-jobs.md)

To try out some of the concepts described in this article, see the following IoT Hub tutorial:

* [Get started with Azure IoT Hub](../iot-develop/quickstart-send-telemetry-iot-hub.md?pivots=programming-language-csharp)

To explore using the IoT Hub Device Provisioning Service to enable zero-touch, just-in-time provisioning, see: 

* [Azure IoT Hub Device Provisioning Service](/azure/iot-dps)
