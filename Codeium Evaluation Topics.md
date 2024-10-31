# Codeium Evaluation Topics

This is a list of topics that an AI code assistant may help in generating artifacts for the TIBCO BusinessEvents (BE) during application development.

## Auto-complete Tasks in VS Code

- [x] Given samples of rules, it can prompt the correct block structure for BE rules as shown in the sample below.  
- [x] Given samples of rule-functions, it can prompt the correct block structure for BE rule-functions as shown in the sample below.  
- [x] Given sample calls to \~1,000 BE library functions and comments describing each function, it can prompt correct calls to BE library functions based on commented descriptions when editing a BE rule or rule-function.  For example, the calls to Log.log(), or Event.consumeEvent() in the rule-function example below are BE library functions.   
- [ ] Prompt correct constructor function calls for a concept or an event.  A constructor lists all attributes of a specified concept or event that are defined in a corresponding XML definition as shown in samples below.  For example, in the sample of BE rule below, the calls to create an instance of ShoppingItem and HTTPStatus are constructor functions for the specified concept and event.  
- [x] Prompt multi-line code including the previous items in the middle of a rule or a rule-function based on description in a comment.  
- [ ] Given access to a repository of multiple BE sample applications, the code style and suitable code blocks in the provided samples may be prompted when editing rules or rule-functions in a new BE application.

## Chat Tasks in VS Code

- [ ] Given a list of attributes, it can generate a BE concept definition in XML as shown below.  Note that LLM may be used to extract attributes only, and then a command script can be used to write the XML file.  
- [ ] Given a list of properties, it can generate a BE event definition in XML as shown below. Note that LLM may be used to extract properties only, and then a command script can be used to write the XML file.  
- [x] Given name, folder, and description of activity steps of a rule, it can generate a BE rule of the specified name at the specified location (which is also used as the package prefix).  The rule action block will implement the specified activities that call suitable BE library functions according to the description.   
- [x] Given name, folder, and description of activity steps of a rule-function, it can generate a BE rule-function of the specified name at the specified location (which is also used as the package prefix).  The body of the rule-function will implement the specified activities that call suitable BE library functions according to the description.   
- [ ] Given some key options, it can generate a channel definition in XML as shown below with or without destination elements.  Note that LLM may be used only to extract key options and specified destination names, and then a command script can be used to write the XML file.  
- [ ] Given a basic description of a usecase, the channel to use, basics around the deployment descriptor, concept/event definition, rule/rulefunction details, it should be able to create a project with standard folder structure, create necessary artifacts with the correct content and put them in the appropriate folders.  
- [ ] Given a project, it should be able to answer “what does this project do?”, explaining functional and technical details like which channel is being used, what are the incoming/outgoing events, etc.  
- [ ] Given the existing documentation, it should be able to answer simple questions like, what are the configuration for Kafka Destination or what properties are configured for Ignite or more contextual questions like, “Given i have a usecase which needs persistent storage which involves more add/updates/delete than get’s, what would be right deployment configuration”

## Commands in VS Code

- [ ] The tasks in the chat section may also be commands that create an empty concept, event, rule, rule-function, or channel.

## Sample BE Artifacts

### Sample BE [rule-function](https://github.com/yxuco/Petstore/blob/main/RuleFunctions/Preprocessors/onListInventory.rulefunction)

```java
/**
 * @description 
 */
void rulefunction RuleFunctions.Preprocessors.onListInventory {
	attribute {
		validity = ACTION;
	}
	scope {
		Events.ListInventory evt;
	}
	body {
		Object logger = Log.getLogger("RuleFunctions.Preprocessors.onListInventory");
		String[] skus = RuleFunctions.productList();
		Log.log(logger, "info", "List %d inventory items", skus@length);
		Event.replyEvent(evt, Events.HTTPStatus.HTTPStatus(null, RuleFunctions.inventoryToJSON(skus), "200"));
		Event.consumeEvent(evt);
	}
}
```

### Sample BE [rule](https://github.com/yxuco/Petstore/blob/main/Rules/AddItemToCart.rule)

