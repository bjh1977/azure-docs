---
title: 'Transform data using Spark in Azure Data Factory | Microsoft Docs'
description: 'This tutorial provides step-by-step instructions for transforming data by using Spark Activity in Azure Data Factory.'
services: data-factory
documentationcenter: ''
author: shengcmsft
manager: jhubbard
editor: spelluru

ms.service: data-factory
ms.workload: data-services
ms.tgt_pltfrm: na
ms.devlang: na
ms.topic: get-started-article
ms.date: 10/06/2017
ms.author: shengc
---
# Transform data in the cloud by using Spark activity in Azure Data Factory

[!INCLUDE [data-factory-what-is-include-md](../../includes/data-factory-what-is-include.md)]

#### This tutorial

> [!NOTE]
> This article applies to version 2 of Data Factory, which is currently in preview. If you are using version 1 of the Data Factory service, which is generally available (GA), see [documentation for Data Factory version 1](v1/data-factory-copy-data-from-azure-blob-storage-to-sql-database.md).

In this tutorial, you use Azure PowerShell to create a Data Factory pipeline that transforms data using Spark Activity and an on-demand HDInsight linked service. You perform the following steps in this tutorial:

> [!div class="checklist"]
> * Create a data factory. 
> * Author and deploy linked services.
> * Author and deploy a pipeline. 
> * Start a pipeline run.
> * Monitor the pipeline run.

If you don't have an Azure subscription, create a [free](https://azure.microsoft.com/free/) account before you begin.

## Prerequisites
* **Azure Storage account**. You create a python script and an input file, and upload them to the Azure storage. The output from the spark program is stored in this storage account. The on-demand Spark cluster uses the same storage account as its primary storage.  
* **Azure PowerShell**. Follow the instructions in [How to install and configure Azure PowerShell](/powershell/azure/install-azurerm-ps).


### Upload python script to your Blob Storage account
1. Create a python file named **WordCount_Spark.py** with the following content: 

    ```python
    import sys
    from operator import add
    
    from pyspark.sql import SparkSession
    
    def main():
        spark = SparkSession\
            .builder\
            .appName("PythonWordCount")\
            .getOrCreate()
    		
        lines = spark.read.text("wasbs://adftutorial@<storageaccountname>.blob.core.windows.net/spark/inputfiles/minecraftstory.txt").rdd.map(lambda r: r[0])
        counts = lines.flatMap(lambda x: x.split(' ')) \
            .map(lambda x: (x, 1)) \
            .reduceByKey(add)
        counts.saveAsTextFile("wasbs://adftutorial@<storageaccountname>.blob.core.windows.net/spark/outputfiles/wordcount")
        
        spark.stop()
    
    if __name__ == "__main__":
    	main()
    ```
2. Replace **&lt;storageAccountName&gt;** with the name of your Azure Storage account. Then, save the file. 
3. In your Azure Blob Storage, create a container named **adftutorial** if it does not exist. 
4. Create a folder named **spark**.
5. Create a subfolder named **script** under **spark** folder. 
6. Upload the **WordCount_Spark.py** file to the **script** subfolder. 


### Upload the input file
1. Create a file named **minecraftstory.txt** with some text. The spark program counts the number of words in this text. 
2. Create a subfolder named `inputfiles` in the `spark` folder. 
3. Upload the `minecraftstory.txt` to the `inputfiles` subfolder. 

## Author linked services
You author two Linked Services in this section: 
	
- An Azure Storage Linked Service that links an Azure Storage account to the data factory. This storage is used by the on-demand HDInsight cluster. It also contains the Spark script to be executed. 
- An On-Demand HDInsight Linked Service. Azure Data Factory automatically creates a HDInsight cluster, run the Spark program, and then deletes the HDInsight cluster after it's idle for a pre-configured time. 

### Azure Storage linked service
Create a JSON file using your preferred editor, copy the following JSON definition of an Azure Storage linked service, and then save the file as **MyStorageLinkedService.json**.  

```json
{
    "name": "MyStorageLinkedService",
    "properties": {
      "type": "AzureStorage",
      "typeProperties": {
        "connectionString": {
          "value": "DefaultEndpointsProtocol=https;AccountName=<storageAccountName>;AccountKey=<storageAccountKey>",
          "type": "SecureString"
        }
      }
    }
}
```
Update the &lt;storageAccountName&gt; and &lt;storageAccountKey&gt; with the name and key of your Azure Storage account. 


### On-demand HDInsight linked service
Create a JSON file using your preferred editor, copy the following JSON definition of an Azure HDInsight linked service, and save the file as **MyOnDemandSparkLinkedService.json**.  

