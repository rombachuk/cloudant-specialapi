# View Design Document Operations via Operational Queue (_api/migrate)
## Overview
The user or application submits a job request for a view design-document to be created or updated on a database, and supplies the design-document as the request.body (using the PUT or POST verb).  

view design documents have a "views" field within them.  

If the job is accepted then an id will be returned.  
The queue processor will then process the job using submission time order ( a fifo queue).  
If the processor finds the design-document is a new document in the required db:  
  
*	it polls the first view in the document and triggers a build of all views in that document  
*	the queue processor log file (migrateworker.log) will show progress indicators  
*	waits until the view is built (results returned from a poll)  
*	this means all views are built in that document  
*	completes its job and looks at next item  
   
If the processor finds the design-document is an update to an existing document in the required db: 
   
*	it executes the move and shift procedure to build a view_NEW in the background which means reads to the existing view are not blocked
*	the queue processor log file (migrateworker.log) will show progress indicators
*	waits until the view_NEW is built and then copies its signature to the existing view
*	it then deletes the temporary views it has made 
*	completes its job and looks at next item

The user or job then polls the api for a result using the id returned from the accepted submission.  
Failed jobs will have status='failed' with job.info storing identified reasons.
## View Examples
### Submissions
Example submission of a job to update/create an existing view

```
curl -u northamd:*** http://activesn.bkp.ibm.com/_api/migrate/  
diagnoosi/_design/potilaan_merkinnan_aika4  
 -X PUT -d @migrate_exampleviews/pmkc2.json  
```  
Successful reply with job id of <dbid>_<ddid>_submissiontime

```
{"rev": "1-c606a8be52162acceccf15d99694636a", "ok": true,  
"id": "diagnoosi_potilaan_merkinnan_aika4_20180116223622553485"}  
``` 

### Deletes
Example submission of a job to delete an existing view

```
curl -u northamd:*** http://activesn.bkp.ibm.com/_api/migrate/  
diagnoosi/_design/potilaan_merkinnan_aika4 -X DELETE
```  
Successful reply   
  
```
{"rev": "15-173a755ac0fcaec166057cde9106a1af", "ok": true,  
"id": "_design/potilaan_merkinnan_aika4"}
```
### Job Results Collection
Poll status with a GET  
  
```
curl -u northamd:*** http://activesn.bkp.ibm.com/_api/migrate/  
diagnoosi_potilaan_merkinnan_aika4_20180116224753678724 | ./jq  
```


Successful response from jq....

```
{
  "status": "success",
  "ddoc": {
    "views": {
      "potilaan_merkinnan_aika": {
        "map": "function (doc) {   
        if ((doc.potilaanHetu.root) && doc.potilaanHetu.extension) && (doc.merkinnanTapahtumaAika.value)) {  
              yr = doc.merkinnanTapahtumaAika.value.substring(0,4);  
              mth = doc.merkinnanTapahtumaAika.value.substring(4,6);      
              dy = doc.merkinnanTapahtumaAika.value.substring(5,8);      
              emit ([doc.potilaanHetu.root+'.'+doc.potilaanHetu.extension,yr,mth,dy],1);  }}",
        "reduce": "_sum"
      },
      "potilaan": {
        "map": "function (doc) {  if ((doc.potilaanHetu.root) && (doc.potilaanHetu.extension)) {    emit(doc.potilaanHetu.root+'.'+doc.potilaanHetu.extension,1);      }}",
        "reduce": "_count"
      }
    }
  },
  "_rev": "3-ab1f0fd96080752e49d60e3945029f51",
  "qtime": "20180116224753678724",
  "db": "diagnoosi",
  "updated": "2018-01-16 22:49:00.464770",
  "submitted": "2018-01-16 22:47:53.678724",
  "ddid": "potilaan_merkinnan_aika4",
  "requester": "northamd",
  "_id": "diagnoosi_potilaan_merkinnan_aika4_20180116224753678724",
  "completed": "2018-01-16 22:49:00.464770"
}
``` 

## Job Results Format
The job document is returned as json. 