```java
/**
 * @description 
 * @author Pro-Branes.local
 */
rule Rules.AddItemToCart {
	attribute {
		priority = 5;
		forwardChain = true;
	}
	declare {
		Events.AddItemToCart evt;
		Concepts.Product prod;
		Concepts.ShoppingCart cart;
	}
	when {
		prod.sku == evt.sku;
		cart.userName == evt.user;
	}
	then {
		Object logger = Log.getLogger("Rules.AddItemToCart");
		double amount = evt.unit * prod.price;
		cart.items[cart.items@length] = Concepts.ShoppingItem.ShoppingItem(
				null /*extId String */,
				evt.sku /*sku String */,
				evt.unit /*unit long */,
				amount /*amount double */);
		cart.amount += amount;
		Log.log(logger, "info", "Added %d units of SKU %s to shopping cart of %s", evt.unit, evt.sku, evt.user);
		Event.replyEvent(evt, Events.HTTPStatus.HTTPStatus(null,RuleFunctions.cartToJSON(cart),"200"));
		Event.consumeEvent(evt);
	}
}
```

### Sample BE [concept definition](https://github.com/yxuco/Petstore/blob/main/Concepts/ShoppingItem.concept)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<element:Concept xmi:version="2.0" 
  xmlns:xmi="http://www.omg.org/XMI" 
  xmlns:element="http:///com/tibco/cep/designtime/core/model/element" 
  folder="/Concepts/" name="ShoppingItem" description="" 
  GUID="16D7E7B5-E6FD-038E-7BC2-FAC7A985F2CE" ownerProjectName="Petstore" 
  superConceptPath="" autoStartStateMachine="true">
    <extendedProperties/>
    <properties name="sku" ownerProjectName="Petstore"/>
    <properties name="unit" ownerProjectName="Petstore" type="long"/>
    <properties name="amount" ownerProjectName="Petstore" type="double"/>
</element:Concept>
```

### Sample BE [event definition](https://github.com/yxuco/Petstore/blob/main/Events/HTTPStatus.event)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<event:SimpleEvent xmi:version="2.0" xmlns:xmi="http://www.omg.org/XMI" 
  xmlns:event="http:///com/tibco/cep/designtime/core/model/event" 
  folder="/Events/" name="HTTPStatus" description="" 
  GUID="5AEA0077-5579-BC1D-AD85-B00AEBDCE43C" ownerProjectName="Petstore" 
  superEventPath="" ttlUnits="Seconds">
    <properties name="HttpStatusCode" ownerProjectName="Petstore"/>
    <expiryAction name="HTTPStatus_expiryAction" ownerProjectName="Petstore" 
      actionText="" conditionText="">
        <symbols>
            <symbolList idName="httpstatus" type="/Events/HTTPStatus"/>
        </symbols>
    </expiryAction>
</event:SimpleEvent>
```

