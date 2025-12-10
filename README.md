# Private preview for Crash consistent RP with Premium SSD v2 and Ultra Disks 
## Overview
This document explains how to create and manage Instant Access (IA) crash consistent restore points for VMs having Premium SSD v2 and Ultra disks, leveraging the latest API features. Instant Access enables rapid disk restoration from snapshots, with configurable access duration and enhanced performance.

**NOTE: This feature should not be used for production workloads until General Availability (GA). Microsoft Privacy Statement: https://privacy.microsoft.com/en-us/privacystatement**

## Prerequisites
- Sign up [here](https://forms.office.com/r/jUVfGNzfJ8) for private preview.
- Create a VM with Premium SSD v2 and/or Ultra disks as data disks and Premium SSD v1 OS disk. 
- API version **2025-04-01** or later is supported.
- Regions Supported: EASTUS2EUAP.
- Client tools supported: REST API.
- VM SKUs that support SCSI disk controller. E.g. Mv2-series, Mdsv2, Msv2, B-series (Bsv2, Basv2), D-series (Dv2, Dsv2), Dasv5 / Dadsv5, Esv5 / Edsv5, F-series (Fsv2), G-series, GPU families (NC, ND, NV)
## Unsupported Configurations
- More than 50 restore points should not be created concurrently at a given time per subscription per region.
- Security types not supported:
  - Trusted Launch
  - Confidential 
- VMs using the below SKU:
  - Intel v6+ e.g. Dsv6-series, Edsv6-series, Esv6 series etc...
  - AMD v7+ e.g. Dasv7-series, Dadsv7-series, Easv7-series, Faldsv7-series etc... 
- VM SKUs that support NVMe disk controller.
## Key Concepts
- Instant Access (IA): Allows immediate restoration of disks from snapshots, with a default duration of 5 hours (configurable between 60 and 300 minutes). Its a boolean property to be enabled at the restore point collection level.
- instantAccessDurationMinutes: Integer property to set IA duration (in minutes) at the restore point level.
- SnapshotAccessState: Indicates the access status of the disk restore point (e.g., Pending, Available, InstantAccess, AvailableWithInstantAccess).
- InstantAccessState: Indicates the access status of all disk restore points within a restore point (e.g., Pending, Available, InstantAccess, AvailableWithInstantAccess).
## Steps to be followed
### Step 1: Enable Instant Access on Restore Point Collection
  Use the following REST API call to create a restore point collection with IA enabled on the VM that was created as mentioned in the perquisites above. 
  ```http
  PUT https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Compute/restorePointCollections/{restorePointCollectionName}?api-version=2025-04-01
  ```
#### Request Body Example:
```json
  {
    "location": "<region>",
    "properties": {
      "source": {
        "id": "<VM Arm Id>"
      },
      "instantAccess": true
    },
    "tags": {
      "myTag1": "tagValue1"
    }
  }
  ```
  - instantAccess: Set to true to enable IA. Default is false if not provided.
  - You can update this property later using a PATCH call if needed to disable the feature.
### Step 2: Create a Restore Point with Custom IA Duration
  Create a restore point and specify the IA duration (between 60 and 300 minutes):
  ```http  
  PUT https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Compute/restorePointCollections/{restorePointCollectionName}/restorePoints/{restorePointName}?api-version=2025-04-01
  ```
#### Request Body Example:
```json
  {
    "name": "<restorePointName>",
    "properties": {
     "instantAccessDurationMinutes": 120,
      "provisioningState": "Succeeded",
      "consistencyMode": "CrashConsistent"
    }
  }
  ```
  - instantAccessDurationMinutes: Optional. Default is 300 (5 hours). Can be set to a lower value, but not higher than 300. 
  Use only PUT restore point without IA duration in the body if you prefer to use the default 5 hours.

### Step 3: Validate Restore Point Creation
To verify the IA status and duration, use the GET API:
```http
GET https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Compute/restorePointCollections/{restorePointCollectionName}/restorePoints/{restorePointName}?api-version=2025-04-01
```
#### Response Example:
```json
{
  "name": "<Restore Point Name>",
  "id": "<Restore Point Arm Id>",
  "properties": {
    "instantAccessDurationMinutes": 120,
    "provisioningState": "Succeeded",
    "instantAccessStatus": "AvailablewithInstantAccess"
  }
}
```
- Check instantAccessDurationMinutes , provisioningState and InstantAccessStatus.


| provisioningState | Description |
|-------------------|-------------|
| instantAccessDurationMinutes |	1 – 300 [default is 300] |
| provisioningState | Creating – The restore point is being created.<br> Succeeded – The restore point was successfully created and is ready.<br> Failed – The creation failed.<br> Deleting – The restore point is being deleted.<br> Cancelled – The operation was cancelled.<br> Updating – The restore point is being updated.<br> InProgress – The operation is still ongoing (sometimes seen in older SDKs).<br> |
| InstantAccessStatus | Pending – Restore points in this state cannot be used for restore, copy, or offline download. A restore point is marked as Pending if any disk restore point within it has a snapshotAccessState of pending. <br><br>  Available – Restore points in this state can be used for restore, copy to a different region, and offline download. A restore point becomes Available when all disk restore points within it have a snapshotAccessState of available. This typically occurs after the instantAccessDuration has expired. <br> <br> InstantAccess - Restore points in this state allow fast disk restore but cannot be copied or downloaded. A restore point is marked as InstantAccess when all disk restore points within it have a snapshotAccessState of InstantAccess. <br> <br> AvailablewithInstantAccess - Restore points in this state allow fast disk restore, and they can also be copied and downloaded. A restore point is marked as AvailableWithInstantAccess when all disks restore points within it have a snapshotAccessState of AvailableWithInstantAccess. This applies when the instantAccessDuration has not yet expired. |

### Step 4: Check Instance View for Disk Restore Point Status
To get detailed status, including IA availability, use the instance view expansion:
```http
GET https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Compute/restorePointCollections/{restorePointCollectionName}/restorePoints/{restorePointName}?$expand=instanceView&api-version=2025-04-01
```
#### Instance View Example:
```json
{
  "instanceView": {
    "diskRestorePoints": [
      {
        "id": "<disk restore point Arm Id>",
        "snapshotAccessState": "Pending/Available/InstantAccess/AvailableWithInstantAccess ",
        "replicationStatus": {
          "status": {
            "code": "ReplicationState/succeeded",
            "level": "Info",
            "displayStatus": "Succeeded"
          },
          "completionPercent": 100
        }
      }
    ]
  }
}
```
- snapshotAccessState: Indicates IA status.
### Step 5: Restore Disk from Disk Restore Point
Before restoring the disk check the instantAcessStatus at restore point. Only when the state is  
```http
GET https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Compute/restorePointCollections/{restorePointCollectionName}/restorePoints/{restorePointName}?api-version=2025-04-01
```
#### Response Example:
```json
{
  "name": "<Restore Point Name>",
  "id": "<Restore Point Arm Id>",
  "properties": {
    "instantAccessDurationMinutes": 120,
    "provisioningState": "Succeeded",
    "instantAccessStatus": "AvailablewithInstantAccess"
  }
}
```
# Next Steps
Learn more about Backup and restore options for virtual machines in Azure.
