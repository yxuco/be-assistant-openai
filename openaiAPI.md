# Write BE Code Using OpenAI API

The [OpenAI APIs](https://platform.openai.com/docs/api-reference/introduction) can be used to fine-tune and configure a supported OpenAI LLM so it can write almost perfectely correct rules and functions for TIBCO BusinessEvents (BE).

We use its Python binding and the `GPT-4o` model to demonstrate how BE rules and functions can be written by AI.  OpenAI also supports APIs through HTTP and Node.js, as well as other [community-supported programming languages](https://platform.openai.com/docs/libraries/community-libraries), including C#, C++, Java, Go, Rust, and many more.

The notebook file [BEPOC.ipnb](./BEPOC.ipynb) contains all code of the demo.  It shows that an LLM can write nearly perfect BE rules and functions as long as you provide it carefully prepared request prompts.  The notebook file also included examples for model [fine-tuning](https://platform.openai.com/docs/api-reference/fine-tuning), although it is not required to fine-tune the `GPT-4o` model for this demo.

## BE Assistant

The demo is implemented by using the [Assistants API](https://platform.openai.com/docs/api-reference/assistants), because OpenAI assistants is already integrated with [text embeddings](https://platform.openai.com/docs/api-reference/embeddings) that can efficiently encode large auxiliary text files into a vector space and support searches by AI assitants.  This feature is required to encode BE documentations of over 1000 BE library functions, and so an AI assistant can find and include correct library function calls in its generated artifacts.

We collect the descriptions of BE library functions in JSON documents, such as [Standard.json](./Standard.json), which contains standard BE function definitions as follows:

```json
    {
        "Function": "Log.getLogger()",
        "Signature": "Object getLogger (String loggerName)",
        "Domain": "ACTION, CONDITION, QUERY, BUI",
        "Description": "Get a logger to be used inside a rule/rulefunction name.",
        "Parameters": [
            {
                "name": "loggerName",
                "type": "String",
                "description": "Name of the Logger to get. Use the project path of the current rule or rulefunction."
            }
        ],
        "Returns": {
            "Type": "Object"
        },
        "Cautions": "none",
        "Example": "Object logger = Log.getLogger(\"Rules.VerifyCreditRF\");"
    }
```

The following API calls create a vector store containing documentation of all BE library functions, which will be referenced by the BE assistant.

```python
client = OpenAI()

# Create a vector store for BE docs
vector_store = client.beta.vector_stores.create(name="BE Assistant Store")

# Ready the files for upload to OpenAI
file_paths = ["Standard.json", "CEP.json", "RDBMS.json"]
file_streams = [open(path, "rb") for path in file_paths]
 
# Use the upload and poll SDK helper to upload the files, add them to the vector store,
# and poll the status of the file batch for completion.
file_batch = client.beta.vector_stores.file_batches.upload_and_poll(
  vector_store_id=vector_store.id, files=file_streams
)
```

We can then create an AI assistant to write BE rules and functions, which can search BE documentations for standard BE functions whenever it is necessary.

```python
assistant = client.beta.assistants.create(
  name="BE Assistant",
  instructions=be_assistant_prompt,
  model="gpt-4o",
  temperature=0.01,
  tools=[{"type": "file_search"}],
  tool_resources={
    "file_search": {
      "vector_store_ids": [vector_store.id]
    }
  }
)
```

The above API call includes a reference of `be_assistant_prompt`, which contains text describing the requirements for the AI assistant.  It is the main challenge of an AI project to carefully specify the system and user requirements, which is called [prompt engineering](https://platform.openai.com/docs/guides/prompt-engineering).  You may follow the [link](https://platform.openai.com/docs/guides/prompt-engineering) to read more about the strategies and tactics of `prompt engineering`.

This demo uses the following prompt to describe the requirements and code examples for a BE assistant.

## BE Assistant Prompt

You are an assistant.  You write TIBCO BE rules and functions according to user requests.

A valid TIBCO BE function must not violate any of the following requirements:

* A function must include a curly braces block of `attribute` that specifies the setting of `validity`.
* A function must include a curly braces block of `scope` that lists all input variables that the function may read and update.
* A function must include a block of `body` that is the code for activities required by user.

Functions may include calls to BE library functions.  You may lookup definitions of BE library functions from BE documentation files in the embeddings storage.

Example of a BE function for pre-processing an event of type SubmitOrder:
```
/**
 * @description 
 * @author AI Assistant
 */
void rulefunction RuleFunctions.onSubmitOrder {
	attribute {
		validity = ACTION;
	}
	scope {
		Events.SubmitOrder evt;
	}
	body {
		Object logger = Log.getLogger("RuleFunctions.onSubmitOrder");
		
		Concepts.ShoppingCart cart = Instance.getByExtIdByUri(RuleFunctions.extIdPrefix("ShoppingCart")+evt.user, "/Concepts/ShoppingCart");
		if (cart == null) {
			Log.log(logger, "info", "Shopping cart of %s does not exist", evt.user);
			// reply 404 Shopping cart not found
			Event.replyEvent(evt, Events.HTTPStatus.HTTPStatus(null, null, "404"));
			Event.consumeEvent(evt);
			return;
		}
		
		if (cart.items@length == 0) {
			Log.log(logger, "info", "Shopping cart of %s is empty", evt.user);
			// reply 400 Shopping cart is empty
			Event.replyEvent(evt, Events.HTTPStatus.HTTPStatus(null, null, "400"));
			Event.consumeEvent(evt);
			return;
		}
		Log.log(logger, "info", "Creating new order for user %s", evt.user);
	}
}
```

To load a concept from cache, you can call BE library functions in the package of `Cluster.DataGrid`, e.g.,
```
    Concepts.ShoppingCart cart = Cluster.DataGrid.CacheLoadConceptById(oid, true);
```

To get a cluster-wide lock, you can also call BE libary function in the package of `Cluster.DataGrid`, e.g.,
```
    boolean lockAcquired = Cluster.DataGrid.Lock(key, 500, false);
```

The following example constructs a JSON text by using BE library functions for strings:
```
/**
 * @description
 * @author AI Assistant
 */
String rulefunction RuleFunctions.userToJSON {
	attribute {
		validity = ACTION;
	}
	scope {
		Concepts.User user;
	}
	body {
		Object buff = String.createBuffer(300);
		String.append(buff, String.format("{\"name\":\"%s\",", user.name));
		String.append(buff, String.format("\"status\":\"%s\",", user.status));
		String.append(buff, String.format("\"rewards\":%d,", user.rewardPoints));
		String.append(buff, String.format("\"orders\":%d,", user.orders@length));
		String.append(buff, "\"coupons\":[");
		for (int i = 0; i < user.coupons@length; i++) {
			String.append(buff, String.format("\"%s\"", RuleFunctions.couponToString(user.coupons[i])));
			if (i != user.coupons@length-1) {
				String.append(buff, ",");
			}
		}
		String.append(buff, "]}");
		return String.convertBufferToString(buff);
	}
}
```

A valid TIBCO BE rule must not violate any of the following requirements:

* A rule must include a curly braces block of `attribute` that specifies settings of `priority` and `forwardChain`.
* A rule must include a curly braces block of `declare` that lists all input variables used by the rule conditions.
* A rule must include a block of `when` that lists rule condisions for matching properties of the declared variables.  Variables must not be declared in this block.
* A rule must include a block of `then` that is the code for required actions of the rule.

Rules may create new instances of a Concept class by using a constructor function that is named by its class name.  The arguments of a concept constructor includes an unique extId and values of all properties specified by the Concept class definition, e.g.,
```
Object item = Concepts.ShoppingItem.ShoppingItem(
	null /*extId String */,
	evt.sku /*sku String */,
	evt.unit /*unit long */,
	amount /*amount double */);
```

Rules may create new instances of an Event class by using a constructor function that is named by its class name.  The arguments of an event constructor includes an unique extId, a event payload string, and values of all properties specified by the Event class definition, e.g.,
```
Object evt = Events.HTTPStatus.HTTPStatus(
    null /*extId String*/,
    RuleFunctions.cartToJSON(cart) /*payload String */,
    "200" /*status String */);
```

Rules may include calls to BE library functions.

Example of a rule that acts on an event of type AddItemToCart, and adds a matching product to a matching shopping cart:
```
/**
 * @description 
 * @author AI Assistant
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

Keep your response brief with only the generated code for a BE rule or function.

## Execute Request to Write BE Code

Following API will execute a specified `request_prompt` and print out BE artifacts generated by the AI assistant.

```python
 # create thread and message
thread = client.beta.threads.create()
message = client.beta.threads.messages.create(
  thread_id=thread.id,
  role="user",
  content=request_prompt
)

# create run stream, and printout response
with client.beta.threads.runs.stream(
  thread_id=thread.id,
  assistant_id=assistant.id,
  event_handler=EventHandler(),
) as stream:
  stream.until_done()
```

## BE Assistant in Action

Following are a few examples of request prompt and resulting artifacts written by the AI assistant.

### Simple BE Function

Request prompt:
```
write a function to load Order concept by specified Id from cache.  create new if it does not exist.
```

Result:
```
/**
 * @description Load Order concept by specified Id from cache. Create new if it does not exist.
 * @param orderId The Id of the Order to be loaded or created.
 * @return The loaded or newly created Order concept.
 */
Concepts.Order rulefunction RuleFunctions.loadOrCreateOrder {
    attribute {
        validity = ACTION;
    }
    scope {
        String orderId;
    }
    body {
        Object logger = Log.getLogger("RuleFunctions.loadOrCreateOrder");
        
        // Attempt to load the Order concept from cache
        Concepts.Order order = Cluster.DataGrid.CacheLoadConceptById(orderId, true);
        
        // If the Order does not exist, create a new one
        if (order == null) {
            Log.log(logger, "info", "Order with Id %s does not exist. Creating new Order.", orderId);
            order = Concepts.Order.Order(orderId /*extId*/, /* other properties with default values */);
            // Optionally, you can assert the new Order to the working memory or cache
            // Instance.assert(order);
        } else {
            Log.log(logger, "info", "Order with Id %s loaded from cache.", orderId);
        }
        
        return order;
    }
}
```

### Preprocessor Function

Request prompt:
```
Write a function to preprocess an OrderStatus event.

Acquire a cluster-wide lock on the orderId specified by the event, then load an Order concept from cache by using the specified orderId.  

Log the status of the operations. Do not unlock the outstanding lock.
```

Result:
```
/**
 * @description Preprocess OrderStatus event by acquiring a cluster-wide lock and loading the Order concept from cache.
 * @param evt The OrderStatus event to preprocess.
 * @returns void
 */
void rulefunction RuleFunctions.preprocessOrderStatus {
    attribute {
        validity = ACTION;
    }
    scope {
        Events.OrderStatus evt;
    }
    body {
        Object logger = Log.getLogger("RuleFunctions.preprocessOrderStatus");
        String orderId = evt.orderId;
        
        // Acquire cluster-wide lock on the orderId
        boolean lockAcquired = Cluster.DataGrid.Lock(orderId, 500, false);
        if (!lockAcquired) {
            Log.log(logger, "error", "Failed to acquire lock for orderId: %s", orderId);
            return;
        }
        Log.log(logger, "info", "Lock acquired for orderId: %s", orderId);
        
        // Load Order concept from cache
        Concepts.Order order = Cluster.DataGrid.CacheLoadConceptById(orderId, true);
        if (order == null) {
            Log.log(logger, "error", "Failed to load Order concept for orderId: %s", orderId);
            return;
        }
        Log.log(logger, "info", "Order concept loaded for orderId: %s", orderId);
    }
}
```

### Function to Query RDBMS

Request prompt:
```
Write a function `Functions.executeQuery` that demonstrates RDBMS functions. The function inputs a jdbcResourceURI, a SQL string, and a conceptURI for returing concepts.  It performs the following tasks:

1. Set the current RDBMS connection to the specified jdbcResourceURI;
2. Log the SQL statement to be executed;
3. Execute the query;
4. Log the number of concepts returned by the query;
5. Release the RDBMS connection;
6. Return the list of concepts from the RDBMS query.
```

Result:
```
/**
 * @description Executes an RDBMS query and returns the list of concepts.
 * @param jdbcResourceURI The JDBC resource URI for the database connection.
 * @param sql The SQL query string to be executed.
 * @param conceptURI The URI of the concept type to be returned.
 * @return List of concepts resulting from the query.
 */
Concept[] rulefunction Functions.executeQuery(String jdbcResourceURI, String sql, String conceptURI) {
    attribute {
        validity = ACTION;
    }
    scope {
        String jdbcResourceURI;
        String sql;
        String conceptURI;
    }
    body {
        // Set the current RDBMS connection
        Database.setCurrentConnection(jdbcResourceURI);
        
        // Log the SQL statement to be executed
        Object logger = Log.getLogger("Functions.executeQuery");
        Log.log(logger, "info", "Executing SQL: %s", sql);
        
        // Execute the query
        Concept[] results = Database.queryUsingSQL(conceptURI, sql, true);
        
        // Log the number of concepts returned by the query
        Log.log(logger, "info", "Number of concepts returned: %d", results@length);
        
        // Release the RDBMS connection
        Database.unsetConnection();
        
        // Return the list of concepts
        return results;
    }
}
```

### Rule to Handle HTTP Request

Request prompt:
```
Write a rule RestockInventory to handle an event of type Events.RestockInventory.

The rule executes when the sku property of the event matches the sku of a concept of type Concepts.Inventory. It will perform the following actions:

1. Set the inStock count of the concept to the amount specified by the event;
2. Write a log message containing the sku and restock amount;
3. Create a response event using the event constructor of HTTPStatus with a payload denoting successful completion and an HTTP status of "200";
4. Send the reply event;
5. Consume the event.
```

Result:
```
/**
 * @description Rule to handle RestockInventory event and update the inventory count.
 */
rule Rules.RestockInventory {
	attribute {
		priority = 5;
		forwardChain = true;
	}
	declare {
		Events.RestockInventory evt;
		Concepts.Inventory inv;
	}
	when {
		inv.sku == evt.sku;
	}
	then {
		Object logger = Log.getLogger("Rules.RestockInventory");
		
		// Set the inStock count of the inventory to the amount specified by the event
		inv.inStock = evt.amount;
		
		// Write a log message containing the sku and restock amount
		Log.log(logger, "info", "Restocked SKU %s with amount %d", evt.sku, evt.amount);
		
		// Create a response event using the event constructor of HTTPStatus
		Object responseEvent = Events.HTTPStatus.HTTPStatus(
			null /*extId String*/,
			String.format("Restocked SKU %s with amount %d", evt.sku, evt.amount) /*payload String*/,
			"200" /*status String*/
		);
		
		// Send the reply event
		Event.replyEvent(evt, responseEvent);
		
		// Consume the event
		Event.consumeEvent(evt);
	}
}
```

### Rule That Calls Library Functions

Request prompt:
```
Write a rule Rules.TestFile that executes on any event of type WriteConcept. The rule will perform the following actions:

1. Acquire a cluster-wide lock on the conceptId specified by the event, with a timeout of 500 ms;
2. If failed to acquire the lock, log the error of failed lock; otherwise,
3. Load a concept of type Concepts.Inventory from BE cache by using the specified conceptId;
4. Lookup a file path specified by a system global variable of name "LogFile";
5. Write a log message containing the conceptId and the file path;
6. Open the file for read and write by using the path;
7. Serialize the loaded concept into JSON string, and then write the result to the file;
8. Close the file;
9. consume the event.
```

Result:
```
/**
 * @description Rule to handle WriteConcept events and perform specified actions
 * @autor AI Assistant
 */
rule Rules.TestFile {
	attribute {
		priority = 5;
		forwardChain = true;
	}
	declare {
		Events.WriteConcept evt;
	}
	when {
		// No specific conditions, rule triggers on any WriteConcept event
	}
	then {
		Object logger = Log.getLogger("Rules.TestFile");
		String conceptId = evt.conceptId;
		
		// Acquire cluster-wide lock
		boolean lockAcquired = Cluster.DataGrid.Lock(conceptId, 500, false);
		if (!lockAcquired) {
			Log.log(logger, "error", "Failed to acquire lock for conceptId: %s", conceptId);
			return;
		}
		
		// Load concept from cache
		Concepts.Inventory inventory = Cluster.DataGrid.CacheLoadConceptById(conceptId, true);
		if (inventory == null) {
			Log.log(logger, "error", "Failed to load concept for conceptId: %s", conceptId);
			Cluster.DataGrid.UnLock(conceptId, false);
			return;
		}
		
		// Lookup file path from global variable
		String logFilePath = System.getGlobalVariableAsString("LogFile", null);
		if (logFilePath == null) {
			Log.log(logger, "error", "Global variable 'LogFile' is not set");
			Cluster.DataGrid.UnLock(conceptId, false);
			return;
		}
		
		// Log message with conceptId and file path
		Log.log(logger, "info", "Writing conceptId: %s to file: %s", conceptId, logFilePath);
		
		// Open file for read and write
		System.IO.fileWrite(logFilePath, "", false, true); // Clear the file before writing
		
		// Serialize concept to JSON and write to file
		String json = Instance.serializeUsingDefaults(inventory);
		System.IO.fileWrite(logFilePath, json, true, true);
		
		// Close the file
		System.IO.fileClose(logFilePath);
		
		// Unlock the concept
		Cluster.DataGrid.UnLock(conceptId, false);
		
		// Consume the event
		Event.consumeEvent(evt);
	}
}
```