﻿<properties
   pageTitle="Continuous integration for Service Fabric | Microsoft Azure"
   description="Get an overview of how to set up continuous integration for a Service Fabric application by using Visual Studio Team Services."
   services="service-fabric"
   documentationCenter="na"
   authors="cawams"
   manager="timlt"
   editor="" />
<tags
   ms.service="multiple"
   ms.devlang="dotnet"
   ms.topic="article"
   ms.tgt_pltfrm="na"
   ms.workload="multiple"
   ms.date="10/16/2015"
   ms.author="cawa" />

# Set up continuous integration for an Azure Service Fabric application by using Visual Studio Team Services

This article takes you through setting up continuous integration for a Service Fabric application by using Visual Studio Team Services so that your application can be built, packaged, and deployed in an automated fashion. Note that this document reflects the current experience and is expected to change as development progresses. Also, these instructions re-create the cluster from scratch every time.

## Prerequisites

To get started, set up your project on Visual Studio Team Services:

1. If you haven't already, create a Team Services account by using your [Microsoft account](http://www.microsoft.com/account).
2. Create a new project on Team Services by using the Microsoft account.
3. Push the source for your new or existing Service Fabric app to this project.

See [Connect to Visual Studio](https://www.visualstudio.com/get-started/setup/connect-to-visual-studio-online) for more information on working with Team Services projects.

## Setup steps

### Set up authentication for automation

Before you can set up the build machine, you need to create a "Service Principal" which automation uses to authenticate to Azure. You also need to create a certificate and upload it to a key vault, because Azure Key Vault does not support Service Principal authentication. You can perform these steps from any machine. Your dev machine is a good choice.

### Install Azure PowerShell and sign in

1.	Install Azure PowerShell.
    - Install PowerShellGet. To do this, install [Windows Management Framework 5.0](http://www.microsoft.com/download/details.aspx?id=48729), which includes PowerShellGet.

    >[AZURE.NOTE] You can skip this step if you are running Windows 10 with the latest updates.

2.	Install and update the AzureRM module.
    1.  If you have any previous version of Azure PowerShell installed, remove it. Right-click on the start button, and then select **Add/Remove Programs**. Search for "Azure PowerShell" and uninstall it.
    2.  Launch a PowerShell command prompt.
    3.	Install the AzureRM module by using the command `Install-Module AzureRM`.
    4.	Update the AzureRM module by using the command `Update-AzureRM`.

3.	Disable (or enable) Azure data collection.

    Azure cmdlets will prompt you to opt in or out of data collection until you make a choice. These prompts will block automation while waiting for user input. To suppress these prompts by making a choice ahead of time, run one of the following commands:

    - Enable-AzureRmDataCollection
    - Disable-AzureRmDataCollection

4.	Sign in to Azure PowerShell.
    1. Launch an admin PowerShell prompt and run the command `Login-AzureRMAccount`.
    2. In the dialog that appears, enter your Azure credentials.
    3. Run the command `Get-AzureRmSubscription`.
    4. Find the subscription you want to use and then run the command `Select-AzureRmSubscription -SubscriptionId <id for your subscription>`."

### Create a Service Principal

1.	Extract [ServiceFabricCIManualScripts.zip](http://go.microsoft.com/fwlink/?LinkId=703773) to a folder on this machine.
2.	In an admin PowerShell command prompt, change to the directory to which you extracted ServiceFabricCIManualScripts.zip.
3.	Choose a password for the Service Principal by using the following command. Remember this password, because it will be used as a build variable.

    ```
    $password = Read-Host -AsSecureString
    ```
4.	Run the PowerShell script Create-ServicePrincipal.ps1 with the following parameters:

    |Parameter|Value|
    |---|---|
    |ServicePrincipalDisplayName|Any name.|
    |ServicePrincipalHomePage|Any URI. Doesn't have to actually exist.|
    |ServicePrincipalIdentifierUri|Any unique URI. Doesn't have to actually exist.|
    |ServicePrincipalSecurePassword|$password|

    When the script finishes, it outputs the following three values. Note the values, because they are used as build variables.

    - ServicePrincipalId
    - ServicePrincipalTenantId
    - ServicePrincipalSubscriptionId

### Create a certificate and upload it to a new key vault

1.	In an admin PowerShell prompt, change to the directory to which you extracted ServiceFabricCIManualScripts.zip.
2.	Run the PowerShell script CreateAndUpload-Certificate.ps1 with the following parameters.

    |Parameter|Value|
    |---|---|
    |ServiceFabricKeyVaultLocation|Any value. Must match the location in which you plan to create the cluster.|
    |ServiceFabricCertificateSecretName|Any value.|
    |ServiceFabricSecureCertificatePassword|Any vlaue. Is used when you import the cert on your build machine.|
    |ServiceFabricKeyVaultResourceGroupName|Any value. However, don't use the resource group name that you plan to use for your cluster.|
    |ServiceFabricKeyVaultName|Any value.|
    |ServiceFabricPfxFileOutputPath|Any value. This file is used to import the cert onto your build machine.|

    When the script finishes, it outputs the following three values. Note these values, because they are used as build variables.

    - ServiceFabricCertificateThumbprint
    - ServiceFabricKeyVaultId
    - ServiceFabricCertificateSecretId


## Set up your build machine

Note that the build definition you create from these instructions doesn't support parallel builds, even on separate machines. (This is because each build would compete for the same resource group/cluster.) If you want to run multiple build agents, you will need to modify the following instructions/scripts to prevent this interference.

### Install Visual Studio 2015

1.	If you have already provisioned a machine (or plan to provide your own), install [Visual Studio 2015](https://www.visualstudio.com/downloads/download-visual-studio-vs.aspx) on that machine.
2.	If you don't yet have a machine, you can quickly provision an Azure virtual machine (VM) with Visual Studio 2015 pre-installed. To do this:
    1.	Sign in to the [Azure portal](http://portal.azure.com).
    2.	Choose the **New** command in the top-left corner of the screen.
    3.	Choose **Marketplace**.
    4.	Search for **Visual Studio 2015**.
    5.	Choose **Compute** > **Virtual Machine** > **From Gallery**.
    6.	Choose the image **Visual Studio Enterprise 2015 With Azure SDK 2.7 on Windows Server 2012 R2**.

        >[AZURE.NOTE] The Azure SDK isn't a required component, but there aren't any images available that have only Visual Studio 2015 installed.

    7.	Follow the instructions in the dialog to create your VM. (We recommend that you choose a D-series VM for best disk and CPU performance.)

### Install Service Fabric SDK

Install the [Service Fabric SDK](https://azure.microsoft.com/campaigns/service-fabric/).

### Register Service Fabric SDK's NuGet repository

1.	Create the directory `%SYSTEMDRIVE%\Windows\ServiceProfiles\LocalService\AppData\Roaming\NuGet` on the build machine (if it does not already exist).
2.	If a file called NuGet.config already exists in this directory, open it and add the following element as a child of `<packageSources>`.

    ```
    <add key="Service Fabric SDK" value="<path to service fabric SDK>\packages" />`
    ```

3.	If NuGet.config does not already exist, create it with the following contents. Replace `<path to service fabric SDK>` with the path to Service Fabric SDK on the build machine.

>[AZURE.NOTE] By default, `<path to service fabric SDK>` is `%ProgramFiles%\Microsoft SDKs\Service Fabric`.

    NuGet.config:

    ```
    <?xml version="1.0" encoding="utf-8"?>
    <configuration>
      <packageSources>
        <!-- Add this repository to the list of available repositories -->
        <add key="Service Fabric SDK" value="<path to service fabric SDK>\packages" />
      </packageSources>
    </configuration>
    ```

### Install Azure PowerShell

To install Azure PowerShell, follow the steps in the previous section **Install Azure PowerShell and sign in**. Skip step 4, **Sign in to Azure PowerShell**.

### Register the Azure PowerShell modules with the Local Service account

1. Press Win + R, and then type **regedit** and hit enter.
2. Right-click on the node `HKEY_Users\.Default\Environment` and select **New > Expandable String Value**.
3. Enter `PSModulePath` for the name, and `%PROGRAMFILES%\WindowsPowerShell\Modules` for the value.

>[AZURE.NOTE] Do this *before* you start the build agent, or it will not pick up the new environment variable.

### Import your automation certificate

1. Import the certificate onto your build machine. To do this:
    1. Copy the PFX file created by the script CreateAndUpload-Certificate.ps1 to your build machine.
    2. Open an admin PowerShell account and issue the following commands, using the password you passed to GenerateCertificate.ps1 earlier:

        ```
        $password = Read-Host -AsSecureString
        Import-PfxCertificate -FilePath <path/to/cert.pfx> -CertStoreLocation Cert:\LocalMachine\My -Password $password -Exportable
        ```  

2. Run the certificate manager.

    1. Open Windows Control Panel. Right-click the Start button and choose **Control Panel**.
    2. Search for **certificate**.
    3. Choose **Administrative Tools** > **Manage computer certificates**.

3. Grant Local Service account permission to use your automation certificate.

    1.	Under **Certificates - Local Computer**, expand **Personal**, and then choose **Certificates**.
    2.	Find your certificate in the list.
    3.	Right-click your certificate and then choose **All Tasks** > **Manage Private Keys**.
    4.	Choose the **Add** button, enter **Local Service** and then choose **Check Names**.
    5.	Choose the **OK** button and then close the certificate manager.

![Picture of the Windows certificate manager UI](media/service-fabric-set-up-continuous-integration/windows-certificate-manager.png)

### Register your build agent

1.	Download agent.zip. To do this:
    1.	Sign in to your team project, such as **https://[your-Team-Services-account-name].visualstudio.com**.
    1.	Choose the 'gear' icon in the upper-right corner of your screen.
    1.	From the control panel, choose the **Agent pools** tab.
    1.	Choose **Download agent** to download the agent.zip file.
    1.	Copy agent.zip to the build machine that you created earlier.
    1.	Unzip agent.zip to `C:\agent` (or any location with a short path) on your build machine.

        >[AZURE.NOTE] If you plan on building ASP.NET 5 web services, we recommend that you choose the shortest name possible for this folder to avoid running into **PathTooLongExceptions** errors during deployment.

2.	From an admin PowerShell command prompt, run `C:\agent\ConfigureAgent.ps1`. The script prompts you for the following parameters:

    |Parameter|Value|
    |---|---|
    |Agent Name|Accept the default value, `Agent-[machine name]`.)
    |TFS Url|Enter the URL to your team project, such as `https://[your-Team-Services-account-name].visualstudio.com`.
    |Agent Pool|Enter the name of your agent pool. (If you haven't created an agent pool, accept the default value.)|
    |Work folder|Accept the default value. This is the folder where the build agent will actually build your application. Note: If you plan on building ASP.NET 5 Web Services, we recommend that you choose the shortest name possible for this folder to avoid running into PathTooLongExceptions errors during deployment.|
    |Install as Windows Service?|Default value is N. Change the value to Y.|
    |User account to run the service|Accept the default value, `NT AUTHORITY\LocalService)`.|
    |Un-configure existing agent?|Accept the default value, **N**.|

3. You will be prompted for credentials. Enter the credentials for your Microsoft account that has rights to your team project.
4. Verify that your build agent was registered. To do this:

    1. Go back to your web browser (should be at the page `https://[your-Team-Services-account-name].visualstudio.com/_admin/_AgentPool`) and then refresh the page.
    1. Choose the agent pool that you selected when running ConfigureAgent.ps1 earlier.
    1. Verify that your build agent shows up in the list and has a green status highlight. If the highlight is red, the build agent is having trouble connecting to Team Services.

![The **Agent pools** tab of the control panel](media/service-fabric-set-up-continuous-integration/vso-configured-agent.png)


## Set up your build definition

### Add the continuous integration scripts to source control for your application

1.	Extract [ServiceFabricCIAutomationScripts.zip](http://go.microsoft.com/fwlink/?LinkId=703775) to any folder in source control.
2.	Check in the resulting files.

### Create the build definition

1.	Create an empty build definition. To do this:

    1.	Open your project in Visual Studio Team Services.
    1.	Choose the **Build** tab.
    1.	Choose the green **+** sign to create a new build definition.
    1.	Choose **Empty** and then choose the **Next** button.
    1.  Verify that the right repository and branch are selected.
    1.  Select the agent queue to which you registered your build agent, and check the **Continuous Integration** check box.
2.	On the **Variables** tab, create the following variables with these values:

    |Variable|Value|Secret|Allow at Queue Time|
    |---|---|---|---|
    |BuildConfiguration|Release||X|
    |BuildPlatform|x64|||
    |ServicePrincipalPassword|The password that you passed to CreateServicePrincipal.ps1|X||
    |ServicePrincipalId|From the output of CreateServicePrincipal.ps1|||
    |ServicePrincipalTenantId|From the output of CreateServicePrincipal.ps1|||
    |ServicePrincipalSubscriptionId|From the output of CreateServicePrincipal.ps1|||
    |ServiceFabricCertificateThumbprint|From the output of GenerateCertificate.ps1|||
    |ServiceFabricKeyVaultId|From the output of GenerateCertificate.ps1|||
    |ServiceFabricCertificateSecretId|From the output of GenerateCertificate.ps1|||
    |ServiceFabricClusterResourceGroupName|Any name you want.|||
    |ServiceFabricClusterName|Any name you want.|||
    |ServiceFabricClusterLocation|Any name that matches the location of your key vault.|||
    |ServiceFabricClusterAdminPassword|Any name you want.|X||
    |ServiceFabricClusterResourceGroupTemplateFilePath|`<path/to/extracted/automation/scripts/ArmTemplate-Full-3xVM-Secure.json>`|||
    |ServiceFabricPublishProfilePath|`<path/to/your/publish/profiles/MyPublishProfile.xml>` Note: The connection endpoint in your publish profile will be ignored. The connection endpoint for your temporary cluster is used instead.|||
    |ServiceFabricDeploymentScriptPath|`<path/to/Deploy-FabricApplication.ps1>`|||
    |ServiceFabricApplicationProjectPath|`<path/to/your/fabric/application/project/folder>` This should be the folder containing your .sfproj file.||||

3.	On the **Triggers** tab, select the **Continuous Integration** and **Batch changes** options.
4.	On the **General** tab, choose the queue to which you registered your build agents.
5.	Save the build definition and give it a name. (You can change this name later if you want.)

### Add a "Build" step

1.	On the **Build** tab, choose the **Add build step…** command."
2.	Choose **Build** > **MSBuild**.
3.	Choose the pencil icon by the build step's name and rename it to **Build**.
4.	Choose the **…** button next to the **Solution** field and then choose your .sln file.
5.	Enter `$(BuildPlatform)` for **Platform**.
6.	Enter `$(BuildConfiguration)` for **Configuration**.
7.	Select the **Restore NuGet Packages** check box (if it isn't already selected).
8.	Save the build definition.

### Add a "Package" step

1.	On the **Build** tab, choose the **Add build step…** command.
2.	Choose **Build** > **MSBuild**.
3.	Choose the pencil icon next to the build step's name and rename it to **Package**.
4.	Choose the **…** button next to the **Solution** field and then select your application project's .sfproj file.
5.	Enter `$(BuildPlatform)` for **Platform**.
6.	Enter `$(BuildConfiguration)` for **Configuration**.
7.	Enter `/t:Package` for **MSBuild Arguments**.
8.	Clear the **Restore NuGet Packages** check box (if it isn't already cleared).
9.	Save the build definition.

### Add a "Remove Cluster Resource Group" step

If a previous build did not clean up after itself (such as if the build was cancelled before it could clean up), there may be an existing resource group that will conflict with the new one. To avoid conflicts, clean up any leftover resource group (and its associated resources) before creating a new one:

1.	On the **Build** tab, choose the **Add build step…** command.
2.	Choose **Utility** > **PowerShell**.
3.	Choose the pencil icon next to the build step's name and then rename it to **Remove Cluster Resource Group**.
4.	Choose the **…** command next to **Script filename**. Navigate to where you extracted the automation scripts and then choose **Remove-ClusterResourceGroup.ps1**.
5.	For **Arguments**, enter `-ServicePrincipalPassword "$(ServicePrincipalPassword)"`.
6.	Save the build definition.

### Add a "Provision and Deploy to Secure Cluster" step

1.	On the **Build** tab, choose the **Add build step…** command.
2.	Choose **Utility** > **PowerShell**.
3.	Choose the pencil icon next to the build step's name and then rename it to **Provision and Deploy to Secure Cluster**.
4.	Choose the **…** button next to **Script filename**. Navigate to where you extracted the automation scripts and then choose **ProvisionAndDeploy-SecureCluster.ps1**.
5.	For "Arguments", enter `-ServicePrincipalPassword "$(ServicePrincipalPassword)" -ServiceFabricClusterAdminPassword "$(ServiceFabricClusterAdminPassword)"`
6.	Save the build definition.

### Add a "Remove Cluster Resource Group" step

Now that you're done with the temporary cluster, you should clean it up. If you don't do this, you'll continue to be charged for the temporary cluster. This procedure removes the resource group, which removes the cluster and all other resources in the group:

1.	On the **Build** tab, choose the **Add build step…** command.
2.	Choose **Utility** > **PowerShell**.
3.	Choose the pencil icon next to the build step's name and then rename it to **Remove Cluster Resource Group**.
4.	Choose the **…** button next to **Script filename**. Navigate to where you extracted the automation scripts and then choose **RemoveClusterResourceGroup.ps1**.
5.	For "Arguments", enter `-ServicePrincipalPassword "$(ServicePrincipalPassword)`."
6.	Under **Control Options**, select the **Always Run** check box.
7.	Save the build definition.

### Try it!

Choose the **Queue Build** command to start a build. Builds will also be triggered upon check in.


## Alternative solutions

The previous instructions create a new cluster for each build and remove it at the end of the build. If you'd rather have each build perform an application upgrade (to an existing cluster) instead, do the following steps:

1.	Manually create a test cluster through the Azure portal or Azure PowerShell. You can refer to the "ProvisionAndDeploy-SecureCluster.ps1" script as a reference.
2.	Configure your publish profile to support application upgrade by following [these instructions](service-fabric-visualstudio-configure-upgrade.md).

3.	Replace the **Provision and Deploy to Secure Cluster** step with a step that calls Deploy-FabricApplication.ps1 directly (and passes it your publish profile).
4.	Remove both of the **Remove Cluster Resource Group** build steps from your build definition.

## Next steps

To learn more about continuous integration with Service Fabric applications, read the following articles:

- [Build documentation home](https://msdn.microsoft.com/Library/vs/alm/Build/overview)
- [Deploy a build agent](https://msdn.microsoft.com/Library/vs/alm/Build/agents/windows)
- [Create and configure a build definition](https://msdn.microsoft.com/Library/vs/alm/Build/vs/define-build)
