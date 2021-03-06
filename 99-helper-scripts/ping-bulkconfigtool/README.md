# Ping Identity Bulk Config Tools

## This document has been deprecated. [Please see this document for bulk config tool information](../../docs/how-to/buildPingFederateProfile.md).

The bulk export process performs the following:
1. Extract configuration from a local sandbox deployment.
2. Process the JSON export provided a process configuration. For more information, see [Building a PingFederate profile from your current deployment](../../docs/how-to/buildPingFederateProfile.md).
    - Search and replace (e.g. hostnames)
    - Cleans, add, and remove JSON members as required.
    - Tokenise the configuration and maintain environment variables.
    - Outputs pretty JSON.

The bulk export process also allows config injection for secrets and keys.


## Pre-requisites

The bulk export utility comes in pre-compiled source code. To build the project, you'll need:
- JDK 11
    - JAVA_HOME and PATH environment settings.
      - JAVA_HOME=/path/to/java
      - PATH=$PATH:$JAVA_HOME/bin
    - https://adoptopenjdk.net/
- Maven
    - MAVEN_HOME and PATH environment settings.
      - MAVEN_HOME=/path/to/maven
      - PATH=$PATH:$MAVEN_HOME/bin
    - https://maven.apache.org/

You'll also need a running instance of PingFederate and/or PingAccess.
  - If you have changed the default settings (e.g. hostnames) you'll need to configure the configuration json files [pa-config.json](./in/pa-config.json), [pa-admin-config.json](./in/pa-config-addconfigquery.json), and [pf-config.json](./in/pf-config.json).

## A little bit about the input configuration

The bulk configuration consumes a configuration file to help identify information it needs to extract and manipulate. It provides configuration to:
- search and replace
- change element values
- remove configuration
- add configuration
- expose parameters as substitutions

### Commands
#### search-replace
- A simple utility to search and replace string values in a bulk config json file.
- Can expose environmental variables.

Example: replacing an expected base hostname with a substition.
```
  "search-replace":[
    {
      "search": "data-holder.local",
      "replace": "${BASE_HOSTNAME}",
      "apply-env-file": false
    }
  ]
```
#### change-value
- Searches for elements with a matching identifier, and updates a parameter with a new value.

Example: update keyPairId against an element with name=ENGINE.
```
  "change-value":[
  	{
          "matching-identifier":
          {
          	"id-name": "name",
          	"id-value": "ENGINE"
          },
  	  "parameter-name": "keyPairId",
  	  "new-value": 8
  	}
  ]
```

#### remove-config
- Allows us to remove configuration from the bulk export.

Example: you may wish to remove the ProvisionerDS data store:
```
  "remove-config":[
  	{
  	  "key": "id",
	  "value": "ProvisionerDS"
  	}
  ]
```

Example: you may wish to remove all SP Connections:
```
  "remove-config":[
  	{
  	  "key": "resourceType",
	  "value": "/idp/spConnections"
  	}
  ]
```

#### add-config
- Allows us to add configuration to the bulk export.

Example: you may wish to add the CONFIG QUERY http listener in PingAccess
```
  "add-config":[
	  {
	    "resourceType": "httpsListeners",
	    "item":
			{
			    "id": 4,
			    "name": "CONFIG QUERY",
			    "keyPairId": 5,
			    "useServerCipherSuiteOrder": true,
			    "restartRequired": false
			}
	  }
  ]
```

Example: you may wish to add an SP connection
```
  "add-config":[
	  {
	    "resourceType": "/idp/spConnections",
	    "item":
		{
                    "name": "httpbin3.org",
                    "active": false,
		    ...
		}
	  }
  ]
```

#### expose-parameters
- Navigates through the JSON and exchanges values for substitions.
- Exposed substition names will be automatically created based on the json path.
    - E.g. ${oauth_clients_items_clientAuth_testclient_secret}
- Can convert encrypted/obfuscated values into clear text inputs (e.g. "encryptedValue" to "value") prior to substituting it. This allows us to inject values in its raw form.

