# Overview
Sync orders from Superfeet.com (Magento 2) into Macola and sync order data back into Magento 2 to update the orders/move their status/add tracking as well as update inventory based on the invetorty levels in Macola.

## Syncs
### Pull Orders
In order to get orders it make a call to the [REST API](https://magento.redoc.ly/2.3.7-admin/tag/orders) filtering by orders we haven't pulled into macola using a custom status ("erp_processed") that we push back after an order is created in Macola.
This is all done use a [C# client](README.md#magento-api-client-ioswagger) generated using the swagger and [code](sf_ERP_Magento2_Sync/ERPToMagento2.cs) that uses the [SyncLib](https://github.com/Superfeet/sfSyncLib) by inheriting and overriding the pull method.
After the data is pulled it is then translated and written into the [order transaction](sf_ERP_Magento2_DatabaseLayer/Model/M2OrderTransaction.cs) & [item transaction](sf_ERP_Magento2_DatabaseLayer/Model/M2ItemTransaction.cs) tables to be consumed by the [process code](#process-orders).

### Process Orders
This symply uses the unsynced records in the [order transaction](sf_ERP_Magento2_DatabaseLayer/Model/M2OrderTransaction.cs) & [item transaction](sf_ERP_Magento2_DatabaseLayer/Model/M2ItemTransaction.cs) tables along with the map table to create new orders and accounts in Macola useing the [Macola Library](https://github.com/Superfeet/sfSyncLib/tree/master/sfMacolaLib).
After an order or account is created it adds it to the corresponding map table and then creates an outgoing [order status transactional](sf_ERP_Magento2_DatabaseLayer/Model/OrderTransaction.cs) record that is then used by the [Order Status Push](#push-orders)

### Push Orders
This takes the [order status transactions](sf_ERP_Magento2_DatabaseLayer/Model/OrderTransaction.cs) that aren't synced and calls uses a [C# client](README.md#magento-api-client-ioswagger) generated using the swagger and [code](sf_ERP_Magento2_Sync/ERPToMagento2.cs) to push them to the API.
Depending on the status a different endpoint is used:
 * "CANCELLED" will call the [cancel endpoint](https://magento.redoc.ly/2.3.7-admin/tag/ordersidcancel). This is mainly used when we get an order missing required data to process the order in the case of an anonymized order, but in the future could be used if an order is canceled in Macola and we want to push that back to the site.
 * "erp_processed" - Uses the generic order edit endpoint and pushes this custom status, so we don't get the order again, along with the macola order number and customer number to the extOrderId and extCustomerId fields.
 * "holded" - Pushes this built in status out to the site using the generic edit endpoint to put the order on hold that we skipped because required data was missing

### Push Inventory
This uses the stored procedure [sp_CreateProductInventoryTransactions.sql](sf_ERP_Magento2_DatabaseLayer/sql/Stored%20Procedures/sp_CreateProductInventoryTransactions.sql) to compare Macolas current state (using a view) against the control table (last pushed records) and see what is new and create transactional records.
These records are then pushed using the Magento API to update inventory sources (done in the [Magento Sync](#magento-sync-sf_erp_magento2_sync) project).
It also has some logic to group transactional records by source and item and figure out the latest records and then compare that against what the inventory for that source/item is using the get invnetory at a source API so we only push changes.

### Push Shipments
This uses the stored procedure [sp_CreateShipmentTransactions.sql](sf_ERP_Magento2_DatabaseLayer/sql/Stored%20Procedures/sp_CreateShipmentTransactions.sql) to compare Macolas current state (using a view) against the control table (last pushed records) and see what is new and create transactional records.
These records are then pushed using the Magento API to invoice and ship the order (done in the [Magento Sync](#magento-sync-sf_erp_magento2_sync) project).

## Projects
There are several projects that make up this repository (all Visual Studio projects in a solution file). Each one is for a different purpose and are listed below.
They are all used for creating the database and the [Sync processes](#syncs).

### Sync Database (sf_ERP_Magento2_DatabaseLayer)
Code first Migration project that holds all the tables and info to deploy the database for syncing.
In the configuration seed method it also populates the ship method and store map tables with data based on the B2C Magento setup.
Lastly there is some sql scripts to create the: views, functions, and stored procedures used in the sync when the seed method runs. It runs everything in the SQL folder.

#### Views
 * [Shipments.sql](sf_ERP_Magento2_DatabaseLayer/sql/Views/Shipments.sql) - This is used to generated the outbound records to invoice and ship orders on the B2C that have posted in Macola and have a shipment record. Note: The carrier code and title are used when calling the API to create a [shipment](https://devdocs.magento.com/guides/v2.4/rest/tutorials/orders/order-create-shipment.html) so they must match the codes setup in Magento in order for the tracking link to work. Title is just sent in the email so can be used as a backup for the customer to guess how to use the tracking number. Also the tracking number must have a value.
 * [OrderReport.sql](sf_ERP_Magento2_DatabaseLayer/sql/Views/OrderReport.sql) - This is used in an Excel report to view the status of orders to see if there are any errors syncing in an order to the ERP and what order it created as well as if it sent out a shipment or not and any issues.
 * [DealerSearchData.sql](sf_ERP_Magento2_DatabaseLayer/sql/Views/DealerSearchData.sql) - This view is used to generate a flat file (CSV) 
#### Stored Procedures
 * [sp_CreateMissingItemMap.sql](sf_ERP_Magento2_DatabaseLayer/sql/Stored%20Procedures/sp_CreateMissingItemMap.sql) - Looks for orders with order maps, but no item maps (items failed to sync) and creates them based on the order in the ERP
 * [sp_CreateProductInventoryTransactions.sql](sf_ERP_Magento2_DatabaseLayer/sql/Stored%20Procedures/sp_CreateProductInventoryTransactions.sql) - Used in the Inventory sync to generate transactional records of inventory changes by item sku and inventory source (warhouse/company). It uses the default warhouse setup in Macola by Company. Values returned are based on an item flag (B2C For Sale) and it sends either 0 or 9999, it then adds back in inventory based on unshipped shipped orders (looks at order & item map). This ensures that when we go to ship the order it won't fail because the site doesn't have the inventory for it.
 * [sp_CreateShipmentTransactions.sql](sf_ERP_Magento2_DatabaseLayer/sql/Stored%20Procedures/sp_CreateShipmentTransactions.sql) - Creates outgoing Shipment transactions using the [Shipments.sql view](#Views) and comparing it against the [control](sf_ERP_Magento2_DatabaseLayer/Model/Shipment.cs) table using a merge statment updating the [control](sf_ERP_Magento2_DatabaseLayer/Model/Shipment.cs) and creating [transactional records](sf_ERP_Magento2_DatabaseLayer/Model/ShipmentTransaction.cs) that are used to send the updates.

### SQL Scripts Project (CustomSqlScripts)
This holds all the views, functions and stored procedures in a SQL project that match those in the [Database Layer Project](#sync-database-sf_erp_magento2_databaselayer), but allows for comparison with the actually deployed database.

### Magento API Client (IO.Swagger)
This is generated C# code with some modifications to support an unlimted amount of query parameters for the get orders endpoint using [SwaggerHub](https://swagger.io/tools/swaggerhub/) and the [swagger file](https://www.superfeet.com/en-us/swagger#/) on the B2C site.

### Macola/ERP Process (sf_ERP_Magento2_Processes)
This project use the transaction data to create Macola accounts and orders. It is a console program and uses the [Macola Library](https://github.com/Superfeet/sfSyncLib/tree/master/sfMacolaLib) for a lot of the work.

### Magento Sync (sf_ERP_Magento2_Sync)
This project handles pushing and pulling data from the Magento API using the [Rest Client](#magento-api-client-ioswagger). It uses the [Sync Database](#sync-database-sf_erp_magento2_databaselayer) to write data pulled and the outgoing shipment, inventory, and order status transactional records to push things out.
