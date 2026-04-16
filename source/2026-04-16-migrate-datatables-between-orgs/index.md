---
title: Data Tables Migration between orgs using Python Script 
tags: data-table,python,migration
date: 2026-04-16
author: david.fradejas
image: datatable migration.png
category: 6
---

Hello everyone! Something really cool and pretty useful is about to migrate datatables between orgs
This blog aims to introduce you to a simple script to help with that.

_**Table of Contents:**_

[Pre-requisites](#pre-requisites)

[Script Steps](#script-steps)


* ## Pre-requisites
* _**Visual Studio**_
  
Persons wishing to use this script should be familiar with visual studio / framework , and the mechanisms for use them.

* _**Python SDK**_
  
Should be installed and configured properly, and dependencies.

* _**Enviroment variables**_

get yor .env file ready to use for multiple orgs, formatting like this (just example):
```
#Credentials ####################################
ORG_A_REGION=eu_west_1
ORG_A_CLIENTID=4ef41437-926e-4de2-90bf-AAAAAAA
ORG_A_SECRET=MvsVRWOeu-AQEFRksTUS24myI_AAAAAAA
ORG_B_REGION=eu_west_1
ORG_B_CLIENTID=4ef41437-926e-4de2-90bf-BBBBB
ORG_B_SECRET=MvsVRWOeu-AQEFRksTUS24myIBBBBBB
#####################################
```

* ## Script Steps
  
### Setting Org origin Credentials
```
################# ORIGIN #################
# Credentials Origin
CLIENT_ID = os.environ[org_origin+"_CLIENTID"]
CLIENT_SECRET = os.environ[org_origin+"_SECRET"]
ORG_REGION = os.environ[org_origin+"_REGION"]
# Set environment
region = PureCloudPlatformClientV2.PureCloudRegionHosts[ORG_REGION]
PureCloudPlatformClientV2.configuration.host = region.get_api_host()

# OAuth when using Client Credentials
api_client = PureCloudPlatformClientV2.api_client.ApiClient() \
            .get_client_credentials_token(CLIENT_ID, CLIENT_SECRET)

tokenOrigin=api_client.access_token
```

### Check credential roles
```
#Check credentials's Roles
api_oauth = PureCloudPlatformClientV2.OAuthApi(api_client)
api_authorization=PureCloudPlatformClientV2.AuthorizationApi(api_client)
try:
    oauth_api = api_oauth.get_oauth_client(CLIENT_ID)
    roleid_list=oauth_api.role_divisions
    role_detail_list=[]
    for role in roleid_list:
            role_detail_name=api_authorization.get_authorization_role(role.role_id).name        
            role_detail_list.append(role_detail_name)

    print("Successfully connected with roles: ",role_detail_list)
except ApiException as e:
        print("Exception when calling OAuthApi->get_oauth_client: %s\n" % e)
        exit()
```

### Get Datatable origin schema
```
architect_api = PureCloudPlatformClientV2.ArchitectApi(api_client)
while True:
    #datatable_name = 'DT_SANTANDER_DYNAMIC_MNU'
    datatable_name = input("Enter your Datatable name: ")    
    #Get DataTable schema on previous enviroment
    print("Getting DataTable schema on previous enviroment...")
    print("")
    time.sleep(0.5)
    expand = 'schema'
    try:
        # Retrieve a list of datatables for the org filter by name
        api_response = architect_api.get_flows_datatables(expand=expand, name=datatable_name)
        if len(api_response.entities)==0:
            print("Not valid Datatable name: ", datatable_name)
        else:
            break
    except ApiException as e:
        print("Exception when calling ArchitectApi->get_flows_datatables: %s\n" % e)
        exit()

pprint(api_response.entities[0].schema)
schema = api_response.entities[0].schema
datatable_id= api_response.entities[0].id
datatable_division_name=api_response.entities[0].division.name
datatable_description=api_response.entities[0].description
print("Successfully get Datatable schema from Datatable. Datatableid: ",datatable_id," Division: ",datatable_division_name)
print("")
```

### Get Total Datatable Rows
```
try:
    # Get Total Datatable Rows
    print("Getting Total Datatable Rows..." )
    page_number=1
    page_size=200            
    api_getrows = architect_api.get_flows_datatable_rows(datatable_id, page_number=page_number, page_size=page_size, showbrief=False)
    api_getrows_list = api_getrows.entities
    #pprint(api_getrows_list)
    #os.system("pause")
    total_pages= api_getrows.page_count
    total_rows=api_getrows.total
    pbar = tqdm.tqdm (range(total_pages-1),leave=False)    
    while total_pages > page_number:    
        page_number += 1                                 
        api_newgetrows = architect_api.get_flows_datatable_rows(datatable_id, page_number=page_number, page_size=page_size, showbrief=False)
        api_getrows_list.extend(api_newgetrows.entities)
        total_newrows= len(api_getrows_list)
        time.sleep(0.0)
        pbar.update(1)
    print(" Total Rows: ",len(api_getrows_list))
    print("")
except ApiException as e:
    print("Exception when calling ArchitectApi->get_flows_datatable_rows: %s\n" % e)
    exit()  
```

### Use Job to get the origin datatable CSV file
```
# Using Jobs
try:
    api_jobrows=architect_api.post_flows_datatable_export_jobs(datatable_id)
    jobid=api_jobrows.id
    #print(" Jobid: ", jobid)
    status="Processing"
    while status !="Succeeded":
        api_getjobstatus=architect_api.get_flows_datatable_export_job(datatable_id, jobid)
        status=api_getjobstatus.status
    downloadURI=api_getjobstatus.download_uri
    headers = {"Authorization": "Bearer " +tokenOrigin}
    requestURI=requests.get(downloadURI, headers=headers, verify=False)
    #decoded_content = requestURI.content.decode('utf-8')
    decoded_content = requestURI.content

    temp_file_name=datatable_name+".csv"
    with open(temp_file_name, 'wb', ) as temp_file:
        temp_file.write(decoded_content)

    breakpoint()
   
    print('File saved in:', os.getcwd())
    temp_file.close()

except ApiException as e:
    print("Exception when calling ArchitectApi->get_flows_datatable_rows: %s\n" % e)
    exit()  
```

### Setting Org Destination Credentials and check credentials roles

```
################# DESTINATION #################

# Credentials Destination
CLIENT_ID = os.environ[org_destination+"_CLIENTID"]
CLIENT_SECRET = os.environ[org_destination+"_SECRET"]
ORG_REGION = os.environ[org_destination+"_REGION"]

# Set environment
region = PureCloudPlatformClientV2.PureCloudRegionHosts[ORG_REGION]
PureCloudPlatformClientV2.configuration.host = region.get_api_host()

# OAuth when using Client Credentials
api_client = PureCloudPlatformClientV2.api_client.ApiClient() \
            .get_client_credentials_token(CLIENT_ID, CLIENT_SECRET)

tokenDestination=api_client.access_token

#Check credentials's Roles
api_oauth = PureCloudPlatformClientV2.OAuthApi(api_client)
api_authorization=PureCloudPlatformClientV2.AuthorizationApi(api_client)
try:
    oauth_api = api_oauth.get_oauth_client(CLIENT_ID)
    roleid_list=oauth_api.role_divisions
    role_detail_list=[]
    for role in roleid_list:
            role_detail_name=api_authorization.get_authorization_role(role.role_id).name        
            role_detail_list.append(role_detail_name)

    print("Successfully connected with roles: ",role_detail_list)
except ApiException as e:
        print("Exception when calling OAuthApi->get_oauth_client: %s\n" % e)
        exit()
```



### Check Datatable name unique/get same schema/exist on destination

```
architect_api = PureCloudPlatformClientV2.ArchitectApi(api_client)
api_response = architect_api.get_flows_datatables(expand=expand, name=datatable_name)

if len(api_response.entities)>0: #If Datatable exist on destination
    datatable_id= api_response.entities[0].id
    # Check if DT destination have same schema
    schema_dest = api_response.entities[0].schema
    if schema_dest == schema:
            migrate_option = input("Datatable name already exist on destination. Migrate all rows instead?(y/n)")            
            if migrate_option == 'y':
                try:
                # Get rows on destination
                    page_number=1
                    page_size=500
                    api_getrows_dest = architect_api.get_flows_datatable_rows(datatable_id, page_number=page_number, page_size=page_size, showbrief=True)
                    api_getrows_list_dest = api_getrows_dest.entities       
                    total_pages= api_getrows_dest.page_count
                    while total_pages > page_number:
                        page_number += 1
                        api_newgetrows = architect_api.get_flows_datatable_rows(datatable_id, page_number=page_number, page_size=page_size, showbrief=True)
                        api_getrows_list_dest.extend(api_newgetrows.entities)
                        time.sleep(0.1)
                    
                    array_keys_d = [getrows["key"] for getrows in api_getrows_list_dest] #array de keys destino
                    # Add rows to the data table (update or new)
                    getrows_list = json.dumps(api_getrows_list)
                    getrows_loads = json.loads(getrows_list)
                    print("Wait until all rows are completed..." )
                    for row in tqdm.tqdm(getrows_loads):
                        row_id=row["key"]
                        wait_secs=0.0
                        try:
                            if row_id in array_keys_d:
                                wait_secs=0.0
                                api_setrows = architect_api.put_flows_datatable_row(datatable_id, row_id, body=row)
                                time.sleep(wait_secs) 
                            else:
                                wait_secs=0.0
                                api_setrows = architect_api.post_flows_datatable_rows(datatable_id, row)
                                time.sleep(wait_secs) 
                        except ApiException as e:
                            status= getattr(e,"status")
                            if status not in (429):
                                print("Exception when calling ArchitectApi->post_flows_datatable_rows: %s\n" % e)
                                exit()
                            else: # Hints 429 Too Many Requests
                                h = e.headers
                                wait_secs = h['Retry-After']
                                continue     
                    print("")
                except ApiException as e:
                    print("Exception when calling ArchitectApi->get_flows_datatable_rows: %s\n" % e)
                    exit()
            else:
                exit()
    else:
        print("Error. Datatable schema  mismatch")
        exit()
```

### If Datatable not exist, should create and migrate data, if needed


```
else:
    #### Create Datatable on destination
    # Get division id by name and prepare Schema
    authorization_api = PureCloudPlatformClientV2.ObjectsApi(api_client);
    try:
        datatable_divisions_api = authorization_api.get_authorization_divisions(name=datatable_division_name)
        division_id = datatable_divisions_api.entities[0].id
        division_name = datatable_divisions_api.entities[0].name
        
    except ApiException as e:
        print("Exception when calling ObjectsApi->get_authorization_divisions: %s\n" % e)
        exit()
        
    datatable_schema = {
        'name': datatable_name,
        'description': datatable_description,
        'schema': schema,
        'division': {
                 'id': division_id,
                 'name': division_name
        }
    }
    try:
        datatable = architect_api.post_flows_datatables(datatable_schema)
        print("Successfully created table.")
        print("")
        migrate_option = input("Migrate all rows?(y/n)")            
        if migrate_option == 'y':
            #statusjob="WaitingForUpload"
            body = PureCloudPlatformClientV2.DataTableImportJob()
            body.import_mode="ReplaceAll"
            #body.status=statusjob
            api_importjob= architect_api.post_flows_datatable_import_jobs(datatable.id, body)
            upload_URI = api_importjob.upload_uri
            fileopt = open(temp_file_name, 'rb')
            files = {'file': fileopt }
            headers = {"Authorization": "Bearer " +tokenDestination }
            requestURI_upload=requests.post(upload_URI, headers=headers, verify=False, files=files)
            if  requestURI_upload.ok:
                 fileopt.close()
                 os.remove(temp_file_name)
                 print("Datatable rows uploaded successfully" )
                 print("")
            else:
                print("Error while uploading Datatable rows: ", )
                pprint(requestURI_upload)
        else:
            exit()
    except ApiException as e:
        print("Exception when calling ArchitectApi->post_flows_datatables: %s\n" % e)
        exit()
```