### Sample BE [channel definition](https://github.com/yxuco/Petstore/blob/main/api/petstore.channel)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<channel:Channel xmi:version="2.0" xmlns:xmi="http://www.omg.org/XMI" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:channel="http:///com/tibco/cep/designtime/core/model/service/channel" xmlns:designtime="http:///com/tibco/cep/designtime/core/model/designtime_ontology.ecore" folder="/api/" name="petstore" description="" GUID="E4E4908A-FA36-8339-91C6-6127EE6E88DA" ownerProjectName="Petstore">
 <driver xsi:type="channel:HttpChannelDriverConfig" configMethod="REFERENCE" reference="/SharedResources/http.sharedhttp" label="HTTP" channel="/">
   <destinations folder="/api/cart/" name="addItemToCart" description="" GUID="D0B75E27-CBAD-4443-E661-B14E01FFF216" ownerProjectName="Petstore" eventURI="/Events/AddItemToCart" serializerDeserializerClass="com.tibco.cep.driver.http.serializer.RESTMessageSerializer" driverConfig="//@driver">
     <properties>
       <properties xsi:type="designtime:SimpleProperty" name="be.http.ResponseEvent" value=""/>
       <properties xsi:type="designtime:SimpleProperty" name="be.http.jsonPayload" value="false"/>
       <properties xsi:type="designtime:SimpleProperty" name="be.http.IncludeEventType" value="ALWAYS"/>
       <properties xsi:type="designtime:SimpleProperty" name="be.http.type" value="DEFAULT"/>
       <properties xsi:type="designtime:SimpleProperty" name="be.http.contextPath" value=""/>
       <properties xsi:type="designtime:SimpleProperty" name="be.http.pageFlowFunction" value=""/>
     </properties>
   </destinations>
   <destinations folder="/api/cart/" name="submitOrder" description="" GUID="62F028E1-8853-05A5-5BF0-4230825F5227" ownerProjectName="Petstore" eventURI="/Events/SubmitOrder" serializerDeserializerClass="com.tibco.cep.driver.http.serializer.RESTMessageSerializer" driverConfig="//@driver">
     <properties>
       <properties xsi:type="designtime:SimpleProperty" name="be.http.ResponseEvent" value=""/>
       <properties xsi:type="designtime:SimpleProperty" name="be.http.jsonPayload" value="false"/>
       <properties xsi:type="designtime:SimpleProperty" name="be.http.IncludeEventType" value="ALWAYS"/>
       <properties xsi:type="designtime:SimpleProperty" name="be.http.type" value="DEFAULT"/>
       <properties xsi:type="designtime:SimpleProperty" name="be.http.contextPath" value=""/>
       <properties xsi:type="designtime:SimpleProperty" name="be.http.pageFlowFunction" value=""/>
     </properties>
   </destinations>
   <extendedConfiguration>
     <properties name="serverType" value="TOMCAT"/>
   </extendedConfiguration>
   <driverType xsi:type="channel:DriverTypeInfo" driverTypeName="HTTP"/>
   <choice value="TOMCAT">
     <extendedConfiguration>
       <properties name="be.http.connectorType" value="NIO"/>
       <properties name="be.http.connectorInstances" value="1"/>
       <properties name="be.http.debug" value="false"/>
       <properties name="be.http.debugFolder" value=""/>
       <properties name="be.http.debugLogPattern" value="%{yyyy MMM dd HH:mm:ss.SSS 'GMT'X}t %A %I [%m] '%U' [%s] %bbytes %Dms"/>
       <properties name="be.http.debugLogRotatable" value="true"/>
       <properties name="be.http.debugLogFilePattern" value=".yyyy-MM-dd"/>
       <properties name="be.http.debugLogRenameOnRotate" value="false"/>
       <properties name="be.http.debugLogMaxDays" value="-1"/>
       <properties name="be.http.connectionTimeout" value="60000"/>
       <properties name="be.http.acceptCount" value="-1"/>
       <properties name="be.http.socketBufferSize" value="9000"/>
       <properties name="be.http.maxProcessors" value="-1"/>
       <properties name="be.http.connectionLinger" value="-1"/>
       <properties name="be.http.enableDNSLookups" value="false"/>
       <properties name="be.http.tcpNoDelay" value="true"/>
       <properties name="be.http.compression" value="off"/>
       <properties name="be.http.useBodyEncodingForURI" value="false"/>
       <properties name="be.http.URIEncoding" value=""/>
       <properties name="be.http.maxKeepAliveRequests" value="-1"/>
       <properties name="be.http.maxHttpHeaderSize" value="4096"/>
       <properties name="be.http.maxPostSize" value="2097152"/>
       <properties name="be.http.maxSavePostSize" value="4096"/>
       <properties name="be.http.scheme" value=""/>
       <properties name="be.http.sessionTimeout" value="1800"/>
       <properties name="be.http.maxSpareThreads" value="50"/>
       <properties name="be.http.minSpareThreads" value="4"/>
       <properties name="be.http.compressableMimeType" value="text/html,text/xml,text/plain"/>
       <properties name="be.http.restrictedUserAgents" value=""/>
       <properties name="be.http.docRoot" value=""/>
       <properties name="be.http.docPage" value=""/>
       <properties name="be.http.ssl_server_useServerCipherOrder" value="false"/>
       <properties name="be.http.serverHeader" value="Apache-Coyote/1.1"/>
     </extendedConfiguration>
   </choice>
 </driver>
</channel:Channel>
```

### Sample BE cluster deployment description ([CDD](https://github.com/yxuco/Petstore/blob/main/Deployments/Petstore.cdd))

CDD is the key deployment configuration that describes the application runtime behavior.  Following are sample sections of this file.

The following section defines a runtime process engine:

```xml
  <processing-units>
       <processing-unit id="default">
           <agents>
               <agent>
                   <ref>inference-class</ref>
                   <key/>
                   <priority/>
               </agent>
           </agents>
           <logs>logConfig</logs>
           <hot-deploy>true</hot-deploy>
           <cache-storage-enabled>false</cache-storage-enabled>
           <db-concepts>false</db-concepts>
           <property-group>
               <property name="be.cluster.ruletemplateinstances.deploy.dir" 
                   value="/Users/yxu/tibco/be63/be/6.3/rms/shared/Petstore"/>
               <property name="be.engine.cluster.externalClasses.path" 
                   value="/Users/yxu/tibco/be63/be/6.3/rms/shared/Petstore/Decision_Tables"/>
           </property-group>
       </processing-unit>
   </processing-units>
