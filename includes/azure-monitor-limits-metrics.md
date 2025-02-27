---
title: "include file" 
description: "include file" 
services: azure-monitor
author: rboucher
tags: azure-service-management
ms.prod: devops
ms.topic: "include"
ms.date: 07/22/2019
ms.author: bwren
ms.custom: "include file"
---

### Ingestion

The following limits apply to the Azure Monitor workspace ingesting your Prometheus metrics.

| Limit | Value |
|:---|:---|
| Active time series with metrics that have been reported in the last ~12 hours.  | 1,000,000<br>You can request an increase.  |
| Events per minute ingested | 1,000,000<br>You can request an increase. |


The following limits apply to the data collection rule (DCR) and data collection endpoint (DCE) sending Prometheus metrics data to your Azure Monitor workspace

| Limit | Value |
|:---|:---|
| Ingestion requests per minute to a data collection endpoint  | 15,000<br>This limit can't be increased.   |
| Data ingestion per minute to a data collection endpoint | 50 GB<br>This limit can't be increased. |

#### Queries 
Prometheus queries are created using PromQL and can be authored in either Azure Managed Grafana or self-managed Grafana. 


| Limit | Value |
|:---|:---|
| Data retention | 18 months<br>This limit can't be increased.  
| Query time range | 32 days between the start time and end time of your PromQL query<br>This limit can't be increased. |
| Query data limits | 240 GB data returned per Azure Monitor workspace over a 30-second window |
| Query time series per metric | 150K time series |
| Query samples returned | 50M samples per query |
| Minimum query step size<br>with time range >= 48 hours | 60 seconds |


**Query pre-parsing limits**<br>
Based on query time range and request type over a 30-second window.<br>
Each value is per user (Azure AD or System Identity).

| Limit | Value |
|:---|:---|
| Query hours per Azure Managed Grafana Workspace | 30,000 |
| Query hours per Azure Monitor workspace | 60,000 |
| Query hours per Azure tenant | 600,000 |


**Query post-parsing limits**<br>
Based on query time range and range vectors in query over a 30-second window.<br>
Each value is per user (Azure AD or System Identity).

| Limit | Value |
|:---|:---|
| Query hours per Azure Managed Grafana Workspace | 2,000,000 |
| Query hours per Azure Monitor workspace | 2,000,000 |
| Query hours per Azure tenant | 20,000,000 |




#### Alert rules 
Prometheus alert rules are defined in PromQL and are performed on the managed Ruler service as part of Azure Monitor managed service for Prometheus. 

| Limit | Value |
|:---|:---|
| Rule groups per Azure Monitor workspace  | 100<br>You can request an increase. |
| Rule groups per Azure subscription  | 100<br>You can request an increase. |
| Rules per rule group | 20<br>You can request an increase. |
| Rule group evaluation interval | Between 1-15 minutes.<br>Default is 1 minute. |
| Active alerts | No limit at this time. |


#### Remote write
Calculations were determined using a remote batch size of 500 which is the default.

| Limit | Value |
|:---|:---|
| CPU usage | 0.25 x (number of metrics) + 1.25 x (avg number of series per metric) |
| CPU request | 0.75 x (CPU usage) |
| CPU limit | 2 x (CPU request) |
| Memory request | 150 Mb |
| Memory limit | 200 Mb |
| Maximum throughput | Remote write container can process up to 150,000 unique time series. The container may throw errors serving requests over 150,000 due to the high number of concurrent connections. This issue can be mitigated by increasing the remote batch size from 500 to 1,000. This reduces the number of open connections. |

