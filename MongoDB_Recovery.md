# DISCLAIMER

**NOTE:** This is not a supported product tool.  This is a utility made available for recovery of the MVI collections in the MongoDB - in case of emergency.  This should be used with guidance from IBM support.

# PROBLEM DESCRIPTION
If the storage Physical Volume Claim (PVC) used by the Maximo Visual Inspection 
(MVI) application is exhausted, the MongoDB data can become corrupted.  When 
there is not enough PVC free space (> 95% utiliziation), an operation can result
in an essential MongoDB file - `WiredTiger.turtle` - being corrupted, becoming 
0-length, or being deleleted.   In some other cases of MongoDB corruption the
data can be recovered using MongoDB processes, but if not this script (given 
the various other files in the MongoDB file system) can possibly recover data 
from each individual collection.

MVI uses around different collections, but there's no real way to know which 
collection in the MongoDB corresponds to the known collection in MVI (i.e., 
DataSet, TrainedModels, etc.).  The number of collections depends on the usage
of the product - i.e., creation of projects, deployment of models.  This script 
follows a methodology documented in 
[Repairing MongoDB When WiredTiger.wt File is Corrupted](https://medium.com/@imunscarred/repairing-mongodb-when-wiredtiger-wt-file-is-corrupted-9405978751b5).

# RECOVERY

## Copying Corrupted MongoDB data
To use this utility first the MongoDB data must be copied from the corrupted 
instance. 
1. Delete the **mongodb** and ***service** pods.  This can be done in the 
   OpenShift console, or using the `oc` command line tool. 
1. Wait for the pods to restart.
1. Copy the mongo directory onto a dev system.  One way to do this is to 
   create a debug pod, use this to create a tar of the MongoDB data, then
   copy out of the debug pod.

### Example commands
#### Run a debug MongoDB pod and create a TAR file with the data
```
$ oc get pods | grep mongodb
masmvi-mongodb-6c97dff547-92qnh                               0/1     CrashLoopBackOff   34         3h11m
$ oc debug masmvi-mongodb-6c97dff547-92qnh
Starting pod/masmvi-mongodb-6c97dff547-92qnh-debug ...
If you don't see a command prompt, try pressing enter.
sh-4.2$ cd /var/lib/mongodb/data
sh-4.2$ tar -zcvf mongo-data.tgz * 
WiredTiger
WiredTiger.lock
...
sizeStorer.wt
storage.bson
sh-4.2$
```
DO NOT EXIT the debug pod until the current MongoDB data is copied out, and preferrably until there is recovered data to restore.

#### Copy data from the debug pod
```
$ oc get pods | grep mongo
masmvi-mongodb-6c97dff547-92qnh                               0/1     CrashLoopBackOff   41         3h54m
masmvi-mongodb-6c97dff547-92qnh-debug                         1/1     Running            0          43m
$ oc cp masmvi-mongodb-6c97dff547-92qnh-debug:/var/lib/mongodb/data/mongo-data.tgz .
tar: Removing leading `/' from member names
$ ls -l mongo*
-rw-r--r--   1 user  staff  217353607 May  6 15:41 mongo-data.tgz
```

#### Copy the data to a system for recovery
Copy the saved TGZ file to a system where MongoDB 4.x will be installed and used for recovery.
```
$ scp mongo-data.tgz user@recovery-sys:/tmp/
user@recovery-sys's password: 
mongo-data.tgz                                                                                                                                                                                                  100%  207MB 650.9KB/s   05:26    
```  
   
## Setup
* Python3 env with `pymongo` module installed
* The mongoDB data should be in a directory `~/mongodata`, and the recovery directory location `~/mongodata-recovery` must exist.

### Example setup
```
$ sudo yum install python3
$ sudo yum install python3-virtualenv
$ virtualenv-3.6 mongo-recovery-p3
$ source ~/mongo-recovery-p3/bin/activate
$ pip3 install pymongo
``` 

## Recreate MongoDB 

### Recover collections
Summarizing the article [Repairing MongoDB When WiredTiger.wt File is Corrupted](https://medium.com/@imunscarred/repairing-mongodb-when-wiredtiger-wt-file-is-corrupted-9405978751b5), the corrupted mongo data must be copied to a debug system with mongodb 4.x (which has a better repair
function), then create a brand-new mongo in "WORKINGDBPATH." When mongo runs for the first time in an empty directory
it'll create a fresh db. The utility code in this repo then reads from the current working directory all collections, and one-by-one:
1. starts mongod
1. creates a sacrificial collection that has only one document in it
1. stops mongo
1. replaces the sacrificial collection on disk with the "real" data
1. issues a repair to mongo
1. starts mongo
1. stops mongo

#### Example
The utility must be run in the directory where the mongodb data has been copied for recovery, and the directory where the recovered data is built must already be created (i.e., `mongodb-recovered`).
```
[vision@visx-auto1 ~]$ cd mongodb-backups/
[vision@visx-auto1 mongodb-backups]$ ls
mongo-data-demo04.tgz  mongo-data-fed2-05Jul.tgz  mongo-fed2
[vision@visx-auto1 mongodb-backups]$ mkdir mongo-demo04
[vision@visx-auto1 mongodb-backups]$ cd mongo-demo04/
[vision@visx-auto1 mongo-demo04]$ tar -zxvf ../mongo-data-demo04.tgz 
WiredTiger
WiredTiger.lock
WiredTiger.turtle
...
[vision@visx-auto1 mongo-demo04]$ source ~/mongo-recovery-p3/bin/activate
(mongo-recovery-p3) [vision@visx-auto1 mongo-demo04]$ pip3 list
Package    Version
---------- -------
pip        21.1.1
pymongo    3.11.4
setuptools 56.1.0
wheel      0.36.2
WARNING: You are using pip version 21.1.1; however, version 21.1.3 is available.
You should consider upgrading via the '/home/vision/mongo-recovery-p3/bin/python3 -m pip install --upgrade pip' command.
(mongo-recovery-p3) [vision@visx-auto1 mongo-demo04]$ python3 ~/mchollin-notebook/utilities/mongo-collection-recovery.py
{"t":{"$date":"2021-07-22T09:52:26.164-05:00"},"s":"I",  "c":"CONTROL",  "id":23285,   "ctx":"main","msg":"Automatically disabling TLS 1.0, to force-enable TLS 1.0 specify --sslDisabledProtocols 'none'"}
``` 

### Correctly name collections
This results in a new MongoDB database **Recovery** with all recoverable collections in it, but they are not
identified by the collection names required by the Maximo Visual Inspection application.
Manual inspection is needed to rename them to the correct collections.  To recover all collections:
- Manually start up mongod after the script completes and use [**robo3t**](https://robomongo.org/download) (Download "Robo 3T Only") or some other client to inspect each collection.
- Note that some extra collections will be there for things like mongo itself. ignore / delete these. 
- Be sure to create empty collections for things that were really empty (e.g. **TrainedModels** will be empty if there were no trained models on the system).
- See "Collection Info" below for the full list of collections.

#### Example
Change directory to where the data was recovered, and start `mongod` to connect with Robo3T.
```
(mongo-recovery-p3) [vision@visx-auto1 mongodata-recovered]$ mongod --dbpath . --bind_ip_all
{"t":{"$date":"2021-07-22T10:00:20.804-05:00"},"s":"I",  "c":"CONTROL",  "id":23285,   "ctx":"main","msg":"Automatically disabling TLS 1.0, to force-enable TLS 1.0 specify --sslDisabledProtocols 'none'"}
{"t":{"$date":"2021-07-22T10:00:20.809-05:00"},"s":"W",  "c":"ASIO",     "id":22601,   "ctx":"main","msg":"No TransportLayer configured during NetworkInterface startup"}
...
````

Connect with Robo3T, and rename Collections in the Recovery database.


### Dump recovered database

Once all collections are renamed and the **Recovery** db looks correct (e.g. **collection-4-xyz.wt** is renamed to **BGTasks**, and any unknown collections are removed), the **mongodump** command can be used to dump the Recovery database to disk.  Running `mongodump --db=Recovery` will create a dump on the recovery system in the current directory.  (You will need to run with authentication options if authentication was enabled, as shown in the example.)

Example output (assuming the recovery db was created with user "mongodbuser" and password <password>:
```
[user@recovery-sys dbdump]$ mongodump --db=Recovery --username=mongodbuser --password=<password> --authenticationDatabase=admin
2021-05-06T19:59:59.376-0500	writing Recovery.DataSetFileUserKeys to dump/Recovery/DataSetFileUserKeys.bson
2021-05-06T19:59:59.376-0500	writing Recovery.DataSetFileObjectLabels to dump/Recovery/DataSetFileObjectLabels.bson
2021-05-06T19:59:59.376-0500	writing Recovery.InferenceDetails to dump/Recovery/InferenceDetails.bson
2021-05-06T19:59:59.377-0500	done dumping Recovery.DataSetFileUserKeys (63 documents)
2021-05-06T19:59:59.377-0500	writing Recovery.DataSetTags to dump/Recovery/DataSetTags.bson
2021-05-06T19:59:59.380-0500	done dumping Recovery.DataSetTags (43 documents)
2021-05-06T19:59:59.380-0500	writing Recovery.DataSetFiles to dump/Recovery/DataSetFiles.bson
2021-05-06T19:59:59.380-0500	writing Recovery.DataSets to dump/Recovery/DataSets.bson
2021-05-06T19:59:59.381-0500	done dumping Recovery.InferenceDetails (766 documents)
2021-05-06T19:59:59.381-0500	done dumping Recovery.DataSets (14 documents)
2021-05-06T19:59:59.381-0500	writing Recovery.ProjectGroups to dump/Recovery/ProjectGroups.bson
2021-05-06T19:59:59.381-0500	writing Recovery.BGTasks to dump/Recovery/BGTasks.bson
2021-05-06T19:59:59.381-0500	done dumping Recovery.DataSetFileObjectLabels (1230 documents)
2021-05-06T19:59:59.382-0500	done dumping Recovery.ProjectGroups (11 documents)
2021-05-06T19:59:59.382-0500	done dumping Recovery.BGTasks (4 documents)
2021-05-06T19:59:59.382-0500	writing Recovery.TrainedModels to dump/Recovery/TrainedModels.bson
2021-05-06T19:59:59.382-0500	writing Recovery.WebAPIs to dump/Recovery/WebAPIs.bson
2021-05-06T19:59:59.382-0500	writing Recovery.DLTasks to dump/Recovery/DLTasks.bson
2021-05-06T19:59:59.382-0500	done dumping Recovery.TrainedModels (3 documents)
2021-05-06T19:59:59.382-0500	done dumping Recovery.WebAPIs (2 documents)
2021-05-06T19:59:59.382-0500	writing Recovery.DataSetCategories to dump/Recovery/DataSetCategories.bson
2021-05-06T19:59:59.383-0500	writing Recovery.ServiceInfo to dump/Recovery/ServiceInfo.bson
2021-05-06T19:59:59.383-0500	done dumping Recovery.DataSetCategories (1 document)
2021-05-06T19:59:59.384-0500	done dumping Recovery.ServiceInfo (1 document)
2021-05-06T19:59:59.384-0500	writing Recovery.Tokens to dump/Recovery/Tokens.bson
2021-05-06T19:59:59.384-0500	done dumping Recovery.DLTasks (2 documents)
2021-05-06T19:59:59.384-0500	writing Recovery.DnnScripts to dump/Recovery/DnnScripts.bson
2021-05-06T19:59:59.384-0500	writing Recovery.InferenceOps to dump/Recovery/InferenceOps.bson
2021-05-06T19:59:59.385-0500	done dumping Recovery.Tokens (1 document)
2021-05-06T19:59:59.385-0500	done dumping Recovery.DnnScripts (1 document)
2021-05-06T19:59:59.385-0500	done dumping Recovery.InferenceOps (1 document)
2021-05-06T19:59:59.395-0500	done dumping Recovery.DataSetFiles (4336 documents)
```

## Database restore

Steps to restore the recovered DB:
* Stop mongod on the dev system.
* On the corrupted system:
  * Erase the contents of the PVC/run/mongo directory that you backed up previously. Restart the mongodb pod and allow it to start up again, and re-init with a fresh mongodb store.  
  * Erase the contents of PVC/data/._init/ (leave the directory). This will cause **vision-service** to do index creation among other things on startup.
* Once the new "empty" mongo is running, get the **vision-secrets** mongo admin password, and use port-forwarding to expose the "destination" mongo
on the dev box, something like `oc port-forward deployment/mas-mongodb :27017` which will bind a random available port (eg :65312)
* We can now restore the recovered data into the destination mongo with the vision secrets mongo admin password, and the path to our recovery db dump, which will restore INTO the DLAAS database FROM the Recovery db's dump.
  ```
  $ mongorestore -d DLAAS -h localhost --port 65312 -u admin -p <mongo_admin_password> <path_to_dbdump>
  2021-05-06T20:43:10.779-0500	The --db and --collection flags are deprecated for this use-case; please use --nsInclude instead, i.e. with --nsInclude=${DATABASE}.${COLLECTION}
  2021-05-06T20:43:10.780-0500	building a list of collections to restore from /home/vision/mongodata-recovered/dbdump/dump/Recovery dir
  2021-05-06T20:43:10.794-0500	restoring to existing collection DLAAS.DataSetFiles without dropping
  2021-05-06T20:43:10.794-0500	reading metadata for DLAAS.DataSetFiles from /home/vision/mongodata-recovered/dbdump/dump/Recovery/DataSetFiles.metadata.json
  2021-05-06T20:43:10.794-0500	restoring to existing collection DLAAS.DataSetFileObjectLabels without dropping
  2021-05-06T20:43:10.794-0500	reading metadata for DLAAS.InferenceDetails from /home/vision/mongodata-recovered/dbdump/dump/Recovery/InferenceDetails.metadata.json
  2021-05-06T20:43:10.794-0500	reading metadata for DLAAS.DataSetFileObjectLabels from /home/vision/mongodata-recovered/dbdump/dump/Recovery/DataSetFileObjectLabels.metadata.json
  <br>
  OR
  <br>
  Use a debug pod for the mongodb pod, and remove all data:
  ```
  $ oc debug fed2-mongodb-56fbc4567b-b7vlp 
  Starting pod/fed2-mongodb-56fbc4567b-b7vlp-debug ...
  If you don't see a command prompt, try pressing enter.
  sh-4.2$ bash
  bash-4.2$ cd /var/lib/mongodb/data
  bash-4.2$ rm -rf *
  ```

* **_NOTE:_** - There can be issues with the `mongorestore` using the `oc port-forward`, especially when restoring multiple collections which is done in parallel, and large collections which are done in many chunks.  
  ```
  Handling connection for 41078
  Handling connection for 41078
  E0705 17:28:38.805104  147776 portforward.go:340] error creating error stream for port 41078 -> 27017: Timeout occured
  Handling connection for 41078
  Handling connection for 41078
  E0705 17:28:41.813943  147776 portforward.go:340] error creating error stream for port 41078 -> 27017: Timeout occured
  Handling connection for 41078
  Handling connection for 41078
  Handling connection for 41078
  Handling connection for 41078
  E0705 17:28:53.415952  147776 portforward.go:340] error creating error stream for port 41078 -> 27017: Timeout occured
  E0705 17:28:56.430000  147776 portforward.go:340] error creating error stream for port 41078 -> 27017: Timeout occured
  E0705 17:29:05.447923  147776 portforward.go:340] error creating error stream for port 41078 -> 27017: Timeout occured
  E0705 17:29:08.465480  147776 portforward.go:340] error creating error stream for port 41078 -> 27017: Timeout occured
  E0705 17:29:13.060818  147776 portforward.go:362] error creating forwarding stream for port 41078 -> 27017: Timeout occured
  E0705 17:29:13.061077  147776 portforward.go:362] error creating forwarding stream for port 41078 -> 27017: Timeout occured
  E0705 17:29:13.513930  147776 portforward.go:362] error creating forwarding stream for port 41078 -> 27017: Timeout occured
  E0705 17:29:13.514573  147776 portforward.go:362] error creating forwarding stream for port 41078 -> 27017: Timeout occured
  E0705 17:29:15.298950  147776 portforward.go:340] error creating error stream for port 41078 -> 27017: Timeout occured
  E0705 17:29:15.715876  147776 portforward.go:362] error creating forwarding stream for port 41078 -> 27017: Timeout occured
  ```
  This can cause restoration of large collections to hang or fail.  In this case, the large collection should be restored separately:
  ```
  [vision@visx-auto1 Recovery]$ ls
  BGTasks.bson                           DataSetFileObjectLabels.bson           DLTasks.bson                    ServiceInfo.bson
  BGTasks.metadata.json                  DataSetFileObjectLabels.metadata.json  DLTasks.metadata.json           ServiceInfo.metadata.json
  DataSetActiontags.bson                 DataSetFiles.bson                      DnnScripts.bson                 Tokens.bson
  DataSetActiontags.metadata.json        DataSetFiles.metadata.json             DnnScripts.metadata.json        Tokens.metadata.json
  DataSetCategories.bson                 DataSetFileUserKeys.bson               InferenceDetails.bson           TrainedModels.bson
  DataSetCategories.metadata.json        DataSetFileUserKeys.metadata.json      InferenceDetails.metadata.json  TrainedModels.metadata.json
  DataSetFileActionLabels.bson           DataSets.bson                          InferenceOps.bson               UploadOperations.bson
  DataSetFileActionLabels.metadata.json  DataSets.metadata.json                 InferenceOps.metadata.json      UploadOperations.metadata.json
  DataSetFileLabels.bson                 DataSetTags.bson                       ProjectGroups.bson              WebAPIs.bson
  DataSetFileLabels.metadata.json        DataSetTags.metadata.json              ProjectGroups.metadata.json     WebAPIs.metadata.json
  [vision@visx-auto1 Recovery]$ mkdir ../Recovery2; cd ../Recovery2
  [vision@visx-auto1 Recovery2]$ mongorestore -d DLAAS -h localhost --port 65312 -u admin -p <mongo_admin_password> <path_to>/Recovery2
  ```
  If it still fails, you can restart and there will be "duplicate key errors" until it arrives at the point of the collection where it previously failed.
  ```
  2021-07-05T20:16:09.117-0500	continuing through error: E11000 duplicate key error collection: DLAAS.DataSetFiles index: _id_ dup key: { : "e58f581f-f5d1-4c89-a1de-f35a805575a9" }
  ```
* Start **vision-service** and confirm datasets, models and projects can be viewed (summary and details) as expected.
  * The database indices are not automatically rebuilt, so performance may be significantly impacted for large datasets or large number of datasets.
  * To regenerate the indices, remove the current data by getting a terminal/shell in the **vision-service** pod, and removing the files in `/opt/powerai-vision/data/._init`:
    ```
    sh-4.4$ cd /opt/powerai-vision/data/._init/*
    ```
    then restart the **vision-service** pod (delete it).  On restart the indices are rebuilt.


# Collection Info

All collections are in the `DLAAS` database.  Collections are:
1. [BGTasks](#bgtasks)
1. [DLTasks](#dltasks)
1. [DataSetActiontags](#datasetactiontags)
1. [DataSetCategories](#datasetcategories)
1. [DataSetFileActionLabels](#datasetfileactionlabels)
1. [DataSetFileLabels](#datasetfilelabels)
1. [DataSetFileObjectLabels](#datasetfileobjectlabels)
1. [DataSetFileUserKeys](#datasetfileuserkeys)
1. [DataSetFiles](#datasetfiles)
1. [DataSetTags](#datasettags)
1. [DataSets](#datasets)
1. [DnnScripts](#dnnscripts)
1. [InferenceDetails](#inferencedetails)
1. [InferenceOps](#inferenceops)
1. [ProjectGroups](#projectgroups)
1. [ServiceInfo](#serviceinfo)
1. [Tokens](#tokens)
1. [TrainedModels](#trainedmodels)
1. [UploadOperations](#uploadoperations)
1. [WebAPIs](#webapis)

**REMINDER:* The **collection-n-xyz.wt** files should be renamed exactly with the collection names above, i.e. "**collection-n-xyz.wt**" to "BGTasks".

## BGTasks

Collection for all "background tasks" - i.e., dataset/model imports, auto capture/label, etc.

### TIPS
Look for fields:
* status
* event_type
* percent_complete
* updated_date

### Examples

* `import_dataset` (16 fields)
```
{
    "_id" : "a173921a-e683-4ac0-981e-9a22031f3403",
    "status" : "completed",
    "event_type" : "import_dataset",
    "operation" : null,
    "owner" : "mviuser1",
    "dataset_id" : "a173921a-e683-4ac0-981e-9a22031f3403",
    "total_item_count" : 121,
    "completed_items" : 121,
    "percent_complete" : 100,
    "updated_date" : "2021-04-08T00:05:50.329",
    "updated_at" : NumberLong(1617840350329),
    "success_count" : 61,
    "skipped_count" : 59,
    "fail_count" : 1
}
```

* `auto_capture` (13 fields)
``` 
{
    "_id" : "5b6310f0-64a3-4c06-bda6-6e206cb4521d",
    "status" : "completed",
    "event_type" : "auto_capture",
    "operation" : "",
    "owner" : "mviuser1",
    "dataset_id" : "d66c75ae-fdd3-41fc-aa04-de819242483d",
    "total_item_count" : 23,
    "completed_items" : 23,
    "percent_complete" : 100,
    "updated_date" : "2021-04-15T03:28:06.038",
    "updated_at" : NumberLong(1618457286038),
    "time_interval" : 1000,
    "file_id" : "1ebf6b51-42de-4c29-8fef-5dc4ccf6119e"
}
``` 

* `auto_label` (14 fields)
``` 
{
    "_id" : "571a085b-ad8c-4260-8c2e-7c1a11a8b420",
    "status" : "completed",
    "event_type" : "auto_label",
    "operation" : "dataset",
    "owner" : "mviuser1",
    "dataset_id" : "e4bc3b34-92f5-4cfc-8b63-3810e37526ca",
    "total_item_count" : 90,
    "completed_items" : 90,
    "percent_complete" : 100,
    "updated_date" : "2021-04-15T15:37:17.314",
    "updated_at" : NumberLong(1618501037314),
    "model_id" : "46cae7bd-3116-41dc-9785-60a54c992639",
    "confidence" : 0.65,
    "label_count" : 18
}
``` 

## DLTasks

Collection for all Deep Learning tasks, i.e. model training tasks.

### TIPS
Look for fields:
* usage
* strategy (5 fields)
* nn_arch
* mark_file_path

### Examples

* `cod` model training
```
{
    "_id" : "f4a3ac3f-04f6-411e-b701-b1785f8bc9dd",
    "name" : "test_COD_dataset_model",
    "dataset_id" : "a173921a-e683-4ac0-981e-9a22031f3403",
    "userdnn_id" : null,
    "owner" : "mviuser1",
    "status" : "trained",
    "usage" : "cod",
    "created_at" : NumberLong(1617840523357),
    "strategy" : {...},
    "method" : "FineTuning",
    "action" : "create-cod-task",
    "nn_arch" : "frcnn",
    "mark_file_path" : "/opt/powerai-vision/data/temp/dltasks/f4a3ac3f-04f6-411e-b701-b1785f8bc9dd/marks.xml",
    "pre_analysis_result" : {...},
    "pre_trained_model_path" : null,
    "solver_path" : "/opt/powerai-vision/data/temp/dltasks/f4a3ac3f-04f6-411e-b701-b1785f8bc9dd/solver.prototxt",
    "training_dir_path" : "/opt/powerai-vision/data/mviuser1/datasets/a173921a-e683-4ac0-981e-9a22031f3403/training/f4a3ac3f-04f6-411e-b701-b1785f8bc9dd",
    "dataset_summary" : {...},
    "status_data" : [...]
}
```

* `cod` model training in a project
```
{
    "_id" : "28337e34-12c0-444e-968a-f5465816b107",
    "name" : "Helmet-hires_modelo",
    "dataset_id" : "684c8199-4f2c-4dca-902a-619d41760043",
    "userdnn_id" : null,
    "owner" : "mviuser1",
    "status" : "trained",
    "usage" : "cod",
    "created_at" : NumberLong(1619216966382),
    "strategy" : {...},
    "method" : "FineTuning",
    "action" : "create-cod-task",
    "nn_arch" : "hires_detectron",
    "mark_file_path" : "/opt/powerai-vision/data/temp/dltasks/28337e34-12c0-444e-968a-f5465816b107/marks.xml",
    "pre_analysis_result" : {...},
    "pre_trained_model_path" : null,
    "solver_path" : "/opt/powerai-vision/data/temp/dltasks/28337e34-12c0-444e-968a-f5465816b107/solver.prototxt",
    "training_dir_path" : "/opt/powerai-vision/data/mviuser1/datasets/684c8199-4f2c-4dca-902a-619d41760043/training/28337e34-12c0-444e-968a-f5465816b107",
    "dataset_summary" : {...},
    "status_data" : [ ... ],
    "production_status" : "untested",
    "project_group_id" : "3b931cca-44bf-4295-bcf4-2d99536cba8a",
    "project_group_name" : "Projeto1"
}
```

## DataSetActiontags

Collection for the set of action tags in a dataset.

### TIPS
Look for fields:
* label_count
* tag_duration

### Examples:

```
{
    "_id" : "f839c8b8-3071-46c5-8914-1f0337165a9d",
    "dataset_id" : "f9cb9f76-d354-454d-a9a7-a49bbd12bc0f",
    "name" : "open",
    "normalized_name" : "open",
    "created_at" : NumberLong(1620765420804),
    "label_count" : 6,
    "tag_duration" : 11819.0
}
``` 

## DataSetCategories

Collection for the categories defined across all datasets.

### TIPS
Look for only having **4** fields:
* _id
* dataset_id
* name
* created_at

### Examples
```
{
    "_id" : "2b7b54e0-39fd-4bef-ba32-e76e00fce05e",
    "dataset_id" : "a7cfd4a1-0352-42c3-bd74-4344ab7c9e23",
    "name" : "Transformador",
    "created_at" : NumberLong(1620086431099)
}
``` 

## DataSetFileActionLabels

Collection for ALL _action_ labels that are defined on all images, across all datasets.

### TIPS
Look for fields:
* tag_id
* tag_name
* start_time
* end_time

### Examples
```
{
    "_id" : "970de909-66c4-4258-a5de-318cd082d28d",
    "dataset_id" : "f9cb9f76-d354-454d-a9a7-a49bbd12bc0f",
    "file_id" : "1659d894-5305-412a-8e4b-50f2688f3bdd",
    "tag_id" : "f839c8b8-3071-46c5-8914-1f0337165a9d",
    "tag_name" : "open",
    "start_time" : 90578,
    "end_time" : 92959,
    "created_at" : NumberLong(1620765421949),
    "duration" : 2381.0
}
``` 

## DataSetFileLabels

_(EMPTY in sample DB)_

## DataSetFileObjectLabels

Collection for ALL _object_ labels (bbox and polygon)  that are defined on all images, across all datasets.

### TIPS
Look for fields:
* tag_id
* name
* generate_type
* bndbox

### Examples
```
{
    "_id" : "92e0362b-45fb-4b30-aa2a-d96983b32b72",
    "dataset_id" : "4342a7ab-be6f-44c8-a568-bafffa1ac114",
    "file_id" : "b7822fc6-dc19-455b-8aef-76572e6921b7",
    "tag_id" : "4342a7ab-be6f-44c8-a568-bafffa1ac114-helmet",
    "name" : "helmet",
    "generate_type" : "manual",
    "bndbox" : {
        "xmin" : 141,
        "ymin" : 48,
        "xmax" : 200,
        "ymax" : 91
    }
}
```

## DataSetFileUserKeys

Collection for ALL metadata fields/types that are defined.

### TIPS
Look for exactly **5** fields:
* _id
* name
* dataset_id
* use_count
* created_at

### Examples
```
{
    "_id" : "87ec5c7e-3b1a-4280-8ab3-778ae409d5c0:EXIF_Exif Image Width",
    "name" : "EXIF_Exif Image Width",
    "dataset_id" : "87ec5c7e-3b1a-4280-8ab3-778ae409d5c0",
    "use_count" : 2629,
    "created_at" : NumberLong(1619539708684)
}
```

## DataSetFiles

Collection for ALL files across all datasets.

### TIPS
Look for fields:
* file_name
* size
* content_type
* meta_data
* original_file_name

### Examples
* Image file (18 fields)
```
{
    "_id" : "a36135ef-4dab-4011-942b-4a49fd339548",
    "file_type" : "image",
    "owner" : "mviuser1",
    "file_name" : "a36135ef-4dab-4011-942b-4a49fd339548.jpg",
    "created_at" : NumberLong(1533988392808),
    "uploaded_at" : NumberLong(1533988392808),
    "dataset_id" : "4342a7ab-be6f-44c8-a568-bafffa1ac114",
    "size" : "23405",
    "content_type" : "image/jpeg",
    "meta_data" : {
        "width" : 300,
        "height" : 450
    },
    "label_file_key" : "4342a7ab-be6f-44c8-a568-bafffa1ac1146306ee96-5494-4e83-ba2f-814ca965893c",
    "original_file_name" : "9f35017b-222e-4e96-94b7-43f90ead3a38.jpg",
    "fps" : 0.0,
    "duration" : 0.0,
    "upload_type" : "file_upload",
    "generate_type" : "manual",
    "label_type" : "manual",
    "tag_list" : [ 
        {
            "tag_id" : "4342a7ab-be6f-44c8-a568-bafffa1ac114-vest",
            "tag_name" : "vest",
            "tag_count" : 1
        }, 
        {
            "tag_id" : "4342a7ab-be6f-44c8-a568-bafffa1ac114-no_glove",
            "tag_name" : "no_glove",
            "tag_count" : 1
        }, 
        {
            "tag_id" : "4342a7ab-be6f-44c8-a568-bafffa1ac114-no_helmet",
            "tag_name" : "no_helmet",
            "tag_count" : 1
        }
    ]
}
``` 
    
* Video file (16 fields)
```
{
    "_id" : "7eaaef53-5997-4963-9d5a-5aa32f8eccdc",
    "file_type" : "video",
    "owner" : "mviuser1",
    "file_name" : "7eaaef53-5997-4963-9d5a-5aa32f8eccdc.mp4",
    "created_at" : NumberLong(-2082844800000),
    "uploaded_at" : NumberLong(1533888925848),
    "dataset_id" : "4342a7ab-be6f-44c8-a568-bafffa1ac114",
    "size" : "3764930",
    "content_type" : "video/mp4",
    "meta_data" : {},
    "label_file_key" : "4342a7ab-be6f-44c8-a568-bafffa1ac114c56277ca-a5a9-409a-8293-606f60b79a51",
    "original_file_name" : "f2abbc37-9409-4e17-a3e2-216792ea326e.mp4",
    "fps" : 29.97003,
    "duration" : 13313.3,
    "upload_type" : "file_upload",
    "generate_type" : null
}
``` 

## DataSetTags

Collection for ALL object labels that have been defined across all datasets.

### TIPS
Look for exactly **6** fields:
* _id
* dataset_id
* name
* normalilzed_name
* label_count
* created_at

### Examples
```
{
    "_id" : "4342a7ab-be6f-44c8-a568-bafffa1ac114-no_helmet",
    "dataset_id" : "4342a7ab-be6f-44c8-a568-bafffa1ac114",
    "name" : "no_helmet",
    "normalized_name" : "no_helmet",
    "label_count" : 4,
    "created_at" : NumberLong(1617836281887)
}
```

## DataSets

Collection for all datasets.

### TIPS
Look for exactly **6** fields:
* name
* usage
* scenario
* owner

### Examples

* Dataset 
``` 
{
    "_id" : "4342a7ab-be6f-44c8-a568-bafffa1ac114",
    "name" : "test_COD_dataset",
    "usage" : "generic",
    "scenario" : "",
    "pre_process" : "",
    "owner" : "mviuser1",
    "locked" : 0,
    "type" : 0,
    "created_at" : NumberLong(1617836281653),
    "updated_at" : NumberLong(1617836283695)
}
```

* Dataset in project:
```
{
    "_id" : "684c8199-4f2c-4dca-902a-619d41760043",
    "name" : "Helmet-hires",
    "usage" : "generic",
    "scenario" : "",
    "pre_process" : "",
    "owner" : "mviuser1",
    "locked" : 0,
    "type" : 0,
    "created_at" : NumberLong(1618452293500),
    "updated_at" : NumberLong(1620347615964),
    "project_group_id" : "3b931cca-44bf-4295-bcf4-2d99536cba8a",
    "project_group_name" : "Projeto1"
}
```

## DnnScripts

Collection of all post-processing scripts defined.

### TIPS
Look for fields:
* name
* description
* owner
* editable

### Examples
``` 
{
    "_id" : "494bcb74-fc2f-4d83-a27f-48b21f5bf702",
    "name" : "Register inference results in Monitor",
    "description" : "Default post processing script to save inference results to configured dataset and register as events to monitor",
    "owner" : "admin",
    "created_at" : NumberLong(1617820631444),
    "editable" : "no"
}
```

## InferenceDetails

Collection of all inferences that are saved.

### TIPS
Look for fields:
* confidence
* label
* infer_id

### Examples
``` 
{
    "_id" : "9bd4de94-dc6d-4902-bf4e-0319b47b46b8.1",
    "confidence" : 0.949366942048073,
    "label" : "helmet",
    "xmin" : 1211,
    "xmax" : 1229,
    "ymin" : 534,
    "ymax" : 549,
    "polygons" : [ ... ],
    "frame_number" : "1",
    "time_offset" : 40,
    "sequence_number" : 1,
    "infer_id" : "9bd4de94-dc6d-4902-bf4e-0319b47b46b8"
}
```

## InferenceOps

Collection of video inference operations.

### TIPS
Look for fields:
* video_in
* video_out
* processed_frames
* total_frames

### Examples
```
{
    "_id" : "9bd4de94-dc6d-4902-bf4e-0319b47b46b8",
    "status" : "completed",
    "video_in" : "/uploads/inferences/9bd4de94-dc6d-4902-bf4e-0319b47b46b8/Construction-site-video.mov",
    "video_out" : "/uploads/inferences/9bd4de94-dc6d-4902-bf4e-0319b47b46b8/Construction-site-video_out.mp4",
    "model_id" : "46cae7bd-3116-41dc-9785-60a54c992639",
    "percent_complete" : 100.0,
    "processed_frames" : 300,
    "total_frames" : 300,
    "sequence_number" : 164,
    "usage" : "cod",
    "params" : {
        "apiEndpoint" : "inference",
        "annotateBoxes" : "false",
        "genCaption" : "true",
        "modelId" : "46cae7bd-3116-41dc-9785-60a54c992639",
        "videourl" : "file:///opt/powerai-vision/data/inferences/9bd4de94-dc6d-4902-bf4e-0319b47b46b8/Construction-site-video.mov",
        "podName" : "/api/v1/namespaces/mas-ckd101v3-visualinspection/pods/ckd101v3-cod-infer-46cae7bd-3116-41dc-9785-60a54c992639-7djtpqq",
        "confthre" : "0.5"
    },
    "created_date" : "2021-05-11T22:27:00.603",
    "updated_date" : "2021-05-11T22:29:28.027",
    "thumbnail_path" : "/uploads/inferences/9bd4de94-dc6d-4902-bf4e-0319b47b46b8/thumbnail.jpg"
}
``` 

## ProjectGroups

Collection of all defined projects.

### TIPS
Look for fields:
* auto_deploy
* dataset_count
* model_count
* latest_model

### Examples
```
{
    "_id" : "3b931cca-44bf-4295-bcf4-2d99536cba8a",
    "owner" : "masadmin",
    "name" : "Projeto1",
    "description" : "",
    "enforce_pwf" : "false",
    "auto_deploy" : "false",
    "created_at" : NumberLong(1620347603793),
    "dataset_count" : 1,
    "model_count" : 1,
    "latest_model" : "28337e34-12c0-444e-968a-f5465816b107",
    "updated_at" : NumberLong(1620350310491)
}
``` 

## ServiceInfo

### TIPS
Look for exactly **2** fields:
* _id
* value

### Examples
```
{
    "_id" : "InstallationTime",
    "value" : NumberLong(1617820631352)
}
```

## Tokens

Collection for all defined API tokens.

### TIPS
Look for fields:
* user_id
* type (apikey)
* roles
* expired_at

### Examples
```
{
    "_id" : "usJe-Dmc3-eYZj-LSX8",
    "user_id" : "mviuser1",
    "type" : "apikey",
    "roles" : [ 
        "premium_user"
    ],
    "created_at" : NumberLong(1618183645591),
    "expired_at" : NumberLong(2248903645591)
}
``` 


## TrainedModels

Collection for all trained models.

### TIPS
Look for fields:
* nn_arch
* userdnn_id
* production_status
* deployed
* accuracy
* thumbnail_path
* version

### Examples
* FRCNN model
``` 
{
    "_id" : "f4a3ac3f-04f6-411e-b701-b1785f8bc9dd",
    "name" : "test_COD_dataset_model",
    "dataset_id" : "a173921a-e683-4ac0-981e-9a22031f3403",
    "usage" : "cod",
    "nn_arch" : "frcnn",
    "userdnn_id" : null,
    "method" : "FineTuning",
    "strategy" : {...},
    "dataset_summary" : {...},
    "production_status" : "untested",
    "categories" : [...],
    "owner" : "mviuser1",
    "deployed" : 0,
    "type" : 0,
    "eval_data" : {...},
    "accuracy" : "0.6538461538461539",
    "thumbnail_path" : "/opt/powerai-vision/data/mviuser1/trained-models/thumbnails/f4a3ac3f-04f6-411e-b701-b1785f8bc9dd.jpg",
    "created_at" : NumberLong(1617840817183),
    "version" : "8.2.0-pre.dev84",
    "assets" : [...]
}
```

* hires_detectron model
```
{
    "_id" : "46cae7bd-3116-41dc-9785-60a54c992639",
    "name" : "Helmet-hires-14Apr",
    "dataset_id" : "684c8199-4f2c-4dca-902a-619d41760043",
    "usage" : "cod",
    "nn_arch" : "hires_detectron",
    "userdnn_id" : null,
    "method" : "FineTuning",
    "strategy" : {...},
    "dataset_summary" : {...},
    "production_status" : "untested",
    "categories" : [...],
    "owner" : "mviuser1",
    "deployed" : 1,
    "type" : 0,
    "eval_data" : {...},
    "accuracy" : "0.4307692307692308",
    "thumbnail_path" : "/opt/powerai-vision/data/mviuser1/trained-models/thumbnails/46cae7bd-3116-41dc-9785-60a54c992639.jpg",
    "created_at" : NumberLong(1618454599143),
    "version" : "8.2.0-pre.dev84"
}
```

## UploadOperations

Collection of all upload operations.

### TIPS

Look for documents of 12 fields including:
* owner
* file_name
* file_size
* total_parts (8.2+)

### Examples
```
{
    "_id" : "55ef5c65-b64b-4621-b65b-6d7af6d6a5f2",
    "owner" : "mviuser2",
    "status" : "completed",
    "status_msg" : null,
    "file_name" : "Action_Detection.zip",
    "file_size" : 1479901252,
    "total_parts" : 142,
    "number_of_parts_received" : 142,
    "parts_status" : [...],
    "created_at_millis" : NumberLong(1617893698928),
    "last_updated_millis" : NumberLong(1617894996907),
    "closed_millis" : NumberLong(1617894996978)
}
```

## WebAPIs

Collection of deployed models.

### TIPS
Look for fields:
* status
* save_inference
* accel_type
* container_id

### Examples
```
{
    "_id" : "46cae7bd-3116-41dc-9785-60a54c992639",
    "owner" : "mviuser1",
    "usage" : "cod",
    "status" : "ready",
    "replicas" : 1,
    "name" : "Helmet-hires-14Apr",
    "created_at" : NumberLong(1620771191691),
    "userdnn_id" : null,
    "dnnscript_id" : null,
    "save_inference" : null,
    "accuracy" : "0.4307692307692308",
    "categories" : [...],
    "nn_arch" : "hires_detectron",
    "production_status" : "untested",
    "version" : "8.2.0-pre.dev84",
    "dataset_id" : "684c8199-4f2c-4dca-902a-619d41760043",
    "accel_type" : "GPU",
    "container_id" : "46cae7bd-3116-41dc-9785-60a54c992639"
}
```
