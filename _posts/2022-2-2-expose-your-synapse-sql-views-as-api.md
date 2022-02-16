#  (Work In Progress) Exposing your Azure Synapse database views as a rest api, source code to be added later
For a customer I was tasked with figuring out an easy way of exposing some database views as an API.

The solution I came up with consists of three components:
- Azure Function, for translating HTTP requests into SQL queries, and for managing access to the database
- Single Page Application, for calling the API in a browser, using a AD user identity
- Application API Caller, for calling the api from an application, using a service principal identity

The caller of the API can select specific schemas and views by formatting the url

GetViews route:
`"schema/{schema}/views"`
User can select specific schema using this pattern {schema} can be a value provided by the user. 


GetView route:
`"schema/{schema}/view/{view}"`
User can query a specific view, in a specific schema, using the {schema} and {view} parameters. 


# Architecture/Token Flow

<figure> 
        <img src="/assets/images/tokenflow.png" />
        <figcaption>Token flow</figcaption>
</figure>

User flow: 
1. User opens SPA hosted in Azure Static Web App
2. User logs in to Active Directory 
3. Token is stored in browser session
4. User makes call to function app with token
5. Function App verifies token
6. Function App exchanges token for token with DB access
7. Function App translates request to SQL query, using querystring
8. Function App executes query on SQL db using access token
9. Function App converts results into JSON
10. Function app returns data

Service Principal flow (green):
1. App gets access token from Active Directory
2. App makes call to function app with token
3. Function App verifies token
4. Function App translates request to SQL query, using querystring
8. Function App executes query on SQL db using access token
6. Function App converts results into JSON
7. Function app returns data

## Azure Function
The Azure Function is written in .NET Framework 5.0 and runs in [isolated mode](https://docs.microsoft.com/en-us/azure/azure-functions/dotnet-isolated-process-guide)

For local development, credentials and other configuration are stored in local.settings.json (not commited in this repository see example.local.settings.json for an example) and config.json.

In the Azure environment, credentials are managed using ***Azure Keyvault***.  The Azure Function is configured to have access to this keyvault.


## Authentication
Before a HTTP request is processed, the function uses Active Directory authentication to verify the identity of the caller. This is configured in the "Authentication" tab of the Azure Function. 
The auth process for a service principal is different from a normal AD user. To solve this problem, the caller of the API must provide the "iuser" header. This can contain a value of either "true" if its a service principal calling the api, or "false", if its a normal user making the call. The function then decides the appropriate flow for logging in to the database.

The Azure Function also supports a traditional database connection (for debug purposes), this can be initialized using the DatabaseConnector class.

## SQL
Identity of user/serviceprincipal is used to connect to database

More on how to add service principal to DB here:
https://techcommunity.microsoft.com/t5/azure-sql-blog/azure-ad-service-principal-authentication-to-sql-db-code-sample/ba-p/481467 

A user can provide options in the querystring of the request. These options are translated to a SQL query using [SQLKata](https://sqlkata.com/docs).
Example: http://localhost:7071/api/schema/api-test/view/meta.getAllColumns?offset=0&limit=5&where={'column':'name','operator':'=','value':'rsid'}",
This request will get the first 5 rows out of the meta.getAllColumns view. Furthermore, it will only return the rows of which the name column is equal to rsid. 
Right now only the following options are added: 

- Limit
- Offset
- Where 
- OrderBy 

``` 
foreach(string key in queryDictionary.Keys)
            {
                switch(key)
                {
                    case "limit":
                        queryBase.Limit(int.Parse(queryDictionary[key].ToString()));
                        break;
                    case "offset":
                        queryBase.Offset(int.Parse(queryDictionary[key].ToString()));
                        break;
                    case "where":
                        JObject json = JObject.Parse(queryDictionary[key].ToString());
                        queryBase.Where(json["column"].ToString(), json["operator"].ToString(), json["value"].ToString());
                        break;
                    case "orderBy":
                        JObject jsonOrderBy = JObject.Parse(queryDictionary[key].ToString());
                        queryBase.OrderBy(jsonOrderBy["column"].ToString(), jsonOrderBy["order"].ToString());
                        break;
                }
            }
```

This piece of code is responsible for translating the querystring into a SQL query. As you can see its concise and easy to read.
Using SQLKata it is trivial to add more filtering options. 

## Active Directory
The active directory configuration for this POC consists of three parts, an app registration for the SPA, an app registration for the service principal (application calling the api) and an app registration for the backend (the azure function which is being called).

WebApi is the app registration for the backend. Its permissions look as following: 

<figure> 
        <img src="/assets/images/backendreg.png" />
        <figcaption>Azure Backend Registration</figcaption>
</figure>

If you have trouble finding the SQL permissions, you can add them to the backend using this command: 

```az ad app permission add --id 4a655c68-c7ec-4f3d-ab1f-463125368f9a --api 022907d3-0f1b-48f7-badc-1ba6abab6d66 --api-permissions c39ef2d1-04ce-46dc-8b5f-e9a5c60f0fc9=Scope```

Be sure to replace the first id *4a655c68-c7ec-4f3d-ab1f-463125368f9a* with your own client id. You may be prompted to perform a second command to activate the permissions, do so.

After adding these permissions, expose the api:


<figure> 
        <img src="/assets/images/azuresqlperm.png" />
        <figcaption>Azure SQL Permissions</figcaption>
</figure>

Now the permissions of the app registration for the service principal and the SPA look like this: 


<figure> 
        <img src="/assets/images/callerappreg.png" />
        <figcaption>Caller App Registration</figcaption>
</figure>

# SPA 

The single page application is a simple HTML/Javascript page. It is currently hosted using Azure Static Web Apps. It allows the user to sign in to Active Directory, it stores this token and uses it when sending requests to the azure function.

# Application API Caller

This is a small demonstration of how a service principal could call the Azure Function. It is written in NodeJS.

## Possible improvements/todos before taking this to production
- Add proper pagination defaults (e.g. dont allow user to get over 100 rows at once)
- Allow end user to discover API options through API
- Store connection objects so you can reuse the same connection instead of opening/closing the db connection for every request
- Cache requests
- Fix "getallviews" route, maybe we dont need a stored procedure for this
- Consider refactoring query builder to get rid of switch statement
- SqlKata says that it protects against SQL injection, verify this using automated testing on the azure function.
- CI/CD setup
- Error handling