```

It refers an agent class definition of inference-class, which is shown below:

```xml
  <agent-classes>
       <inference-agent-class id="inference-class">
           <rules>
               <ref>all-rules</ref>
           </rules>
           <destinations>
               <destination id="addItem_795DC577">
                   <pre-processor>/RuleFunctions/Preprocessors/onAddItemToCart</pre-processor>
                   <queue-size/>
                   <thread-count/>
                   <threading-model>shared-queue</threading-model>
                   <thread-affinity-rule-function/>
                   <uri>/api/petstore/addItemToCart</uri>
               </destination>
               <destination id="submitOrder_4ECE82C4">
                   <pre-processor>/RuleFunctions/Preprocessors/onSubmitOrder</pre-processor>
                   <queue-size/>
                   <thread-count/>
                   <threading-model>shared-queue</threading-model>
                   <thread-affinity-rule-function/>
                   <uri>/api/petstore/submitOrder</uri>
               </destination>
           </destinations>
           <startup>
               <uri>/RuleFunctions/startup</uri>
           </startup>
           <shutdown/>
           <local-cache>
               <eviction>
                   <max-size>1024</max-size>
                   <max-time>900</max-time>
               </eviction>
           </local-cache>
           <shared-queue>
               <size>1024</size>
               <workers>10</workers>
           </shared-queue>
           <load>
               <max-active/>
           </load>
           <concurrent-rtc>false</concurrent-rtc>
           <check-for-duplicates>false</check-for-duplicates>
           <businessworks>
               <uri/>
           </businessworks>
           <property-group/>
       </inference-agent-class>
   </agent-classes>
```

Each destination defines an optional pre-processor function and a reference of a channel destination that is defined in a corresponding channel definition file. 

## Observation on Simple Auto-completion Test

Open a sample BE project [Petstore](https://github.com/yxuco/Petstore) in VS Code, and install Codeium extension for AI code assistant.

Given a context file that implements a BE rule-function that showcases the usage of BE library functions for RDBMS operations, Codeium LLM can suggest relevant code snippets from the context file when a user inserts a comment in a new rule-function.

### [Context File](https://github.com/yxuco/Petstore/blob/main/TestData/FunctionSamples/rdbmFunctions.rulefunction)

```java
/**
 * @description Code examples for RDBMS library functions.  
 * It is provided as a context file for testing AI code assistant.
 */
