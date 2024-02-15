# Microsoft Fabric-Pause, Resume, Scale a Capacity with a Fabric Data Factory Pipeline

---


There are a number of ways to pause, resume or scale a Fabric Capacity. Note this only applies to F-SKUs on Azure, as the Premium Capacity (P-SKU) can only be scaled.
We are discussing how to do this using a Fabric Data Factory pipeline, so that you can orchestrate one or many capacities and workloads.
It is assumed that the concepts of Fabric capacities, workspaces and workloads is already understood, as well as good working knowledge of Azure. You will also need a Fabric Capacity, ideally one that is always on.

## Overall approach
- Provision a Fabric Capacity in Azure 
- Create a Service Principal in Azure
- Give appropriate permissions (RBAC) to the Service Principal
- Create a Fabric Data Factory pipeline that executes a web activity to post to the relevant API (not currently formally supported)

This will allow you create a Pipeline that you can call from anywhere in Fabric to pause, resume or scale a Capacity.

**Provision**
To provision a capacity, choose Microsoft Fabric in the list of services and specify:
- Subscription and Resource Group
- Capacity Name and Region (note that this can by anywhere globally, but it may be best to start somewhere where your Power BI tenant is already located (Check under Help and Support in Power BI, look for "About Power BI")
- Size (F2-F2048)
- A Fabric Capacity Administrator 

**Service Principal**
Create a SPN using whatever method you prefer. My favourite is using the Azure CLI.
Permissions
Ensure your service principal has contributor rights to the Fabric Capacity.

**Create a Data Pipeline**
Navigate to the Workspace in Microsoft Fabric where you intend to create the Pipeline. Create a new pipeline and give it a useful name: ManageCapacityPauseResume

Add some parameters to make your pipeline more flexible, complete them appropriately.

Add a Web Activity to the canvas. You will need to specify the service principal you created earlier.

For the relative URL in the Web activity add the following expression:
```
@concat('/subscriptions/',pipeline().parameters.subscription_id,'/resourceGroups/',pipeline().parameters.resourcegroup,'/providers/Microsoft.Fabric/capacities/',pipeline().parameters.capacities,'/',pipeline().parameters.action,'?api-version=2022–07–01-preview')
```

Choose the POST method and put {} in the Body.
Note the action parameter, this can be "suspend" or "resume" for pause or resume as desired.

You can now save and run the pipeline to test it out. If you capacity is running, execute a "suspend". Observe how the Azure portal shows the capacity is paused.

Note that if the capacity is not running and you try and pause it, it will fail the activity. To deal with this you can handle that specific error and only fail the pipeline if it occurs, something like the below image. 

To scale a capacity, use a similar pipeline to above. 
Change the expression in the Web activity to
```
@concat('/subscriptions/',pipeline().parameters.subscription_id,'/resourceGroups/',pipeline().parameters.resourcegroup,'/providers/Microsoft.Fabric/capacities/',pipeline().parameters.capacities,'?api-version=2022–07–01-preview')
```

You will also need to add a new parameter for the SKU value.
Change the Body to
```
@json(concat('{"sku":{"name":"',pipeline().parameters.sku,'","tier":"Fabric"}}'))
```
So that it looks something like ![](./images/Scale%20capacity.png)

## Additional Topics
It is possible to pass variable and arrays between pipelines, as well as to schedule them. This means that if you have development workspaces that reside on a specific capacity or capacities, you can schedule the pause pipeline to run daily at a set time to save costs, or to scale capacities as appropriate.

JSON for Pipelines can be found here
- [Outer Pipeline to run multiple capacities](./pipelines/ManageCapacity_Resume.json)
- [Inner Pipeline to run pause or resume, can be run standalone](./pipelines/ManageCapacityPauseResume.json)
- [Scale Pipeline](./pipelines/ScaleCapacity.json)
Note that while it possible to view JSON for each pipeline, it not currently possible to use this deploy another version of this pipeline, as in Azure Data Factory (Feb 2024).

## Error handling information
Add a set variable activity to harvest the possible from stopping a stopped capacity. Use the following expression for the Pipeline variable
```
@activity('PauseResumeCapacity').output.error.message
```
Add an if condition with the following expression
```
@or(equals(variables('CapacityErrorResponse'),'Service is not ready to be updated'),equals(variables('CapacityErrorResponse'),''))
```
For True, use a Wait for 1 second, for False fail the pipeline with a Fail activity.

![Pause or Resume Pipeline with error handling](./images/More%20Complex%20Pipeline.png)

## Final thoughts
While the APIs used here for the Fabric capacities are not yet formally documented at time of writing (Feb 2024), using Fabric pipelines forms a useful orchestration alternative to the more common use of Logic Apps for this purpose. This means you can use a Fabric SaaS solution for Fabric capacity management.


## Useful links (that I found after writing this)
 - [Using Powershell to manage a capacity](https://github.com/sergeig888/ps-fskumgmt-fabric)
 - [Useful info on using ADF, Fabric or Postman to manage capacities](https://github.com/nocsi-zz/fabric-capacity-management)