```json
{
    "name": "MyOnDemandSparkLinkedService",
    "properties": {
      "type": "HDInsightOnDemand",
      "typeProperties": {
        "clusterSize": 2,
        "clusterType": "spark",
        "timeToLive": "00:15:00",
        "hostSubscriptionId": "<subscriptionID> ",
        "servicePrincipalId": "<servicePrincipalID>",
        "servicePrincipalKey": {
          "value": "<servicePrincipalKey>",
          "type": "SecureString"
        },
        "tenant": "<tenant ID>",
        "clusterResourceGroup": "<resourceGroupofHDICluster>",
        "version": "3.6",
        "osType": "Linux",
        "clusterNamePrefix":"ADFSparkSample",
        "linkedServiceName": {
          "referenceName": "MyStorageLinkedService",
          "type": "LinkedServiceReference"
        }
      }
    }
}
```
Update values for the following properties in the linked service definition: 

- **hostSubscriptionId**. Replace &lt;subscriptionID&gt; with the ID of your Azure subscription. The on-demand HDInsight cluster is created in this subscription. 
- **tenant**. Replace &lt;tenantID&gt; with ID of your Azure tenant. 
- **servicePrincipalId**, **servicePrincipalKey**. Replace &lt;servicePrincipalID&gt; and &lt;servicePrincipalKey&gt; with ID and key of your service principal in the Azure Active Directory. This service principal needs to be a member of the Contributor role of the subscription or the resource Group in which the cluster is created. See [create Azure Active Directory application and service principal](../azure-resource-manager/resource-group-create-service-principal-portal.md) for details. 
- **clusterResourceGroup**. Replace &lt;resourceGroupOfHDICluster&gt; with the name of the resource group in which the HDInsight cluster needs to be created. 

> [!NOTE]
> Azure HDInsight has limitation on the total number of cores you can use in each Azure region it supports. For On-Demand HDInsight Linked Service, the HDInsight cluster will be created in the same location of the Azure Storage used as its primary storage. Ensure that you have enough core quotas for the cluster to be created successfully. For more information, see [Set up clusters in HDInsight with Hadoop, Spark, Kafka, and more](../hdinsight/hdinsight-hadoop-provision-linux-clusters.md). 


## Author a pipeline 
In this step, you create a new pipeline with a Spark activity. The activity uses the **word count** sample. Download the contents from this location if you haven't already done so.

Create a JSON file in your preferred editor, copy the following JSON definition of a pipeline definition, and save it as **MySparkOnDemandPipeline.json**. 

```json
{
  "name": "MySparkOnDemandPipeline",
  "properties": {
    "activities": [
      {
        "name": "MySparkActivity",
        "type": "HDInsightSpark",
        "linkedServiceName": {
            "referenceName": "MyOnDemandSparkLinkedService",
            "type": "LinkedServiceReference"
        },
        "typeProperties": {
          "rootPath": "adftutorial/spark",
          "entryFilePath": "script/WordCount_Spark.py",
          "getDebugInfo": "Failure",
          "sparkJobLinkedService": {
            "referenceName": "MyStorageLinkedService",
            "type": "LinkedServiceReference"
          }
        }
      }
    ]
  }
}
```

Note the following points: 

- rootPath points to the spark folder of the adftutorial container. 
- entryFilePath points to the WordCount_Spark.py file in the script sub folder of the spark folder. 


## Create a data factory 
You have authored linked service and pipeline definitions in JSON files. Now, let’s create a data factory, and deploy the linked Service and pipeline JSON files by using PowerShell cmdlets. Run the following PowerShell commands one by one: 

1. Set variables one by one.

    ```powershell
    $subscriptionID = "<subscription ID>" # Your Azure subscription ID
    $resourceGroupName = "ADFTutorialResourceGroup" # Name of the resource group
    $dataFactoryName = "MyDataFactory09102017" # Globally unique name of the data factory
    $pipelineName = "MySparkOnDemandPipeline" # Name of the pipeline
    ```
2. Launch **PowerShell**. Keep Azure PowerShell open until the end of this quickstart. If you close and reopen, you need to run the commands again.

    Run the following command, and enter the user name and password that you use to sign in to the Azure portal:
        
    ```powershell
    Login-AzureRmAccount
    ```        
    Run the following command to view all the subscriptions for this account:

    ```powershell
    Get-AzureRmSubscription
    ```
    Run the following command to select the subscription that you want to work with. Replace **SubscriptionId** with the ID of your Azure subscription:

    ```powershell
    Select-AzureRmSubscription -SubscriptionId "<SubscriptionId>"    
    ```  
3. Create the resource group: ADFTutorialResourceGroup. 

    ```powershell
    New-AzureRmResourceGroup -Name $resourceGroupName -Location "East Us" 
    ```
4. Create the data factory. 

    ```powershell
     $df = Set-AzureRmDataFactoryV2 -Location EastUS -Name $dataFactoryName -ResourceGroupName $resourceGroupName
    ```

    Execute the following command to see the output: 

    ```powershell
    $df
    ```
5. Switch to the folder where you created JSON files, and run the following command to deploy an Azure Storage linked service: 
       
    ```powershell
    Set-AzureRmDataFactoryV2LinkedService -DataFactoryName $dataFactoryName -ResourceGroupName $resourceGroupName -Name "MyStorageLinkedService" -File "MyStorageLinkedService.json"
    ```