void rulefunction TestData.FunctionSamples.rdbmFunctions {
	attribute {
		validity = ACTION;
	}
	scope {
		
	}
	body {
		// Get a logger for use inside a rule/rulefunction
		Object logger = Log.getLogger("TestData.FunctionSamples.rdbmFunctions");
		
		String jdbcResourceUri = "/SharedResources/jdbc";
		try {
			// Use case 1: Simple DB query
			
			String sql = "select sysdate from dual";
			// Log sql statement
			Log.log(logger, "info", "SQL statement: %s", sql);
		
			// Sets the connection to use for database operations.
			Database.setCurrentConnection(jdbcResourceUri);
			String resultConceptUri = "/Concepts/Count";
			boolean queryChildren = false;
			// Queries underlying database using the supplied SQL.
			Concept[] result = Database.queryUsingSQL(resultConceptUri, sql, queryChildren);
			for (int i = 0; i < result@length; i++) {
				String root = "Count";
				boolean pretty = false;
				// Serializes a Concept instance into an JSON string.
				String json = Instance.serializeToJSON(result[i], pretty, root);
				// Log the content of a concept
				Log.log(logger, "info", "Query returned concept %d of type %s: %s", i, resultConceptUri, json);
			}
		} catch (Exception e) {
			// Log DB exception
			Log.logException(logger, "error", "Failed to execute DB query %s", e, "");
		} finally {
			// Releases the current connection back to the connection pool. 
			Database.unsetConnection();
		}
		
		try {
			// Use case 2: Invoke DB stored procedure in a DB transaction
		
			// Sets the connection to use for database operations.
			Database.setCurrentConnection(jdbcResourceUri);
			
			// Begins a transaction on the current connection
			Database.beginTransaction();
			try {
				String storedProc = "ResetInventory";
				Object[] args = Object[]{"5348529", 20};
			
				// Initializes the given stored procedure
				Object sp = Database.sp.initStoredProc(storedProc);
				for (int i = 0; i < args@length; i++) {
					// Sets input parameter of the stored procedure at the given index.
					Database.sp.setInputParameter(sp, i+1, args[i]);
				}
				int outputType = 6;
				// For stored procedure with out parameters, sets the sqltype of the return type at the given index.
				// The return type options are STRING=0; INT=1; LONG=2; DOUBLE=3; BOOLEAN=4; DATETIME=5; CONCEPT=6; CLOB=7.
				Database.sp.setOutputParameterType(sp, args@length+1, outputType);
			
				// Executes the given stored procedure.
				Database.sp.executeStoredProc(sp);
			
				// Retrieve the output object returned by stored procedure at the given index. 
				Object data = Database.sp.getObjectAtIndex(sp, args@length+1);
				// Log the returned concept
				Log.log(logger, "info", "Stored procedure returned concept: %s", Instance.serializeToJSON(data, true, ""));
			
				// Commits the current transaction.
				Database.commit();
			} catch (Exception ex) {
				// Log database exception
				Log.logException(logger, "error", "Failed to execute DB stored proc %s", ex, "");
			} finally {
				Log.log(logger, "info", "Rollback DB transaction");
				// Rolls back current transaction.
				Database.rollback();
			}
		} catch (Exception e) {
			// Log DB exception
			Log.logException(logger, "error", "Failed to execute DB stored procedure %s", e, "");
		} finally {
			// Releases the current connection back to the connection pool. 
			Database.unsetConnection();
		}
		
		try {
			// Use case 3: Iterating over DB query cursor
			
			// Sets the connection to use for database operations.
			Database.setCurrentConnection(jdbcResourceUri);
			
			String sql = "select * from inventory";
			String cursor = "AllInventory";
			String resultConceptUri = "/Concepts/Inventory";
			int pageSize = 50;
			
			// Opens a database cursor for the given sql query. Once the cursor is open, a user can page on the resultset of the query
			Database.createQuery(jdbcResourceUri, cursor, resultConceptUri, sql, pageSize, null);
			
			try {
				while (true) {
					// Gets the next page from the database cursor
					Concept[] data = Database.getNextPage(cursor, pageSize);
					if (data != null) {
						Log.log(logger, "info", "DB cursor returned a page of %d concepts", data@length);
					}
					if (data == null || data@length < pageSize) {
						Log.log(logger, "info", "Reached end of DB cursor %s", cursor);
						break;
					}
				}
			} finally {
				// Closes the cursor for the query
				Database.closeQuery(cursor);
			}
		} catch (Exception e) {
			// Log DB exception
			Log.logException(logger, "error", "Failed to process query cursor %s", e, "");
		} finally {
			// Releases the current connection back to the connection pool. 
			Database.unsetConnection();
		}
	}
}
```

### Edit new rule-function

When creating a new rule-function, the correct syntactic structure for rule-functions is prompted.  Then, developer need to type in only the following 2 comment lines:

* query database for record count of inventory  
* execute stored procedure createUser to create new user of specific name and return the new user as an output parameter

Relevant code snippets are identified from the context file, and inserted into the new rule-function.  However, this is only a very simple use-case with a very unique context of valid code snippets.  We need to define an approach to benchmark more complex cases that involve not so clear-cut context provided by more general code repositories.

```java
void rulefunction TestData.queryInventory {
    attribute {
        validity = ACTION;
    }
    scope {
        
    }
    body {
        Object logger = Log.getLogger("TestData.queryInventory");
        String jdbcResourceUri = "/SharedResources/jdbc";

        // query database for record count of inventory
        String sql = "select count(*) from inventory";
        String resultConceptUri = "/Concepts/Count";
        boolean queryChildren = false;
        // Queries underlying database using the supplied SQL.
        Concept[] result = Database.queryUsingSQL(resultConceptUri, sql, queryChildren);
        for (int i = 0; i < result@length; i++) {
            String root = "Count";
            boolean pretty = false;
            // Serializes a Concept instance into an JSON string.
            String json = Instance.serializeToJSON(result[i], pretty, root);
            Log.log(logger, "info", "Query returned concept %d of type %s: %s", i, resultConceptUri, json);
        }

        // execute stored procedure createUser to create new user of specific name 
        // and return the new user as an output parameter
        String storedProc = "CreateUser";
        Object[] args = Object[]{"John Doe"};
        Object sp = Database.sp.initStoredProc(storedProc);
        for (int i = 0; i < args@length; i++) {
            // Sets input parameter of the stored procedure at the given index. Indices are 1-based.
            Database.sp.setInputParameter(sp, i+1, args[i]);
        }
        int outputType = 6;
        // For stored procedure with out parameters, sets the sqltype of the return type at the given index.
        // The return type options are STRING=0; INT=1; LONG=2; DOUBLE=3; BOOLEAN=4; DATETIME=5; CONCEPT=6; CLOB=7.
        Database.sp.setOutputParameterType(sp, args@length+1, outputType);
        // Executes the given stored procedure.
        Database.sp.executeStoredProc(sp);
        // Retrieve the output object returned by stored procedure at the given index. 
        Object data = Database.sp.getObjectAtIndex(sp, args@length+1);
        // Log the returned concept
        Log.log(logger, "info", "Stored procedure returned concept: %s", Instance.serializeToJSON(data, true, ""));
    }
}
```

## Observation on chat tasks

### Rule-function generated by chat

In the chat window, the following request would generate a reasonably correct rule-function. 

```
write a rulefunction to query DB and return record count of the inventory table.
```

However, the generated code contains 3 errors:

* It assumes that input arguments can be initialized in the block of “scope”, but BE does not allow that.  
* It picked up a call to “Event.replyEvent()” from other rule-functions in the workspace, and assumed that is valid, but it does not make sense here.  
* It assumes that a concept object contains an attribute of “value”, but it does not.

How could we avoid such issues?

```java
void rulefunction TestData.getInventoryRecordCount {
    attribute {
      validity = ACTION;
    }
    scope {
      String jdbcResourceUri = "/SharedResources/jdbc";
      String sql = "select count(*) from inventory";
      String resultConceptUri = "/Concepts/Count";
      boolean queryChildren = false;
    }
    body {
      // Get a logger for use inside a rule/rulefunction
      Object logger = Log.getLogger("TestData.FunctionSamples.getInventoryRecordCount");
      
      try {
        // Sets the connection to use for database operations.
        Database.setCurrentConnection(jdbcResourceUri);
        
        // Queries underlying database using the supplied SQL.
        Concept[] result = Database.queryUsingSQL(resultConceptUri, sql, queryChildren);
        
        // Log the record count
        Log.log(logger, "info", "Inventory record count: %d", result[0].value);
        
        // Return the record count
        Event.replyEvent(Events.HTTPStatus.HTTPStatus(null, result[0].value.toString(), "200"));
      } catch (Exception e) {
        // Log DB exception
        Log.logException(logger, "error", "Failed to execute DB query %s", e, "");
      } finally {
        // Releases the current connection back to the connection pool. 
        Database.unsetConnection();
      }
    }
  }
