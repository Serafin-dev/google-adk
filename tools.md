# Documentación: Tools


## Tools - Agent Development Kit

URL: https://google.github.io/adk-docs/tools/

# Tools[¶](#tools "Permanent link")

## What is a Tool?[¶](#what-is-a-tool "Permanent link")

In the context of ADK, a Tool represents a specific
capability provided to an AI agent, enabling it to perform actions and interact
with the world beyond its core text generation and reasoning abilities. What
distinguishes capable agents from basic language models is often their effective
use of tools.

Technically, a tool is typically a modular code component—**like a Python
function**, a class method, or even another specialized agent—designed to
execute a distinct, predefined task. These tasks often involve interacting with
external systems or data.

![Agent tool call](../assets/agent-tool-call.png)

### Key Characteristics[¶](#key-characteristics "Permanent link")

**Action-Oriented:** Tools perform specific actions, such as:

* Querying databases
* Making API requests (e.g., fetching weather data, booking systems)
* Searching the web
* Executing code snippets
* Retrieving information from documents (RAG)
* Interacting with other software or services

**Extends Agent capabilities:** They empower agents to access real-time information, affect external systems, and overcome the knowledge limitations inherent in their training data.

**Execute predefined logic:** Crucially, tools execute specific, developer-defined logic. They do not possess their own independent reasoning capabilities like the agent's core Large Language Model (LLM). The LLM reasons about which tool to use, when, and with what inputs, but the tool itself just executes its designated function.

## How Agents Use Tools[¶](#how-agents-use-tools "Permanent link")

Agents leverage tools dynamically through mechanisms often involving function calling. The process generally follows these steps:

1. **Reasoning:** The agent's LLM analyzes its system instruction, conversation history, and user request.
2. **Selection:** Based on the analysis, the LLM decides on which tool, if any, to execute, based on the tools available to the agent and the docstrings that describes each tool.
3. **Invocation:** The LLM generates the required arguments (inputs) for the selected tool and triggers its execution.
4. **Observation:** The agent receives the output (result) returned by the tool.
5. **Finalization:** The agent incorporates the tool's output into its ongoing reasoning process to formulate the next response, decide the subsequent step, or determine if the goal has been achieved.

Think of the tools as a specialized toolkit that the agent's intelligent core (the LLM) can access and utilize as needed to accomplish complex tasks.

## Tool Types in ADK[¶](#tool-types-in-adk "Permanent link")

ADK offers flexibility by supporting several types of tools:

