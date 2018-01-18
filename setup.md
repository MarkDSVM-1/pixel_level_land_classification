# Prerequisties and setup

In this section of the tutorial on [pixel-level land classification from aerial imagery](https://github.com/Azure/pixel_level_land_classification), we describe the steps needed to create an Azure Batch AI cluster with access to all necessary files to complete this tutorial. Once you have completed this section, you'll be ready to [train a model from scratch](./train.md) using our sample data and provided scripts.

## Prerequisites

This tutorial will require an [Azure subscription](https://azure.microsoft.com/en-us/free/) with sufficient quota to create a storage account and two NC6 (single-GPU) VMs as a Batch AI cluster. This tutorial will likely take two hours to complete on the first pass.

You will also need to install the following programs:
- [Azure CLI 2.0](https://docs.microsoft.com/cli/azure/install-azure-cli)
- [AzCopy](https://docs.microsoft.com/azure/storage/common/storage-use-azcopy)

These programs are available for Windows and Linux. The commands included in this tutorial were written and tested in Windows, but readers will likely find it straightforward to adapt for Linux.

Once these programs are installed, open a command line interface and check that the binaries are available on the system path by issuing the commands below:
```
az
azcopy
```

If you prefer not to install these programs locally, you may instead provision an [Azure Data Science Virtual Machine](https://docs.microsoft.com/azure/machine-learning/data-science-virtual-machine/provision-vm). (Both programs are pre-installed on these VMs and available on the system path.)

### Prepare to use the Azure CLI

In your command line interface, execute the following command. The output will contain a URL and token that you must visit to authenticate your login.
```
az login
```

You will now indicate which Azure subscription should be charged for the resources you create in this tutorial. List all Azure subscriptions associated with your account:
```
az account list
```

Identify the subscription of interest in the JSON-formatted output. Copy its "id" value into the bracketed expression in the command below, then issue the command to set the current subscription.
```
az account set -s [subscription id]
```

Register the Batch/BatchAI providers and grant Batch AI "Network Contributor" access on your subscription using the following commands. Note that you will need to copy your subscription's id into the bracketed expression before executing the command.
```
az provider register -n Microsoft.Batch
az provider register -n Microsoft.BatchAI
az role assignment create --scope /subscriptions/[subscription id] --role "Network Contributor" --assignee 9fcb3732-5f52-4135-8c08-9d4bbaf203ea
```

It may take ~10 minutes for the provider registration process to complete. You may proceed with the tutorial in the meantime.

## Create the necessary Azure resources

### Create an Azure resource group

We will create all resources for this tutorial in a single resource group, so that you may easily delete them when finished. Choose a name for your resource group and insert it into the bracketed expression below, then issue the commands:
```
az configure --defaults location=eastus
set AZURE_RESOURCE_GROUP=[resource group name]
az group create --name %AZURE_RESOURCE_GROUP%
az configure --defaults group=%AZURE_RESOURCE_GROUP%
```

### Create an Azure storage account and populate it with files

We will create an Azure storage account to hold training and evaluation data, scripts, and output files. Choose a unique name for this storage account and insert it into the bracketed expression below. Then, issue the following commands to create your storage account and store its randomly-assigned access key:
```
set STORAGE_ACCOUNT_NAME=[storage account name]
az storage account create --name %STORAGE_ACCOUNT_NAME% --sku Standard_LRS
for /f "delims=" %a in ('az storage account keys list --account-name %STORAGE_ACCOUNT_NAME% --query "[0].value"') do @set STORAGE_ACCOUNT_KEY=%a
```

With the commands below, we will create an Azure File Share to hold setup and job-specific logs, as well as an Azure Blob container for fast file I/O during model training and evaluation. Then, we'll use AzCopy to copy the necessary data files for this tutorial to your own storage account.  Note that we will copy over only a subset of the available data, to save time and resources.
```
az storage share create --account-name %STORAGE_ACCOUNT_NAME% --account-key %STORAGE_ACCOUNT_KEY% --name batchai
az storage container create --account-name %STORAGE_ACCOUNT_NAME% --account-key %STORAGE_ACCOUNT_KEY% --name blobfuse
AzCopy /Source:https://aiforearthcollateral.blob.core.windows.net/imagesegmentationtutorial /SourceSAS:"?st=2018-01-16T10%3A40%3A00Z&se=2028-01-17T10%3A40%3A00Z&sp=rl&sv=2017-04-17&sr=c&sig=KeEzmTaFvVo2ptu2GZQqv5mJ8saaPpeNRNPoasRS0RE%3D" /Dest:https://%STORAGE_ACCOUNT_NAME%.blob.core.windows.net/blobfuse /DestKey:%STORAGE_ACCOUNT_KEY% /S
```

Expect the copy step to take 5-10 minutes.

### Create an Azure Batch AI cluster

We will create an Azure Batch AI cluster containing two NC6 Ubuntu DSVMs. This two-GPU cluster will be used to train our model and then apply it to previously-unseen data. Before executing the command below, ensure that the `cluster.json` file provided in this repository (which specifies the Python packages that should be installed during setup) has been downloaded to your computer and is available on the path.
```
az batchai cluster create -n batchaidemo -u lcuser -p lcpassword --afs-name batchai --image UbuntuDSVM --vm-size STANDARD_NC6 --max 2 --min 2 --storage-account-name %STORAGE_ACCOUNT_NAME% --container-name blobfuse --container-mount-path blobfuse -c cluster.json
```

It will take approximately ten minutes for cluster creation to complete. You can check on progress of the provisioning process using the command below: when provisioning is complete, you should see that the "errors" field is null and that your cluster has two "idle" nodes.
```
az batchai cluster show -n batchaidemo
```