```

### Question on CDD and channel definitions

Add [CDD](https://github.com/yxuco/Petstore/blob/main/Deployments/Petstore.cdd) and [channel definition](https://github.com/yxuco/Petstore/blob/main/api/petstore.channel) files as context, in chat window, ask the following question:

```
List processing units, their startup function and their input destinations. For each destination, list its pre-processor, and find its corresponding eventURI from the channel definition file.
```

Codeium correctly returns the following response:

```
Based on the provided context, here is the list of processing units, their startup function, and their input destinations. For each destination, I have listed its pre-processor and found its corresponding eventURI from the channel definition file:


 * Processing Unit: default
    * Startup Function: /RuleFunctions/startup
    * Input Destinations:
       * addItem_795DC577 (/api/petstore/addItemToCart) ->
         Pre-processor: /RuleFunctions/Preprocessors/onAddItemToCart -> 
         eventURI: /Events/AddItemToCart (from api/petstore.channel)
       * submitOrder_4ECE82C4 (/api/petstore/submitOrder) -> 
         Pre-processor: /RuleFunctions/Preprocessors/onSubmitOrder -> 
         eventURI: /Events/SubmitOrder (from api/petstore.channel)
       * reload_2F94FD6D (/api/petstore/inventoryReload) -> 
         Pre-processor: (none) -> 
         eventURI: /Events/ReloadInventory (from api/petstore.channel)
       * alert_267592D1 (/api/petstore/inventoryAlerts) -> 
         Pre-processor: /RuleFunctions/Preprocessors/onListAlert -> 
         eventURI: /Events/ListAlert (from api/petstore.channel)
       * list_211B3F98 (/api/petstore/inventoryList) -> 
         Pre-processor: /RuleFunctions/Preprocessors/onListInventory -> 
         eventURI: /Events/ListInventory (from api/petstore.channel)


