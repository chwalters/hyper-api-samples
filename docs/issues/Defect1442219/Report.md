# Summary
There is a bug in the Tableau SDKs that causes publishing a Hyper file with more than 2 related tables to fail with an error 400011: Bad Request.  Publishing a Hyper file with 2 related tables works fine; adding any more than 2 will fail.  This document describes the steps to reproduce the issue.  This bug might either be in the Hyper SDK itself, or possibly in the Tableau Server REST API; or the Python TSC library.

## Background
The Hyper API is a set of functions published by Tableau to manipulate Hyper files.  Hyper files are the native format for data extracts in Tableau.  According to Tableau's [documentation][1] the Hyper API allows developers to:

* Create extract files for data sources not currently supported by Tableau.
* Automate custom extract, transform and load (ETL) processes (for example, implement rolling window updates or custom incremental updates).
* Retrieve data from an extract file.
* Create, read, update, and delete data in .hyper files (also known as CRUD operations).
* Leverage the full speed of Hyper for creating and updating extract files.
* Load data directly from CSV files, much faster, and without having to write special code to do so.
* Use the power of SQL to interact with data in .hyper files. The Hyper API provides methods for executing SQL on .hyper files.

To programmatically publish the extracts to Tableau Server, developers use the [Tableau Server REST API][2] or the [Tableau Server Client (Python) library (TSC)][3].

Tableau provides a set of [examples][4] that show 1) how to create a multi-table Hyper file that has embedded relationships, and 2) how to publish the data using the REST API or TSC.

We started using the Hyper SDK to automate an ETL process from a non-supported data source (Universe from Rocket).  We quickly discovered that our data files failed to publish.  So in order to learn more about the capabilities of the SDK, or whether we were using the SDK incorrectly, we tried to use the examples published on Github.

## Steps to Reproduce the Problem

### Clone the Example Repo, Install the Hyper API

```
% git clone git@github.com:tableau/hyper-api-samples.git
% cd hyper-api-samples
% python3 --version
Python 3.8.6
% python3 -m venv venv
% source venv/bin/activate
% pip install tableauhyperapi
% pip install tableauserverclient==0.10
```

### Use the Example to Create a 2-table Hyper file and Publish to Tableau

```
cd Community-Supported/publish-multi-table-hyper

```
You will need to update the file `config.json` with values that work for you (I cannot share our connection details):

```
{
    "hyper_name": "Test.hyper",
    "server_address": "my.tableau.server",
    "site_name": "my_site",
    "project_name": "my_project",
    "tableau_token_name": "token_name",
    "tableau_token": "token_goes_here"
}
```
Next, run the example after ensuring you have valid values for your configuration.  This should work and produce output similar to what is shown below (redacted for confidentiality):

```
% python publish-multi-table-hyper.py
Config file loaded.
Creating hyper file for publishing.
Signing into *redacted* at https://us-east-1.online.tableau.com/
Publishing Test.hyper to *redacted*...
Datasource published. Datasource ID: 45*******43*******bb***2
% 
```

### View the Result in Tableau Desktop

Open Tableau Desktop.  Your new Hyper file data source uploaded in the previous step is now usable.  Note that the relationships and data types are pre-defined; no additional steps are required for the user to start creating a report/workbook.  The data source is completely inferred by Tableau Server.  Our entire automation requirement is driven by the desire to have *predefined relationships preserved in the resulting data source* which is why we want to use the Hyper SDK for our project in the first place.

![Connect to Data Source in Tableau Desktop][5]

Double click to use the data source in a new workbook.

![Using the Data Source in Tableau Desktop][6]

As you can see, creating the Hyper file and publishing to Tableau has worked perfectly with the default example.

Now, we can illustrate the bug by adding a very simple third table.

### Modify the Example to Add a third table

Modify the Python script to add a third table.  In this example, we will add a 'Customer Emails' table that is constructed in the same way as the other tables.

Add at line 45:

```
customer_emails_table = TableDefinition(
    table_name="CustomerEmails",
    columns=[
        TableDefinition.Column(name="Email Address", type=SqlType.varchar(1024), nullability=NOT_NULLABLE),
        TableDefinition.Column(name="Customer ID", type=SqlType.text(), nullability=NOT_NULLABLE)
    ]
)

```
Add at line 68:

