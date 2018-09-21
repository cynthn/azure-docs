---
title: Scale discovery and assessment by using Azure Migrate | Microsoft Docs
description: Describes how to assess large numbers of on-premises machines by using the Azure Migrate service.
author: rayne-wiselman
ms.service: azure-migrate
ms.topic: article
ms.date: 05/18/2018
ms.author: raynew
---

# Discover and assess a large VMware environment

This article describes how to assess large numbers of on-premises virtual machines (VMs) by using [Azure Migrate](migrate-overview.md). Azure Migrate assesses machines to check whether they're suitable for migration to Azure. The service provides sizing and cost estimations for running the machines in Azure.

## Prerequisites

- **VMware**: The VMs that you plan to migrate must be managed by vCenter Server version 5.5, 6.0, or 6.5. Additionally, you need one ESXi host running version 5.0 or later to deploy the collector VM.
- **vCenter account**: You need a read-only account to access vCenter Server. Azure Migrate uses this account to discover the on-premises VMs.
- **Permissions**: In vCenter Server, you need permissions to create a VM by importing a file in OVA format.
- **Statistics settings**: The statistics settings for vCenter Server should be set to level 3 before you start deployment. If the level is lower than 3, the assessment will work, but performance data for storage and network won't be collected. The size recommendations in this case will be based on performance data for CPU and memory, and configuration data for disk and network adapters.

## Plan Azure Migrate projects

Plan your discoveries and assessments based on the following limits:

| **Entity** | **Machine limit** |
| ---------- | ----------------- |
| Project    | 1,500             |
| Discovery  | 1,500             |
| Assessment | 1,500             |

<!--
- If you have fewer than 400 machines to discover and assess, you need a single project and a single discovery. Depending on your requirements, you can either assess all the machines in a single assessment or split the machines into multiple assessments.
- If you have 400 to 1,000 machines to discover, you need a single project with a single discovery. But you will need multiple assessments to assess these machines, because a single assessment can hold up to 400 machines.
- If you have 1,001 to 1,500 machines, you need a single project with two discoveries in it.
- If you have more than 1,500 machines, you need to create multiple projects, and perform multiple discoveries, according to your requirements. For example:
    - If you have 3,000 machines, you can set up two projects with two discoveries, or three projects with a single discovery.
    - If you have 5,000 machines, you can set up four projects: three with a discovery of 1,500 machines, and one with a discovery of 500 machines. Alternatively, you can set up five projects with a single discovery in each one.
      -->

## Plan multiple discoveries

You can use the same Azure Migrate collector to do multiple discoveries to one or more projects. Keep these planning considerations in mind:

- When you do a discovery by using the Azure Migrate collector, you can set the discovery scope to a vCenter Server folder, datacenter, cluster, or host.
- To do more than one discovery, verify in vCenter Server that the VMs you want to discover are in folders, datacenters, clusters, or hosts that support the limitation of 1,500 machines.
- We recommend that for assessment purposes, you keep machines with interdependencies within the same project and assessment. In vCenter Server, make sure that dependent machines are in the same folder, datacenter, or cluster for the assessment.


## Create a project

Create an Azure Migrate project in accordance with your requirements:

1. In the Azure portal, select **Create a resource**.
2. Search for **Azure Migrate**, and select the service **Azure Migrate (preview)** in the search results. Then select **Create**.
3. Specify a project name and the Azure subscription for the project.
4. Create a new resource group.
5. Specify the location in which you want to create the project, and then select **Create**. Note that you can still assess your VMs for a different target location. The location specified for the project is used to store the metadata gathered from on-premises VMs.

## Set up the collector appliance

Azure Migrate creates an on-premises VM known as the collector appliance. This VM discovers on-premises VMware VMs, and it sends metadata about them to the Azure Migrate service. To set up the collector appliance, you download an OVA file and import it to the on-premises vCenter Server instance.

### Download the collector appliance

If you have multiple projects, you need to download the collector appliance only once to vCenter Server. After you download and set up the appliance, you run it for each project, and you specify the unique project ID and key.

1. In the Azure Migrate project, select **Getting Started** > **Discover & Assess** > **Discover Machines**.
2. In **Discover machines**, select **Download**, to download the OVA file.
3. In **Copy project credentials**, copy the ID and key for the project. You need these when you configure the collector.


### Verify the collector appliance

Check that the OVA file is secure before you deploy it:

1. On the machine to which you downloaded the file, open an administrator command window.

2. Run the following command to generate the hash for the OVA:

   ```C:\>CertUtil -HashFile <file_location> [Hashing Algorithm]```

   Example usage: ```C:\>CertUtil -HashFile C:\AzureMigrate\AzureMigrate.ova SHA256```

