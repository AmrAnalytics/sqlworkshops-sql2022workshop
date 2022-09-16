# Exercise for Parameter Sensitive Plan Optimization (PSP) in SQL Server 2022

The following is an exercise to learn Parameter Sensitive Plan Optimization in SQL Server 2022.

## Prerequisites

- SQL Server 2022 Evaluation Edition
- VM or computer with at min 2 CPUs and 8Gb RAM.
- SQL Server Management Studio (SSMS). The latest 18.x build or 19.x build will work.
- Download **ostress.exe** from https://aka.ms/ostress. Install using the RMLSetup.msi file that is downloaded. Use all defaults.

## Setup the exercise

1. Create a directory called **c:\sql_sample_databases** to store backups and files.
1. Download the backup of WideWorldImporters from https://github.com/Microsoft/sql-server-samples/releases/download/wide-world-importers-v1.0/WideWorldImporters-Full.bak and copy it into c:\sql_sample_databases directory.
1. Restore the WideWorldImporters backup. You can edit and use the **restorewwi.sql** script. This script was designed for a SQL Server in Azure Virtual Machine marketplace image which has separate disks for data and log files. Edit this file to match your disk storage.
1. Execute the **populatedata.sql** script to load more data into the Warehouse.StockItems table. This script will take 5-10 mins to run (timing depends on how many CPUs and the speed of your disk).
1. Rebuild an index associated with the table using the script **rebuild_index.sql**. **IMPORTANT**: If you miss this step you will not be able to see the performance improvement for PSP optimization.
1. Create a new procedure to be used for the workload test using **proc.sql**.
1. Execute the script **setup.sql** from SSMS. This will ensure the WideWorldImporters database is at dbcompat 150 and clear the query store.

## See a PSP problem for a single query execution

1. Set the actual execution plan option in SSMS. Run **query_plan_seek.sql** **twice** in a query window in SSMS. Note the query execution time is fast (< 1 second). Check the timings from SET STATISTICS TIME ON from the second execution. The query is run twice so the 2nd execution will not require a compile. This is the time we want to compare. Note the query plan uses an Index Seek.
2. In a different query window set the actual execution option in SSMS. Run **query_plan_scan.sql** in a query windows in SSMS. Note the query plan uses an Clustered Index Scan and parallelism.
3. Now go back and run **query_plan_seek.sql** again. Note that even though the query executes quickly (< 1 sec), the timing from SET STATISTICS TIME is significantly longer than the previous execution. Also note the query plan also uses a clustered index scan and parallelism.

## See a workload problem for PSP

1. Execute the script **clear.sql** to clear plan cache and query store.
1. Setup perfmon to capture % processor time and batch requests/second.
1. Run **workload_index_seek.cmd 10** from the command prompt. This should finish very quickly. The parameter is the number of users. You may want to increase this for machines with 8 CPUs or more. Observe perfmon counters.
1. Run **workload_index_scan.cmd**. This should take longer but now locks into cache a plan for a scan.
1. Run **workload_index_seek.cmd 10** again. Observe perfmon counters. Notice much higher CPU and much lower batch requests/sec. Also note the workload doesn't finish in a few seconds as before.
1. Hit `<Ctrl>`+`<C>` in the command window for **workload_index_seek.cmd** as it can take minutes to complete.
1. Use the script **suppliercount.sql** to see the skew in supplierID values in the table. This explains why "one size does not fit all" for the stored procedure based on parameter values.

## Solve the problem in SQL Server 2022 with no code changes

1. Let's get this workload to run much faster and consistently using PSP optimization. Execute the T-SQL script **dbcompat160.sql** with SSMS.
2. Run **workload_index_seek.cmd 10** again. Should finish in a few seconds.
3. Run **workload_index_scan.cmd** again.
4. Run **workload_index_seek.cmd 10** again and see that it now finishes again in a few seconds. Observe perfmon counters and see consistent performance.
5. Run Top Resource Consuming Queries report from SSMS and see that there are two plans for the same stored procedure. The one difference is that there is new OPTION applied to the query for each procedure which is why there are two different "queries" in the Query Store.
6. Execute the script **query_store_plans.sql**. Look into the details of the results to see the query text is the same but slightly different with the option to use variants. But notice the query_hash is the same value.
7. Execute the script **query_store_parent_query.sql** and observe this is the text of the query from the stored procedure without variant options. This is the text from the *parent plan*.
8. Execute the script **query_store_dispatcher_plan.sql**. If you click on the dispatcher_plan value you will see a graphical plan operator called Multiple Plan.