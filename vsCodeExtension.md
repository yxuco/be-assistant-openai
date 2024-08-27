# AI Pair Programming

When you develop software using [VS Code](https://code.visualstudio.com/) or a [JetBrains](https://www.jetbrains.com/) IDE.  AI models can present as a pair programer to help you write code faster and smarter.  The [GitHub Copilot](https://code.visualstudio.com/docs/copilot/overview) extension is the most popular of such tools.

A more flexible and configurable tool is, however, the [Continue](https://docs.continue.dev/intro) extension that is an open-source AI code assistant, and it can be configured to use different types of LLMs, including OpenAI, Gemini, Llama, Mistral, and more.  The LLMs can run in cloud by service providers, or locally on your workstation, although when it runs locally on a MacBook Pro, it would better be equipped with newer M series chips.

## Setup Continue Extension

In VS Code, Open the `Extensions` explorer, search and find the extension `Continue - Codestral, Claude, and more` by [continue.dev](https://www.continue.dev/).  Install the extension, and then open your code to start development.

The code assistant can be accessed by 2 different ways.  Enter `command-L` would open a chat window, in which you can chat with an AI model to generate artifacts, which you can cut-and-paste into code editor; or from anywhere in the code editor, enter `command-I` would open an in-line request window, where you can type a request prompt for the AI model to add or update your code in-line that you can then accept or reject.

For this demo, we open a sample project of TIBCO BusinessEvents, [Petstore](https://github.com/yxuco/Petstore), in VS Code.

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

The first model is `GPT-4o` running in Azure by an Open AI service provider.  You need an API-key to login to this API service.  The second model is `llama3.1` running locally on your workstation, which we shall setup in a following section.  These models are also configured with a `systemMessage` that is the same as the [BE Assistant Prompt](./openaiAPI.md#be-assistant-prompt) specified to generate artifacts for TIBCO BusinessEvents.

We can also configure the embedding provider used to encode large context files.  For example, we use an Open AI embedding model as follows.

```json
  "embeddingsProvider": {
    "provider": "openai",
    "model": "text-embedding-3-small",
    "apiKey": "sk-proj-...your-api-key..."
  },
```