* _id	
-- set as submissiontime down to microseconds  
--	cloudant unique id for document  
-- returned as id field of submission response.
* _rev	
-- cloudant revision   	
*  ddoc  	
-- the code of the designdoc which was submitted (request body)
* requester  	
-- useraccount of job submitter  	
-- extracted from cookie or from basicauth header
* status  
-- one of [submitted,processing,failed,success]  
--	marker of progress for operational queue  
--	queue only runs entries marked _submitted_  
-- updating the status to failed or canceled will drop it from the queue (no api support for this yet)
* info	
-- failure reason	(only exists on error)
* qtime  	
-- time of submission	
-- used to order queue, so smallest qtime goes first
* submitted  
--	submission time of job into queue (pretty format)
* updated  
--	time the entry was updated by the processor	progress marker of job in progress (pretty format)
-- equals completed at end of job
* completed  
-- time job completed or failed (pretty format)	
##	Index Examples
###	Submissions
Example submission of a job to update/create an existing index

```
curl -u northamd:*** http://activesn.bkp.ibm.com/_api/migrate/  
diagnoosi/_design/potilaan_merkinnan_aika4  
 -X PUT -d @migrate_exampleviews/pmkc2.json  
```  
Successful reply with job id of <dbid>_<ddid>_submissiontime

```
{"rev": "1-c606a8be52162acceccf15d99694636a", "ok": true,  
"id": "diagnoosi_potilaan_merkinnan_aika4_20180116223622553485"}  
``` 

###	Deletes
Example submission of a job to delete an existing index

```
curl -u northamd:*** http://activesn.bkp.ibm.com/_api/migrate/  
diagnoosi/_design/potilaan_merkinnan_aika4 -X DELETE
```  
Successful reply   
  
```
{"rev": "15-173a755ac0fcaec166057cde9106a1af", "ok": true,  
"id": "_design/potilaan_merkinnan_aika4"}
```
### Job Results Collection
Poll status with a GET  
  
```
curl -u northamd:*** http://activesn.bkp.ibm.com/_api/migrate/  
diagnoosi_potilaan_merkinnan_aika4_20180116224753678724 | ./jq  
```  
Successful response from jq....

```
{
  "status": "success",
  "ddoc": {
    "views": {
      "potilaan_merkinnan_aika": {
        "map": "function (doc) {   
        if ((doc.potilaanHetu.root) && doc.potilaanHetu.extension) && (doc.merkinnanTapahtumaAika.value)) {  
              yr = doc.merkinnanTapahtumaAika.value.substring(0,4);  
              mth = doc.merkinnanTapahtumaAika.value.substring(4,6);      
              dy = doc.merkinnanTapahtumaAika.value.substring(5,8);      
              emit ([doc.potilaanHetu.root+'.'+doc.potilaanHetu.extension,yr,mth,dy],1);  }}",
        "reduce": "_sum"
      },
      "potilaan": {
        "map": "function (doc) {  if ((doc.potilaanHetu.root) && (doc.potilaanHetu.extension)) {    emit(doc.potilaanHetu.root+'.'+doc.potilaanHetu.extension,1);      }}",
        "reduce": "_count"
      }
    }
  },
  "_rev": "3-ab1f0fd96080752e49d60e3945029f51",
  "qtime": "20180116224753678724",
  "db": "diagnoosi",
  "updated": "2018-01-16 22:49:00.464770",
  "submitted": "2018-01-16 22:47:53.678724",
  "ddid": "potilaan_merkinnan_aika4",
  "requester": "northamd",
  "_id": "diagnoosi_potilaan_merkinnan_aika4_20180116224753678724",
  "completed": "2018-01-16 22:49:00.464770"
}
``` 

##Job Results Format
The job document is returned as json. 

* _id	
-- set as submissiontime down to microseconds  
--	cloudant unique id for document  
-- returned as id field of submission response.
* _rev	
-- cloudant revision   	
*  ddoc  	
-- the code of the designdoc which was submitted (request body)
* requester  	
-- useraccount of job submitter  	
-- extracted from cookie or from basicauth header
* status  
-- one of [submitted,processing,failed,success]  
--	marker of progress for operational queue  
--	queue only runs entries marked _submitted_  
-- updating the status to failed or canceled will drop it from the queue (no api support for this yet)
* info	
-- failure reason	(only exists on error)
* qtime  	
-- time of submission	
-- used to order queue, so smallest qtime goes first
* submitted  
--	submission time of job into queue (pretty format)
* updated  
--	time the entry was updated by the processor	progress marker of job in progress (pretty format)
-- equals completed at end of job
* completed  
-- time job completed or failed (pretty format)	