Note that I extracted the pre-processor information from the pre-processor elements in the destination elements, and matched the destination URIs with the eventURIs from the channel definition file (api/petstore.channel) based on the provided context.
```

This would be the first step to provide insight about the runtime behavior of a BE application.

### Explains a rule-function

For a sample [rule-function](https://github.com/yxuco/Petstore/blob/main/RuleFunctions/Preprocessors/onAddItemToCart.rulefunction) as follows:

```java
/**
 * @description 
 */
void rulefunction RuleFunctions.Preprocessors.onAddItemToCart {
	attribute {
		validity = ACTION;
	}
	scope {
		Events.AddItemToCart evt;
	}
	body {
		Object logger = Log.getLogger("RuleFunctions.Preprocessors.onAddItemToCart");
		
		Concepts.Product prod = Instance.getByExtIdByUri(RuleFunctions.extIdPrefix("Product")+evt.sku, "/Concepts/Product");
		if (prod == null) {
			Log.log(logger, "info", "Product SKU %s does not exist", evt.sku);
			// reply 404 Product not found
			Event.replyEvent(evt, Events.HTTPStatus.HTTPStatus(null, null, "404"));
			Event.consumeEvent(evt);
			return;
		}
		
		Concepts.User user = Instance.getByExtIdByUri(RuleFunctions.extIdPrefix("User")+evt.user, "/Concepts/User");
		if (user == null) {
			// create new user
			user = Concepts.User.User(
				RuleFunctions.extIdPrefix("User")+evt.user /*extId String */,
				evt.user /*name String */,
				"basic" /*status String */,
				Concepts.PurchaseOrder[]{} /*orders Concepts.PurchaseOrder[] */,
				0 /*rewardPoints long */,
				Concepts.Coupon[]{} /*coupons Concepts.Coupon[] */);
		}
		
		Concepts.ShoppingCart cart = Instance.getByExtIdByUri(RuleFunctions.extIdPrefix("ShoppingCart")+evt.user, "/Concepts/ShoppingCart");
		if (cart == null) {
			// create new shopping cart
			cart = Concepts.ShoppingCart.ShoppingCart(
				RuleFunctions.extIdPrefix("ShoppingCart")+evt.user /*extId String */,
				evt.user /*userName String */,
				Concepts.ShoppingItem[]{} /*items Concepts.ShoppingItem[] */,
				0.0 /*amount double */);
		}
		
		Log.log(logger, "info", "Adding %d units of SKU %s to shopping cart of %s", evt.unit, evt.sku, evt.user);
	}
}
```

In the chat window, ask the following question:

```
explain the function onAddItemToCart
```

Codeium returns a pretty reasonable response, except that it does not know the fact that the database is actually the working memory of the rule-engine.

```
The function onAddItemToCart is a rulefunction that is triggered when an item is added to the cart. It is defined in the file RuleFunctions/Preprocessors/onAddItemToCart.rulefunction.

Here is a high-level overview of what the function does:

 1. It checks if the product with the specified SKU exists in the database. If it doesn't, it returns a 404 error.
 2. It checks if the user with the specified username exists in the database. If they don't, it creates a new user.
 3. It checks if the user has a shopping cart. If they don't, it creates a new shopping cart for the user.
 4. It adds the item to the user's shopping cart.
 5. It updates the total amount of the shopping cart.
 6. It logs a message indicating that the item has been added to the cart.
 
