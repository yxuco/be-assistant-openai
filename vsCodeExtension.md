# AI Pair Programming

When you develop software using [VS Code](https://code.visualstudio.com/) or a [JetBrains](https://www.jetbrains.com/) IDE.  AI models can present as a pair programmer to help you write code faster and smarter.  The [GitHub Copilot](https://code.visualstudio.com/docs/copilot/overview) extension is the most popular of such tools.

A more flexible and configurable tool is, however, the [Continue](https://docs.continue.dev/intro) extension, which is an open-source AI code assistant, and it can be configured to use different types of LLMs, including OpenAI, Gemini, Llama, Mistral, and more.  The LLMs can run in cloud by service providers, or locally on your workstation, although when it runs locally on a MacBook Pro, it would better be equipped with newer M series chips.

## Setup Continue Extension

In VS Code, Open the `Extensions` explorer, search and find the extension `Continue - Codestral, Claude, and more` by [continue.dev](https://www.continue.dev/).  Install the extension, and then open your code to start development.

The code assistant can be accessed by 2 different ways.  Enter `command-L` would open a chat window, in which you can chat with an AI model to generate artifacts, and then cut-and-paste into code editor; or from anywhere in the code editor, enter `command-I` would open an in-line request window, where you can type a request prompt for the AI model to add or update code in-line that you can then accept or reject.

In this demo, we open a sample project of TIBCO BusinessEvents, [Petstore](https://github.com/yxuco/Petstore), in VS Code.

To configure the LLM for AI chat, click the `Continue` label at the bottom-right of the VS code status bar, and then select the menu option `Configure autocomplete options`. It opens the file [~/.continue/config.json](./continue/config.json) that defines the AI models used by the continue extension.

For example, we can add the following 2 models for AI chat.

```json
  "models": [
    {
      "title": "GPT-4o (OpenAI)",
      "provider": "openai",
      "model": "gpt-4o",
      "apiKey": "sk-proj-...your-api-key...",
      "useLegacyCompletionsEndpoint": false,
      "systemMessage": "You are an assistant.  You write TIBCO BE rules and functions according to user requests. ...more "
    },
    {
      "title": "Llama 3.1",
      "provider": "ollama",
      "model": "llama3.1",
      "systemMessage": "You are an assistant.  You write TIBCO BE rules and functions according to user requests. ...more "
    }
  ]
```

The first model is `GPT-4o` running in Azure by an OpenAI service provider.  You need an API-key to login to this API service.  The second model is `llama3.1` running locally on your workstation, which we shall setup in a following section.  These models are also configured with a `systemMessage` that is the same as the [BE Assistant Prompt](./openaiAPI.md#be-assistant-prompt) specified to generate artifacts for TIBCO BusinessEvents.

We can also configure the embedding provider used to encode large context files.  For example, we use an OpenAI embedding model as follows.

```json
  "embeddingsProvider": {
    "provider": "openai",
    "model": "text-embedding-3-small",
    "apiKey": "sk-proj-...your-api-key..."
  },
```

## Chat Using OpenAI GPT-4o

In VS Code, open the [Petstore](https://github.com/yxuco/Petstore) project, then use `command-L` to start a chat session.

In the chat window, select AI model `GPT-4o (OpenAI)`, and add context file `libs/Standard.json`.  Following chat conversation shows BE artifacts generated by AI.

### BE Preprocessor Function

Request prompt:
```
@Standard.json Write a function to preprocess an OrderStatus event.

Acquire a cluster-wide lock on the orderId specified by the event, then load an Order concept from cache by using the specified orderId.  

Log the status of the operations. Do not unlock the outstanding lock.
```

Result:
```
/**
 * @description Preprocess an OrderStatus event: acquire a cluster-wide lock on the orderId and load Order concept from cache. Log the status of operations. Do not unlock the lock. 
 * @autho AI Assistant
 */
void rulefunction RuleFunctions.onOrderStatus {
    attribute {
        validity = ACTION;
    }
    scope {
        Events.OrderStatus evt;
    }
    body {
        Object logger = Log.getLogger("RuleFunctions.onOrderStatus");

        String lockKey = "OrderLock_" + evt.orderId;
        boolean lockAcquired = Cluster.DataGrid.Lock(lockKey, 500, false);

        if (!lockAcquired) {
            Log.log(logger, "warn", "Failed to acquire lock on Order ID: %s", evt.orderId);
            return;
        }

        Concepts.Order order = Cluster.DataGrid.CacheLoadConceptById(evt.orderId, true);
        
        if (order == null) {
            Log.log(logger, "error", "Order with ID: %s not found in cache", evt.orderId);
            return;
        }
        
        Log.log(logger, "info", "Successfully acquired lock and loaded Order with ID: %s", evt.orderId);
    }
}
```

### BE Rule Sending HTTP Response

Request prompt:
```
@Standard.json Write a rule RestockInventory to handle an event of type Events.RestockInventory.

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
 * @description Rule to handle restocking of inventory based on Events.RestockInventory event type. 
 * It updates the inStock count of the corresponding Concepts.Inventory and sends back an HTTPStatus event as a response indicating successful restock operation.
 * The rule logs details of the restock.
 * The event is consumed after processing.
 * 
 * Note: It is assumed that the sku property of the event and the Concepts.Inventory concept are unique identifiers and should match exactly.
 * 
 * Author: AI Assistant
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
		evt.sku == inv.sku;
	}
	then {
		Object logger = Log.getLogger("Rules.RestockInventory");

		// Update inStock count with restock amount from event
		inv.inStock = evt.amount;

		// Log restocking info
		Log.log(logger, "info", "Restocked SKU %s with amount %d", evt.sku, evt.amount);

		// Create HTTPStatus response event
		Object response = Events.HTTPStatus.HTTPStatus(
			null /*extId*/,
			String.format("Restock completed for SKU %s with amount %d", evt.sku, evt.amount) /*payload*/,
			"200" /*status*/);

		// Send reply event
		Event.replyEvent(evt, response);

		// Consume the original event
		Event.consumeEvent(evt);
	}
}
```

### BE Function Executing RDBMS Query

Request prompt:
```
@RDBMS.json Write a function `Functions.executeQuery` that demonstrates RDBMS functions. The function inputs a jdbcResourceURI, a SQL string, and a conceptURI for returing concepts.  It performs the following tasks:

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
 * @description Demonstrates RDBMS functions by executing a query and returning the result concepts
 */
Concept[] rulefunction Functions.executeQuery {
	attribute {
		validity = ACTION;
	}
	scope {
		String jdbcResourceURI;
		String sql;
		String conceptURI;
	}
	body {
		// Step 1: Set the current RDBMS connection to the specified jdbcResourceURI
		Database.setCurrentConnection(jdbcResourceURI);
		
		// Step 2: Log the SQL statement to be executed
		Object logger = Log.getLogger("Functions.executeQuery");
		Log.log(logger, "info", "Executing SQL: %s", sql);
		
		// Step 3: Execute the query
		Concept[] resultConcepts = Database.queryUsingSQL(conceptURI, sql, true);
		
		// Step 4: Log the number of concepts returned by the query
		Log.log(logger, "info", "Number of concepts returned: %d", resultConcepts@length);
		
		// Step 5: Release the RDBMS connection
		Database.unsetConnection();
		
		// Step 6: Return the list of concepts from the RDBMS query
		return resultConcepts;
	}
}
```

## Run Local Llama 3.1 Model

If you do not subscribe any AI service providers such as OpenAI, or you want to test AI models on a local workstation, the [LM Studio](https://lmstudio.ai/) is a popular UI tool that allows you to run various LLMs locally.  However, on a Mac, LM Studio requires a newer laptop equipped with M series chips.  This demo uses the [Ollama](https://ollama.com/) to run local LLMs, which supports older laptops.

Follow the link to download [Ollama](https://ollama.com/) for a Mac, unzip and drop the app in `Applications` folder, then launch `Ollama` and following the instructions to install CLI tool `ollama`.

You can then start a supported LLM on your workstation, e.g., `llama3.1`:

```
ollama run llama3.1
```

## Chat Using Local llama3.1

Since the `Continue` extension has already been configured to use `llama3.1`, we can use this model in a VS Code chat session after the model has started locally.

### BE Preprocessor Function

Request prompt:
```
Write a function to preprocess an OrderStatus event.

Acquire a cluster-wide lock on the orderId specified by the event, then load an Order concept from cache by using the specified orderId.  

Log the status of the operations. Do not unlock the outstanding lock.
```

Result:
```
/**
 * @description 
 * @author AI Assistant
 */
void rulefunction RuleFunctions.onOrderStatus {
	attribute {
		validity = ACTION;
	}
	scope {
		Events.OrderStatus evt;
	}
	body {
		Object logger = Log.getLogger("RuleFunctions.onOrderStatus");
		
		boolean lockAcquired = Cluster.DataGrid.Lock(evt.orderId, 500, false);
		if (!lockAcquired) {
			Log.log(logger, "error", "Failed to acquire lock on orderId %s", evt.orderId);
			return;
		}
		
		Concepts.Order order = Cluster.DataGrid.CacheLoadConceptById(oid, true);
		if (order == null) {
			Log.log(logger, "error", "Failed to load Order concept with orderId %s from cache", evt.orderId);
		} else {
			Log.log(logger, "info", "Loaded Order concept with orderId %s from cache", evt.orderId);
		}
	}
}
```

### BE Rule Sending HTTP response

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
 * @description 
 * @author AI Assistant
 */
rule Rules.RestockInventory {
	attribute {
		priority = 10;
		forwardChain = true;
	}
	declare {
		Events.RestockInventory evt;
		Concepts.Inventory inventory;
	}
	when {
		inventory.sku == evt.sku;
	}
	then {
		Object logger = Log.getLogger("Rules.RestockInventory");
		inventory.inStock = evt.amount;
		Log.log(logger, "info", "Restocked %d units of SKU %s", evt.amount, evt.sku);
		Event.replyEvent(evt, Events.HTTPStatus.HTTPStatus(null,RuleFunctions.inventoryToJSON(inventory),"200"));
		Event.consumeEvent(evt);
	}
}
```

## In-line Auto-completion

The most useful code assistant feature is auto-completion.  By default, the `Continue` extension is configured to use the following AI model for auto-completion.

```json
  "tabAutocompleteModel": {
    "title": "Tab Autocomplete",
    "provider": "free-trial",
    "model": "codestral-latest"
  },
```

It prompts code suggestions as you type in VS Code editor, which you can `TAB` to accept.  You may also type `command-I` to enter special request for in-line code generation or updates.

However, the auto-completion feature does not appear work as expected even after the configuration is updated to use `GPT-4o`, i.e.,

```json
  "tabAutocompleteModel": {
    "title": "Tab Autocomplete",
      "provider": "openai",
      "model": "gpt-4o",
      "apiKey": "sk-proj-...your-api-key-here...",
      "useLegacyCompletionsEndpoint": false,
      "systemMessage": "You are an assistant.  You write TIBCO BE rules and functions according to user requests. ...more"
    },
```

The in-line code generation or auto-completion would always be done using standard programming languages such as Python or Javascript.  It does not take special instructions required by TIBCO BusinssEvents.