3. Make sure that the generated hash matches the following settings.

    For OVA version 1.0.9.8

    **Algorithm** | **Hash value**
    --- | ---
    MD5 | b5d9f0caf15ca357ac0563468c2e6251
    SHA1 | d6179b5bfe84e123fabd37f8a1e4930839eeb0e5
    SHA256 | 09c68b168719cb93bd439ea6a5fe21a3b01beec0e15b84204857061ca5b116ff

    For OVA version 1.0.9.7

    **Algorithm** | **Hash value**
    --- | ---
    MD5 | d5b6a03701203ff556fa78694d6d7c35
    SHA1 | f039feaa10dccd811c3d22d9a59fb83d0b01151e
    SHA256 | e5e997c003e29036f62bf3fdce96acd4a271799211a84b34b35dfd290e9bea9c

    For OVA version 1.0.9.5

    **Algorithm** | **Hash value**
    --- | ---
    MD5 | fb11ca234ed1f779a61fbb8439d82969
    SHA1 | 5bee071a6334b6a46226ec417f0d2c494709a42e
    SHA256 | b92ad637e7f522c1d7385b009e7d20904b7b9c28d6f1592e8a14d88fbdd3241c  

    For OVA version 1.0.9.2

    **Algorithm** | **Hash value**
    --- | ---
    MD5 | 7326020e3b83f225b794920b7cb421fc
    SHA1 | a2d8d496fdca4bd36bfa11ddf460602fa90e30be
    SHA256 | f3d9809dd977c689dda1e482324ecd3da0a6a9a74116c1b22710acc19bea7bb2  

## Create the collector VM

Import the downloaded file to vCenter Server:

1. In the vSphere Client console, select **File** > **Deploy OVF Template**.

    ![Deploy OVF](./media/how-to-scale-assessment/vcenter-wizard.png)

2. In the Deploy OVF Template Wizard > **Source**, specify the location of the OVA file.
3. In **Name** and **Location**, specify a friendly name for the collector VM, and the inventory object in which the VM
   will be hosted.
4. In **Host/Cluster**, specify the host or cluster on which the collector VM will run.
5. In storage, specify the storage destination for the collector VM.
6. In **Disk Format**, specify the disk type and size.
7. In **Network Mapping**, specify the network to which the collector VM will connect. The network needs internet connectivity to send metadata to Azure.
8. Review and confirm the settings, and then select **Finish**.

## Identify the ID and key for each project

If you have multiple projects, be sure to identify the ID and key for each one. You need the key when you run the collector to discover the VMs.

1. In the project, select **Getting Started** > **Discover & Assess** > **Discover Machines**.
2. In **Copy project credentials**, copy the ID and key for the project.
    ![Copy project credentials](./media/how-to-scale-assessment/copy-project-credentials.png)

## Set the vCenter statistics level
Following is the list of performance counters that are collected during the discovery. The counters are by default available at various levels in vCenter Server.

We recommend that you set the highest common level (3) for the statistics level so that all the counters are collected correctly. If you have vCenter set at a lower level, only a few counters might be collected completely, with the rest set to 0. The assessment might then show incomplete data.

The following table also lists the assessment results that will be affected if a particular counter is not collected.

| Counter                                 | Level | Per-device level | Assessment impact                    |
| --------------------------------------- | ----- | ---------------- | ------------------------------------ |
| cpu.usage.average                       | 1     | NA               | Recommended VM size and cost         |
| mem.usage.average                       | 1     | NA               | Recommended VM size and cost         |
| virtualDisk.read.average                | 2     | 2                | Disk size, storage cost, and VM size |
| virtualDisk.write.average               | 2     | 2                | Disk size, storage cost, and VM size |
| virtualDisk.numberReadAveraged.average  | 1     | 3                | Disk size, storage cost, and VM size |
| virtualDisk.numberWriteAveraged.average | 1     | 3                | Disk size, storage cost, and VM size |
| net.received.average                    | 2     | 3                | VM size and network cost             |
| net.transmitted.average                 | 2     | 3                | VM size and network cost             |

> [!WARNING]
> If you have just set a higher statistics level, it will take up to a day to generate the performance counters. So, we recommend that you run the discovery after one day.

## Run the collector to discover VMs

For each discovery that you need to perform, you run the collector to discover VMs in the required scope. Run the discoveries one after the other. Concurrent discoveries aren't supported, and each discovery must have a different scope.

1.  In the vSphere Client console, right-click the VM > **Open Console**.
2.  Provide the language, time zone, and password preferences for the appliance.
3.  On the desktop, select the **Run collector** shortcut.
4.  In the Azure Migrate collector, open **Set up prerequisites** and then:

    a. Accept the license terms, and read the third-party information.

    The collector checks that the VM has internet access.

    b. If the VM accesses the internet via a proxy, select **Proxy settings**, and specify the proxy address and listening port. Specify credentials if the proxy needs authentication.

    The collector checks that the collector service is running. The service is installed by default on the collector VM.

    c. Download and install VMware PowerCLI.

5.  In **Specify vCenter Server details**, do the following:
    - Specify the name (FQDN) or IP address of vCenter Server.
    - In **User name** and **Password**, specify the read-only account credentials that the collector will use to discover VMs in vCenter Server.
    - In **Select scope**, select a scope for VM discovery. The collector can discover only VMs within the specified scope. Scope can be set to a specific folder, datacenter, or cluster. It shouldn't contain more than 1,000 VMs.

6.  In **Specify migration project**, specify the ID and key for the project. If you didn't copy them, open the Azure portal from the collector VM. On the project's **Overview** page, select **Discover Machines** and copy the values.  
7.  In **View collection progress**, monitor the discovery process and check that metadata collected from the VMs is in scope. The collector provides an approximate discovery time.


### Verify VMs in the portal

Discovery time depends on how many VMs you are discovering. Typically, for 100 VMs, discovery finishes around an hour after the collector finishes running.

1. In the Migration Planner project, select **Manage** > **Machines**.
2. Check that the VMs you want to discover appear in the portal.


## Next steps

- Learn how to [create a group](how-to-create-a-group.md) for assessment.
- [Learn more](concepts-assessment-calculation.md) about how assessments are calculated.