The function uses several concepts and instances, including:

 * Concepts.Product: represents a product in the database
 * Concepts.User: represents a user in the database
 * Concepts.ShoppingCart: represents a shopping cart in the database
 * Events.AddItemToCart: represents the event of adding an item to the cart
 * RuleFunctions.extIdPrefix: a function that returns a prefix for external IDs

The function also uses several actions, including:

 * Instance.getByExtIdByUri: retrieves an instance by its external ID and URI
 * Instance.PropertyArray.add: adds an item to a property array
 * Log.log: logs a message
 * Event.replyEvent: replies to an event with a response

Overall, the onAddItemToCart function is responsible for handling the logic of adding an item to a user's shopping cart, including checking for the existence of the product and user, creating a new shopping cart if necessary, and updating the total amount of the cart.
```

### Explains a rule

In the chat window, open the [rule AddItemToCart](https://github.com/yxuco/Petstore/blob/main/Rules/AddItemToCart.rule), and ask the following question:

```
explain the condition and action of the rule AddItemToCart
```

Codeium returns a reasonable explanation:

```
Based on the provided context, the rule AddItemToCart is defined in the file Rules/AddItemToCart.rule. Here is an explanation of the condition and action of the rule:

Condition:

The condition of the rule is defined in the when clause:
(when block list here)

The condition is true when the following two conditions are met:

 1. The SKU of the product (prod.sku) matches the SKU of the event (evt.sku).
 2. The username of the shopping cart (cart.userName) matches the username of the event (evt.user).

Action:

The action of the rule is defined in the then clause:
(then block list here)

The action performs the following steps:

 1. Logs a message indicating that the item has been added to the cart.
 2. Calculates the total amount of the item by multiplying the unit price by the quantity.
 3. Adds the item to the shopping cart by creating a new ShoppingItem instance and adding it to the items array of the cart.
 4. Updates the total amount of the shopping cart by adding the amount of the item.
 5. Replies to the event with a JSON representation of the shopping cart and a 200 status code.
 6. Consumes the event to prevent further processing.

Overall, the AddItemToCart rule is responsible for adding an item to a user's shopping cart when the condition is met, and updating the cart's total amount and logging a message accordingly.
```

### Summary of a BE application

It can provide high-level descriptions of a BE application.

Open a [petstore](https://github.com/yxuco/Petstore) BE sample application.  Add context files to include [CDD](https://github.com/yxuco/Petstore/blob/main/Deployments/Petstore.cdd), [channel definition](https://github.com/yxuco/Petstore/blob/main/api/petstore.channel), and [OpenAPI definition](https://github.com/yxuco/Petstore/blob/main/Deployments/Petstore.yaml), and then in the chat window, ask the following question:

```
explain what this petstore application is doing
```

Codeium returns the following response:

```
The Petstore application is a demo application that simulates a simple e-commerce system for a pet store. It allows users to perform the following actions:

 1. Add items to cart: Users can add pet-related items to their shopping cart.
 2. Submit orders: Users can submit their orders, which includes the items in their shopping cart.
 3. View inventory: Users can view the current inventory of pet-related items.
 4. View alerts: Users can view alerts related to low inventory levels or other important notifications.

The application uses TIBCO BusinessEvents (BE) to manage the business logic and rules for the pet store. BE is a complex event processing (CEP) platform that allows developers to define and manage business rules, events, and processes.

The application consists of several components, including:

 1. Swagger Editor: A user interface for defining and testing RESTful APIs.
 2. TIBCO BusinessEvents (BE) RMS (Rules Management System): A graphical user interface for defining and managing business rules.
 3. Inference Engine: The core component of the application that processes events and applies business rules to generate responses.

The application follows the following workflow:

 1. A user sends an HTTP request to perform an action (e.g., add item to cart, submit order, etc.).
 2. The request is received by the BE HTTP Adapter and converted into an event.
 3. The event is processed by the inference engine, which applies the business rules defined in the RMS.
 4. The inference engine generates a response based on the business rules and sends it back to the user via HTTP.

The application demonstrates the use of TIBCO BusinessEvents to implement a real-time event-driven system for a pet store, with features such as:

 * Real-time inventory management
 * Automated alerts for low inventory levels
 * Business rules for managing orders and inventory
 * Integration with a user interface for testing and demonstration purposes.
```