Example: replace the "encryptedPassword" member with a substitution enabled "password" member for any elements with "id" or "username" members. The following will remove "encryptedPassword" and create "password": "${...}".
```
    {
      "parameter-name": "encryptedPassword",
      "replace-name": "password",
      "unique-identifiers": [
          "id",
          "username"
      ]
    }
```

#### config-aliases
- The bulk config tool generates substitution names, however sometimes you wish to simplify them or reuse existing environment variables.

Example: Renaming the Administrator's substitution name to leverage the common PING_IDENTITY_PASSWORD environmental variable.
```
  "config-aliases":[
	{
	  "config-names":[
	    "administrativeAccounts_items_Administrator_password"
	  ],
  	  "replace-name": "PING_IDENTITY_PASSWORD",
  	  "is-apply-envfile": false
  	}
  ]
```

#### sort-arrays
- Configure the array members that need to be sorted. This ensures the array is created consistently to improve git diff.

Example: Sort the roles and scopes arrays.
```
  "sort-arrays":[
        "roles","scopes"
  ]
```

## Run the export utility.

1. In terminal, navigate to the ping-bulkconfigtools folder.
2. Compile the tool if you haven't already done so.
    - cmd: ./_compile_bulkexporttool.sh
3. Edit env.properties and configure details for PA and/or PF.
3. Export PingAccess configuration
    - cmd: ./_pa_export-config.sh
    - GETs configuration from: {{PINGACCESS_ADMIN_BASEURL}}/pa-admin-api/v3/config/export
    - This will then create 2 exports:
        1) ./out/pingaccess/standalone/data.json.subst
        2) ./out/pingaccess/clustered/data.json.subst (contains CONFIG QUERY http listener).
    - Creates/maintains the following environment variable files:
      - ./out/pingaccess/pa.env
4. Export PingFederate configuration
    - cmd: ./_pf_export-config.sh
    - GETs configuration from: {{PINGFEDERATE_ADMIN_BASEURL}}/pf-admin-api/v1/bulk/export
    - This will then create the following:
        1) ./out/pingfederate/import-bulkconfig.json.subst
    - Creates/maintains the following environment variable files:
      - ./out/pingfederate/pf.env

## Configure and commit

You'll need to configure the following environment variable files when deploying. The export process will maintain values inside these files that have been previously set however, new parameters may be present so you should look out for them.
- ./out/pingaccess/pa.env
- ./out/pingfederate/pf.env

You do not need to commit the environment variables. You should consider excluding these files from being committed as they may contain sensitive information such as certificate keys and passwords.

Commit the following files into the correct locations of your server profile:
- ./out/pingaccess/data.json.subst
- ./out/pingfederate/import-bulkconfig.json.subst

## Known Issues
Bulk configuration requires the ability for PingFederate to reload configuration while running. When introduced, certain elements in hivemodule requires model="autoreloadable" to allow reloading of configuration.

If you have issues, you may have inherited an old hivemodule.xml in your server profile. Check and compare it aligns to the version of PingFederate that you are running.

### LDAP authentication to Admin Console

PingFederate can be configured to enable LDAP authentication to the admin console (web/api). This has not been tested and may require alterations to the startup deployer calls stored under server/default/drop-in-config.

## Appendix

### Applying encoded file configuration

PingFederate and PingAccess both provide API's to import file content. This may be used for:
- Importing certificates and PKCS12 keystores.
- Importing property or binary files to adapter configuration.

The bulk export process will expose fileData configuration options in the environment variable files (i.e. pf.env and pa.env). This provides the administrator the ability to inject certificate/file based configuration.

You'll need to base64 encode into a JSON friendly value:
1. Encode the file content in base64,
2. Remove line breaks, and
3. Escape back slashes.

Here's a command that can do that for you:
- openssl base64 -in ~/Downloads/admin_signing.p12 | tr -d '\n' | sed 's/\\\//\\\\\\\//g'