```
connection.catalog.create_table(table_definition=customer_emails_table)
```

After defining the new table, we want to define the relationship, just like the example does for the other tables.

After previous insertion, new code can be inserted at line 76;

```
connection.execute_command(f'''ALTER TABLE {customer_emails_table.table_name} ADD ASSUMED FOREIGN KEY 
     ("Customer ID") REFERENCES  {customer_table.table_name} ( "Customer ID" )''')
```

Now create some sample data in the new table, adding at approx Line 100

```
# Insert data into Customer Emails table
email_data_to_insert = [
    ["DK-13375", "test@abc.com"],
    ["DK-13375", "test@anotherdomain.com"],
    ["EB-13705", "blah@foobar.com"]
 ]

with Inserter(connection, customer_emails_table) as inserter:
    inserter.add_rows(rows=email_data_to_insert)
    inserter.execute()
```

### Attempt to Publish the Result to Tableau

Now run the program again.  In my case, I have changed the hyper table name to explicitly illustrate that we are trying to upload a new file; not overwrite an existing one.

```
% python publish-multi-table-hyper.py
Config file loaded.
Creating hyper file for publishing.
Signing into *redacted* at https://us-east-1.online.tableau.com/
Publishing TestMulti.hyper to *redacted*...
Traceback (most recent call last):
  File "publish-multi-table-hyper.py", line 162, in <module>
    publish_hyper(config['tableau_token_name'], config['tableau_token'], config['site_name'], config['server_address'],
  File "publish-multi-table-hyper.py", line 138, in publish_hyper
    datasource = server.datasources.publish(datasource, path_to_database, publish_mode)
  File "/Users/walters/Projects/FOSS/hyper-api-samples/venv/lib/python3.8/site-packages/tableauserverclient/server/endpoint/endpoint.py", line 127, in wrapper
    return func(self, *args, **kwargs)
  File "/Users/walters/Projects/FOSS/hyper-api-samples/venv/lib/python3.8/site-packages/tableauserverclient/server/endpoint/endpoint.py", line 165, in wrapper
    return func(self, *args, **kwargs)
  File "/Users/walters/Projects/FOSS/hyper-api-samples/venv/lib/python3.8/site-packages/tableauserverclient/server/endpoint/endpoint.py", line 165, in wrapper
    return func(self, *args, **kwargs)
  File "/Users/walters/Projects/FOSS/hyper-api-samples/venv/lib/python3.8/site-packages/tableauserverclient/server/endpoint/datasources_endpoint.py", line 204, in publish
    server_response = self.post_request(url, xml_request, content_type)
  File "/Users/walters/Projects/FOSS/hyper-api-samples/venv/lib/python3.8/site-packages/tableauserverclient/server/endpoint/endpoint.py", line 99, in post_request
    return self._make_request(self.parent_srv.session.post, url,
  File "/Users/walters/Projects/FOSS/hyper-api-samples/venv/lib/python3.8/site-packages/tableauserverclient/server/endpoint/endpoint.py", line 55, in _make_request
    self._check_status(server_response)
  File "/Users/walters/Projects/FOSS/hyper-api-samples/venv/lib/python3.8/site-packages/tableauserverclient/server/endpoint/endpoint.py", line 70, in _check_status
    raise ServerResponseError.from_response(server_response.content, self.parent_srv.namespace)
tableauserverclient.server.endpoint.exceptions.ServerResponseError: 

	400011: Bad Request
		There was a problem publishing the file 'TestMulti.hyper'.
 
```

All of the modifications to the original example code described in this document can be found in [this Github repo][7]; this is a fork of the original examples.


  [1]: https://help.tableau.com/current/api/hyper_api/en-us/index.html
  [2]: https://help.tableau.com/current/api/rest_api/en-us/REST/rest_api.htm
  [3]: https://tableau.github.io/server-client-python/#
  [4]: https://github.com/tableau/hyper-api-samples
  [5]: https://github.com/chwalters/hyper-api-samples/blob/main/docs/issues/Defect1442219/TDI_00.png?raw=true
  [6]: https://github.com/chwalters/hyper-api-samples/blob/main/docs/issues/Defect1442219/TDI_01.png?raw=true
  [7]: https://github.com/chwalters/hyper-api-samples