6. Run the following command to deploy an on-demand Spark linked service: 
       
    ```powershell
    Set-AzureRmDataFactoryV2LinkedService -DataFactoryName $dataFactoryName -ResourceGroupName $resourceGroupName -Name "MyOnDemandSparkLinkedService" -File "MyOnDemandSparkLinkedService.json"
    ```
7. Run the following command to deploy a pipeline: 
       
    ```powershell
    Set-AzureRmDataFactoryV2Pipeline -DataFactoryName $dataFactoryName -ResourceGroupName $resourceGroupName -Name $pipelineName -File "MySparkOnDemandPipeline.json"
    ```
    
## Start and monitor a pipeline run  

1. Start a pipeline run. It also captures the pipeline run ID for future monitoring.

    ```powershell
    $runId = Invoke-AzureRmDataFactoryV2Pipeline -DataFactoryName $dataFactoryName -ResourceGroupName $resourceGroupName -PipelineName $pipelineName  
    ```
2. Run the following script to continuously check the pipeline run status until it finishes.

    ```powershell
	while ($True) {
	    $result = Get-AzureRmDataFactoryV2ActivityRun -DataFactoryName $dataFactoryName -ResourceGroupName $resourceGroupName -PipelineRunId $runId -RunStartedAfter (Get-Date).AddMinutes(-30) -RunStartedBefore (Get-Date).AddMinutes(30)
	
	    if(!$result) {
	        Write-Host "Waiting for pipeline to start..." -foregroundcolor "Yellow"
	    }
	    elseif (($result | Where-Object { $_.Status -eq "InProgress" } | Measure-Object).count -ne 0) {
	        Write-Host "Pipeline run status: In Progress" -foregroundcolor "Yellow"
	    }
	    else {
	        Write-Host "Pipeline '"$pipelineName"' run finished. Result:" -foregroundcolor "Yellow"
	        $result
	        break
	    }
	    ($result | Format-List | Out-String)
	    Start-Sleep -Seconds 15
	}

	Write-Host "Activity `Output` section:" -foregroundcolor "Yellow"
	$result.Output -join "`r`n"

	Write-Host "Activity `Error` section:" -foregroundcolor "Yellow"
	$result.Error -join "`r`n" 
    ```  
3. Here is the output of the sample run: 

	```
	Pipeline run status: In Progress
	ResourceGroupName : ADFTutorialResourceGroup
	DataFactoryName   : 
	ActivityName      : MySparkActivity
	PipelineRunId     : 94e71d08-a6fa-4191-b7d1-cf8c71cb4794
	PipelineName      : MySparkOnDemandPipeline
	Input             : {rootPath, entryFilePath, getDebugInfo, sparkJobLinkedService}
	Output            : 
	LinkedServiceName : 
	ActivityRunStart  : 9/20/2017 6:33:47 AM
	ActivityRunEnd    : 
	DurationInMs      : 
	Status            : InProgress
	Error             :
	…
	
	Pipeline ' MySparkOnDemandPipeline' run finished. Result:
	ResourceGroupName : ADFTutorialResourceGroup
	DataFactoryName   : MyDataFactory09102017
	ActivityName      : MySparkActivity
	PipelineRunId     : 94e71d08-a6fa-4191-b7d1-cf8c71cb4794
	PipelineName      : MySparkOnDemandPipeline
	Input             : {rootPath, entryFilePath, getDebugInfo, sparkJobLinkedService}
	Output            : {clusterInUse, jobId, ExecutionProgress, effectiveIntegrationRuntime}
	LinkedServiceName : 
	ActivityRunStart  : 9/20/2017 6:33:47 AM
	ActivityRunEnd    : 9/20/2017 6:46:30 AM
	DurationInMs      : 763466
	Status            : Succeeded
	Error             : {errorCode, message, failureType, target}
	
	Activity Output section:
	"clusterInUse": "https://ADFSparkSamplexxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx.azurehdinsight.net/"
	"jobId": "0"
	"ExecutionProgress": "Succeeded"
	"effectiveIntegrationRuntime": "DefaultIntegrationRuntime (East US)"
	Activity Error section:
	"errorCode": ""
	"message": ""
	"failureType": ""
	"target": "MySparkActivity"
    ```
4. Confirm that a folder named `outputfiles` is created in the `spark` folder of adftutorial container with the output from the spark program. 


## Next steps
The pipeline in this sample copies data from one location to another location in an Azure blob storage. You learned how to: 

> [!div class="checklist"]
> * Create a data factory. 
> * Author and deploy linked services.
> * Author and deploy a pipeline. 
> * Start a pipeline run.
> * Monitor the pipeline run.

Advance to the next tutorial to learn how to transform data by running Hive script on an Azure HDInsight cluster that is in a virtual network. 

> [!div class="nextstepaction"]
> [Tutorial: transform data using Hive in Azure Virtual Network](tutorial-transform-data-hive-virtual-network.md).