1. **[Function Tools](function-tools/):** Tools created by you, tailored to your specific application's needs.
   * **[Functions/Methods](function-tools/#1-function-tool):** Define standard synchronous functions or methods in your code (e.g., Python def).
   * **[Agents-as-Tools](function-tools/#3-agent-as-a-tool):** Use another, potentially specialized, agent as a tool for a parent agent.
   * **[Long Running Function Tools](function-tools/#2-long-running-function-tool):** Support for tools that perform asynchronous operations or take significant time to complete.
2. **[Built-in Tools](built-in-tools/):** Ready-to-use tools provided by the framework for common tasks.
   Examples: Google Search, Code Execution, Retrieval-Augmented Generation (RAG).
3. **[Third-Party Tools](third-party-tools/):** Integrate tools seamlessly from popular external libraries.
   Examples: LangChain Tools, CrewAI Tools.

Navigate to the respective documentation pages linked above for detailed information and examples for each tool type.

## Referencing Tool in Agent’s Instructions[¶](#referencing-tool-in-agents-instructions "Permanent link")

Within an agent's instructions, you can directly reference a tool by using its **function name.** If the tool's **function name** and **docstring** are sufficiently descriptive, your instructions can primarily focus on **when the Large Language Model (LLM) should utilize the tool**. This promotes clarity and helps the model understand the intended use of each tool.

It is **crucial to clearly instruct the agent on how to handle different return values** that a tool might produce. For example, if a tool returns an error message, your instructions should specify whether the agent should retry the operation, give up on the task, or request additional information from the user.

Furthermore, ADK supports the sequential use of tools, where the output of one tool can serve as the input for another. When implementing such workflows, it's important to **describe the intended sequence of tool usage** within the agent's instructions to guide the model through the necessary steps.

### Example[¶](#example "Permanent link")

The following example showcases how an agent can use tools by **referencing their function names in its instructions**. It also demonstrates how to guide the agent to **handle different return values from tools**, such as success or error messages, and how to orchestrate the **sequential use of multiple tools** to accomplish a task.

```
from google.adk.agents import Agent
from google.adk.tools import FunctionTool
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService
from google.genai import types

APP_NAME="weather_sentiment_agent"
USER_ID="user1234"
SESSION_ID="1234"
MODEL_ID="gemini-2.0-flash"

# Tool 1
def get_weather_report(city: str) -> dict:
    """Retrieves the current weather report for a specified city.

    Returns:
        dict: A dictionary containing the weather information with a 'status' key ('success' or 'error') and a 'report' key with the weather details if successful, or an 'error_message' if an error occurred.
    """
    if city.lower() == "london":
        return {"status": "success", "report": "The current weather in London is cloudy with a temperature of 18 degrees Celsius and a chance of rain."}
    elif city.lower() == "paris":
        return {"status": "success", "report": "The weather in Paris is sunny with a temperature of 25 degrees Celsius."}
    else:
        return {"status": "error", "error_message": f"Weather information for '{city}' is not available."}

weather_tool = FunctionTool(func=get_weather_report)


# Tool 2
def analyze_sentiment(text: str) -> dict:
    """Analyzes the sentiment of the given text.

    Returns:
        dict: A dictionary with 'sentiment' ('positive', 'negative', or 'neutral') and a 'confidence' score.
    """
    if "good" in text.lower() or "sunny" in text.lower():
        return {"sentiment": "positive", "confidence": 0.8}
    elif "rain" in text.lower() or "bad" in text.lower():
        return {"sentiment": "negative", "confidence": 0.7}
    else:
        return {"sentiment": "neutral", "confidence": 0.6}

sentiment_tool = FunctionTool(func=analyze_sentiment)


# Agent
weather_sentiment_agent = Agent(
    model=MODEL_ID,
    name='weather_sentiment_agent',
    instruction="""You are a helpful assistant that provides weather information and analyzes the sentiment of user feedback.
**If the user asks about the weather in a specific city, use the 'get_weather_report' tool to retrieve the weather details.**
**If the 'get_weather_report' tool returns a 'success' status, provide the weather report to the user.**
**If the 'get_weather_report' tool returns an 'error' status, inform the user that the weather information for the specified city is not available and ask if they have another city in mind.**
**After providing a weather report, if the user gives feedback on the weather (e.g., 'That's good' or 'I don't like rain'), use the 'analyze_sentiment' tool to understand their sentiment.** Then, briefly acknowledge their sentiment.
You can handle these tasks sequentially if needed.""",
    tools=[weather_tool, sentiment_tool]
)

# Session and Runner
session_service = InMemorySessionService()
session = session_service.create_session(app_name=APP_NAME, user_id=USER_ID, session_id=SESSION_ID)
runner = Runner(agent=weather_sentiment_agent, app_name=APP_NAME, session_service=session_service)


# Agent Interaction
def call_agent(query):
    content = types.Content(role='user', parts=[types.Part(text=query)])
    events = runner.run(user_id=USER_ID, session_id=SESSION_ID, new_message=content)

    for event in events:
        if event.is_final_response():
            final_response = event.content.parts[0].text
            print("Agent Response: ", final_response)

call_agent("weather in london?")

```

## Tool Context[¶](#tool-context "Permanent link")

For more advanced scenarios, ADK allows you to access additional contextual information within your tool function by including the special parameter `tool_context: ToolContext`. By including this in the function signature, ADK will **automatically** provide an **instance of the ToolContext** class when your tool is called during agent execution.

The **ToolContext** provides access to several key pieces of information and control levers:

* `state: State`: Read and modify the current session's state. Changes made here are tracked and persisted.
* `actions: EventActions`: Influence the agent's subsequent actions after the tool runs (e.g., skip summarization, transfer to another agent).
* `function_call_id: str`: The unique identifier assigned by the framework to this specific invocation of the tool. Useful for tracking and correlating with authentication responses. This can also be helpful when multiple tools are called within a single model response.
* `function_call_event_id: str`: This attribute provides the unique identifier of the **event** that triggered the current tool call. This can be useful for tracking and logging purposes.
* `auth_response: Any`: Contains the authentication response/credentials if an authentication flow was completed before this tool call.
* Access to Services: Methods to interact with configured services like Artifacts and Memory.

### **State Management**[¶](#state-management "Permanent link")

The `tool_context.state` attribute provides direct read and write access to the state associated with the current session. It behaves like a dictionary but ensures that any modifications are tracked as deltas and persisted by the session service. This enables tools to maintain and share information across different interactions and agent steps.

* **Reading State**: Use standard dictionary access (`tool_context.state['my_key']`) or the `.get()` method (`tool_context.state.get('my_key', default_value)`).
* **Writing State**: Assign values directly (`tool_context.state['new_key'] = 'new_value'`). These changes are recorded in the state\_delta of the resulting event.
* **State Prefixes**: Remember the standard state prefixes:

  + `app:*`: Shared across all users of the application.
  + `user:*`: Specific to the current user across all their sessions.
  + (No prefix): Specific to the current session.
  + `temp:*`: Temporary, not persisted across invocations (useful for passing data within a single run call but generally less useful inside a tool context which operates between LLM calls).

```
from google.adk.tools import ToolContext, FunctionTool

def update_user_preference(preference: str, value: str, tool_context: ToolContext):
    """Updates a user-specific preference."""
    user_prefs_key = "user:preferences"
    # Get current preferences or initialize if none exist
    preferences = tool_context.state.get(user_prefs_key, {})
    preferences[preference] = value
    # Write the updated dictionary back to the state
    tool_context.state[user_prefs_key] = preferences
    print(f"Tool: Updated user preference '{preference}' to '{value}'")
    return {"status": "success", "updated_preference": preference}

pref_tool = FunctionTool(func=update_user_preference)

# In an Agent:
# my_agent = Agent(..., tools=[pref_tool])

# When the LLM calls update_user_preference(preference='theme', value='dark', ...):
# The tool_context.state will be updated, and the change will be part of the
# resulting tool response event's actions.state_delta.

```

### **Controlling Agent Flow**[¶](#controlling-agent-flow "Permanent link")

The `tool_context.actions` attribute holds an **EventActions** object. Modifying attributes on this object allows your tool to influence what the agent or framework does after the tool finishes execution.

* **`skip_summarization: bool`**: (Default: False) If set to True, instructs the ADK to bypass the LLM call that typically summarizes the tool's output. This is useful if your tool's return value is already a user-ready message.
* **`transfer_to_agent: str`**: Set this to the name of another agent. The framework will halt the current agent's execution and **transfer control of the conversation to the specified agent**. This allows tools to dynamically hand off tasks to more specialized agents.
* **`escalate: bool`**: (Default: False) Setting this to True signals that the current agent cannot handle the request and should pass control up to its parent agent (if in a hierarchy). In a LoopAgent, setting **escalate=True** in a sub-agent's tool will terminate the loop.

#### Example[¶](#example_1 "Permanent link")

```
from google.adk.agents import Agent
from google.adk.tools import FunctionTool
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService
from google.adk.tools import ToolContext
from google.genai import types

APP_NAME="customer_support_agent"
USER_ID="user1234"
SESSION_ID="1234"


def check_and_transfer(query: str, tool_context: ToolContext) -> str:
    """Checks if the query requires escalation and transfers to another agent if needed."""
    if "urgent" in query.lower():
        print("Tool: Detected urgency, transferring to the support agent.")
        tool_context.actions.transfer_to_agent = "support_agent"
        return "Transferring to the support agent..."
    else:
        return f"Processed query: '{query}'. No further action needed."

escalation_tool = FunctionTool(func=check_and_transfer)

main_agent = Agent(
    model='gemini-2.0-flash',
    name='main_agent',
    instruction="""You are the first point of contact for customer support of an analytics tool. Answer general queries. If the user indicates urgency, use the 'check_and_transfer' tool.""",
    tools=[check_and_transfer]
)

support_agent = Agent(
    model='gemini-2.0-flash',
    name='support_agent',
    instruction="""You are the dedicated support agent. Mentioned you are a support handler and please help the user with their urgent issue."""
)

main_agent.sub_agents = [support_agent]

# Session and Runner
session_service = InMemorySessionService()
session = session_service.create_session(app_name=APP_NAME, user_id=USER_ID, session_id=SESSION_ID)
runner = Runner(agent=main_agent, app_name=APP_NAME, session_service=session_service)


# Agent Interaction
def call_agent(query):
    content = types.Content(role='user', parts=[types.Part(text=query)])
    events = runner.run(user_id=USER_ID, session_id=SESSION_ID, new_message=content)

    for event in events:
        if event.is_final_response():
            final_response = event.content.parts[0].text
            print("Agent Response: ", final_response)

call_agent("this is urgent, i cant login")

```

##### Explanation[¶](#explanation "Permanent link")

* We define two agents: `main_agent` and `support_agent`. The `main_agent` is designed to be the initial point of contact.
* The `check_and_transfer` tool, when called by `main_agent`, examines the user's query.
* If the query contains the word "urgent", the tool accesses the `tool_context`, specifically **`tool_context.actions`**, and sets the transfer\_to\_agent attribute to `support_agent`.
* This action signals to the framework to **transfer the control of the conversation to the agent named `support_agent`**.
* When the `main_agent` processes the urgent query, the `check_and_transfer` tool triggers the transfer. The subsequent response would ideally come from the `support_agent`.
* For a normal query without urgency, the tool simply processes it without triggering a transfer.

This example illustrates how a tool, through EventActions in its ToolContext, can dynamically influence the flow of the conversation by transferring control to another specialized agent.

### **Authentication**[¶](#authentication "Permanent link")

ToolContext provides mechanisms for tools interacting with authenticated APIs. If your tool needs to handle authentication, you might use the following:

* **`auth_response`**: Contains credentials (e.g., a token) if authentication was already handled by the framework before your tool was called (common with RestApiTool and OpenAPI security schemes).
* **`request_credential(auth_config: dict)`**: Call this method if your tool determines authentication is needed but credentials aren't available. This signals the framework to start an authentication flow based on the provided auth\_config.
* **`get_auth_response()`**: Call this in a subsequent invocation (after request\_credential was successfully handled) to retrieve the credentials the user provided.

For detailed explanations of authentication flows, configuration, and examples, please refer to the dedicated Tool Authentication documentation page.

### **Context-Aware Data Access Methods**[¶](#context-aware-data-access-methods "Permanent link")

These methods provide convenient ways for your tool to interact with persistent data associated with the session or user, managed by configured services.

* **`list_artifacts()`**: Returns a list of filenames (or keys) for all artifacts currently stored for the session via the artifact\_service. Artifacts are typically files (images, documents, etc.) uploaded by the user or generated by tools/agents.
* **`load_artifact(filename: str)`**: Retrieves a specific artifact by its filename from the **artifact\_service**. You can optionally specify a version; if omitted, the latest version is returned. Returns a `google.genai.types.Part` object containing the artifact data and mime type, or None if not found.
* **`save_artifact(filename: str, artifact: types.Part)`**: Saves a new version of an artifact to the artifact\_service. Returns the new version number (starting from 0).
* **`search_memory(query: str)`**: Queries the user's long-term memory using the configured `memory_service`. This is useful for retrieving relevant information from past interactions or stored knowledge. The structure of the **SearchMemoryResponse** depends on the specific memory service implementation but typically contains relevant text snippets or conversation excerpts.

#### Example[¶](#example_2 "Permanent link")

```
from google.adk.tools import ToolContext, FunctionTool
from google.genai import types

def process_document(document_name: str, analysis_query: str, tool_context: ToolContext) -> dict:
    """Analyzes a document using context from memory."""

    # 1. Load the artifact
    print(f"Tool: Attempting to load artifact: {document_name}")
    document_part = tool_context.load_artifact(document_name)

    if not document_part:
        return {"status": "error", "message": f"Document '{document_name}' not found."}

    document_text = document_part.text # Assuming it's text for simplicity
    print(f"Tool: Loaded document '{document_name}' ({len(document_text)} chars).")

    # 2. Search memory for related context
    print(f"Tool: Searching memory for context related to: '{analysis_query}'")
    memory_response = tool_context.search_memory(f"Context for analyzing document about {analysis_query}")
    memory_context = "\n".join([m.events[0].content.parts[0].text for m in memory_response.memories if m.events and m.events[0].content]) # Simplified extraction
    print(f"Tool: Found memory context: {memory_context[:100]}...")

    # 3. Perform analysis (placeholder)
    analysis_result = f"Analysis of '{document_name}' regarding '{analysis_query}' using memory context: [Placeholder Analysis Result]"
    print("Tool: Performed analysis.")

    # 4. Save the analysis result as a new artifact
    analysis_part = types.Part.from_text(text=analysis_result)
    new_artifact_name = f"analysis_{document_name}"
    version = tool_context.save_artifact(new_artifact_name, analysis_part)
    print(f"Tool: Saved analysis result as '{new_artifact_name}' version {version}.")

    return {"status": "success", "analysis_artifact": new_artifact_name, "version": version}

doc_analysis_tool = FunctionTool(func=process_document)

# In an Agent:
# Assume artifact 'report.txt' was previously saved.
# Assume memory service is configured and has relevant past data.
# my_agent = Agent(..., tools=[doc_analysis_tool], artifact_service=..., memory_service=...)

```

By leveraging the **ToolContext**, developers can create more sophisticated and context-aware custom tools that seamlessly integrate with ADK's architecture and enhance the overall capabilities of their agents.

## Defining Effective Tool Functions[¶](#defining-effective-tool-functions "Permanent link")

When using a standard Python function as an ADK Tool, how you define it significantly impacts the agent's ability to use it correctly. The agent's Large Language Model (LLM) relies heavily on the function's **name**, **parameters (arguments)**, **type hints**, and **docstring** to understand its purpose and generate the correct call.

Here are key guidelines for defining effective tool functions:

* **Function Name:**

  + Use descriptive, verb-noun based names that clearly indicate the action (e.g., `get_weather`, `search_documents`, `schedule_meeting`).
  + Avoid generic names like `run`, `process`, `handle_data`, or overly ambiguous names like `do_stuff`. Even with a good description, a name like `do_stuff` might confuse the model about when to use the tool versus, for example, `cancel_flight`.
  + The LLM uses the function name as a primary identifier during tool selection.
* **Parameters (Arguments):**

  + Your function can have any number of parameters.
  + Use clear and descriptive names (e.g., `city` instead of `c`, `search_query` instead of `q`).
  + **Provide type hints** for all parameters (e.g., `city: str`, `user_id: int`, `items: list[str]`). This is essential for ADK to generate the correct schema for the LLM.
  + Ensure all parameter types are **JSON serializable**. Standard Python types like `str`, `int`, `float`, `bool`, `list`, `dict`, and their combinations are generally safe. Avoid complex custom class instances as direct parameters unless they have a clear JSON representation.
  + **Do not set default values** for parameters. E.g., `def my_func(param1: str = "default")`. Default values are not reliably supported or used by the underlying models during function call generation. All necessary information should be derived by the LLM from the context or explicitly requested if missing.
* **Return Type:**

  + The function's return value **must be a dictionary (`dict`)**.
  + If your function returns a non-dictionary type (e.g., a string, number, list), the ADK framework will automatically wrap it into a dictionary like `{'result': your_original_return_value}` before passing the result back to the model.
  + Design the dictionary keys and values to be **descriptive and easily understood *by the LLM***. Remember, the model reads this output to decide its next step.
  + Include meaningful keys. For example, instead of returning just an error code like `500`, return `{'status': 'error', 'error_message': 'Database connection failed'}`.
  + It's a **highly recommended practice** to include a `status` key (e.g., `'success'`, `'error'`, `'pending'`, `'ambiguous'`) to clearly indicate the outcome of the tool execution for the model.
* **Docstring:**

  + **This is critical.** The docstring is the primary source of descriptive information for the LLM.
  + **Clearly state what the tool *does*.** Be specific about its purpose and limitations.
  + **Explain *when* the tool should be used.** Provide context or example scenarios to guide the LLM's decision-making.
  + **Describe *each parameter* clearly.** Explain what information the LLM needs to provide for that argument.
  + Describe the **structure and meaning of the expected `dict` return value**, especially the different `status` values and associated data keys.

  **Example of a good definition:**

  ```
  def lookup_order_status(order_id: str) -> dict:
    """Fetches the current status of a customer's order using its ID.

    Use this tool ONLY when a user explicitly asks for the status of
    a specific order and provides the order ID. Do not use it for
    general inquiries.

    Args:
        order_id: The unique identifier of the order to look up.

    Returns:
        A dictionary containing the order status.
        Possible statuses: 'shipped', 'processing', 'pending', 'error'.
        Example success: {'status': 'shipped', 'tracking_number': '1Z9...'}
        Example error: {'status': 'error', 'error_message': 'Order ID not found.'}
    """
    # ... function implementation to fetch status ...
    if status := fetch_status_from_backend(order_id):
         return {"status": status.state, "tracking_number": status.tracking} # Example structure
    else:
         return {"status": "error", "error_message": f"Order ID {order_id} not found."}

  ```
* **Simplicity and Focus:**

  + **Keep Tools Focused:** Each tool should ideally perform one well-defined task.
  + **Fewer Parameters are Better:** Models generally handle tools with fewer, clearly defined parameters more reliably than those with many optional or complex ones.
  + **Use Simple Data Types:** Prefer basic types (`str`, `int`, `bool`, `float`, `List[str]`, etc.) over complex custom classes or deeply nested structures as parameters when possible.
  + **Decompose Complex Tasks:** Break down functions that perform multiple distinct logical steps into smaller, more focused tools. For instance, instead of a single `update_user_profile(profile: ProfileObject)` tool, consider separate tools like `update_user_name(name: str)`, `update_user_address(address: str)`, `update_user_preferences(preferences: list[str])`, etc. This makes it easier for the LLM to select and use the correct capability.

By adhering to these guidelines, you provide the LLM with the clarity and structure it needs to effectively utilize your custom function tools, leading to more capable and reliable agent behavior.

Back to top

---


## Function tools - Agent Development Kit

URL: https://google.github.io/adk-docs/tools/function-tools/

# Function tools[¶](#function-tools "Permanent link")

## What are function tools?[¶](#what-are-function-tools "Permanent link")

When out-of-the-box tools don't fully meet specific requirements, developers can create custom function tools. This allows for **tailored functionality**, such as connecting to proprietary databases or implementing unique algorithms.

*For example,* a function tool, "myfinancetool", might be a function that calculates a specific financial metric. ADK also supports long running functions, so if that calculation takes a while, the agent can continue working on other tasks.

ADK offers several ways to create functions tools, each suited to different levels of complexity and control:

1. Function Tool
2. Long Running Function Tool
3. Agents-as-a-Tool

## 1. Function Tool[¶](#1-function-tool "Permanent link")

Transforming a function into a tool is a straightforward way to integrate custom logic into your agents. This approach offers flexibility and quick integration.

### Parameters[¶](#parameters "Permanent link")

Define your function parameters using standard **JSON-serializable types** (e.g., string, integer, list, dictionary). It's important to avoid setting default values for parameters, as the language model (LLM) does not currently support interpreting them.

### Return Type[¶](#return-type "Permanent link")

The preferred return type for a Python Function Tool is a **dictionary**. This allows you to structure the response with key-value pairs, providing context and clarity to the LLM. If your function returns a type other than a dictionary, the framework automatically wraps it into a dictionary with a single key named **"result"**.

Strive to make your return values as descriptive as possible. *For example,* instead of returning a numeric error code, return a dictionary with an "error\_message" key containing a human-readable explanation. **Remember that the LLM**, not a piece of code, needs to understand the result. As a best practice, include a "status" key in your return dictionary to indicate the overall outcome (e.g., "success", "error", "pending"), providing the LLM with a clear signal about the operation's state.

### Docstring[¶](#docstring "Permanent link")

The docstring of your function serves as the tool's description and is sent to the LLM. Therefore, a well-written and comprehensive docstring is crucial for the LLM to understand how to use the tool effectively. Clearly explain the purpose of the function, the meaning of its parameters, and the expected return values.

Example

This tool is a python function which obtains the Stock price of a given Stock ticker/ symbol.

Note: You need to `pip install yfinance` library before using this tool.

```
from google.adk.agents import Agent
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService
from google.genai import types

import yfinance as yf


APP_NAME = "stock_app"
USER_ID = "1234"
SESSION_ID = "session1234"

def get_stock_price(symbol: str):
    """
    Retrieves the current stock price for a given symbol.

    Args:
        symbol (str): The stock symbol (e.g., "AAPL", "GOOG").

    Returns:
        float: The current stock price, or None if an error occurs.
    """
    try:
        stock = yf.Ticker(symbol)
        historical_data = stock.history(period="1d")
        if not historical_data.empty:
            current_price = historical_data['Close'].iloc[-1]
            return current_price
        else:
            return None
    except Exception as e:
        print(f"Error retrieving stock price for {symbol}: {e}")
        return None


stock_price_agent = Agent(
    model='gemini-2.0-flash',
    name='stock_agent',
    instruction= 'You are an agent who retrieves stock prices. If a ticker symbol is provided, fetch the current price. If only a company name is given, first perform a Google search to find the correct ticker symbol before retrieving the stock price. If the provided ticker symbol is invalid or data cannot be retrieved, inform the user that the stock price could not be found.',
    description='This agent specializes in retrieving real-time stock prices. Given a stock ticker symbol (e.g., AAPL, GOOG, MSFT) or the stock name, use the tools and reliable data sources to provide the most up-to-date price.',
    tools=[get_stock_price],
)


# Session and Runner
session_service = InMemorySessionService()
session = session_service.create_session(app_name=APP_NAME, user_id=USER_ID, session_id=SESSION_ID)
runner = Runner(agent=stock_price_agent, app_name=APP_NAME, session_service=session_service)


# Agent Interaction
def call_agent(query):
    content = types.Content(role='user', parts=[types.Part(text=query)])
    events = runner.run(user_id=USER_ID, session_id=SESSION_ID, new_message=content)

    for event in events:
        if event.is_final_response():
            final_response = event.content.parts[0].text
            print("Agent Response: ", final_response)

call_agent("stock price of GOOG")

```

The return value from this tool will be wrapped into a dictionary.

```
{"result": "$123"}

```

### Best Practices[¶](#best-practices "Permanent link")

While you have considerable flexibility in defining your function, remember that simplicity enhances usability for the LLM. Consider these guidelines:

* **Fewer Parameters are Better:** Minimize the number of parameters to reduce complexity.
* **Simple Data Types:** Favor primitive data types like `str` and `int` over custom classes whenever possible.
* **Meaningful Names:** The function's name and parameter names significantly influence how the LLM interprets and utilizes the tool. Choose names that clearly reflect the function's purpose and the meaning of its inputs. Avoid generic names like `do_stuff()`.

## 2. Long Running Function Tool[¶](#2-long-running-function-tool "Permanent link")

Designed for tasks that require a significant amount of processing time without blocking the agent's execution. This tool is a subclass of `FunctionTool`.

When using a `LongRunningFunctionTool`, your Python function can initiate the long-running operation and optionally return an **intermediate result** to keep the model and user informed about the progress. The agent can then continue with other tasks. An example is the human-in-the-loop scenario where the agent needs human approval before proceeding with a task.

### How it Works[¶](#how-it-works "Permanent link")

You wrap a Python *generator* function (a function using `yield`) with `LongRunningFunctionTool`.

1. **Initiation:** When the LLM calls the tool, your generator function starts executing.
2. **Intermediate Updates (`yield`):** Your function should yield intermediate Python objects (typically dictionaries) periodically to report progress. The ADK framework takes each yielded value and sends it back to the LLM packaged within a `FunctionResponse`. This allows the LLM to inform the user (e.g., status, percentage complete, messages).
3. **Completion (`return`):** When the task is finished, the generator function uses `return` to provide the final Python object result.
4. **Framework Handling:** The ADK framework manages the execution. It sends each yielded value back as an intermediate `FunctionResponse`. When the generator completes, the framework sends the returned value as the content of the final `FunctionResponse`, signaling the end of the long-running operation to the LLM.

### Creating the Tool[¶](#creating-the-tool "Permanent link")

Define your generator function and wrap it using the `LongRunningFunctionTool` class:

```
from google.adk.tools import LongRunningFunctionTool

# Define your generator function (see example below)
def my_long_task_generator(*args, **kwargs):
    # ... setup ...
    yield {"status": "pending", "message": "Starting task..."} # Framework sends this as FunctionResponse
    # ... perform work incrementally ...
    yield {"status": "pending", "progress": 50}               # Framework sends this as FunctionResponse
    # ... finish work ...
    return {"status": "completed", "result": "Final outcome"} # Framework sends this as final FunctionResponse

# Wrap the function
my_tool = LongRunningFunctionTool(func=my_long_task_generator)

```

### Intermediate Updates[¶](#intermediate-updates "Permanent link")

Yielding structured Python objects (like dictionaries) is crucial for providing meaningful updates. Include keys like:

* status: e.g., "pending", "running", "waiting\_for\_input"
* progress: e.g., percentage, steps completed
* message: Descriptive text for the user/LLM
* estimated\_completion\_time: If calculable

Each value you yield is packaged into a FunctionResponse by the framework and sent to the LLM.

### Final Result[¶](#final-result "Permanent link")

The Python object your generator function returns is considered the final result of the tool execution. The framework packages this value (even if it's None) into the content of the final `FunctionResponse` sent back to the LLM, indicating the tool execution is complete.

Example: File Processing Simulation

```
import time
from google.adk.agents import Agent
from google.adk.runners import Runner
from google.adk.tools import LongRunningFunctionTool
from google.adk.sessions import InMemorySessionService
from google.genai import types

# 1. Define the generator function
def process_large_file(file_path: str) -> dict:
    """
    Simulates processing a large file, yielding progress updates.

    Args:
      file_path: Path to the file being processed.

    Returns: 
      A final status dictionary.
    """
    total_steps = 5

    # This dict will be sent in the first FunctionResponse
    yield {"status": "pending", "message": f"Starting processing for {file_path}..."}

    for i in range(total_steps):
        time.sleep(1)  # Simulate work for one step
        progress = (i + 1) / total_steps
        # Each yielded dict is sent in a subsequent FunctionResponse
        yield {
            "status": "pending",
            "progress": f"{int(progress * 100)}%",
            "estimated_completion_time": f"~{total_steps - (i + 1)} seconds remaining"
        }

    # This returned dict will be sent in the final FunctionResponse
    return {"status": "completed", "result": f"Successfully processed file: {file_path}"}

# 2. Wrap the function with LongRunningFunctionTool
long_running_tool = LongRunningFunctionTool(func=process_large_file)

# 3. Use the tool in an Agent
file_processor_agent = Agent(
    # Use a model compatible with function calling
    model="gemini-2.0-flash",
    name='file_processor_agent',
    instruction="""You are an agent that processes large files. When the user provides a file path, use the 'process_large_file' tool. Keep the user informed about the progress based on the tool's updates (which arrive as function responses). Only provide the final result when the tool indicates completion in its final function response.""",
    tools=[long_running_tool]
)


APP_NAME = "file_processor"
USER_ID = "1234"
SESSION_ID = "session1234"

# Session and Runner
session_service = InMemorySessionService()
session = session_service.create_session(app_name=APP_NAME, user_id=USER_ID, session_id=SESSION_ID)
runner = Runner(agent=file_processor_agent, app_name=APP_NAME, session_service=session_service)


# Agent Interaction
def call_agent(query):
    content = types.Content(role='user', parts=[types.Part(text=query)])
    events = runner.run(user_id=USER_ID, session_id=SESSION_ID, new_message=content)

    for event in events:
        if event.is_final_response():
            final_response = event.content.parts[0].text
            print("Agent Response: ", final_response)

call_agent("Replace with a path to your file...")

```

#### Key aspects of this example[¶](#key-aspects-of-this-example "Permanent link")

* **process\_large\_file**: This generator simulates a lengthy operation, yielding intermediate status/progress dictionaries.
* **`LongRunningFunctionTool`**: Wraps the generator; the framework handles sending yielded updates and the final return value as sequential FunctionResponses.
* **Agent instruction**: Directs the LLM to use the tool and understand the incoming FunctionResponse stream (progress vs. completion) for user updates.
* **Final return**: The function returns the final result dictionary, which is sent in the concluding FunctionResponse to indicate completion.

## 3. Agent-as-a-Tool[¶](#3-agent-as-a-tool "Permanent link")

This powerful feature allows you to leverage the capabilities of other agents within your system by calling them as tools. The Agent-as-a-Tool enables you to invoke another agent to perform a specific task, effectively **delegating responsibility**. This is conceptually similar to creating a Python function that calls another agent and uses the agent's response as the function's return value.

### Key difference from sub-agents[¶](#key-difference-from-sub-agents "Permanent link")

It's important to distinguish an Agent-as-a-Tool from a Sub-Agent.

* **Agent-as-a-Tool:** When Agent A calls Agent B as a tool (using Agent-as-a-Tool), Agent B's answer is **passed back** to Agent A, which then summarizes the answer and generates a response to the user. Agent A retains control and continues to handle future user input.
* **Sub-agent:** When Agent A calls Agent B as a sub-agent, the responsibility of answering the user is completely **transferred to Agent B**. Agent A is effectively out of the loop. All subsequent user input will be answered by Agent B.

### Usage[¶](#usage "Permanent link")

To use an agent as a tool, wrap the agent with the AgentTool class.

```
tools=[AgentTool(agent=agent_b)]

```

### Customization[¶](#customization "Permanent link")

The `AgentTool` class provides the following attributes for customizing its behavior:

* **skip\_summarization: bool:** If set to True, the framework will **bypass the LLM-based summarization** of the tool agent's response. This can be useful when the tool's response is already well-formatted and requires no further processing.

Example

```
from google.adk.agents import Agent
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService
from google.adk.tools.agent_tool import AgentTool
from google.genai import types

APP_NAME="summary_agent"
USER_ID="user1234"
SESSION_ID="1234"

summary_agent = Agent(
    model="gemini-2.0-flash",
    name="summary_agent",
    instruction="""You are an expert summarizer. Please read the following text and provide a concise summary.""",
    description="Agent to summarize text",
)

root_agent = Agent(
    model='gemini-2.0-flash',
    name='root_agent',
    instruction="""You are a helpful assistant. When the user provides a text, use the 'summarize' tool to generate a summary. Always forward the user's message exactly as received to the 'summarize' tool, without modifying or summarizing it yourself. Present the response from the tool to the user.""",
    tools=[AgentTool(agent=summary_agent)]
)

# Session and Runner
session_service = InMemorySessionService()
session = session_service.create_session(app_name=APP_NAME, user_id=USER_ID, session_id=SESSION_ID)
runner = Runner(agent=root_agent, app_name=APP_NAME, session_service=session_service)


# Agent Interaction
def call_agent(query):
    content = types.Content(role='user', parts=[types.Part(text=query)])
    events = runner.run(user_id=USER_ID, session_id=SESSION_ID, new_message=content)

    for event in events:
        if event.is_final_response():
            final_response = event.content.parts[0].text
            print("Agent Response: ", final_response)


long_text = """Quantum computing represents a fundamentally different approach to computation, 
leveraging the bizarre principles of quantum mechanics to process information. Unlike classical computers 
that rely on bits representing either 0 or 1, quantum computers use qubits which can exist in a state of superposition - effectively 
being 0, 1, or a combination of both simultaneously. Furthermore, qubits can become entangled, 
meaning their fates are intertwined regardless of distance, allowing for complex correlations. This parallelism and 
interconnectedness grant quantum computers the potential to solve specific types of incredibly complex problems - such 
as drug discovery, materials science, complex system optimization, and breaking certain types of cryptography - far 
faster than even the most powerful classical supercomputers could ever achieve, although the technology is still largely in its developmental stages."""


call_agent(long_text)

```

### How it works[¶](#how-it-works_1 "Permanent link")

1. When the `main_agent` receives the long text, its instruction tells it to use the 'summarize' tool for long texts.
2. The framework recognizes 'summarize' as an `AgentTool` that wraps the `summary_agent`.
3. Behind the scenes, the `main_agent` will call the `summary_agent` with the long text as input.
4. The `summary_agent` will process the text according to its instruction and generate a summary.
5. **The response from the `summary_agent` is then passed back to the `main_agent`.**
6. The `main_agent` can then take the summary and formulate its final response to the user (e.g., "Here's a summary of the text: ...")

Back to top

---


## Built-in tools - Agent Development Kit

URL: https://google.github.io/adk-docs/tools/built-in-tools/

# Built-in tools[¶](#built-in-tools "Permanent link")

These built-in tools provide ready-to-use functionality such as Google Search or
code executors that provide agents with common capabilities. For instance, an
agent that needs to retrieve information from the web can directly use the
**google\_search** tool without any additional setup.

## How to Use[¶](#how-to-use "Permanent link")

1. **Import:** Import the desired tool from the `agents.tools` module.
2. **Configure:** Initialize the tool, providing required parameters if any.
3. **Register:** Add the initialized tool to the **tools** list of your Agent.

Once added to an agent, the agent can decide to use the tool based on the **user
prompt** and its **instructions**. The framework handles the execution of the
tool when the agent calls it.

## Available Built-in tools[¶](#available-built-in-tools "Permanent link")

### Google Search[¶](#google-search "Permanent link")

The `google_search` tool allows the agent to perform web searches using Google
Search. It is compatible with Gemini 2 models, and you can add this tool to the
agent's tools list.

```
from google.adk.agents import Agent
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService
from google.adk.tools import google_search
from google.genai import types

APP_NAME="google_search_agent"
USER_ID="user1234"
SESSION_ID="1234"


root_agent = Agent(
    name="basic_search_agent",
    model="gemini-2.0-flash",
    description="Agent to answer questions using Google Search.",
    instruction="I can answer your questions by searching the internet. Just ask me anything!",
    # google_search is a pre-built tool which allows the agent to perform Google searches.
    tools=[google_search]
)

# Session and Runner
session_service = InMemorySessionService()
session = session_service.create_session(app_name=APP_NAME, user_id=USER_ID, session_id=SESSION_ID)
runner = Runner(agent=root_agent, app_name=APP_NAME, session_service=session_service)


# Agent Interaction
def call_agent(query):
    """
    Helper function to call the agent with a query.
    """
    content = types.Content(role='user', parts=[types.Part(text=query)])
    events = runner.run(user_id=USER_ID, session_id=SESSION_ID, new_message=content)

    for event in events:
        if event.is_final_response():
            final_response = event.content.parts[0].text
            print("Agent Response: ", final_response)

call_agent("what's the latest ai news?")

```

### Code Execution[¶](#code-execution "Permanent link")

The `built_in_code_execution` tool enables the agent to execute code,
specifically when using Gemini 2 models. This allows the model to perform tasks
like calculations, data manipulation, or running small scripts.

```
import asyncio
from google.adk.agents import LlmAgent
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService
from google.adk.tools import built_in_code_execution
from google.genai import types

AGENT_NAME="calculator_agent"
APP_NAME="calculator"
USER_ID="user1234"
SESSION_ID="session_code_exec_async"
GEMINI_MODEL = "gemini-2.0-flash"

# Agent Definition
code_agent = LlmAgent(
    name=AGENT_NAME,
    model=GEMINI_MODEL,
    tools=[built_in_code_execution],
    instruction="""You are a calculator agent.
    When given a mathematical expression, write and execute Python code to calculate the result.
    Return only the final numerical result as plain text, without markdown or code blocks.
    """,
    description="Executes Python code to perform calculations.",
)

# Session and Runner
session_service = InMemorySessionService()
session = session_service.create_session(app_name=APP_NAME, user_id=USER_ID, session_id=SESSION_ID)
runner = Runner(agent=code_agent, app_name=APP_NAME, session_service=session_service)

# Agent Interaction (Async)
async def call_agent_async(query):
    content = types.Content(role='user', parts=[types.Part(text=query)])
    print(f"\n--- Running Query: {query} ---")
    final_response_text = "No final text response captured."
    try:
        # Use run_async
        async for event in runner.run_async(user_id=USER_ID, session_id=SESSION_ID, new_message=content):
            print(f"Event ID: {event.id}, Author: {event.author}")

            # --- Check for specific parts FIRST ---
            has_specific_part = False
            if event.content and event.content.parts:
                for part in event.content.parts: # Iterate through all parts
                    if part.executable_code:
                        # Access the actual code string via .code
                        print(f"  Debug: Agent generated code:\n```python\n{part.executable_code.code}\n```")
                        has_specific_part = True
                    elif part.code_execution_result:
                        # Access outcome and output correctly
                        print(f"  Debug: Code Execution Result: {part.code_execution_result.outcome} - Output:\n{part.code_execution_result.output}")
                        has_specific_part = True
                    # Also print any text parts found in any event for debugging
                    elif part.text and not part.text.isspace():
                        print(f"  Text: '{part.text.strip()}'")
                        # Do not set has_specific_part=True here, as we want the final response logic below

            # --- Check for final response AFTER specific parts ---
            # Only consider it final if it doesn't have the specific code parts we just handled
            if not has_specific_part and event.is_final_response():
                if event.content and event.content.parts and event.content.parts[0].text:
                    final_response_text = event.content.parts[0].text.strip()
                    print(f"==> Final Agent Response: {final_response_text}")
                else:
                    print("==> Final Agent Response: [No text content in final event]")


    except Exception as e:
        print(f"ERROR during agent run: {e}")
    print("-" * 30)


# Main async function to run the examples
async def main():
    await call_agent_async("Calculate the value of (5 + 7) * 3")
    await call_agent_async("What is 10 factorial?")

# Execute the main async function
try:
    asyncio.run(main())
except RuntimeError as e:
    # Handle specific error when running asyncio.run in an already running loop (like Jupyter/Colab)
    if "cannot be called from a running event loop" in str(e):
        print("\nRunning in an existing event loop (like Colab/Jupyter).")
        print("Please run `await main()` in a notebook cell instead.")
        # If in an interactive environment like a notebook, you might need to run:
        # await main()
    else:
        raise e # Re-raise other runtime errors

```

### Vertex AI Search[¶](#vertex-ai-search "Permanent link")

The `vertex_ai_search_tool` uses Google Cloud's Vertex AI Search, enabling the
agent to search across your private, configured data stores (e.g., internal
documents, company policies, knowledge bases). This built-in tool requires you
to provide the specific data store ID during configuration.

```
import asyncio

from google.adk.agents import LlmAgent
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService
from google.genai import types
from google.adk.tools import VertexAiSearchTool

# Replace with your actual Vertex AI Search Datastore ID
# Format: projects/<PROJECT_ID>/locations/<LOCATION>/collections/default_collection/dataStores/<DATASTORE_ID>
# e.g., "projects/12345/locations/us-central1/collections/default_collection/dataStores/my-datastore-123"
YOUR_DATASTORE_ID = "YOUR_DATASTORE_ID_HERE"

# Constants
APP_NAME_VSEARCH = "vertex_search_app"
USER_ID_VSEARCH = "user_vsearch_1"
SESSION_ID_VSEARCH = "session_vsearch_1"
AGENT_NAME_VSEARCH = "doc_qa_agent"
GEMINI_2_FLASH = "gemini-2.0-flash"

# Tool Instantiation
# You MUST provide your datastore ID here.
vertex_search_tool = VertexAiSearchTool(data_store_id=YOUR_DATASTORE_ID)

# Agent Definition
doc_qa_agent = LlmAgent(
    name=AGENT_NAME_VSEARCH,
    model=GEMINI_2_FLASH, # Requires Gemini model
    tools=[vertex_search_tool],
    instruction=f"""You are a helpful assistant that answers questions based on information found in the document store: {YOUR_DATASTORE_ID}.
    Use the search tool to find relevant information before answering.
    If the answer isn't in the documents, say that you couldn't find the information.
    """,
    description="Answers questions using a specific Vertex AI Search datastore.",
)

# Session and Runner Setup
session_service_vsearch = InMemorySessionService()
runner_vsearch = Runner(
    agent=doc_qa_agent, app_name=APP_NAME_VSEARCH, session_service=session_service_vsearch
)
session_vsearch = session_service_vsearch.create_session(
    app_name=APP_NAME_VSEARCH, user_id=USER_ID_VSEARCH, session_id=SESSION_ID_VSEARCH
)

# Agent Interaction Function
async def call_vsearch_agent_async(query):
    print("\n--- Running Vertex AI Search Agent ---")
    print(f"Query: {query}")
    if "YOUR_DATASTORE_ID_HERE" in YOUR_DATASTORE_ID:
        print("Skipping execution: Please replace YOUR_DATASTORE_ID_HERE with your actual datastore ID.")
        print("-" * 30)
        return

    content = types.Content(role='user', parts=[types.Part(text=query)])
    final_response_text = "No response received."
    try:
        async for event in runner_vsearch.run_async(
            user_id=USER_ID_VSEARCH, session_id=SESSION_ID_VSEARCH, new_message=content
        ):
            # Like Google Search, results are often embedded in the model's response.
            if event.is_final_response() and event.content and event.content.parts:
                final_response_text = event.content.parts[0].text.strip()
                print(f"Agent Response: {final_response_text}")
                # You can inspect event.grounding_metadata for source citations
                if event.grounding_metadata:
                    print(f"  (Grounding metadata found with {len(event.grounding_metadata.grounding_attributions)} attributions)")

    except Exception as e:
        print(f"An error occurred: {e}")
        print("Ensure your datastore ID is correct and the service account has permissions.")
    print("-" * 30)

# --- Run Example ---
async def run_vsearch_example():
    # Replace with a question relevant to YOUR datastore content
    await call_vsearch_agent_async("Summarize the main points about the Q2 strategy document.")
    await call_vsearch_agent_async("What safety procedures are mentioned for lab X?")

# Execute the example
# await run_vsearch_example()

# Running locally due to potential colab asyncio issues with multiple awaits
try:
    asyncio.run(run_vsearch_example())
except RuntimeError as e:
    if "cannot be called from a running event loop" in str(e):
        print("Skipping execution in running event loop (like Colab/Jupyter). Run locally.")
    else:
        raise e

```

## Use Built-in tools with other tools[¶](#use-built-in-tools-with-other-tools "Permanent link")

The following code sample demonstrates how to use multiple built-in tools or how
to use built-in tools with other tools by using multiple agents:

```
from google.adk.tools import agent_tool
from google.adk.agents import Agent
from google.adk.tools import google_search, built_in_code_execution

search_agent = Agent(
    model='gemini-2.0-flash',
    name='SearchAgent',
    instruction="""
    You're a specialist in Google Search
    """,
    tools=[google_search],
)
coding_agent = Agent(
    model='gemini-2.0-flash',
    name='CodeAgent',
    instruction="""
    You're a specialist in Code Execution
    """,
    tools=[built_in_code_execution],
)
root_agent = Agent(
    name="RootAgent",
    model="gemini-2.0-flash",
    description="Root Agent",
    tools=[agent_tool.AgentTool(agent=search_agent), agent_tool.AgentTool(agent=coding_agent)],
)

```

### Limitations[¶](#limitations "Permanent link")

Warning

Currently, for each root agent or single agent, only one built-in tool is
supported.

For example, the following approach that uses two or more built-in tools within
a root agent (or a single agent) is **not** currently supported:

```
root_agent = Agent(
    name="RootAgent",
    model="gemini-2.0-flash",
    description="Root Agent",
    tools=[built_in_code_execution, custom_function],
)

```

Warning

Built-in tools cannot be used within a sub-agent.

For example, the following approach that uses built-in tools within sub-agents
is **not** currently supported:

```
search_agent = Agent(
    model='gemini-2.0-flash',
    name='SearchAgent',
    instruction="""
    You're a specialist in Google Search
    """,
    tools=[google_search],
)
coding_agent = Agent(
    model='gemini-2.0-flash',
    name='CodeAgent',
    instruction="""
    You're a specialist in Code Execution
    """,
    tools=[built_in_code_execution],
)
root_agent = Agent(
    name="RootAgent",
    model="gemini-2.0-flash",
    description="Root Agent",
    sub_agents=[
        search_agent,
        coding_agent
    ],
)

```

Back to top

---


## Third party tools - Agent Development Kit

URL: https://google.github.io/adk-docs/tools/third-party-tools/

# Third Party Tools[¶](#third-party-tools "Permanent link")

ADK is designed to be **highly extensible, allowing you to seamlessly integrate tools from other AI Agent frameworks** like CrewAI and LangChain. This interoperability is crucial because it allows for faster development time and allows you to reuse existing tools.

## 1. Using LangChain Tools[¶](#1-using-langchain-tools "Permanent link")

ADK provides the `LangchainTool` wrapper to integrate tools from the LangChain ecosystem into your agents.

### Example: Web Search using LangChain's Tavily tool[¶](#example-web-search-using-langchains-tavily-tool "Permanent link")

[Tavily](https://tavily.com/) provides a search API that returns answers derived from real-time search results, intended for use by applications like AI agents.

1. Follow [ADK installation and setup](../../get-started/installation/) guide.
2. **Install Dependencies:** Ensure you have the necessary LangChain packages installed. For example, to use the Tavily search tool, install its specific dependencies:

   ```
   pip install langchain_community tavily-python

   ```
3. Obtain a [Tavily](https://tavily.com/) API KEY and export it as an environment variable.

   ```
   export TAVILY_API_KEY=<REPLACE_WITH_API_KEY>

   ```
4. **Import:** Import the `LangchainTool` wrapper from ADK and the specific `LangChain` tool you wish to use (e.g, `TavilySearchResults`).

   ```
   from google.adk.tools.langchain_tool import LangchainTool
   from langchain_community.tools import TavilySearchResults

   ```
5. **Instantiate & Wrap:** Create an instance of your LangChain tool and pass it to the `LangchainTool` constructor.

   ```
   # Instantiate the LangChain tool
   tavily_tool_instance = TavilySearchResults(
       max_results=5,
       search_depth="advanced",
       include_answer=True,
       include_raw_content=True,
       include_images=True,
   )

   # Wrap it with LangchainTool for ADK
   adk_tavily_tool = LangchainTool(tool=tavily_tool_instance)

   ```
6. **Add to Agent:** Include the wrapped `LangchainTool` instance in your agent's `tools` list during definition.

   ```
   from google.adk import Agent

   # Define the ADK agent, including the wrapped tool
   my_agent = Agent(
       name="langchain_tool_agent",
       model="gemini-2.0-flash",
       description="Agent to answer questions using TavilySearch.",
       instruction="I can answer your questions by searching the internet. Just ask me anything!",
       tools=[adk_tavily_tool] # Add the wrapped tool here
   )

   ```

### Full Example: Tavily Search[¶](#full-example-tavily-search "Permanent link")

Here's the full code combining the steps above to create and run an agent using the LangChain Tavily search tool.

```
import os
from google.adk import Agent, Runner
from google.adk.sessions import InMemorySessionService
from google.adk.tools.langchain_tool import LangchainTool
from google.genai import types
from langchain_community.tools import TavilySearchResults

# Ensure TAVILY_API_KEY is set in your environment
if not os.getenv("TAVILY_API_KEY"):
    print("Warning: TAVILY_API_KEY environment variable not set.")

APP_NAME = "news_app"
USER_ID = "1234"
SESSION_ID = "session1234"

# Instantiate LangChain tool
tavily_search = TavilySearchResults(
    max_results=5,
    search_depth="advanced",
    include_answer=True,
    include_raw_content=True,
    include_images=True,
)

# Wrap with LangchainTool
adk_tavily_tool = LangchainTool(tool=tavily_search)

# Define Agent with the wrapped tool
my_agent = Agent(
    name="langchain_tool_agent",
    model="gemini-2.0-flash",
    description="Agent to answer questions using TavilySearch.",
    instruction="I can answer your questions by searching the internet. Just ask me anything!",
    tools=[adk_tavily_tool] # Add the wrapped tool here
)

session_service = InMemorySessionService()
session = session_service.create_session(app_name=APP_NAME, user_id=USER_ID, session_id=SESSION_ID)
runner = Runner(agent=my_agent, app_name=APP_NAME, session_service=session_service)


# Agent Interaction
def call_agent(query):
    content = types.Content(role='user', parts=[types.Part(text=query)])
    events = runner.run(user_id=USER_ID, session_id=SESSION_ID, new_message=content)

    for event in events:
        if event.is_final_response():
            final_response = event.content.parts[0].text
            print("Agent Response: ", final_response)

call_agent("stock price of GOOG")

```

## 2. Using CrewAI tools[¶](#2-using-crewai-tools "Permanent link")

ADK provides the `CrewaiTool` wrapper to integrate tools from the CrewAI library.

### Example: Web Search using CrewAI's Serper API[¶](#example-web-search-using-crewais-serper-api "Permanent link")

[Serper API](https://serper.dev/) provides access to Google Search results programmatically. It allows applications, like AI agents, to perform real-time Google searches (including news, images, etc.) and get structured data back without needing to scrape web pages directly.

1. Follow [ADK installation and setup](../../get-started/installation/) guide.
2. **Install Dependencies:** Install the necessary CrewAI tools package. For example, to use the SerperDevTool:

   ```
   pip install crewai-tools

   ```
3. Obtain a [Serper API KEY](https://serper.dev/) and export it as an environment variable.

   ```
   export SERPER_API_KEY=<REPLACE_WITH_API_KEY>

   ```
4. **Import:** Import `CrewaiTool` from ADK and the desired CrewAI tool (e.g, `SerperDevTool`).

   ```
   from google.adk.tools.crewai_tool import CrewaiTool
   from crewai_tools import SerperDevTool

   ```
5. **Instantiate & Wrap:** Create an instance of the CrewAI tool. Pass it to the `CrewaiTool` constructor. **Crucially, you must provide a name and description** to the ADK wrapper, as these are used by ADK's underlying model to understand when to use the tool.

   ```
   # Instantiate the CrewAI tool
   serper_tool_instance = SerperDevTool(
       n_results=10,
       save_file=False,
       search_type="news",
   )

   # Wrap it with CrewaiTool for ADK, providing name and description
   adk_serper_tool = CrewaiTool(
       name="InternetNewsSearch",
       description="Searches the internet specifically for recent news articles using Serper.",
       tool=serper_tool_instance
   )

   ```
6. **Add to Agent:** Include the wrapped `CrewaiTool` instance in your agent's `tools` list.

   ```
   from google.adk import Agent

   # Define the ADK agent
   my_agent = Agent(
       name="crewai_search_agent",
       model="gemini-2.0-flash",
       description="Agent to find recent news using the Serper search tool.",
       instruction="I can find the latest news for you. What topic are you interested in?",
       tools=[adk_serper_tool] # Add the wrapped tool here
   )

   ```

### Full Example: Serper API[¶](#full-example-serper-api "Permanent link")

Here's the full code combining the steps above to create and run an agent using the CrewAI Serper API search tool.

```
import os
from google.adk import Agent, Runner
from google.adk.sessions import InMemorySessionService
from google.adk.tools.crewai_tool import CrewaiTool
from google.genai import types
from crewai_tools import SerperDevTool


# Constants
APP_NAME = "news_app"
USER_ID = "user1234"
SESSION_ID = "1234"

# Ensure SERPER_API_KEY is set in your environment
if not os.getenv("SERPER_API_KEY"):
    print("Warning: SERPER_API_KEY environment variable not set.")

serper_tool_instance = SerperDevTool(
    n_results=10,
    save_file=False,
    search_type="news",
)

adk_serper_tool = CrewaiTool(
    name="InternetNewsSearch",
    description="Searches the internet specifically for recent news articles using Serper.",
    tool=serper_tool_instance
)

serper_agent = Agent(
    name="basic_search_agent",
    model="gemini-2.0-flash",
    description="Agent to answer questions using Google Search.",
    instruction="I can answer your questions by searching the internet. Just ask me anything!",
    # Add the Serper tool
    tools=[adk_serper_tool]
)

# Session and Runner
session_service = InMemorySessionService()
session = session_service.create_session(app_name=APP_NAME, user_id=USER_ID, session_id=SESSION_ID)
runner = Runner(agent=serper_agent, app_name=APP_NAME, session_service=session_service)


# Agent Interaction
def call_agent(query):
    content = types.Content(role='user', parts=[types.Part(text=query)])
    events = runner.run(user_id=USER_ID, session_id=SESSION_ID, new_message=content)

    for event in events:
        if event.is_final_response():
            final_response = event.content.parts[0].text
            print("Agent Response: ", final_response)

call_agent("what's the latest news on AI Agents?")

```

Back to top

---


## MCP tools - Agent Development Kit

URL: https://google.github.io/adk-docs/tools/mcp-tools/

# Model Context Protocol Tools[¶](#model-context-protocol-tools "Permanent link")

This guide walks you through two ways of integrating Model Context Protocol (MCP) with ADK.

## What is Model Context Protocol (MCP)?[¶](#what-is-model-context-protocol-mcp "Permanent link")

The Model Context Protocol (MCP) is an open standard designed to standardize how Large Language Models (LLMs) like Gemini and Claude communicate with external applications, data sources, and tools. Think of it as a universal connection mechanism that simplifies how LLMs obtain context, execute actions, and interact with various systems.

MCP follows a client-server architecture, defining how **data** (resources), **interactive templates** (prompts), and **actionable functions** (tools) are exposed by an **MCP server** and consumed by an **MCP client** (which could be an LLM host application or an AI agent).

This guide covers two primary integration patterns:

1. **Using Existing MCP Servers within ADK:** An ADK agent acts as an MCP client, leveraging tools provided by external MCP servers.
2. **Exposing ADK Tools via an MCP Server:** Building an MCP server that wraps ADK tools, making them accessible to any MCP client.

## Prerequisites[¶](#prerequisites "Permanent link")

Before you begin, ensure you have the following set up:

* **Set up ADK:** Follow the standard ADK [setup]() instructions in the quickstart.
* **Install/update Python:** MCP requires Python version of 3.9 or higher.
* **Setup Node.js and npx:** Many community MCP servers are distributed as Node.js packages and run using `npx`. Install Node.js (which includes npx) if you haven't already. For details, see <https://nodejs.org/en>.
* **Verify Installations:** Confirm `adk` and `npx` are in your PATH within the activated virtual environment:

```
# Both commands should print the path to the executables.
which adk
which npx

```

## 1. **Using MCP servers with ADK agents (ADK as an MCP client)**[¶](#1-using-mcp-servers-with-adk-agents-adk-as-an-mcp-client "Permanent link")

This section shows two examples of using MCP servers with ADK agents. This is the most common integration pattern. Your ADK agent needs to use functionality provided by an existing service that exposes itself as an MCP Server.

### `MCPToolset` class[¶](#mcptoolset-class "Permanent link")

The examples use the `MCPToolset` class in ADK which acts as the bridge to the MCP server. Your ADK agent uses `MCPToolset` to:

1. **Connect:** Establish a connection to an MCP server process. This can be a local server communicating over standard input/output (`StdioServerParameters`) or a remote server using Server-Sent Events (`SseServerParams`).
2. **Discover:** Query the MCP server for its available tools (`list_tools` MCP method).
3. **Adapt:** Convert the MCP tool schemas into ADK-compatible `BaseTool` instances.
4. **Expose:** Present these adapted tools to the ADK `LlmAgent`.
5. **Proxy Calls:** When the `LlmAgent` decides to use one of these tools, `MCPToolset` forwards the call (`call_tool` MCP method) to the MCP server and returns the result.
6. **Manage Connection:** Handle the lifecycle of the connection to the MCP server process, often requiring explicit cleanup.

### Example 1: File System MCP Server[¶](#example-1-file-system-mcp-server "Permanent link")

This example demonstrates connecting to a local MCP server that provides file system operations.

#### Step 1: Attach the MCP Server to your ADK agent via `MCPToolset`[¶](#step-1-attach-the-mcp-server-to-your-adk-agent-via-mcptoolset "Permanent link")

Create `agent.py` in `./adk_agent_samples/mcp_agent/` and use the following code snippet to define a function that initializes the `MCPToolset`.

* **Important:** Replace `"/path/to/your/folder"` with the **absolute path** to an actual folder on your system.

```
# ./adk_agent_samples/mcp_agent/agent.py
import asyncio
from dotenv import load_dotenv
from google.genai import types
from google.adk.agents.llm_agent import LlmAgent
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService
from google.adk.artifacts.in_memory_artifact_service import InMemoryArtifactService # Optional
from google.adk.tools.mcp_tool.mcp_toolset import MCPToolset, SseServerParams, StdioServerParameters

# Load environment variables from .env file in the parent directory
# Place this near the top, before using env vars like API keys
load_dotenv('../.env')

# --- Step 1: Import Tools from MCP Server ---
async def get_tools_async():
  """Gets tools from the File System MCP Server."""
  print("Attempting to connect to MCP Filesystem server...")
  tools, exit_stack = await MCPToolset.from_server(
      # Use StdioServerParameters for local process communication
      connection_params=StdioServerParameters(
          command='npx', # Command to run the server
          args=["-y",    # Arguments for the command
                "@modelcontextprotocol/server-filesystem",
                # TODO: IMPORTANT! Change the path below to an ABSOLUTE path on your system.
                "/path/to/your/folder"],
      )
      # For remote servers, you would use SseServerParams instead:
      # connection_params=SseServerParams(url="http://remote-server:port/path", headers={...})
  )
  print("MCP Toolset created successfully.")
  # MCP requires maintaining a connection to the local MCP Server.
  # exit_stack manages the cleanup of this connection.
  return tools, exit_stack

# --- Step 2: Agent Definition ---
async def get_agent_async():
  """Creates an ADK Agent equipped with tools from the MCP Server."""
  tools, exit_stack = await get_tools_async()
  print(f"Fetched {len(tools)} tools from MCP server.")
  root_agent = LlmAgent(
      model='gemini-2.0-flash', # Adjust model name if needed based on availability
      name='filesystem_assistant',
      instruction='Help user interact with the local filesystem using available tools.',
      tools=tools, # Provide the MCP tools to the ADK agent
  )
  return root_agent, exit_stack

# --- Step 3: Main Execution Logic ---
async def async_main():
  session_service = InMemorySessionService()
  # Artifact service might not be needed for this example
  artifacts_service = InMemoryArtifactService()

  session = session_service.create_session(
      state={}, app_name='mcp_filesystem_app', user_id='user_fs'
  )

  # TODO: Change the query to be relevant to YOUR specified folder.
  # e.g., "list files in the 'documents' subfolder" or "read the file 'notes.txt'"
  query = "list files in the tests folder"
  print(f"User Query: '{query}'")
  content = types.Content(role='user', parts=[types.Part(text=query)])

  root_agent, exit_stack = await get_agent_async()

  runner = Runner(
      app_name='mcp_filesystem_app',
      agent=root_agent,
      artifact_service=artifacts_service, # Optional
      session_service=session_service,
  )

  print("Running agent...")
  events_async = runner.run_async(
      session_id=session.id, user_id=session.user_id, new_message=content
  )

  async for event in events_async:
    print(f"Event received: {event}")

  # Crucial Cleanup: Ensure the MCP server process connection is closed.
  print("Closing MCP server connection...")
  await exit_stack.aclose()
  print("Cleanup complete.")

if __name__ == '__main__':
  try:
    asyncio.run(async_main())
  except Exception as e:
    print(f"An error occurred: {e}")

```

#### Step 2: Observe the result[¶](#step-2-observe-the-result "Permanent link")

Run the script from the adk\_agent\_samples directory (ensure your virtual environment is active):

```
cd ./adk_agent_samples
python3 ./mcp_agent/agent.py

```

The following shows the expected output for the connection attempt, the MCP server starting (via npx), the ADK agent events (including the FunctionCall to list\_directory and the FunctionResponse), and the final agent text response based on the file listing. Ensure the exit\_stack.aclose() runs at the end.

```
User Query: 'list files in the tests folder'
Attempting to connect to MCP Filesystem server...
# --> npx process starts here, potentially logging to stderr/stdout
Secure MCP Filesystem Server running on stdio
Allowed directories: [
  '/path/to/your/folder'
]
# <-- npx process output ends
MCP Toolset created successfully.
Fetched [N] tools from MCP server. # N = number of tools like list_directory, read_file etc.
Running agent...
Event received: content=Content(parts=[Part(..., function_call=FunctionCall(id='...', args={'path': 'tests'}, name='list_directory'), ...)], role='model') ...
Event received: content=Content(parts=[Part(..., function_response=FunctionResponse(id='...', name='list_directory', response={'result': CallToolResult(..., content=[TextContent(...)], ...)}), ...)], role='user') ...
Event received: content=Content(parts=[Part(..., text='https://developers.google.com/maps/get-started#enable-api-sdk')], role='model') ...
Closing MCP server connection...
Cleanup complete.

```

### Example 2: Google Maps MCP Server[¶](#example-2-google-maps-mcp-server "Permanent link")

This follows the same pattern but targets the Google Maps MCP server.

#### Step 1: Get API Key and Enable APIs[¶](#step-1-get-api-key-and-enable-apis "Permanent link")

Follow the directions at [Use API keys](https://developers.google.com/maps/documentation/javascript/get-api-key#create-api-keys) to get a Google Maps API Key.

Enable Directions API and Routes API in your Google Cloud project. For instructions, see [Getting started with Google Maps Platform](https://developers.google.com/maps/get-started#enable-api-sdk) topic.

#### Step 2: Update get\_tools\_async[¶](#step-2-update-get_tools_async "Permanent link")

Modify get\_tools\_async in agent.py to connect to the Maps server, passing your API key via the env parameter of StdioServerParameters.

```
# agent.py (modify get_tools_async and other parts as needed)
import asyncio
from dotenv import load_dotenv
from google.genai import types
from google.adk.agents.llm_agent import LlmAgent
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService
from google.adk.artifacts.in_memory_artifact_service import InMemoryArtifactService # Optional
from google.adk.tools.mcp_tool.mcp_toolset import MCPToolset, SseServerParams, StdioServerParameters

load_dotenv('../.env')

async def get_tools_async():
  """ Step 1: Gets tools from the Google Maps MCP Server."""
  # IMPORTANT: Replace with your actual key
  google_maps_api_key = "YOUR_API_KEY_FROM_STEP_1"
  if "YOUR_API_KEY" in google_maps_api_key:
      raise ValueError("Please replace 'YOUR_API_KEY_FROM_STEP_1' with your actual Google Maps API key.")

  print("Attempting to connect to MCP Google Maps server...")
  tools, exit_stack = await MCPToolset.from_server(
      connection_params=StdioServerParameters(
          command='npx',
          args=["-y",
                "@modelcontextprotocol/server-google-maps",
          ],
          # Pass the API key as an environment variable to the npx process
          env={
              "GOOGLE_MAPS_API_KEY": google_maps_api_key
          }
      )
  )
  print("MCP Toolset created successfully.")
  return tools, exit_stack

# --- Step 2: Agent Definition ---
async def get_agent_async():
  """Creates an ADK Agent equipped with tools from the MCP Server."""
  tools, exit_stack = await get_tools_async()
  print(f"Fetched {len(tools)} tools from MCP server.")
  root_agent = LlmAgent(
      model='gemini-2.0-flash', # Adjust if needed
      name='maps_assistant',
      instruction='Help user with mapping and directions using available tools.',
      tools=tools,
  )
  return root_agent, exit_stack

# --- Step 3: Main Execution Logic (modify query) ---
async def async_main():
  session_service = InMemorySessionService()
  artifacts_service = InMemoryArtifactService() # Optional

  session = session_service.create_session(
      state={}, app_name='mcp_maps_app', user_id='user_maps'
  )

  # TODO: Use specific addresses for reliable results with this server
  query = "What is the route from 1600 Amphitheatre Pkwy to 1165 Borregas Ave"
  print(f"User Query: '{query}'")
  content = types.Content(role='user', parts=[types.Part(text=query)])

  root_agent, exit_stack = await get_agent_async()

  runner = Runner(
      app_name='mcp_maps_app',
      agent=root_agent,
      artifact_service=artifacts_service, # Optional
      session_service=session_service,
  )

  print("Running agent...")
  events_async = runner.run_async(
      session_id=session.id, user_id=session.user_id, new_message=content
  )

  async for event in events_async:
    print(f"Event received: {event}")

  print("Closing MCP server connection...")
  await exit_stack.aclose()
  print("Cleanup complete.")

if __name__ == '__main__':
  try:
    asyncio.run(async_main())
  except Exception as e:
      print(f"An error occurred: {e}")

```

#### Step 3: Observe the Result[¶](#step-3-observe-the-result "Permanent link")

Run the script from the adk\_agent\_samples directory (ensure your virtual environment is active):

```
cd ./adk_agent_samples
python3 ./mcp_agent/agent.py

```

A successful run will show events indicating the agent called the relevant Google Maps tool (likely related to directions or routes) and a final response containing the directions. An example is shown below.

```
User Query: 'What is the route from 1600 Amphitheatre Pkwy to 1165 Borregas Ave'
Attempting to connect to MCP Google Maps server...
# --> npx process starts...
MCP Toolset created successfully.
Fetched [N] tools from MCP server.
Running agent...
Event received: content=Content(parts=[Part(..., function_call=FunctionCall(name='get_directions', ...))], role='model') ...
Event received: content=Content(parts=[Part(..., function_response=FunctionResponse(name='get_directions', ...))], role='user') ...
Event received: content=Content(parts=[Part(..., text='Head north toward Amphitheatre Pkwy...')], role='model') ...
Closing MCP server connection...
Cleanup complete.

```

## 2. **Building an MCP server with ADK tools (MCP server exposing ADK)**[¶](#2-building-an-mcp-server-with-adk-tools-mcp-server-exposing-adk "Permanent link")

This pattern allows you to wrap ADK's tools and make them available to any standard MCP client application. The example in this section exposes the load\_web\_page ADK tool through the MCP server.

### Summary of steps[¶](#summary-of-steps "Permanent link")

You will create a standard Python MCP server application using the model-context-protocol library. Within this server, you will:

1. Instantiate the ADK tool(s) you want to expose (e.g., FunctionTool(load\_web\_page)).
2. Implement the MCP server's @app.list\_tools handler to advertise the ADK tool(s), converting the ADK tool definition to the MCP schema using adk\_to\_mcp\_tool\_type.
3. Implement the MCP server's @app.call\_tool handler to receive requests from MCP clients, identify if the request targets your wrapped ADK tool, execute the ADK tool's .run\_async() method, and format the result into an MCP-compliant response (e.g., types.TextContent).

### Prerequisites[¶](#prerequisites_1 "Permanent link")

Install the MCP server library in the same environment as ADK:

```
pip install mcp

```

### Step 1: Create the MCP Server Script[¶](#step-1-create-the-mcp-server-script "Permanent link")

Create a new Python file, e.g., adk\_mcp\_server.py.

### Step 2: Implement the Server Logic[¶](#step-2-implement-the-server-logic "Permanent link")

Add the following code, which sets up an MCP server exposing the ADK load\_web\_page tool.

```
# adk_mcp_server.py
import asyncio
import json
from dotenv import load_dotenv

# MCP Server Imports
from mcp import types as mcp_types # Use alias to avoid conflict with genai.types
from mcp.server.lowlevel import Server, NotificationOptions
from mcp.server.models import InitializationOptions
import mcp.server.stdio

# ADK Tool Imports
from google.adk.tools.function_tool import FunctionTool
from google.adk.tools.load_web_page import load_web_page # Example ADK tool
# ADK <-> MCP Conversion Utility
from google.adk.tools.mcp_tool.conversion_utils import adk_to_mcp_tool_type

# --- Load Environment Variables (If ADK tools need them) ---
load_dotenv()

# --- Prepare the ADK Tool ---
# Instantiate the ADK tool you want to expose
print("Initializing ADK load_web_page tool...")
adk_web_tool = FunctionTool(load_web_page)
print(f"ADK tool '{adk_web_tool.name}' initialized.")
# --- End ADK Tool Prep ---

# --- MCP Server Setup ---
print("Creating MCP Server instance...")
# Create a named MCP Server instance
app = Server("adk-web-tool-mcp-server")

# Implement the MCP server's @app.list_tools handler
@app.list_tools()
async def list_tools() -> list[mcp_types.Tool]:
  """MCP handler to list available tools."""
  print("MCP Server: Received list_tools request.")
  # Convert the ADK tool's definition to MCP format
  mcp_tool_schema = adk_to_mcp_tool_type(adk_web_tool)
  print(f"MCP Server: Advertising tool: {mcp_tool_schema.name}")
  return [mcp_tool_schema]

# Implement the MCP server's @app.call_tool handler
@app.call_tool()
async def call_tool(
    name: str, arguments: dict
) -> list[mcp_types.TextContent | mcp_types.ImageContent | mcp_types.EmbeddedResource]:
  """MCP handler to execute a tool call."""
  print(f"MCP Server: Received call_tool request for '{name}' with args: {arguments}")

  # Check if the requested tool name matches our wrapped ADK tool
  if name == adk_web_tool.name:
    try:
      # Execute the ADK tool's run_async method
      # Note: tool_context is None as we are not within a full ADK Runner invocation
      adk_response = await adk_web_tool.run_async(
          args=arguments,
          tool_context=None, # No ADK context available here
      )
      print(f"MCP Server: ADK tool '{name}' executed successfully.")
      # Format the ADK tool's response (often a dict) into MCP format.
      # Here, we serialize the response dictionary as a JSON string within TextContent.
      # Adjust formatting based on the specific ADK tool's output and client needs.
      response_text = json.dumps(adk_response, indent=2)
      return [mcp_types.TextContent(type="text", text=response_text)]

    except Exception as e:
      print(f"MCP Server: Error executing ADK tool '{name}': {e}")
      # Return an error message in MCP format
      # Creating a proper MCP error response might be more robust
      error_text = json.dumps({"error": f"Failed to execute tool '{name}': {str(e)}"})
      return [mcp_types.TextContent(type="text", text=error_text)]
  else:
      # Handle calls to unknown tools
      print(f"MCP Server: Tool '{name}' not found.")
      error_text = json.dumps({"error": f"Tool '{name}' not implemented."})
      # Returning error as TextContent for simplicity
      return [mcp_types.TextContent(type="text", text=error_text)]

# --- MCP Server Runner ---
async def run_server():
  """Runs the MCP server over standard input/output."""
  # Use the stdio_server context manager from the MCP library
  async with mcp.server.stdio.stdio_server() as (read_stream, write_stream):
    print("MCP Server starting handshake...")
    await app.run(
        read_stream,
        write_stream,
        InitializationOptions(
            server_name=app.name, # Use the server name defined above
            server_version="0.1.0",
            capabilities=app.get_capabilities(
                # Define server capabilities - consult MCP docs for options
                notification_options=NotificationOptions(),
                experimental_capabilities={},
            ),
        ),
    )
    print("MCP Server run loop finished.")

if __name__ == "__main__":
  print("Launching MCP Server exposing ADK tools...")
  try:
    asyncio.run(run_server())
  except KeyboardInterrupt:
    print("\nMCP Server stopped by user.")
  except Exception as e:
    print(f"MCP Server encountered an error: {e}")
  finally:
    print("MCP Server process exiting.")
# --- End MCP Server ---

```

### Step 3: Test your MCP Server with ADK[¶](#step-3-test-your-mcp-server-with-adk "Permanent link")

Follow the same instructions in “Example 1: File System MCP Server” and create a MCP client. This time use your MCP Server file created above as input command:

```
# ./adk_agent_samples/mcp_agent/agent.py

# ...

async def get_tools_async():
  """Gets tools from the File System MCP Server."""
  print("Attempting to connect to MCP Filesystem server...")
  tools, exit_stack = await MCPToolset.from_server(
      # Use StdioServerParameters for local process communication
      connection_params=StdioServerParameters(
          command='python3', # Command to run the server
          args=[
                "/absolute/path/to/adk_mcp_server.py"],
      )
  )

```

Execute the agent script from your terminal similar to above (ensure necessary libraries like model-context-protocol and google-adk are installed in your environment):

```
cd ./adk_agent_samples
python3 ./mcp_agent/agent.py

```

The script will print startup messages and then wait for an MCP client to connect via its standard input/output to your MCP Server in adk\_mcp\_server.py. Any MCP-compliant client (like Claude Desktop, or a custom client using the MCP libraries) can now connect to this process, discover the load\_web\_page tool, and invoke it. The server will print log messages indicating received requests and ADK tool execution. Refer to the [documentation](https://modelcontextprotocol.io/quickstart/server#core-mcp-concepts), to try it out with Claude Desktop.

## Key considerations[¶](#key-considerations "Permanent link")

When working with MCP and ADK, keep these points in mind:

* **Protocol vs. Library:** MCP is a protocol specification, defining communication rules. ADK is a Python library/framework for building agents. MCPToolset bridges these by implementing the client side of the MCP protocol within the ADK framework. Conversely, building an MCP server in Python requires using the model-context-protocol library.
* **ADK Tools vs. MCP Tools:**

  + ADK Tools (BaseTool, FunctionTool, AgentTool, etc.) are Python objects designed for direct use within the ADK's LlmAgent and Runner.
  + MCP Tools are capabilities exposed by an MCP Server according to the protocol's schema. MCPToolset makes these look like ADK tools to an LlmAgent.
  + Langchain/CrewAI Tools are specific implementations within those libraries, often simple functions or classes, lacking the server/protocol structure of MCP. ADK offers wrappers (LangchainTool, CrewaiTool) for some interoperability.
* **Asynchronous nature:** Both ADK and the MCP Python library are heavily based on the asyncio Python library. Tool implementations and server handlers should generally be async functions.
* **Stateful sessions (MCP):** MCP establishes stateful, persistent connections between a client and server instance. This differs from typical stateless REST APIs.

  + **Deployment:** This statefulness can pose challenges for scaling and deployment, especially for remote servers handling many users. The original MCP design often assumed client and server were co-located. Managing these persistent connections requires careful infrastructure considerations (e.g., load balancing, session affinity).
  + **ADK MCPToolset:** Manages this connection lifecycle. The exit\_stack pattern shown in the examples is crucial for ensuring the connection (and potentially the server process) is properly terminated when the ADK agent finishes.

## Further Resources[¶](#further-resources "Permanent link")

* [Model Context Protocol Documentation](https://modelcontextprotocol.io/)
* [MCP Specification](https://modelcontextprotocol.io/specification/)
* [MCP Python SDK & Examples](https://github.com/modelcontextprotocol/)

Back to top

---


## OpenAPI tools - Agent Development Kit

URL: https://google.github.io/adk-docs/tools/openapi-tools/

# OpenAPI Integration[¶](#openapi-integration "Permanent link")

## Integrating REST APIs with OpenAPI[¶](#integrating-rest-apis-with-openapi "Permanent link")

ADK simplifies interacting with external REST APIs by automatically generating callable tools directly from an [OpenAPI Specification (v3.x)](https://swagger.io/specification/). This eliminates the need to manually define individual function tools for each API endpoint.

Core Benefit

Use `OpenAPIToolset` to instantly create agent tools (`RestApiTool`) from your existing API documentation (OpenAPI spec), enabling agents to seamlessly call your web services.

## Key Components[¶](#key-components "Permanent link")

* **`OpenAPIToolset`**: This is the primary class you'll use. You initialize it with your OpenAPI specification, and it handles the parsing and generation of tools.
* **`RestApiTool`**: This class represents a single, callable API operation (like `GET /pets/{petId}` or `POST /pets`). `OpenAPIToolset` creates one `RestApiTool` instance for each operation defined in your spec.

## How it Works[¶](#how-it-works "Permanent link")

The process involves these main steps when you use `OpenAPIToolset`:

1. **Initialization & Parsing**:

   * You provide the OpenAPI specification to `OpenAPIToolset` either as a Python dictionary, a JSON string, or a YAML string.
   * The toolset internally parses the spec, resolving any internal references (`$ref`) to understand the complete API structure.
2. **Operation Discovery**:

   * It identifies all valid API operations (e.g., `GET`, `POST`, `PUT`, `DELETE`) defined within the `paths` object of your specification.
3. **Tool Generation**:

   * For each discovered operation, `OpenAPIToolset` automatically creates a corresponding `RestApiTool` instance.
   * **Tool Name**: Derived from the `operationId` in the spec (converted to `snake_case`, max 60 chars). If `operationId` is missing, a name is generated from the method and path.
   * **Tool Description**: Uses the `summary` or `description` from the operation for the LLM.
   * **API Details**: Stores the required HTTP method, path, server base URL, parameters (path, query, header, cookie), and request body schema internally.
4. **`RestApiTool` Functionality**: Each generated `RestApiTool`:

   * **Schema Generation**: Dynamically creates a `FunctionDeclaration` based on the operation's parameters and request body. This schema tells the LLM how to call the tool (what arguments are expected).
   * **Execution**: When called by the LLM, it constructs the correct HTTP request (URL, headers, query params, body) using the arguments provided by the LLM and the details from the OpenAPI spec. It handles authentication (if configured) and executes the API call using the `requests` library.
   * **Response Handling**: Returns the API response (typically JSON) back to the agent flow.
5. **Authentication**: You can configure global authentication (like API keys or OAuth - see [Authentication](../authentication/) for details) when initializing `OpenAPIToolset`. This authentication configuration is automatically applied to all generated `RestApiTool` instances.

## Usage Workflow[¶](#usage-workflow "Permanent link")

Follow these steps to integrate an OpenAPI spec into your agent:

1. **Obtain Spec**: Get your OpenAPI specification document (e.g., load from a `.json` or `.yaml` file, fetch from a URL).
2. **Instantiate Toolset**: Create an `OpenAPIToolset` instance, passing the spec content and type (`spec_str`/`spec_dict`, `spec_str_type`). Provide authentication details (`auth_scheme`, `auth_credential`) if required by the API.

   ```
   from google.adk.tools.openapi_tool.openapi_spec_parser.openapi_toolset import OpenAPIToolset

   # Example with a JSON string
   openapi_spec_json = '...' # Your OpenAPI JSON string
   toolset = OpenAPIToolset(spec_str=openapi_spec_json, spec_str_type="json")

   # Example with a dictionary
   # openapi_spec_dict = {...} # Your OpenAPI spec as a dict
   # toolset = OpenAPIToolset(spec_dict=openapi_spec_dict)

   ```
3. **Retrieve Tools**: Get the list of generated `RestApiTool` instances from the toolset.

   ```
   api_tools = toolset.get_tools()
   # Or get a specific tool by its generated name (snake_case operationId)
   # specific_tool = toolset.get_tool("list_pets")

   ```
4. **Add to Agent**: Include the retrieved tools in your `LlmAgent`'s `tools` list.

   ```
   from google.adk.agents import LlmAgent

   my_agent = LlmAgent(
       name="api_interacting_agent",
       model="gemini-2.0-flash", # Or your preferred model
       tools=api_tools, # Pass the list of generated tools
       # ... other agent config ...
   )

   ```
5. **Instruct Agent**: Update your agent's instructions to inform it about the new API capabilities and the names of the tools it can use (e.g., `list_pets`, `create_pet`). The tool descriptions generated from the spec will also help the LLM.
6. **Run Agent**: Execute your agent using the `Runner`. When the LLM determines it needs to call one of the APIs, it will generate a function call targeting the appropriate `RestApiTool`, which will then handle the HTTP request automatically.

## Example[¶](#example "Permanent link")

This example demonstrates generating tools from a simple Pet Store OpenAPI spec (using `httpbin.org` for mock responses) and interacting with them via an agent.

Code: Pet Store API

openapi\_example.py

```
import asyncio
import uuid # For unique session IDs
from google.adk.agents import LlmAgent
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService
from google.genai import types

# --- OpenAPI Tool Imports ---
from google.adk.tools.openapi_tool.openapi_spec_parser.openapi_toolset import OpenAPIToolset

# --- Constants ---
APP_NAME_OPENAPI = "openapi_petstore_app"
USER_ID_OPENAPI = "user_openapi_1"
SESSION_ID_OPENAPI = f"session_openapi_{uuid.uuid4()}" # Unique session ID
AGENT_NAME_OPENAPI = "petstore_manager_agent"
GEMINI_MODEL = "gemini-2.0-flash"

# --- Sample OpenAPI Specification (JSON String) ---
# A basic Pet Store API example using httpbin.org as a mock server
openapi_spec_string = """
{
  "openapi": "3.0.0",
  "info": {
    "title": "Simple Pet Store API (Mock)",
    "version": "1.0.1",
    "description": "An API to manage pets in a store, using httpbin for responses."
  },
  "servers": [
    {
      "url": "https://httpbin.org",
      "description": "Mock server (httpbin.org)"
    }
  ],
  "paths": {
    "/get": {
      "get": {
        "summary": "List all pets (Simulated)",
        "operationId": "listPets",
        "description": "Simulates returning a list of pets. Uses httpbin's /get endpoint which echoes query parameters.",
        "parameters": [
          {
            "name": "limit",
            "in": "query",
            "description": "Maximum number of pets to return",
            "required": false,
            "schema": { "type": "integer", "format": "int32" }
          },
          {
             "name": "status",
             "in": "query",
             "description": "Filter pets by status",
             "required": false,
             "schema": { "type": "string", "enum": ["available", "pending", "sold"] }
          }
        ],
        "responses": {
          "200": {
            "description": "A list of pets (echoed query params).",
            "content": { "application/json": { "schema": { "type": "object" } } }
          }
        }
      }
    },
    "/post": {
      "post": {
        "summary": "Create a pet (Simulated)",
        "operationId": "createPet",
        "description": "Simulates adding a new pet. Uses httpbin's /post endpoint which echoes the request body.",
        "requestBody": {
          "description": "Pet object to add",
          "required": true,
          "content": {
            "application/json": {
              "schema": {
                "type": "object",
                "required": ["name"],
                "properties": {
                  "name": {"type": "string", "description": "Name of the pet"},
                  "tag": {"type": "string", "description": "Optional tag for the pet"}
                }
              }
            }
          }
        },
        "responses": {
          "201": {
            "description": "Pet created successfully (echoed request body).",
            "content": { "application/json": { "schema": { "type": "object" } } }
          }
        }
      }
    },
    "/get?petId={petId}": {
      "get": {
        "summary": "Info for a specific pet (Simulated)",
        "operationId": "showPetById",
        "description": "Simulates returning info for a pet ID. Uses httpbin's /get endpoint.",
        "parameters": [
          {
            "name": "petId",
            "in": "path",
            "description": "This is actually passed as a query param to httpbin /get",
            "required": true,
            "schema": { "type": "integer", "format": "int64" }
          }
        ],
        "responses": {
          "200": {
            "description": "Information about the pet (echoed query params)",
            "content": { "application/json": { "schema": { "type": "object" } } }
          },
          "404": { "description": "Pet not found (simulated)" }
        }
      }
    }
  }
}
"""

# --- Create OpenAPIToolset ---
generated_tools_list = []
try:
    # Instantiate the toolset with the spec string
    petstore_toolset = OpenAPIToolset(
        spec_str=openapi_spec_string,
        spec_str_type="json"
        # No authentication needed for httpbin.org
    )
    # Get all tools generated from the spec
    generated_tools_list = petstore_toolset.get_tools()
    print(f"Generated {len(generated_tools_list)} tools from OpenAPI spec:")
    for tool in generated_tools_list:
        # Tool names are snake_case versions of operationId
        print(f"- Tool Name: '{tool.name}', Description: {tool.description[:60]}...")

except ValueError as ve:
    print(f"Validation Error creating OpenAPIToolset: {ve}")
    # Handle error appropriately, maybe exit or skip agent creation
except Exception as e:
    print(f"Unexpected Error creating OpenAPIToolset: {e}")
    # Handle error appropriately

# --- Agent Definition ---
openapi_agent = LlmAgent(
    name=AGENT_NAME_OPENAPI,
    model=GEMINI_MODEL,
    tools=generated_tools_list, # Pass the list of RestApiTool objects
    instruction=f"""You are a Pet Store assistant managing pets via an API.
    Use the available tools to fulfill user requests.
    Available tools: {', '.join([t.name for t in generated_tools_list])}.
    When creating a pet, confirm the details echoed back by the API.
    When listing pets, mention any filters used (like limit or status).
    When showing a pet by ID, state the ID you requested.
    """,
    description="Manages a Pet Store using tools generated from an OpenAPI spec."
)

# --- Session and Runner Setup ---
session_service_openapi = InMemorySessionService()
runner_openapi = Runner(
    agent=openapi_agent, app_name=APP_NAME_OPENAPI, session_service=session_service_openapi
)
session_openapi = session_service_openapi.create_session(
    app_name=APP_NAME_OPENAPI, user_id=USER_ID_OPENAPI, session_id=SESSION_ID_OPENAPI
)

# --- Agent Interaction Function ---
async def call_openapi_agent_async(query):
    print("\n--- Running OpenAPI Pet Store Agent ---")
    print(f"Query: {query}")
    if not generated_tools_list:
        print("Skipping execution: No tools were generated.")
        print("-" * 30)
        return

    content = types.Content(role='user', parts=[types.Part(text=query)])
    final_response_text = "Agent did not provide a final text response."
    try:
        async for event in runner_openapi.run_async(
            user_id=USER_ID_OPENAPI, session_id=SESSION_ID_OPENAPI, new_message=content
            ):
            # Optional: Detailed event logging for debugging
            # print(f"  DEBUG Event: Author={event.author}, Type={'Final' if event.is_final_response() else 'Intermediate'}, Content={str(event.content)[:100]}...")
            if event.get_function_calls():
                call = event.get_function_calls()[0]
                print(f"  Agent Action: Called function '{call.name}' with args {call.args}")
            elif event.get_function_responses():
                response = event.get_function_responses()[0]
                print(f"  Agent Action: Received response for '{response.name}'")
                # print(f"  Tool Response Snippet: {str(response.response)[:200]}...") # Uncomment for response details
            elif event.is_final_response() and event.content and event.content.parts:
                # Capture the last final text response
                final_response_text = event.content.parts[0].text.strip()

        print(f"Agent Final Response: {final_response_text}")

    except Exception as e:
        print(f"An error occurred during agent run: {e}")
        import traceback
        traceback.print_exc() # Print full traceback for errors
    print("-" * 30)

# --- Run Examples ---
async def run_openapi_example():
    # Trigger listPets
    await call_openapi_agent_async("Show me the pets available.")
    # Trigger createPet
    await call_openapi_agent_async("Please add a new dog named 'Dukey'.")
    # Trigger showPetById
    await call_openapi_agent_async("Get info for pet with ID 123.")

# --- Execute ---
if __name__ == "__main__":
    print("Executing OpenAPI example...")
    # Use asyncio.run() for top-level execution
    try:
        asyncio.run(run_openapi_example())
    except RuntimeError as e:
        if "cannot be called from a running event loop" in str(e):
            print("Info: Cannot run asyncio.run from a running event loop (e.g., Jupyter/Colab).")
            # If in Jupyter/Colab, you might need to run like this:
            # await run_openapi_example()
        else:
            raise e
    print("OpenAPI example finished.")

```

Back to top

---


## Authentication - Agent Development Kit

URL: https://google.github.io/adk-docs/tools/authentication/

# Authenticating with Tools[¶](#authenticating-with-tools "Permanent link")

## Core Concepts[¶](#core-concepts "Permanent link")

Many tools need to access protected resources (like user data in Google Calendar, Salesforce records, etc.) and require authentication. ADK provides a system to handle various authentication methods securely.

The key components involved are:

1. **`AuthScheme`**: Defines *how* an API expects authentication credentials (e.g., as an API Key in a header, an OAuth 2.0 Bearer token). ADK supports the same types of authentication schemes as OpenAPI 3.0. To know more about what each type of credential is, refer to [OpenAPI doc: Authentication](https://swagger.io/docs/specification/v3_0/authentication/). ADK uses specific classes like `APIKey`, `HTTPBearer`, `OAuth2`, `OpenIdConnectWithConfig`.
2. **`AuthCredential`**: Holds the *initial* information needed to *start* the authentication process (e.g., your application's OAuth Client ID/Secret, an API key value). It includes an `auth_type` (like `API_KEY`, `OAUTH2`, `SERVICE_ACCOUNT`) specifying the credential type.

The general flow involves providing these details when configuring a tool. ADK then attempts to automatically exchange the initial credential for a usable one (like an access token) before the tool makes an API call. For flows requiring user interaction (like OAuth consent), a specific interactive process involving the Agent Client application is triggered.

## Supported Initial Credential Types[¶](#supported-initial-credential-types "Permanent link")

* **API\_KEY:** For simple key/value authentication. Usually requires no exchange.
* **HTTP:** Can represent Basic Auth (not recommended/supported for exchange) or already obtained Bearer tokens. If it's a Bearer token, no exchange is needed.
* **OAUTH2:** For standard OAuth 2.0 flows. Requires configuration (client ID, secret, scopes) and often triggers the interactive flow for user consent.
* **OPEN\_ID\_CONNECT:** For authentication based on OpenID Connect. Similar to OAuth2, often requires configuration and user interaction.
* **SERVICE\_ACCOUNT:** For Google Cloud Service Account credentials (JSON key or Application Default Credentials). Typically exchanged for a Bearer token.

## Configuring Authentication on Tools[¶](#configuring-authentication-on-tools "Permanent link")

You set up authentication when defining your tool:

* **RestApiTool / OpenAPIToolset**: Pass `auth_scheme` and `auth_credential` during initialization
* **GoogleApiToolSet Tools**: ADK has built-in 1st party tools like Google Calendar, BigQuery etc,. Use the toolset's specific method.
* **APIHubToolset / ApplicationIntegrationToolset**: Pass `auth_scheme` and `auth_credential`during initialization, if the API managed in API Hub / provided by Application Integration requires authentication.

WARNING

Storing sensitive credentials like access tokens and especially refresh tokens directly in the session state might pose security risks depending on your session storage backend (`SessionService`) and overall application security posture.

* **`InMemorySessionService`:** Suitable for testing and development, but data is lost when the process ends. Less risk as it's transient.
* **Database/Persistent Storage:** **Strongly consider encrypting** the token data before storing it in the database using a robust encryption library (like `cryptography`) and managing encryption keys securely (e.g., using a key management service).
* **Secure Secret Stores:** For production environments, storing sensitive credentials in a dedicated secret manager (like Google Cloud Secret Manager or HashiCorp Vault) is the **most recommended approach**. Your tool could potentially store only short-lived access tokens or secure references (not the refresh token itself) in the session state, fetching the necessary secrets from the secure store when needed.

---

## Journey 1: Building Agentic Applications with Authenticated Tools[¶](#journey-1-building-agentic-applications-with-authenticated-tools "Permanent link")

This section focuses on using pre-existing tools (like those from `RestApiTool/ OpenAPIToolset`, `APIHubToolset`, `GoogleApiToolSet`, or custom `FunctionTools`) that require authentication within your agentic application. Your main responsibility is configuring the tools and handling the client-side part of interactive authentication flows (if required by the tool).

![Authentication](../../assets/auth_part1.svg)

### 1. Configuring Tools with Authentication[¶](#1-configuring-tools-with-authentication "Permanent link")

When adding an authenticated tool to your agent, you need to provide its required `AuthScheme` and your application's initial `AuthCredential`.

**A. Using OpenAPI-based Toolsets (`OpenAPIToolset`, `APIHubToolset`, etc.)**

Pass the scheme and credential during toolset initialization. The toolset applies them to all generated tools. Here are few ways to create tools with authentication in ADK.

API KeyOAuth2Service AccountOpenID connect

Create a tool requiring an API Key.

```
from google.adk.tools.openapi_tool.auth.auth_helpers import token_to_scheme_credential
from google.adk.tools.apihub_tool.apihub_toolset import APIHubToolset
auth_scheme, auth_credential = token_to_scheme_credential(
   "apikey", "query", "apikey", YOUR_API_KEY_STRING
)
sample_api_toolset = APIHubToolset(
   name="sample-api-requiring-api-key",
   description="A tool using an API protected by API Key",
   apihub_resource_name="...",
   auth_scheme=auth_scheme,
   auth_credential=auth_credential,
)

```

Create a tool requiring OAuth2.

```
from google.adk.tools.openapi_tool.openapi_spec_parser.openapi_toolset import OpenAPIToolset
from fastapi.openapi.models import OAuth2
from fastapi.openapi.models import OAuthFlowAuthorizationCode
from fastapi.openapi.models import OAuthFlows
from google.adk.auth import AuthCredential
from google.adk.auth import AuthCredentialTypes
from google.adk.auth import OAuth2Auth

auth_scheme = OAuth2(
   flows=OAuthFlows(
      authorizationCode=OAuthFlowAuthorizationCode(
            authorizationUrl="https://accounts.google.com/o/oauth2/auth",
            tokenUrl="https://oauth2.googleapis.com/token",
            scopes={
               "https://www.googleapis.com/auth/calendar": "calendar scope"
            },
      )
   )
)
auth_credential = AuthCredential(
   auth_type=AuthCredentialTypes.OAUTH2,
   oauth2=OAuth2Auth(
      client_id=YOUR_OAUTH_CLIENT_ID, 
      client_secret=YOUR_OAUTH_CLIENT_SECRET
   ),
)

calendar_api_toolset = OpenAPIToolset(
   spec_str=google_calendar_openapi_spec_str, # Fill this with an openapi spec
   spec_str_type='yaml',
   auth_scheme=auth_scheme,
   auth_credential=auth_credential,
)

```

Create a tool requiring Service Account.

```
from google.adk.tools.openapi_tool.auth.auth_helpers import service_account_dict_to_scheme_credential
from google.adk.tools.openapi_tool.openapi_spec_parser.openapi_toolset import OpenAPIToolset

service_account_cred = json.loads(service_account_json_str)auth_scheme, auth_credential = service_account_dict_to_scheme_credential(
   config=service_account_cred,
   scopes=["https://www.googleapis.com/auth/cloud-platform"],
)
sample_toolset = OpenAPIToolset(
   spec_str=sa_openapi_spec_str, # Fill this with an openapi spec
   spec_str_type='json',
   auth_scheme=auth_scheme,
   auth_credential=auth_credential,
)

```

Create a tool requiring OpenID connect.

```
from google.adk.auth.auth_schemes import OpenIdConnectWithConfig
from google.adk.auth.auth_credential import AuthCredential, AuthCredentialTypes, OAuth2Auth
from google.adk.tools.openapi_tool.openapi_spec_parser.openapi_toolset import OpenAPIToolset

auth_scheme = OpenIdConnectWithConfig(
   authorization_endpoint=OAUTH2_AUTH_ENDPOINT_URL,
   token_endpoint=OAUTH2_TOKEN_ENDPOINT_URL,
   scopes=['openid', 'YOUR_OAUTH_SCOPES"]
)
auth_credential = AuthCredential(
auth_type=AuthCredentialTypes.OPEN_ID_CONNECT,
oauth2=OAuth2Auth(
   client_id="...",
   client_secret="...",
)
)

userinfo_toolset = OpenAPIToolset(
   spec_str=content, # Fill in an actual spec
   spec_str_type='yaml',
   auth_scheme=auth_scheme,
   auth_credential=auth_credential,
)

```

**B. Using Google API Toolsets (e.g., `calendar_tool_set`)**

These toolsets often have dedicated configuration methods.

Tip: For how to create a Google OAuth Client ID & Secret, see this guide: [Get your Google API Client ID](https://developers.google.com/identity/gsi/web/guides/get-google-api-clientid#get_your_google_api_client_id)

```
# Example: Configuring Google Calendar Tools
from google.adk.tools.google_api_tool import calendar_tool_set

client_id = "YOUR_GOOGLE_OAUTH_CLIENT_ID.apps.googleusercontent.com"
client_secret = "YOUR_GOOGLE_OAUTH_CLIENT_SECRET"

calendar_tools = calendar_tool_set.get_tools()
for tool in calendar_tools:
    # Use the specific configure method for this tool type
    tool.configure_auth(client_id=client_id, client_secret=client_secret)

# agent = LlmAgent(..., tools=calendar_tools)

```

### 2. Handling the Interactive OAuth/OIDC Flow (Client-Side)[¶](#2-handling-the-interactive-oauthoidc-flow-client-side "Permanent link")

If a tool requires user login/consent (typically OAuth 2.0 or OIDC), the ADK framework pauses execution and signals your **Agent Client** application (the code calling `runner.run_async`, like your UI backend, CLI app, or Spark job) to handle the user interaction.

Here's the step-by-step process for your client application:

**Step 1: Run Agent & Detect Auth Request**

* Initiate the agent interaction using `runner.run_async`.
* Iterate through the yielded events.
* Look for a specific event where the agent calls the special function `adk_request_credential`. This event signals that user interaction is needed. Use helper functions to identify this event and extract necessary information.

```
# runner = Runner(...)
# session = session_service.create_session(...)
# content = types.Content(...) # User's initial query

print("\nRunning agent...")
events_async = runner.run_async(
    session_id=session.id, user_id='user', new_message=content
)

auth_request_event_id, auth_config = None, None

async for event in events_async:
    # Use helper to check for the specific auth request event
    if is_pending_auth_event(event):
        print("--> Authentication required by agent.")
        # Store the ID needed to respond later
        auth_request_event_id = get_function_call_id(event)
        # Get the AuthConfig containing the auth_uri etc.
        auth_config = get_function_call_auth_config(event)
        break # Stop processing events for now, need user interaction

if not auth_request_event_id:
    print("\nAuth not required or agent finished.")
    # return # Or handle final response if received

```

*Helper functions `helpers.py`:*

```
from google.adk.events import Event
from google.adk.auth import AuthConfig # Import necessary type

def is_pending_auth_event(event: Event) -> bool:
  # Checks if the event is the special auth request function call
  return (
      event.content and event.content.parts and event.content.parts[0]
      and event.content.parts[0].function_call
      and event.content.parts[0].function_call.name == 'adk_request_credential'
      # Check if it's marked as long running (optional but good practice)
      and event.long_running_tool_ids
      and event.content.parts[0].function_call.id in event.long_running_tool_ids
  )

def get_function_call_id(event: Event) -> str:
  # Extracts the ID of the function call (works for any call, including auth)
  if ( event and event.content and event.content.parts and event.content.parts[0]
      and event.content.parts[0].function_call and event.content.parts[0].function_call.id ):
    return event.content.parts[0].function_call.id
  raise ValueError(f'Cannot get function call id from event {event}')

def get_function_call_auth_config(event: Event) -> AuthConfig:
    # Extracts the AuthConfig object from the arguments of the auth request event
    auth_config_dict = None
    try:
        auth_config_dict = event.content.parts[0].function_call.args.get('auth_config')
        if auth_config_dict and isinstance(auth_config_dict, dict):
            # Reconstruct the AuthConfig object
            return AuthConfig.model_validate(auth_config_dict)
        else:
            raise ValueError("auth_config missing or not a dict in event args")
    except (AttributeError, IndexError, KeyError, TypeError, ValueError) as e:
        raise ValueError(f'Cannot get auth config from event {event}') from e

```

**Step 2: Redirect User for Authorization**

* Get the authorization URL (`auth_uri`) from the `auth_config` extracted in the previous step.
* **Crucially, append your application's** redirect\_uri as a query parameter to this `auth_uri`. This `redirect_uri` must be pre-registered with your OAuth provider (e.g., [Google Cloud Console](https://developers.google.com/identity/protocols/oauth2/web-server#creatingcred), [Okta admin panel](https://developer.okta.com/docs/guides/sign-into-web-app-redirect/spring-boot/main/#create-an-app-integration-in-the-admin-console)).
* Direct the user to this complete URL (e.g., open it in their browser).

```
# (Continuing after detecting auth needed)

if auth_request_event_id and auth_config:
    # Get the base authorization URL from the AuthConfig
    base_auth_uri = auth_config.exchanged_auth_credential.oauth2.auth_uri

    if base_auth_uri:
        redirect_uri = 'http://localhost:8000/callback' # MUST match your OAuth client config
        # Append redirect_uri (use urlencode in production)
        auth_request_uri = base_auth_uri + f'&redirect_uri={redirect_uri}'

        print("\n--- User Action Required ---")
        print(f'1. Please open this URL in your browser:\n   {auth_request_uri}\n')
        print(f'2. Log in and grant the requested permissions.')
        print(f'3. After authorization, you will be redirected to: {redirect_uri}')
        print(f'   Copy the FULL URL from your browser\'s address bar (it includes a `code=...`).')
        # Next step: Get this callback URL from the user (or your web server handler)
    else:
         print("ERROR: Auth URI not found in auth_config.")
         # Handle error

```

![Authentication](../../assets/auth_part2.svg)

**Step 3. Handle the Redirect Callback (Client):**

* Your application must have a mechanism (e.g., a web server route at the `redirect_uri`) to receive the user after they authorize the application with the provider.
* The provider redirects the user to your `redirect_uri` and appends an `authorization_code` (and potentially `state`, `scope`) as query parameters to the URL.
* Capture the **full callback URL** from this incoming request.
* (This step happens outside the main agent execution loop, in your web server or equivalent callback handler.)

**Step 4. Send Authentication Result Back to ADK (Client):**

* Once you have the full callback URL (containing the authorization code), retrieve the `auth_request_event_id` and the `AuthConfig` object saved in Client Step 1.
* **Update the** Set the captured callback URL into the `exchanged_auth_credential.oauth2.auth_response_uri` field. Also ensure `exchanged_auth_credential.oauth2.redirect_uri` contains the redirect URI you used.
* **Construct a** Create a `types.Content` object containing a `types.Part` with a `types.FunctionResponse`.
  + Set `name` to `"adk_request_credential"`. (Note: This is a special name for ADK to proceed with authentication. Do not use other names.)
  + Set `id` to the `auth_request_event_id` you saved.
  + Set `response` to the *serialized* (e.g., `.model_dump()`) updated `AuthConfig` object.
* Call `runner.run_async` **again** for the same session, passing this `FunctionResponse` content as the `new_message`.

```
# (Continuing after user interaction)

    # Simulate getting the callback URL (e.g., from user paste or web handler)
    auth_response_uri = await get_user_input(
        f'Paste the full callback URL here:\n> '
    )
    auth_response_uri = auth_response_uri.strip() # Clean input

    if not auth_response_uri:
        print("Callback URL not provided. Aborting.")
        return

    # Update the received AuthConfig with the callback details
    auth_config.exchanged_auth_credential.oauth2.auth_response_uri = auth_response_uri
    # Also include the redirect_uri used, as the token exchange might need it
    auth_config.exchanged_auth_credential.oauth2.redirect_uri = redirect_uri

    # Construct the FunctionResponse Content object
    auth_content = types.Content(
        role='user', # Role can be 'user' when sending a FunctionResponse
        parts=[
            types.Part(
                function_response=types.FunctionResponse(
                    id=auth_request_event_id,       # Link to the original request
                    name='adk_request_credential', # Special framework function name
                    response=auth_config.model_dump() # Send back the *updated* AuthConfig
                )
            )
        ],
    )

    # --- Resume Execution ---
    print("\nSubmitting authentication details back to the agent...")
    events_async_after_auth = runner.run_async(
        session_id=session.id,
        user_id='user',
        new_message=auth_content, # Send the FunctionResponse back
    )

    # --- Process Final Agent Output ---
    print("\n--- Agent Response after Authentication ---")
    async for event in events_async_after_auth:
        # Process events normally, expecting the tool call to succeed now
        print(event) # Print the full event for inspection

```

**Step 5: ADK Handles Token Exchange & Tool Retry and gets Tool result**

* ADK receives the `FunctionResponse` for `adk_request_credential`.
* It uses the information in the updated `AuthConfig` (including the callback URL containing the code) to perform the OAuth **token exchange** with the provider's token endpoint, obtaining the access token (and possibly refresh token).
* ADK internally makes these tokens available (often via `tool_context.get_auth_response()` or by updating session state).
* ADK **automatically retries** the original tool call (the one that initially failed due to missing auth).
* This time, the tool finds the valid tokens and successfully executes the authenticated API call.
* The agent receives the actual result from the tool and generates its final response to the user.

---

## Journey 2: Building Custom Tools (`FunctionTool`) Requiring Authentication[¶](#journey-2-building-custom-tools-functiontool-requiring-authentication "Permanent link")

This section focuses on implementing the authentication logic *inside* your custom Python function when creating a new ADK Tool. We will implement a `FunctionTool` as an example.

### Prerequisites[¶](#prerequisites "Permanent link")

Your function signature *must* include [`tool_context: ToolContext`](../#tool-context). ADK automatically injects this object, providing access to state and auth mechanisms.

```
from google.adk.tools import FunctionTool, ToolContext
from typing import Dict

def my_authenticated_tool_function(param1: str, ..., tool_context: ToolContext) -> dict:
    # ... your logic ...
    pass

my_tool = FunctionTool(func=my_authenticated_tool_function)

```

### Authentication Logic within the Tool Function[¶](#authentication-logic-within-the-tool-function "Permanent link")

Implement the following steps inside your function:

**Step 1: Check for Cached & Valid Credentials:**

Inside your tool function, first check if valid credentials (e.g., access/refresh tokens) are already stored from a previous run in this session. Credentials for the current sessions should be stored in `tool_context.invocation_context.session.state` (a dictionary of state) Check existence of existing credentials by checking `tool_context.invocation_context.session.state.get(credential_name, None)`.

```
# Inside your tool function
TOKEN_CACHE_KEY = "my_tool_tokens" # Choose a unique key
SCOPES = ["scope1", "scope2"] # Define required scopes

creds = None
cached_token_info = tool_context.state.get(TOKEN_CACHE_KEY)
if cached_token_info:
    try:
        creds = Credentials.from_authorized_user_info(cached_token_info, SCOPES)
        if not creds.valid and creds.expired and creds.refresh_token:
            creds.refresh(Request())
            tool_context.state[TOKEN_CACHE_KEY] = json.loads(creds.to_json()) # Update cache
        elif not creds.valid:
            creds = None # Invalid, needs re-auth
            tool_context.state.pop(TOKEN_CACHE_KEY, None)
    except Exception as e:
        print(f"Error loading/refreshing cached creds: {e}")
        creds = None
        tool_context.state.pop(TOKEN_CACHE_KEY, None)

if creds and creds.valid:
    # Skip to Step 5: Make Authenticated API Call
    pass
else:
    # Proceed to Step 2...
    pass

```

**Step 2: Check for Auth Response from Client**

* If Step 1 didn't yield valid credentials, check if the client just completed the interactive flow by calling `auth_response_config = tool_context.get_auth_response()`.
* This returns the updated `AuthConfig` object sent back by the client (containing the callback URL in `auth_response_uri`).

```
# Use auth_scheme and auth_credential configured in the tool.
# exchanged_credential: AuthCredential|None

exchanged_credential = tool_context.get_auth_response(AuthConfig(
  auth_scheme=auth_scheme,
  raw_auth_credential=auth_credential,
))
# If exchanged_credential is not None, then there is already an exchanged credetial from the auth response. Use it instea, and skip to step 5

```

**Step 3: Initiate Authentication Request**

If no valid credentials (Step 1.) and no auth response (Step 2.) are found, the tool needs to start the OAuth flow. Define the AuthScheme and initial AuthCredential and call `tool_context.request_credential()`. Return a status indicating authorization is needed.

```
# Use auth_scheme and auth_credential configured in the tool.

  tool_context.request_credential(AuthConfig(
    auth_scheme=auth_scheme,
    raw_auth_credential=auth_credential,
  ))
  return {'pending': true, 'message': 'Awaiting user authentication.'}

# By setting request_credential, ADK detects a pending authentication event. It pauses execution and ask end user to login.

```

**Step 4: Exchange Authorization Code for Tokens**

ADK automatically generates oauth authorization URL and presents it to your Agent Client application. Once a user completes the login flow following the authorization URL, ADK extracts the authentication callback url from Agent Client applications, automatically parses the auth code, and generates auth token. At the next Tool call, `tool_context.get_auth_response` in step 2 will contain a valid credential to use in subsequent API calls.

**Step 5: Cache Obtained Credentials**

After successfully obtaining the token from ADK (Step 2) or if the token is still valid (Step 1), **immediately store** the new `Credentials` object in `tool_context.state` (serialized, e.g., as JSON) using your cache key.

```
# Inside your tool function, after obtaining 'creds' (either refreshed or newly exchanged)
# Cache the new/refreshed tokens
tool_context.state[TOKEN_CACHE_KEY] = json.loads(creds.to_json())
print(f"DEBUG: Cached/updated tokens under key: {TOKEN_CACHE_KEY}")
# Proceed to Step 6 (Make API Call)

```

**Step 6: Make Authenticated API Call**

* Once you have a valid `Credentials` object (`creds` from Step 1 or Step 4), use it to make the actual call to the protected API using the appropriate client library (e.g., `googleapiclient`, `requests`). Pass the `credentials=creds` argument.
* Include error handling, especially for `HttpError` 401/403, which might mean the token expired or was revoked between calls. If you get such an error, consider clearing the cached token (`tool_context.state.pop(...)`) and potentially returning the `auth_required` status again to force re-authentication.

```
# Inside your tool function, using the valid 'creds' object
# Ensure creds is valid before proceeding
if not creds or not creds.valid:
   return {"status": "error", "error_message": "Cannot proceed without valid credentials."}

try:
   service = build("calendar", "v3", credentials=creds) # Example
   api_result = service.events().list(...).execute()
   # Proceed to Step 7
except Exception as e:
   # Handle API errors (e.g., check for 401/403, maybe clear cache and re-request auth)
   print(f"ERROR: API call failed: {e}")
   return {"status": "error", "error_message": f"API call failed: {e}"}

```

**Step 7: Return Tool Result**

* After a successful API call, process the result into a dictionary format that is useful for the LLM.
* **Crucially, include a** along with the data.

```
# Inside your tool function, after successful API call
    processed_result = [...] # Process api_result for the LLM
    return {"status": "success", "data": processed_result}

```

Full Code

Tools and AgentAgent CLIHelperSpec

tools\_and\_agent.py

```
import asyncio
from dotenv import load_dotenv
from google.adk.artifacts.in_memory_artifact_service import InMemoryArtifactService
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService
from google.genai import types

from .helpers import is_pending_auth_event, get_function_call_id, get_function_call_auth_config, get_user_input
from .tools_and_agent import root_agent

load_dotenv()

agent = root_agent

async def async_main():
  """
  Main asynchronous function orchestrating the agent interaction and authentication flow.
  """
  # --- Step 1: Service Initialization ---
  # Use in-memory services for session and artifact storage (suitable for demos/testing).
  session_service = InMemorySessionService()
  artifacts_service = InMemoryArtifactService()

  # Create a new user session to maintain conversation state.
  session = session_service.create_session(
      state={},  # Optional state dictionary for session-specific data
      app_name='my_app', # Application identifier
      user_id='user' # User identifier
  )

  # --- Step 2: Initial User Query ---
  # Define the user's initial request.
  query = 'Show me my user info'
  print(f"user: {query}")

  # Format the query into the Content structure expected by the ADK Runner.
  content = types.Content(role='user', parts=[types.Part(text=query)])

  # Initialize the ADK Runner
  runner = Runner(
      app_name='my_app',
      agent=agent,
      artifact_service=artifacts_service,
      session_service=session_service,
  )

  # --- Step 3: Send Query and Handle Potential Auth Request ---
  print("\nRunning agent with initial query...")
  events_async = runner.run_async(
      session_id=session.id, user_id='user', new_message=content
  )

  # Variables to store details if an authentication request occurs.
  auth_request_event_id, auth_config = None, None

  # Iterate through the events generated by the first run.
  async for event in events_async:
    # Check if this event is the specific 'adk_request_credential' function call.
    if is_pending_auth_event(event):
      print("--> Authentication required by agent.")
      auth_request_event_id = get_function_call_id(event)
      auth_config = get_function_call_auth_config(event)
      # Once the auth request is found and processed, exit this loop.
      # We need to pause execution here to get user input for authentication.
      break


  # If no authentication request was detected after processing all events, exit.
  if not auth_request_event_id or not auth_config:
      print("\nAuthentication not required for this query or processing finished.")
      return # Exit the main function

  # --- Step 4: Manual Authentication Step (Simulated OAuth 2.0 Flow) ---
  # This section simulates the user interaction part of an OAuth 2.0 flow.
  # In a real web application, this would involve browser redirects.

  # Define the Redirect URI. This *must* match one of the URIs registered
  # with the OAuth provider for your application. The provider sends the user
  # back here after they approve the request.
  redirect_uri = 'http://localhost:8000/dev-ui' # Example for local development

  # Construct the Authorization URL that the user must visit.
  # This typically includes the provider's authorization endpoint URL,
  # client ID, requested scopes, response type (e.g., 'code'), and the redirect URI.
  # Here, we retrieve the base authorization URI from the AuthConfig provided by ADK
  # and append the redirect_uri.
  # NOTE: A robust implementation would use urlencode and potentially add state, scope, etc.
  auth_request_uri = (
      auth_config.exchanged_auth_credential.oauth2.auth_uri
      + f'&redirect_uri={redirect_uri}' # Simple concatenation; ensure correct query param format
  )

  print("\n--- User Action Required ---")
  # Prompt the user to visit the authorization URL, log in, grant permissions,
  # and then paste the *full* URL they are redirected back to (which contains the auth code).
  auth_response_uri = await get_user_input(
      f'1. Please open this URL in your browser to log in:\n   {auth_request_uri}\n\n'
      f'2. After successful login and authorization, your browser will be redirected.\n'
      f'   Copy the *entire* URL from the browser\'s address bar.\n\n'
      f'3. Paste the copied URL here and press Enter:\n\n> '
  )

  # --- Step 5: Prepare Authentication Response for the Agent ---
  # Update the AuthConfig object with the information gathered from the user.
  # The ADK framework needs the full response URI (containing the code)
  # and the original redirect URI to complete the OAuth token exchange process internally.
  auth_config.exchanged_auth_credential.oauth2.auth_response_uri = auth_response_uri
  auth_config.exchanged_auth_credential.oauth2.redirect_uri = redirect_uri

  # Construct a FunctionResponse Content object to send back to the agent/runner.
  # This response explicitly targets the 'adk_request_credential' function call
  # identified earlier by its ID.
  auth_content = types.Content(
      role='user',
      parts=[
          types.Part(
              function_response=types.FunctionResponse(
                  # Crucially, link this response to the original request using the saved ID.
                  id=auth_request_event_id,
                  # The special name of the function call we are responding to.
                  name='adk_request_credential',
                  # The payload containing all necessary authentication details.
                  response=auth_config.model_dump(),
              )
          )
      ],
  )

  # --- Step 6: Resume Execution with Authentication ---
  print("\nSubmitting authentication details back to the agent...")
  # Run the agent again, this time providing the `auth_content` (FunctionResponse).
  # The ADK Runner intercepts this, processes the 'adk_request_credential' response
  # (performs token exchange, stores credentials), and then allows the agent
  # to retry the original tool call that required authentication, now succeeding with
  # a valid access token embedded.
  events_async = runner.run_async(
      session_id=session.id,
      user_id='user',
      new_message=auth_content, # Provide the prepared auth response
  )

  # Process and print the final events from the agent after authentication is complete.
  # This stream now contain the actual result from the tool (e.g., the user info).
  print("\n--- Agent Response after Authentication ---")
  async for event in events_async:
    print(event)


if __name__ == '__main__':
  asyncio.run(async_main())

```

agent\_cli.py

```
import asyncio
from dotenv import load_dotenv
from google.adk.artifacts.in_memory_artifact_service import InMemoryArtifactService
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService
from google.genai import types

from .helpers import is_pending_auth_event, get_function_call_id, get_function_call_auth_config, get_user_input
from .tools_and_agent import root_agent

load_dotenv()

agent = root_agent

async def async_main():
  """
  Main asynchronous function orchestrating the agent interaction and authentication flow.
  """
  # --- Step 1: Service Initialization ---
  # Use in-memory services for session and artifact storage (suitable for demos/testing).
  session_service = InMemorySessionService()
  artifacts_service = InMemoryArtifactService()

  # Create a new user session to maintain conversation state.
  session = session_service.create_session(
      state={},  # Optional state dictionary for session-specific data
      app_name='my_app', # Application identifier
      user_id='user' # User identifier
  )

  # --- Step 2: Initial User Query ---
  # Define the user's initial request.
  query = 'Show me my user info'
  print(f"user: {query}")

  # Format the query into the Content structure expected by the ADK Runner.
  content = types.Content(role='user', parts=[types.Part(text=query)])

  # Initialize the ADK Runner
  runner = Runner(
      app_name='my_app',
      agent=agent,
      artifact_service=artifacts_service,
      session_service=session_service,
  )

  # --- Step 3: Send Query and Handle Potential Auth Request ---
  print("\nRunning agent with initial query...")
  events_async = runner.run_async(
      session_id=session.id, user_id='user', new_message=content
  )

  # Variables to store details if an authentication request occurs.
  auth_request_event_id, auth_config = None, None

  # Iterate through the events generated by the first run.
  async for event in events_async:
    # Check if this event is the specific 'adk_request_credential' function call.
    if is_pending_auth_event(event):
      print("--> Authentication required by agent.")
      auth_request_event_id = get_function_call_id(event)
      auth_config = get_function_call_auth_config(event)
      # Once the auth request is found and processed, exit this loop.
      # We need to pause execution here to get user input for authentication.
      break


  # If no authentication request was detected after processing all events, exit.
  if not auth_request_event_id or not auth_config:
      print("\nAuthentication not required for this query or processing finished.")
      return # Exit the main function

  # --- Step 4: Manual Authentication Step (Simulated OAuth 2.0 Flow) ---
  # This section simulates the user interaction part of an OAuth 2.0 flow.
  # In a real web application, this would involve browser redirects.

  # Define the Redirect URI. This *must* match one of the URIs registered
  # with the OAuth provider for your application. The provider sends the user
  # back here after they approve the request.
  redirect_uri = 'http://localhost:8000/dev-ui' # Example for local development

  # Construct the Authorization URL that the user must visit.
  # This typically includes the provider's authorization endpoint URL,
  # client ID, requested scopes, response type (e.g., 'code'), and the redirect URI.
  # Here, we retrieve the base authorization URI from the AuthConfig provided by ADK
  # and append the redirect_uri.
  # NOTE: A robust implementation would use urlencode and potentially add state, scope, etc.
  auth_request_uri = (
      auth_config.exchanged_auth_credential.oauth2.auth_uri
      + f'&redirect_uri={redirect_uri}' # Simple concatenation; ensure correct query param format
  )

  print("\n--- User Action Required ---")
  # Prompt the user to visit the authorization URL, log in, grant permissions,
  # and then paste the *full* URL they are redirected back to (which contains the auth code).
  auth_response_uri = await get_user_input(
      f'1. Please open this URL in your browser to log in:\n   {auth_request_uri}\n\n'
      f'2. After successful login and authorization, your browser will be redirected.\n'
      f'   Copy the *entire* URL from the browser\'s address bar.\n\n'
      f'3. Paste the copied URL here and press Enter:\n\n> '
  )

  # --- Step 5: Prepare Authentication Response for the Agent ---
  # Update the AuthConfig object with the information gathered from the user.
  # The ADK framework needs the full response URI (containing the code)
  # and the original redirect URI to complete the OAuth token exchange process internally.
  auth_config.exchanged_auth_credential.oauth2.auth_response_uri = auth_response_uri
  auth_config.exchanged_auth_credential.oauth2.redirect_uri = redirect_uri

  # Construct a FunctionResponse Content object to send back to the agent/runner.
  # This response explicitly targets the 'adk_request_credential' function call
  # identified earlier by its ID.
  auth_content = types.Content(
      role='user',
      parts=[
          types.Part(
              function_response=types.FunctionResponse(
                  # Crucially, link this response to the original request using the saved ID.
                  id=auth_request_event_id,
                  # The special name of the function call we are responding to.
                  name='adk_request_credential',
                  # The payload containing all necessary authentication details.
                  response=auth_config.model_dump(),
              )
          )
      ],
  )

  # --- Step 6: Resume Execution with Authentication ---
  print("\nSubmitting authentication details back to the agent...")
  # Run the agent again, this time providing the `auth_content` (FunctionResponse).
  # The ADK Runner intercepts this, processes the 'adk_request_credential' response
  # (performs token exchange, stores credentials), and then allows the agent
  # to retry the original tool call that required authentication, now succeeding with
  # a valid access token embedded.
  events_async = runner.run_async(
      session_id=session.id,
      user_id='user',
      new_message=auth_content, # Provide the prepared auth response
  )

  # Process and print the final events from the agent after authentication is complete.
  # This stream now contain the actual result from the tool (e.g., the user info).
  print("\n--- Agent Response after Authentication ---")
  async for event in events_async:
    print(event)


if __name__ == '__main__':
  asyncio.run(async_main())

```

helpers.py

```
from google.adk.auth import AuthConfig
from google.adk.events import Event
import asyncio

# --- Helper Functions ---
async def get_user_input(prompt: str) -> str:
  """
  Asynchronously prompts the user for input in the console.

  Uses asyncio's event loop and run_in_executor to avoid blocking the main
  asynchronous execution thread while waiting for synchronous `input()`.

  Args:
    prompt: The message to display to the user.

  Returns:
    The string entered by the user.
  """
  loop = asyncio.get_event_loop()
  # Run the blocking `input()` function in a separate thread managed by the executor.
  return await loop.run_in_executor(None, input, prompt)


def is_pending_auth_event(event: Event) -> bool:
  """
  Checks if an ADK Event represents a request for user authentication credentials.

  The ADK framework emits a specific function call ('adk_request_credential')
  when a tool requires authentication that hasn't been previously satisfied.

  Args:
    event: The ADK Event object to inspect.

  Returns:
    True if the event is an 'adk_request_credential' function call, False otherwise.
  """
  # Safely checks nested attributes to avoid errors if event structure is incomplete.
  return (
      event.content
      and event.content.parts
      and event.content.parts[0] # Assuming the function call is in the first part
      and event.content.parts[0].function_call
      # The specific function name indicating an auth request from the ADK framework.
      and event.content.parts[0].function_call.name == 'adk_request_credential'
  )


def get_function_call_id(event: Event) -> str:
  """
  Extracts the unique ID of the function call from an ADK Event.

  This ID is crucial for correlating a function *response* back to the specific
  function *call* that the agent initiated to request for auth credentials.

  Args:
    event: The ADK Event object containing the function call.

  Returns:
    The unique identifier string of the function call.

  Raises:
    ValueError: If the function call ID cannot be found in the event structure.
                (Corrected typo from `contents` to `content` below)
  """
  # Navigate through the event structure to find the function call ID.
  if (
      event
      and event.content
      and event.content.parts
      and event.content.parts[0] # Use content, not contents
      and event.content.parts[0].function_call
      and event.content.parts[0].function_call.id
  ):
    return event.content.parts[0].function_call.id
  # If the ID is missing, raise an error indicating an unexpected event format.
  raise ValueError(f'Cannot get function call id from event {event}')


def get_function_call_auth_config(event: Event) -> AuthConfig:
  """
  Extracts the authentication configuration details from an 'adk_request_credential' event.

  Client should use this AuthConfig to necessary authentication details (like OAuth codes and state)
  and sent it back to the ADK to continue OAuth token exchanging.

  Args:
    event: The ADK Event object containing the 'adk_request_credential' call.

  Returns:
    An AuthConfig object populated with details from the function call arguments.

  Raises:
    ValueError: If the 'auth_config' argument cannot be found in the event.
                (Corrected typo from `contents` to `content` below)
  """
  if (
      event
      and event.content
      and event.content.parts
      and event.content.parts[0] # Use content, not contents
      and event.content.parts[0].function_call
      and event.content.parts[0].function_call.args
      and event.content.parts[0].function_call.args.get('auth_config')
  ):
    # Reconstruct the AuthConfig object using the dictionary provided in the arguments.
    # The ** operator unpacks the dictionary into keyword arguments for the constructor.
    return AuthConfig(
          **event.content.parts[0].function_call.args.get('auth_config')
      )
  raise ValueError(f'Cannot get auth config from event {event}')

```

```
openapi: 3.0.1
info:
title: Okta User Info API
version: 1.0.0
description: |-
   API to retrieve user profile information based on a valid Okta OIDC Access Token.
   Authentication is handled via OpenID Connect with Okta.
contact:
   name: API Support
   email: support@example.com # Replace with actual contact if available
servers:
- url: <substitute with your server name>
   description: Production Environment
paths:
/okta-jwt-user-api:
   get:
      summary: Get Authenticated User Info
      description: |-
      Fetches profile details for the user
      operationId: getUserInfo
      tags:
      - User Profile
      security:
      - okta_oidc:
            - openid
            - email
            - profile
      responses:
      '200':
         description: Successfully retrieved user information.
         content:
            application/json:
            schema:
               type: object
               properties:
                  sub:
                  type: string
                  description: Subject identifier for the user.
                  example: "abcdefg"
                  name:
                  type: string
                  description: Full name of the user.
                  example: "Example LastName"
                  locale:
                  type: string
                  description: User's locale, e.g., en-US or en_US.
                  example: "en_US"
                  email:
                  type: string
                  format: email
                  description: User's primary email address.
                  example: "username@example.com"
                  preferred_username:
                  type: string
                  description: Preferred username of the user (often the email).
                  example: "username@example.com"
                  given_name:
                  type: string
                  description: Given name (first name) of the user.
                  example: "Example"
                  family_name:
                  type: string
                  description: Family name (last name) of the user.
                  example: "LastName"
                  zoneinfo:
                  type: string
                  description: User's timezone, e.g., America/Los_Angeles.
                  example: "America/Los_Angeles"
                  updated_at:
                  type: integer
                  format: int64 # Using int64 for Unix timestamp
                  description: Timestamp when the user's profile was last updated (Unix epoch time).
                  example: 1743617719
                  email_verified:
                  type: boolean
                  description: Indicates if the user's email address has been verified.
                  example: true
               required:
                  - sub
                  - name
                  - locale
                  - email
                  - preferred_username
                  - given_name
                  - family_name
                  - zoneinfo
                  - updated_at
                  - email_verified
      '401':
         description: Unauthorized. The provided Bearer token is missing, invalid, or expired.
         content:
            application/json:
            schema:
               $ref: '#/components/schemas/Error'
      '403':
         description: Forbidden. The provided token does not have the required scopes or permissions to access this resource.
         content:
            application/json:
            schema:
               $ref: '#/components/schemas/Error'
components:
securitySchemes:
   okta_oidc:
      type: openIdConnect
      description: Authentication via Okta using OpenID Connect. Requires a Bearer Access Token.
      openIdConnectUrl: https://your-endpoint.okta.com/.well-known/openid-configuration
schemas:
   Error:
      type: object
      properties:
      code:
         type: string
         description: An error code.
      message:
         type: string
         description: A human-readable error message.
      required:
         - code
         - message

```

Back to top

---

