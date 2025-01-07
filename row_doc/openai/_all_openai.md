Assistants API quickstart

Beta

=================================

Step-by-step guide to creating an assistant.

A typical integration of the Assistants API has the following flow:

1.  Create an [Assistant](/docs/api-reference/assistants/createAssistant) by defining its custom instructions and picking a model. If helpful, add files and enable tools like Code Interpreter, File Search, and Function calling.
2.  Create a [Thread](/docs/api-reference/threads) when a user starts a conversation.
3.  Add [Messages](/docs/api-reference/messages) to the Thread as the user asks questions.
4.  [Run](/docs/api-reference/runs) the Assistant on the Thread to generate a response by calling the model and the tools.

This starter guide walks through the key steps to create and run an Assistant that uses [Code Interpreter](/docs/assistants/tools/code-interpreter). In this example, we're [creating an Assistant](/docs/api-reference/assistants/createAssistant) that is a personal math tutor, with the Code Interpreter tool enabled.

Calls to the Assistants API require that you pass a beta HTTP header. This is handled automatically if you’re using OpenAI’s official Python or Node.js SDKs. `OpenAI-Beta: assistants=v2`

Step 1: Create an Assistant
---------------------------

An [Assistant](/docs/api-reference/assistants/object) represents an entity that can be configured to respond to a user's messages using several parameters like `model`, `instructions`, and `tools`.

Create an Assistant

```python
from openai import OpenAI
client = OpenAI()

assistant = client.beta.assistants.create(
name="Math Tutor",
instructions="You are a personal math tutor. Write and run code to answer math questions.",
tools=[{"type": "code_interpreter"}],
model="gpt-4o",
)
```

```javascript
import OpenAI from "openai";
const openai = new OpenAI();

async function main() {
const assistant = await openai.beta.assistants.create({
  name: "Math Tutor",
  instructions: "You are a personal math tutor. Write and run code to answer math questions.",
  tools: [{ type: "code_interpreter" }],
  model: "gpt-4o"
});
}

main();
```

```bash
curl "https://api.openai.com/v1/assistants" \
-H "Content-Type: application/json" \
-H "Authorization: Bearer $OPENAI_API_KEY" \
-H "OpenAI-Beta: assistants=v2" \
-d '{
  "instructions": "You are a personal math tutor. Write and run code to answer math questions.",
  "name": "Math Tutor",
  "tools": [{"type": "code_interpreter"}],
  "model": "gpt-4o"
}'
```

Step 2: Create a Thread
-----------------------

A [Thread](/docs/api-reference/threads/object) represents a conversation between a user and one or many Assistants. You can create a Thread when a user (or your AI application) starts a conversation with your Assistant.

Create a Thread

```python
thread = client.beta.threads.create()
```

```javascript
const thread = await openai.beta.threads.create();
```

```bash
curl https://api.openai.com/v1/threads \
-H "Content-Type: application/json" \
-H "Authorization: Bearer $OPENAI_API_KEY" \
-H "OpenAI-Beta: assistants=v2" \
-d ''
```

Step 3: Add a Message to the Thread
-----------------------------------

The contents of the messages your users or applications create are added as [Message](/docs/api-reference/messages/object) objects to the Thread. Messages can contain both text and files. There is a limit of 100,000 Messages per Thread and we smartly truncate any context that does not fit into the model's context window.

Add a Message to the Thread

```python
message = client.beta.threads.messages.create(
thread_id=thread.id,
role="user",
content="I need to solve the equation `3x + 11 = 14`. Can you help me?"
)
```

```javascript
const message = await openai.beta.threads.messages.create(
thread.id,
{
  role: "user",
  content: "I need to solve the equation `3x + 11 = 14`. Can you help me?"
}
);
```

```bash
curl https://api.openai.com/v1/threads/thread_abc123/messages \
-H "Content-Type: application/json" \
-H "Authorization: Bearer $OPENAI_API_KEY" \
-H "OpenAI-Beta: assistants=v2" \
-d '{
    "role": "user",
    "content": "I need to solve the equation `3x + 11 = 14`. Can you help me?"
  }'
```

Step 4: Create a Run
--------------------

Once all the user Messages have been added to the Thread, you can [Run](/docs/api-reference/runs/object) the Thread with any Assistant. Creating a Run uses the model and tools associated with the Assistant to generate a response. These responses are added to the Thread as `assistant` Messages.

With streamingWithout streaming

You can use the 'create and stream' helpers in the Python and Node SDKs to create a run and stream the response.

Create and Stream a Run

```python
from typing_extensions import override
from openai import AssistantEventHandler

# First, we create a EventHandler class to define
# how we want to handle the events in the response stream.

class EventHandler(AssistantEventHandler):    
@override
def on_text_created(self, text) -> None:
  print(f"\nassistant > ", end="", flush=True)
    
@override
def on_text_delta(self, delta, snapshot):
  print(delta.value, end="", flush=True)
    
def on_tool_call_created(self, tool_call):
  print(f"\nassistant > {tool_call.type}\n", flush=True)

def on_tool_call_delta(self, delta, snapshot):
  if delta.type == 'code_interpreter':
    if delta.code_interpreter.input:
      print(delta.code_interpreter.input, end="", flush=True)
    if delta.code_interpreter.outputs:
      print(f"\n\noutput >", flush=True)
      for output in delta.code_interpreter.outputs:
        if output.type == "logs":
          print(f"\n{output.logs}", flush=True)

# Then, we use the `stream` SDK helper 
# with the `EventHandler` class to create the Run 
# and stream the response.

with client.beta.threads.runs.stream(
thread_id=thread.id,
assistant_id=assistant.id,
instructions="Please address the user as Jane Doe. The user has a premium account.",
event_handler=EventHandler(),
) as stream:
stream.until_done()
```

```javascript
// We use the stream SDK helper to create a run with
// streaming. The SDK provides helpful event listeners to handle 
// the streamed response.

const run = openai.beta.threads.runs.stream(thread.id, {
  assistant_id: assistant.id
})
  .on('textCreated', (text) => process.stdout.write('\nassistant > '))
  .on('textDelta', (textDelta, snapshot) => process.stdout.write(textDelta.value))
  .on('toolCallCreated', (toolCall) => process.stdout.write(`\nassistant > ${toolCall.type}\n\n`))
  .on('toolCallDelta', (toolCallDelta, snapshot) => {
    if (toolCallDelta.type === 'code_interpreter') {
      if (toolCallDelta.code_interpreter.input) {
        process.stdout.write(toolCallDelta.code_interpreter.input);
      }
      if (toolCallDelta.code_interpreter.outputs) {
        process.stdout.write("\noutput >\n");
        toolCallDelta.code_interpreter.outputs.forEach(output => {
          if (output.type === "logs") {
            process.stdout.write(`\n${output.logs}\n`);
          }
        });
      }
    }
  });
```

See the full list of Assistants streaming events in our API reference [here](/docs/api-reference/assistants-streaming/events). You can also see a list of SDK event listeners for these events in the [Python](https://github.com/openai/openai-python/blob/main/helpers.md#assistant-events) & [Node](https://github.com/openai/openai-node/blob/master/helpers.md#assistant-events) repository documentation.

Next steps
----------

1.  Continue learning about Assistants Concepts in the [Deep Dive](/docs/assistants/deep-dive)
2.  Learn more about [Tools](/docs/assistants/tools)
3.  Explore the [Assistants playground](/playground?mode=assistant)
4.  Check out our [Assistants Quickstart app](https://github.com/openai/openai-assistants-quickstart) on github

===

Assistants API deep dive

Beta

================================

In-depth guide to creating and managing assistants.

As described in the [Assistants Overview](/docs/assistants/overview), there are several concepts involved in building an app with the Assistants API.

This guide goes deeper into each of these concepts.

If you want to get started coding right away, check out the [Assistants API Quickstart](/docs/assistants/quickstart).

Creating Assistants
-------------------

We recommend using OpenAI's [latest models](/docs/models#gpt-4-turbo-and-gpt-4) with the Assistants API for best results and maximum compatibility with tools.

To get started, creating an Assistant only requires specifying the `model` to use. But you can further customize the behavior of the Assistant:

1.  Use the `instructions` parameter to guide the personality of the Assistant and define its goals. Instructions are similar to system messages in the Chat Completions API.
2.  Use the `tools` parameter to give the Assistant access to up to 128 tools. You can give it access to OpenAI-hosted tools like `code_interpreter` and `file_search`, or call a third-party tools via a `function` calling.
3.  Use the `tool_resources` parameter to give the tools like `code_interpreter` and `file_search` access to files. Files are uploaded using the `File` [upload endpoint](/docs/api-reference/files/create) and must have the `purpose` set to `assistants` to be used with this API.

For example, to create an Assistant that can create data visualization based on a `.csv` file, first upload a file.

```python
file = client.files.create(
file=open("revenue-forecast.csv", "rb"),
purpose='assistants'
)
```

```javascript
const file = await openai.files.create({
file: fs.createReadStream("revenue-forecast.csv"),
purpose: "assistants",
});
```

```bash
curl https://api.openai.com/v1/files \
-H "Authorization: Bearer $OPENAI_API_KEY" \
-F purpose="assistants" \
-F file="@revenue-forecast.csv"
```

Then, create the Assistant with the `code_interpreter` tool enabled and provide the file as a resource to the tool.

```python
assistant = client.beta.assistants.create(
name="Data visualizer",
description="You are great at creating beautiful data visualizations. You analyze data present in .csv files, understand trends, and come up with data visualizations relevant to those trends. You also share a brief text summary of the trends observed.",
model="gpt-4o",
tools=[{"type": "code_interpreter"}],
tool_resources={
  "code_interpreter": {
    "file_ids": [file.id]
  }
}
)
```

```javascript
const assistant = await openai.beta.assistants.create({
name: "Data visualizer",
description: "You are great at creating beautiful data visualizations. You analyze data present in .csv files, understand trends, and come up with data visualizations relevant to those trends. You also share a brief text summary of the trends observed.",
model: "gpt-4o",
tools: [{"type": "code_interpreter"}],
tool_resources: {
  "code_interpreter": {
    "file_ids": [file.id]
  }
}
});
```

```bash
curl https://api.openai.com/v1/assistants \
-H "Authorization: Bearer $OPENAI_API_KEY" \
-H "Content-Type: application/json" \
-H "OpenAI-Beta: assistants=v2" \
-d '{
  "name": "Data visualizer",
  "description": "You are great at creating beautiful data visualizations. You analyze data present in .csv files, understand trends, and come up with data visualizations relevant to those trends. You also share a brief text summary of the trends observed.",
  "model": "gpt-4o",
  "tools": [{"type": "code_interpreter"}],
  "tool_resources": {
    "code_interpreter": {
      "file_ids": ["file-BK7bzQj3FfZFXr7DbL6xJwfo"]
    }
  }
}'
```

You can attach a maximum of 20 files to `code_interpreter` and 10,000 files to `file_search` (using `vector_store` [objects](/docs/api-reference/vector-stores/object)).

Each file can be at most 512 MB in size and have a maximum of 5,000,000 tokens. By default, the size of all the files uploaded in your project cannot exceed 100 GB, but you can reach out to our support team to increase this limit.

Managing Threads and Messages
-----------------------------

Threads and Messages represent a conversation session between an Assistant and a user. There is a limit of 100,000 Messages per Thread. Once the size of the Messages exceeds the context window of the model, the Thread will attempt to smartly truncate messages, before fully dropping the ones it considers the least important.

You can create a Thread with an initial list of Messages like this:

```python
thread = client.beta.threads.create(
messages=[
  {
    "role": "user",
    "content": "Create 3 data visualizations based on the trends in this file.",
    "attachments": [
      {
        "file_id": file.id,
        "tools": [{"type": "code_interpreter"}]
      }
    ]
  }
]
)
```

```javascript
const thread = await openai.beta.threads.create({
messages: [
  {
    "role": "user",
    "content": "Create 3 data visualizations based on the trends in this file.",
    "attachments": [
      {
        file_id: file.id,
        tools: [{type: "code_interpreter"}]
      }
    ]
  }
]
});
```

```bash
curl https://api.openai.com/v1/threads \
-H "Authorization: Bearer $OPENAI_API_KEY" \
-H "Content-Type: application/json" \
-H "OpenAI-Beta: assistants=v2" \
-d '{
  "messages": [
    {
      "role": "user",
      "content": "Create 3 data visualizations based on the trends in this file.",
      "attachments": [
        {
          "file_id": "file-ACq8OjcLQm2eIG0BvRM4z5qX",
          "tools": [{"type": "code_interpreter"}]
        }
      ]
    }
  ]
}'
```

Messages can contain text, images, or file attachment. Message `attachments` are helper methods that add files to a thread's `tool_resources`. You can also choose to add files to the `thread.tool_resources` directly.

### Creating image input content

Message content can contain either external image URLs or File IDs uploaded via the [File API](/docs/api-reference/files/create). Only [models](/docs/models) with Vision support can accept image input. Supported image content types include png, jpg, gif, and webp. When creating image files, pass `purpose="vision"` to allow you to later download and display the input content. Currently, there is a 100GB limit per project. Please contact us to request a limit increase.

Tools cannot access image content unless specified. To pass image files to Code Interpreter, add the file ID in the message `attachments` list to allow the tool to read and analyze the input. Image URLs cannot be downloaded in Code Interpreter today.

```python
file = client.files.create(
file=open("myimage.png", "rb"),
purpose="vision"
)
thread = client.beta.threads.create(
messages=[
  {
    "role": "user",
    "content": [
      {
        "type": "text",
        "text": "What is the difference between these images?"
      },
      {
        "type": "image_url",
        "image_url": {"url": "https://example.com/image.png"}
      },
      {
        "type": "image_file",
        "image_file": {"file_id": file.id}
      },
    ],
  }
]
)
```

```javascript
import fs from "fs";
const file = await openai.files.create({
file: fs.createReadStream("myimage.png"),
purpose: "vision",
});
const thread = await openai.beta.threads.create({
messages: [
  {
    "role": "user",
    "content": [
      {
        "type": "text",
        "text": "What is the difference between these images?"
      },
      {
        "type": "image_url",
        "image_url": {"url": "https://example.com/image.png"}
      },
      {
        "type": "image_file",
        "image_file": {"file_id": file.id}
      },
    ]
  }
]
});
```

```bash
# Upload a file with an "vision" purpose
curl https://api.openai.com/v1/files \
-H "Authorization: Bearer $OPENAI_API_KEY" \
-F purpose="vision" \
-F file="@/path/to/myimage.png"

## Pass the file ID in the content
curl https://api.openai.com/v1/threads \
-H "Authorization: Bearer $OPENAI_API_KEY" \
-H "Content-Type: application/json" \
-H "OpenAI-Beta: assistants=v2" \
-d '{
  "messages": [
    {
      "role": "user",
      "content": [
        {
          "type": "text",
          "text": "What is the difference between these images?"
        },
        {
          "type": "image_url",
          "image_url": {"url": "https://example.com/image.png"}
        },
        {
          "type": "image_file",
          "image_file": {"file_id": file.id}
        }
      ]
    }
  ]
}'
```

#### Low or high fidelity image understanding

By controlling the `detail` parameter, which has three options, `low`, `high`, or `auto`, you have control over how the model processes the image and generates its textual understanding.

*   `low` will enable the "low res" mode. The model will receive a low-res 512px x 512px version of the image, and represent the image with a budget of 85 tokens. This allows the API to return faster responses and consume fewer input tokens for use cases that do not require high detail.
*   `high` will enable "high res" mode, which first allows the model to see the low res image and then creates detailed crops of input images based on the input image size. Use the [pricing calculator](https://openai.com/api/pricing/) to see token counts for various image sizes.

```python
thread = client.beta.threads.create(
messages=[
  {
    "role": "user",
    "content": [
      {
        "type": "text",
        "text": "What is this an image of?"
      },
      {
        "type": "image_url",
        "image_url": {
          "url": "https://example.com/image.png",
          "detail": "high"
        }
      },
    ],
  }
]
)
```

```javascript
const thread = await openai.beta.threads.create({
messages: [
  {
    "role": "user",
    "content": [
        {
          "type": "text",
          "text": "What is this an image of?"
        },
        {
          "type": "image_url",
          "image_url": {
            "url": "https://example.com/image.png",
            "detail": "high"
          }
        },
    ]
  }
]
});
```

```bash
curl https://api.openai.com/v1/threads \
-H "Authorization: Bearer $OPENAI_API_KEY" \
-H "Content-Type: application/json" \
-H "OpenAI-Beta: assistants=v2" \
-d '{
  "messages": [
    {
      "role": "user",
      "content": [
        {
          "type": "text",
          "text": "What is this an image of?"
        },
        {
          "type": "image_url",
          "image_url": {
            "url": "https://example.com/image.png",
            "detail": "high"
          }
        },
      ]
    }
  ]
}'
```

### Context window management

The Assistants API automatically manages the truncation to ensure it stays within the model's maximum context length. You can customize this behavior by specifying the maximum tokens you'd like a run to utilize and/or the maximum number of recent messages you'd like to include in a run.

#### Max Completion and Max Prompt Tokens

To control the token usage in a single Run, set `max_prompt_tokens` and `max_completion_tokens` when creating the Run. These limits apply to the total number of tokens used in all completions throughout the Run's lifecycle.

For example, initiating a Run with `max_prompt_tokens` set to 500 and `max_completion_tokens` set to 1000 means the first completion will truncate the thread to 500 tokens and cap the output at 1000 tokens. If only 200 prompt tokens and 300 completion tokens are used in the first completion, the second completion will have available limits of 300 prompt tokens and 700 completion tokens.

If a completion reaches the `max_completion_tokens` limit, the Run will terminate with a status of `incomplete`, and details will be provided in the `incomplete_details` field of the Run object.

When using the File Search tool, we recommend setting the max\_prompt\_tokens to no less than 20,000. For longer conversations or multiple interactions with File Search, consider increasing this limit to 50,000, or ideally, removing the max\_prompt\_tokens limits altogether to get the highest quality results.

#### Truncation Strategy

You may also specify a truncation strategy to control how your thread should be rendered into the model's context window. Using a truncation strategy of type `auto` will use OpenAI's default truncation strategy. Using a truncation strategy of type `last_messages` will allow you to specify the number of the most recent messages to include in the context window.

### Message annotations

Messages created by Assistants may contain [`annotations`](/docs/api-reference/messages/object#messages/object-content) within the `content` array of the object. Annotations provide information around how you should annotate the text in the Message.

There are two types of Annotations:

1.  `file_citation`: File citations are created by the [`file_search`](/docs/assistants/tools/file-search) tool and define references to a specific file that was uploaded and used by the Assistant to generate the response.
2.  `file_path`: File path annotations are created by the [`code_interpreter`](/docs/assistants/tools/code-interpreter) tool and contain references to the files generated by the tool.

When annotations are present in the Message object, you'll see illegible model-generated substrings in the text that you should replace with the annotations. These strings may look something like `【13†source】` or `sandbox:/mnt/data/file.csv`. Here’s an example python code snippet that replaces these strings with information present in the annotations.

```python
# Retrieve the message object
message = client.beta.threads.messages.retrieve(
thread_id="...",
message_id="..."
)
# Extract the message content
message_content = message.content[0].text
annotations = message_content.annotations
citations = []
# Iterate over the annotations and add footnotes
for index, annotation in enumerate(annotations):
  # Replace the text with a footnote
  message_content.value = message_content.value.replace(annotation.text, f' [{index}]')
  # Gather citations based on annotation attributes
  if (file_citation := getattr(annotation, 'file_citation', None)):
      cited_file = client.files.retrieve(file_citation.file_id)
      citations.append(f'[{index}] {file_citation.quote} from {cited_file.filename}')
  elif (file_path := getattr(annotation, 'file_path', None)):
      cited_file = client.files.retrieve(file_path.file_id)
      citations.append(f'[{index}] Click <here> to download {cited_file.filename}')
      # Note: File download functionality not implemented above for brevity
# Add footnotes to the end of the message before displaying to user
message_content.value += '\n' + '\n'.join(citations)
```

Runs and Run Steps
------------------

When you have all the context you need from your user in the Thread, you can run the Thread with an Assistant of your choice.

```python
run = client.beta.threads.runs.create(
thread_id=thread.id,
assistant_id=assistant.id
)
```

```javascript
const run = await openai.beta.threads.runs.create(
thread.id,
{ assistant_id: assistant.id }
);
```

```bash
curl https://api.openai.com/v1/threads/THREAD_ID/runs \
-H "Authorization: Bearer $OPENAI_API_KEY" \
-H "Content-Type: application/json" \
-H "OpenAI-Beta: assistants=v2" \
-d '{
  "assistant_id": "asst_ToSF7Gb04YMj8AMMm50ZLLtY"
}'
```

By default, a Run will use the `model` and `tools` configuration specified in Assistant object, but you can override most of these when creating the Run for added flexibility:

```python
run = client.beta.threads.runs.create(
thread_id=thread.id,
assistant_id=assistant.id,
model="gpt-4o",
instructions="New instructions that override the Assistant instructions",
tools=[{"type": "code_interpreter"}, {"type": "file_search"}]
)
```

```javascript
const run = await openai.beta.threads.runs.create(
thread.id,
{
  assistant_id: assistant.id,
  model: "gpt-4o",
  instructions: "New instructions that override the Assistant instructions",
  tools: [{"type": "code_interpreter"}, {"type": "file_search"}]
}
);
```

```bash
curl https://api.openai.com/v1/threads/THREAD_ID/runs \
-H "Authorization: Bearer $OPENAI_API_KEY" \
-H "Content-Type: application/json" \
-H "OpenAI-Beta: assistants=v2" \
-d '{
  "assistant_id": "ASSISTANT_ID",
  "model": "gpt-4o",
  "instructions": "New instructions that override the Assistant instructions",
  "tools": [{"type": "code_interpreter"}, {"type": "file_search"}]
}'
```

Note: `tool_resources` associated with the Assistant cannot be overridden during Run creation. You must use the [modify Assistant](/docs/api-reference/assistants/modifyAssistant) endpoint to do this.

#### Run lifecycle

Run objects can have multiple statuses.

![Run lifecycle - diagram showing possible status transitions](https://cdn.openai.com/API/docs/images/diagram-run-statuses-v2.png)

|Status|Definition|
|---|---|
|queued|When Runs are first created or when you complete the required_action, they are moved to a queued status. They should almost immediately move to in_progress.|
|in_progress|While in_progress, the Assistant uses the model and tools to perform steps. You can view progress being made by the Run by examining the Run Steps.|
|completed|The Run successfully completed! You can now view all Messages the Assistant added to the Thread, and all the steps the Run took. You can also continue the conversation by adding more user Messages to the Thread and creating another Run.|
|requires_action|When using the Function calling tool, the Run will move to a required_action state once the model determines the names and arguments of the functions to be called. You must then run those functions and submit the outputs before the run proceeds. If the outputs are not provided before the expires_at timestamp passes (roughly 10 mins past creation), the run will move to an expired status.|
|expired|This happens when the function calling outputs were not submitted before expires_at and the run expires. Additionally, if the runs take too long to execute and go beyond the time stated in expires_at, our systems will expire the run.|
|cancelling|You can attempt to cancel an in_progress run using the Cancel Run endpoint. Once the attempt to cancel succeeds, status of the Run moves to cancelled. Cancellation is attempted but not guaranteed.|
|cancelled|Run was successfully cancelled.|
|failed|You can view the reason for the failure by looking at the last_error object in the Run. The timestamp for the failure will be recorded under failed_at.|
|incomplete|Run ended due to max_prompt_tokens or max_completion_tokens reached. You can view the specific reason by looking at the incomplete_details object in the Run.|

#### Polling for updates

If you are not using [streaming](/docs/assistants/overview#step-4-create-a-run?context=with-streaming), in order to keep the status of your run up to date, you will have to periodically [retrieve the Run](/docs/api-reference/runs/getRun) object. You can check the status of the run each time you retrieve the object to determine what your application should do next.

You can optionally use Polling Helpers in our [Node](https://github.com/openai/openai-node?tab=readme-ov-file#polling-helpers) and [Python](https://github.com/openai/openai-python?tab=readme-ov-file#polling-helpers) SDKs to help you with this. These helpers will automatically poll the Run object for you and return the Run object when it's in a terminal state.

#### Thread locks

When a Run is `in_progress` and not in a terminal state, the Thread is locked. This means that:

*   New Messages cannot be added to the Thread.
*   New Runs cannot be created on the Thread.

#### Run steps

![Run steps lifecycle - diagram showing possible status transitions](https://cdn.openai.com/API/docs/images/diagram-2.png)

Run step statuses have the same meaning as Run statuses.

Most of the interesting detail in the Run Step object lives in the `step_details` field. There can be two types of step details:

1.  `message_creation`: This Run Step is created when the Assistant creates a Message on the Thread.
2.  `tool_calls`: This Run Step is created when the Assistant calls a tool. Details around this are covered in the relevant sections of the [Tools](/docs/assistants/tools) guide.

Data Access Guidance
--------------------

Currently, Assistants, Threads, Messages, and Vector Stores created via the API are scoped to the Project they're created in. As such, any person with API key access to that Project is able to read or write Assistants, Threads, Messages, and Runs in the Project.

We strongly recommend the following data access controls:

*   _Implement authorization._ Before performing reads or writes on Assistants, Threads, Messages, and Vector Stores, ensure that the end-user is authorized to do so. For example, store in your database the object IDs that the end-user has access to, and check it before fetching the object ID with the API.
*   _Restrict API key access._ Carefully consider who in your organization should have API keys and be part of a Project. Periodically audit this list. API keys enable a wide range of operations including reading and modifying sensitive information, such as Messages and Files.
*   _Create separate accounts._ Consider creating separate Projects for different applications in order to isolate data across multiple applications.

===

Assistants API overview

Beta

===============================

Build AI Assistants with essential tools and integrations.

The Assistants API allows you to build AI assistants within your own applications. An Assistant has instructions and can leverage models, tools, and files to respond to user queries. The Assistants API currently supports three types of [tools](/docs/assistants/tools): Code Interpreter, File Search, and Function calling.

You can explore the capabilities of the Assistants API using the [Assistants playground](/playground?mode=assistant) or by building a step-by-step integration outlined in our [Assistants API quickstart](/docs/assistants/quickstart).

How Assistants work
-------------------

The Assistants API is designed to help developers build powerful AI assistants capable of performing a variety of tasks.

The Assistants API is in **beta** and we are actively working on adding more functionality. Share your feedback in our [Developer Forum](https://community.openai.com/)!

1.  Assistants can call OpenAI’s **[models](/docs/models)** with specific instructions to tune their personality and capabilities.
2.  Assistants can access **multiple tools in parallel**. These can be both OpenAI-hosted tools — like [code\_interpreter](/docs/assistants/tools/code-interpreter) and [file\_search](/docs/assistants/tools/file-search) — or tools you build / host (via [function calling](/docs/assistants/tools/function-calling)).
3.  Assistants can access **persistent Threads**. Threads simplify AI application development by storing message history and truncating it when the conversation gets too long for the model’s context length. You create a Thread once, and simply append Messages to it as your users reply.
4.  Assistants can access files in several formats — either as part of their creation or as part of Threads between Assistants and users. When using tools, Assistants can also create files (e.g., images, spreadsheets, etc) and cite files they reference in the Messages they create.

Objects
-------

![Assistants object architecture diagram](https://cdn.openai.com/API/docs/images/diagram-assistant.webp)

|Object|What it represents|
|---|---|
|Assistant|Purpose-built AI that uses OpenAI’s models and calls tools|
|Thread|A conversation session between an Assistant and a user. Threads store Messages and automatically handle truncation to fit content into a model’s context.|
|Message|A message created by an Assistant or a user. Messages can include text, images, and other files. Messages stored as a list on the Thread.|
|Run|An invocation of an Assistant on a Thread. The Assistant uses its configuration and the Thread’s Messages to perform tasks by calling models and tools. As part of a Run, the Assistant appends Messages to the Thread.|
|Run Step|A detailed list of steps the Assistant took as part of a Run. An Assistant can call tools or create Messages during its run. Examining Run Steps allows you to introspect how the Assistant is getting to its final results.|

Was this page useful?

===

Assistants Code Interpreter

Beta

===================================

Code Interpreter allows Assistants to write and run Python code in a sandboxed execution environment. This tool can process files with diverse data and formatting, and generate files with data and images of graphs. Code Interpreter allows your Assistant to run code iteratively to solve challenging code and math problems. When your Assistant writes code that fails to run, it can iterate on this code by attempting to run different code until the code execution succeeds.

See a quickstart of how to get started with Code Interpreter [here](/docs/assistants/overview#step-1-create-an-assistant?context=with-streaming).

How it works
------------

Code Interpreter is charged at $0.03 per session. If your Assistant calls Code Interpreter simultaneously in two different threads (e.g., one thread per end-user), two Code Interpreter sessions are created. Each session is active by default for one hour, which means that you only pay for one session per if users interact with Code Interpreter in the same thread for up to one hour.

### Enabling Code Interpreter

Pass `code_interpreter` in the `tools` parameter of the Assistant object to enable Code Interpreter:

```python
assistant = client.beta.assistants.create(
instructions="You are a personal math tutor. When asked a math question, write and run code to answer the question.",
model="gpt-4o",
tools=[{"type": "code_interpreter"}]
)
```

```javascript
const assistant = await openai.beta.assistants.create({
instructions: "You are a personal math tutor. When asked a math question, write and run code to answer the question.",
model: "gpt-4o",
tools: [{"type": "code_interpreter"}]
});
```

```bash
curl https://api.openai.com/v1/assistants \
-u :$OPENAI_API_KEY \
-H 'Content-Type: application/json' \
-H 'OpenAI-Beta: assistants=v2' \
-d '{
  "instructions": "You are a personal math tutor. When asked a math question, write and run code to answer the question.",
  "tools": [
    { "type": "code_interpreter" }
  ],
  "model": "gpt-4o"
}'
```

The model then decides when to invoke Code Interpreter in a Run based on the nature of the user request. This behavior can be promoted by prompting in the Assistant's `instructions` (e.g., “write code to solve this problem”).

### Passing files to Code Interpreter

Files that are passed at the Assistant level are accessible by all Runs with this Assistant:

```python
# Upload a file with an "assistants" purpose
file = client.files.create(
file=open("mydata.csv", "rb"),
purpose='assistants'
)

# Create an assistant using the file ID
assistant = client.beta.assistants.create(
instructions="You are a personal math tutor. When asked a math question, write and run code to answer the question.",
model="gpt-4o",
tools=[{"type": "code_interpreter"}],
tool_resources={
  "code_interpreter": {
    "file_ids": [file.id]
  }
}
)
```

```javascript
// Upload a file with an "assistants" purpose
const file = await openai.files.create({
file: fs.createReadStream("mydata.csv"),
purpose: "assistants",
});

// Create an assistant using the file ID
const assistant = await openai.beta.assistants.create({
instructions: "You are a personal math tutor. When asked a math question, write and run code to answer the question.",
model: "gpt-4o",
tools: [{"type": "code_interpreter"}],
tool_resources: {
  "code_interpreter": {
    "file_ids": [file.id]
  }
}
});
```

```bash
# Upload a file with an "assistants" purpose
curl https://api.openai.com/v1/files \
-H "Authorization: Bearer $OPENAI_API_KEY" \
-F purpose="assistants" \
-F file="@/path/to/mydata.csv"

# Create an assistant using the file ID
curl https://api.openai.com/v1/assistants \
-u :$OPENAI_API_KEY \
-H 'Content-Type: application/json' \
-H 'OpenAI-Beta: assistants=v2' \
-d '{
  "instructions": "You are a personal math tutor. When asked a math question, write and run code to answer the question.",
  "tools": [{"type": "code_interpreter"}],
  "model": "gpt-4o",
  "tool_resources": {
    "code_interpreter": {
      "file_ids": ["file-BK7bzQj3FfZFXr7DbL6xJwfo"]
    }
  }
}'
```

Files can also be passed at the Thread level. These files are only accessible in the specific Thread. Upload the File using the [File upload](/docs/api-reference/files/create) endpoint and then pass the File ID as part of the Message creation request:

```python
thread = client.beta.threads.create(
messages=[
  {
    "role": "user",
    "content": "I need to solve the equation `3x + 11 = 14`. Can you help me?",
    "attachments": [
      {
        "file_id": file.id,
        "tools": [{"type": "code_interpreter"}]
      }
    ]
  }
]
)
```

```javascript
const thread = await openai.beta.threads.create({
messages: [
  {
    "role": "user",
    "content": "I need to solve the equation `3x + 11 = 14`. Can you help me?",
    "attachments": [
      {
        file_id: file.id,
        tools: [{type: "code_interpreter"}]
      }
    ]
  }
]
});
```

```bash
curl https://api.openai.com/v1/threads/thread_abc123/messages \
-u :$OPENAI_API_KEY \
-H 'Content-Type: application/json' \
-H 'OpenAI-Beta: assistants=v2' \
-d '{
  "role": "user",
  "content": "I need to solve the equation `3x + 11 = 14`. Can you help me?",
  "attachments": [
    {
      "file_id": "file-ACq8OjcLQm2eIG0BvRM4z5qX",
      "tools": [{"type": "code_interpreter"}]
    }
  ]
}'
```

Files have a maximum size of 512 MB. Code Interpreter supports a variety of file formats including `.csv`, `.pdf`, `.json` and many more. More details on the file extensions (and their corresponding MIME-types) supported can be found in the [Supported files](#supported-files) section below.

### Reading images and files generated by Code Interpreter

Code Interpreter in the API also outputs files, such as generating image diagrams, CSVs, and PDFs. There are two types of files that are generated:

1.  Images
2.  Data files (e.g. a `csv` file with data generated by the Assistant)

When Code Interpreter generates an image, you can look up and download this file in the `file_id` field of the Assistant Message response:

```json
{
    "id": "msg_abc123",
    "object": "thread.message",
    "created_at": 1698964262,
    "thread_id": "thread_abc123",
    "role": "assistant",
    "content": [
    {
      "type": "image_file",
      "image_file": {
        "file_id": "file-abc123"
      }
    }
  ]
  # ...
}
```

The file content can then be downloaded by passing the file ID to the Files API:

```python
from openai import OpenAI

client = OpenAI()

image_data = client.files.content("file-abc123")
image_data_bytes = image_data.read()

with open("./my-image.png", "wb") as file:
  file.write(image_data_bytes)
```

```javascript
import fs from "fs";
import OpenAI from "openai";

const openai = new OpenAI();

async function main() {
const response = await openai.files.content("file-abc123");

// Extract the binary data from the Response object
const image_data = await response.arrayBuffer();

// Convert the binary data to a Buffer
const image_data_buffer = Buffer.from(image_data);

// Save the image to a specific location
fs.writeFileSync("./my-image.png", image_data_buffer);
}

main();
```

```bash
curl https://api.openai.com/v1/files/file-abc123/content \
-H "Authorization: Bearer $OPENAI_API_KEY" \
--output image.png
```

When Code Interpreter references a file path (e.g., ”Download this csv file”), file paths are listed as annotations. You can convert these annotations into links to download the file:

```json
{
  "id": "msg_abc123",
  "object": "thread.message",
  "created_at": 1699073585,
  "thread_id": "thread_abc123",
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": {
        "value": "The rows of the CSV file have been shuffled and saved to a new CSV file. You can download the shuffled CSV file from the following link:\n\n[Download Shuffled CSV File](sandbox:/mnt/data/shuffled_file.csv)",
        "annotations": [
          {
            "type": "file_path",
            "text": "sandbox:/mnt/data/shuffled_file.csv",
            "start_index": 167,
            "end_index": 202,
            "file_path": {
              "file_id": "file-abc123"
            }
          }
          ...
```

### Input and output logs of Code Interpreter

By listing the steps of a Run that called Code Interpreter, you can inspect the code `input` and `outputs` logs of Code Interpreter:

```python
run_steps = client.beta.threads.runs.steps.list(
thread_id=thread.id,
run_id=run.id
)
```

```javascript
const runSteps = await openai.beta.threads.runs.steps.list(
thread.id,
run.id
);
```

```bash
curl https://api.openai.com/v1/threads/thread_abc123/runs/RUN_ID/steps \
-H "Authorization: Bearer $OPENAI_API_KEY" \
-H "OpenAI-Beta: assistants=v2" \
```

```bash
{
  "object": "list",
  "data": [
    {
      "id": "step_abc123",
      "object": "thread.run.step",
      "type": "tool_calls",
      "run_id": "run_abc123",
      "thread_id": "thread_abc123",
      "status": "completed",
      "step_details": {
        "type": "tool_calls",
        "tool_calls": [
          {
            "type": "code",
            "code": {
              "input": "# Calculating 2 + 2\nresult = 2 + 2\nresult",
              "outputs": [
                {
                  "type": "logs",
                  "logs": "4"
                }
                        ...
 }
```

Supported files
---------------

|File format|MIME type|
|---|---|
|.c|text/x-c|
|.cs|text/x-csharp|
|.cpp|text/x-c++|
|.csv|text/csv|
|.doc|application/msword|
|.docx|application/vnd.openxmlformats-officedocument.wordprocessingml.document|
|.html|text/html|
|.java|text/x-java|
|.json|application/json|
|.md|text/markdown|
|.pdf|application/pdf|
|.php|text/x-php|
|.pptx|application/vnd.openxmlformats-officedocument.presentationml.presentation|
|.py|text/x-python|
|.py|text/x-script.python|
|.rb|text/x-ruby|
|.tex|text/x-tex|
|.txt|text/plain|
|.css|text/css|
|.js|text/javascript|
|.sh|application/x-sh|
|.ts|application/typescript|
|.csv|application/csv|
|.jpeg|image/jpeg|
|.jpg|image/jpeg|
|.gif|image/gif|
|.pkl|application/octet-stream|
|.png|image/png|
|.tar|application/x-tar|
|.xlsx|application/vnd.openxmlformats-officedocument.spreadsheetml.sheet|
|.xml|application/xml or "text/xml"|
|.zip|application/zip|

===

Assistants Function Calling

Beta

===================================

Similar to the Chat Completions API, the Assistants API supports function calling. Function calling allows you to describe functions to the Assistants API and have it intelligently return the functions that need to be called along with their arguments.

Quickstart
----------

In this example, we'll create a weather assistant and define two functions, `get_current_temperature` and `get_rain_probability`, as tools that the Assistant can call. Depending on the user query, the model will invoke parallel function calling if using our latest models released on or after Nov 6, 2023. In our example that uses parallel function calling, we will ask the Assistant what the weather in San Francisco is like today and the chances of rain. We also show how to output the Assistant's response with streaming.

With the launch of Structured Outputs, you can now use the parameter `strict: true` when using function calling with the Assistants API. For more information, refer to the [Function calling guide](/docs/guides/function-calling#function-calling-with-structured-outputs). Please note that Structured Outputs are not supported in the Assistants API when using vision.

### Step 1: Define functions

When creating your assistant, you will first define the functions under the `tools` param of the assistant.

```python
from openai import OpenAI
client = OpenAI()

assistant = client.beta.assistants.create(
instructions="You are a weather bot. Use the provided functions to answer questions.",
model="gpt-4o",
tools=[
  {
    "type": "function",
    "function": {
      "name": "get_current_temperature",
      "description": "Get the current temperature for a specific location",
      "parameters": {
        "type": "object",
        "properties": {
          "location": {
            "type": "string",
            "description": "The city and state, e.g., San Francisco, CA"
          },
          "unit": {
            "type": "string",
            "enum": ["Celsius", "Fahrenheit"],
            "description": "The temperature unit to use. Infer this from the user's location."
          }
        },
        "required": ["location", "unit"]
      }
    }
  },
  {
    "type": "function",
    "function": {
      "name": "get_rain_probability",
      "description": "Get the probability of rain for a specific location",
      "parameters": {
        "type": "object",
        "properties": {
          "location": {
            "type": "string",
            "description": "The city and state, e.g., San Francisco, CA"
          }
        },
        "required": ["location"]
      }
    }
  }
]
)
```

```javascript
const assistant = await client.beta.assistants.create({
model: "gpt-4o",
instructions:
  "You are a weather bot. Use the provided functions to answer questions.",
tools: [
  {
    type: "function",
    function: {
      name: "getCurrentTemperature",
      description: "Get the current temperature for a specific location",
      parameters: {
        type: "object",
        properties: {
          location: {
            type: "string",
            description: "The city and state, e.g., San Francisco, CA",
          },
          unit: {
            type: "string",
            enum: ["Celsius", "Fahrenheit"],
            description:
              "The temperature unit to use. Infer this from the user's location.",
          },
        },
        required: ["location", "unit"],
      },
    },
  },
  {
    type: "function",
    function: {
      name: "getRainProbability",
      description: "Get the probability of rain for a specific location",
      parameters: {
        type: "object",
        properties: {
          location: {
            type: "string",
            description: "The city and state, e.g., San Francisco, CA",
          },
        },
        required: ["location"],
      },
    },
  },
],
});
```

### Step 2: Create a Thread and add Messages

Create a Thread when a user starts a conversation and add Messages to the Thread as the user asks questions.

```python
thread = client.beta.threads.create()
message = client.beta.threads.messages.create(
thread_id=thread.id,
role="user",
content="What's the weather in San Francisco today and the likelihood it'll rain?",
)
```

```javascript
const thread = await client.beta.threads.create();
const message = client.beta.threads.messages.create(thread.id, {
role: "user",
content: "What's the weather in San Francisco today and the likelihood it'll rain?",
});
```

### Step 3: Initiate a Run

When you initiate a Run on a Thread containing a user Message that triggers one or more functions, the Run will enter a `pending` status. After it processes, the run will enter a `requires_action` state which you can verify by checking the Run’s `status`. This indicates that you need to run tools and submit their outputs to the Assistant to continue Run execution. In our case, we will see two `tool_calls`, which indicates that the user query resulted in parallel function calling.

Note that a runs expire ten minutes after creation. Be sure to submit your tool outputs before the 10 min mark.

You will see two `tool_calls` within `required_action`, which indicates the user query triggered parallel function calling.

```json
{
"id": "run_qJL1kI9xxWlfE0z1yfL0fGg9",
...
"status": "requires_action",
"required_action": {
  "submit_tool_outputs": {
    "tool_calls": [
      {
        "id": "call_FthC9qRpsL5kBpwwyw6c7j4k",
        "function": {
          "arguments": "{"location": "San Francisco, CA"}",
          "name": "get_rain_probability"
        },
        "type": "function"
      },
      {
        "id": "call_RpEDoB8O0FTL9JoKTuCVFOyR",
        "function": {
          "arguments": "{"location": "San Francisco, CA", "unit": "Fahrenheit"}",
          "name": "get_current_temperature"
        },
        "type": "function"
      }
    ]
  },
  ...
  "type": "submit_tool_outputs"
}
}
```

Run object truncated here for readability

  

How you initiate a Run and submit `tool_calls` will differ depending on whether you are using streaming or not, although in both cases all `tool_calls` need to be submitted at the same time. You can then complete the Run by submitting the tool outputs from the functions you called. Pass each `tool_call_id` referenced in the `required_action` object to match outputs to each function call.

With streamingWithout streaming

For the streaming case, we create an EventHandler class to handle events in the response stream and submit all tool outputs at once with the “submit tool outputs stream” helper in the Python and Node SDKs.

```python
from typing_extensions import override
from openai import AssistantEventHandler

class EventHandler(AssistantEventHandler):
  @override
  def on_event(self, event):
    # Retrieve events that are denoted with 'requires_action'
    # since these will have our tool_calls
    if event.event == 'thread.run.requires_action':
      run_id = event.data.id  # Retrieve the run ID from the event data
      self.handle_requires_action(event.data, run_id)

  def handle_requires_action(self, data, run_id):
    tool_outputs = []
      
    for tool in data.required_action.submit_tool_outputs.tool_calls:
      if tool.function.name == "get_current_temperature":
        tool_outputs.append({"tool_call_id": tool.id, "output": "57"})
      elif tool.function.name == "get_rain_probability":
        tool_outputs.append({"tool_call_id": tool.id, "output": "0.06"})
      
    # Submit all tool_outputs at the same time
    self.submit_tool_outputs(tool_outputs, run_id)

  def submit_tool_outputs(self, tool_outputs, run_id):
    # Use the submit_tool_outputs_stream helper
    with client.beta.threads.runs.submit_tool_outputs_stream(
      thread_id=self.current_run.thread_id,
      run_id=self.current_run.id,
      tool_outputs=tool_outputs,
      event_handler=EventHandler(),
    ) as stream:
      for text in stream.text_deltas:
        print(text, end="", flush=True)
      print()

with client.beta.threads.runs.stream(
thread_id=thread.id,
assistant_id=assistant.id,
event_handler=EventHandler()
) as stream:
stream.until_done()
```

```javascript
class EventHandler extends EventEmitter {
constructor(client) {
  super();
  this.client = client;
}

async onEvent(event) {
  try {
    console.log(event);
    // Retrieve events that are denoted with 'requires_action'
    // since these will have our tool_calls
    if (event.event === "thread.run.requires_action") {
      await this.handleRequiresAction(
        event.data,
        event.data.id,
        event.data.thread_id,
      );
    }
  } catch (error) {
    console.error("Error handling event:", error);
  }
}

async handleRequiresAction(data, runId, threadId) {
  try {
    const toolOutputs =
      data.required_action.submit_tool_outputs.tool_calls.map((toolCall) => {
        if (toolCall.function.name === "getCurrentTemperature") {
          return {
            tool_call_id: toolCall.id,
            output: "57",
          };
        } else if (toolCall.function.name === "getRainProbability") {
          return {
            tool_call_id: toolCall.id,
            output: "0.06",
          };
        }
      });
    // Submit all the tool outputs at the same time
    await this.submitToolOutputs(toolOutputs, runId, threadId);
  } catch (error) {
    console.error("Error processing required action:", error);
  }
}

async submitToolOutputs(toolOutputs, runId, threadId) {
  try {
    // Use the submitToolOutputsStream helper
    const stream = this.client.beta.threads.runs.submitToolOutputsStream(
      threadId,
      runId,
      { tool_outputs: toolOutputs },
    );
    for await (const event of stream) {
      this.emit("event", event);
    }
  } catch (error) {
    console.error("Error submitting tool outputs:", error);
  }
}
}

const eventHandler = new EventHandler(client);
eventHandler.on("event", eventHandler.onEvent.bind(eventHandler));

const stream = await client.beta.threads.runs.stream(
threadId,
{ assistant_id: assistantId },
eventHandler,
);

for await (const event of stream) {
eventHandler.emit("event", event);
}
```

### Using Structured Outputs

When you enable [Structured Outputs](/docs/guides/structured-outputs) by supplying `strict: true`, the OpenAI API will pre-process your supplied schema on your first request, and then use this artifact to constrain the model to your schema.

```python
from openai import OpenAI
client = OpenAI()

assistant = client.beta.assistants.create(
instructions="You are a weather bot. Use the provided functions to answer questions.",
model="gpt-4o-2024-08-06",
tools=[
  {
    "type": "function",
    "function": {
      "name": "get_current_temperature",
      "description": "Get the current temperature for a specific location",
      "parameters": {
        "type": "object",
        "properties": {
          "location": {
            "type": "string",
            "description": "The city and state, e.g., San Francisco, CA"
          },
          "unit": {
            "type": "string",
            "enum": ["Celsius", "Fahrenheit"],
            "description": "The temperature unit to use. Infer this from the user's location."
          }
        },
        "required": ["location", "unit"],
        "additionalProperties": False
      },
      "strict": True
    }
  },
  {
    "type": "function",
    "function": {
      "name": "get_rain_probability",
      "description": "Get the probability of rain for a specific location",
      "parameters": {
        "type": "object",
        "properties": {
          "location": {
            "type": "string",
            "description": "The city and state, e.g., San Francisco, CA"
          }
        },
        "required": ["location"],
        "additionalProperties": False
      },
      "strict": True
    }
  }
]
)
```

```javascript
const assistant = await client.beta.assistants.create({
model: "gpt-4o-2024-08-06",
instructions:
  "You are a weather bot. Use the provided functions to answer questions.",
tools: [
  {
    type: "function",
    function: {
      name: "getCurrentTemperature",
      description: "Get the current temperature for a specific location",
      parameters: {
        type: "object",
        properties: {
          location: {
            type: "string",
            description: "The city and state, e.g., San Francisco, CA",
          },
          unit: {
            type: "string",
            enum: ["Celsius", "Fahrenheit"],
            description:
              "The temperature unit to use. Infer this from the user's location.",
          },
        },
        required: ["location", "unit"],
        additionalProperties: false
      },
      strict: true
    },
  },
  {
    type: "function",
    function: {
      name: "getRainProbability",
      description: "Get the probability of rain for a specific location",
      parameters: {
        type: "object",
        properties: {
          location: {
            type: "string",
            description: "The city and state, e.g., San Francisco, CA",
          },
        },
        required: ["location"],
        additionalProperties: false
      },
      strict: true
    },
  },
],
});
```

Audio generation
================

Learn how to generate audio from a text or audio prompt.

In addition to generating [text](/docs/guides/text-generation) and [images](/docs/guides/images), some [models](/docs/models) enable you to generate a spoken audio response to a prompt, and to use audio inputs to prompt the model. Audio inputs can contain richer data than text alone, allowing the model to detect tone, inflection, and other nuances within the input.

You can use these audio capabilities to:

*   Generate a spoken audio summary of a body of text (text in, audio out)
*   Perform sentiment analysis on a recording (audio in, text out)
*   Async speech to speech interactions with a model (audio in, audio out)

OpenAI provides other models for simple [speech to text](/docs/guides/speech-to-text) and [text to speech](/docs/guides/text-to-speech) - when your task requires those conversions (and not dynamic content from a model), the TTS and STT models will be more performant and cost-efficient.

Quickstart
----------

To generate audio or use audio as an input, you can use the [chat completions endpoint](/docs/api-reference/chat/) in the REST API, as seen in the examples below. You can either use the [REST API](/docs/api-reference) from the HTTP client of your choice, or use one of OpenAI's [official SDKs](/docs/libraries) for your preferred programming language.

Audio output from model

Create a human-like audio response to a prompt

```javascript
import { writeFileSync } from "node:fs";
import OpenAI from "openai";

const openai = new OpenAI();

// Generate an audio response to the given prompt
const response = await openai.chat.completions.create({
  model: "gpt-4o-audio-preview",
  modalities: ["text", "audio"],
  audio: { voice: "alloy", format: "wav" },
  messages: [
    {
      role: "user",
      content: "Is a golden retriever a good family dog?"
    }
  ]
});

// Inspect returned data
console.log(response.choices[0]);

// Write audio data to a file
writeFileSync(
  "dog.wav",
  Buffer.from(response.choices[0].message.audio.data, 'base64'),
  { encoding: "utf-8" }
);
```

```python
import base64
from openai import OpenAI

client = OpenAI()

completion = client.chat.completions.create(
    model="gpt-4o-audio-preview",
    modalities=["text", "audio"],
    audio={"voice": "alloy", "format": "wav"},
    messages=[
        {
            "role": "user",
            "content": "Is a golden retriever a good family dog?"
        }
    ]
)

print(completion.choices[0])

wav_bytes = base64.b64decode(completion.choices[0].message.audio.data)
with open("dog.wav", "wb") as f:
    f.write(wav_bytes)
```

```bash
curl "https://api.openai.com/v1/chat/completions" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $OPENAI_API_KEY" \
    -d '{
      "model": "gpt-4o-audio-preview",
      "modalities": ["text", "audio"],
      "audio": { "voice": "alloy", "format": "wav" },
      "messages": [
        {
          "role": "user",
          "content": "Is a golden retriever a good family dog?"
        }
      ]
    }'
```

Audio input to model

Use audio inputs for prompting a model

```javascript
import OpenAI from "openai";
const openai = new OpenAI();

// Fetch an audio file and convert it to a base64 string
const url = "https://openaiassets.blob.core.windows.net/$web/API/docs/audio/alloy.wav";
const audioResponse = await fetch(url);
const buffer = await audioResponse.arrayBuffer();
const base64str = Buffer.from(buffer).toString("base64");

const response = await openai.chat.completions.create({
  model: "gpt-4o-audio-preview",
  modalities: ["text", "audio"],
  audio: { voice: "alloy", format: "wav" },
  messages: [
    {
      role: "user",
      content: [
        { type: "text", text: "What is in this recording?" },
        { type: "input_audio", input_audio: { data: base64str, format: "wav" }}
      ]
    }
  ]
});

console.log(response.choices[0]);
```

```python
import base64
import requests
from openai import OpenAI

client = OpenAI()

# Fetch the audio file and convert it to a base64 encoded string
url = "https://openaiassets.blob.core.windows.net/$web/API/docs/audio/alloy.wav"
response = requests.get(url)
response.raise_for_status()
wav_data = response.content
encoded_string = base64.b64encode(wav_data).decode('utf-8')

completion = client.chat.completions.create(
    model="gpt-4o-audio-preview",
    modalities=["text", "audio"],
    audio={"voice": "alloy", "format": "wav"},
    messages=[
        {
            "role": "user",
            "content": [
                { 
                    "type": "text",
                    "text": "What is in this recording?"
                },
                {
                    "type": "input_audio",
                    "input_audio": {
                        "data": encoded_string,
                        "format": "wav"
                    }
                }
            ]
        },
    ]
)

print(completion.choices[0].message)
```

```bash
curl "https://api.openai.com/v1/chat/completions" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $OPENAI_API_KEY" \
    -d '{
      "model": "gpt-4o-audio-preview",
      "modalities": ["text", "audio"],
      "audio": { "voice": "alloy", "format": "wav" },
      "messages": [
        {
          "role": "user",
          "content": [
            { "type": "text", "text": "What is in this recording?" },
            { 
              "type": "input_audio", 
              "input_audio": { 
                "data": "<base64 bytes here>", 
                "format": "wav" 
              }
            }
          ]
        }
      ]
    }'
```

Multi-turn conversations
------------------------

Using audio outputs from the model as inputs to multi-turn conversations requires a generated ID that appears in the response data for an audio generation. Below is an example JSON data structure for a [message you might receive](/docs/api-reference/chat/object#chat/object-choices) from `/chat/completions`:

```json
{
  "index": 0,
  "message": {
    "role": "assistant",
    "content": null,
    "refusal": null,
    "audio": {
      "id": "audio_abc123",
      "expires_at": 1729018505,
      "data": "<bytes omitted>",
      "transcript": "Yes, golden retrievers are known to be ..."
    }
  },
  "finish_reason": "stop"
}
```

The value of `message.audio.id` above provides an identifier you can use in an `assistant` message for a new `/chat/completions` request, as in the example below.

```bash
curl "https://api.openai.com/v1/chat/completions" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $OPENAI_API_KEY" \
    -d '{
        "model": "gpt-4o-audio-preview",
        "modalities": ["text", "audio"],
        "audio": { "voice": "alloy", "format": "wav" },
        "messages": [
            {
                "role": "user",
                "content": "Is a golden retriever a good family dog?"
            },
            {
                "role": "assistant",
                "audio": {
                    "id": "audio_abc123"
                }
            },
            {
                "role": "user",
                "content": "Why do you say they are loyal?"
            }
        ]
    }'
```

FAQ
---

### What modalities are supported by gpt-4o-audio-preview

`gpt-4o-audio-preview` requires either audio output or audio input to be used at this time. Acceptable combinations of input and output are:

*   text in → text + audio out
*   audio in → text + audio out
*   audio in → text out
*   text + audio in → text + audio out
*   text + audio in → text out

### How is audio in Chat Completions different from the Realtime API?

The underlying GPT-4o audio model is exactly the same. The Realtime API operates the same model at lower latency.

### How do I think about audio input to the model in terms of tokens?

We are working on better tooling to expose this, but roughly one hour of audio input is equal to 128k tokens, the max context window currently supported by this model.

### How do I control which output modalities I receive?

Currently the model only programmatically allows modalities = `[“text”, “audio”]`. In the future, this parameter will give more controls.

### How does tool/function calling work?

Tool (and function) calling works the same as it does for other models in Chat Completions - [learn more](/docs/guides/function-calling).

Next steps
----------

Now that you know how to generate audio outputs and send audio inputs, there are a few other techniques you might want to master.

[

Text to speech

Use a specialized model to turn text into speech.

](/docs/guides/text-to-speech)[

Speech to text

Use a specialized model to turn audio files with speech into text.

](/docs/guides/speech-to-text)[

Realtime API

Learn to use the Realtime API to prompt a model over a WebSocket.

](/docs/guides/realtime)[

Full API reference

Check out all the options for audio generation in the API reference.

](/docs/api-reference/chat)

Batch API
=========

Process jobs asynchronously with Batch API.

Learn how to use OpenAI's Batch API to send asynchronous groups of requests with 50% lower costs, a separate pool of significantly higher rate limits, and a clear 24-hour turnaround time. The service is ideal for processing jobs that don't require immediate responses. You can also [explore the API reference directly here](/docs/api-reference/batch).

Overview
--------

While some uses of the OpenAI Platform require you to send synchronous requests, there are many cases where requests do not need an immediate response or [rate limits](/docs/guides/rate-limits) prevent you from executing a large number of queries quickly. Batch processing jobs are often helpful in use cases like:

1.  running evaluations
2.  classifying large datasets
3.  embedding content repositories

The Batch API offers a straightforward set of endpoints that allow you to collect a set of requests into a single file, kick off a batch processing job to execute these requests, query for the status of that batch while the underlying requests execute, and eventually retrieve the collected results when the batch is complete.

Compared to using standard endpoints directly, Batch API has:

1.  **Better cost efficiency:** 50% cost discount compared to synchronous APIs
2.  **Higher rate limits:** [Substantially more headroom](/settings/organization/limits) compared to the synchronous APIs
3.  **Fast completion times:** Each batch completes within 24 hours (and often more quickly)

Getting Started
---------------

### 1\. Preparing Your Batch File

Batches start with a `.jsonl` file where each line contains the details of an individual request to the API. For now, the available endpoints are `/v1/chat/completions` ([Chat Completions API](/docs/api-reference/chat)) and `/v1/embeddings` ([Embeddings API](/docs/api-reference/embeddings)). For a given input file, the parameters in each line's `body` field are the same as the parameters for the underlying endpoint. Each request must include a unique `custom_id` value, which you can use to reference results after completion. Here's an example of an input file with 2 requests. Note that each input file can only include requests to a single model.

```jsonl
{"custom_id": "request-1", "method": "POST", "url": "/v1/chat/completions", "body": {"model": "gpt-3.5-turbo-0125", "messages": [{"role": "system", "content": "You are a helpful assistant."},{"role": "user", "content": "Hello world!"}],"max_tokens": 1000}}
{"custom_id": "request-2", "method": "POST", "url": "/v1/chat/completions", "body": {"model": "gpt-3.5-turbo-0125", "messages": [{"role": "system", "content": "You are an unhelpful assistant."},{"role": "user", "content": "Hello world!"}],"max_tokens": 1000}}
```

### 2\. Uploading Your Batch Input File

Similar to our [Fine-tuning API](/docs/guides/fine-tuning), you must first upload your input file so that you can reference it correctly when kicking off batches. Upload your `.jsonl` file using the [Files API](/docs/api-reference/files).

Upload files for Batch API

```javascript
import fs from "fs";
import OpenAI from "openai";
const openai = new OpenAI();

const file = await openai.files.create({
  file: fs.createReadStream("batchinput.jsonl"),
  purpose: "batch",
});

console.log(file);
```

```python
from openai import OpenAI
client = OpenAI()

batch_input_file = client.files.create(
    file=open("batchinput.jsonl", "rb"),
    purpose="batch"
)

print(batch_input_file)
```

```bash
curl https://api.openai.com/v1/files \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -F purpose="batch" \
  -F file="@batchinput.jsonl"
```

### 3\. Creating the Batch

Once you've successfully uploaded your input file, you can use the input File object's ID to create a batch. In this case, let's assume the file ID is `file-abc123`. For now, the completion window can only be set to `24h`. You can also provide custom metadata via an optional `metadata` parameter.

Create the Batch

```javascript
import OpenAI from "openai";
const openai = new OpenAI();

const batch = await openai.batches.create({
  input_file_id: "file-abc123",
  endpoint: "/v1/chat/completions",
  completion_window: "24h"
});

console.log(batch);
```

```python
from openai import OpenAI
client = OpenAI()

batch_input_file_id = batch_input_file.id
client.batches.create(
    input_file_id=batch_input_file_id,
    endpoint="/v1/chat/completions",
    completion_window="24h",
    metadata={
        "description": "nightly eval job"
    }
)
```

```bash
curl https://api.openai.com/v1/batches \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "input_file_id": "file-abc123",
    "endpoint": "/v1/chat/completions",
    "completion_window": "24h"
  }'
```

This request will return a [Batch object](/docs/api-reference/batch/object) with metadata about your batch:

```python
{
  "id": "batch_abc123",
  "object": "batch",
  "endpoint": "/v1/chat/completions",
  "errors": null,
  "input_file_id": "file-abc123",
  "completion_window": "24h",
  "status": "validating",
  "output_file_id": null,
  "error_file_id": null,
  "created_at": 1714508499,
  "in_progress_at": null,
  "expires_at": 1714536634,
  "completed_at": null,
  "failed_at": null,
  "expired_at": null,
  "request_counts": {
    "total": 0,
    "completed": 0,
    "failed": 0
  },
  "metadata": null
}
```

### 4\. Checking the Status of a Batch

You can check the status of a batch at any time, which will also return a Batch object.

Check the status of a batch

```javascript
import OpenAI from "openai";
const openai = new OpenAI();

const batch = await openai.batches.retrieve("batch_abc123");
console.log(batch);
```

```python
from openai import OpenAI
client = OpenAI()

const batch = client.batches.retrieve("batch_abc123")
print(batch)
```

```bash
curl https://api.openai.com/v1/batches/batch_abc123 \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json"
```

The status of a given Batch object can be any of the following:

|Status|Description|
|---|---|
|validating|the input file is being validated before the batch can begin|
|failed|the input file has failed the validation process|
|in_progress|the input file was successfully validated and the batch is currently being run|
|finalizing|the batch has completed and the results are being prepared|
|completed|the batch has been completed and the results are ready|
|expired|the batch was not able to be completed within the 24-hour time window|
|cancelling|the batch is being cancelled (may take up to 10 minutes)|
|cancelled|the batch was cancelled|

### 5\. Retrieving the Results

Once the batch is complete, you can download the output by making a request against the [Files API](/docs/api-reference/files) via the `output_file_id` field from the Batch object and writing it to a file on your machine, in this case `batch_output.jsonl`

Retrieving the batch results

```javascript
import OpenAI from "openai";
const openai = new OpenAI();

const fileResponse = await openai.files.content("file-xyz123");
const fileContents = await fileResponse.text();

console.log(fileContents);
```

```python
from openai import OpenAI
client = OpenAI()

file_response = client.files.content("file-xyz123")
print(file_response.text)
```

```bash
curl https://api.openai.com/v1/files/file-xyz123/content \
  -H "Authorization: Bearer $OPENAI_API_KEY" > batch_output.jsonl
```

The output `.jsonl` file will have one response line for every successful request line in the input file. Any failed requests in the batch will have their error information written to an error file that can be found via the batch's `error_file_id`.

Note that the output line order **may not match** the input line order. Instead of relying on order to process your results, use the custom\_id field which will be present in each line of your output file and allow you to map requests in your input to results in your output.

```jsonl
{"id": "batch_req_123", "custom_id": "request-2", "response": {"status_code": 200, "request_id": "req_123", "body": {"id": "chatcmpl-123", "object": "chat.completion", "created": 1711652795, "model": "gpt-3.5-turbo-0125", "choices": [{"index": 0, "message": {"role": "assistant", "content": "Hello."}, "logprobs": null, "finish_reason": "stop"}], "usage": {"prompt_tokens": 22, "completion_tokens": 2, "total_tokens": 24}, "system_fingerprint": "fp_123"}}, "error": null}
{"id": "batch_req_456", "custom_id": "request-1", "response": {"status_code": 200, "request_id": "req_789", "body": {"id": "chatcmpl-abc", "object": "chat.completion", "created": 1711652789, "model": "gpt-3.5-turbo-0125", "choices": [{"index": 0, "message": {"role": "assistant", "content": "Hello! How can I assist you today?"}, "logprobs": null, "finish_reason": "stop"}], "usage": {"prompt_tokens": 20, "completion_tokens": 9, "total_tokens": 29}, "system_fingerprint": "fp_3ba"}}, "error": null}
```

### 6\. Cancelling a Batch

If necessary, you can cancel an ongoing batch. The batch's status will change to `cancelling` until in-flight requests are complete (up to 10 minutes), after which the status will change to `cancelled`.

Cancelling a batch

```javascript
import OpenAI from "openai";
const openai = new OpenAI();

const batch = await openai.batches.cancel("batch_abc123");
console.log(batch);
```

```python
from openai import OpenAI
client = OpenAI()

client.batches.cancel("batch_abc123")
```

```bash
curl https://api.openai.com/v1/batches/batch_abc123/cancel \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -X POST
```

### 7\. Getting a List of All Batches

At any time, you can see all your batches. For users with many batches, you can use the `limit` and `after` parameters to paginate your results.

Getting a list of all batches

```javascript
import OpenAI from "openai";
const openai = new OpenAI();

const list = await openai.batches.list();

for await (const batch of list) {
  console.log(batch);
}
```

```python
from openai import OpenAI
client = OpenAI()

client.batches.list(limit=10)
```

```bash
curl https://api.openai.com/v1/batches?limit=10 \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json"
```

Model Availability
------------------

The Batch API can currently be used to execute queries against the following models. The Batch API supports text and vision inputs in the same format as the endpoints for these models:

*   `gpt-4o`
*   `gpt-4o-2024-08-06`
*   `gpt-4o-mini`
*   `gpt-4-turbo`
*   `gpt-4`
*   `gpt-4-32k`
*   `gpt-3.5-turbo`
*   `gpt-3.5-turbo-16k`
*   `gpt-4-turbo-preview`
*   `gpt-4-vision-preview`
*   `gpt-4-turbo-2024-04-09`
*   `gpt-4-0314`
*   `gpt-4-32k-0314`
*   `gpt-4-32k-0613`
*   `gpt-3.5-turbo-0301`
*   `gpt-3.5-turbo-16k-0613`
*   `gpt-3.5-turbo-1106`
*   `gpt-3.5-turbo-0613`
*   `text-embedding-3-large`
*   `text-embedding-3-small`
*   `text-embedding-ada-002`

The Batch API also supports [fine-tuned models](/docs/guides/fine-tuning#what-models-can-be-fine-tuned).

Rate Limits
-----------

Batch API rate limits are separate from existing per-model rate limits. The Batch API has two new types of rate limits:

1.  **Per-batch limits:** A single batch may include up to 50,000 requests, and a batch input file can be up to 200 MB in size. Note that `/v1/embeddings` batches are also restricted to a maximum of 50,000 embedding inputs across all requests in the batch.
2.  **Enqueued prompt tokens per model:** Each model has a maximum number of enqueued prompt tokens allowed for batch processing. You can find these limits on the [Platform Settings page](/settings/organization/limits).

There are no limits for output tokens or number of submitted requests for the Batch API today. Because Batch API rate limits are a new, separate pool, **using the Batch API will not consume tokens from your standard per-model rate limits**, thereby offering you a convenient way to increase the number of requests and processed tokens you can use when querying our API.

Batch Expiration
----------------

Batches that do not complete in time eventually move to an `expired` state; unfinished requests within that batch are cancelled, and any responses to completed requests are made available via the batch's output file. You will be charged for tokens consumed from any completed requests.

Expired requests will be written to your error file with the message as shown below. You can use the `custom_id` to retrieve the request data for expired requests.

```jsonl
{"id": "batch_req_123", "custom_id": "request-3", "response": null, "error": {"code": "batch_expired", "message": "This request could not be executed before the completion window expired."}}
{"id": "batch_req_123", "custom_id": "request-7", "response": null, "error": {"code": "batch_expired", "message": "This request could not be executed before the completion window expired."}}
```

Other Resources
---------------

For more concrete examples, visit **[the OpenAI Cookbook](https://cookbook.openai.com/examples/batch_processing)**, which contains sample code for use cases like classification, sentiment analysis, and summary generation.

Vector embeddings
=================

Learn how to turn text into numbers, unlocking use cases like search.

**New embedding models**

`text-embedding-3-small` and `text-embedding-3-large`, our newest and most performant embedding models are now available, with lower costs, higher multilingual performance, and new parameters to control the overall size.

What are embeddings?
--------------------

OpenAI’s text embeddings measure the relatedness of text strings. Embeddings are commonly used for:

*   **Search** (where results are ranked by relevance to a query string)
*   **Clustering** (where text strings are grouped by similarity)
*   **Recommendations** (where items with related text strings are recommended)
*   **Anomaly detection** (where outliers with little relatedness are identified)
*   **Diversity measurement** (where similarity distributions are analyzed)
*   **Classification** (where text strings are classified by their most similar label)

An embedding is a vector (list) of floating point numbers. The [distance](#which-distance-function-should-i-use) between two vectors measures their relatedness. Small distances suggest high relatedness and large distances suggest low relatedness.

Visit our [pricing page](https://openai.com/api/pricing/) to learn about Embeddings pricing. Requests are billed based on the number of [tokens](/tokenizer) in the [input](/docs/api-reference/embeddings/create#embeddings/create-input).

How to get embeddings
---------------------

To get an embedding, send your text string to the [embeddings API endpoint](/docs/api-reference/embeddings) along with the embedding model name (e.g. `text-embedding-3-small`). The response will contain an embedding (list of floating point numbers), which you can extract, save in a vector database, and use for many different use cases:

Example: Getting embeddings

```javascript
import OpenAI from "openai";
const openai = new OpenAI();

const embedding = await openai.embeddings.create({
  model: "text-embedding-3-small",
  input: "Your text string goes here",
  encoding_format: "float",
});

console.log(embedding);
```

```python
from openai import OpenAI
client = OpenAI()

response = client.embeddings.create(
    input="Your text string goes here",
    model="text-embedding-3-small"
)

print(response.data[0].embedding)
```

```bash
curl https://api.openai.com/v1/embeddings \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -d '{
    "input": "Your text string goes here",
    "model": "text-embedding-3-small"
  }'
```

The response will contain the embedding vector along with some additional metadata.

```json
{
  "object": "list",
  "data": [
    {
      "object": "embedding",
      "index": 0,
      "embedding": [
        -0.006929283495992422,
        -0.005336422007530928,
        -4.547132266452536e-05,
        -0.024047505110502243
      ],
    }
  ],
  "model": "text-embedding-3-small",
  "usage": {
    "prompt_tokens": 5,
    "total_tokens": 5
  }
}
```

By default, the length of the embedding vector will be 1536 for `text-embedding-3-small` or 3072 for `text-embedding-3-large`. You can reduce the dimensions of the embedding by passing in the [dimensions parameter](/docs/api-reference/embeddings/create#embeddings-create-dimensions) without the embedding losing its concept-representing properties. We go into more detail on embedding dimensions in the [embedding use case section](#use-cases).

Embedding models
----------------

OpenAI offers two powerful third-generation embedding model (denoted by `-3` in the model ID). You can read the embedding v3 [announcement blog post](https://openai.com/blog/new-embedding-models-and-api-updates) for more details.

Usage is priced per input token, below is an example of pricing pages of text per US dollar (assuming ~800 tokens per page):

|Model|~ Pages per dollar|Performance on MTEB eval|Max input|
|---|---|---|---|
|text-embedding-3-small|62,500|62.3%|8191|
|text-embedding-3-large|9,615|64.6%|8191|
|text-embedding-ada-002|12,500|61.0%|8191|

Use cases
---------

Here we show some representative use cases. We will use the [Amazon fine-food reviews dataset](https://www.kaggle.com/snap/amazon-fine-food-reviews) for the following examples.

### Obtaining the embeddings

The dataset contains a total of 568,454 food reviews Amazon users left up to October 2012. We will use a subset of 1,000 most recent reviews for illustration purposes. The reviews are in English and tend to be positive or negative. Each review has a ProductId, UserId, Score, review title (Summary) and review body (Text). For example:

|Product Id|User Id|Score|Summary|Text|
|---|---|---|---|---|
|B001E4KFG0|A3SGXH7AUHU8GW|5|Good Quality Dog Food|I have bought several of the Vitality canned...|
|B00813GRG4|A1D87F6ZCVE5NK|1|Not as Advertised|Product arrived labeled as Jumbo Salted Peanut...|

We will combine the review summary and review text into a single combined text. The model will encode this combined text and output a single vector embedding.

[

Get\_embeddings\_from\_dataset.ipynb

](https://cookbook.openai.com/examples/get_embeddings_from_dataset)

```python
from openai import OpenAI
client = OpenAI()

def get_embedding(text, model="text-embedding-3-small"):
    text = text.replace("\n", " ")
    return client.embeddings.create(input = [text], model=model).data[0].embedding

df['ada_embedding'] = df.combined.apply(lambda x: get_embedding(x, model='text-embedding-3-small'))
df.to_csv('output/embedded_1k_reviews.csv', index=False)
```

To load the data from a saved file, you can run the following:

```python
import pandas as pd

df = pd.read_csv('output/embedded_1k_reviews.csv')
df['ada_embedding'] = df.ada_embedding.apply(eval).apply(np.array)
```

Reducing embedding dimensions

Using larger embeddings, for example storing them in a vector store for retrieval, generally costs more and consumes more compute, memory and storage than using smaller embeddings.

Both of our new embedding models were trained [with a technique](https://arxiv.org/abs/2205.13147) that allows developers to trade-off performance and cost of using embeddings. Specifically, developers can shorten embeddings (i.e. remove some numbers from the end of the sequence) without the embedding losing its concept-representing properties by passing in the [`dimensions` API parameter](/docs/api-reference/embeddings/create#embeddings-create-dimensions). For example, on the MTEB benchmark, a `text-embedding-3-large` embedding can be shortened to a size of 256 while still outperforming an unshortened `text-embedding-ada-002` embedding with a size of 1536. You can read more about how changing the dimensions impacts performance in our [embeddings v3 launch blog post](https://openai.com/blog/new-embedding-models-and-api-updates#:~:text=Native%20support%20for%20shortening%20embeddings).

In general, using the `dimensions` parameter when creating the embedding is the suggested approach. In certain cases, you may need to change the embedding dimension after you generate it. When you change the dimension manually, you need to be sure to normalize the dimensions of the embedding as is shown below.

```python
from openai import OpenAI
import numpy as np

client = OpenAI()

def normalize_l2(x):
    x = np.array(x)
    if x.ndim == 1:
        norm = np.linalg.norm(x)
        if norm == 0:
            return x
        return x / norm
    else:
        norm = np.linalg.norm(x, 2, axis=1, keepdims=True)
        return np.where(norm == 0, x, x / norm)

response = client.embeddings.create(
    model="text-embedding-3-small", input="Testing 123", encoding_format="float"
)

cut_dim = response.data[0].embedding[:256]
norm_dim = normalize_l2(cut_dim)

print(norm_dim)
```

Dynamically changing the dimensions enables very flexible usage. For example, when using a vector data store that only supports embeddings up to 1024 dimensions long, developers can now still use our best embedding model `text-embedding-3-large` and specify a value of 1024 for the `dimensions` API parameter, which will shorten the embedding down from 3072 dimensions, trading off some accuracy in exchange for the smaller vector size.

Question answering using embeddings-based search

[Question\_answering\_using\_embeddings.ipynb](https://cookbook.openai.com/examples/question_answering_using_embeddings)

There are many common cases where the model is not trained on data which contains key facts and information you want to make accessible when generating responses to a user query. One way of solving this, as shown below, is to put additional information into the context window of the model. This is effective in many use cases but leads to higher token costs. In this notebook, we explore the tradeoff between this approach and embeddings bases search.

```python
query = f"""Use the below article on the 2022 Winter Olympics to answer the subsequent question. If the answer cannot be found, write "I don't know."

Article:
\"\"\"
{wikipedia_article_on_curling}
\"\"\"

Question: Which athletes won the gold medal in curling at the 2022 Winter Olympics?"""

response = client.chat.completions.create(
    messages=[
        {'role': 'system', 'content': 'You answer questions about the 2022 Winter Olympics.'},
        {'role': 'user', 'content': query},
    ],
    model=GPT_MODEL,
    temperature=0,
)

print(response.choices[0].message.content)
```

Text search using embeddings

[Semantic\_text\_search\_using\_embeddings.ipynb](https://cookbook.openai.com/examples/semantic_text_search_using_embeddings)

To retrieve the most relevant documents we use the cosine similarity between the embedding vectors of the query and each document, and return the highest scored documents.

```python
from openai.embeddings_utils import get_embedding, cosine_similarity

def search_reviews(df, product_description, n=3, pprint=True):
    embedding = get_embedding(product_description, model='text-embedding-3-small')
    df['similarities'] = df.ada_embedding.apply(lambda x: cosine_similarity(x, embedding))
    res = df.sort_values('similarities', ascending=False).head(n)
    return res

res = search_reviews(df, 'delicious beans', n=3)
```

Code search using embeddings

[Code\_search.ipynb](https://cookbook.openai.com/examples/code_search_using_embeddings)

Code search works similarly to embedding-based text search. We provide a method to extract Python functions from all the Python files in a given repository. Each function is then indexed by the `text-embedding-3-small` model.

To perform a code search, we embed the query in natural language using the same model. Then we calculate cosine similarity between the resulting query embedding and each of the function embeddings. The highest cosine similarity results are most relevant.

```python
from openai.embeddings_utils import get_embedding, cosine_similarity

df['code_embedding'] = df['code'].apply(lambda x: get_embedding(x, model='text-embedding-3-small'))

def search_functions(df, code_query, n=3, pprint=True, n_lines=7):
    embedding = get_embedding(code_query, model='text-embedding-3-small')
    df['similarities'] = df.code_embedding.apply(lambda x: cosine_similarity(x, embedding))

    res = df.sort_values('similarities', ascending=False).head(n)
    return res

res = search_functions(df, 'Completions API tests', n=3)
```

Recommendations using embeddings

[Recommendation\_using\_embeddings.ipynb](https://cookbook.openai.com/examples/recommendation_using_embeddings)

Because shorter distances between embedding vectors represent greater similarity, embeddings can be useful for recommendation.

Below, we illustrate a basic recommender. It takes in a list of strings and one 'source' string, computes their embeddings, and then returns a ranking of the strings, ranked from most similar to least similar. As a concrete example, the linked notebook below applies a version of this function to the [AG news dataset](http://groups.di.unipi.it/~gulli/AG_corpus_of_news_articles.html) (sampled down to 2,000 news article descriptions) to return the top 5 most similar articles to any given source article.

```python
def recommendations_from_strings(
    strings: List[str],
    index_of_source_string: int,
    model="text-embedding-3-small",
) -> List[int]:
    """Return nearest neighbors of a given string."""

    # get embeddings for all strings
    embeddings = [embedding_from_string(string, model=model) for string in strings]

    # get the embedding of the source string
    query_embedding = embeddings[index_of_source_string]

    # get distances between the source embedding and other embeddings (function from embeddings_utils.py)
    distances = distances_from_embeddings(query_embedding, embeddings, distance_metric="cosine")

    # get indices of nearest neighbors (function from embeddings_utils.py)
    indices_of_nearest_neighbors = indices_of_nearest_neighbors_from_distances(distances)
    return indices_of_nearest_neighbors
```

Data visualization in 2D

[Visualizing\_embeddings\_in\_2D.ipynb](https://cookbook.openai.com/examples/visualizing_embeddings_in_2d)

The size of the embeddings varies with the complexity of the underlying model. In order to visualize this high dimensional data we use the t-SNE algorithm to transform the data into two dimensions.

We color the individual reviews based on the star rating which the reviewer has given:

*   1-star: red
*   2-star: dark orange
*   3-star: gold
*   4-star: turquoise
*   5-star: dark green

![Amazon ratings visualized in language using t-SNE](https://cdn.openai.com/API/docs/images/embeddings-tsne.png)

The visualization seems to have produced roughly 3 clusters, one of which has mostly negative reviews.

```python
import pandas as pd
from sklearn.manifold import TSNE
import matplotlib.pyplot as plt
import matplotlib

df = pd.read_csv('output/embedded_1k_reviews.csv')
matrix = df.ada_embedding.apply(eval).to_list()

# Create a t-SNE model and transform the data
tsne = TSNE(n_components=2, perplexity=15, random_state=42, init='random', learning_rate=200)
vis_dims = tsne.fit_transform(matrix)

colors = ["red", "darkorange", "gold", "turquiose", "darkgreen"]
x = [x for x,y in vis_dims]
y = [y for x,y in vis_dims]
color_indices = df.Score.values - 1

colormap = matplotlib.colors.ListedColormap(colors)
plt.scatter(x, y, c=color_indices, cmap=colormap, alpha=0.3)
plt.title("Amazon ratings visualized in language using t-SNE")
```

Embedding as a text feature encoder for ML algorithms

[Regression\_using\_embeddings.ipynb](https://cookbook.openai.com/examples/regression_using_embeddings)

An embedding can be used as a general free-text feature encoder within a machine learning model. Incorporating embeddings will improve the performance of any machine learning model, if some of the relevant inputs are free text. An embedding can also be used as a categorical feature encoder within a ML model. This adds most value if the names of categorical variables are meaningful and numerous, such as job titles. Similarity embeddings generally perform better than search embeddings for this task.

We observed that generally the embedding representation is very rich and information dense. For example, reducing the dimensionality of the inputs using SVD or PCA, even by 10%, generally results in worse downstream performance on specific tasks.

This code splits the data into a training set and a testing set, which will be used by the following two use cases, namely regression and classification.

```python
from sklearn.model_selection import train_test_split

X_train, X_test, y_train, y_test = train_test_split(
    list(df.ada_embedding.values),
    df.Score,
    test_size = 0.2,
    random_state=42
)
```

#### Regression using the embedding features

Embeddings present an elegant way of predicting a numerical value. In this example we predict the reviewer’s star rating, based on the text of their review. Because the semantic information contained within embeddings is high, the prediction is decent even with very few reviews.

We assume the score is a continuous variable between 1 and 5, and allow the algorithm to predict any floating point value. The ML algorithm minimizes the distance of the predicted value to the true score, and achieves a mean absolute error of 0.39, which means that on average the prediction is off by less than half a star.

```python
from sklearn.ensemble import RandomForestRegressor

rfr = RandomForestRegressor(n_estimators=100)
rfr.fit(X_train, y_train)
preds = rfr.predict(X_test)
```

Classification using the embedding features

[Classification\_using\_embeddings.ipynb](https://cookbook.openai.com/examples/classification_using_embeddings)

This time, instead of having the algorithm predict a value anywhere between 1 and 5, we will attempt to classify the exact number of stars for a review into 5 buckets, ranging from 1 to 5 stars.

After the training, the model learns to predict 1 and 5-star reviews much better than the more nuanced reviews (2-4 stars), likely due to more extreme sentiment expression.

```python
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import classification_report, accuracy_score

clf = RandomForestClassifier(n_estimators=100)
clf.fit(X_train, y_train)
preds = clf.predict(X_test)
```

Zero-shot classification

[Zero-shot\_classification\_with\_embeddings.ipynb](https://cookbook.openai.com/examples/zero-shot_classification_with_embeddings)

We can use embeddings for zero shot classification without any labeled training data. For each class, we embed the class name or a short description of the class. To classify some new text in a zero-shot manner, we compare its embedding to all class embeddings and predict the class with the highest similarity.

```python
from openai.embeddings_utils import cosine_similarity, get_embedding

df= df[df.Score!=3]
df['sentiment'] = df.Score.replace({1:'negative', 2:'negative', 4:'positive', 5:'positive'})

labels = ['negative', 'positive']
label_embeddings = [get_embedding(label, model=model) for label in labels]

def label_score(review_embedding, label_embeddings):
    return cosine_similarity(review_embedding, label_embeddings[1]) - cosine_similarity(review_embedding, label_embeddings[0])

prediction = 'positive' if label_score('Sample Review', label_embeddings) > 0 else 'negative'
```

Obtaining user and product embeddings for cold-start recommendation

[User\_and\_product\_embeddings.ipynb](https://cookbook.openai.com/examples/user_and_product_embeddings)

We can obtain a user embedding by averaging over all of their reviews. Similarly, we can obtain a product embedding by averaging over all the reviews about that product. In order to showcase the usefulness of this approach we use a subset of 50k reviews to cover more reviews per user and per product.

We evaluate the usefulness of these embeddings on a separate test set, where we plot similarity of the user and product embedding as a function of the rating. Interestingly, based on this approach, even before the user receives the product we can predict better than random whether they would like the product.

![Boxplot grouped by Score](https://cdn.openai.com/API/docs/images/embeddings-boxplot.png)

```python
user_embeddings = df.groupby('UserId').ada_embedding.apply(np.mean)
prod_embeddings = df.groupby('ProductId').ada_embedding.apply(np.mean)
```

Clustering

[Clustering.ipynb](https://cookbook.openai.com/examples/clustering)

Clustering is one way of making sense of a large volume of textual data. Embeddings are useful for this task, as they provide semantically meaningful vector representations of each text. Thus, in an unsupervised way, clustering will uncover hidden groupings in our dataset.

In this example, we discover four distinct clusters: one focusing on dog food, one on negative reviews, and two on positive reviews.

![Clusters identified visualized in language 2d using t-SNE](https://cdn.openai.com/API/docs/images/embeddings-cluster.png)

```python
import numpy as np
from sklearn.cluster import KMeans

matrix = np.vstack(df.ada_embedding.values)
n_clusters = 4

kmeans = KMeans(n_clusters = n_clusters, init='k-means++', random_state=42)
kmeans.fit(matrix)
df['Cluster'] = kmeans.labels_
```

FAQ
---

### How can I tell how many tokens a string has before I embed it?

In Python, you can split a string into tokens with OpenAI's tokenizer [`tiktoken`](https://github.com/openai/tiktoken).

Example code:

```python
import tiktoken

def num_tokens_from_string(string: str, encoding_name: str) -> int:
    """Returns the number of tokens in a text string."""
    encoding = tiktoken.get_encoding(encoding_name)
    num_tokens = len(encoding.encode(string))
    return num_tokens

num_tokens_from_string("tiktoken is great!", "cl100k_base")
```

For third-generation embedding models like `text-embedding-3-small`, use the `cl100k_base` encoding.

More details and example code are in the OpenAI Cookbook guide [how to count tokens with tiktoken](https://cookbook.openai.com/examples/how_to_count_tokens_with_tiktoken).

### How can I retrieve K nearest embedding vectors quickly?

For searching over many vectors quickly, we recommend using a vector database. You can find examples of working with vector databases and the OpenAI API [in our Cookbook](https://cookbook.openai.com/examples/vector_databases/readme) on GitHub.

### Which distance function should I use?

We recommend [cosine similarity](https://en.wikipedia.org/wiki/Cosine_similarity). The choice of distance function typically doesn't matter much.

OpenAI embeddings are normalized to length 1, which means that:

*   Cosine similarity can be computed slightly faster using just a dot product
*   Cosine similarity and Euclidean distance will result in the identical rankings

### Can I share my embeddings online?

Yes, customers own their input and output from our models, including in the case of embeddings. You are responsible for ensuring that the content you input to our API does not violate any applicable law or our [Terms of Use](https://openai.com/policies/terms-of-use).

### Do V3 embedding models know about recent events?

No, the `text-embedding-3-large` and `text-embedding-3-small` models lack knowledge of events that occurred after September 2021. This is generally not as much of a limitation as it would be for text generation models but in certain edge cases it can reduce performance.

Fine-tuning
===========

Fine-tune models for better results and efficiency.

Fine-tuning lets you get more out of the models available through the API by providing:

*   Higher quality results than prompting
*   Ability to train on more examples than can fit in a prompt
*   Token savings due to shorter prompts
*   Lower latency requests

OpenAI's text generation models have been pre-trained on a vast amount of text. To use the models effectively, we include instructions and sometimes several examples in a prompt. Using demonstrations to show how to perform a task is often called "few-shot learning."

Fine-tuning improves on few-shot learning by training on many more examples than can fit in the prompt, letting you achieve better results on a wide number of tasks. **Once a model has been fine-tuned, you won't need to provide as many examples in the prompt.** This saves costs and enables lower-latency requests.

At a high level, fine-tuning involves the following steps:

1.  Prepare and upload training data
2.  Train a new fine-tuned model
3.  Evaluate results and go back to step 1 if needed
4.  Use your fine-tuned model

Visit our [pricing page](https://openai.com/api/pricing) to learn more about how fine-tuned model training and usage are billed.

### Which models can be fine-tuned?

Fine-tuning is currently available for the following models:

*   `gpt-4o-2024-08-06`
*   `gpt-4o-mini-2024-07-18`
*   `gpt-4-0613`
*   `gpt-3.5-turbo-0125`
*   `gpt-3.5-turbo-1106`
*   `gpt-3.5-turbo-0613`

You can also fine-tune a fine-tuned model, which is useful if you acquire additional data and don't want to repeat the previous training steps.

We expect `gpt-4o-mini` to be the right model for most users in terms of performance, cost, and ease of use.

When to use fine-tuning
-----------------------

Fine-tuning OpenAI text generation models can make them better for specific applications, but it requires a careful investment of time and effort. We recommend first attempting to get good results with prompt engineering, prompt chaining (breaking complex tasks into multiple prompts), and [function calling](/docs/guides/function-calling), with the key reasons being:

*   There are many tasks at which our models may not initially appear to perform well, but results can be improved with the right prompts - thus fine-tuning may not be necessary
*   Iterating over prompts and other tactics has a much faster feedback loop than iterating with fine-tuning, which requires creating datasets and running training jobs
*   In cases where fine-tuning is still necessary, initial prompt engineering work is not wasted - we typically see best results when using a good prompt in the fine-tuning data (or combining prompt chaining / tool use with fine-tuning)

Our [prompt engineering guide](/docs/guides/prompt-engineering) provides a background on some of the most effective strategies and tactics for getting better performance without fine-tuning. You may find it helpful to iterate quickly on prompts in our [playground](/playground).

### Common use cases

Some common use cases where fine-tuning can improve results:

*   Setting the style, tone, format, or other qualitative aspects
*   Improving reliability at producing a desired output
*   Correcting failures to follow complex prompts
*   Handling many edge cases in specific ways
*   Performing a new skill or task that’s hard to articulate in a prompt

One high-level way to think about these cases is when it’s easier to "show, not tell". In the sections to come, we will explore how to set up data for fine-tuning and various examples where fine-tuning improves the performance over the baseline model.

Another scenario where fine-tuning is effective is reducing cost and/or latency by replacing a more expensive model like `gpt-4o` with a fine-tuned `gpt-4o-mini` model. If you can achieve good results with `gpt-4o`, you can often reach similar quality with a fine-tuned `gpt-4o-mini` model by fine-tuning on the `gpt-4o` completions, possibly with a shortened instruction prompt.

Preparing your dataset
----------------------

Once you have determined that fine-tuning is the right solution (i.e. you’ve optimized your prompt as far as it can take you and identified problems that the model still has), you’ll need to prepare data for training the model. You should create a diverse set of demonstration conversations that are similar to the conversations you will ask the model to respond to at inference time in production.

Each example in the dataset should be a conversation in the same format as our [Chat Completions API](/docs/api-reference/chat/create), specifically a list of messages where each message has a role, content, and [optional name](/docs/api-reference/chat/create#chat/create-chat/create-messages-name). At least some of the training examples should directly target cases where the prompted model is not behaving as desired, and the provided assistant messages in the data should be the ideal responses you want the model to provide.

### Example format

In this example, our goal is to create a chatbot that occasionally gives sarcastic responses, these are three training examples (conversations) we could create for a dataset:

```jsonl
{"messages": [{"role": "system", "content": "Marv is a factual chatbot that is also sarcastic."}, {"role": "user", "content": "What's the capital of France?"}, {"role": "assistant", "content": "Paris, as if everyone doesn't know that already."}]}
{"messages": [{"role": "system", "content": "Marv is a factual chatbot that is also sarcastic."}, {"role": "user", "content": "Who wrote 'Romeo and Juliet'?"}, {"role": "assistant", "content": "Oh, just some guy named William Shakespeare. Ever heard of him?"}]}
{"messages": [{"role": "system", "content": "Marv is a factual chatbot that is also sarcastic."}, {"role": "user", "content": "How far is the Moon from Earth?"}, {"role": "assistant", "content": "Around 384,400 kilometers. Give or take a few, like that really matters."}]}
```

### Multi-turn chat examples

Examples in the chat format can have multiple messages with the assistant role. The default behavior during fine-tuning is to train on all assistant messages within a single example. To skip fine-tuning on specific assistant messages, a `weight` key can be added disable fine-tuning on that message, allowing you to control which assistant messages are learned. The allowed values for `weight` are currently 0 or 1. Some examples using `weight` for the chat format are below.

```jsonl
{"messages": [{"role": "system", "content": "Marv is a factual chatbot that is also sarcastic."}, {"role": "user", "content": "What's the capital of France?"}, {"role": "assistant", "content": "Paris", "weight": 0}, {"role": "user", "content": "Can you be more sarcastic?"}, {"role": "assistant", "content": "Paris, as if everyone doesn't know that already.", "weight": 1}]}
{"messages": [{"role": "system", "content": "Marv is a factual chatbot that is also sarcastic."}, {"role": "user", "content": "Who wrote 'Romeo and Juliet'?"}, {"role": "assistant", "content": "William Shakespeare", "weight": 0}, {"role": "user", "content": "Can you be more sarcastic?"}, {"role": "assistant", "content": "Oh, just some guy named William Shakespeare. Ever heard of him?", "weight": 1}]}
{"messages": [{"role": "system", "content": "Marv is a factual chatbot that is also sarcastic."}, {"role": "user", "content": "How far is the Moon from Earth?"}, {"role": "assistant", "content": "384,400 kilometers", "weight": 0}, {"role": "user", "content": "Can you be more sarcastic?"}, {"role": "assistant", "content": "Around 384,400 kilometers. Give or take a few, like that really matters.", "weight": 1}]}
```

### Crafting prompts

We generally recommend taking the set of instructions and prompts that you found worked best for the model prior to fine-tuning, and including them in every training example. This should let you reach the best and most general results, especially if you have relatively few (e.g. under a hundred) training examples.

If you would like to shorten the instructions or prompts that are repeated in every example to save costs, keep in mind that the model will likely behave as if those instructions were included, and it may be hard to get the model to ignore those "baked-in" instructions at inference time.

It may take more training examples to arrive at good results, as the model has to learn entirely through demonstration and without guided instructions.

### Example count recommendations

To fine-tune a model, you are required to provide at least 10 examples. We typically see clear improvements from fine-tuning on 50 to 100 training examples with `gpt-4o-mini` and `gpt-3.5-turbo`, but the right number varies greatly based on the exact use case.

We recommend starting with 50 well-crafted demonstrations and seeing if the model shows signs of improvement after fine-tuning. In some cases that may be sufficient, but even if the model is not yet production quality, clear improvements are a good sign that providing more data will continue to improve the model. No improvement suggests that you may need to rethink how to set up the task for the model or restructure the data before scaling beyond a limited example set.

### Train and test splits

After collecting the initial dataset, we recommend splitting it into a training and test portion. When submitting a fine-tuning job with both training and test files, we will provide statistics on both during the course of training. These statistics will be your initial signal of how much the model is improving. Additionally, constructing a test set early on will be useful in making sure you are able to evaluate the model after training, by generating samples on the test set.

### Token limits

Token limits depend on the model you select. Here is an overview of the maximum inference context length and training examples context length for `gpt-4o-mini` and `gpt-3.5-turbo` models:

|Model|Inference context length|Training examples context length|
|---|---|---|
|gpt-4o-2024-08-06|128,000 tokens|65,536 tokens (128k coming soon)|
|gpt-4o-mini-2024-07-18|128,000 tokens|65,536 tokens (128k coming soon)|
|gpt-3.5-turbo-0125|16,385 tokens|16,385 tokens|
|gpt-3.5-turbo-1106|16,385 tokens|16,385 tokens|
|gpt-3.5-turbo-0613|16,385 tokens|4,096 tokens|

Examples longer than the default will be truncated to the maximum context length which removes tokens from the end of the training example(s). To be sure that your entire training example fits in context, consider checking that the total token counts in the message contents are under the limit.

You can compute token counts using our [counting tokens notebook](https://cookbook.openai.com/examples/How_to_count_tokens_with_tiktoken.ipynb) from the OpenAI cookbook.

### Estimate costs

For detailed pricing on training costs, as well as input and output costs for a deployed fine-tuned model, visit our [pricing page](https://openai.com/api/pricing). Note that we don't charge for tokens used for training validation. To estimate the cost of a specific fine-tuning training job, use the following formula:

> (base training cost per 1M input tokens ÷ 1M) × number of tokens in the input file × number of epochs trained

For a training file with 100,000 tokens trained over 3 epochs, the expected cost would be:

*   ~$0.90 USD with `gpt-4o-mini-2024-07-18` after the free period ends on October 31, 2024.
*   ~$2.40 USD with `gpt-3.5-turbo-0125`.

### Check data formatting

Once you have compiled a dataset and before you create a fine-tuning job, it is important to check the data formatting. To do this, we created a simple Python script which you can use to find potential errors, review token counts, and estimate the cost of a fine-tuning job.

[

Fine-tuning data format validation

Learn about fine-tuning data formatting

](https://cookbook.openai.com/examples/chat_finetuning_data_prep)

### Upload a training file

Once you have the data validated, the file needs to be uploaded using the [Files API](/docs/api-reference/files/create) in order to be used with a fine-tuning jobs:

Create a fine-tuning job with DPO

```javascript
import OpenAI from "openai";

const openai = new OpenAI();

const job = await openai.fineTuning.jobs.create({
  training_file: "file-all-about-the-weather",
  model: "gpt-4o-2024-08-06",
  method: {
    type: "dpo",
    dpo: {
      hyperparameters: { beta: 0.1 },
    },
  },
});
```

```python
from openai import OpenAI

client = OpenAI()

job = client.fine_tuning.jobs.create(
    training_file="file-all-about-the-weather",
    model="gpt-4o-2024-08-06",
    method={
        "type": "dpo",
        "dpo": {
            "hyperparameters": {"beta": 0.1},
        },
    },
)
```

After you upload the file, it may take some time to process. While the file is processing, you can still create a fine-tuning job but it will not start until the file processing has completed.

The maximum file upload size is 1 GB, though we do not suggest fine-tuning with that amount of data since you are unlikely to need that large of an amount to see improvements.

Create a fine-tuned model
-------------------------

After ensuring you have the right amount and structure for your dataset, and have uploaded the file, the next step is to create a fine-tuning job. We support creating fine-tuning jobs via the [fine-tuning UI](/finetune) or programmatically.

To start a fine-tuning job using the OpenAI SDK:

Create a fine-tuning job

```javascript
import OpenAI from "openai";

const openai = new OpenAI();

const fineTune = await openai.fineTuning.jobs.create({
  training_file: 'file-abc123',
  model: 'gpt-4o-mini-2024-07-18'
});
```

```python
from openai import OpenAI
client = OpenAI()

client.fine_tuning.jobs.create(
    training_file="file-abc123",
    model="gpt-4o-mini-2024-07-18"
)
```

In this example, `model` is the name of the model you want to fine-tune. Note that only specific model snapshots (like `gpt-4o-mini-2024-07-18` in this case) can be used for this parameter, as listed in our [supported models](#which-models-can-be-fine-tuned). The `training_file` parameter is the file ID that was returned when the training file was uploaded to the OpenAI API. You can customize your fine-tuned model's name using the [suffix parameter](/docs/api-reference/fine-tuning/create#fine-tuning/create-suffix).

If you choose not to specify a method, the default is Supervised Fine-Tuning (SFT).

To set additional fine-tuning parameters like the `validation_file` or `hyperparameters`, please refer to the [API specification for fine-tuning](/docs/api-reference/fine-tuning/create).

After you've started a fine-tuning job, it may take some time to complete. Your job may be queued behind other jobs in our system, and training a model can take minutes or hours depending on the model and dataset size. After the model training is completed, the user who created the fine-tuning job will receive an email confirmation.

In addition to creating a fine-tuning job, you can also list existing jobs, retrieve the status of a job, or cancel a job.

Work with fine-tuning jobs

```javascript
import OpenAI from "openai";

const openai = new OpenAI();

// List 10 fine-tuning jobs
let page = await openai.fineTuning.jobs.list({ limit: 10 });

// Retrieve the state of a fine-tune
let fineTune = await openai.fineTuning.jobs.retrieve('ftjob-abc123');

// Cancel a job
let status = await openai.fineTuning.jobs.cancel('ftjob-abc123');

// List up to 10 events from a fine-tuning job
let events = await openai.fineTuning.jobs.listEvents(fineTune.id, {
  limit: 10
});

// Delete a fine-tuned model
let model = await openai.models.delete(
  'ft:gpt-3.5-turbo:acemeco:suffix:abc123'
);
```

```python
from openai import OpenAI
client = OpenAI()

# List 10 fine-tuning jobs
client.fine_tuning.jobs.list(limit=10)

# Retrieve the state of a fine-tune
client.fine_tuning.jobs.retrieve("ftjob-abc123")

# Cancel a job
client.fine_tuning.jobs.cancel("ftjob-abc123")

# List up to 10 events from a fine-tuning job
client.fine_tuning.jobs.list_events(fine_tuning_job_id="ftjob-abc123", limit=10)

# Delete a fine-tuned model
client.models.delete("ft:gpt-3.5-turbo:acemeco:suffix:abc123")
```

Use a fine-tuned model
----------------------

When a job has succeeded, you will see the `fine_tuned_model` field populated with the name of the model when you retrieve the job details. You may now specify this model as a parameter to in the [Chat Completions](/docs/api-reference/chat) API, and make requests to it using the [Playground](/playground).

After your job is completed, the model should be available right away for inference use. In some cases, it may take several minutes for your model to become ready to handle requests. If requests to your model time out or the model name cannot be found, it is likely because your model is still being loaded. If this happens, try again in a few minutes.

Use fine-tuned model

```javascript
import OpenAI from "openai";

const openai = new OpenAI();

const fineTune = await openai.fineTuning.jobs.create({
  training_file: 'file-abc123',
  model: 'gpt-4o-mini-2024-07-18'
});
```

```python
from openai import OpenAI
client = OpenAI()

completion = client.chat.completions.create(
    model="ft:gpt-4o-mini:my-org:custom_suffix:id",
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "Hello!"}
    ]
)

print(completion.choices[0].message)
```

You can start making requests by passing the model name as shown above and in our [GPT guide](/docs/guides/text-generation).

Use a checkpointed model
------------------------

In addition to creating a final fine-tuned model at the end of each fine-tuning job, OpenAI will create one full model checkpoint for you at the end of each training epoch. These checkpoints are themselves full models that can be used within our completions and chat-completions endpoints. Checkpoints are useful as they potentially provide a version of your fine-tuned model from before it experienced overfitting.

To access these checkpoints,

1.  Wait until a job succeeds, which you can verify by [querying the status of a job.](/docs/api-reference/fine-tuning/retrieve)
2.  [Query the checkpoints endpoint](/docs/api-reference/fine-tuning/list-checkpoints) with your fine-tuning job ID to access a list of model checkpoints for the fine-tuning job.

For each checkpoint object, you will see the `fine_tuned_model_checkpoint` field populated with the name of the model checkpoint. You may now use this model just like you would with the [final fine-tuned model](#use-a-fine-tuned-model).

```json
{
    "object": "fine_tuning.job.checkpoint",
    "id": "ftckpt_zc4Q7MP6XxulcVzj4MZdwsAB",
    "created_at": 1519129973,
    "fine_tuned_model_checkpoint": "ft:gpt-3.5-turbo-0125:my-org:custom-suffix:96olL566:ckpt-step-2000",
    "metrics": {
        "full_valid_loss": 0.134,
        "full_valid_mean_token_accuracy": 0.874
    },
    "fine_tuning_job_id": "ftjob-abc123",
    "step_number": 2000
}
```

Each checkpoint will specify its:

*   `step_number`: The step at which the checkpoint was created (where each epoch is number of steps in the training set divided by the batch size)
*   `metrics`: an object containing the metrics for your fine-tuning job at the step when the checkpoint was created.

Currently, only the checkpoints for the last 3 epochs of the job are saved and available for use. We plan to release more complex and flexible checkpointing strategies in the near future.

Analyzing your fine-tuned model
-------------------------------

We provide the following training metrics computed over the course of training:

*   training loss
*   training token accuracy
*   valid loss
*   valid token accuracy

Valid loss and valid token accuracy are computed in two different ways - on a small batch of the data during each step, and on the full valid split at the end of each epoch. The full valid loss and full valid token accuracy metrics are the most accurate metric tracking the overall performance of your model. These statistics are meant to provide a sanity check that training went smoothly (loss should decrease, token accuracy should increase). While an active fine-tuning jobs is running, you can view an event object which contains some useful metrics:

```json
{
    "object": "fine_tuning.job.event",
    "id": "ftevent-abc-123",
    "created_at": 1693582679,
    "level": "info",
    "message": "Step 300/300: training loss=0.15, validation loss=0.27, full validation loss=0.40",
    "data": {
        "step": 300,
        "train_loss": 0.14991648495197296,
        "valid_loss": 0.26569826706596045,
        "total_steps": 300,
        "full_valid_loss": 0.4032616495084362,
        "train_mean_token_accuracy": 0.9444444179534912,
        "valid_mean_token_accuracy": 0.9565217391304348,
        "full_valid_mean_token_accuracy": 0.9089635854341737
    },
    "type": "metrics"
}
```

After a fine-tuning job has finished, you can also see metrics around how the training process went by [querying a fine-tuning job](/docs/api-reference/fine-tuning/retrieve), extracting a file ID from the `result_files`, and then [retrieving that files content](/docs/api-reference/files/retrieve-contents). Each results CSV file has the following columns: `step`, `train_loss`, `train_accuracy`, `valid_loss`, and `valid_mean_token_accuracy`.

```csv
step,train_loss,train_accuracy,valid_loss,valid_mean_token_accuracy
1,1.52347,0.0,,
2,0.57719,0.0,,
3,3.63525,0.0,,
4,1.72257,0.0,,
5,1.52379,0.0,,
```

While metrics can be helpful, evaluating samples from the fine-tuned model provides the most relevant sense of model quality. We recommend generating samples from both the base model and the fine-tuned model on a test set, and comparing the samples side by side. The test set should ideally include the full distribution of inputs that you might send to the model in a production use case. If manual evaluation is too time-consuming, consider using our [Evals library](https://github.com/openai/evals) to automate future evaluations.

### Iterating on data quality

If the results from a fine-tuning job are not as good as you expected, consider the following ways to adjust the training dataset:

*   Collect examples to target remaining issues
    *   If the model still isn’t good at certain aspects, add training examples that directly show the model how to do these aspects correctly
*   Scrutinize existing examples for issues
    *   If your model has grammar, logic, or style issues, check if your data has any of the same issues. For instance, if the model now says "I will schedule this meeting for you" (when it shouldn’t), see if existing examples teach the model to say it can do new things that it can’t do
*   Consider the balance and diversity of data
    *   If 60% of the assistant responses in the data says "I cannot answer this", but at inference time only 5% of responses should say that, you will likely get an overabundance of refusals
*   Make sure your training examples contain all of the information needed for the response
    *   If we want the model to compliment a user based on their personal traits and a training example includes assistant compliments for traits not found in the preceding conversation, the model may learn to hallucinate information
*   Look at the agreement / consistency in the training examples
    *   If multiple people created the training data, it’s likely that model performance will be limited by the level of agreement / consistency between people. For instance, in a text extraction task, if people only agreed on 70% of extracted snippets, the model would likely not be able to do better than this
*   Make sure your all of your training examples are in the same format, as expected for inference

### Iterating on data quantity

Once you’re satisfied with the quality and distribution of the examples, you can consider scaling up the number of training examples. This tends to help the model learn the task better, especially around possible "edge cases". We expect a similar amount of improvement every time you double the number of training examples. You can loosely estimate the expected quality gain from increasing the training data size by:

*   Fine-tuning on your current dataset
*   Fine-tuning on half of your current dataset
*   Observing the quality gap between the two

In general, if you have to make a trade-off, a smaller amount of high-quality data is generally more effective than a larger amount of low-quality data.

### Iterating on hyperparameters

We allow you to specify the following hyperparameters:

*   epochs
*   learning rate multiplier
*   batch size

We recommend initially training without specifying any of these, allowing us to pick a default for you based on dataset size, then adjusting if you observe the following:

*   If the model does not follow the training data as much as expected increase the number of epochs by 1 or 2
    *   This is more common for tasks for which there is a single ideal completion (or a small set of ideal completions which are similar). Some examples include classification, entity extraction, or structured parsing. These are often tasks for which you can compute a final accuracy metric against a reference answer.
*   If the model becomes less diverse than expected decrease the number of epochs by 1 or 2
    *   This is more common for tasks for which there are a wide range of possible good completions
*   If the model does not appear to be converging, increase the learning rate multiplier

You can set the hyperparameters as is shown below:

Setting hyperparameters

```javascript
const fineTune = await openai.fineTuning.jobs.create({
  training_file: "file-abc123",
  model: "gpt-4o-mini-2024-07-18",
  method: {
    type: "supervised",
    supervised: {
      hyperparameters: { n_epochs: 2 },
    },
  },
});
```

```python
from openai import OpenAI
client = OpenAI()

client.fine_tuning.jobs.create(
    training_file="file-abc123",
    model="gpt-4o-mini-2024-07-18",
    method={
        "type": "supervised",
        "supervised": {
            "hyperparameters": {"n_epochs": 2},
        },
    },
)
```

Vision fine-tuning

----------------------

Fine-tuning is also possible with images in your JSONL files. Just as you can [send one or many image inputs to chat completions](/docs/guides/vision), you can include those same message types within your training data. Images can be provided either as HTTP URLs or data URLs containing [base64 encoded images](/docs/guides/vision#uploading-base-64-encoded-images).

Here's an example of an image message on a line of your JSONL file. Below, the JSON object is expanded for readibility, but typically this JSON would appear on a single line in your data file:

```json
{
  "messages": [
    { "role": "system", "content": "You are an assistant that identifies uncommon cheeses." },
    { "role": "user", "content": "What is this cheese?" },
    { "role": "user", "content": [
        {
          "type": "image_url",
          "image_url": {
            "url": "https://upload.wikimedia.org/wikipedia/commons/3/36/Danbo_Cheese.jpg"
          }
        }
      ]
    },
    { "role": "assistant", "content": "Danbo" }
  ]
}
```

### Image dataset requirements

#### Size

*   Your training file can contain a maximum of 50,000 examples that contain images (not including text examples).
*   Each example can have at most 10 images.
*   Each image can be at most 10 MB.

#### Format

*   Images must be JPEG, PNG, or WEBP format.
*   Your images must be in the RGB or RGBA image mode.
*   You cannot include images as output from messages with the `assistant` role.

#### Content moderation policy

We scan your images before training to ensure that they comply with our usage policy. This may introduce latency in file validation before fine tuning begins.

Images containing the following will be excluded from your dataset and not used for training:

*   People
*   Faces
*   Children
*   CAPTCHAs

### Help

#### What to do if your images get skipped

Your images can get skipped for the following reasons:

*   **contains CAPTCHAs**, **contains people**, **contains faces**, **contains children**
    *   Remove the image. For now, we cannot fine-tune models with images containing these entities.
*   **inaccessible URL**
    *   Ensure that the image URL is publicly accessible.
*   **image too large**
    *   Please ensure that your images fall within our [dataset size limits](#size).
*   **invalid image format**
    *   Please ensure that your images fall within our [dataset format](#format).

#### How to upload large files

*   Your training files might get quite large. You can upload files up to 8 GB in multiple parts using the [Uploads API](/docs/api-reference/uploads) as opposed to the [Files API](/docs/api-reference/files), which only allows file uploads of up to 512 MB.

#### Reducing training cost

If you set the `detail` parameter for an image to `low`, the image is resized to 512 by 512 pixels and is only represented by 85 tokens regardless of its size. This will reduce the cost of training. [See here for more information.](/docs/guides/vision#low-or-high-fidelity-image-understanding)

```text
{
    "type": "image_url",
    "image_url": {
        "url": "https://upload.wikimedia.org/wikipedia/commons/3/36/Danbo_Cheese.jpg",
        "detail": "low"
    }
}
```

#### Other considerations for vision fine-tuning

*   To control the fidelity of image understanding, set the `detail` parameter of `image_url` to `low`, `high`, or `auto` for each image. This will also affect the number of tokens per image that the model sees during training time, and will affect the cost of training. [See here for more information.](/docs/guides/vision#low-or-high-fidelity-image-understanding)

Preference fine-tuning

--------------------------

[Direct Preference Optimization](https://arxiv.org/abs/2305.18290) (DPO) fine-tuning allows you to fine-tune models based on prompts and pairs of responses. This approach enables the model to learn from human preferences, optimizing for outputs that are more likely to be favored. Note that we currently support text-only DPO fine-tuning.

#### Preparing your dataset for DPO

Each example in your dataset should contain:

*   A prompt, like a user message.
*   A preferred output (an ideal assistant response).
*   A non-preferred output (a suboptimal assistant response).

The data should be formatted in JSONL format, with each line representing an example in the following structure:

```json
{
  "input": {
    "messages": [
      {
        "role": "user",
        "content": "Hello, can you tell me how cold San Francisco is today?"
      }
    ],
    "tools": [],
    "parallel_tool_calls": true
  },
  "preferred_output": [
    {
      "role": "assistant",
      "content": "Today in San Francisco, it is not quite cold as expected. Morning clouds will give away to sunshine, with a high near 68°F (20°C) and a low around 57°F (14°C)."
    }
  ],
  "non_preferred_output": [
    {
      "role": "assistant",
      "content": "It is not particularly cold in San Francisco today."
    }
  ]
}
```

Currently, we only train on one-turn conversations for each example, where the preferred and non-preferred messages need to be the last assistant message.

#### Stacking methods: Supervised + DPO

Currently, OpenAI offers Supervised Fine-Tuning (SFT) as the default method for fine-tuning jobs. Performing SFT on your preferred responses (or a subset) before running another DPO job afterwards can significantly enhance model alignment and performance. By first fine-tuning the model on the desired responses, it can better identify correct patterns, providing a strong foundation for DPO to refine behavior.

A recommended workflow is as follows:

1.  Fine-tune the base model with SFT using a subset of your preferred responses. Focus on ensuring the data quality and representativeness of the tasks.
2.  Use the SFT fine-tuned model as the starting point, and apply DPO to adjust the model based on preference comparisons.

#### Configuring a DPO fine-tuning job

We have introduced a `method` field in the fine-tuning job creation endpoint, where you can specify `type` as well as any associated `hyperparameters`. For DPO:

*   set the `type` parameter to `dpo`
*   optionally set the `hyperparameters` property with any options you'd like to configure.

The `beta` hyperparameter is a new option that is only available for DPO. It's a floating point number between `0` and `2` that controls how strictly the new model will adhere to its previous behavior, versus aligning with the provided preferences. A high number will be more conservative (favoring previous behavior), and a lower number will be more aggressive (favor the newly provided preferences more often).

You can also set this value to `auto` (the default) to use a value configured by the platform.

The example below shows how to configure a DPO fine-tuning job using the OpenAI SDK. For more information on creating fine-tuning jobs in general, please refer to the [next section of the guide](#create-a-fine-tuned-model).

Create a fine-tuning job with DPO

```javascript
import OpenAI from "openai";

const openai = new OpenAI();

const job = await openai.fineTuning.jobs.create({
  training_file: "file-all-about-the-weather",
  model: "gpt-4o-2024-08-06",
  method: {
    type: "dpo",
    dpo: {
      hyperparameters: { beta: 0.1 },
    },
  },
});
```

```python
from openai import OpenAI

client = OpenAI()

job = client.fine_tuning.jobs.create(
    training_file="file-all-about-the-weather",
    model="gpt-4o-2024-08-06",
    method={
        "type": "dpo",
        "dpo": {
            "hyperparameters": {"beta": 0.1},
        },
    },
)
```

Fine-tuning examples
--------------------

Now that we have explored the basics of the fine-tuning API, let’s look at going through the fine-tuning lifecycle for a few different use cases.

Style and tone

In this example, we will explore how to build a fine-tuned model which gets the model follow specific style and tone guidance beyond what is possible with prompting alone.

To begin, we create a sample set of messages showing what the model should which in this case is misspelled words.

```jsonl
{"messages": [{"role": "system", "content": "Marv is a factual chatbot that is also sarcastic."}, {"role": "user", "content": "What's the capital of France?"}, {"role": "assistant", "content": "Paris, as if everyone doesn't know that already."}]}
{"messages": [{"role": "system", "content": "Marv is a factual chatbot that is also sarcastic."}, {"role": "user", "content": "Who wrote 'Romeo and Juliet'?"}, {"role": "assistant", "content": "Oh, just some guy named William Shakespeare. Ever heard of him?"}]}
{"messages": [{"role": "system", "content": "Marv is a factual chatbot that is also sarcastic."}, {"role": "user", "content": "How far is the Moon from Earth?"}, {"role": "assistant", "content": "Around 384,400 kilometers. Give or take a few, like that really matters."}]}
```

If you want to follow along and create a fine-tuned model yourself, you will need at least 10 examples.

After getting the data that will potentially improve the model, the next step is to check if the data meets all the [formatting requirements](#check-data-formatting).

Now that we have the data formatted and validated, the final training step is to kick off a job to create the fine-tuned model. You can do this via the OpenAI CLI or one of our SDKs as shown below:

```python
from openai import OpenAI
client = OpenAI()

file = client.files.create(
  file=open("marv.jsonl", "rb"),
  purpose="fine-tune"
)

client.fine_tuning.jobs.create(
  training_file=file.id,
  model="gpt-4o-mini-2024-07-18"
)
```

Once the training job is done, you will be able to [use your fine-tuned model](#use-a-fine-tuned-model).

Structured output

Another type of use case which works really well with fine-tuning is getting the model to provide structured information, in this case about sports headlines:

```jsonl
{"messages": [{"role": "system", "content": "Given a sports headline, provide the following fields in a JSON dict, where applicable: \"player\" (full name), \"team\", \"sport\", and \"gender\"."}, {"role": "user", "content": "Sources: Colts grant RB Taylor OK to seek trade"}, {"role": "assistant", "content": "{\"player\": \"Jonathan Taylor\", \"team\": \"Colts\", \"sport\": \"football\", \"gender\": \"male\" }"}]}
{"messages": [{"role": "system", "content": "Given a sports headline, provide the following fields in a JSON dict, where applicable: \"player\" (full name), \"team\", \"sport\", and \"gender\"."}, {"role": "user", "content": "OSU 'split down middle' on starting QB battle"}, {"role": "assistant", "content": "{\"player\": null, \"team\": \"OSU\", \"sport\": \"football\", \"gender\": null }"}]}
```

If you want to follow along and create a fine-tuned model yourself, you will need at least 10 examples.

After getting the data that will potentially improve the model, the next step is to check if the data meets all the [formatting requirements](#check-data-formatting).

Now that we have the data formatted and validated, the final training step is to kick off a job to create the fine-tuned model. You can do this via the OpenAI CLI or one of our SDKs as shown below:

```python
from openai import OpenAI
client = OpenAI()

file = client.files.create(
  file=open("sports-context.jsonl", "rb"),
  purpose="fine-tune"
)

client.fine_tuning.jobs.create(
  training_file=file.id,
  model="gpt-4o-mini-2024-07-18"
)
```

Once the training job is done, you will be able to [use your fine-tuned model](#use-a-fine-tuned-model) and make a request that looks like the following:

```python
completion = client.chat.completions.create(
  model="ft:gpt-4o-mini:my-org:custom_suffix:id",
  messages=[
    {"role": "system", "content": "Given a sports headline, provide the following fields in a JSON dict, where applicable: player (full name), team, sport, and gender"},
    {"role": "user", "content": "Richardson wins 100m at worlds to cap comeback"}
  ]
)

print(completion.choices[0].message)
```

Based on the formatted training data, the response should look like the following:

```json
{
    "player": "Sha'Carri Richardson",
    "team": null,
    "sport": "track and field",
    "gender": "female"
}
```

Tool calling

The chat completions API supports [tool calling](/docs/guides/function-calling). Including a long list of tools in the completions API can consume a considerable number of prompt tokens and sometimes the model hallucinates or does not provide valid JSON output.

Fine-tuning a model with tool calling examples can allow you to:

*   Get similarly formatted responses even when the full tool definition isn't present
*   Get more accurate and consistent outputs

Format your examples as shown, with each line including a list of "messages" and an optional list of "tools":

```json
{
    "messages": [
        { "role": "user", "content": "What is the weather in San Francisco?" },
        {
            "role": "assistant",
            "tool_calls": [
                {
                    "id": "call_id",
                    "type": "function",
                    "function": {
                        "name": "get_current_weather",
                        "arguments": "{\"location\": \"San Francisco, USA\", \"format\": \"celsius\"}"
                    }
                }
            ]
        }
    ],
    "tools": [
        {
            "type": "function",
            "function": {
                "name": "get_current_weather",
                "description": "Get the current weather",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "location": {
                            "type": "string",
                            "description": "The city and country, eg. San Francisco, USA"
                        },
                        "format": { "type": "string", "enum": ["celsius", "fahrenheit"] }
                    },
                    "required": ["location", "format"]
                }
            }
        }
    ]
}
```

If you want to follow along and create a fine-tuned model yourself, you will need at least 10 examples.

If your goal is to use less tokens, some useful techniques are:

*   Omit function and parameter descriptions: remove the description field from function and parameters
*   Omit parameters: remove the entire properties field from the parameters object
*   Omit function entirely: remove the entire function object from the functions array

If your goal is to maximize the correctness of the function calling output, we recommend using the same tool definitions for both training and querying the fine-tuned model.

Fine-tuning on function calling can also be used to customize the model's response to function outputs. To do this you can include a function response message and an assistant message interpreting that response:

```json
{
    "messages": [
        {"role": "user", "content": "What is the weather in San Francisco?"},
        {"role": "assistant", "tool_calls": [{"id": "call_id", "type": "function", "function": {"name": "get_current_weather", "arguments": "{\"location\": \"San Francisco, USA\", \"format\": \"celsius\"}"}}]}
        {"role": "tool", "tool_call_id": "call_id", "content": "21.0"},
        {"role": "assistant", "content": "It is 21 degrees celsius in San Francisco, CA"}
    ],
    "tools": [] // same as before
}
```

[Parallel function calling](/docs/guides/function-calling) is enabled by default and can be disabled by using `parallel_tool_calls: false` in the training example.

Function calling

`function_call` and `functions` have been deprecated in favor of `tools` it is recommended to use the `tools` parameter instead.

The chat completions API supports [function calling](/docs/guides/function-calling). Including a long list of functions in the completions API can consume a considerable number of prompt tokens and sometimes the model hallucinates or does not provide valid JSON output.

Fine-tuning a model with function calling examples can allow you to:

*   Get similarly formatted responses even when the full function definition isn't present
*   Get more accurate and consistent outputs

Format your examples as shown, with each line including a list of "messages" and an optional list of "functions":

```json
{
    "messages": [
        { "role": "user", "content": "What is the weather in San Francisco?" },
        {
            "role": "assistant",
            "function_call": {
                "name": "get_current_weather",
                "arguments": "{\"location\": \"San Francisco, USA\", \"format\": \"celsius\"}"
            }
        }
    ],
    "functions": [
        {
            "name": "get_current_weather",
            "description": "Get the current weather",
            "parameters": {
                "type": "object",
                "properties": {
                    "location": {
                        "type": "string",
                        "description": "The city and country, eg. San Francisco, USA"
                    },
                    "format": { "type": "string", "enum": ["celsius", "fahrenheit"] }
                },
                "required": ["location", "format"]
            }
        }
    ]
}
```

If you want to follow along and create a fine-tuned model yourself, you will need at least 10 examples.

If your goal is to use less tokens, some useful techniques are:

*   Omit function and parameter descriptions: remove the description field from function and parameters
*   Omit parameters: remove the entire properties field from the parameters object
*   Omit function entirely: remove the entire function object from the functions array

If your goal is to maximize the correctness of the function calling output, we recommend using the same function definitions for both training and querying the fine-tuned model.

Fine-tuning on function calling can also be used to customize the model's response to function outputs. To do this you can include a function response message and an assistant message interpreting that response:

```json
{
    "messages": [
        {"role": "user", "content": "What is the weather in San Francisco?"},
        {"role": "assistant", "function_call": {"name": "get_current_weather", "arguments": "{\"location\": \"San Francisco, USA\", \"format\": \"celsius\"}"}}
        {"role": "function", "name": "get_current_weather", "content": "21.0"},
        {"role": "assistant", "content": "It is 21 degrees celsius in San Francisco, CA"}
    ],
    "functions": [] // same as before
}
```

Fine-tuning integrations
========================

OpenAI provides the ability for you to integrate your fine-tuning jobs with 3rd parties via our integration framework. Integrations generally allow you to track job state, status, metrics, hyperparameters, and other job-related information in a 3rd party system. You can also use integrations to trigger actions in a 3rd party system based on job state changes. Currently, the only supported integration is with [Weights and Biases](https://wandb.ai), but more are coming soon.

Weights and Biases Integration
------------------------------

[Weights and Biases (W&B)](https://wandb.ai) is a popular tool for tracking machine learning experiments. You can use the OpenAI integration with W&B to track your fine-tuning jobs in W&B. This integration will automatically log metrics, hyperparameters, and other job-related information to the W&B project you specify.

To integrate your fine-tuning jobs with W&B, you'll need to

1.  Provide authentication credentials for your Weights and Biases account to OpenAI
2.  Configure the W&B integration when creating new fine-tuning jobs

### Authenticate your Weights and Biases account with OpenAI

Authentication is done by submitting a valid W&B API key to OpenAI. Currently, this can only be done via the [Account Dashboard](/settings/organization/organization), and only by account administrators. Your W&B API key will be stored encrypted within OpenAI and will allow OpenAI to post metrics and metadata on your behalf to W&B when your fine-tuning jobs are running. Attempting to enable a W&B integration on a fine-tuning job without first authenticating your OpenAI organization with WandB will result in an error.

![](https://cdn.openai.com/API/images/guides/WandB_Integration.png)

### Enable the Weights and Biases integration

When creating a new fine-tuning job, you can enable the W&B integration by including a new `"wandb"` integration under the `integrations` field in the job creation request. This integration allows you to specify the W&B Project that you wish the newly created W&B Run to show up under.

Here's an example of how to enable the W&B integration when creating a new fine-tuning job:

```bash
curl -X POST \\
    -H "Content-Type: application/json" \\
    -H "Authorization: Bearer $OPENAI_API_KEY" \\
    -d '{
    "model": "gpt-4o-mini-2024-07-18",
    "training_file": "file-ABC123",
    "validation_file": "file-DEF456",
    "integrations": [
        {
            "type": "wandb",
            "wandb": {
                "project": "custom-wandb-project",
                "tags": ["project:tag", "lineage"]
            }
        }
    ]
}' https://api.openai.com/v1/fine_tuning/jobs
```

By default, the Run ID and Run display name are the ID of your fine-tuning job (e.g. `ftjob-abc123`). You can customize the display name of the run by including a `"name"` field in the `wandb` object. You can also include a `"tags"` field in the `wandb` object to add tags to the W&B Run (tags must be <= 64 character strings and there is a maximum of 50 tags).

Sometimes it is convenient to explicitly set the [W&B Entity](https://docs.wandb.ai/guides/runs/manage-runs#send-new-runs-to-a-team) to be associated with the run. You can do this by including an `"entity"` field in the `wandb` object. If you do not include an `"entity"` field, the W&B entity will default to the default W&B entity associated with the API key you registered previously.

The full specification for the integration can be found in our [fine-tuning job creation](/docs/api-reference/fine-tuning/create) documentation.

### View your fine-tuning job in Weights and Biases

Once you've created a fine-tuning job with the W&B integration enabled, you can view the job in W&B by navigating to the W&B project you specified in the job creation request. Your run should be located at the URL: `https://wandb.ai/<WANDB-ENTITY>/<WANDB-PROJECT>/runs/ftjob-ABCDEF`.

You should see a new run with the name and tags you specified in the job creation request. The Run Config will contain relevant job metadata such as:

*   `model`: The model you are fine-tuning
*   `training_file`: The ID of the training file
*   `validation_file`: The ID of the validation file
*   `hyperparameters`: The hyperparameters used for the job (e.g. `n_epochs`, `learning_rate_multiplier`, `batch_size`)
*   `seed`: The random seed used for the job

Likewise, OpenAI will set some default tags on the run to make it easier for your to search and filter. These tags will be prefixed with `"openai/"` and will include:

*   `openai/fine-tuning`: Tag to let you know this run is a fine-tuning job
*   `openai/ft-abc123`: The ID of the fine-tuning job
*   `openai/gpt-4o-mini`: The model you are fine-tuning

An example W&B run generated from an OpenAI fine-tuning job is shown below:

![](https://cdn.openai.com/API/images/guides/WandB_Integration_Dashboard1.png)

Metrics for each step of the fine-tuning job will be logged to the W&B run. These metrics are the same metrics provided in the [fine-tuning job event](/docs/api-reference/fine-tuning/list-events) object and are the same metrics your can view via the [OpenAI fine-tuning Dashboard](https://platform.openai.com/finetune). You can use W&B's visualization tools to track the progress of your fine-tuning job and compare it to other fine-tuning jobs you've run.

An example of the metrics logged to a W&B run is shown below:

![](https://cdn.openai.com/API/images/guides/WandB_Integration_Dashboard2.png)

FAQ
---

### When should I use fine-tuning vs embeddings / retrieval augmented generation?

Embeddings with retrieval is best suited for cases when you need to have a large database of documents with relevant context and information.

By default OpenAI’s models are trained to be helpful generalist assistants. Fine-tuning can be used to make a model which is narrowly focused, and exhibits specific ingrained behavior patterns. Retrieval strategies can be used to make new information available to a model by providing it with relevant context before generating its response. Retrieval strategies are not an alternative to fine-tuning and can in fact be complementary to it.

You can explore the differences between these options further in this Developer Day talk:

### How do I know if my fine-tuned model is actually better than the base model?

We recommend generating samples from both the base model and the fine-tuned model on a test set of chat conversations, and comparing the samples side by side. For more comprehensive evaluations, consider using the [OpenAI evals framework](https://github.com/openai/evals) to create an eval specific to your use case.

### Can I continue fine-tuning a model that has already been fine-tuned?

Yes, you can pass the name of a fine-tuned model into the `model` parameter when creating a fine-tuning job. This will start a new fine-tuning job using the fine-tuned model as the starting point.

### How can I estimate the cost of fine-tuning a model?

Please refer to the [estimate cost](#estimate-costs) section above.

### How many fine-tuning jobs can I have running at once?

Please refer to our [rate limit page](/docs/guides/rate-limits#what-are-the-rate-limits-for-our-api) for the most up to date information on the limits.

### How do rate limits work on fine-tuned models?

A fine-tuned model pulls from the same shared rate limit as the model it is based off of. For example, if you use half your TPM rate limit in a given time period with the standard `gpt-4o-mini` model, any model(s) you fine-tuned from `gpt-4o-mini` would only have the remaining half of the TPM rate limit accessible since the capacity is shared across all models of the same type.

Put another way, having fine-tuned models does not give you more capacity to use our models from a total throughput perspective.

===

Function calling
================

Connect models to external data and systems.

**Function calling** enables developers to connect language models to external data and systems. You can define a set of functions as tools that the model has access to, and it can use them when appropriate based on the conversation history. You can then execute those functions on the application side, and provide results back to the model.

Learn how to extend the capabilities of OpenAI models through function calling in this guide.

Experiment with function calling in the [Playground](/playground) by providing your own function definition or generate a ready-to-use definition.

Generate

Overview
--------

Many applications require models to call custom functions to trigger actions within the application or interact with external systems.

Here is how you can define a function as a tool for the model to use:

```python
from openai import OpenAI

client = OpenAI()

tools = [
  {
      "type": "function",
      "function": {
          "name": "get_weather",
          "parameters": {
              "type": "object",
              "properties": {
                  "location": {"type": "string"}
              },
          },
      },
  }
]

completion = client.chat.completions.create(
  model="gpt-4o",
  messages=[{"role": "user", "content": "What's the weather like in Paris today?"}],
  tools=tools,
)

print(completion.choices[0].message.tool_calls)
```

```javascript
import { OpenAI } from "openai";
const openai = new OpenAI();

const tools = [
  {
      type: "function",
      function: {
          name: "get_weather",
          parameters: {
              type: "object",
              properties: {
                  location: { type: "string" }
              },
          },
      },
  }
];

const response = await openai.chat.completions.create({
  model: "gpt-4o",
  messages: [{"role": "user", "content": "What's the weather like in Paris today?"}],
  tools,
});

console.log(response.choices[0].message.tool_calls);
```

This will return a function call that looks like this:

```json
[
  {
    "id": "call_12345xyz",
    "type": "function",
    "function": { "name": "get_weather", "arguments": "{'location':'Paris'}" }
  }
]
```

Functions are the only type of tools supported in the Chat Completions API, but the Assistants API also supports [built-in tools](/docs/assistants/tools).

Here are a few examples where function calling can be useful:

1.  **Fetching data:** enable a conversational assistant to retrieve data from internal systems before responding to the user.
2.  **Taking action:** allow an assistant to trigger actions based on the conversation, like scheduling meetings or initiating order returns.
3.  **Building rich workflows:** allow assistants to execute multi-step workflows, like data extraction pipelines or content personalization.
4.  **Interacting with Application UIs:** use function calls to update the user interface based on user input, like rendering a pin on a map or navigating a website.

You can find example use cases in the [examples](#examples) section below.

### The lifecycle of a function call

When you use the OpenAI API with function calling, the model never actually executes functions itself - instead, it simply generates parameters that can be used to call your function. You are then responsible for handling how the function is executed in your code.

Read our [integration guide](#integration-guide) below for more details on how to handle function calls.

![Function Calling diagram](https://cdn.openai.com/API/docs/images/function-calling-diagram.png)

### Function calling support

Function calling is supported in the [Chat Completions API](/docs/guides/text-generation), [Assistants API](/docs/assistants/overview), [Batch API](/docs/guides/batch) and [Realtime API](/docs/guides/realtime).

This guide focuses on function calling using the Chat Completions API. We have separate guides for [function calling using the Assistants API](/docs/assistants/tools/function-calling), and for [function calling using the Realtime API](/docs/guides/realtime#tool-calling).

#### Models supporting function calling

Function calling was introduced with the release of `gpt-4-turbo` on June 13, 2023. All `gpt-*` models released after this date support function calling.

Legacy models released before this date were not trained to support function calling.

Support for parallel function calling

Parallel function calling is supported on models released on or after Nov 6, 2023. This includes: `gpt-4o`, `gpt-4o-2024-08-06`, `gpt-4o-2024-05-13`, `gpt-4o-mini`, `gpt-4o-mini-2024-07-18`, `gpt-4-turbo`, `gpt-4-turbo-2024-04-09`, `gpt-4-turbo-preview`, `gpt-4-0125-preview`, `gpt-4-1106-preview`, `gpt-3.5-turbo`, `gpt-3.5-turbo-0125`, and `gpt-3.5-turbo-1106`.

You can find a complete list of models and their release date on our [models page](/docs/models).

Integration guide
-----------------

In this integration guide, we will walk through integrating function calling into an application, taking an order delivery assistant as an example. Rather than requiring users to interact with a form, we can let them ask the assistant for help in natural language.

We will cover how to define functions and instructions, then how to handle model responses and function execution results.

If you want to learn more about how to handle function calls in a streaming fashion, how to customize tool calling behavior or how to handle edge cases, refer to our [advanced usage](#advanced-usage) section.

### Function definition

The starting point for function calling is choosing a function in your own codebase that you'd like to enable the model to generate arguments for.

For this example, let's imagine you want to allow the model to call the `get_delivery_date` function in your codebase which accepts an `order_id` and queries your database to determine the delivery date for a given package. Your function might look something like the following:

```python
# This is the function that we want the model to be able to call
def get_delivery_date(order_id: str) -> datetime:
  # Connect to the database
  conn = sqlite3.connect('ecommerce.db')
  cursor = conn.cursor()
  # ...
```

```javascript
// This is the function that we want the model to be able to call
const getDeliveryDate = async (orderId: string): datetime => { 
  const connection = await createConnection({
      host: 'localhost',
      user: 'root',
      // ...
  });
}
```

Now that we know which function we wish to allow the model to call, we will create a “function definition” that describes the function to the model. This definition describes both what the function does (and potentially when it should be called) and what parameters are required to call the function.

The `parameters` section of your function definition should be described using JSON Schema. If and when the model generates a function call, it will use this information to generate arguments according to your provided schema.

If you want to ensure the model always adheres to your supplied schema, you can enable [Structured Outputs](#structured-outputs) with function calling.

In this example it may look like this:

```json
{
    "name": "get_delivery_date",
    "description": "Get the delivery date for a customer's order. Call this whenever you need to know the delivery date, for example when a customer asks 'Where is my package'",
    "parameters": {
        "type": "object",
        "properties": {
            "order_id": {
                "type": "string",
                "description": "The customer's order ID."
            }
        },
        "required": ["order_id"],
        "additionalProperties": false
    }
}
```

Next we need to provide our function definitions within an array of available “tools” when calling the Chat Completions API.

As always, we will provide an array of “messages”, which could for example contain your prompt or a back and forth conversation between the user and an assistant.

This example shows how you may call the Chat Completions API providing relevant tools and messages for an assistant that handles customer inquiries for a store.

```python
tools = [
  {
      "type": "function",
      "function": {
          "name": "get_delivery_date",
          "description": "Get the delivery date for a customer's order. Call this whenever you need to know the delivery date, for example when a customer asks 'Where is my package'",
          "parameters": {
              "type": "object",
              "properties": {
                  "order_id": {
                      "type": "string",
                      "description": "The customer's order ID.",
                  },
              },
              "required": ["order_id"],
              "additionalProperties": False,
          },
      }
  }
]

messages = [
  {
      "role": "system",
      "content": "You are a helpful customer support assistant. Use the supplied tools to assist the user."
  },
  {
      "role": "user",
      "content": "Hi, can you tell me the delivery date for my order?"
  }
]

response = openai.chat.completions.create(
  model="gpt-4o",
  messages=messages,
  tools=tools,
)
```

```javascript
const tools = [
  {
      type: "function",
      function: {
          name: "get_delivery_date",
          description: "Get the delivery date for a customer's order. Call this whenever you need to know the delivery date, for example when a customer asks 'Where is my package'",
          parameters: {
              type: "object",
              properties: {
                  order_id: {
                      type: "string",
                      description: "The customer's order ID.",
                  },
              },
              required: ["order_id"],
              additionalProperties: false,
          },
      },
  },
];

const messages = [
  {
      role: "system",
      content: "You are a helpful customer support assistant. Use the supplied tools to assist the user."
  },
  {
      role: "user",
      content: "Hi, can you tell me the delivery date for my order?"
  }
];

const response = await openai.chat.completions.create({
  model: "gpt-4o",
  messages,
  tools,
});
```

### Model instructions

While you should define in the function definitions how to call them, we recommend including instructions regarding when to call functions in the system prompt.

For example, you can tell the model when to use the function by saying something like: `"Use the 'get_delivery_date' function when the user asks about their delivery date."`

### Handling model responses

The model only suggests function calls and generates arguments for the defined functions when appropriate. It is then up to you to decide how your application handles the execution of the functions based on these suggestions.

If the model determines that a function should be called, it will return a `tool_calls` field in the response, which you can use to determine if the model generated a function call and what the arguments were.

Unless you customize the tool calling behavior, the model will determine when to call functions based on the instructions and conversation.

Read the [Tool calling behavior](#tool-calling-behavior) section below for more details on how you can force the model to call one or several tools.

#### If the model decides that no function should be called

If the model does not generate a function call, then the response will contain a direct reply to the user as a regular chat completion response.

For example, in this case `chat_response.choices[0].message` may contain:

```python
chat.completionsMessage(
  content='Hi there! I can help with that. Can you please provide your order ID?',
  role='assistant', 
  function_call=None, 
  tool_calls=None
)
```

```javascript
{
  role: 'assistant',
  content: "I'd be happy to help with that. Could you please provide me with your order ID?",
}
```

In an assistant use case you will typically want to show this response to the user and let them respond to it, in which case you will call the API again (with both the latest responses from the assistant and user appended to the `messages`).

Let's assume our user responded with their order id, and we sent the following request to the API.

```python
tools = [
  {
      "type": "function",
      "function": {
          "name": "get_delivery_date",
          "description": "Get the delivery date for a customer's order. Call this whenever you need to know the delivery date, for example when a customer asks 'Where is my package'",
          "parameters": {
              "type": "object",
              "properties": {
                  "order_id": {
                      "type": "string",
                      "description": "The customer's order ID."
                  }
              },
              "required": ["order_id"],
              "additionalProperties": False
          }
      }
  }
]

messages = []
messages.append({"role": "system", "content": "You are a helpful customer support assistant. Use the supplied tools to assist the user."})
messages.append({"role": "user", "content": "Hi, can you tell me the delivery date for my order?"})
messages.append({"role": "assistant", "content": "Hi there! I can help with that. Can you please provide your order ID?"})
messages.append({"role": "user", "content": "i think it is order_12345"})

response = client.chat.completions.create(
  model='gpt-4o',
  messages=messages,
  tools=tools
)
```

```javascript
const tools = [
  {
      type: "function",
      function: {
          name: "get_delivery_date",
          description: "Get the delivery date for a customer's order. Call this whenever you need to know the delivery date, for example when a customer asks 'Where is my package'",
          parameters: {
              type: "object",
              properties: {
                  order_id: {
                      type: "string",
                      escription: "The customer's order ID."
                  }
              },
              required: ["order_id"],
              additionalProperties: false,
          },
      },
  },
];

const messages = [];
messages.push({ role: "system", content: "You are a helpful customer support assistant. Use the supplied tools to assist the user." });
messages.push({ role: "user", content: "Hi, can you tell me the delivery date for my order?" });
messages.push({ role: "assistant", content: "Hi there! I can help with that. Can you please provide your order ID?" });
messages.push({ role: "user", content: "i think it is order_12345" });

const response = await client.chat.completions.create({
  model: 'gpt-4o',
  messages,
  tools,
});
```

#### If the model generated a function call

If the model generated a function call, it will generate the arguments for the call (based on the `parameters` definition you provided).

Here is an example response showing this:

```python
Choice(
  finish_reason='tool_calls', 
  index=0, 
  logprobs=None, 
  message=chat.completionsMessage(
      content=None, 
      role='assistant', 
      function_call=None, 
      tool_calls=[
          chat.completionsMessageToolCall(
              id='call_62136354', 
              function=Function(
                  arguments='{"order_id":"order_12345"}', 
                  name='get_delivery_date'), 
              type='function')
      ])
)
```

```javascript
{
  finish_reason: 'tool_calls',
  index: 0,
  logprobs: null,
  message: {
      content: null,
      role: 'assistant',
      function_call: null,
      tool_calls: [
          {
              id: 'call_62136354',
              function: {
                  arguments: '{"order_id":"order_12345"}',
                  name: 'get_delivery_date'
              },
              type: 'function'
          }
      ]
  }
}
```

#### Handling the model response indicating that a function should be called

Assuming the response indicates that a function should be called, your code will now handle this:

```python
# Extract the arguments for get_delivery_date
# Note this code assumes we have already determined that the model generated a function call. See below for a more production ready example that shows how to check if the model generated a function call
tool_call = response.choices[0].message.tool_calls[0]
arguments = json.loads(tool_call['function']['arguments'])

order_id = arguments.get('order_id')

# Call the get_delivery_date function with the extracted order_id

delivery_date = get_delivery_date(order_id)
```

```javascript
// Extract the arguments for get_delivery_date
// Note this code assumes we have already determined that the model generated a function call. See below for a more production ready example that shows how to check if the model generated a function call
const toolCall = response.choices[0].message.tool_calls[0];
const arguments = JSON.parse(toolCall.function.arguments);

const order_id = arguments.order_id;

// Call the get_delivery_date function with the extracted order_id
const delivery_date = get_delivery_date(order_id);
```

### Submitting function output

Once the function has been executed in the code, you need to submit the result of the function call back to the model.

This will trigger another model response, taking into account the function call result.

For example, this is how you can commit the result of the function call to a conversation history:

```python
# Simulate the order_id and delivery_date
order_id = "order_12345"
delivery_date = datetime.now()

# Simulate the tool call response

response = {
  "choices": [
      {
          "message": {
              "role": "assistant",
              "tool_calls": [
                  {
                      "id": "call_62136354",
                      "type": "function",
                      "function": {
                          "arguments": "{'order_id': 'order_12345'}",
                          "name": "get_delivery_date"
                      }
                  }
              ]
          }
      }
  ]
}

# Create a message containing the result of the function call

function_call_result_message = {
  "role": "tool",
  "content": json.dumps({
      "order_id": order_id,
      "delivery_date": delivery_date.strftime('%Y-%m-%d %H:%M:%S')
  }),
  "tool_call_id": response['choices'][0]['message']['tool_calls'][0]['id']
}

# Prepare the chat completion call payload

completion_payload = {
  "model": "gpt-4o",
  "messages": [
      {"role": "system", "content": "You are a helpful customer support assistant. Use the supplied tools to assist the user."},
      {"role": "user", "content": "Hi, can you tell me the delivery date for my order?"},
      {"role": "assistant", "content": "Hi there! I can help with that. Can you please provide your order ID?"},
      {"role": "user", "content": "i think it is order_12345"},
      response['choices'][0]['message'],
      function_call_result_message
  ]
}

# Call the OpenAI API's chat completions endpoint to send the tool call result back to the model

response = openai.chat.completions.create(
  model=completion_payload["model"],
  messages=completion_payload["messages"]
)

# Print the response from the API. In this case it will typically contain a message such as "The delivery date for your order #12345 is xyz. Is there anything else I can help you with?"

print(response)
```

```javascript
// Simulate the order_id and delivery_date
const order_id = "order_12345";
const delivery_date = moment();

// Simulate the tool call response
const response = {
  choices: [
      {
          message: {
              tool_calls: [
                  { id: "tool_call_1" }
              ]
          }
      }
  ]
};

// Create a message containing the result of the function call
const function_call_result_message = {
  role: "tool",
  content: JSON.stringify({
      order_id: order_id,
      delivery_date: delivery_date.format('YYYY-MM-DD HH:mm:ss')
  }),
  tool_call_id: response.choices[0].message.tool_calls[0].id
};

// Prepare the chat completion call payload
const completion_payload = {
  model: "gpt-4o",
  messages: [
      { role: "system", content: "You are a helpful customer support assistant. Use the supplied tools to assist the user." },
      { role: "user", content: "Hi, can you tell me the delivery date for my order?" },
      { role: "assistant", content: "Hi there! I can help with that. Can you please provide your order ID?" },
      { role: "user", content: "i think it is order_12345" },
      response.choices[0].message,
      function_call_result_message
  ]
};

// Call the OpenAI API's chat completions endpoint to send the tool call result back to the model
const final_response = await openai.chat.completions.create({
  model: completion_payload.model,
  messages: completion_payload.messages
});

// Print the response from the API. In this case it will typically contain a message such as "The delivery date for your order #12345 is xyz. Is there anything else I can help you with?"
console.log(final_response);
```

Note that an assistant message containing tool calls should always be followed by tool response messages (one per tool call). Making an API call with a messages array that does not follow this pattern will result in an error.

Structured Outputs
------------------

In August 2024, we launched Structured Outputs, which ensures that a model's output exactly matches a specified JSON schema.

By default, when using function calling, the API will offer best-effort matching for your parameters, which means that occasionally the model may miss parameters or get their types wrong when using complicated schemas.

You can enable Structured Outputs for function calling by setting the parameter `strict: true` in your function definition. You should also include the parameter `additionalProperties: false` and mark all arguments as required in your request. When this is enabled, the function arguments generated by the model will be constrained to match the JSON Schema provided in the function definition.

As an alternative to function calling you can instead constrain the model's regular output to match a JSON Schema of your choosing. [Learn more](/docs/guides/structured-outputs#function-calling-vs-response-format) about when to use function calling vs when to control the model's normal output by using `response_format`.

### Parallel function calling and Structured Outputs

When the model outputs multiple function calls via parallel function calling, model outputs may not match strict schemas supplied in tools.

In order to ensure strict schema adherence, disable parallel function calls by supplying `parallel_tool_calls: false`. With this setting, the model will generate one function call at a time.

### Why might I not want to turn on Structured Outputs?

The main reasons to not use Structured Outputs are:

*   If you need to use some feature of JSON Schema that is not yet supported ([learn more](/docs/guides/structured-outputs#supported-schemas)), for example recursive schemas.
*   If each of your API requests includes a novel schema (i.e. your schemas are not fixed, but are generated on-demand and rarely repeat). The first request with a novel JSON schema will have increased latency as the schema is pre-processed and cached for future generations to constrain the output of the model.

### What does Structured Outputs mean for Zero Data Retention?

When Structured Outputs is turned on, schemas provided are not eligible for [zero data retention](/docs/models#how-we-use-your-data).

### Supported schemas

Function calling with Structured Outputs supports a subset of the JSON Schema language.

For more information on supported schemas, see the [Structured Outputs guide](/docs/guides/structured-outputs#supported-schemas).

### Example

You can use zod in nodeJS and Pydantic in python when using the SDKs to pass your function definitions to the model.

```javascript
import OpenAI from "openai";
import { z } from "zod";
import { zodFunction } from "openai/helpers/zod";

const OrderParameters = z.object({
  order_id: z.string().describe("The customer's order ID."),
});

const tools = [
  zodFunction({ name: "getDeliveryDate", parameters: OrderParameters }),
];

const messages = [
  {
    role: "system",
    content: "You are a helpful customer support assistant. Use the supplied tools to assist the user.",
  },
  {
    role: "user",
    content: "Hi, can you tell me the delivery date for my order #12345?",
  },
];

const openai = new OpenAI();

const response = await openai.beta.chat.completions.create({
  model: "gpt-4o-2024-08-06",
  messages,
  tools,
});

console.log(response.choices[0].message.tool_calls?.[0].function);
```

```python
from enum import Enum
from typing import Union
from pydantic import BaseModel
import openai
from openai import OpenAI

client = OpenAI()

class GetDeliveryDate(BaseModel):
    order_id: str

tools = [openai.pydantic_function_tool(GetDeliveryDate)]

messages = []
messages.append({"role": "system", "content": "You are a helpful customer support assistant. Use the supplied tools to assist the user."})
messages.append({"role": "user", "content": "Hi, can you tell me the delivery date for my order #12345?"})

response = client.beta.chat.completions.create(
    model='gpt-4o-2024-08-06',
    messages=messages,
    tools=tools
)

print(response.choices[0].message.tool_calls[0].function)
```

```bash
curl https://api.openai.com/v1/chat/completions \
    -H "Authorization: Bearer $OPENAI_API_KEY" \
    -H "Content-Type: application/json" \
    -d '{
        "model": "gpt-4o-2024-08-06",
        "messages": [
            {
                "role": "system",
                "content": "You are a helpful customer support assistant. Use the supplied tools to assist the user."
            },
            {
                "role": "user",
                "content": "Hi, can you tell me the delivery date for my order #12345?"
            }
        ],
        "tools": [
            {
                "type": "function",
                "function": {
                    "name": "get_delivery_date",
                    "description": "Get the delivery date for a customer’s order. Call this whenever you need to know the delivery date, for example when a customer asks \"Where is my package\"",
                    "parameters": {
                        "type": "object",
                        "properties": {
                            "order_id": {
                                "type": "string",
                                "description": "The customer’s order ID."
                            }
                        },
                        "required": ["order_id"],
                        "additionalProperties": false
                    }
                },
                "strict": true
            }
        ]
    }'
```

If you are not using the SDKs, add the `strict: true` parameter to your function definition. Additionally, all parameters must be included in the `required` array, and `additionalProperties: false` must be set.

```json
{
    "name": "get_delivery_date",
    "description": "Get the delivery date for a customer's order. Call this whenever you need to know the delivery date, for example when a customer asks \\"Where is my package\\"",
    "strict": true,
    "parameters": {
        "type": "object",
        "properties": {
            "order_id": { "type": "string" }
        },
        "required": ["order_id"],
        "additionalProperties": false,
    }
}
```

### Limitations

When you use Structured Outputs with function calling, the model will always follow your exact schema, except in a few circumstances:

*   When the model's response is cut off (either due to `max_tokens`, `stop_tokens`, or maximum context length)
*   When a model [refusal](/docs/guides/structured-outputs#refusals) happens
*   When there is a `content_filter` finish reason

Note that the first time you send a request with a new schema using Structured Outputs, there will be additional latency as the schema is processed, but subsequent requests should incur no overhead.

Advanced usage
--------------

### Streaming tool calls

You can stream tool calls and process function arguments as they are being generated. This is especially useful if you want to display the function arguments in your UI, or if you don't need to wait for the whole function parameters to be generated before executing the function.

To enable streaming tool calls, you can set `stream: true` in your request. You can then process the streaming delta and check for any new tool calls delta.

You can find more information on streaming in the [API reference](/docs/api-reference/streaming).

Here is an example of how you can handle streaming tool calls with the node and python SDKs:

```python
from openai import OpenAI
import json

client = OpenAI()

# Define functions
tools = [
  {
      "type": "function",
      "function": {
          "name": "generate_recipe",
          "description": "Generate a recipe based on the user's input",
          "parameters": {
              "type": "object",
              "properties": {
                  "title": {
                      "type": "string",
                      "description": "Title of the recipe.",
                  },
                  "ingredients": {
                      "type": "array",
                      "items": {"type": "string"},
                      "description": "List of ingredients required for the recipe.",
                  },
                  "instructions": {
                      "type": "array",
                      "items": {"type": "string"},
                      "description": "Step-by-step instructions for the recipe.",
                  },
              },
              "required": ["title", "ingredients", "instructions"],
              "additionalProperties": False,
          },
      },
  }
]

response_stream = client.chat.completions.create(
  model="gpt-4o",
  messages=[
      {
          "role": "system",
          "content": (
              "You are an expert cook who can help turn any user input into a delicious recipe."
              "As soon as the user tells you what they want, use the generate_recipe tool to create a detailed recipe for them."
          ),
      },
      {
          "role": "user",
          "content": "I want to make pancakes for 4.",
      },
  ],
  tools=tools,
  stream=True,
)

function_arguments = ""
function_name = ""
is_collecting_function_args = False

for part in response_stream:
  delta = part.choices[0].delta
  finish_reason = part.choices[0].finish_reason

  # Process assistant content
  if 'content' in delta:
      print("Assistant:", delta.content)

  if delta.tool_calls:
      is_collecting_function_args = True
      tool_call = delta.tool_calls[0]

      if tool_call.function.name:
          function_name = tool_call.function.name
          print(f"Function name: '{function_name}'")
      
      # Process function arguments delta
      if tool_call.function.arguments:
          function_arguments += tool_call.function.arguments
          print(f"Arguments: {function_arguments}")

  # Process tool call with complete arguments
  if finish_reason == "tool_calls" and is_collecting_function_args:
      print(f"Function call '{function_name}' is complete.")
      args = json.loads(function_arguments)
      print("Complete function arguments:")
      print(json.dumps(args, indent=2))
   
      # Reset for the next potential function call
      function_arguments = ""
      function_name = ""
      is_collecting_function_args = False
```

```javascript
import OpenAI from "openai";

const openai = new OpenAI();

// Define functions
const tools = [
{
  type: "function",
  function: {
    name: "generate_recipe",
    description: "Generate a recipe based on the user's input",
    parameters: {
      type: "object",
      properties: {
        title: {
          type: "string",
          description: "Title of the recipe.",
        },
        ingredients: {
          type: "array",
          items: { type: "string" },
          description: "List of ingredients required for the recipe.",
        },
        instructions: {
          type: "array",
          items: { type: "string" },
          description: "Step-by-step instructions for the recipe.",
        },
      },
      required: ["title", "ingredients", "instructions"],
      additionalProperties: false,
    },
  },
},
];

const responseStream = await openai.chat.completions.create({
model: "gpt-4o",
messages: [
  {
    role: "system",
    content:
      "You are an expert cook who can help turn any user input into a delicious recipe. As soon as the user tells you what they want, use the generate_recipe tool to create a detailed recipe for them.",
  },
  {
    role: "user",
    content: "I want to make pancakes for 4.",
  },
],
tools: tools,
stream: true,
});

let functionArguments = "";
let functionName = "";
let isCollectingFunctionArgs = false;

for await (const part of responseStream) {
const delta = part.choices[0].delta;
const finishReason = part.choices[0].finish_reason;

if (delta.content) {
  // Process assistant content
  console.log("Assistant:", delta.content);
}

if (delta.tool_calls) {
  isCollectingFunctionArgs = true;
  const toolCall = delta.tool_calls[0];

  if (toolCall.function?.name) {
    functionName = toolCall.function.name;
    console.log("Function name:", functionName);
  }

  if (toolCall.function?.arguments) {
    functionArguments += toolCall.function.arguments;
    // Process function arguments delta
    console.log(`Arguments: ${functionArguments}`);
  }
}

// Process tool call with complete arguments
if (finishReason === "tool_calls" && isCollectingFunctionArgs) {
  console.log(`Function call '${functionName}' is complete.`);

  const args = JSON.parse(functionArguments);
  console.log("Complete function arguments:", args);

  // Reset for the next potential function call
  functionArguments = "";
  functionName = "";
  isCollectingFunctionArgs = false;
}
}
```

### Tool calling behavior

The API supports advanced features such as parallel tool calling and the ability to force tool calls.

You can disable parallel tool calling by setting `parallel_tool_calls: false`.

Parallel tool calling

Any models released on or after Nov 6, 2023 may by default generate multiple tool calls in a single response, indicating that they should be called in parallel.

This is especially useful if executing the given functions takes a long time. For example, the model may call functions to get the weather in 3 different locations at the same time, which will result in a message with 3 function calls in the tool\_calls array.

Example response:

```python
response = Choice(
  finish_reason='tool_calls', 
  index=0, 
  logprobs=None, 
  message=chat.completionsMessage(
      content=None, 
      role='assistant', 
      function_call=None, 
      tool_calls=[
          chat.completionsMessageToolCall(
              id='call_62136355', 
              function=Function(
                  arguments='{"city":"New York"}', 
                  name='check_weather'), 
              type='function'),
          chat.completionsMessageToolCall(
              id='call_62136356', 
              function=Function(
                  arguments='{"city":"London"}', 
                  name='check_weather'), 
              type='function'),
          chat.completionsMessageToolCall(
              id='call_62136357', 
              function=Function(
                  arguments='{"city":"Tokyo"}', 
                  name='check_weather'), 
              type='function')
      ])
)

# Iterate through tool calls to handle each weather check

for tool_call in response.message.tool_calls:
  arguments = json.loads(tool_call.function.arguments)
  city = arguments['city']
  weather_info = check_weather(city)
  print(f"Weather in {city}: {weather_info}")
```

```javascript
const response = {
  finish_reason: 'tool_calls',
  index: 0,
  logprobs: null,
  message: {
      content: null,
      role: 'assistant',
      function_call: null,
      tool_calls: [
          {
              id: 'call_62136355',
              function: {
                  arguments: '{"city":"New York"}',
                  name: 'check_weather'
              },
              type: 'function'
          },
          {
              id: 'call_62136356',
              function: {
                  arguments: '{"city":"London"}',
                  name: 'check_weather'
              },
              type: 'function'
          },
          {
              id: 'call_62136357',
              function: {
                  arguments: '{"city":"Tokyo"}',
                  name: 'check_weather'
              },
              type: 'function'
          }
      ]
  }
};

// Iterate through tool calls to handle each weather check
response.message.tool_calls.forEach(tool_call => {
  const arguments = JSON.parse(tool_call.function.arguments);
  const city = arguments.city;
  check_weather(city).then(weather_info => {
      console.log(`Weather in ${city}: ${weather_info}`);
  });
});
```

Each function call in the array has a unique `id`.

Once you've executed these function calls in your application, you can provide the result back to the model by adding one new message to the conversation for each function call, each containing the result of one function call, with a `tool_call_id` referencing the `id` from `tool_calls`, for example:

```python
# Assume we have fetched the weather data from somewhere
weather_data = {
  "New York": {"temperature": "22°C", "condition": "Sunny"},
  "London": {"temperature": "15°C", "condition": "Cloudy"},
  "Tokyo": {"temperature": "25°C", "condition": "Rainy"}
}
  
# Prepare the chat completion call payload with inline function call result creation
completion_payload = {
  "model": "gpt-4o",
  "messages": [
      {"role": "system", "content": "You are a helpful assistant providing weather updates."},
      {"role": "user", "content": "Can you tell me the weather in New York, London, and Tokyo?"},
      # Append the original function calls to the conversation
      response['message'],
      # Include the result of the function calls
      {
          "role": "tool",
          "content": json.dumps({
              "city": "New York",
              "weather": weather_data["New York"]
          }),
          # Here we specify the tool_call_id that this result corresponds to
          "tool_call_id": response['message']['tool_calls'][0]['id']
      },
      {
          "role": "tool",
          "content": json.dumps({
              "city": "London",
              "weather": weather_data["London"]
          }),
          "tool_call_id": response['message']['tool_calls'][1]['id']
      },
      {
          "role": "tool",
          "content": json.dumps({
              "city": "Tokyo",
              "weather": weather_data["Tokyo"]
          }),
          "tool_call_id": response['message']['tool_calls'][2]['id']
      }
  ]
}
  
# Call the OpenAI API's chat completions endpoint to send the tool call result back to the model
response = openai.chat.completions.create(
  model=completion_payload["model"],
  messages=completion_payload["messages"]
)
  
# Print the response from the API, which will return something like "In New York the weather is..."
print(response)
```

```javascript
// Assume we have fetched the weather data from somewhere
const weather_data = {
  "New York": { "temperature": "22°C", "condition": "Sunny" },
  "London": { "temperature": "15°C", "condition": "Cloudy" },
  "Tokyo": { "temperature": "25°C", "condition": "Rainy" }
};

// Prepare the chat completion call payload with inline function call result creation
const completion_payload = {
  model: "gpt-4o",
  messages: [
      { role: "system", content: "You are a helpful assistant providing weather updates." },
      { role: "user", content: "Can you tell me the weather in New York, London, and Tokyo?" },
      // Append the original function calls to the conversation
      response.message,
      // Include the result of the function calls
      {
          role: "tool",
          content: JSON.stringify({
              city: "New York",
              weather: weather_data["New York"]
          }),
          // Here we specify the tool_call_id that this result corresponds to
          tool_call_id: response.message.tool_calls[0].id
      },
      {
          role: "tool",
          content: JSON.stringify({
              city: "London",
              weather: weather_data["London"]
          }),
          tool_call_id: response.message.tool_calls[1].id
      },
      {
          role: "tool",
          content: JSON.stringify({
              city: "Tokyo",
              weather: weather_data["Tokyo"]
          }),
          tool_call_id: response.message.tool_calls[2].id
      }
  ]
};

// Call the OpenAI API's chat completions endpoint to send the tool call result back to the model
const response = await openai.chat.completions.create({
  model: completion_payload.model,
  messages: completion_payload.messages
});

// Print the response from the API, which will return something like "In New York the weather is..."
console.log(response);
```

Forcing tool calls

By default, the model is configured to automatically select which tools to call, as determined by the `tool_choice: "auto"` setting.

We offer three ways to customize the default behavior:

1.  To force the model to always call one or more tools, you can set `tool_choice: "required"`. The model will then always select one or more tool(s) to call. This is useful for example if you want the model to pick between multiple actions to perform next
2.  To force the model to call a specific function, you can set `tool_choice: {"type": "function", "function": {"name": "my_function"}}`
3.  To disable function calling and force the model to only generate a user-facing message, you can either provide no tools, or set `tool_choice: "none"`

Note that if you do either 1 or 2 (i.e. force the model to call a function) then the subsequent `finish_reason` will be `"stop"` instead of being `"tool_calls"`.

```python
from openai import OpenAI

client = OpenAI()

tools = [
  {
      "type": "function",
      "function": {
          "name": "get_weather",
          "strict": True,
          "parameters": {
              "type": "object",
              "properties": {
                  "location": {"type": "string"},
                  "unit": {"type": "string", "enum": ["c", "f"]},
              },
              "required": ["location", "unit"],
              "additionalProperties": False,
          },
      },
  },
  {
      "type": "function",
      "function": {
          "name": "get_stock_price",
          "strict": True,
          "parameters": {
              "type": "object",
              "properties": {
                  "symbol": {"type": "string"},
              },
              "required": ["symbol"],
              "additionalProperties": False,
          },
      },
  },
]

messages = [{"role": "user", "content": "What's the weather like in Boston today?"}]
completion = client.chat.completions.create(
  model="gpt-4o",
  messages=messages,
  tools=tools,
  tool_choice="required"
)

print(completion)
```

```javascript
import { OpenAI } from "openai";
const openai = new OpenAI();

// Define a set of tools to use
const tools = [
  {
      type: "function",
      function: {
          name: "get_weather",
          strict: true,
          parameters: {
              type: "object",
              properties: {
                  location: { type: "string" },
                  unit: { type: "string", enum: ["c", "f"] },
              },
              required: ["location", "unit"],
              additionalProperties: false,
          },
      },
  },
  {
      type: "function",
      function: {
          name: "get_stock_price",
          strict: true,
          parameters: {
              type: "object",
              properties: {
                  symbol: { type: "string" },
              },
              required: ["symbol"],
              additionalProperties: false,
          },
      },
  },
];

// Call the OpenAI API's chat completions endpoint
const response = await openai.chat.completions.create({
  model: "gpt-4o",
  messages: [
      {
          role: "user",
          content: "Can you tell me the weather in Tokyo?",
      },
  ],
  tool_choice: "required",
  tools,
});

// Print the response from the API
console.log(response);
```

To see a practical example of how to force tool calls, see our cookbook:

[

Customer service with tool required

Learn how to add an element of determinism to your customer service assistant

](https://cookbook.openai.com/examples/using_tool_required_for_customer_service)

### Edge cases

We recommend using the SDK to handle the edge cases described below. If for any reason you cannot use the SDK, you should handle these cases in your code.

When you receive a response from the API, if you're not using the SDK, there are a number of edge cases that production code should handle.

In general, the API will return a valid function call, but there are some edge cases when this won't happen:

*   When you have specified `max_tokens` and the model's response is cut off as a result
*   When the model's output includes copyrighted material

Also, when you force the model to call a function, the `finish_reason` will be `"stop"` instead of `"tool_calls"`.

This is how you can handle these different cases in your code:

```python
# Check if the conversation was too long for the context window
if response['choices'][0]['message']['finish_reason'] == "length":
  print("Error: The conversation was too long for the context window.")
  # Handle the error as needed, e.g., by truncating the conversation or asking for clarification
  handle_length_error(response)
  
# Check if the model's output included copyright material (or similar)
if response['choices'][0]['message']['finish_reason'] == "content_filter":
  print("Error: The content was filtered due to policy violations.")
  # Handle the error as needed, e.g., by modifying the request or notifying the user
  handle_content_filter_error(response)
  
# Check if the model has made a tool_call. This is the case either if the "finish_reason" is "tool_calls" or if the "finish_reason" is "stop" and our API request had forced a function call
if (response['choices'][0]['message']['finish_reason'] == "tool_calls" or 
  # This handles the edge case where if we forced the model to call one of our functions, the finish_reason will actually be "stop" instead of "tool_calls"
  (our_api_request_forced_a_tool_call and response['choices'][0]['message']['finish_reason'] == "stop")):
  # Handle tool call
  print("Model made a tool call.")
  # Your code to handle tool calls
  handle_tool_call(response)
  
# Else finish_reason is "stop", in which case the model was just responding directly to the user
elif response['choices'][0]['message']['finish_reason'] == "stop":
  # Handle the normal stop case
  print("Model responded directly to the user.")
  # Your code to handle normal responses
  handle_normal_response(response)
  
# Catch any other case, this is unexpected
else:
  print("Unexpected finish_reason:", response['choices'][0]['message']['finish_reason'])
  # Handle unexpected cases as needed
  handle_unexpected_case(response)
```

```javascript
// Check if the conversation was too long for the context window
if (response.choices[0].message.finish_reason === "length") {
  console.log("Error: The conversation was too long for the context window.");
  // Handle the error as needed, e.g., by truncating the conversation or asking for clarification
  handleLengthError(response);
}

// Check if the model's output included copyright material (or similar)
if (response.choices[0].message.finish_reason === "content_filter") {
  console.log("Error: The content was filtered due to policy violations.");
  // Handle the error as needed, e.g., by modifying the request or notifying the user
  handleContentFilterError(response);
}

// Check if the model has made a tool_call. This is the case either if the "finish_reason" is "tool_calls" or if the "finish_reason" is "stop" and our API request had forced a function call
if (
  response.choices[0].message.finish_reason === "tool_calls" ||
  (ourApiRequestForcedAToolCall && response.choices[0].message.finish_reason === "stop")
) {
  // Handle tool call
  console.log("Model made a tool call.");
  // Your code to handle tool calls
  handleToolCall(response);
}

// Else finish_reason is "stop", in which case the model was just responding directly to the user
else if (response.choices[0].message.finish_reason === "stop") {
  // Handle the normal stop case
  console.log("Model responded directly to the user.");
  // Your code to handle normal responses
  handleNormalResponse(response);
}

// Catch any other case, this is unexpected
else {
  console.log("Unexpected finish_reason:", response.choices[0].message.finish_reason);
  // Handle unexpected cases as needed
  handleUnexpectedCase(response);
}
```

### Token usage

Under the hood, functions are injected into the system message in a syntax the model has been trained on. This means functions count against the model's context limit and are billed as input tokens. If you run into token limits, we suggest limiting the number of functions or the length of the descriptions you provide for function parameters.

It is also possible to use fine-tuning to reduce the number of tokens used if you have many functions defined in your tools specification.

Examples
--------

The [OpenAI Cookbook](https://cookbook.openai.com/) has several end-to-end examples to help you implement function calling.

In our introductory cookbook [how to call functions with chat models](https://cookbook.openai.com/examples/how_to_call_functions_with_chat_models), we outline two examples of how the models can use function calling. This is a great resource to follow as you get started:

[

Function calling

Learn from more examples demonstrating function calling

](https://cookbook.openai.com/examples/how_to_call_functions_with_chat_models)

You will also find examples of function definitions for common use cases below.

Shopping Assistant

#### Scenario

A shopping assistant helps users navigate an e-commerce site. It needs to fetch product data from a structured database, and suggest recommendations based on the user's query. Once the user has found a product they are interested in, the assistant can add it to the shopping cart on their behalf.

#### Function definitions

To recommend products to the user, the assistant needs to query the products database. To find more details about a product, such as reviews and additional information (e.g. materials, dimensions...), the assistant needs to fetch more information on the specific product. It also needs a tool to add items to the shopping cart.

We will then define the following functions:

*   `get_product_recommendations`: Recommends products based on filters
*   `get_product_details`: Fetches more details about a product
*   `add_to_cart`: Adds a product to the shopping cart

```json
[
    {
        "type": "function",
        "function": {
            "name": "get_product_recommendations",
            "description": "Searches for products matching certain criteria in the database",
            "parameters": {
                "type": "object",
                "properties": {
                    "categories": {
                        "description": "categories that could be a match",
                        "type": "array",
                        "items": {
                            "type": "string",
                            "enum": [
                                "coats & jackets",
                                "accessories",
                                "tops",
                                "jeans & trousers",
                                "skirts & dresses",
                                "shoes"
                            ]
                        }
                    },
                    "colors": {
                        "description": "colors that could be a match, empty array if N/A",
                        "type": "array",
                        "items": {
                            "type": "string",
                            "enum": [
                                "black",
                                "white",
                                "brown",
                                "red",
                                "blue",
                                "green",
                                "orange",
                                "yellow",
                                "pink",
                                "gold",
                                "silver"
                            ]
                        }
                    },
                    "keywords": {
                        "description": "keywords that should be present in the item title or description",
                        "type": "array",
                        "items": {
                            "type": "string"
                        }
                    },
                    "price_range": {
                        "type": "object",
                        "properties": {
                            "min": {
                                "type": "number"
                            },
                            "max": {
                                "type": "number"
                            }
                        },
                        "required": [
                        "min",
                        "max"
                        ],
                        "additionalProperties": false
                    },
                    "limit": {
                        "type": "integer",
                        "description": "The maximum number of products to return, use 5 by default if nothing is specified by the user"
                    }
                },
                "required": [
                    "categories",
                    "colors",
                    "keywords",
                    "price_range",
                    "limit"
                ],
                "additionalProperties": false
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "get_product_details",
            "description": "Fetches more details about a product",
            "parameters": {
                "type": "object",
                "properties": {
                    "product_id": {
                        "type": "string",
                        "description": "The ID of the product to fetch details for"
                    }
                },
                "required": [
                    "product_id"
                ],
                "additionalProperties": false
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "add_to_cart",
            "description": "Add items to cart when the user has confirmed their interest.",
            "parameters": {
                "type": "object",
                "properties": {
                    "items": {
                        "type": "array",
                        "items": {
                            "type": "object",
                            "properties": {
                                "product_id": {
                                    "type": "string",
                                    "description": "ID of the product to add to the cart"
                                },
                                "quantity": {
                                    "type": "integer",
                                    "description": "Quantity of the product to add to the cart"
                                }
                            },
                            "required": [
                                "product_id",
                                "quantity"
                            ],
                            "additionalProperties": false
                        }
                    },
                    "required": [
                        "items"
                    ],
                    "additionalProperties": false
                }
            }
        }
    }
]
```

Customer Service Agent

#### Scenario

A customer service assistant on an e-commerce site helps users after they have made a purchase. It can answer questions about their orders or the company policy regarding returns and refunds. It can also help customers process a return and give status updates.

#### Function definitions

To answer questions about orders, the assistant needs to fetch order details from the orders database. In case the user doesn't know their order number, the assistant should also be able to fetch the last orders for a given user. It should also be able to search the FAQ to respond to general questions. Lastly, it should be equipped with tools to process returns and find a return status.

We will then define the following functions:

*   `get_order_details`: Fetches details about a specific order
*   `get_user_orders`: Fetches the last orders for a given user
*   `search_faq`: Searches the FAQ for an answer to the user's question
*   `process_return`: Processes a return and creates a return label
*   `get_return_status`: Finds the status of a return

```json
[
    {
        "type": "function",
        "function": {
            "name": "get_order_details",
            "description": "Fetches details about a specific order",
            "parameters": {
                "type": "object",
                "properties": {
                    "order_id": {
                        "type": "string",
                        "description": "The ID of the order to fetch details for"
                    }
                },
                "required": [
                    "order_id"
                ],
                "additionalProperties": false
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "get_user_orders",
            "description": "Fetches the last orders for a given user",
            "parameters": {
                "type": "object",
                "properties": {
                    "user_id": {
                        "type": "string",
                        "description": "The ID of the user to fetch orders for"
                    },
                    "limit": {
                        "type": "integer",
                        "description": "The maximum number of orders to return, use 5 by default and increase the number if the relevant order is not found."
                    }
                },
                "required": [
                    "user_id", "limit"
                ],
                "additionalProperties": false
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "search_faq",
            "description": "Searches the FAQ for an answer to the user's question",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {
                        "type": "string",
                        "description": "The question to search the FAQ for"
                    }
                },
                "required": [
                    "query"
                ],
                "additionalProperties": false
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "process_return",
            "description": "Processes a return and creates a return label",
            "parameters": {
                "type": "object",
                "properties": {
                    "order_id": {
                        "type": "string",
                        "description": "The ID of the order to process a return for"
                    },
                    "items": {
                        "type": "array",
                        "description": "The items to return",
                        "items": {
                            "type": "object",
                            "properties": {
                                "product_id": {
                                    "type": "string",
                                    "description": "The ID of the product to return"
                                },
                                "quantity": {
                                    "type": "integer",
                                    "description": "The quantity of the product to return"
                                }
                            },
                            "required": [
                                "product_id",
                                "quantity"
                            ],
                            "additionalProperties": false
                        }
                    }
                },
                "required": [
                    "order_id",
                    "items"
                ],
                "additionalProperties": false
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "get_return_status",
            "description": "Finds the status of a return",
            "parameters": {
                "type": "object",
                "properties": {
                    "order_id": {
                        "type": "string",
                        "description": "The ID of the order to fetch the return status for"
                    }
                },
                "required": [
                    "order_id"
                ],
                "additionalProperties": false
            }
        }
    }
]
```

Interactive Booking Experience

#### Scenario

A user is on an interactive website to find a place to eat or stay. After they mention their preferences, the website updates to show recommendations on a map. Once they have found a place they are interested in, a booking is programmatically made on their behalf.

#### Function definitions

To be able to show recommendations, the app needs to first fetch recommendations based on the user's preferences, and then pin those recommendations on the map. To book a place, the assistant needs to fetch the availability of the place, and then create a booking on the user's behalf.

We will then define the following functions:

*   `get_recommendations`: Fetche recommendations based on the user's preferences
*   `show_on_map`: Place pins on the map
*   `fetch_availability`: Fetch the availability for a given place
*   `create_booking`: Create a booking on the user's behalf

```json
[
    {
        "type": "function",
        "function": {
            "name": "get_recommendations",
            "description": "Fetches recommendations based on the user's preferences",
            "parameters": {
                "type": "object",
                "properties": {
                    "type": {
                        "type": "string",
                        "description": "The type of place to search recommendations for",
                        "enum": ["restaurant", "hotel"]
                    },
                    "keywords": {
                        "type": "array",
                        "description": "Keywords that should be present in the recommendations",
                        "items": {
                            "type": "string"
                        }
                    },
                    "location": {
                        "type": "string",
                        "description": "The location to search recommendations for"
                    }
                },
                "required": [
                    "type",
                    "keywords",
                    "location"
                ],
                "additionalProperties": false
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "show_on_map",
            "description": "Places pins on the map for relevant locations",
            "parameters": {
                "type": "object",
                "properties": {
                    "pins": {
                        "type": "array",
                        "description": "The pins to place on the map",
                        "items": {
                            "type": "object",
                            "properties": {
                                "name": {
                                    "type": "string",
                                    "description": "The name of the place"
                                },
                                "coordinates": {
                                    "type": "object",
                                    "properties": {
                                        "latitude": { "type": "number" },
                                        "longitude": { "type": "number" }
                                    },
                                    "required": [
                                        "latitude",
                                        "longitude"
                                    ],
                                    "additionalProperties": false
                                }
                            },
                            "required": [
                                "name",
                                "coordinates"
                            ],
                            "additionalProperties": false
                        }
                    }
                },
                "required": [
                    "pins"
                ],
                "additionalProperties": false
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "fetch_availability",
            "description": "Fetches the availability for a given place",
            "parameters": {
                "type": "object",
                "properties": {
                    "place_id": {
                        "type": "string",
                        "description": "The ID of the place to fetch availability for"
                    }
                }
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "create_booking",
            "description": "Creates a booking on the user's behalf",
            "parameters": {
                "type": "object",
                "properties": {
                    "place_id": {
                        "type": "string",
                        "description": "The ID of the place to create a booking for"
                    },
                    "booking_details": {
                        "anyOf": [
                            {
                                "type": "object",
                                "description": "Restaurant booking with specific date and time",
                                "properties": {
                                    "date": {
                                        "type": "string",
                                        "description": "The date of the booking, in format YYYY-MM-DD"
                                    },
                                    "time": {
                                        "type": "string",
                                        "description": "The time of the booking, in format HH:MM"
                                    }
                                },
                                "required": [
                                    "date",
                                    "time"
                                ]
                            },
                            {
                                "type": "object",
                                "description": "Hotel booking with specific check-in and check-out dates",
                                "properties": {
                                    "check_in": {
                                        "type": "string",
                                        "description": "The check-in date of the booking, in format YYYY-MM-DD"
                                    },
                                    "check_out": {
                                        "type": "string",
                                        "description": "The check-out date of the booking, in format YYYY-MM-DD"
                                    }
                                },
                                "required": [
                                    "check_in",
                                    "check_out"
                                ]
                            }
                        ]
                    }
                },
                "required": [
                    "place_id",
                    "booking_details"
                ],
                "additionalProperties": false
            }
        }
    }
]
```

Best practices
--------------

### Turn on Structured Outputs by setting `strict: "true"`

When Structured Outputs is turned on, the arguments generated by the model for function calls will reliably match the JSON Schema that you provide.

If you are not using Structured Outputs, then the structure of arguments is not guaranteed to be correct, so we recommend the use of a validation library like Pydantic to first verify the arguments prior to using them.

### Name functions intuitively, with detailed descriptions

If you find the model does not generate calls to the correct functions, you may need to update your function names and descriptions so the model more clearly understands when it should select each function. Avoid using abbreviations or acronyms to shorten function and argument names.

You can also include detailed descriptions for when a function should be called. For complex functions, you should include descriptions for each of the arguments to help the model know what it needs to ask the user to collect that argument.

### Name function parameters intuitively, with detailed descriptions

Use clear and descriptive names for function parameters. If applicable, specify the expected format for a parameter in the description (e.g., YYYY-mm-dd or dd/mm/yy for a date).

### Consider providing additional information about how and when to call functions in your system message

Providing clear instructions in your system message can significantly improve the model's function calling accuracy. For example, guide the model with instructions like the following:

`"Use check_order_status when the user inquires about the status of their order, such as 'Where is my order?' or 'Has my order shipped yet?'".`

Provide context for complex scenarios. For example:

`"Before scheduling a meeting with schedule_meeting, check the user's calendar for availability using check_availability to avoid conflicts."`

### Use enums for function arguments when possible

If your use case allows, you can use enums to constrain the possible values for arguments. This can help reduce hallucinations.

For example, if you have an AI assistant that helps with ordering a T-shirt, you likely have a fixed set of sizes for the T-shirt that you might want to constrain the model to choose from. If you want the model to output “s”, “m”, “l”, for small, medium, and large, you could provide those values in an enum, like this:

```json
{
    "name": "pick_tshirt_size",
    "description": "Call this if the user specifies which size t-shirt they want",
    "parameters": {
        "type": "object",
        "properties": {
            "size": {
                "type": "string",
                "enum": ["s", "m", "l"],
                "description": "The size of the t-shirt that the user would like to order"
            }
        },
        "required": ["size"],
        "additionalProperties": false
    }
}
```

If you don't constrain the output, a user may say “large” or “L”, and the model may use these values as a parameter. As your code may expect a specific structure, it's helpful to limit the possible values the model can choose from.

### Keep the number of functions low for higher accuracy

We recommend that you use no more than 20 tools in a single API call.

Developers typically see a reduction in the model's ability to select the correct tool once they have between 10-20 tools defined.

If your use case requires the model to be able to pick between a large number of custom functions, you may want to explore fine-tuning ([learn more](https://cookbook.openai.com/examples/fine_tuning_for_function_calling)) or break out the tools and group them logically to create a multi-agent system.

### Set up evals to act as an aid in prompt engineering your function definitions and system messages

We recommend for non-trivial uses of function calling setting up an evaluation system that allow you to measure how frequently the correct function is called or correct arguments are generated for a wide variety of possible user messages. Learn more about setting up evals on the [OpenAI Cookbook](https://cookbook.openai.com/examples/evaluation/getting_started_with_openai_evals).

You can then use these to measure whether adjustments to your function definitions and system messages will improve or hurt your integration.

### Fine-tuning may help improve accuracy for function calling

Fine-tuning a model can improve performance at function calling for your use case, especially if you have a large number of functions, or complex, nuanced or similar functions.

See our fine-tuning for function calling [cookbook](https://cookbook.openai.com/examples/fine_tuning_for_function_calling) for more information.

[

Fine-tuning for function calling

Learn how to fine-tune a model for function calling

](https://cookbook.openai.com/examples/fine_tuning_for_function_calling)

===

Function calling
================

Connect models to external data and systems.

**Function calling** enables developers to connect language models to external data and systems. You can define a set of functions as tools that the model has access to, and it can use them when appropriate based on the conversation history. You can then execute those functions on the application side, and provide results back to the model.

Learn how to extend the capabilities of OpenAI models through function calling in this guide.

Experiment with function calling in the [Playground](/playground) by providing your own function definition or generate a ready-to-use definition.

Generate

Overview
--------

Many applications require models to call custom functions to trigger actions within the application or interact with external systems.

Here is how you can define a function as a tool for the model to use:

```python
from openai import OpenAI

client = OpenAI()

tools = [
  {
      "type": "function",
      "function": {
          "name": "get_weather",
          "parameters": {
              "type": "object",
              "properties": {
                  "location": {"type": "string"}
              },
          },
      },
  }
]

completion = client.chat.completions.create(
  model="gpt-4o",
  messages=[{"role": "user", "content": "What's the weather like in Paris today?"}],
  tools=tools,
)

print(completion.choices[0].message.tool_calls)
```

```javascript
import { OpenAI } from "openai";
const openai = new OpenAI();

const tools = [
  {
      type: "function",
      function: {
          name: "get_weather",
          parameters: {
              type: "object",
              properties: {
                  location: { type: "string" }
              },
          },
      },
  }
];

const response = await openai.chat.completions.create({
  model: "gpt-4o",
  messages: [{"role": "user", "content": "What's the weather like in Paris today?"}],
  tools,
});

console.log(response.choices[0].message.tool_calls);
```

This will return a function call that looks like this:

```json
[
  {
    "id": "call_12345xyz",
    "type": "function",
    "function": { "name": "get_weather", "arguments": "{'location':'Paris'}" }
  }
]
```

Functions are the only type of tools supported in the Chat Completions API, but the Assistants API also supports [built-in tools](/docs/assistants/tools).

Here are a few examples where function calling can be useful:

1.  **Fetching data:** enable a conversational assistant to retrieve data from internal systems before responding to the user.
2.  **Taking action:** allow an assistant to trigger actions based on the conversation, like scheduling meetings or initiating order returns.
3.  **Building rich workflows:** allow assistants to execute multi-step workflows, like data extraction pipelines or content personalization.
4.  **Interacting with Application UIs:** use function calls to update the user interface based on user input, like rendering a pin on a map or navigating a website.

You can find example use cases in the [examples](#examples) section below.

### The lifecycle of a function call

When you use the OpenAI API with function calling, the model never actually executes functions itself - instead, it simply generates parameters that can be used to call your function. You are then responsible for handling how the function is executed in your code.

Read our [integration guide](#integration-guide) below for more details on how to handle function calls.

![Function Calling diagram](https://cdn.openai.com/API/docs/images/function-calling-diagram.png)

### Function calling support

Function calling is supported in the [Chat Completions API](/docs/guides/text-generation), [Assistants API](/docs/assistants/overview), [Batch API](/docs/guides/batch) and [Realtime API](/docs/guides/realtime).

This guide focuses on function calling using the Chat Completions API. We have separate guides for [function calling using the Assistants API](/docs/assistants/tools/function-calling), and for [function calling using the Realtime API](/docs/guides/realtime#tool-calling).

#### Models supporting function calling

Function calling was introduced with the release of `gpt-4-turbo` on June 13, 2023. All `gpt-*` models released after this date support function calling.

Legacy models released before this date were not trained to support function calling.

Support for parallel function calling

Parallel function calling is supported on models released on or after Nov 6, 2023. This includes: `gpt-4o`, `gpt-4o-2024-08-06`, `gpt-4o-2024-05-13`, `gpt-4o-mini`, `gpt-4o-mini-2024-07-18`, `gpt-4-turbo`, `gpt-4-turbo-2024-04-09`, `gpt-4-turbo-preview`, `gpt-4-0125-preview`, `gpt-4-1106-preview`, `gpt-3.5-turbo`, `gpt-3.5-turbo-0125`, and `gpt-3.5-turbo-1106`.

You can find a complete list of models and their release date on our [models page](/docs/models).

Integration guide
-----------------

In this integration guide, we will walk through integrating function calling into an application, taking an order delivery assistant as an example. Rather than requiring users to interact with a form, we can let them ask the assistant for help in natural language.

We will cover how to define functions and instructions, then how to handle model responses and function execution results.

If you want to learn more about how to handle function calls in a streaming fashion, how to customize tool calling behavior or how to handle edge cases, refer to our [advanced usage](#advanced-usage) section.

### Function definition

The starting point for function calling is choosing a function in your own codebase that you'd like to enable the model to generate arguments for.

For this example, let's imagine you want to allow the model to call the `get_delivery_date` function in your codebase which accepts an `order_id` and queries your database to determine the delivery date for a given package. Your function might look something like the following:

```python
# This is the function that we want the model to be able to call
def get_delivery_date(order_id: str) -> datetime:
  # Connect to the database
  conn = sqlite3.connect('ecommerce.db')
  cursor = conn.cursor()
  # ...
```

```javascript
// This is the function that we want the model to be able to call
const getDeliveryDate = async (orderId: string): datetime => { 
  const connection = await createConnection({
      host: 'localhost',
      user: 'root',
      // ...
  });
}
```

Now that we know which function we wish to allow the model to call, we will create a “function definition” that describes the function to the model. This definition describes both what the function does (and potentially when it should be called) and what parameters are required to call the function.

The `parameters` section of your function definition should be described using JSON Schema. If and when the model generates a function call, it will use this information to generate arguments according to your provided schema.

If you want to ensure the model always adheres to your supplied schema, you can enable [Structured Outputs](#structured-outputs) with function calling.

In this example it may look like this:

```json
{
    "name": "get_delivery_date",
    "description": "Get the delivery date for a customer's order. Call this whenever you need to know the delivery date, for example when a customer asks 'Where is my package'",
    "parameters": {
        "type": "object",
        "properties": {
            "order_id": {
                "type": "string",
                "description": "The customer's order ID."
            }
        },
        "required": ["order_id"],
        "additionalProperties": false
    }
}
```

Next we need to provide our function definitions within an array of available “tools” when calling the Chat Completions API.

As always, we will provide an array of “messages”, which could for example contain your prompt or a back and forth conversation between the user and an assistant.

This example shows how you may call the Chat Completions API providing relevant tools and messages for an assistant that handles customer inquiries for a store.

```python
tools = [
  {
      "type": "function",
      "function": {
          "name": "get_delivery_date",
          "description": "Get the delivery date for a customer's order. Call this whenever you need to know the delivery date, for example when a customer asks 'Where is my package'",
          "parameters": {
              "type": "object",
              "properties": {
                  "order_id": {
                      "type": "string",
                      "description": "The customer's order ID.",
                  },
              },
              "required": ["order_id"],
              "additionalProperties": False,
          },
      }
  }
]

messages = [
  {
      "role": "system",
      "content": "You are a helpful customer support assistant. Use the supplied tools to assist the user."
  },
  {
      "role": "user",
      "content": "Hi, can you tell me the delivery date for my order?"
  }
]

response = openai.chat.completions.create(
  model="gpt-4o",
  messages=messages,
  tools=tools,
)
```

```javascript
const tools = [
  {
      type: "function",
      function: {
          name: "get_delivery_date",
          description: "Get the delivery date for a customer's order. Call this whenever you need to know the delivery date, for example when a customer asks 'Where is my package'",
          parameters: {
              type: "object",
              properties: {
                  order_id: {
                      type: "string",
                      description: "The customer's order ID.",
                  },
              },
              required: ["order_id"],
              additionalProperties: false,
          },
      },
  },
];

const messages = [
  {
      role: "system",
      content: "You are a helpful customer support assistant. Use the supplied tools to assist the user."
  },
  {
      role: "user",
      content: "Hi, can you tell me the delivery date for my order?"
  }
];

const response = await openai.chat.completions.create({
  model: "gpt-4o",
  messages,
  tools,
});
```

### Model instructions

While you should define in the function definitions how to call them, we recommend including instructions regarding when to call functions in the system prompt.

For example, you can tell the model when to use the function by saying something like: `"Use the 'get_delivery_date' function when the user asks about their delivery date."`

### Handling model responses

The model only suggests function calls and generates arguments for the defined functions when appropriate. It is then up to you to decide how your application handles the execution of the functions based on these suggestions.

If the model determines that a function should be called, it will return a `tool_calls` field in the response, which you can use to determine if the model generated a function call and what the arguments were.

Unless you customize the tool calling behavior, the model will determine when to call functions based on the instructions and conversation.

Read the [Tool calling behavior](#tool-calling-behavior) section below for more details on how you can force the model to call one or several tools.

#### If the model decides that no function should be called

If the model does not generate a function call, then the response will contain a direct reply to the user as a regular chat completion response.

For example, in this case `chat_response.choices[0].message` may contain:

```python
chat.completionsMessage(
  content='Hi there! I can help with that. Can you please provide your order ID?',
  role='assistant', 
  function_call=None, 
  tool_calls=None
)
```

```javascript
{
  role: 'assistant',
  content: "I'd be happy to help with that. Could you please provide me with your order ID?",
}
```

In an assistant use case you will typically want to show this response to the user and let them respond to it, in which case you will call the API again (with both the latest responses from the assistant and user appended to the `messages`).

Let's assume our user responded with their order id, and we sent the following request to the API.

```python
tools = [
  {
      "type": "function",
      "function": {
          "name": "get_delivery_date",
          "description": "Get the delivery date for a customer's order. Call this whenever you need to know the delivery date, for example when a customer asks 'Where is my package'",
          "parameters": {
              "type": "object",
              "properties": {
                  "order_id": {
                      "type": "string",
                      "description": "The customer's order ID."
                  }
              },
              "required": ["order_id"],
              "additionalProperties": False
          }
      }
  }
]

messages = []
messages.append({"role": "system", "content": "You are a helpful customer support assistant. Use the supplied tools to assist the user."})
messages.append({"role": "user", "content": "Hi, can you tell me the delivery date for my order?"})
messages.append({"role": "assistant", "content": "Hi there! I can help with that. Can you please provide your order ID?"})
messages.append({"role": "user", "content": "i think it is order_12345"})

response = client.chat.completions.create(
  model='gpt-4o',
  messages=messages,
  tools=tools
)
```

```javascript
const tools = [
  {
      type: "function",
      function: {
          name: "get_delivery_date",
          description: "Get the delivery date for a customer's order. Call this whenever you need to know the delivery date, for example when a customer asks 'Where is my package'",
          parameters: {
              type: "object",
              properties: {
                  order_id: {
                      type: "string",
                      escription: "The customer's order ID."
                  }
              },
              required: ["order_id"],
              additionalProperties: false,
          },
      },
  },
];

const messages = [];
messages.push({ role: "system", content: "You are a helpful customer support assistant. Use the supplied tools to assist the user." });
messages.push({ role: "user", content: "Hi, can you tell me the delivery date for my order?" });
messages.push({ role: "assistant", content: "Hi there! I can help with that. Can you please provide your order ID?" });
messages.push({ role: "user", content: "i think it is order_12345" });

const response = await client.chat.completions.create({
  model: 'gpt-4o',
  messages,
  tools,
});
```

#### If the model generated a function call

If the model generated a function call, it will generate the arguments for the call (based on the `parameters` definition you provided).

Here is an example response showing this:

```python
Choice(
  finish_reason='tool_calls', 
  index=0, 
  logprobs=None, 
  message=chat.completionsMessage(
      content=None, 
      role='assistant', 
      function_call=None, 
      tool_calls=[
          chat.completionsMessageToolCall(
              id='call_62136354', 
              function=Function(
                  arguments='{"order_id":"order_12345"}', 
                  name='get_delivery_date'), 
              type='function')
      ])
)
```

```javascript
{
  finish_reason: 'tool_calls',
  index: 0,
  logprobs: null,
  message: {
      content: null,
      role: 'assistant',
      function_call: null,
      tool_calls: [
          {
              id: 'call_62136354',
              function: {
                  arguments: '{"order_id":"order_12345"}',
                  name: 'get_delivery_date'
              },
              type: 'function'
          }
      ]
  }
}
```

#### Handling the model response indicating that a function should be called

Assuming the response indicates that a function should be called, your code will now handle this:

```python
# Extract the arguments for get_delivery_date
# Note this code assumes we have already determined that the model generated a function call. See below for a more production ready example that shows how to check if the model generated a function call
tool_call = response.choices[0].message.tool_calls[0]
arguments = json.loads(tool_call['function']['arguments'])

order_id = arguments.get('order_id')

# Call the get_delivery_date function with the extracted order_id

delivery_date = get_delivery_date(order_id)
```

```javascript
// Extract the arguments for get_delivery_date
// Note this code assumes we have already determined that the model generated a function call. See below for a more production ready example that shows how to check if the model generated a function call
const toolCall = response.choices[0].message.tool_calls[0];
const arguments = JSON.parse(toolCall.function.arguments);

const order_id = arguments.order_id;

// Call the get_delivery_date function with the extracted order_id
const delivery_date = get_delivery_date(order_id);
```

### Submitting function output

Once the function has been executed in the code, you need to submit the result of the function call back to the model.

This will trigger another model response, taking into account the function call result.

For example, this is how you can commit the result of the function call to a conversation history:

```python
# Simulate the order_id and delivery_date
order_id = "order_12345"
delivery_date = datetime.now()

# Simulate the tool call response

response = {
  "choices": [
      {
          "message": {
              "role": "assistant",
              "tool_calls": [
                  {
                      "id": "call_62136354",
                      "type": "function",
                      "function": {
                          "arguments": "{'order_id': 'order_12345'}",
                          "name": "get_delivery_date"
                      }
                  }
              ]
          }
      }
  ]
}

# Create a message containing the result of the function call

function_call_result_message = {
  "role": "tool",
  "content": json.dumps({
      "order_id": order_id,
      "delivery_date": delivery_date.strftime('%Y-%m-%d %H:%M:%S')
  }),
  "tool_call_id": response['choices'][0]['message']['tool_calls'][0]['id']
}

# Prepare the chat completion call payload

completion_payload = {
  "model": "gpt-4o",
  "messages": [
      {"role": "system", "content": "You are a helpful customer support assistant. Use the supplied tools to assist the user."},
      {"role": "user", "content": "Hi, can you tell me the delivery date for my order?"},
      {"role": "assistant", "content": "Hi there! I can help with that. Can you please provide your order ID?"},
      {"role": "user", "content": "i think it is order_12345"},
      response['choices'][0]['message'],
      function_call_result_message
  ]
}

# Call the OpenAI API's chat completions endpoint to send the tool call result back to the model

response = openai.chat.completions.create(
  model=completion_payload["model"],
  messages=completion_payload["messages"]
)

# Print the response from the API. In this case it will typically contain a message such as "The delivery date for your order #12345 is xyz. Is there anything else I can help you with?"

print(response)
```

```javascript
// Simulate the order_id and delivery_date
const order_id = "order_12345";
const delivery_date = moment();

// Simulate the tool call response
const response = {
  choices: [
      {
          message: {
              tool_calls: [
                  { id: "tool_call_1" }
              ]
          }
      }
  ]
};

// Create a message containing the result of the function call
const function_call_result_message = {
  role: "tool",
  content: JSON.stringify({
      order_id: order_id,
      delivery_date: delivery_date.format('YYYY-MM-DD HH:mm:ss')
  }),
  tool_call_id: response.choices[0].message.tool_calls[0].id
};

// Prepare the chat completion call payload
const completion_payload = {
  model: "gpt-4o",
  messages: [
      { role: "system", content: "You are a helpful customer support assistant. Use the supplied tools to assist the user." },
      { role: "user", content: "Hi, can you tell me the delivery date for my order?" },
      { role: "assistant", content: "Hi there! I can help with that. Can you please provide your order ID?" },
      { role: "user", content: "i think it is order_12345" },
      response.choices[0].message,
      function_call_result_message
  ]
};

// Call the OpenAI API's chat completions endpoint to send the tool call result back to the model
const final_response = await openai.chat.completions.create({
  model: completion_payload.model,
  messages: completion_payload.messages
});

// Print the response from the API. In this case it will typically contain a message such as "The delivery date for your order #12345 is xyz. Is there anything else I can help you with?"
console.log(final_response);
```

Note that an assistant message containing tool calls should always be followed by tool response messages (one per tool call). Making an API call with a messages array that does not follow this pattern will result in an error.

Structured Outputs
------------------

In August 2024, we launched Structured Outputs, which ensures that a model's output exactly matches a specified JSON schema.

By default, when using function calling, the API will offer best-effort matching for your parameters, which means that occasionally the model may miss parameters or get their types wrong when using complicated schemas.

You can enable Structured Outputs for function calling by setting the parameter `strict: true` in your function definition. You should also include the parameter `additionalProperties: false` and mark all arguments as required in your request. When this is enabled, the function arguments generated by the model will be constrained to match the JSON Schema provided in the function definition.

As an alternative to function calling you can instead constrain the model's regular output to match a JSON Schema of your choosing. [Learn more](/docs/guides/structured-outputs#function-calling-vs-response-format) about when to use function calling vs when to control the model's normal output by using `response_format`.

### Parallel function calling and Structured Outputs

When the model outputs multiple function calls via parallel function calling, model outputs may not match strict schemas supplied in tools.

In order to ensure strict schema adherence, disable parallel function calls by supplying `parallel_tool_calls: false`. With this setting, the model will generate one function call at a time.

### Why might I not want to turn on Structured Outputs?

The main reasons to not use Structured Outputs are:

*   If you need to use some feature of JSON Schema that is not yet supported ([learn more](/docs/guides/structured-outputs#supported-schemas)), for example recursive schemas.
*   If each of your API requests includes a novel schema (i.e. your schemas are not fixed, but are generated on-demand and rarely repeat). The first request with a novel JSON schema will have increased latency as the schema is pre-processed and cached for future generations to constrain the output of the model.

### What does Structured Outputs mean for Zero Data Retention?

When Structured Outputs is turned on, schemas provided are not eligible for [zero data retention](/docs/models#how-we-use-your-data).

### Supported schemas

Function calling with Structured Outputs supports a subset of the JSON Schema language.

For more information on supported schemas, see the [Structured Outputs guide](/docs/guides/structured-outputs#supported-schemas).

### Example

You can use zod in nodeJS and Pydantic in python when using the SDKs to pass your function definitions to the model.

```javascript
import OpenAI from "openai";
import { z } from "zod";
import { zodFunction } from "openai/helpers/zod";

const OrderParameters = z.object({
  order_id: z.string().describe("The customer's order ID."),
});

const tools = [
  zodFunction({ name: "getDeliveryDate", parameters: OrderParameters }),
];

const messages = [
  {
    role: "system",
    content: "You are a helpful customer support assistant. Use the supplied tools to assist the user.",
  },
  {
    role: "user",
    content: "Hi, can you tell me the delivery date for my order #12345?",
  },
];

const openai = new OpenAI();

const response = await openai.beta.chat.completions.create({
  model: "gpt-4o-2024-08-06",
  messages,
  tools,
});

console.log(response.choices[0].message.tool_calls?.[0].function);
```

```python
from enum import Enum
from typing import Union
from pydantic import BaseModel
import openai
from openai import OpenAI

client = OpenAI()

class GetDeliveryDate(BaseModel):
    order_id: str

tools = [openai.pydantic_function_tool(GetDeliveryDate)]

messages = []
messages.append({"role": "system", "content": "You are a helpful customer support assistant. Use the supplied tools to assist the user."})
messages.append({"role": "user", "content": "Hi, can you tell me the delivery date for my order #12345?"})

response = client.beta.chat.completions.create(
    model='gpt-4o-2024-08-06',
    messages=messages,
    tools=tools
)

print(response.choices[0].message.tool_calls[0].function)
```

```bash
curl https://api.openai.com/v1/chat/completions \
    -H "Authorization: Bearer $OPENAI_API_KEY" \
    -H "Content-Type: application/json" \
    -d '{
        "model": "gpt-4o-2024-08-06",
        "messages": [
            {
                "role": "system",
                "content": "You are a helpful customer support assistant. Use the supplied tools to assist the user."
            },
            {
                "role": "user",
                "content": "Hi, can you tell me the delivery date for my order #12345?"
            }
        ],
        "tools": [
            {
                "type": "function",
                "function": {
                    "name": "get_delivery_date",
                    "description": "Get the delivery date for a customer’s order. Call this whenever you need to know the delivery date, for example when a customer asks \"Where is my package\"",
                    "parameters": {
                        "type": "object",
                        "properties": {
                            "order_id": {
                                "type": "string",
                                "description": "The customer’s order ID."
                            }
                        },
                        "required": ["order_id"],
                        "additionalProperties": false
                    }
                },
                "strict": true
            }
        ]
    }'
```

If you are not using the SDKs, add the `strict: true` parameter to your function definition. Additionally, all parameters must be included in the `required` array, and `additionalProperties: false` must be set.

```json
{
    "name": "get_delivery_date",
    "description": "Get the delivery date for a customer's order. Call this whenever you need to know the delivery date, for example when a customer asks \\"Where is my package\\"",
    "strict": true,
    "parameters": {
        "type": "object",
        "properties": {
            "order_id": { "type": "string" }
        },
        "required": ["order_id"],
        "additionalProperties": false,
    }
}
```

### Limitations

When you use Structured Outputs with function calling, the model will always follow your exact schema, except in a few circumstances:

*   When the model's response is cut off (either due to `max_tokens`, `stop_tokens`, or maximum context length)
*   When a model [refusal](/docs/guides/structured-outputs#refusals) happens
*   When there is a `content_filter` finish reason

Note that the first time you send a request with a new schema using Structured Outputs, there will be additional latency as the schema is processed, but subsequent requests should incur no overhead.

Advanced usage
--------------

### Streaming tool calls

You can stream tool calls and process function arguments as they are being generated. This is especially useful if you want to display the function arguments in your UI, or if you don't need to wait for the whole function parameters to be generated before executing the function.

To enable streaming tool calls, you can set `stream: true` in your request. You can then process the streaming delta and check for any new tool calls delta.

You can find more information on streaming in the [API reference](/docs/api-reference/streaming).

Here is an example of how you can handle streaming tool calls with the node and python SDKs:

```python
from openai import OpenAI
import json

client = OpenAI()

# Define functions
tools = [
  {
      "type": "function",
      "function": {
          "name": "generate_recipe",
          "description": "Generate a recipe based on the user's input",
          "parameters": {
              "type": "object",
              "properties": {
                  "title": {
                      "type": "string",
                      "description": "Title of the recipe.",
                  },
                  "ingredients": {
                      "type": "array",
                      "items": {"type": "string"},
                      "description": "List of ingredients required for the recipe.",
                  },
                  "instructions": {
                      "type": "array",
                      "items": {"type": "string"},
                      "description": "Step-by-step instructions for the recipe.",
                  },
              },
              "required": ["title", "ingredients", "instructions"],
              "additionalProperties": False,
          },
      },
  }
]

response_stream = client.chat.completions.create(
  model="gpt-4o",
  messages=[
      {
          "role": "system",
          "content": (
              "You are an expert cook who can help turn any user input into a delicious recipe."
              "As soon as the user tells you what they want, use the generate_recipe tool to create a detailed recipe for them."
          ),
      },
      {
          "role": "user",
          "content": "I want to make pancakes for 4.",
      },
  ],
  tools=tools,
  stream=True,
)

function_arguments = ""
function_name = ""
is_collecting_function_args = False

for part in response_stream:
  delta = part.choices[0].delta
  finish_reason = part.choices[0].finish_reason

  # Process assistant content
  if 'content' in delta:
      print("Assistant:", delta.content)

  if delta.tool_calls:
      is_collecting_function_args = True
      tool_call = delta.tool_calls[0]

      if tool_call.function.name:
          function_name = tool_call.function.name
          print(f"Function name: '{function_name}'")
      
      # Process function arguments delta
      if tool_call.function.arguments:
          function_arguments += tool_call.function.arguments
          print(f"Arguments: {function_arguments}")

  # Process tool call with complete arguments
  if finish_reason == "tool_calls" and is_collecting_function_args:
      print(f"Function call '{function_name}' is complete.")
      args = json.loads(function_arguments)
      print("Complete function arguments:")
      print(json.dumps(args, indent=2))
   
      # Reset for the next potential function call
      function_arguments = ""
      function_name = ""
      is_collecting_function_args = False
```

```javascript
import OpenAI from "openai";

const openai = new OpenAI();

// Define functions
const tools = [
{
  type: "function",
  function: {
    name: "generate_recipe",
    description: "Generate a recipe based on the user's input",
    parameters: {
      type: "object",
      properties: {
        title: {
          type: "string",
          description: "Title of the recipe.",
        },
        ingredients: {
          type: "array",
          items: { type: "string" },
          description: "List of ingredients required for the recipe.",
        },
        instructions: {
          type: "array",
          items: { type: "string" },
          description: "Step-by-step instructions for the recipe.",
        },
      },
      required: ["title", "ingredients", "instructions"],
      additionalProperties: false,
    },
  },
},
];

const responseStream = await openai.chat.completions.create({
model: "gpt-4o",
messages: [
  {
    role: "system",
    content:
      "You are an expert cook who can help turn any user input into a delicious recipe. As soon as the user tells you what they want, use the generate_recipe tool to create a detailed recipe for them.",
  },
  {
    role: "user",
    content: "I want to make pancakes for 4.",
  },
],
tools: tools,
stream: true,
});

let functionArguments = "";
let functionName = "";
let isCollectingFunctionArgs = false;

for await (const part of responseStream) {
const delta = part.choices[0].delta;
const finishReason = part.choices[0].finish_reason;

if (delta.content) {
  // Process assistant content
  console.log("Assistant:", delta.content);
}

if (delta.tool_calls) {
  isCollectingFunctionArgs = true;
  const toolCall = delta.tool_calls[0];

  if (toolCall.function?.name) {
    functionName = toolCall.function.name;
    console.log("Function name:", functionName);
  }

  if (toolCall.function?.arguments) {
    functionArguments += toolCall.function.arguments;
    // Process function arguments delta
    console.log(`Arguments: ${functionArguments}`);
  }
}

// Process tool call with complete arguments
if (finishReason === "tool_calls" && isCollectingFunctionArgs) {
  console.log(`Function call '${functionName}' is complete.`);

  const args = JSON.parse(functionArguments);
  console.log("Complete function arguments:", args);

  // Reset for the next potential function call
  functionArguments = "";
  functionName = "";
  isCollectingFunctionArgs = false;
}
}
```

### Tool calling behavior

The API supports advanced features such as parallel tool calling and the ability to force tool calls.

You can disable parallel tool calling by setting `parallel_tool_calls: false`.

Parallel tool calling

Any models released on or after Nov 6, 2023 may by default generate multiple tool calls in a single response, indicating that they should be called in parallel.

This is especially useful if executing the given functions takes a long time. For example, the model may call functions to get the weather in 3 different locations at the same time, which will result in a message with 3 function calls in the tool\_calls array.

Example response:

```python
response = Choice(
  finish_reason='tool_calls', 
  index=0, 
  logprobs=None, 
  message=chat.completionsMessage(
      content=None, 
      role='assistant', 
      function_call=None, 
      tool_calls=[
          chat.completionsMessageToolCall(
              id='call_62136355', 
              function=Function(
                  arguments='{"city":"New York"}', 
                  name='check_weather'), 
              type='function'),
          chat.completionsMessageToolCall(
              id='call_62136356', 
              function=Function(
                  arguments='{"city":"London"}', 
                  name='check_weather'), 
              type='function'),
          chat.completionsMessageToolCall(
              id='call_62136357', 
              function=Function(
                  arguments='{"city":"Tokyo"}', 
                  name='check_weather'), 
              type='function')
      ])
)

# Iterate through tool calls to handle each weather check

for tool_call in response.message.tool_calls:
  arguments = json.loads(tool_call.function.arguments)
  city = arguments['city']
  weather_info = check_weather(city)
  print(f"Weather in {city}: {weather_info}")
```

```javascript
const response = {
  finish_reason: 'tool_calls',
  index: 0,
  logprobs: null,
  message: {
      content: null,
      role: 'assistant',
      function_call: null,
      tool_calls: [
          {
              id: 'call_62136355',
              function: {
                  arguments: '{"city":"New York"}',
                  name: 'check_weather'
              },
              type: 'function'
          },
          {
              id: 'call_62136356',
              function: {
                  arguments: '{"city":"London"}',
                  name: 'check_weather'
              },
              type: 'function'
          },
          {
              id: 'call_62136357',
              function: {
                  arguments: '{"city":"Tokyo"}',
                  name: 'check_weather'
              },
              type: 'function'
          }
      ]
  }
};

// Iterate through tool calls to handle each weather check
response.message.tool_calls.forEach(tool_call => {
  const arguments = JSON.parse(tool_call.function.arguments);
  const city = arguments.city;
  check_weather(city).then(weather_info => {
      console.log(`Weather in ${city}: ${weather_info}`);
  });
});
```

Each function call in the array has a unique `id`.

Once you've executed these function calls in your application, you can provide the result back to the model by adding one new message to the conversation for each function call, each containing the result of one function call, with a `tool_call_id` referencing the `id` from `tool_calls`, for example:

```python
# Assume we have fetched the weather data from somewhere
weather_data = {
  "New York": {"temperature": "22°C", "condition": "Sunny"},
  "London": {"temperature": "15°C", "condition": "Cloudy"},
  "Tokyo": {"temperature": "25°C", "condition": "Rainy"}
}
  
# Prepare the chat completion call payload with inline function call result creation
completion_payload = {
  "model": "gpt-4o",
  "messages": [
      {"role": "system", "content": "You are a helpful assistant providing weather updates."},
      {"role": "user", "content": "Can you tell me the weather in New York, London, and Tokyo?"},
      # Append the original function calls to the conversation
      response['message'],
      # Include the result of the function calls
      {
          "role": "tool",
          "content": json.dumps({
              "city": "New York",
              "weather": weather_data["New York"]
          }),
          # Here we specify the tool_call_id that this result corresponds to
          "tool_call_id": response['message']['tool_calls'][0]['id']
      },
      {
          "role": "tool",
          "content": json.dumps({
              "city": "London",
              "weather": weather_data["London"]
          }),
          "tool_call_id": response['message']['tool_calls'][1]['id']
      },
      {
          "role": "tool",
          "content": json.dumps({
              "city": "Tokyo",
              "weather": weather_data["Tokyo"]
          }),
          "tool_call_id": response['message']['tool_calls'][2]['id']
      }
  ]
}
  
# Call the OpenAI API's chat completions endpoint to send the tool call result back to the model
response = openai.chat.completions.create(
  model=completion_payload["model"],
  messages=completion_payload["messages"]
)
  
# Print the response from the API, which will return something like "In New York the weather is..."
print(response)
```

```javascript
// Assume we have fetched the weather data from somewhere
const weather_data = {
  "New York": { "temperature": "22°C", "condition": "Sunny" },
  "London": { "temperature": "15°C", "condition": "Cloudy" },
  "Tokyo": { "temperature": "25°C", "condition": "Rainy" }
};

// Prepare the chat completion call payload with inline function call result creation
const completion_payload = {
  model: "gpt-4o",
  messages: [
      { role: "system", content: "You are a helpful assistant providing weather updates." },
      { role: "user", content: "Can you tell me the weather in New York, London, and Tokyo?" },
      // Append the original function calls to the conversation
      response.message,
      // Include the result of the function calls
      {
          role: "tool",
          content: JSON.stringify({
              city: "New York",
              weather: weather_data["New York"]
          }),
          // Here we specify the tool_call_id that this result corresponds to
          tool_call_id: response.message.tool_calls[0].id
      },
      {
          role: "tool",
          content: JSON.stringify({
              city: "London",
              weather: weather_data["London"]
          }),
          tool_call_id: response.message.tool_calls[1].id
      },
      {
          role: "tool",
          content: JSON.stringify({
              city: "Tokyo",
              weather: weather_data["Tokyo"]
          }),
          tool_call_id: response.message.tool_calls[2].id
      }
  ]
};

// Call the OpenAI API's chat completions endpoint to send the tool call result back to the model
const response = await openai.chat.completions.create({
  model: completion_payload.model,
  messages: completion_payload.messages
});

// Print the response from the API, which will return something like "In New York the weather is..."
console.log(response);
```

Forcing tool calls

By default, the model is configured to automatically select which tools to call, as determined by the `tool_choice: "auto"` setting.

We offer three ways to customize the default behavior:

1.  To force the model to always call one or more tools, you can set `tool_choice: "required"`. The model will then always select one or more tool(s) to call. This is useful for example if you want the model to pick between multiple actions to perform next
2.  To force the model to call a specific function, you can set `tool_choice: {"type": "function", "function": {"name": "my_function"}}`
3.  To disable function calling and force the model to only generate a user-facing message, you can either provide no tools, or set `tool_choice: "none"`

Note that if you do either 1 or 2 (i.e. force the model to call a function) then the subsequent `finish_reason` will be `"stop"` instead of being `"tool_calls"`.

```python
from openai import OpenAI

client = OpenAI()

tools = [
  {
      "type": "function",
      "function": {
          "name": "get_weather",
          "strict": True,
          "parameters": {
              "type": "object",
              "properties": {
                  "location": {"type": "string"},
                  "unit": {"type": "string", "enum": ["c", "f"]},
              },
              "required": ["location", "unit"],
              "additionalProperties": False,
          },
      },
  },
  {
      "type": "function",
      "function": {
          "name": "get_stock_price",
          "strict": True,
          "parameters": {
              "type": "object",
              "properties": {
                  "symbol": {"type": "string"},
              },
              "required": ["symbol"],
              "additionalProperties": False,
          },
      },
  },
]

messages = [{"role": "user", "content": "What's the weather like in Boston today?"}]
completion = client.chat.completions.create(
  model="gpt-4o",
  messages=messages,
  tools=tools,
  tool_choice="required"
)

print(completion)
```

```javascript
import { OpenAI } from "openai";
const openai = new OpenAI();

// Define a set of tools to use
const tools = [
  {
      type: "function",
      function: {
          name: "get_weather",
          strict: true,
          parameters: {
              type: "object",
              properties: {
                  location: { type: "string" },
                  unit: { type: "string", enum: ["c", "f"] },
              },
              required: ["location", "unit"],
              additionalProperties: false,
          },
      },
  },
  {
      type: "function",
      function: {
          name: "get_stock_price",
          strict: true,
          parameters: {
              type: "object",
              properties: {
                  symbol: { type: "string" },
              },
              required: ["symbol"],
              additionalProperties: false,
          },
      },
  },
];

// Call the OpenAI API's chat completions endpoint
const response = await openai.chat.completions.create({
  model: "gpt-4o",
  messages: [
      {
          role: "user",
          content: "Can you tell me the weather in Tokyo?",
      },
  ],
  tool_choice: "required",
  tools,
});

// Print the response from the API
console.log(response);
```

To see a practical example of how to force tool calls, see our cookbook:

[

Customer service with tool required

Learn how to add an element of determinism to your customer service assistant

](https://cookbook.openai.com/examples/using_tool_required_for_customer_service)

### Edge cases

We recommend using the SDK to handle the edge cases described below. If for any reason you cannot use the SDK, you should handle these cases in your code.

When you receive a response from the API, if you're not using the SDK, there are a number of edge cases that production code should handle.

In general, the API will return a valid function call, but there are some edge cases when this won't happen:

*   When you have specified `max_tokens` and the model's response is cut off as a result
*   When the model's output includes copyrighted material

Also, when you force the model to call a function, the `finish_reason` will be `"stop"` instead of `"tool_calls"`.

This is how you can handle these different cases in your code:

```python
# Check if the conversation was too long for the context window
if response['choices'][0]['message']['finish_reason'] == "length":
  print("Error: The conversation was too long for the context window.")
  # Handle the error as needed, e.g., by truncating the conversation or asking for clarification
  handle_length_error(response)
  
# Check if the model's output included copyright material (or similar)
if response['choices'][0]['message']['finish_reason'] == "content_filter":
  print("Error: The content was filtered due to policy violations.")
  # Handle the error as needed, e.g., by modifying the request or notifying the user
  handle_content_filter_error(response)
  
# Check if the model has made a tool_call. This is the case either if the "finish_reason" is "tool_calls" or if the "finish_reason" is "stop" and our API request had forced a function call
if (response['choices'][0]['message']['finish_reason'] == "tool_calls" or 
  # This handles the edge case where if we forced the model to call one of our functions, the finish_reason will actually be "stop" instead of "tool_calls"
  (our_api_request_forced_a_tool_call and response['choices'][0]['message']['finish_reason'] == "stop")):
  # Handle tool call
  print("Model made a tool call.")
  # Your code to handle tool calls
  handle_tool_call(response)
  
# Else finish_reason is "stop", in which case the model was just responding directly to the user
elif response['choices'][0]['message']['finish_reason'] == "stop":
  # Handle the normal stop case
  print("Model responded directly to the user.")
  # Your code to handle normal responses
  handle_normal_response(response)
  
# Catch any other case, this is unexpected
else:
  print("Unexpected finish_reason:", response['choices'][0]['message']['finish_reason'])
  # Handle unexpected cases as needed
  handle_unexpected_case(response)
```

```javascript
// Check if the conversation was too long for the context window
if (response.choices[0].message.finish_reason === "length") {
  console.log("Error: The conversation was too long for the context window.");
  // Handle the error as needed, e.g., by truncating the conversation or asking for clarification
  handleLengthError(response);
}

// Check if the model's output included copyright material (or similar)
if (response.choices[0].message.finish_reason === "content_filter") {
  console.log("Error: The content was filtered due to policy violations.");
  // Handle the error as needed, e.g., by modifying the request or notifying the user
  handleContentFilterError(response);
}

// Check if the model has made a tool_call. This is the case either if the "finish_reason" is "tool_calls" or if the "finish_reason" is "stop" and our API request had forced a function call
if (
  response.choices[0].message.finish_reason === "tool_calls" ||
  (ourApiRequestForcedAToolCall && response.choices[0].message.finish_reason === "stop")
) {
  // Handle tool call
  console.log("Model made a tool call.");
  // Your code to handle tool calls
  handleToolCall(response);
}

// Else finish_reason is "stop", in which case the model was just responding directly to the user
else if (response.choices[0].message.finish_reason === "stop") {
  // Handle the normal stop case
  console.log("Model responded directly to the user.");
  // Your code to handle normal responses
  handleNormalResponse(response);
}

// Catch any other case, this is unexpected
else {
  console.log("Unexpected finish_reason:", response.choices[0].message.finish_reason);
  // Handle unexpected cases as needed
  handleUnexpectedCase(response);
}
```

### Token usage

Under the hood, functions are injected into the system message in a syntax the model has been trained on. This means functions count against the model's context limit and are billed as input tokens. If you run into token limits, we suggest limiting the number of functions or the length of the descriptions you provide for function parameters.

It is also possible to use fine-tuning to reduce the number of tokens used if you have many functions defined in your tools specification.

Examples
--------

The [OpenAI Cookbook](https://cookbook.openai.com/) has several end-to-end examples to help you implement function calling.

In our introductory cookbook [how to call functions with chat models](https://cookbook.openai.com/examples/how_to_call_functions_with_chat_models), we outline two examples of how the models can use function calling. This is a great resource to follow as you get started:

[

Function calling

Learn from more examples demonstrating function calling

](https://cookbook.openai.com/examples/how_to_call_functions_with_chat_models)

You will also find examples of function definitions for common use cases below.

Shopping Assistant

#### Scenario

A shopping assistant helps users navigate an e-commerce site. It needs to fetch product data from a structured database, and suggest recommendations based on the user's query. Once the user has found a product they are interested in, the assistant can add it to the shopping cart on their behalf.

#### Function definitions

To recommend products to the user, the assistant needs to query the products database. To find more details about a product, such as reviews and additional information (e.g. materials, dimensions...), the assistant needs to fetch more information on the specific product. It also needs a tool to add items to the shopping cart.

We will then define the following functions:

*   `get_product_recommendations`: Recommends products based on filters
*   `get_product_details`: Fetches more details about a product
*   `add_to_cart`: Adds a product to the shopping cart

```json
[
    {
        "type": "function",
        "function": {
            "name": "get_product_recommendations",
            "description": "Searches for products matching certain criteria in the database",
            "parameters": {
                "type": "object",
                "properties": {
                    "categories": {
                        "description": "categories that could be a match",
                        "type": "array",
                        "items": {
                            "type": "string",
                            "enum": [
                                "coats & jackets",
                                "accessories",
                                "tops",
                                "jeans & trousers",
                                "skirts & dresses",
                                "shoes"
                            ]
                        }
                    },
                    "colors": {
                        "description": "colors that could be a match, empty array if N/A",
                        "type": "array",
                        "items": {
                            "type": "string",
                            "enum": [
                                "black",
                                "white",
                                "brown",
                                "red",
                                "blue",
                                "green",
                                "orange",
                                "yellow",
                                "pink",
                                "gold",
                                "silver"
                            ]
                        }
                    },
                    "keywords": {
                        "description": "keywords that should be present in the item title or description",
                        "type": "array",
                        "items": {
                            "type": "string"
                        }
                    },
                    "price_range": {
                        "type": "object",
                        "properties": {
                            "min": {
                                "type": "number"
                            },
                            "max": {
                                "type": "number"
                            }
                        },
                        "required": [
                        "min",
                        "max"
                        ],
                        "additionalProperties": false
                    },
                    "limit": {
                        "type": "integer",
                        "description": "The maximum number of products to return, use 5 by default if nothing is specified by the user"
                    }
                },
                "required": [
                    "categories",
                    "colors",
                    "keywords",
                    "price_range",
                    "limit"
                ],
                "additionalProperties": false
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "get_product_details",
            "description": "Fetches more details about a product",
            "parameters": {
                "type": "object",
                "properties": {
                    "product_id": {
                        "type": "string",
                        "description": "The ID of the product to fetch details for"
                    }
                },
                "required": [
                    "product_id"
                ],
                "additionalProperties": false
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "add_to_cart",
            "description": "Add items to cart when the user has confirmed their interest.",
            "parameters": {
                "type": "object",
                "properties": {
                    "items": {
                        "type": "array",
                        "items": {
                            "type": "object",
                            "properties": {
                                "product_id": {
                                    "type": "string",
                                    "description": "ID of the product to add to the cart"
                                },
                                "quantity": {
                                    "type": "integer",
                                    "description": "Quantity of the product to add to the cart"
                                }
                            },
                            "required": [
                                "product_id",
                                "quantity"
                            ],
                            "additionalProperties": false
                        }
                    },
                    "required": [
                        "items"
                    ],
                    "additionalProperties": false
                }
            }
        }
    }
]
```

Customer Service Agent

#### Scenario

A customer service assistant on an e-commerce site helps users after they have made a purchase. It can answer questions about their orders or the company policy regarding returns and refunds. It can also help customers process a return and give status updates.

#### Function definitions

To answer questions about orders, the assistant needs to fetch order details from the orders database. In case the user doesn't know their order number, the assistant should also be able to fetch the last orders for a given user. It should also be able to search the FAQ to respond to general questions. Lastly, it should be equipped with tools to process returns and find a return status.

We will then define the following functions:

*   `get_order_details`: Fetches details about a specific order
*   `get_user_orders`: Fetches the last orders for a given user
*   `search_faq`: Searches the FAQ for an answer to the user's question
*   `process_return`: Processes a return and creates a return label
*   `get_return_status`: Finds the status of a return

```json
[
    {
        "type": "function",
        "function": {
            "name": "get_order_details",
            "description": "Fetches details about a specific order",
            "parameters": {
                "type": "object",
                "properties": {
                    "order_id": {
                        "type": "string",
                        "description": "The ID of the order to fetch details for"
                    }
                },
                "required": [
                    "order_id"
                ],
                "additionalProperties": false
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "get_user_orders",
            "description": "Fetches the last orders for a given user",
            "parameters": {
                "type": "object",
                "properties": {
                    "user_id": {
                        "type": "string",
                        "description": "The ID of the user to fetch orders for"
                    },
                    "limit": {
                        "type": "integer",
                        "description": "The maximum number of orders to return, use 5 by default and increase the number if the relevant order is not found."
                    }
                },
                "required": [
                    "user_id", "limit"
                ],
                "additionalProperties": false
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "search_faq",
            "description": "Searches the FAQ for an answer to the user's question",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {
                        "type": "string",
                        "description": "The question to search the FAQ for"
                    }
                },
                "required": [
                    "query"
                ],
                "additionalProperties": false
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "process_return",
            "description": "Processes a return and creates a return label",
            "parameters": {
                "type": "object",
                "properties": {
                    "order_id": {
                        "type": "string",
                        "description": "The ID of the order to process a return for"
                    },
                    "items": {
                        "type": "array",
                        "description": "The items to return",
                        "items": {
                            "type": "object",
                            "properties": {
                                "product_id": {
                                    "type": "string",
                                    "description": "The ID of the product to return"
                                },
                                "quantity": {
                                    "type": "integer",
                                    "description": "The quantity of the product to return"
                                }
                            },
                            "required": [
                                "product_id",
                                "quantity"
                            ],
                            "additionalProperties": false
                        }
                    }
                },
                "required": [
                    "order_id",
                    "items"
                ],
                "additionalProperties": false
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "get_return_status",
            "description": "Finds the status of a return",
            "parameters": {
                "type": "object",
                "properties": {
                    "order_id": {
                        "type": "string",
                        "description": "The ID of the order to fetch the return status for"
                    }
                },
                "required": [
                    "order_id"
                ],
                "additionalProperties": false
            }
        }
    }
]
```

Interactive Booking Experience

#### Scenario

A user is on an interactive website to find a place to eat or stay. After they mention their preferences, the website updates to show recommendations on a map. Once they have found a place they are interested in, a booking is programmatically made on their behalf.

#### Function definitions

To be able to show recommendations, the app needs to first fetch recommendations based on the user's preferences, and then pin those recommendations on the map. To book a place, the assistant needs to fetch the availability of the place, and then create a booking on the user's behalf.

We will then define the following functions:

*   `get_recommendations`: Fetche recommendations based on the user's preferences
*   `show_on_map`: Place pins on the map
*   `fetch_availability`: Fetch the availability for a given place
*   `create_booking`: Create a booking on the user's behalf

```json
[
    {
        "type": "function",
        "function": {
            "name": "get_recommendations",
            "description": "Fetches recommendations based on the user's preferences",
            "parameters": {
                "type": "object",
                "properties": {
                    "type": {
                        "type": "string",
                        "description": "The type of place to search recommendations for",
                        "enum": ["restaurant", "hotel"]
                    },
                    "keywords": {
                        "type": "array",
                        "description": "Keywords that should be present in the recommendations",
                        "items": {
                            "type": "string"
                        }
                    },
                    "location": {
                        "type": "string",
                        "description": "The location to search recommendations for"
                    }
                },
                "required": [
                    "type",
                    "keywords",
                    "location"
                ],
                "additionalProperties": false
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "show_on_map",
            "description": "Places pins on the map for relevant locations",
            "parameters": {
                "type": "object",
                "properties": {
                    "pins": {
                        "type": "array",
                        "description": "The pins to place on the map",
                        "items": {
                            "type": "object",
                            "properties": {
                                "name": {
                                    "type": "string",
                                    "description": "The name of the place"
                                },
                                "coordinates": {
                                    "type": "object",
                                    "properties": {
                                        "latitude": { "type": "number" },
                                        "longitude": { "type": "number" }
                                    },
                                    "required": [
                                        "latitude",
                                        "longitude"
                                    ],
                                    "additionalProperties": false
                                }
                            },
                            "required": [
                                "name",
                                "coordinates"
                            ],
                            "additionalProperties": false
                        }
                    }
                },
                "required": [
                    "pins"
                ],
                "additionalProperties": false
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "fetch_availability",
            "description": "Fetches the availability for a given place",
            "parameters": {
                "type": "object",
                "properties": {
                    "place_id": {
                        "type": "string",
                        "description": "The ID of the place to fetch availability for"
                    }
                }
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "create_booking",
            "description": "Creates a booking on the user's behalf",
            "parameters": {
                "type": "object",
                "properties": {
                    "place_id": {
                        "type": "string",
                        "description": "The ID of the place to create a booking for"
                    },
                    "booking_details": {
                        "anyOf": [
                            {
                                "type": "object",
                                "description": "Restaurant booking with specific date and time",
                                "properties": {
                                    "date": {
                                        "type": "string",
                                        "description": "The date of the booking, in format YYYY-MM-DD"
                                    },
                                    "time": {
                                        "type": "string",
                                        "description": "The time of the booking, in format HH:MM"
                                    }
                                },
                                "required": [
                                    "date",
                                    "time"
                                ]
                            },
                            {
                                "type": "object",
                                "description": "Hotel booking with specific check-in and check-out dates",
                                "properties": {
                                    "check_in": {
                                        "type": "string",
                                        "description": "The check-in date of the booking, in format YYYY-MM-DD"
                                    },
                                    "check_out": {
                                        "type": "string",
                                        "description": "The check-out date of the booking, in format YYYY-MM-DD"
                                    }
                                },
                                "required": [
                                    "check_in",
                                    "check_out"
                                ]
                            }
                        ]
                    }
                },
                "required": [
                    "place_id",
                    "booking_details"
                ],
                "additionalProperties": false
            }
        }
    }
]
```

Best practices
--------------

### Turn on Structured Outputs by setting `strict: "true"`

When Structured Outputs is turned on, the arguments generated by the model for function calls will reliably match the JSON Schema that you provide.

If you are not using Structured Outputs, then the structure of arguments is not guaranteed to be correct, so we recommend the use of a validation library like Pydantic to first verify the arguments prior to using them.

### Name functions intuitively, with detailed descriptions

If you find the model does not generate calls to the correct functions, you may need to update your function names and descriptions so the model more clearly understands when it should select each function. Avoid using abbreviations or acronyms to shorten function and argument names.

You can also include detailed descriptions for when a function should be called. For complex functions, you should include descriptions for each of the arguments to help the model know what it needs to ask the user to collect that argument.

### Name function parameters intuitively, with detailed descriptions

Use clear and descriptive names for function parameters. If applicable, specify the expected format for a parameter in the description (e.g., YYYY-mm-dd or dd/mm/yy for a date).

### Consider providing additional information about how and when to call functions in your system message

Providing clear instructions in your system message can significantly improve the model's function calling accuracy. For example, guide the model with instructions like the following:

`"Use check_order_status when the user inquires about the status of their order, such as 'Where is my order?' or 'Has my order shipped yet?'".`

Provide context for complex scenarios. For example:

`"Before scheduling a meeting with schedule_meeting, check the user's calendar for availability using check_availability to avoid conflicts."`

### Use enums for function arguments when possible

If your use case allows, you can use enums to constrain the possible values for arguments. This can help reduce hallucinations.

For example, if you have an AI assistant that helps with ordering a T-shirt, you likely have a fixed set of sizes for the T-shirt that you might want to constrain the model to choose from. If you want the model to output “s”, “m”, “l”, for small, medium, and large, you could provide those values in an enum, like this:

```json
{
    "name": "pick_tshirt_size",
    "description": "Call this if the user specifies which size t-shirt they want",
    "parameters": {
        "type": "object",
        "properties": {
            "size": {
                "type": "string",
                "enum": ["s", "m", "l"],
                "description": "The size of the t-shirt that the user would like to order"
            }
        },
        "required": ["size"],
        "additionalProperties": false
    }
}
```

If you don't constrain the output, a user may say “large” or “L”, and the model may use these values as a parameter. As your code may expect a specific structure, it's helpful to limit the possible values the model can choose from.

### Keep the number of functions low for higher accuracy

We recommend that you use no more than 20 tools in a single API call.

Developers typically see a reduction in the model's ability to select the correct tool once they have between 10-20 tools defined.

If your use case requires the model to be able to pick between a large number of custom functions, you may want to explore fine-tuning ([learn more](https://cookbook.openai.com/examples/fine_tuning_for_function_calling)) or break out the tools and group them logically to create a multi-agent system.

### Set up evals to act as an aid in prompt engineering your function definitions and system messages

We recommend for non-trivial uses of function calling setting up an evaluation system that allow you to measure how frequently the correct function is called or correct arguments are generated for a wide variety of possible user messages. Learn more about setting up evals on the [OpenAI Cookbook](https://cookbook.openai.com/examples/evaluation/getting_started_with_openai_evals).

You can then use these to measure whether adjustments to your function definitions and system messages will improve or hurt your integration.

### Fine-tuning may help improve accuracy for function calling

Fine-tuning a model can improve performance at function calling for your use case, especially if you have a large number of functions, or complex, nuanced or similar functions.

See our fine-tuning for function calling [cookbook](https://cookbook.openai.com/examples/fine_tuning_for_function_calling) for more information.

[

Fine-tuning for function calling

Learn how to fine-tune a model for function calling

](https://cookbook.openai.com/examples/fine_tuning_for_function_calling)

===

Image generation
================

Learn how to generate or manipulate images with DALL·E.

Introduction
------------

The Images API provides three methods for interacting with images:

1.  Creating images from scratch based on a text prompt (DALL·E 3 and DALL·E 2)
2.  Creating edited versions of images by having the model replace some areas of a pre-existing image, based on a new text prompt (DALL·E 2 only)
3.  Creating variations of an existing image (DALL·E 2 only)

This guide covers the basics of using these three API endpoints with useful code samples. To try DALL·E 3, head to [ChatGPT](https://chatgpt.com/).

Usage
-----

### Generations

The [image generations](/docs/api-reference/images/create) endpoint allows you to create an original image given a text prompt. When using DALL·E 3, images can have a size of 1024x1024, 1024x1792 or 1792x1024 pixels.

By default, images are generated at `standard` quality, but when using DALL·E 3 you can set `quality: "hd"` for enhanced detail. Square, standard quality images are the fastest to generate.

You can request 1 image at a time with DALL·E 3 (request more by making parallel requests) or up to 10 images at a time using DALL·E 2 with the [n parameter](/docs/api-reference/images/create#images/create-n).

Generate an image

```javascript
import OpenAI from "openai";
const openai = new OpenAI();

const response = await openai.images.generate({
  model: "dall-e-3",
  prompt: "a white siamese cat",
  n: 1,
  size: "1024x1024",
});

console.log(response.data[0].url);
```

```python
from openai import OpenAI
client = OpenAI()

response = client.images.generate(
    model="dall-e-3",
    prompt="a white siamese cat",
    size="1024x1024",
    quality="standard",
    n=1,
)

print(response.data[0].url)
```

```bash
curl https://api.openai.com/v1/images/generations \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -d '{
    "model": "dall-e-3",
    "prompt": "a white siamese cat",
    "n": 1,
    "size": "1024x1024"
  }'
```

[

What is new with DALL·E 3

Explore what is new with DALL·E 3 in the OpenAI Cookbook

](https://cookbook.openai.com/articles/what_is_new_with_dalle_3)

Prompting
---------

With the release of DALL·E 3, the model now takes in the default prompt provided and automatically re-write it for safety reasons, and to add more detail (more detailed prompts generally result in higher quality images).

While it is not currently possible to disable this feature, you can use prompting to get outputs closer to your requested image by adding the following to your prompt: `I NEED to test how the tool works with extremely simple prompts. DO NOT add any detail, just use it AS-IS:`.

The updated prompt is visible in the `revised_prompt` field of the data response object.

Example DALL·E 3 generations
----------------------------

|Prompt|Generation|
|---|---|
|A photograph of a white Siamese cat.||

Each image can be returned as either a URL or Base64 data, using the [response\_format](/docs/api-reference/images/create#images/create-response_format) parameter. URLs will expire after an hour.

### Edits (DALL·E 2 only)

Also known as "inpainting", the [image edits](/docs/api-reference/images/create-edit) endpoint allows you to edit or extend an image by uploading an image and mask indicating which areas should be replaced. The transparent areas of the mask indicate where the image should be edited, and the prompt should describe the full new image, **not just the erased area**. This endpoint can enable experiences like DALL·E image editing in ChatGPT Plus.

Edit an image

```javascript
import OpenAI from "openai";
const openai = new OpenAI();

const response = await openai.images.generate({
  model: "dall-e-3",
  prompt: "a white siamese cat",
  n: 1,
  size: "1024x1024",
});

console.log(response.data[0].url);
```

```python
from openai import OpenAI
client = OpenAI()

response = client.images.edit(
    model="dall-e-2",
    image=open("sunlit_lounge.png", "rb"),
    mask=open("mask.png", "rb"),
    prompt="A sunlit indoor lounge area with a pool containing a flamingo",
    n=1,
    size="1024x1024",
)

print(response.data[0].url)
```

```bash
curl https://api.openai.com/v1/images/edits \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -F model="dall-e-2" \
  -F image="@sunlit_lounge.png" \
  -F mask="@mask.png" \
  -F prompt="A sunlit indoor lounge area with a pool containing a flamingo" \
  -F n=1 \
  -F size="1024x1024"
```

|Image|Mask|Output|
|---|---|---|
||||

Prompt: a sunlit indoor lounge area with a pool containing a flamingo

The uploaded image and mask must both be square PNG images less than 4MB in size, and also must have the same dimensions as each other. The non-transparent areas of the mask are not used when generating the output, so they don’t necessarily need to match the original image like the example above.

### Variations (DALL·E 2 only)

The [image variations](/docs/api-reference/images/create-variation) endpoint allows you to generate a variation of a given image.

Generate an image variation

```javascript
import OpenAI from "openai";
const openai = new OpenAI();

const response = await openai.images.createVariation({
  model: "dall-e-2",
  image: fs.createReadStream("corgi_and_cat_paw.png"),
  n: 1,
  size: "1024x1024"
});

console.log(response.data[0].url);
```

```python
from openai import OpenAI
client = OpenAI()

response = client.images.create_variation(
    model="dall-e-2",
    image=open("corgi_and_cat_paw.png", "rb"),
    n=1,
    size="1024x1024"
)

print(response.data[0].url)
```

```bash
curl https://api.openai.com/v1/images/variations \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -F model="dall-e-2" \
  -F image="@corgi_and_cat_paw.png" \
  -F n=1 \
  -F size="1024x1024"
```

|Image|Output|
|---|---|
|||

Similar to the edits endpoint, the input image must be a square PNG image less than 4MB in size.

### Content moderation

Prompts and images are filtered based on our [content policy](https://labs.openai.com/policies/content-policy), returning an error when a prompt or image is flagged.

Language-specific tips
----------------------

Node.jsPython

### Using in-memory image data

The Node.js examples in the guide above use the `fs` module to read image data from disk. In some cases, you may have your image data in memory instead. Here's an example API call that uses image data stored in a Node.js `Buffer` object:

```javascript
import OpenAI from "openai";

const openai = new OpenAI();

// This is the Buffer object that contains your image data
const buffer = [your image data];

// Set a `name` that ends with .png so that the API knows it's a PNG image
buffer.name = "image.png";

async function main() {
  const image = await openai.images.createVariation({ model: "dall-e-2", image: buffer, n: 1, size: "1024x1024" });
  console.log(image.data);
}
main();
```

### Working with TypeScript

If you're using TypeScript, you may encounter some quirks with image file arguments. Here's an example of working around the type mismatch by explicitly casting the argument:

```javascript
import fs from "fs";
import OpenAI from "openai";

const openai = new OpenAI();

async function main() {
  // Cast the ReadStream to `any` to appease the TypeScript compiler
  const image = await openai.images.createVariation({
    image: fs.createReadStream("image.png") as any,
  });

  console.log(image.data);
}
main();
```

And here's a similar example for in-memory image data:

```javascript
import fs from "fs";
import OpenAI from "openai";

const openai = new OpenAI();

// This is the Buffer object that contains your image data
const buffer: Buffer = [your image data];

// Cast the buffer to `any` so that we can set the `name` property
const file: any = buffer;

// Set a `name` that ends with .png so that the API knows it's a PNG image
file.name = "image.png";

async function main() {
  const image = await openai.images.createVariation({
    file,
    1,
    "1024x1024"
  });
  console.log(image.data);
}
main();
```

### Error handling

API requests can potentially return errors due to invalid inputs, rate limits, or other issues. These errors can be handled with a `try...catch` statement, and the error details can be found in either `error.response` or `error.message`:

```javascript
import fs from "fs";
import OpenAI from "openai";

const openai = new OpenAI();

async function main() {
    try {
        const image = await openai.images.createVariation({
            image: fs.createReadStream("image.png"),
            n: 1,
            size: "1024x1024",
        });
        console.log(image.data);
    } catch (error) {
        if (error.response) {
            console.log(error.response.status);
            console.log(error.response.data);
        } else {
            console.log(error.message);
        }
    }
}

main();
```

===

Model distillation
==================

Improve smaller models with distillation techniques.

Model Distillation allows you to leverage the outputs of a large model to [fine-tune](/docs/guides/fine-tuning) a smaller model, enabling it to achieve similar performance on a specific task. This process can significantly reduce both cost and latency, as smaller models are typically more efficient.

Here's how it works:

1.  Store high-quality outputs of a large model using the [`store`](/docs/api-reference/chat/create#chat-create-store) parameter in the chat completions API to store them.
2.  [Evaluate](/docs/guides/evals) the stored completions with both the large and the small model to establish a baseline.
3.  Select the stored completions that you'd like to use to for distillation and use them to [fine-tune](/docs/guides/fine-tuning) the smaller model.
4.  [Evaluate](/docs/guides/evals) the performance of the fine-tuned model to see how it compares to the large model.

Let's go through these steps to see how it's done.

Store high-quality outputs of a large model

-----------------------------------------------

The first step in the distillation process is to generate good results with a large model like `o1-preview` or `gpt-4o` that meet your bar. As you generate these results, you can store them using the `store: true` option in the [Chat Completions API](/docs/api-reference/chat/create#chat-create-store). We also recommend you use the [metadata](/docs/api-reference/chat/create#chat-create-metadata) property to tag these completions for easy filtering later.

These stored completion can then be viewed and filtered in the [dashboard](/chat-completions).

Store high-quality outputs of a large model

```javascript
import OpenAI from "openai";
const openai = new OpenAI();

const response = await openai.chat.completions.create({
  model: "gpt-4o",
  messages: [
    { role: "system", content: "You are a corporate IT support expert." },
    { role: "user", content: "How can I hide the dock on my Mac?"},
  ],
  store: true,
  metadata: {
    role: "manager",
    department: "accounting",
    source: "homepage"
  }
});

console.log(response.choices[0]);
```

When using the `store: true` option, completions are stored for 30 days. Your completions may contain sensitive information and so, you may want to consider creating a new [Project](https://help.openai.com/en/articles/9186755-managing-your-work-in-the-api-platform-with-projects) with limited access to store these completions.

Evaluate to establish a baseline
--------------------------------

You can use your stored completions to evaluate the performance of both the larger model and a smaller model on your task to establish a baseline. This can be done using the [evals](/docs/guides/evals) product.

Typically, the large model will outperform the smaller model on your evaluations. Establishing this baseline allows you to measure the improvements gained through the distillation / fine-tuning process.

Create training dataset to fine-tune smaller model
--------------------------------------------------

Next you can select a subset of your stored completions to use as training data for fine-tuning a smaller model like `gpt-4o-mini`. [Filter your stored completions](/chat-completions) to those that you would like to use to train the small model, and click the "Distill" button. A few hundred samples might be sufficient, but sometimes a more diverse range of thousands of samples can yield better results.

![distill results](https://openaidevs.retool.com/api/file/7c0009a4-e9f9-4b66-af50-c4e58e0d267d)

This action will open a dialog to begin a [fine-tuning job](/docs/guides/fine-tuning), with your selected completions as the training dataset. Configure the parameters as needed, choosing the base model you wish to fine-tune. In this example, we're going to choose the [latest snapshot of GPT-4o-mini](/docs/models#gpt-4o-mini).

![fine tune job](https://openaidevs.retool.com/api/file/ab8d0ccf-df5d-4099-80e1-2f257d82a92f)

After configuring, click "Run" to start the fine-tuning job. The process may take 15 minutes or longer, depending on the size of your training dataset.

Evaluate the fine-tuned small model
-----------------------------------

When your fine-tuning job is complete, you can run evals against it to see how it stacks up against the base small and large models. You can select fine-tuned models in the [Evals](/evaluations) product to generate new completions with the fine-tuned small model.

![eval using ft model](https://openaidevs.retool.com/api/file/8fcfdb03-1385-47d8-81d6-735af29594cc)

Alternately, you could also store [new chat completions](\(/docs/guides/distillation#send-fine-tuned\)) generated by the fine-tuned model, and use them to evaluate performance. By continually tweaking and improving:

*   The diversity of the training data
*   Your prompts and outputs on the large model
*   The accuracy of your eval graders

You can bring the performance of the smaller model up to the same levels as the large model, for a specific subset of tasks.

Next steps
----------

Distilling large model results to a small model is one powerful way to improve the results you generate from your models, but not the only one. Check out these resources to learn more about optimizing your outputs.

[

Fine-tuning

Improve a model's ability to generate responses tailored to your use case.

](/docs/guides/fine-tuning)[

Evals

Run tests on your model outputs to ensure you're getting the right results.

](/docs/guides/evals)


===

Models
======

Flagship models
---------------

[

GPT-4o

Our versatile, high-intelligence flagship model

Text and image input, text output

128k context length

Smarter model, higher price per token

](/docs/models#gpt-4o)

[

GPT-4o mini

Our fast, affordable small model for focused tasks

Text and image input, text output

128k context length

Faster model, lower price per token

](/docs/models#gpt-4o-mini)

[

o1 & o1-mini

Reasoning models that excel at complex, multi-step tasks

Text and image input, text output

128k context length

Uses additional tokens for reasoning

](/docs/models#o1)

[Model pricing details](https://openai.com/api/pricing)

Models overview
---------------

The OpenAI API is powered by a diverse set of models with different capabilities and price points. You can also make customizations to our models for your specific use case with [fine-tuning](/docs/guides/fine-tuning).

||
|GPT-4o|Our versatile, high-intelligence flagship model|
|GPT-4o-mini|Our fast, affordable small model for focused tasks|
|o1 and o1-mini|Reasoning models that excel at complex, multi-step tasks|
|GPT-4o Realtime|GPT-4o models capable of realtime text and audio inputs and outputs|
|GPT-4o Audio|GPT-4o models capable of audio inputs and outputs via REST API|
|GPT-4 Turbo and GPT-4|The previous set of high-intelligence models|
|GPT-3.5 Turbo|A fast model for simple tasks, superceded by GPT-4o-mini|
|DALL·E|A model that can generate and edit images given a natural language prompt|
|TTS|A set of models that can convert text into natural sounding spoken audio|
|Whisper|A model that can convert audio into text|
|Embeddings|A set of models that can convert text into a numerical form|
|Moderation|A fine-tuned model that can detect whether text may be sensitive or unsafe|
|Deprecated|A full list of models that have been deprecated along with the suggested replacement|

We have also published open source models including [Point-E](https://github.com/openai/point-e), [Whisper](https://github.com/openai/whisper), [Jukebox](https://github.com/openai/jukebox), and [CLIP](https://github.com/openai/CLIP).

Context window
--------------

Models on this page will list a **context window**, which refers to the maximum number of tokens that can be used in a single request, inclusive of both input, output, and reasoning tokens. For example, when making an API request to [chat completions](/docs/api-reference/chat) with the [o1 model](/docs/guides/reasoning), the following token counts will apply toward the context window total:

*   Input tokens (inputs you include in the `messages` array with [chat completions](/docs/api-reference/chat))
*   Output tokens (tokens generated in response to your prompt)
*   Reasoning tokens (used by the model to plan a response)

Tokens generated in excess of the context window limit may be truncated in API responses.

![context window visualization](https://cdn.openai.com/API/docs/images/context-window.png)

You can estimate the number of tokens your messages will use with the [tokenizer tool](/tokenizer).

Model ID aliases and snapshots
------------------------------

In the tables below, you will see **model IDs** that can be used in REST APIs like [chat completions](/docs/api-reference/chat) to generate outputs. Some of these model IDs are **aliases** which point to specific **dated snapshots**.

For example, the `gpt-4o` model ID is an alias that points to a specific dated snapshot of GPT-4o. The dated snapshots that these aliases point to are periodically updated to newer snapshots a few months after a newer snapshot becomes available. Model IDs that are aliases note the model ID they currently point to in the tables below.

API request using a model alias

```javascript
import OpenAI from "openai";
const openai = new OpenAI();

const completion = await openai.chat.completions.create({
    model: "gpt-4o",
    messages: [
        { role: "developer", content: "You are a helpful assistant." },
        {
            role: "user",
            content: "Write a haiku about recursion in programming.",
        },
    ],
});

console.log(completion.choices[0].message);
```

```python
from openai import OpenAI
client = OpenAI()

completion = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "developer", "content": "You are a helpful assistant."},
        {
            "role": "user",
            "content": "Write a haiku about recursion in programming."
        }
    ]
)

print(completion.choices[0].message)
```

```bash
curl "https://api.openai.com/v1/chat/completions" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $OPENAI_API_KEY" \
    -d '{
        "model": "gpt-4o",
        "messages": [
            {
                "role": "developer",
                "content": "You are a helpful assistant."
            },
            {
                "role": "user",
                "content": "Write a haiku about recursion in programming."
            }
        ]
    }'
```

In API requests where an alias was used as a model ID, the body of the response will contain the actual model ID used to generate the response.

```json
{
  "id": "chatcmpl-Af6LFgbOPpqu2fhGsVktc9xFaYUVh",
  "object": "chat.completion",
  "created": 1734359189,
  "model": "gpt-4o-2024-08-06",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Code within a loop,  \nFunction calls itself again,  \nInfinite echoes.",
        "refusal": null
      },
      "logprobs": null,
      "finish_reason": "stop"
    }
  ],
  "usage": {}
}
```

Current model aliases
---------------------

Below, please find current model aliases, and guidance on when they will be updated to new versions (if guidance is available).

||
|gpt-4o|gpt-4o-2024-08-06|-|
|chatgpt-4o-latest|Latest used in ChatGPT|Continuously updated|
|gpt-4o-mini|gpt-4o-mini-2024-07-18|-|
|o1|o1-2024-12-17|-|
|o1-mini|o1-mini-2024-09-12|-|
|o1-preview|o1-preview-2024-09-12|-|
|gpt-4o-realtime-preview|gpt-4o-realtime-preview-2024-10-01|gpt-4o-realtime-preview-2024-12-17  Effective Jan. 9 2025|
|gpt-4o-mini-realtime-preview|gpt-4o-mini-realtime-preview-2024-12-17|-|
|gpt-4o-audio-preview|gpt-4o-audio-preview-2024-10-01|gpt-4o-audio-preview-2024-12-17  Effective Jan. 9 2025|

In production applications, **it is a best practice to use dated model snapshot IDs** instead of aliases, which may change periodically.

GPT-4o
------

GPT-4o (“o” for “omni”) is our versatile, high-intelligence flagship model. It accepts both text and [image inputs](/docs/guides/vision), and produces [text outputs](/docs/guides/text-generation) (including [Structured Outputs](/docs/guides/structured-outputs)). Learn how to use GPT-4o in our [text generation guide](/docs/guides/text-generation).

The `chatgpt-4o-latest` model ID below continuously points to the version of GPT-4o used in [ChatGPT](https://chatgpt.com). It is updated frequently, when there are significant changes to ChatGPT's GPT-4o model.

The knowledge cutoff for GPT-4o models is **October, 2023**.

||
|gpt-4o↳ gpt-4o-2024-08-06|128,000 tokens|16,384 tokens|
|gpt-4o-2024-11-20|128,000 tokens|16,384 tokens|
|gpt-4o-2024-08-06|128,000 tokens|16,384 tokens|
|gpt-4o-2024-05-13|128,000 tokens|4,096 tokens|
|chatgpt-4o-latest↳ GPT-4o used in ChatGPT|128,000 tokens|16,384 tokens|

GPT-4o mini
-----------

GPT-4o mini (“o” for “omni”) is a fast, affordable small model for focused tasks. It accepts both text and [image inputs](/docs/guides/vision), and produces [text outputs](/docs/guides/text-generation) (including [Structured Outputs](/docs/guides/structured-outputs)). It is ideal for [fine-tuning](/docs/guides/fine-tuning), and model outputs from a larger model like GPT-4o can be [distilled](/docs/guides/distillation) to GPT-4o-mini to produce similar results at lower cost and latency.

The knowledge cutoff for GPT-4o-mini models is **October, 2023**.

||
|gpt-4o-mini↳ gpt-4o-mini-2024-07-18|128,000 tokens|16,384 tokens|
|gpt-4o-mini-2024-07-18|128,000 tokens|16,384 tokens|

o1 and o1-mini

------------------

The **o1 series** of models are trained with reinforcement learning to perform complex reasoning. o1 models think before they answer, producing a long internal chain of thought before responding to the user. Learn about the capabilities of o1 models in our [reasoning guide](/docs/guides/reasoning).

There are two model types available today:

*   **o1**: reasoning model designed to solve hard problems across domains
*   **o1-mini**: fast and affordable reasoning model for specialized tasks

The latest o1 model supports both text and [image inputs](/docs/guides/vision), and produces [text outputs](/docs/guides/text-generation) (including [Structured Outputs](/docs/guides/structured-outputs)). o1-mini currently only supports text inputs and outputs.

The knowledge cutoff for o1 and o1-mini models is **October, 2023**.

||
|o1↳ o1-2024-12-17|200,000 tokens|100,000 tokens|
|o1-2024-12-17|200,000 tokens|100,000 tokens|
|o1-mini↳ o1-mini-2024-09-12|128,000 tokens|65,536 tokens|
|o1-mini-2024-09-12|128,000 tokens|65,536 tokens|
|o1-preview↳ o1-preview-2024-09-12|128,000 tokens|32,768 tokens|
|o1-preview-2024-09-12|128,000 tokens|32,768 tokens|

GPT-4o and GPT-4o-mini Realtime

Beta

-----------------------------------------

This is a preview release of the GPT-4o and GPT-4o-mini Realtime models. These models are capable of responding to audio and text inputs in realtime over WebRTC or a WebSocket interface. Learn more in the [Realtime API guide](/docs/guides/realtime).

The knowledge cutoff for GPT-4o Realtime models is **October, 2023**.

||
|gpt-4o-realtime-preview↳ gpt-4o-realtime-preview-2024-10-01|128,000 tokens|4,096 tokens|
|gpt-4o-realtime-preview-2024-12-17|128,000 tokens|4,096 tokens|
|gpt-4o-realtime-preview-2024-10-01|128,000 tokens|4,096 tokens|
|gpt-4o-mini-realtime-preview↳ gpt-4o-mini-realtime-preview-2024-12-17|128,000 tokens|4,096 tokens|
|gpt-4o-mini-realtime-preview-2024-12-17|128,000 tokens|4,096 tokens|

GPT-4o Audio

Beta

----------------------

This is a preview release of the GPT-4o Audio models. These models accept audio inputs and outputs, and can be used in the Chat Completions REST API. [Learn more](/docs/guides/audio).

The knowledge cutoff for GPT-4o Audio models is **October, 2023**.

||
|gpt-4o-audio-preview↳ gpt-4o-audio-preview-2024-10-01|128,000 tokens|16,384 tokens|
|gpt-4o-audio-preview-2024-12-17|128,000 tokens|16,384 tokens|
|gpt-4o-audio-preview-2024-10-01|128,000 tokens|16,384 tokens|

GPT-4 Turbo and GPT-4
---------------------

GPT-4 is an older version of a high-intelligence GPT model, usable in [Chat Completions](/docs/api-reference/chat). Learn more in the [text generation guide](/docs/guides/text-generation). The knowledge cutoff for the latest GPT-4 Turbo version is **December, 2023**.

||
|gpt-4-turbo↳ gpt-4-turbo-2024-04-09|128,000 tokens|4,096 tokens|
|gpt-4-turbo-2024-04-09|128,000 tokens|4,096 tokens|
|gpt-4-turbo-preview↳ gpt-4-0125-preview|128,000 tokens|4,096 tokens|
|gpt-4-0125-preview|128,000 tokens|4,096 tokens|
|gpt-4-1106-preview|128,000 tokens|4,096 tokens|
|gpt-4↳ gpt-4-0613|8,192 tokens|8,192 tokens|
|gpt-4-0613|8,192 tokens|8,192 tokens|
|gpt-4-0314|8,192 tokens|8,192 tokens|

GPT-3.5 Turbo
-------------

GPT-3.5 Turbo models can understand and generate natural language or code and have been optimized for chat using the [Chat Completions API](/docs/api-reference/chat) but work well for non-chat tasks as well.

As of July 2024, `gpt-4o-mini` should be used in place of `gpt-3.5-turbo`, as it is cheaper, more capable, multimodal, and just as fast. `gpt-3.5-turbo` is still available for use in the API.

|Model|Context window|Max output tokens|Knowledge cutoff|
|---|---|---|---|
|gpt-3.5-turbo-0125The latest GPT-3.5 Turbo model with higher accuracy at responding in requested formats and a fix for a bug which caused a text encoding issue for non-English language function calls. Learn more.|16,385 tokens|4,096 tokens|Sep 2021|
|gpt-3.5-turboCurrently points to gpt-3.5-turbo-0125.|16,385 tokens|4,096 tokens|Sep 2021|
|gpt-3.5-turbo-1106GPT-3.5 Turbo model with improved instruction following, JSON mode, reproducible outputs, parallel function calling, and more. Learn more.|16,385 tokens|4,096 tokens|Sep 2021|
|gpt-3.5-turbo-instructSimilar capabilities as GPT-3 era models. Compatible with legacy Completions endpoint and not Chat Completions.|4,096 tokens|4,096 tokens|Sep 2021|

DALL·E
------

DALL·E is a AI system that can create realistic images and art from a description in natural language. DALL·E 3 currently supports the ability, given a prompt, to create a new image with a specific size. DALL·E 2 also support the ability to edit an existing image, or create variations of a user provided image.

[DALL·E 3](https://openai.com/dall-e-3) is available through our [Images API](/docs/guides/images) along with [DALL·E 2](https://openai.com/blog/dall-e-api-now-available-in-public-beta). You can try DALL·E 3 through [ChatGPT Plus](https://chatgpt.com).

|Model|Description|
|---|---|
|dall-e-3|The latest DALL·E model released in Nov 2023. Learn more.|
|dall-e-2|The previous DALL·E model released in Nov 2022. The 2nd iteration of DALL·E with more realistic, accurate, and 4x greater resolution images than the original model.|

TTS
---

TTS is an AI model that converts text to natural sounding spoken text. We offer two different model variates, `tts-1` is optimized for real time text to speech use cases and `tts-1-hd` is optimized for quality. These models can be used with the [Speech endpoint in the Audio API](/docs/guides/text-to-speech).

|Model|Description|
|---|---|
|tts-1|The latest text to speech model, optimized for speed.|
|tts-1-hd|The latest text to speech model, optimized for quality.|

Whisper
-------

Whisper is a general-purpose speech recognition model. It is trained on a large dataset of diverse audio and is also a multi-task model that can perform multilingual speech recognition as well as speech translation and language identification. The Whisper v2-large model is currently available through our API with the `whisper-1` model name.

Currently, there is no difference between the [open source version of Whisper](https://github.com/openai/whisper) and the version available through our API. However, [through our API](/docs/guides/speech-to-text), we offer an optimized inference process which makes running Whisper through our API much faster than doing it through other means. For more technical details on Whisper, you can [read the paper](https://arxiv.org/abs/2212.04356).

Embeddings
----------

Embeddings are a numerical representation of text that can be used to measure the relatedness between two pieces of text. Embeddings are useful for search, clustering, recommendations, anomaly detection, and classification tasks. You can read more about our latest embedding models in the [announcement blog post](https://openai.com/blog/new-embedding-models-and-api-updates).

|Model|Output Dimension|
|---|---|
|text-embedding-3-largeMost capable embedding model for both english and non-english tasks|3,072|
|text-embedding-3-smallIncreased performance over 2nd generation ada embedding model|1,536|
|text-embedding-ada-002Most capable 2nd generation embedding model, replacing 16 first generation models|1,536|

* * *

Moderation
----------

The Moderation models are designed to check whether content complies with OpenAI's [usage policies](https://openai.com/policies/usage-policies). The models provide classification capabilities that look for content in categories like hate, self-harm, sexual content, violence, and others. Learn more about moderating text and images in our [moderation guide](/docs/guides/moderation).

|Model|Max tokens|
|---|---|
|omni-moderation-latestCurrently points to omni-moderation-2024-09-26.|32,768|
|omni-moderation-2024-09-26Latest pinned version of our new multi-modal moderation model, capable of analyzing both text and images.|32,768|
|text-moderation-latestCurrently points to text-moderation-007.|32,768|
|text-moderation-stableCurrently points to text-moderation-007.|32,768|
|text-moderation-007Previous generation text-only moderation. We expect omni-moderation-* models to be the best default moving forward.|32,768|

GPT base
--------

GPT base models can understand and generate natural language or code but are not trained with instruction following. These models are made to be replacements for our original GPT-3 base models and use the legacy Completions API. Most customers should use GPT-3.5 or GPT-4.

|Model|Max tokens|Knowledge cutoff|
|---|---|---|
|babbage-002Replacement for the GPT-3 ada and babbage base models.|16,384 tokens|Sep 2021|
|davinci-002Replacement for the GPT-3 curie and davinci base models.|16,384 tokens|Sep 2021|

How we use your data
--------------------

Your data is your data.

As of March 1, 2023, data sent to the OpenAI API will not be used to train or improve OpenAI models (unless you explicitly opt-in to share data with us, such as by [providing feedback in the Playground](https://help.openai.com/en/articles/9883556-providing-feedback-in-the-api-playground)). One advantage to opting in is that the models may get better at your use case over time.

To help identify abuse, API data may be retained for up to 30 days, after which it will be deleted (unless otherwise required by law). For trusted customers with sensitive applications, zero data retention may be available. With zero data retention, request and response bodies are not persisted to any logging mechanism and exist only in memory in order to serve the request.

Note that this data policy does not apply to OpenAI's non-API consumer services like [ChatGPT](https://chatgpt.com/) or [DALL·E Labs](https://labs.openai.com/).

### Default usage policies by endpoint

|Endpoint|Data used for training|Default retention|Eligible for zero retention|
|---|---|---|---|
|/v1/chat/completions*|No|30 days|Yes, except (a) image inputs, (b) schemas provided for Structured Outputs, or (c) audio outputs. *|
|/v1/assistants|No|30 days **|No|
|/v1/threads|No|30 days **|No|
|/v1/threads/messages|No|30 days **|No|
|/v1/threads/runs|No|30 days **|No|
|/v1/vector_stores|No|30 days **|No|
|/v1/threads/runs/steps|No|30 days **|No|
|/v1/images/generations|No|30 days|No|
|/v1/images/edits|No|30 days|No|
|/v1/images/variations|No|30 days|No|
|/v1/embeddings|No|30 days|Yes|
|/v1/audio/transcriptions|No|Zero data retention|-|
|/v1/audio/translations|No|Zero data retention|-|
|/v1/audio/speech|No|30 days|Yes|
|/v1/files|No|Until deleted by customer|No|
|/v1/fine_tuning/jobs|No|Until deleted by customer|No|
|/v1/batches|No|Until deleted by customer|No|
|/v1/moderations|No|Zero data retention|-|
|/v1/completions|No|30 days|Yes|
|/v1/realtime (beta)|No|30 days|Yes|

**\* Chat Completions:**

*   Image inputs via the `o1`, `gpt-4o`, `gpt-4o-mini`, `chatgpt-4o-latest`, or `gpt-4-turbo` models (or previously `gpt-4-vision-preview`) are not eligible for zero retention.
*   Audio outputs are stored for 1 hour to enable [multi-turn conversations](/docs/guides/audio), and are not currently eligible for zero retention.
*   When Structured Outputs is enabled, schemas provided (either as the `response_format` or in the function definition) are not eligible for zero retention, though the completions themselves are.
*   When using Stored Completions via the `store: true` option in the API, those completions are stored for 30 days. Completions are stored in an unfiltered form after an API response, so please avoid storing completions that contain sensitive data.

**\*\* Assistants API:**

*   Objects related to the Assistants API are deleted from our servers 30 days after you delete them via the API or the dashboard. Objects that are not deleted via the API or dashboard are retained indefinitely.

**Evaluations:**

*   [Evaluation](/evaluations) data: When you create an evaluation, the data related to that evaluation is deleted from our servers 30 days after you delete it via the dashboard. Evaluation data that is not deleted via the dashboard is retained indefinitely.

For details, see our [API data usage policies](https://openai.com/policies/api-data-usage-policies). To learn more about zero retention, get in touch with our [sales team](https://openai.com/contact-sales).

Model endpoint compatibility
----------------------------

|Endpoint|Latest models|
|---|---|
|/v1/assistants|All GPT-4o (except chatgpt-4o-latest), GPT-4o-mini, GPT-4, and GPT-3.5 Turbo models. The retrieval tool requires gpt-4-turbo-preview (and subsequent dated model releases) or gpt-3.5-turbo-1106 (and subsequent versions).|
|/v1/audio/transcriptions|whisper-1|
|/v1/audio/translations|whisper-1|
|/v1/audio/speech|tts-1,  tts-1-hd|
|/v1/chat/completions|All GPT-4o (except for Realtime preview), GPT-4o-mini, GPT-4, and GPT-3.5 Turbo models and their dated releases. chatgpt-4o-latest dynamic model. Fine-tuned versions of gpt-4o,  gpt-4o-mini,  gpt-4,  and gpt-3.5-turbo.|
|/v1/completions (Legacy)|gpt-3.5-turbo-instruct,  babbage-002,  davinci-002|
|/v1/embeddings|text-embedding-3-small,  text-embedding-3-large,  text-embedding-ada-002|
|/v1/fine_tuning/jobs|gpt-4o,  gpt-4o-mini,  gpt-4,  gpt-3.5-turbo|
|/v1/moderations|text-moderation-stable,  text-moderation-latest|
|/v1/images/generations|dall-e-2,  dall-e-3|
|/v1/realtime (beta)|gpt-4o-realtime-preview, gpt-4o-realtime-preview-2024-10-01|

This list excludes all of our [deprecated models](/docs/deprecations).

===

Moderation
==========

Identify potentially harmful content in text and images.

The [moderations](/docs/api-reference/moderations) endpoint is a tool you can use to check whether text or images are potentially harmful. Once harmful content is identified, developers can take corrective action like filtering content or intervening with user accounts creating offending content. The moderation endpoint is free to use.

The models available for this endpoint are:

*   `omni-moderation-latest`: This model and all snapshots support more categorization options and multi-modal inputs.
*   `text-moderation-latest` **(Legacy)**: Older model that supports only text inputs and fewer input categorizations. The newer omni-moderation models will be the best choice for new applications.

Quickstart
----------

The [moderation endpoint](/docs/api-reference/moderations) can be used to classify both text and images. Below, you can find a few examples using our [official SDKs](/docs/libraries). These examples use the `omni-moderation-latest` [model](/docs/models#moderation):

Moderate text inputs

Get classification information for a text input

```python
from openai import OpenAI
client = OpenAI()

response = client.moderations.create(
  model="omni-moderation-latest",
  input="...text to classify goes here...",
)

print(response)
```

```javascript
import OpenAI from "openai";
const openai = new OpenAI();

const moderation = await openai.moderations.create({
  model: "omni-moderation-latest",
  input: "...text to classify goes here...",
});

console.log(moderation);
```

```bash
curl https://api.openai.com/v1/moderations \
-X POST \
-H "Content-Type: application/json" \
-H "Authorization: Bearer $OPENAI_API_KEY" \
-d '{
  "model": "omni-moderation-latest",
  "input": "...text to classify goes here..."
}'
```

Moderate images and text

Get classification information for image and text input

```python
from openai import OpenAI
client = OpenAI()

response = client.moderations.create(
  model="omni-moderation-latest",
  input=[
      {"type": "text", "text": "...text to classify goes here..."},
      {
          "type": "image_url",
          "image_url": {
              "url": "https://example.com/image.png",
              # can also use base64 encoded image URLs
              # "url": "data:image/jpeg;base64,abcdefg..."
          }
      },
  ],
)

print(response)
```

```javascript
import OpenAI from "openai";
const openai = new OpenAI();

const moderation = await openai.moderations.create({
  model: "omni-moderation-latest",
  input: [
      { type: "text", text: "...text to classify goes here..." },
      {
          type: "image_url",
          image_url: {
              url: "https://example.com/image.png"
              // can also use base64 encoded image URLs
              // url: "data:image/jpeg;base64,abcdefg..."
          }
      }
  ],
});

console.log(moderation);
```

```bash
curl https://api.openai.com/v1/moderations \
-X POST \
-H "Content-Type: application/json" \
-H "Authorization: Bearer $OPENAI_API_KEY" \
-d '{
  "model": "omni-moderation-latest",
  "input": [
    { "type": "text", "text": "...text to classify goes here..." },
    {
      "type": "image_url",
      "image_url": {
        "url": "https://example.com/image.png"
      }
    }
  ]
}'
```

Here is the full example output for an image input from a single frame of a war movie. The model correctly predicts indicators of violence in the image, with a `violence` category score of greater than 0.8.

```json
{
  "id": "modr-970d409ef3bef3b70c73d8232df86e7d",
  "model": "omni-moderation-latest",
  "results": [
    {
      "flagged": true,
      "categories": {
        "sexual": false,
        "sexual/minors": false,
        "harassment": false,
        "harassment/threatening": false,
        "hate": false,
        "hate/threatening": false,
        "illicit": false,
        "illicit/violent": false,
        "self-harm": false,
        "self-harm/intent": false,
        "self-harm/instructions": false,
        "violence": true,
        "violence/graphic": false
      },
      "category_scores": {
        "sexual": 2.34135824776394e-7,
        "sexual/minors": 1.6346470245419304e-7,
        "harassment": 0.0011643905680426018,
        "harassment/threatening": 0.0022121340080906377,
        "hate": 3.1999824407395835e-7,
        "hate/threatening": 2.4923252458203563e-7,
        "illicit": 0.0005227032493135171,
        "illicit/violent": 3.682979260160596e-7,
        "self-harm": 0.0011175734280627694,
        "self-harm/intent": 0.0006264858507989037,
        "self-harm/instructions": 7.368592981140821e-8,
        "violence": 0.8599265510337075,
        "violence/graphic": 0.37701736389561064
      },
      "category_applied_input_types": {
        "sexual": [
          "image"
        ],
        "sexual/minors": [],
        "harassment": [],
        "harassment/threatening": [],
        "hate": [],
        "hate/threatening": [],
        "illicit": [],
        "illicit/violent": [],
        "self-harm": [
          "image"
        ],
        "self-harm/intent": [
          "image"
        ],
        "self-harm/instructions": [
          "image"
        ],
        "violence": [
          "image"
        ],
        "violence/graphic": [
          "image"
        ]
      }
    }
  ]
}
```

The output from the models is described below. The JSON response contains information about what (if any) categories of content are present in the inputs, and to what degree the model believes them to be present.

||
|flagged|Set to true if the model classifies the content as potentially harmful, false otherwise.|
|categories|Contains a dictionary of per-category violation flags. For each category, the value is true if the model flags the corresponding category as violated, false otherwise.|
|category_scores|Contains a dictionary of per-category scores output by the model, denoting the model's confidence that the input violates the OpenAI's policy for the category. The value is between 0 and 1, where higher values denote higher confidence.|
|category_applied_input_types|This property contains information on which input types were flagged in the response, for each category. For example, if the both the image and text inputs to the model are flagged for "violence/graphic", the violence/graphic property will be set to ["image", "text"]. This is only available on omni models.|

We plan to continuously upgrade the moderation endpoint's underlying model. Therefore, custom policies that rely on `category_scores` may need recalibration over time.

Content classifications
-----------------------

The table below describes the types of content that can be detected in the moderation API, along with what models and input types are supported for each category.

||
|harassment|Content that expresses, incites, or promotes harassing language towards any target.|All|Text only|
|harassment/threatening|Harassment content that also includes violence or serious harm towards any target.|All|Text only|
|hate|Content that expresses, incites, or promotes hate based on race, gender, ethnicity, religion, nationality, sexual orientation, disability status, or caste. Hateful content aimed at non-protected groups (e.g. chess players) is harassment.|All|Text only|
|hate/threatening|Hateful content that also includes violence or serious harm towards the targeted group based on race, gender, ethnicity, religion, nationality, sexual orientation, disability status, or caste.|All|Text only|
|illicit|Content that gives advice or instruction on how to commit illicit acts. A phrase like "how to shoplift" would fit this category.|Omni only|Text only|
|illicit/violent|The same types of content flagged by the illicit category, but also includes references to violence or procuring a weapon.|Omni only|Text only|
|self-harm|Content that promotes, encourages, or depicts acts of self-harm, such as suicide, cutting, and eating disorders.|All|Text and image|
|self-harm/intent|Content where the speaker expresses that they are engaging or intend to engage in acts of self-harm, such as suicide, cutting, and eating disorders.|All|Text and image|
|self-harm/instructions|Content that encourages performing acts of self-harm, such as suicide, cutting, and eating disorders, or that gives instructions or advice on how to commit such acts.|All|Text and image|
|sexual|Content meant to arouse sexual excitement, such as the description of sexual activity, or that promotes sexual services (excluding sex education and wellness).|All|Text and image|
|sexual/minors|Sexual content that includes an individual who is under 18 years old.|All|Text only|
|violence|Content that depicts death, violence, or physical injury.|All|Text and images|
|violence/graphic|Content that depicts death, violence, or physical injury in graphic detail.|All|Text and images|

===

Predicted Outputs
=================

Reduce latency for model responses where much of the response is known ahead of time.

**Predicted Outputs** enable you to speed up API responses from [Chat Completions](/docs/api-reference/chat/create) when many of the output tokens are known ahead of time. This is most common when you are regenerating a text or code file with minor modifications. You can provide your prediction using the [`prediction` request parameter in Chat Completions](/docs/api-reference/chat/create#chat-create-prediction).

Predicted Outputs are available today using the latest `gpt-4o` and `gpt-4o-mini` models. Read on to learn how to use Predicted Outputs to reduce latency in your applicatons.

Code refactoring example
------------------------

Predicted Outputs are particularly useful for regenerating text documents and code files with small modifications. Let's say you want the [GPT-4o model](/docs/models#gpt-4o) to refactor a piece of TypeScript code, and convert the `username` property of the `User` class to be `email` instead:

```typescript
class User {
  firstName: string = "";
  lastName: string = "";
  username: string = "";
}

export default User;
```

Most of the file will be unchanged, except for line 4 above. If you use the current text of the code file as your prediction, you can regenerate the entire file with lower latency. These time savings add up quickly for larger files.

Below is an example of using the `prediction` parameter in our SDKs to predict that the final output of the model will be very similar to our original code file, which we use as the prediction text.

Refactor a TypeScript class with a Predicted Output

```javascript
import OpenAI from "openai";

const code = `
class User {
  firstName: string = "";
  lastName: string = "";
  username: string = "";
}

export default User;
`.trim();

const openai = new OpenAI();

const refactorPrompt = `
Replace the "username" property with an "email" property. Respond only 
with code, and with no markdown formatting.
`;

const completion = await openai.chat.completions.create({
  model: "gpt-4o",
  messages: [
    {
      role: "user",
      content: refactorPrompt
    },
    {
      role: "user",
      content: code
    }
  ],
  prediction: {
    type: "content",
    content: code
  }
});

// Inspect returned data
console.log(completion);
console.log(completion.choices[0].message.content);
```

```python
from openai import OpenAI

code = """
class User {
  firstName: string = "";
  lastName: string = "";
  username: string = "";
}

export default User;
"""

refactor_prompt = """
Replace the "username" property with an "email" property. Respond only 
with code, and with no markdown formatting.
"""

client = OpenAI()

completion = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {
            "role": "user",
            "content": refactor_prompt
        },
        {
            "role": "user",
            "content": code
        }
    ],
    prediction={
        "type": "content",
        "content": code
    }
)

print(completion)
print(completion.choices[0].message.content)
```

```bash
curl https://api.openai.com/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -d '{
    "model": "gpt-4o",
    "messages": [
      {
        "role": "user",
        "content": "Replace the username property with an email property. Respond only with code, and with no markdown formatting."
      },
      {
        "role": "user",
        "content": "$CODE_CONTENT_HERE"
      }
    ],
    "prediction": {
        "type": "content",
        "content": "$CODE_CONTENT_HERE"
    }
  }'
```

In addition to the refactored code, the model response will contain data that looks something like this:

```javascript
{
  id: 'chatcmpl-xxx',
  object: 'chat.completion',
  created: 1730918466,
  model: 'gpt-4o-2024-08-06',
  choices: [ /* ...actual text response here... */],
  usage: {
    prompt_tokens: 81,
    completion_tokens: 39,
    total_tokens: 120,
    prompt_tokens_details: { cached_tokens: 0, audio_tokens: 0 },
    completion_tokens_details: {
      reasoning_tokens: 0,
      audio_tokens: 0,
      accepted_prediction_tokens: 18,
      rejected_prediction_tokens: 10
    }
  },
  system_fingerprint: 'fp_159d8341cc'
}
```

Note both the `accepted_prediction_tokens` and `rejected_prediction_tokens` in the `usage` object. In this example, 18 tokens from the prediction were used to speed up the response, while 10 were rejected.

Note that any rejected tokens are still billed like other completion tokens generated by the API, so Predicted Outputs can introduce higher costs for your requests.

Streaming example
-----------------

The latency gains of Predicted Outputs are even greater when you use streaming for API responses. Here is an example of the same code refactoring use case, but using streaming in the OpenAI SDKs instead.

Predicted Outputs with streaming

```javascript
import OpenAI from "openai";

const code = `
class User {
  firstName: string = "";
  lastName: string = "";
  username: string = "";
}

export default User;
`.trim();

const openai = new OpenAI();

const refactorPrompt = `
Replace the "username" property with an "email" property. Respond only 
with code, and with no markdown formatting.
`;

const completion = await openai.chat.completions.create({
  model: "gpt-4o",
  messages: [
    {
      role: "user",
      content: refactorPrompt
    },
    {
      role: "user",
      content: code
    }
  ],
  prediction: {
    type: "content",
    content: code
  },
  stream: true
});

// Inspect returned data
for await (const chunk of stream) {
  process.stdout.write(chunk.choices[0]?.delta?.content || "");
}
```

```python
from openai import OpenAI

code = """
class User {
  firstName: string = "";
  lastName: string = "";
  username: string = "";
}

export default User;
"""

refactor_prompt = """
Replace the "username" property with an "email" property. Respond only 
with code, and with no markdown formatting.
"""

client = OpenAI()

stream = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {
            "role": "user",
            "content": refactor_prompt
        },
        {
            "role": "user",
            "content": code
        }
    ],
    prediction={
        "type": "content",
        "content": code
    },
    stream=True
)

for chunk in stream:
    if chunk.choices[0].delta.content is not None:
        print(chunk.choices[0].delta.content, end="")
```

Position of predicted text in response
--------------------------------------

When providing prediction text, your prediction can appear anywhere within the generated response, and still provide latency reduction for the response. Let's say your predicted text is the simple [Hono](https://hono.dev/) server shown below:

```typescript
import { serveStatic } from "@hono/node-server/serve-static";
import { serve } from "@hono/node-server";
import { Hono } from "hono";

const app = new Hono();

app.get("/api", (c) => {
  return c.text("Hello Hono!");
});

// You will need to build the client code first `pnpm run ui:build`
app.use(
  "/*",
  serveStatic({
    rewriteRequestPath: (path) => `./dist${path}`,
  })
);

const port = 3000;
console.log(`Server is running on port ${port}`);

serve({
  fetch: app.fetch,
  port,
});
```

You could prompt the model to regenerate the file with a prompt like:

```text
Add a get route to this application that responds with 
the text "hello world". Generate the entire application 
file again with this route added, and with no other 
markdown formatting.
```

The response to the prompt might look something like this:

```typescript
import { serveStatic } from "@hono/node-server/serve-static";
import { serve } from "@hono/node-server";
import { Hono } from "hono";

const app = new Hono();

app.get("/api", (c) => {
  return c.text("Hello Hono!");
});

app.get("/hello", (c) => {
  return c.text("hello world");
});

// You will need to build the client code first `pnpm run ui:build`
app.use(
  "/*",
  serveStatic({
    rewriteRequestPath: (path) => `./dist${path}`,
  })
);

const port = 3000;
console.log(`Server is running on port ${port}`);

serve({
  fetch: app.fetch,
  port,
});
```

You would still see accepted prediction tokens in the response, even though the prediction text appeared both before and after the new content added to the response:

```javascript
{
  id: 'chatcmpl-xxx',
  object: 'chat.completion',
  created: 1731014771,
  model: 'gpt-4o-2024-08-06',
  choices: [ /* completion here... */],
  usage: {
    prompt_tokens: 203,
    completion_tokens: 159,
    total_tokens: 362,
    prompt_tokens_details: { cached_tokens: 0, audio_tokens: 0 },
    completion_tokens_details: {
      reasoning_tokens: 0,
      audio_tokens: 0,
      accepted_prediction_tokens: 60,
      rejected_prediction_tokens: 0
    }
  },
  system_fingerprint: 'fp_9ee9e968ea'
}
```

This time, there were no rejected prediction tokens, because the entire content of the file we predicted was used in the final response. Nice! 🔥

Limitations
-----------

When using Predicted Outputs, you should consider the following factors and limitations.

*   Predicted Outputs are only supported with the GPT-4o and GPT-4o-mini series of models.
*   When providing a prediction, any tokens provided that are not part of the final completion are still charged at completion token rates. See the [`rejected_prediction_tokens` property of the `usage` object](/docs/api-reference/chat/object#chat/object-usage) to see how many tokens are not used in the final response.
*   The following [API parameters](/docs/api-reference/chat/create) are not supported when using Predicted Outputs:
    *   `n`: values higher than 1 are not supported
    *   `logprobs`: not supported
    *   `presence_penalty`: values greater than 0 are not supported
    *   `frequency_penalty`: values greater than 0 are not supported
    *   `audio`: Predicted Outputs are not compatible with [audio inputs and outputs](/docs/guides/audio)
    *   `modalities`: Only `text` modalities are supported
    *   `max_completion_tokens`: not supported
    *   `tools`: Function calling is not currently supported with Predicted Outputs

===

Developer quickstart
====================

Learn how to make your first API request.

The OpenAI API provides a simple interface to state-of-the-art AI [models](/docs/models) for natural language processing, image generation, semantic search, and speech recognition. Follow this guide to learn how to generate human-like responses to [natural language prompts](/docs/guides/text-generation), [create vector embeddings](/docs/guides/embeddings) for semantic search, and [generate images](/docs/guides/images) from textual descriptions.

Create and export an API key
----------------------------

[Create an API key in the dashboard here](/api-keys), which you’ll use to securely [access the API](/docs/api-reference/authentication). Store the key in a safe location, like a [`.zshrc` file](https://www.freecodecamp.org/news/how-do-zsh-configuration-files-work/) or another text file on your computer. Once you’ve generated an API key, export it as an [environment variable](https://en.wikipedia.org/wiki/Environment_variable) in your terminal.

macOS / Linux

Export an environment variable on macOS or Linux systems

```bash
export OPENAI_API_KEY="your_api_key_here"
```

Windows

Export an environment variable in PowerShell

```bash
setx OPENAI_API_KEY "your_api_key_here"
```

Make your first API request
---------------------------

With your OpenAI API key exported as an environment variable, you're ready to make your first API request. You can either use the [REST API](/docs/api-reference) directly with the HTTP client of your choice, or use one of our [official SDKs](/docs/libraries) as shown below.

JavaScript

To use the OpenAI API in server-side JavaScript environments like Node.js, Deno, or Bun, you can use the official [OpenAI SDK for TypeScript and JavaScript](https://github.com/openai/openai-node). Get started by installing the SDK using [npm](https://www.npmjs.com/) or your preferred package manager:

Install the OpenAI SDK with npm

```bash
npm install openai
```

With the OpenAI SDK installed, create a file called `example.mjs` and copy one of the following examples into it:

Generate text

Create a human-like response to a prompt

```javascript
import OpenAI from "openai";
const openai = new OpenAI();

const completion = await openai.chat.completions.create({
    model: "gpt-4o-mini",
    messages: [
        { role: "system", content: "You are a helpful assistant." },
        {
            role: "user",
            content: "Write a haiku about recursion in programming.",
        },
    ],
});

console.log(completion.choices[0].message);
```

Generate an image

Generate an image based on a textual prompt

```javascript
import OpenAI from "openai";
const openai = new OpenAI();

const image = await openai.images.generate({ prompt: "A cute baby sea otter" });

console.log(image.data[0].url);
```

Create vector embeddings

Create vector embeddings for a string of text

```javascript
import OpenAI from "openai";
const openai = new OpenAI();

const embedding = await openai.embeddings.create({
    model: "text-embedding-3-large",
    input: "The quick brown fox jumped over the lazy dog",
});

console.log(embedding);
```

Execute the code with `node example.mjs` (or the equivalent command for Deno or Bun). In a few moments, you should see the output of your API request!

Python

To use the OpenAI API in Python, you can use the official [OpenAI SDK for Python](https://github.com/openai/openai-python). Get started by installing the SDK using [pip](https://pypi.org/project/pip/):

Install the OpenAI SDK with pip

```bash
pip install openai
```

With the OpenAI SDK installed, create a file called `example.py` and copy one of the following examples into it:

Generate text

Create a human-like response to a prompt

```python
from openai import OpenAI
client = OpenAI()

completion = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {
            "role": "user",
            "content": "Write a haiku about recursion in programming."
        }
    ]
)

print(completion.choices[0].message)
```

Generate an image

Generate an image based on a textual prompt

```python
from openai import OpenAI
client = OpenAI()

response = client.images.generate(
    prompt="A cute baby sea otter",
    n=2,
    size="1024x1024"
)

print(response.data[0].url)
```

Create vector embeddings

Create vector embeddings for a string of text

```python
from openai import OpenAI
client = OpenAI()

response = client.embeddings.create(
    model="text-embedding-3-large",
    input="The food was delicious and the waiter..."
)

print(response)
```

Execute the code with `python example.py`. In a few moments, you should see the output of your API request!

curl

On Unix-based systems, you can test out the [OpenAI REST API](/docs/api-reference) using [curl](https://curl.se/). The following commands assume that you have exported the `OPENAI_API_KEY` system environment variable as shown above.

Generate text

Create a human-like response to a prompt

```bash
curl "https://api.openai.com/v1/chat/completions" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $OPENAI_API_KEY" \
    -d '{
        "model": "gpt-4o-mini",
        "messages": [
            {
                "role": "system",
                "content": "You are a helpful assistant."
            },
            {
                "role": "user",
                "content": "Write a haiku that explains the concept of recursion."
            }
        ]
    }'
```

Generate an image

Generate an image based on a textual prompt

```bash
curl "https://api.openai.com/v1/images/generations" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $OPENAI_API_KEY" \
    -d '{
        "prompt": "A cute baby sea otter",
        "n": 2,
        "size": "1024x1024"
    }'
```

Create vector embeddings

Create vector embeddings for a string of text

```bash
curl "https://api.openai.com/v1/embeddings" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $OPENAI_API_KEY" \
    -d '{
        "input": "The food was delicious and the waiter...",
        "model": "text-embedding-3-large"
    }'
```

Execute the curl commands above in your terminal. In a few moments, you should see the output of your API request!

===

Realtime API with WebRTC

Beta

================================

Use WebRTC to connect client-side applications to the Realtime API.

[WebRTC](https://webrtc.org/) is a powerful set of standard interfaces for building real-time applications. The OpenAI Realtime API supports connecting to realtime models through a WebRTC peer connection. Follow this guide to learn how to configure a WebRTC connection to the Realtime API.

Overview
--------

In scenarios where you would like to connect to a Realtime model from an insecure client over the network (like a web browser), we recommend using the WebRTC connection method. WebRTC is better equipped to handle variable connection states, and provides a number of convenient APIs for capturing user audio inputs and playing remote audio streams from the model.

Connecting to the Realtime API from the browser should be done with an ephemeral API key, [generated via the OpenAI REST API](/docs/api-reference/realtime-sessions). The process for initializing a WebRTC connection is as follows (assuming a web browser client):

1.  A browser makes a request to a developer-controlled server to mint an ephemeral API key.
2.  The developer's server uses a [standard API key](/settings/api-keys) to request an ephemeral key from the [OpenAI REST API](/docs/api-reference/realtime-sessions), and returns that new key to the browser. Note that ephemeral keys currently expire one minute after being issued.
3.  The browser uses the ephemeral key to authenticate a session directly with the OpenAI Realtime API as a [WebRTC peer connection](https://developer.mozilla.org/en-US/docs/Web/API/RTCPeerConnection).

![connect to realtime via WebRTC](https://openaidevs.retool.com/api/file/55b47800-9aaf-48b9-90d5-793ab227ddd3)

While it is technically possible to use a [standard API key](/settings/api-keys) to authenticate WebRTC sessions, **this is a dangerous and insecure practice**. Standard API keys grant access to your full OpenAI API account, and should only be used in secure server-side environments. You should use ephemeral keys in client-side applications whenever possible.

Connection details
------------------

Connecting via WebRTC requires the following connection information:

|URL|https://api.openai.com/v1/realtime|
|Query Parameters|modelRealtime model ID to connect to, like gpt-4o-realtime-preview-2024-12-17|
|Headers|Authorization: Bearer EPHEMERAL_KEYSubstitute EPHEMERAL_KEY with an ephemeral API token - see below for details on how to generate one.|

The following example shows how to initialize a [WebRTC session](https://webrtc.org/getting-started/overview) (including the data channel to send and receive Realtime API events). It assumes you have already fetched an ephemeral API token (example server code for this can be found in the [next section](#creating-an-ephemeral-token)).

```javascript
async function init() {
  // Get an ephemeral key from your server - see server code below
  const tokenResponse = await fetch("/session");
  const data = await tokenResponse.json();
  const EPHEMERAL_KEY = data.client_secret.value;

  // Create a peer connection
  const pc = new RTCPeerConnection();

  // Set up to play remote audio from the model
  const audioEl = document.createElement("audio");
  audioEl.autoplay = true;
  pc.ontrack = e => audioEl.srcObject = e.streams[0];

  // Add local audio track for microphone input in the browser
  const ms = await navigator.mediaDevices.getUserMedia({
    audio: true
  });
  pc.addTrack(ms.getTracks()[0]);

  // Set up data channel for sending and receiving events
  const dc = pc.createDataChannel("oai-events");
  dc.addEventListener("message", (e) => {
    // Realtime server events appear here!
    console.log(e);
  });

  // Start the session using the Session Description Protocol (SDP)
  const offer = await pc.createOffer();
  await pc.setLocalDescription(offer);

  const baseUrl = "https://api.openai.com/v1/realtime";
  const model = "gpt-4o-realtime-preview-2024-12-17";
  const sdpResponse = await fetch(`${baseUrl}?model=${model}`, {
    method: "POST",
    body: offer.sdp,
    headers: {
      Authorization: `Bearer ${EPHEMERAL_KEY}`,
      "Content-Type": "application/sdp"
    },
  });

  const answer = {
    type: "answer",
    sdp: await sdpResponse.text(),
  };
  await pc.setRemoteDescription(answer);
}

init();
```

The WebRTC APIs provide rich controls for handling media streams and input devices. For more guidance on building user interfaces on top of WebRTC, [refer to the docs on MDN](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API).

Creating an ephemeral token
---------------------------

To create an ephemeral token to use on the client-side, you will need to build a small server-side application (or integrate with an existing one) to make an [OpenAI REST API](/docs/api-reference/realtime-sessions) request for an ephemeral key. You will use a [standard API key](/settings/api-keys) to authenticate this request on your backend server.

Below is an example of a simple Node.js [express](https://expressjs.com/) server which mints an ephemeral API key using the REST API:

```javascript
import express from "express";

const app = express();

// An endpoint which would work with the client code above - it returns
// the contents of a REST API request to this protected endpoint
app.get("/session", async (req, res) => {
  const r = await fetch("https://api.openai.com/v1/realtime/sessions", {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${process.env.OPENAI_API_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      model: "gpt-4o-realtime-preview-2024-12-17",
      voice: "verse",
    }),
  });
  const data = await r.json();

  // Send back the JSON we received from the OpenAI REST API
  res.send(data);
});

app.listen(3000);
```

You can create a server endpoint like this one on any platform that can send and receive HTTP requests. Just ensure that **you only use standard OpenAI API keys on the server, not in the browser.**

Sending and receiving events
----------------------------

To interact with the Realtime models, you will send and receive messages over the WebRTC data channel, and send and receive audio over media streams with the Realtime API as a connected peer. The full list of messages that clients can send, and that will be sent from the server, are found in the [API reference](/docs/api-reference/realtime-client-events). Once connected, you'll send and receive events which represent text, audio, function calls, interruptions, configuration updates, and more.

Here is how you can send and receive events over the data channel:

```javascript
// Create a data channel from a peer connection
const dc = pc.createDataChannel("oai-events");

// Listen for server-sent events on the data channel - event data 
// will need to be parsed from a JSON string
dc.addEventListener("message", (e) => {
  const realtimeEvent = JSON.parse(e.data);
  console.log(realtimeEvent);
});

// Send client events by serializing a valid client event to
// JSON, and sending it over the data channel
const responseCreate = {
  type: "response.create",
  response: {
    modalities: ["text"],
    instructions: "Write a haiku about code",
  },
};
dc.send(JSON.stringify(responseCreate));
```

Next steps
----------

Now that you have a functioning WebRTC connection to the Realtime API, it's time to learn more about building applications with Realtime models.

[

Realtime model capabilities

Learn about sessions with a Realtime model, where you can send and receive audio, manage conversations, make one-off requests to the model, and execute function calls.

](/docs/guides/realtime-model-capabilities)[

Event API reference

A complete listing of client and server events in the Realtime API

](/docs/api-reference/realtime-client-events)

===

Realtime API with WebSockets

Beta

====================================

Use WebSockets to connect to the Realtime API in server-to-server applications.

[WebSockets](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API) are a broadly supported API for realtime data transfer, and a great choice for connecting to the OpenAI Realtime API in server-to-server applications. For browser and mobile clients, we recommend connecting via [WebRTC](/docs/guides/realtime-webrtc). Follow this guide to connect to the Realtime API via WebSocket and start interacting with a Realtime model.

Overview
--------

In a server-to-server integration with Realtime, your backend system will connect via WebSocket directly to the Realtime API. You can use a [standard API key](/settings/api-keys) to authenticate this connection, since the token will only be available on your secure backend server.

![connect directly to realtime API](https://openaidevs.retool.com/api/file/464d4334-c467-4862-901b-d0c6847f003a)

WebSocket connections can also be authenticated with an ephemeral client token ([as shown here in the WebRTC connection guide](/docs/guides/realtime-webrtc)) if you choose to connect to the Realtime API via WebSocket on a client device.

  

Standard OpenAI API tokens **should only be used in secure server-side environments**.

Connection details
------------------

Connecting via WebSocket requires the following connection information:

|URL|wss://api.openai.com/v1/realtime|
|Query Parameters|modelRealtime model ID to connect to, like gpt-4o-realtime-preview-2024-12-17|
|Headers|Authorization: Bearer YOUR_API_KEYSubstitute YOUR_API_KEY with a standard API key on the server, or an ephemeral token on insecure clients (note that WebRTC is recommended for this use case).OpenAI-Beta: realtime=v1This header is required during the beta period.|

Below are several examples of using these connection details to initialize a WebSocket connection to the Realtime API.

ws module (Node.js)

Connect using the ws module (Node.js)

```javascript
import WebSocket from "ws";

const url = "wss://api.openai.com/v1/realtime?model=gpt-4o-realtime-preview-2024-12-17";
const ws = new WebSocket(url, {
  headers: {
    "Authorization": "Bearer " + process.env.OPENAI_API_KEY,
    "OpenAI-Beta": "realtime=v1",
  },
});

ws.on("open", function open() {
  console.log("Connected to server.");
});

ws.on("message", function incoming(message) {
  console.log(JSON.parse(message.data));
});
```

websocket-client (Python)

Connect with websocket-client (Python)

```python
# example requires websocket-client library:
# pip install websocket-client

import os
import json
import websocket

OPENAI_API_KEY = os.environ.get("OPENAI_API_KEY")

url = "wss://api.openai.com/v1/realtime?model=gpt-4o-realtime-preview-2024-12-17"
headers = [
    "Authorization: Bearer " + OPENAI_API_KEY,
    "OpenAI-Beta: realtime=v1"
]

def on_open(ws):
    print("Connected to server.")

def on_message(ws, message):
    data = json.loads(message)
    print("Received event:", json.dumps(data, indent=2))

ws = websocket.WebSocketApp(
    url,
    header=headers,
    on_open=on_open,
    on_message=on_message,
)

ws.run_forever()
```

WebSocket (browsers)

Connect with standard WebSocket (browsers)

```javascript
/*
Note that in client-side environments like web browsers, we recommend
using WebRTC instead. It is possible, however, to use the standard 
WebSocket interface in browser-like environments like Deno and 
Cloudflare Workers.
*/

const ws = new WebSocket(
  "wss://api.openai.com/v1/realtime?model=gpt-4o-realtime-preview-2024-12-17",
  [
    "realtime",
    // Auth
    "openai-insecure-api-key." + OPENAI_API_KEY, 
    // Optional
    "openai-organization." + OPENAI_ORG_ID,
    "openai-project." + OPENAI_PROJECT_ID,
    // Beta protocol, required
    "openai-beta.realtime-v1"
  ]
);

ws.on("open", function open() {
  console.log("Connected to server.");
});

ws.on("message", function incoming(message) {
  console.log(message.data);
});
```

Sending and receiving events
----------------------------

To interact with the Realtime models, you will send and receive messages over the WebSocket interface. The full list of messages that clients can send, and that will be sent from the server, are found in the [API reference](/docs/api-reference/realtime-client-events). Once connected, you'll send and receive events which represent text, audio, function calls, interruptions, configuration updates, and more.

Below, you'll find examples of how to send and receive events over the WebSocket interface in several programming environments.

WebSocket (Node.js / browser)

Send and receive events on a WebSocket (Node.js / browser)

```javascript
// Server-sent events will come in as messages...
ws.on("message", function incoming(message) {
  // Message data payloads will need to be parsed from JSON:
  const serverEvent = JSON.parse(message.data)
  console.log(serverEvent);
});

// To send events, create a JSON-serializeable data structure that
// matches a client-side event (see API reference)
const event = {
  type: "response.create",
  response: {
    modalities: ["audio", "text"],
    instructions: "Give me a haiku about code.",
  }
};
ws.send(JSON.stringify(event));
```

websocket-client (Python)

Send and receive messages with websocket-client (Python)

```python
# To send a client event, serialize a dictionary to JSON
# of the proper event type
def on_open(ws):
    print("Connected to server.")
    
    event = {
        "type": "response.create",
        "response": {
            "modalities": ["text"],
            "instructions": "Please assist the user."
        }
    }
    ws.send(json.dumps(event))

# Receiving messages will require parsing message payloads
# from JSON
def on_message(ws, message):
    data = json.loads(message)
    print("Received event:", json.dumps(data, indent=2))
```

Next steps
----------

Now that you have a functioning WebSocket connection to the Realtime API, it's time to learn more about building applications with Realtime models.

[

Realtime model capabilities

Learn about sessions with a Realtime model, where you can send and receive audio, manage conversations, make one-off requests to the model, and execute function calls.

](/docs/guides/realtime-model-capabilities)[

Event API reference

A complete listing of client and server events in the Realtime API

](/docs/api-reference/realtime-client-events)


Realtime model capabilities

Beta

===================================

Learn how to manage Realtime sessions, conversations, model responses, and function calls.

Once you have connected to the Realtime API through either [WebRTC](/docs/guides/realtime-webrtc) or [WebSocket](/docs/guides/realtime-websocket), you can build applications with a Realtime AI model. Doing so will require you to **send client events** to initiate actions, and **listen for server events** to respond to actions taken by the Realtime API. This guide will walk through the event flows required to use model capabilities like audio and text generation, and how to think about the state of a Realtime session.

About Realtime sessions
-----------------------

A Realtime session is a stateful interaction between the model and a connected client. The key components of the session are:

*   The **session** object, which controls the parameters of the interaction, like the model being used, the voice used to generate output, and other configuration.
*   A **conversation**, which represents user inputs and model outputs generated during the current session.
*   **Responses**, which are model-generated audio or text outputs that are added to the conversation.

**Input audio buffer and WebSockets**

If you are using WebRTC, much of the media handling required to send and receive audio from the model is assisted by WebRTC browser APIs.

  

If you are using WebSockets for audio, you will need to manually interact with the **input audio buffer** as well as the objects listed above. You'll be responsible for sending and receiving Base64-encoded audio bytes, and handling those as appropriate in your integration code.

All these components together make up a Realtime session. You will use client-sent events to update the state of the session, and listen for server-sent events to react to state changes within the session.

![diagram realtime state](https://openaidevs.retool.com/api/file/11fe71d2-611e-4a26-a587-881719a90e56)

Session lifecycle events
------------------------

After initiating a session via either [WebRTC](/docs/guides/realtime-webrtc) or [WebSockets](/docs/guides/realtime-websockets), the server will send a [`session.created`](/docs/api-reference/realtime-server-events/session/created) event indicating the session is ready. On the client, you can update the current session configuration with the [`session.update`](/docs/api-reference/realtime-client-events/session/update) event. Most session properties can be updated at any time, except for the `voice` the model uses for audio output, after the model has responded with audio once during the session. The maximum duration of a Realtime session is **30 minutes**.

The following example shows updating the session with a `session.update` client event. See the [WebRTC](/docs/guides/realtime-webrtc#sending-and-receiving-events) or [WebSocket](/docs/guides/realtime-websocket#sending-and-receiving-events) guide for more on sending client events over these channels.

Update the system instructions used by the model in this session

```javascript
const event = {
  type: "session.update",
  session: {
    instructions: "Never use the word 'moist' in your responses!"
  },
};

// WebRTC data channel and WebSocket both have .send()
dataChannel.send(JSON.stringify(event));
```

```python
event = {
    "type": "session.update",
    "session": {
        "instructions": "Never use the word 'moist' in your responses!"
    }
}
ws.send(json.dumps(event))
```

When the session has been updated, the server will emit a [`session.updated`](/docs/api-reference/realtime-server-events/session/updated) event with the new state of the session.

||
|session.update|session.createdsession.updated|

Text inputs and outputs
-----------------------

To generate text with a Realtime model, you can add text inputs to the current conversation, ask the model to generate a response, and listen for server-sent events indicating the progress of the model's response. In order to generate text, the [session must be configured](/docs/api-reference/realtime-client-events/session/update) with the `text` modality (this is true by default).

Create a new text conversation item using the [`conversation.item.create`](/docs/api-reference/realtime-client-events/conversation/item/create) client event. This is similar to sending a [user message (prompt) in chat completions](/docs/guides/text-generation) in the REST API.

Create a conversation item with user input

```javascript
const event = {
  type: "conversation.item.create",
  item: {
    type: "message",
    role: "user",
    content: [
      {
        type: "input_text",
        text: "What Prince album sold the most copies?",
      }
    ]
  },
};

// WebRTC data channel and WebSocket both have .send()
dataChannel.send(JSON.stringify(event));
```

```python
event = {
    "type": "conversation.item.create",
    "item": {
        "type": "message",
        "role": "user",
        "content": [
            {
                "type": "input_text",
                "text": "What Prince album sold the most copies?",
            }
        ]
    }
}
ws.send(json.dumps(event))
```

After adding the user message to the conversation, send the [`response.create`](/docs/api-reference/realtime-client-events/response/create) event to initiate a response from the model. If both audio and text are enabled for the current session, the model will respond with both audio and text content. If you'd like to generate text only, you can specify that when sending the `response.create` client event, as shown below.

Generate a text-only response

```javascript
const event = {
  type: "response.create",
  response: {
    modalities: [ "text" ]
  },
};

// WebRTC data channel and WebSocket both have .send()
dataChannel.send(JSON.stringify(event));
```

```python
event = {
    "type": "response.create",
    "response": {
        "modalities": [ "text" ]
    }
}
ws.send(json.dumps(event))
```

When the response is completely finished, the server will emit the [`response.done`](/docs/api-reference/realtime-server-events/response/done) event. This event will contain the full text generated by the model, as shown below.

Listen for response.done to see the final results

```javascript
function handleEvent(e) {
  const serverEvent = JSON.parse(e.data);
  if (serverEvent.type === "response.done") {
    console.log(serverEvent.response.output[0]);
  }
}

// Listen for server messages (WebRTC)
dataChannel.addEventListener("message", handleEvent);

// Listen for server messages (WebSocket)
// ws.on("message", handleEvent);
```

```python
def on_message(ws, message):
    server_event = json.loads(message)
    if server_event.type == "response.done":
        print(server_event.response.output[0])
```

While the model response is being generated, the server will emit a number of lifecycle events during the process. You can listen for these events, such as [`response.text.delta`](/docs/api-reference/realtime-server-events/response/text/delta), to provide realtime feedback to users as the response is generated. A full listing of the events emitted by there server are found below under **related server events**. They are provided in the rough order of when they are emitted, along with relevant client-side events for text generation.

||
|conversation.item.createresponse.create|conversation.item.createdresponse.createdresponse.output_item.addedresponse.content_part.addedresponse.text.deltaresponse.text.doneresponse.content_part.doneresponse.output_item.doneresponse.donerate_limits.updated|

Audio inputs and outputs
------------------------

One of the most powerful features of the Realtime API is voice-to-voice interaction with the model, without an intermediate text-to-speech or speech-to-text step. This enables lower latency for voice interfaces, and gives the model more data to work with around the tone and inflection of voice input.

### Handling audio with WebRTC

If you are connecting to the Realtime API using WebRTC, the Realtime API is acting as a [peer connection](https://developer.mozilla.org/en-US/docs/Web/API/RTCPeerConnection) to your client. Audio output from the model is delivered to your client as a [remote media stream](hhttps://developer.mozilla.org/en-US/docs/Web/API/MediaStream). Audio input to the model is collected using audio devices ([`getUserMedia`](https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getUserMedia)), and media streams are added as tracks to to the peer connection.

The example code from the [WebRTC connection guide](/docs/guides/realtime-webrtc) shows a basic example of configuring both local and remote audio:

```javascript
// Create a peer connection
const pc = new RTCPeerConnection();

// Set up to play remote audio from the model
const audioEl = document.createElement("audio");
audioEl.autoplay = true;
pc.ontrack = e => audioEl.srcObject = e.streams[0];

// Add local audio track for microphone input in the browser
const ms = await navigator.mediaDevices.getUserMedia({
  audio: true
});
pc.addTrack(ms.getTracks()[0]);
```

The snippet above should suffice for simple integrations with the Realtime API, but there's much more that can be done with the WebRTC APIs. For more examples of different kinds of user interfaces, check out the [WebRTC samples](https://github.com/webrtc/samples) repository. Live demos of these samples can also be [found here](https://webrtc.github.io/samples/).

Using [media captures and streams](https://developer.mozilla.org/en-US/docs/Web/API/Media_Capture_and_Streams_API) in the browser enables you to do things like mute and unmute microphones, select which device to collect input from, and more.

### Client and server events for audio in WebRTC

By default, WebRTC clients don't need to send any client events to the Realtime API to start sending audio inputs. Once a local audio track is added to the peer connection, your users can just start talking!

However, WebRTC clients still receive a number of server-sent lifecycle events as audio is moving back and forth between client and server over the peer connection. An incomplete sample of server events that are sent during a WebRTC session:

*   When input is sent over the local media track, you will receive [`input_audio_buffer.speech_started`](/docs/api-reference/realtime-server-events/input_audio_buffer/speech_started) events from the server.
*   When local audio input stops, you'll receive the [`input_audio_buffer.speech_stopped`](/docs/api-reference/realtime-server-events/input_audio_buffer/speech_started) event.
*   You'll receive [delta events for the in-progress audio transcript](/docs/api-reference/realtime-server-events/response/audio_transcript/delta).
*   You'll receive a [`response.done`](/docs/api-reference/realtime-server-events/response/done) event when the model has transcribed and completed sending a response.

Manipulating WebRTC APIs for media streams may give you all the control you need in your application. However, it may occasionally be necessary to use lower-level interfaces for audio input and output. Refer to the WebSockets section below for more information and a listing of events required for granular audio input handling.

### Handling audio with WebSockets

When sending and receiving audio over a WebSocket, you will have a bit more work to do in order to send media from the client, and receive media from the server. Below, you'll find a table describing the flow of events during a WebSocket session that are necessary to send and receive audio over the WebSocket.

The events below are given in lifecycle order, though some events (like the `delta` events) may happen concurrently.

||
|Session initialization|session.update|session.createdsession.updated|
|User audio input|conversation.item.create  (send whole audio message)input_audio_buffer.append  (stream audio in chunks)input_audio_buffer.commit  (used when VAD is disabled)response.create  (used when VAD is disabled)|input_audio_buffer.speech_startedinput_audio_buffer.speech_stoppedinput_audio_buffer.committed|
|Server audio output|input_audio_buffer.clear  (used when VAD is disabled)|conversation.item.createdresponse.createdresponse.output_item.createdresponse.content_part.addedresponse.audio.deltaresponse.audio_transcript.deltaresponse.text.deltaresponse.audio.doneresponse.audio_transcript.doneresponse.text.doneresponse.content_part.doneresponse.output_item.doneresponse.donerate_limits.updated|

### Streaming audio input to the server

To stream audio input to the server, you can use the [`input_audio_buffer.append`](/docs/api-reference/realtime-client-events/input_audio_buffer/append) client event. This event requires you to send chunks of **Base64-encoded audio bytes** to the Realtime API over the socket. Each chunk cannot exceed 15 MB in size.

The format of the input chunks can be configured either for the entire session, or per response.

*   Session: `session.input_audio_format` in [`session.update`](/docs/api-reference/realtime-client-events/session/update)
*   Response: `response.input_audio_format` in [`response.create`](/docs/api-reference/realtime-client-events/response/create)

Append audio input bytes to the conversation

```javascript
import fs from 'fs';
import decodeAudio from 'audio-decode';

// Converts Float32Array of audio data to PCM16 ArrayBuffer
function floatTo16BitPCM(float32Array) {
  const buffer = new ArrayBuffer(float32Array.length * 2);
  const view = new DataView(buffer);
  let offset = 0;
  for (let i = 0; i < float32Array.length; i++, offset += 2) {
    let s = Math.max(-1, Math.min(1, float32Array[i]));
    view.setInt16(offset, s < 0 ? s * 0x8000 : s * 0x7fff, true);
  }
  return buffer;
}

// Converts a Float32Array to base64-encoded PCM16 data
base64EncodeAudio(float32Array) {
  const arrayBuffer = floatTo16BitPCM(float32Array);
  let binary = '';
  let bytes = new Uint8Array(arrayBuffer);
  const chunkSize = 0x8000; // 32KB chunk size
  for (let i = 0; i < bytes.length; i += chunkSize) {
    let chunk = bytes.subarray(i, i + chunkSize);
    binary += String.fromCharCode.apply(null, chunk);
  }
  return btoa(binary);
}

// Fills the audio buffer with the contents of three files,
// then asks the model to generate a response.
const files = [
  './path/to/sample1.wav',
  './path/to/sample2.wav',
  './path/to/sample3.wav'
];

for (const filename of files) {
  const audioFile = fs.readFileSync(filename);
  const audioBuffer = await decodeAudio(audioFile);
  const channelData = audioBuffer.getChannelData(0);
  const base64Chunk = base64EncodeAudio(channelData);
  ws.send(JSON.stringify({
    type: 'input_audio_buffer.append',
    audio: base64Chunk
  }));
});

ws.send(JSON.stringify({type: 'input_audio_buffer.commit'}));
ws.send(JSON.stringify({type: 'response.create'}));
```

```python
import base64
import json
import struct
import soundfile as sf
from websocket import create_connection

# ... create websocket-client named ws ...

def float_to_16bit_pcm(float32_array):
    clipped = [max(-1.0, min(1.0, x)) for x in float32_array]
    pcm16 = b''.join(struct.pack('<h', int(x * 32767)) for x in clipped)
    return pcm16

def base64_encode_audio(float32_array):
    pcm_bytes = float_to_16bit_pcm(float32_array)
    encoded = base64.b64encode(pcm_bytes).decode('ascii')
    return encoded

files = [
    './path/to/sample1.wav',
    './path/to/sample2.wav',
    './path/to/sample3.wav'
]

for filename in files:
    data, samplerate = sf.read(filename, dtype='float32')  
    channel_data = data[:, 0] if data.ndim > 1 else data
    base64_chunk = base64_encode_audio(channel_data)
    
    # Send the client event
    event = {
        "type": "input_audio_buffer.append",
        "audio": base64_chunk
    }
    ws.send(json.dumps(event))
```

### Send full audio messages

It is also possible to create conversation messages that are full audio recordings. Use the [`conversation.item.create`](/docs/api-reference/realtime-client-events/conversation/item/create) client event to create messages with `input_audio` content.

Create full audio input conversation items

```javascript
const fullAudio = "<a base64-encoded string of audio bytes>";

const event = {
  type: "conversation.item.create",
  item: {
    type: "message",
    role: "user",
    content: [
      {
        type: "input_audio",
        audio: fullAudio,
      },
    ],
  },
};

// WebRTC data channel and WebSocket both have .send()
dataChannel.send(JSON.stringify(event));
```

```python
fullAudio = "<a base64-encoded string of audio bytes>"

event = {
    "type": "conversation.item.create",
    "item": {
        "type": "message",
        "role": "user",
        "content": [
            {
                "type": "input_audio",
                "audio": fullAudio,
            }
        ],
    },
}

ws.send(json.dumps(event))
```

### Working with audio output from a WebSocket

**To play output audio back on a client device like a web browser, we recommend using WebRTC rather than WebSockets**. WebRTC will be more robust sending media to client devices over uncertain network conditions.

But to work with audio output in server-to-server applications using a WebSocket, you will need to listen for [`response.audio.delta`](/docs/api-reference/realtime-server-events/response/audio/delta) events containing the Base64-encoded chunks of audio data from the model. You will either need to buffer these chunks and write them out to a file, or maybe immediately stream them to another source like [a phone call with Twilio](https://www.twilio.com/en-us/blog/twilio-openai-realtime-api-launch-integration).

Note that the [`response.audio.done`](/docs/api-reference/realtime-server-events/response/audio/done) and [`response.done`](/docs/api-reference/realtime-server-events/response/done) events won't actually contain audio data in them - just audio content transcriptions. To get the actual bytes, you'll need to listen for the [`response.audio.delta`](/docs/api-reference/realtime-server-events/response/audio/delta) events.

The format of the output chunks can be configured either for the entire session, or per response.

*   Session: `session.output_audio_format` in [`session.update`](/docs/api-reference/realtime-client-events/session/update)
*   Response: `response.output_audio_format` in [`response.create`](/docs/api-reference/realtime-client-events/response/create)

Listen for response.audio.delta events

```javascript
function handleEvent(e) {
  const serverEvent = JSON.parse(e.data);
  if (serverEvent.type === "response.audio.delta") {
    // Access Base64-encoded audio chunks
    // console.log(serverEvent.delta);
  }
}

// Listen for server messages (WebSocket)
ws.on("message", handleEvent);
```

```python
def on_message(ws, message):
    server_event = json.loads(message)
    if server_event.type == "response.audio.delta":
        # Access Base64-encoded audio chunks:
        # print(server_event.delta)
```

Voice activity detection (VAD)
------------------------------

By default, Realtime sessions have **voice activity detection (VAD)** enabled, which means the API will determine when the user has started or stopped speaking, and automatically start to respond. The behavior and sensitivity of VAD can be configured through the `session.turn_detection` property of the [`session.update`](/docs/api-reference/realtime-client-events/session/update) client event.

VAD can be disabled by setting `turn_detection` to `null` with the [`session.update`](/docs/api-reference/realtime-client-events/session/update) client event. This can be useful for interfaces where you would like to take granular control over audio input, like [push to talk](https://en.wikipedia.org/wiki/Push-to-talk) interfaces.

When VAD is disabled, the client will have to manually emit some additional client events to trigger audio responses:

*   Manually send [`input_audio_buffer.commit`](/docs/api-reference/realtime-client-events/input_audio_buffer/commit), which will create a new user input item for the conversation.
*   Manually send [`response.create`](/docs/api-reference/realtime-client-events/response/create) to trigger an audio response from the model.
*   Send [`input_audio_buffer.clear`](/docs/api-reference/realtime-client-events/input_audio_buffer/clear) before beginning a new user input.

### Keep VAD, but disable automatic responses

If you would like to keep VAD mode enabled, but would just like to retain the ability to manually decide when a response is generated, you can set `turn_detection.create_response` to `false` with the [`session.update`](/docs/api-reference/realtime-client-events/session/update) client event. This will retain all the behavior of VAD, but still require you to manually send a [`response.create`](/docs/api-reference/realtime-client-events/response/create) event before a response is generated by the model.

This can be useful for moderation or input validation, where you're comfortable trading a bit more latency in the interaction for control over inputs.

Create responses outside the default conversation
-------------------------------------------------

By default, all responses generated during a session are added to the session's conversation state (the "default conversation"). However, you may want to generate model responses outside the context of the session's default conversation, or have multiple responses generated concurrently. You might also want to have more granular control over which conversation items are considered while the model generates a response (e.g. only the last N number of turns).

Generating "out-of-band" responses which are not added to the default conversation state is possible by setting the `response.conversation` field to the string `none` when creating a response with the [`response.create`](/docs/api-reference/realtime-client-events/response/create) client event.

When creating an out-of-band response, you will probably also want some way to identify which server-sent events pertain to this response. You can provide `metadata` for your model response that will help you identify which response is being generated for this client-sent event.

Create an out-of-band model response

```javascript
const prompt = `
Analyze the conversation so far. If it is related to support, output
"support". If it is related to sales, output "sales".
`;

const event = {
  type: "response.create",
  response: {
    // Setting to "none" indicates the response is out of band
    // and will not be added to the default conversation
    conversation: "none",

    // Set metadata to help identify responses sent back from the model
    metadata: { topic: "classification" },
    
    // Set any other available response fields
    modalities: [ "text" ],
    instructions: prompt,
  },
};

// WebRTC data channel and WebSocket both have .send()
dataChannel.send(JSON.stringify(event));
```

```python
prompt = """
Analyze the conversation so far. If it is related to support, output
"support". If it is related to sales, output "sales".
"""

event = {
    "type": "response.create",
    "response": {
        # Setting to "none" indicates the response is out of band,
        # and will not be added to the default conversation
        "conversation": "none",

        # Set metadata to help identify responses sent back from the model
        "metadata": { "topic": "classification" },

        # Set any other available response fields
        "modalities": [ "text" ],
        "instructions": prompt,
    },
}

ws.send(json.dumps(event))
```

Now, when you listen for the [`response.done`](/docs/api-reference/realtime-server-events/response/done) server event, you can identify the result of your out-of-band response.

Create an out-of-band model response

```javascript
function handleEvent(e) {
  const serverEvent = JSON.parse(e.data);
  if (
    serverEvent.type === "response.done" &&
    serverEvent.response.metadata?.topic === "classification"
  ) {
    // this server event pertained to our OOB model response
    console.log(serverEvent.response.output[0]);
  }
}

// Listen for server messages (WebRTC)
dataChannel.addEventListener("message", handleEvent);

// Listen for server messages (WebSocket)
// ws.on("message", handleEvent);
```

```python
def on_message(ws, message):
    server_event = json.loads(message)
    topic = ""

    # See if metadata is present
    try:
        topic = server_event.response.metadata.topic
    except AttributeError:
        print("topic not set")
    
    if server_event.type == "response.done" and topic == "classification":
        # this server event pertained to our OOB model response
        print(server_event.response.output[0])
```

### Create a custom context for responses

You can also construct a custom context that the model will use to generate a response, outside the default/current conversation. This can be done using the `input` array on a [`response.create`](/docs/api-reference/realtime-client-events/response/create) client event. You can use new inputs, or reference existing input items in the conversation by ID.

Listen for out-of-band model response with custom context

```javascript
const event = {
  type: "response.create",
  response: {
    conversation: "none",
    metadata: { topic: "pizza" },
    modalities: [ "text" ],

    // Create a custom input array for this request with whatever context
    // is appropriate
    input: [
      // potentially include existing conversation items:
      {
        type: "item_reference",
        id: "some_conversation_item_id"
      },
      {
        type: "message",
        role: "user",
        content: [
          {
            type: "input_text",
            text: "Is it okay to put pineapple on pizza?",
          },
        ],
      },
    ],
  },
};

// WebRTC data channel and WebSocket both have .send()
dataChannel.send(JSON.stringify(event));
```

```python
event = {
    "type": "response.create",
    "response": {
        "conversation": "none",
        "metadata": { "topic": "pizza" },
        "modalities": [ "text" ],

        # Create a custom input array for this request with whatever 
        # context is appropriate
        "input": [
            # potentially include existing conversation items:
            {
                "type": "item_reference",
                "id": "some_conversation_item_id"
            },

            # include new content as well
            {
                "type": "message",
                "role": "user",
                "content": [
                    {
                        "type": "input_text",
                        "text": "Is it okay to put pineapple on pizza?",
                    }
                ],
            }
        ],
    },
}

ws.send(json.dumps(event))
```

### Create responses with no context

You can also insert responses into the default conversation, ignoring all other instructions and context. Do this by setting `input` to an empty array.

Insert no-context model responses into the default conversation

```javascript
const prompt = `
Say exactly the following:
I'm a little teapot, short and stout! 
This is my handle, this is my spout!
`;

const event = {
  type: "response.create",
  response: {
    // An empty input array removes existing context
    input: [],
    instructions: prompt,
  },
};

// WebRTC data channel and WebSocket both have .send()
dataChannel.send(JSON.stringify(event));
```

```python
prompt = """
Say exactly the following:
I'm a little teapot, short and stout! 
This is my handle, this is my spout!
"""

event = {
    "type": "response.create",
    "response": {
        # An empty input array removes all prior context
        "input": [],
        "instructions": prompt,
    },
}

ws.send(json.dumps(event))
```

Function calling
----------------

The Realtime models also support **function calling**, which enables you to execute custom code to extend the capabilities of the model. Here's how it works at a high level:

1.  When [updating the session](/docs/api-reference/realtime-client-events/session/update) or [creating a response](/docs/api-reference/realtime-client-events/response/create), you can specify a list of available functions for the model to call.
2.  If when processing input, the model determines it should make a function call, it will add items to the conversation representing arguments to a function call.
3.  When the client detects conversation items that contain function call arguments, it will execute custom code using those arguments
4.  When the custom code has been executed, the client will create new conversation items that contain the output of the function call, and ask the model to respond.

Let's see how this would work in practice by adding a callable function that will provide today's horoscope to users of the model. We'll show the shape of the client event objects that need to be sent, and what the server will emit in turn.

### Configure callable functions

First, we must give the model a selection of functions it can call based on user input. Available functions can be configured either at the session level, or the individual response level.

*   Session: `session.tools` property in [`session.update`](/docs/api-reference/realtime-client-events/session/update)
*   Response: `response.tools` property in [`response.create`](/docs/api-reference/realtime-client-events/response/create)

Here's an example client event payload for a `session.update` that configures a horoscope generation function, that takes a single argument (the astrological sign for which the horoscope should be generated):

[`session.update`](/docs/api-reference/realtime-client-events/session/update)

```json
{
  "type": "session.update",
  "session": {
    "tools": [
      {
        "type": "function",
        "name": "generate_horoscope",
        "description": "Give today's horoscope for an astrological sign.",
        "parameters": {
          "type": "object",
          "properties": {
            "sign": {
              "type": "string",
              "description": "The sign for the horoscope.",
              "enum": [
                "Aries",
                "Taurus",
                "Gemini",
                "Cancer",
                "Leo",
                "Virgo",
                "Libra",
                "Scorpio",
                "Sagittarius",
                "Capricorn",
                "Aquarius",
                "Pisces"
              ]
            }
          },
          "required": ["sign"]
        }
      }
    ],
    "tool_choice": "auto",
  }
}
```

The `description` fields for the function and the parameters help the model choose whether or not to call the function, and what data to include in each parameter. If the model receives input that indicates the user wants their horoscope, it will call this function with a `sign` parameter.

### Detect when the model wants to call a function

Based on inputs to the model, the model may decide to call a function in order to generate the best response. Let's say our application adds the following conversation item and attempts to generate a response:

[`conversation.item.create`](/docs/api-reference/realtime-client-events/conversation/item/create)

```json
{
  "type": "conversation.item.create",
  "item": {
    "type": "message",
    "role": "user",
    "content": [
      {
        "type": "input_text",
        "text": "What is my horoscope? I am an aquarius."
      }
    ]
  }
}
```

Followed by a client event to generate a response:

[`response.create`](/docs/api-reference/realtime-client-events/response/create)

```json
{
  "type": "response.create"
}
```

Instead of immediately returning a text or audio response, the model will instead generate a response that contains the arguments that should be passed to a function in the developer's application. You can listen for realtime updates to function call arguments using the [`response.function_call_arguments.delta`](/docs/api-reference/realtime-server-events/response/function_call_arguments/delta) server event, but `response.done` will also have the complete data we need to call our function.

[`response.done`](/docs/api-reference/realtime-server-events/response/done)

```json
{
  "type": "response.done",
  "event_id": "event_AeqLA8iR6FK20L4XZs2P6",
  "response": {
    "object": "realtime.response",
    "id": "resp_AeqL8XwMUOri9OhcQJIu9",
    "status": "completed",
    "status_details": null,
    "output": [
      {
        "object": "realtime.item",
        "id": "item_AeqL8gmRWDn9bIsUM2T35",
        "type": "function_call",
        "status": "completed",
        "name": "generate_horoscope",
        "call_id": "call_sHlR7iaFwQ2YQOqm",
        "arguments": "{\"sign\":\"Aquarius\"}"
      }
    ],
    "usage": {
      "total_tokens": 541,
      "input_tokens": 521,
      "output_tokens": 20,
      "input_token_details": {
        "text_tokens": 292,
        "audio_tokens": 229,
        "cached_tokens": 0,
        "cached_tokens_details": { "text_tokens": 0, "audio_tokens": 0 }
      },
      "output_token_details": {
        "text_tokens": 20,
        "audio_tokens": 0
      }
    },
    "metadata": null
  }
}
```

In the JSON emitted by the server, we can detect that the model wants to call a custom function:

|Property|Function calling purpose|
|---|---|
|response.output[0].type|When set to function_call, indicates this response contains arguments for a named function call.|
|response.output[0].name|The name of the configured function to call, in this case generate_horoscope|
|response.output[0].arguments|A JSON string containing arguments to the function. In our case, "{\"sign\":\"Aquarius\"}".|
|response.output[0].call_id|A system-generated ID for this function call - you will need this ID to pass a function call result back to the model.|

Given this information, we can execute code in our application to generate the horoscope, and then provide that information back to the model so it can generate a response.

### Provide the results of a function call to the model

Upon receiving a response from the model with arguments to a function call, your application can execute code that satisfies the function call. This could be anything you want, like talking to external APIs or accessing databases.

Once you are ready to give the model the results of your custom code, you can create a new conversation item containing the result via the `conversation.item.create` client event.

[`conversation.item.create`](/docs/api-reference/realtime-client-events/conversation/item/create)

```json
{
  "type": "conversation.item.create",
  "item": {
    "type": "function_call_output",
    "call_id": "call_sHlR7iaFwQ2YQOqm",
    "output": "{\"horoscope\": \"You will soon meet a new friend.\"}"
  }
}
```

*   The conversation item type is `function_call_output`
*   `item.call_id` is the same ID we got back in the `response.done` event above
*   `item.output` is a JSON string containing the results of our function call

Once we have added the conversation item containing our function call results, we again emit the `response.create` event from the client. This will trigger a model response using the data from the function call.

[`response.create`](/docs/api-reference/realtime-client-events/response/create)

```json
{
  "type": "response.create"
}
```

Error handling
--------------

The [`error`](/docs/api-reference/realtime-server-events/error) event is emitted by the server whenever an error condition is encountered on the server during the session. Occasionally, these errors can be traced to a client event that was emitted by your application.

Unlike HTTP requests and responses, where a response is implicitly tied to a request from the client, we need to use an `event_id` property on client events to know when one of them has triggered an error condition on the server. This technique is shown in the code below, where the client attempts to emit an unsupported event type.

```javascript
const event = {
  event_id: "my_awesome_event",
  type: "scooby.dooby.doo",
};

dataChannel.send(JSON.stringify(event));
```

This unsuccessful event sent from the client will emit an error event like the following:

```json
{
  "type": "invalid_request_error",
  "code": "invalid_value",
  "message": "Invalid value: 'scooby.dooby.doo' ...",
  "param": "type",
  "event_id": "my_awesome_event"
}
```

Next steps
----------

Realtime models unlock new possibilities for AI interactions. We can't wait to hear about what you create with the Realtime API! As you continue to explore, here are a few other resources that may be useful.

[

Realtime Console

The Realtime console sample app shows how to exercise function calling, client and server events, and much more.

](https://github.com/openai/openai-realtime-console)[

Event API reference

A complete listing of client and server events in the Realtime API

](/docs/api-reference/realtime-client-events)


Reasoning models
================

Explore advanced reasoning and problem-solving models.

**OpenAI o1** series models are new large language models trained with reinforcement learning to perform complex reasoning. o1 models [think before they answer](https://openai.com/index/introducing-openai-o1-preview/), producing a long internal chain of thought before responding to the user. o1 models excel in scientific reasoning, ranking in the 89th percentile on competitive programming questions (Codeforces), placing among the top 500 students in the US in a qualifier for the USA Math Olympiad (AIME), and exceeding human PhD-level accuracy on a benchmark of physics, biology, and chemistry problems (GPQA).

There are two reasoning models available in the API:

1.  `o1`: designed to reason about hard problems using broad general knowledge about the world.
2.  `o1-mini`: a faster and more affordable version of o1, particularly adept at coding, math, and science tasks where extensive general knowledge isn't required.

Quickstart
----------

Both `o1` and `o1-mini` are available through the [chat completions](/docs/api-reference/chat/create) endpoint.

Using the o1 model

```javascript
import OpenAI from "openai";
const openai = new OpenAI();

const prompt = `
Write a bash script that takes a matrix represented as a string with 
format '[1,2],[3,4],[5,6]' and prints the transpose in the same format.
`;
 
const completion = await openai.chat.completions.create({
  model: "o1",
  messages: [
    {
      role: "user", 
      content: prompt
    }
  ]
});

console.log(completion.choices[0].message.content);
```

```python
from openai import OpenAI
client = OpenAI()

prompt = """
Write a bash script that takes a matrix represented as a string with 
format '[1,2],[3,4],[5,6]' and prints the transpose in the same format.
"""

response = client.chat.completions.create(
    model="o1",
    messages=[
        {
            "role": "user", 
            "content": prompt
        }
    ]
)

print(response.choices[0].message.content)
```

```bash
curl https://api.openai.com/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -d '{
    "model": "o1-preview",
    "messages": [
      {
        "role": "user",
        "content": "Write a bash script that takes a matrix represented as a string with format \"[1,2],[3,4],[5,6]\" and prints the transpose in the same format."
      }
    ]
  }'
```

Note that depending on the amount of reasoning required by the model to solve the problem, these requests can take significantly longer than other models - sometimes on the order of minutes.

How reasoning works
-------------------

The o1 models introduce **reasoning tokens**. The models use these reasoning tokens to "think", breaking down their understanding of the prompt and considering multiple approaches to generating a response. After generating reasoning tokens, the model produces an answer as visible completion tokens, and discards the reasoning tokens from its context.

Here is an example of a multi-step conversation between a user and an assistant. Input and output tokens from each step are carried over, while reasoning tokens are discarded.

![Reasoning tokens aren't retained in context](https://cdn.openai.com/API/docs/images/context-window.png)

While reasoning tokens are not visible via the API, they still occupy space in the model's context window and are billed as [output tokens](https://openai.com/api/pricing).

### Managing the context window

The o1 and o1-mini models offer a context window of 200,000 and 128,000 tokens respectively. Each completion has an upper limit on the maximum number of output tokens — this includes both the invisible reasoning tokens and the visible completion tokens. The maximum output token limits are:

*   o1: Up to 100,000 tokens
*   o1-mini: Up to 65,536 tokens

It's important to ensure there's enough space in the context window for reasoning tokens when creating completions. Depending on the problem's complexity, the models may generate anywhere from a few hundred to tens of thousands of reasoning tokens. The exact number of reasoning tokens used is visible in the [usage object of the chat completion response object](/docs/api-reference/chat/object), under `completion_tokens_details`:

```json
{
  "usage": {
    "prompt_tokens": 9,
    "completion_tokens": 12,
    "total_tokens": 21,
    "completion_tokens_details": {
      "reasoning_tokens": 0,
      "accepted_prediction_tokens": 0,
      "rejected_prediction_tokens": 0
    }
  }
}
```

### Controlling costs

To manage costs with the o1 series models, you can limit the total number of tokens the model generates (including both reasoning and completion tokens) by using the [`max_completion_tokens`](/docs/api-reference/chat/create#chat-create-max_completion_tokens) parameter.

In previous models, the `max_tokens` parameter controlled both the number of tokens generated and the number of tokens visible to the user, which were always equal. However, with the o1 series, the total tokens generated can exceed the number of visible tokens due to the internal reasoning tokens.

Because some applications might rely on `max_tokens` matching the number of tokens received from the API, the o1 series introduces `max_completion_tokens` to explicitly control the total number of tokens generated by the model, including both reasoning and visible completion tokens. This explicit opt-in ensures no existing applications break when using the new models. The `max_tokens` parameter continues to function as before for all previous models.

### Allocating space for reasoning

If the generated tokens reach the context window limit or the `max_completion_tokens` value you've set, you'll receive a chat completion response with the `finish_reason` set to `length`. This might occur before any visible completion tokens are produced, meaning you could incur costs for input and reasoning tokens without receiving a visible response.

To prevent this, ensure there's sufficient space in the context window or adjust the `max_completion_tokens` value to a higher number. OpenAI recommends reserving at least 25,000 tokens for reasoning and outputs when you start experimenting with these models. As you become familiar with the number of reasoning tokens your prompts require, you can adjust this buffer accordingly.

Advice on prompting
-------------------

These models perform best with straightforward prompts. Some prompt engineering techniques, like few-shot learning or instructing the model to "think step by step," may not enhance performance (and can sometimes hinder it). Here are some best practices:

*   **Developer messages are the new system messages:** Starting with `o1-2024-12-17`, o1 models support `developer` messages rather than `system` messages, to align with the [chain of command behavior described in the model spec](https://cdn.openai.com/spec/model-spec-2024-05-08.html#follow-the-chain-of-command). [Learn more](/docs/guides/text-generation#building-prompts).
*   **Keep prompts simple and direct:** The models excel at understanding and responding to brief, clear instructions without the need for extensive guidance.
*   **Avoid chain-of-thought prompts:** Since these models perform reasoning internally, prompting them to "think step by step" or "explain your reasoning" is unnecessary.
*   **Use delimiters for clarity:** Use delimiters like triple quotation marks, XML tags, or section titles to clearly indicate distinct parts of the input, helping the model interpret different sections appropriately.
*   **Limit additional context in retrieval-augmented generation (RAG):** When providing additional context or documents, include only the most relevant information to prevent the model from overcomplicating its response.
*   **Markdown formatting:** Starting with `o1-2024-12-17`, o1 models in the API will avoid generating responses with markdown formatting. To signal to the model when you **do** want markdown formatting in the response, include the string `Formatting re-enabled` on the first line of your `developer` message.

### Prompt examples

Coding (refactoring)

OpenAI o1 series models are able to implement complex algorithms and produce code. This prompt asks o1 to refactor a React component based on some specific criteria.

Ask o1 models to refactor code

```javascript
import OpenAI from "openai";
const openai = new OpenAI();

const prompt = `
Instructions:
- Given the React component below, change it so that nonfiction books have red
  text. 
- Return only the code in your reply
- Do not include any additional formatting, such as markdown code blocks
- For formatting, use four space tabs, and do not allow any lines of code to 
  exceed 80 columns

const books = [
  { title: 'Dune', category: 'fiction', id: 1 },
  { title: 'Frankenstein', category: 'fiction', id: 2 },
  { title: 'Moneyball', category: 'nonfiction', id: 3 },
];

export default function BookList() {
  const listItems = books.map(book =>
    <li>
      {book.title}
    </li>
  );

  return (
    <ul>{listItems}</ul>
  );
}
`.trim();

const completion = await openai.chat.completions.create({
  model: "o1-mini",
  messages: [
    {
      role: "user",
      content: prompt,
    },
  ]
});

console.log(completion.usage.completion_tokens_details);
```

```python
from openai import OpenAI

client = OpenAI()

prompt = """
Instructions:
- Given the React component below, change it so that nonfiction books have red
  text. 
- Return only the code in your reply
- Do not include any additional formatting, such as markdown code blocks
- For formatting, use four space tabs, and do not allow any lines of code to 
  exceed 80 columns

const books = [
  { title: 'Dune', category: 'fiction', id: 1 },
  { title: 'Frankenstein', category: 'fiction', id: 2 },
  { title: 'Moneyball', category: 'nonfiction', id: 3 },
];

export default function BookList() {
  const listItems = books.map(book =>
    <li>
      {book.title}
    </li>
  );

  return (
    <ul>{listItems}</ul>
  );
}
"""

response = client.chat.completions.create(
    model="o1-mini",
    messages=[
        {
            "role": "user",
            "content": [
                {
                    "type": "text",
                    "text": prompt
                },
            ],
        }
    ]
)

print(response.choices[0].message.content)
```

Coding (planning)

OpenAI o1 series models are also adept in creating multi-step plans. This example prompt asks o1 to create a filesystem structure for a full solution, along with Python code that implements the desired use case.

Ask o1 models to plan and create a Python project

```javascript
import OpenAI from "openai";
const openai = new OpenAI();

const prompt = `
I want to build a Python app that takes user questions and looks 
them up in a database where they are mapped to answers. If there 
is close match, it retrieves the matched answer. If there isn't, 
it asks the user to provide an answer and stores the 
question/answer pair in the database. Make a plan for the directory 
structure you'll need, then return each file in full. Only supply 
your reasoning at the beginning and end, not throughout the code.
`.trim();

const completion = await openai.chat.completions.create({
  model: "o1-mini",
  messages: [
    {
      role: "user",
      content: prompt,
    },
  ]
});

console.log(completion.usage.completion_tokens_details);
```

```python
from openai import OpenAI

client = OpenAI()

prompt = """
I want to build a Python app that takes user questions and looks 
them up in a database where they are mapped to answers. If there 
is close match, it retrieves the matched answer. If there isn't, 
it asks the user to provide an answer and stores the 
question/answer pair in the database. Make a plan for the directory 
structure you'll need, then return each file in full. Only supply 
your reasoning at the beginning and end, not throughout the code.
"""

response = client.chat.completions.create(
    model="o1-preview",
    messages=[
        {
            "role": "user",
            "content": [
                {
                    "type": "text",
                    "text": prompt
                },
            ],
        }
    ]
)

print(response.choices[0].message.content)
```

STEM Research

OpenAI o1 series models have shown excellent performance in STEM research. Prompts asking for support of basic research tasks should show strong results.

Ask o1 models questions related to basic scientific research

```javascript
import OpenAI from "openai";
const openai = new OpenAI();

const prompt = `
What are three compounds we should consider investigating to 
advance research into new antibiotics? Why should we consider 
them?
`;
 
const completion = await openai.chat.completions.create({
  model: "o1",
  messages: [
    {
      role: "user", 
      content: prompt,
    }
  ]
});

console.log(completion.choices[0].message.content);
```

```python
from openai import OpenAI
client = OpenAI()

prompt = """
What are three compounds we should consider investigating to 
advance research into new antibiotics? Why should we consider 
them?
"""

response = client.chat.completions.create(
    model="o1",
    messages=[
        {
            "role": "user", 
            "content": prompt
        }
    ]
)

print(response.choices[0].message.content)
```

Limitations
-----------

The o1 models are among the newest models from OpenAI, and as such have a few limitations to be aware of:

*   Current `o1` models
    *   Available to [Tier 5 customers only](/docs/guides/rate-limits#usage-tiers)
    *   Streaming support not available in the REST API
    *   Parallel tool calls not yet supported
    *   Not available in the Batch API
    *   Currently unsupported API parameters: `temperature`, `top_p`, `presence_penalty`, `frequency_penalty`, `logprobs`, `top_logprobs`, `logit_bias`

Use case examples
-----------------

Some examples of using o1 for real-world use cases can be found in [the cookbook](https://cookbook.openai.com).

[](https://cookbook.openai.com/examples/o1/using_reasoning_for_data_validation)

[](https://cookbook.openai.com/examples/o1/using_reasoning_for_data_validation)

[Using reasoning for data validation](https://cookbook.openai.com/examples/o1/using_reasoning_for_data_validation)

[](https://cookbook.openai.com/examples/o1/using_reasoning_for_data_validation)

[Evaluate a synthetic medical data set for discrepancies.](https://cookbook.openai.com/examples/o1/using_reasoning_for_data_validation)

[](https://cookbook.openai.com/examples/o1/using_reasoning_for_routine_generation)

[](https://cookbook.openai.com/examples/o1/using_reasoning_for_routine_generation)

[Using reasoning for routine generation](https://cookbook.openai.com/examples/o1/using_reasoning_for_routine_generation)

[](https://cookbook.openai.com/examples/o1/using_reasoning_for_routine_generation)

[Use help center articles to generate actions that an agent could perform.](https://cookbook.openai.com/examples/o1/using_reasoning_for_routine_generation)

===

Speech to text
==============

Learn how to turn audio into text.

Overview
--------

The Audio API provides two speech to text endpoints, `transcriptions` and `translations`, based on our state-of-the-art open source large-v2 [Whisper model](https://openai.com/blog/whisper/). They can be used to:

*   Transcribe audio into whatever language the audio is in.
*   Translate and transcribe the audio into english.

File uploads are currently limited to 25 MB and the following input file types are supported: `mp3`, `mp4`, `mpeg`, `mpga`, `m4a`, `wav`, and `webm`.

Quickstart
----------

### Transcriptions

The transcriptions API takes as input the audio file you want to transcribe and the desired output file format for the transcription of the audio. We currently support multiple input and output file formats.

Transcribe audio

```javascript
import fs from "fs";
import OpenAI from "openai";

const openai = new OpenAI();

const transcription = await openai.audio.transcriptions.create({
  file: fs.createReadStream("/path/to/file/audio.mp3"),
  model: "whisper-1",
});

console.log(transcription.text);
```

```python
from openai import OpenAI
client = OpenAI()

audio_file= open("/path/to/file/audio.mp3", "rb")
transcription = client.audio.transcriptions.create(
    model="whisper-1", 
    file=audio_file
)

print(transcription.text)
```

```bash
curl --request POST \
  --url https://api.openai.com/v1/audio/transcriptions \
  --header "Authorization: Bearer $OPENAI_API_KEY" \
  --header 'Content-Type: multipart/form-data' \
  --form file=@/path/to/file/audio.mp3 \
  --form model=whisper-1
```

By default, the response type will be json with the raw text included.

{ "text": "Imagine the wildest idea that you've ever had, and you're curious about how it might scale to something that's a 100, a 1,000 times bigger. .... }

The Audio API also allows you to set additional parameters in a request. For example, if you want to set the `response_format` as `text`, your request would look like the following:

Additional options

```javascript
import fs from "fs";
import OpenAI from "openai";

const openai = new OpenAI();

const transcription = await openai.audio.transcriptions.create({
  file: fs.createReadStream("/path/to/file/speech.mp3"),
  model: "whisper-1",
  response_format: "text",
});

console.log(transcription.text);
```

```python
from openai import OpenAI
client = OpenAI()

audio_file = open("/path/to/file/speech.mp3", "rb")
transcription = client.audio.transcriptions.create(
    model="whisper-1", 
    file=audio_file, 
    response_format="text"
)

print(transcription.text)
```

```bash
curl --request POST \
  --url https://api.openai.com/v1/audio/transcriptions \
  --header "Authorization: Bearer $OPENAI_API_KEY" \
  --header 'Content-Type: multipart/form-data' \
  --form file=@/path/to/file/speech.mp3 \
  --form model=whisper-1 \
  --form response_format=text
```

The [API Reference](/docs/api-reference/audio) includes the full list of available parameters.

### Translations

The translations API takes as input the audio file in any of the supported languages and transcribes, if necessary, the audio into English. This differs from our /Transcriptions endpoint since the output is not in the original input language and is instead translated to English text.

Translate audio

```javascript
import fs from "fs";
import OpenAI from "openai";

const openai = new OpenAI();

const transcription = await openai.audio.translations.create({
  file: fs.createReadStream("/path/to/file/german.mp3"),
  model: "whisper-1",
});

console.log(transcription.text);
```

```python
from openai import OpenAI
client = OpenAI()

audio_file = open("/path/to/file/german.mp3", "rb")
transcription = client.audio.translations.create(
    model="whisper-1", 
    file=audio_file,
)

print(transcription.text)
```

```bash
curl --request POST \
  --url https://api.openai.com/v1/audio/translations \
  --header "Authorization: Bearer $OPENAI_API_KEY" \
  --header 'Content-Type: multipart/form-data' \
  --form file=@/path/to/file/german.mp3 \
  --form model=whisper-1 \
```

In this case, the inputted audio was german and the outputted text looks like:

Hello, my name is Wolfgang and I come from Germany. Where are you heading today?

We only support translation into English at this time.

Supported languages
-------------------

We currently [support the following languages](https://github.com/openai/whisper#available-models-and-languages) through both the `transcriptions` and `translations` endpoint:

Afrikaans, Arabic, Armenian, Azerbaijani, Belarusian, Bosnian, Bulgarian, Catalan, Chinese, Croatian, Czech, Danish, Dutch, English, Estonian, Finnish, French, Galician, German, Greek, Hebrew, Hindi, Hungarian, Icelandic, Indonesian, Italian, Japanese, Kannada, Kazakh, Korean, Latvian, Lithuanian, Macedonian, Malay, Marathi, Maori, Nepali, Norwegian, Persian, Polish, Portuguese, Romanian, Russian, Serbian, Slovak, Slovenian, Spanish, Swahili, Swedish, Tagalog, Tamil, Thai, Turkish, Ukrainian, Urdu, Vietnamese, and Welsh.

While the underlying model was trained on 98 languages, we only list the languages that exceeded <50% [word error rate](https://en.wikipedia.org/wiki/Word_error_rate) (WER) which is an industry standard benchmark for speech to text model accuracy. The model will return results for languages not listed above but the quality will be low.

Timestamps
----------

By default, the Whisper API will output a transcript of the provided audio in text. The [`timestamp_granularities[]` parameter](/docs/api-reference/audio/createTranscription#audio-createtranscription-timestamp_granularities) enables a more structured and timestamped json output format, with timestamps at the segment, word level, or both. This enables word-level precision for transcripts and video edits, which allows for the removal of specific frames tied to individual words.

Timestamp options

```javascript
import fs from "fs";
import OpenAI from "openai";

const openai = new OpenAI();

const transcription = await openai.audio.transcriptions.create({
  file: fs.createReadStream("audio.mp3"),
  model: "whisper-1",
  response_format: "verbose_json",
  timestamp_granularities: ["word"]
});

console.log(transcription.words);
```

```python
from openai import OpenAI
client = OpenAI()

audio_file = open("/path/to/file/speech.mp3", "rb")
transcription = client.audio.transcriptions.create(
    file=audio_file,
    model="whisper-1",
    response_format="verbose_json",
    timestamp_granularities=["word"]
)

print(transcription.words)
```

```bash
curl https://api.openai.com/v1/audio/transcriptions \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: multipart/form-data" \
  -F file="@/path/to/file/audio.mp3" \
  -F "timestamp_granularities[]=word" \
  -F model="whisper-1" \
  -F response_format="verbose_json"
```

Longer inputs
-------------

By default, the Whisper API only supports files that are less than 25 MB. If you have an audio file that is longer than that, you will need to break it up into chunks of 25 MB's or less or used a compressed audio format. To get the best performance, we suggest that you avoid breaking the audio up mid-sentence as this may cause some context to be lost.

One way to handle this is to use the [PyDub open source Python package](https://github.com/jiaaro/pydub) to split the audio:

```python
from pydub import AudioSegment

song = AudioSegment.from_mp3("good_morning.mp3")

# PyDub handles time in milliseconds
ten_minutes = 10 * 60 * 1000

first_10_minutes = song[:ten_minutes]

first_10_minutes.export("good_morning_10.mp3", format="mp3")
```

_OpenAI makes no guarantees about the usability or security of 3rd party software like PyDub._

Prompting
---------

You can use a [prompt](/docs/api-reference/audio/createTranscription#audio/createTranscription-prompt) to improve the quality of the transcripts generated by the Whisper API. The model will try to match the style of the prompt, so it will be more likely to use capitalization and punctuation if the prompt does too. However, the current prompting system is much more limited than our other language models and only provides limited control over the generated audio. Here are some examples of how prompting can help in different scenarios:

1.  Prompts can be very helpful for correcting specific words or acronyms that the model may misrecognize in the audio. For example, the following prompt improves the transcription of the words DALL·E and GPT-3, which were previously written as "GDP 3" and "DALI": "The transcript is about OpenAI which makes technology like DALL·E, GPT-3, and ChatGPT with the hope of one day building an AGI system that benefits all of humanity"
2.  To preserve the context of a file that was split into segments, you can prompt the model with the transcript of the preceding segment. This will make the transcript more accurate, as the model will use the relevant information from the previous audio. The model will only consider the final 224 tokens of the prompt and ignore anything earlier. For multilingual inputs, Whisper uses a custom tokenizer. For English only inputs, it uses the standard GPT-2 tokenizer which are both accessible through the open source [Whisper Python package](https://github.com/openai/whisper/blob/main/whisper/tokenizer.py#L361).
3.  Sometimes the model might skip punctuation in the transcript. You can avoid this by using a simple prompt that includes punctuation: "Hello, welcome to my lecture."
4.  The model may also leave out common filler words in the audio. If you want to keep the filler words in your transcript, you can use a prompt that contains them: "Umm, let me think like, hmm... Okay, here's what I'm, like, thinking."
5.  Some languages can be written in different ways, such as simplified or traditional Chinese. The model might not always use the writing style that you want for your transcript by default. You can improve this by using a prompt in your preferred writing style.

Improving reliability
---------------------

As we explored in the prompting section, one of the most common challenges faced when using Whisper is the model often does not recognize uncommon words or acronyms. To address this, we have highlighted different techniques which improve the reliability of Whisper in these cases:

Using the prompt parameter

The first method involves using the optional prompt parameter to pass a dictionary of the correct spellings.

Since it wasn't trained using instruction-following techniques, Whisper operates more like a base GPT model. It's important to keep in mind that Whisper only considers the first 244 tokens of the prompt.

Prompt parameter

```javascript
import fs from "fs";
import OpenAI from "openai";

const openai = new OpenAI();

const transcription = await openai.audio.transcriptions.create({
  file: fs.createReadStream("/path/to/file/speech.mp3"),
  model: "whisper-1",
  response_format: "text",
  prompt:"ZyntriQix, Digique Plus, CynapseFive, VortiQore V8, EchoNix Array, OrbitalLink Seven, DigiFractal Matrix, PULSE, RAPT, B.R.I.C.K., Q.U.A.R.T.Z., F.L.I.N.T.",
});

console.log(transcription.text);
```

```python
from openai import OpenAI
client = OpenAI()

audio_file = open("/path/to/file/speech.mp3", "rb")
transcription = client.audio.transcriptions.create(
    model="whisper-1", 
    file=audio_file, 
    response_format="text",
    prompt="ZyntriQix, Digique Plus, CynapseFive, VortiQore V8, EchoNix Array, OrbitalLink Seven, DigiFractal Matrix, PULSE, RAPT, B.R.I.C.K., Q.U.A.R.T.Z., F.L.I.N.T."
)

print(transcription.text)
```

While it will increase reliability, this technique is limited to only 244 characters so your list of SKUs would need to be relatively small in order for this to be a scalable solution.

Post-processing with GPT-4

The second method involves a post-processing step using GPT-4 or GPT-3.5-Turbo.

We start by providing instructions for GPT-4 through the `system_prompt` variable. Similar to what we did with the prompt parameter earlier, we can define our company and product names.

Post-processing

```javascript
const systemPrompt = `
You are a helpful assistant for the company ZyntriQix. Your task is 
to correct any spelling discrepancies in the transcribed text. Make 
sure that the names of the following products are spelled correctly: 
ZyntriQix, Digique Plus, CynapseFive, VortiQore V8, EchoNix Array, 
OrbitalLink Seven, DigiFractal Matrix, PULSE, RAPT, B.R.I.C.K., 
Q.U.A.R.T.Z., F.L.I.N.T. Only add necessary punctuation such as 
periods, commas, and capitalization, and use only the context provided.
`;

const transcript = await transcribe(audioFile);
const completion = await openai.chat.completions.create({
  model: "gpt-4o",
  temperature: temperature,
  messages: [
    {
      role: "system",
      content: systemPrompt
    },
    {
      role: "user",
      content: transcript
    }
  ]
});

console.log(completion.choices[0].message.content);
```

```python
system_prompt = """
You are a helpful assistant for the company ZyntriQix. Your task is to correct 
any spelling discrepancies in the transcribed text. Make sure that the names of 
the following products are spelled correctly: ZyntriQix, Digique Plus, 
CynapseFive, VortiQore V8, EchoNix Array, OrbitalLink Seven, DigiFractal 
Matrix, PULSE, RAPT, B.R.I.C.K., Q.U.A.R.T.Z., F.L.I.N.T. Only add necessary 
punctuation such as periods, commas, and capitalization, and use only the 
context provided.
"""

def generate_corrected_transcript(temperature, system_prompt, audio_file):
    response = client.chat.completions.create(
        model="gpt-4o",
        temperature=temperature,
        messages=[
            {
                "role": "system",
                "content": system_prompt
            },
            {
                "role": "user",
                "content": transcribe(audio_file, "")
            }
        ]
    )
    return completion.choices[0].message.content
corrected_text = generate_corrected_transcript(
    0, system_prompt, fake_company_filepath
)
```

If you try this on your own audio file, you can see that GPT-4 manages to correct many misspellings in the transcript. Due to its larger context window, this method might be more scalable than using Whisper's prompt parameter and is more reliable since GPT-4 can be instructed and guided in ways that aren't possible with Whisper given the lack of instruction following.


Structured Outputs
==================

Ensure responses adhere to a JSON schema.

Try it out
----------

Try it out in the [Playground](/playground) or generate a ready-to-use schema definition to experiment with structured outputs.

Generate

Introduction
------------

JSON is one of the most widely used formats in the world for applications to exchange data.

Structured Outputs is a feature that ensures the model will always generate responses that adhere to your supplied [JSON Schema](https://json-schema.org/overview/what-is-jsonschema), so you don't need to worry about the model omitting a required key, or hallucinating an invalid enum value.

Some benefits of Structured Outputs include:

1.  **Reliable type-safety:** No need to validate or retry incorrectly formatted responses
2.  **Explicit refusals:** Safety-based model refusals are now programmatically detectable
3.  **Simpler prompting:** No need for strongly worded prompts to achieve consistent formatting

In addition to supporting JSON Schema in the REST API, the OpenAI SDKs for [Python](https://github.com/openai/openai-python/blob/main/helpers.md#structured-outputs-parsing-helpers) and [JavaScript](https://github.com/openai/openai-node/blob/master/helpers.md#structured-outputs-parsing-helpers) also make it easy to define object schemas using [Pydantic](https://docs.pydantic.dev/latest/) and [Zod](https://zod.dev/) respectively. Below, you can see how to extract information from unstructured text that conforms to a schema defined in code.

Getting a structured response

```javascript
import OpenAI from "openai";
import { zodResponseFormat } from "openai/helpers/zod";
import { z } from "zod";

const openai = new OpenAI();

const CalendarEvent = z.object({
  name: z.string(),
  date: z.string(),
  participants: z.array(z.string()),
});

const completion = await openai.beta.chat.completions.parse({
  model: "gpt-4o-2024-08-06",
  messages: [
    { role: "system", content: "Extract the event information." },
    { role: "user", content: "Alice and Bob are going to a science fair on Friday." },
  ],
  response_format: zodResponseFormat(CalendarEvent, "event"),
});

const event = completion.choices[0].message.parsed;
```

```python
from pydantic import BaseModel
from openai import OpenAI

client = OpenAI()

class CalendarEvent(BaseModel):
    name: str
    date: str
    participants: list[str]

completion = client.beta.chat.completions.parse(
    model="gpt-4o-2024-08-06",
    messages=[
        {"role": "system", "content": "Extract the event information."},
        {"role": "user", "content": "Alice and Bob are going to a science fair on Friday."},
    ],
    response_format=CalendarEvent,
)

event = completion.choices[0].message.parsed
```

### Supported models

Structured Outputs are available in our [latest large language models](/docs/models), starting with GPT-4o:

*   `o1-2024-12-17` and later
*   `gpt-4o-mini-2024-07-18` and later
*   `gpt-4o-2024-08-06` and later

Older models like `gpt-4-turbo` and earlier may use [JSON mode](#json-mode) instead.

When to use Structured Outputs via function calling vs via response\_format

-------------------------------------------------------------------------------

Structured Outputs is available in two forms in the OpenAI API:

1.  When using [function calling](/docs/guides/function-calling)
2.  When using a `json_schema` response format

Function calling is useful when you are building an application that bridges the models and functionality of your application.

For example, you can give the model access to functions that query a database in order to build an AI assistant that can help users with their orders, or functions that can interact with the UI.

Conversely, Structured Outputs via `response_format` are more suitable when you want to indicate a structured schema for use when the model responds to the user, rather than when the model calls a tool.

For example, if you are building a math tutoring application, you might want the assistant to respond to your user using a specific JSON Schema so that you can generate a UI that displays different parts of the model's output in distinct ways.

Put simply:

*   If you are connecting the model to tools, functions, data, etc. in your system, then you should use function calling
*   If you want to structure the model's output when it responds to the user, then you should use a structured `response_format`

The remainder of this guide will focus on non-function calling use cases in the Chat Completions API. To learn more about how to use Structured Outputs with function calling, check out the [Function Calling](/docs/guides/function-calling#function-calling-with-structured-outputs) guide.

### Structured Outputs vs JSON mode

Structured Outputs is the evolution of [JSON mode](#json-mode). While both ensure valid JSON is produced, only Structured Outputs ensure schema adherance. Both Structured Outputs and JSON mode are supported in the Chat Completions API, Assistants API, Fine-tuning API and Batch API.

We recommend always using Structured Outputs instead of JSON mode when possible.

However, Structured Outputs with `response_format: {type: "json_schema", ...}` is only supported with the `gpt-4o-mini`, `gpt-4o-mini-2024-07-18`, and `gpt-4o-2024-08-06` model snapshots and later.

||Structured Outputs|JSON Mode|
|---|---|---|
|Outputs valid JSON|Yes|Yes|
|Adheres to schema|Yes (see supported schemas)|No|
|Compatible models|gpt-4o-mini, gpt-4o-2024-08-06, and later|gpt-3.5-turbo, gpt-4-* and gpt-4o-* models|
|Enabling|response_format: { type: "json_schema", json_schema: {"strict": true, "schema": ...} }|response_format: { type: "json_object" }|

Examples
--------

Chain of thoughtStructured data extractionUI generationModeration

### Chain of thought

You can ask the model to output an answer in a structured, step-by-step way, to guide the user through the solution.

Structured Outputs for chain-of-thought math tutoring

```javascript
import OpenAI from "openai";
import { z } from "zod";
import { zodResponseFormat } from "openai/helpers/zod";

const openai = new OpenAI();

const Step = z.object({
  explanation: z.string(),
  output: z.string(),
});

const MathReasoning = z.object({
  steps: z.array(Step),
  final_answer: z.string(),
});

const completion = await openai.beta.chat.completions.parse({
  model: "gpt-4o-2024-08-06",
  messages: [
    { role: "system", content: "You are a helpful math tutor. Guide the user through the solution step by step." },
    { role: "user", content: "how can I solve 8x + 7 = -23" },
  ],
  response_format: zodResponseFormat(MathReasoning, "math_reasoning"),
});

const math_reasoning = completion.choices[0].message.parsed;
```

```python
from pydantic import BaseModel
from openai import OpenAI

client = OpenAI()

class Step(BaseModel):
    explanation: str
    output: str

class MathReasoning(BaseModel):
    steps: list[Step]
    final_answer: str

completion = client.beta.chat.completions.parse(
    model="gpt-4o-2024-08-06",
    messages=[
        {"role": "system", "content": "You are a helpful math tutor. Guide the user through the solution step by step."},
        {"role": "user", "content": "how can I solve 8x + 7 = -23"}
    ],
    response_format=MathReasoning,
)

math_reasoning = completion.choices[0].message.parsed
```

```bash
curl https://api.openai.com/v1/chat/completions \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o-2024-08-06",
    "messages": [
      {
        "role": "system",
        "content": "You are a helpful math tutor. Guide the user through the solution step by step."
      },
      {
        "role": "user",
        "content": "how can I solve 8x + 7 = -23"
      }
    ],
    "response_format": {
      "type": "json_schema",
      "json_schema": {
        "name": "math_reasoning",
        "schema": {
          "type": "object",
          "properties": {
            "steps": {
              "type": "array",
              "items": {
                "type": "object",
                "properties": {
                  "explanation": { "type": "string" },
                  "output": { "type": "string" }
                },
                "required": ["explanation", "output"],
                "additionalProperties": false
              }
            },
            "final_answer": { "type": "string" }
          },
          "required": ["steps", "final_answer"],
          "additionalProperties": false
        },
        "strict": true
      }
    }
  }'
```

#### Example response

```json
{
  "steps": [
    {
      "explanation": "Start with the equation 8x + 7 = -23.",
      "output": "8x + 7 = -23"
    },
    {
      "explanation": "Subtract 7 from both sides to isolate the term with the variable.",
      "output": "8x = -23 - 7"
    },
    {
      "explanation": "Simplify the right side of the equation.",
      "output": "8x = -30"
    },
    {
      "explanation": "Divide both sides by 8 to solve for x.",
      "output": "x = -30 / 8"
    },
    {
      "explanation": "Simplify the fraction.",
      "output": "x = -15 / 4"
    }
  ],
  "final_answer": "x = -15 / 4"
}
```

How to use Structured Outputs with response\_format

-------------------------------------------------------

You can use Structured Outputs with the new SDK helper to parse the model's output into your desired format, or you can specify the JSON schema directly.

**Note:** the first request you make with any schema will have additional latency as our API processes the schema, but subsequent requests with the same schema will not have additional latency.

SDK objectsManual schema

Step 1: Define your object

First you must define an object or data structure to represent the JSON Schema that the model should be constrained to follow. See the [examples](/docs/guides/structured-outputs#examples) at the top of this guide for reference.

While Structured Outputs supports much of JSON Schema, some features are unavailable either for performance or technical reasons. See [here](/docs/guides/structured-outputs#supported-schemas) for more details.

For example, you can define an object like this:

```python
from pydantic import BaseModel

class Step(BaseModel):
  explanation: str
  output: str

class MathResponse(BaseModel):
  steps: list[Step]
  final_answer: str
```

```javascript
import { z } from "zod";
import { zodResponseFormat } from "openai/helpers/zod";

const Step = z.object({
explanation: z.string(),
output: z.string(),
});

const MathResponse = z.object({
steps: z.array(Step),
final_answer: z.string(),
});
```

#### Tips for your data structure

To maximize the quality of model generations, we recommend the following:

*   Name keys clearly and intuitively
*   Create clear titles and descriptions for important keys in your structure
*   Create and use evals to determine the structure that works best for your use case

Step 2: Supply your object in the API call

You can use the `parse` method to automatically parse the JSON response into the object you defined.

Under the hood, the SDK takes care of supplying the JSON schema corresponding to your data structure, and then parsing the response as an object.

```python
completion = client.beta.chat.completions.parse(
  model="gpt-4o-2024-08-06",
  messages=[
      {"role": "system", "content": "You are a helpful math tutor. Guide the user through the solution step by step."},
      {"role": "user", "content": "how can I solve 8x + 7 = -23"}
  ],
  response_format=MathResponse
)
```

```javascript
const completion = await openai.beta.chat.completions.parse({
model: "gpt-4o-2024-08-06",
messages: [
  { role: "system", content: "You are a helpful math tutor. Guide the user through the solution step by step." },
  { role: "user", content: "how can I solve 8x + 7 = -23" },
],
response_format: zodResponseFormat(MathResponse, "math_response"),
});
```

Step 3: Handle edge cases

In some cases, the model might not generate a valid response that matches the provided JSON schema.

This can happen in the case of a refusal, if the model refuses to answer for safety reasons, or if for example you reach a max tokens limit and the response is incomplete.

```python
try:
  completion = client.beta.chat.completions.parse(
      model="gpt-4o-2024-08-06",
      messages=[
          {"role": "system", "content": "You are a helpful math tutor. Guide the user through the solution step by step."},
          {"role": "user", "content": "how can I solve 8x + 7 = -23"}
      ],
      response_format=MathResponse,
      max_tokens=50
  )
  math_response = completion.choices[0].message
  if math_response.parsed:
      print(math_response.parsed)
  elif math_response.refusal:
      # handle refusal
      print(math_response.refusal)
except Exception as e:
  # Handle edge cases
  if type(e) == openai.LengthFinishReasonError:
      # Retry with a higher max tokens
      print("Too many tokens: ", e)
      pass
  else:
      # Handle other exceptions
      print(e)
      pass
```

```javascript
try {
const completion = await openai.beta.chat.completions.parse({
  model: "gpt-4o-2024-08-06",
  messages: [
    {
      role: "system",
      content:
        "You are a helpful math tutor. Guide the user through the solution step by step.",
    },
    { role: "user", content: "how can I solve 8x + 7 = -23" },
  ],
  response_format: zodResponseFormat(MathResponse, "math_response"),
  max_tokens: 50,
});

const math_response = completion.choices[0].message;
console.log(math_response);
if (math_response.parsed) {
  console.log(math_response.parsed);
} else if (math_response.refusal) {
  // handle refusal
  console.log(math_response.refusal);
}
} catch (e) {
// Handle edge cases
if (e.constructor.name == "LengthFinishReasonError") {
  // Retry with a higher max tokens
  console.log("Too many tokens: ", e.message);
} else {
  // Handle other exceptions
  console.log("An error occurred: ", e.message);
}
}
```

Step 4: Use the generated structured data in a type-safe way

When using the SDK, you can use the `parsed` attribute to access the parsed JSON response as an object. This object will be of the type you defined in the `response_format` parameter.

```python
math_response = completion.choices[0].message.parsed
print(math_response.steps)
print(math_response.final_answer)
```

```javascript
const math_response = completion.choices[0].message.parsed;
console.log(math_response.steps);
console.log(math_response.final_answer);
```

### 

Refusals with Structured Outputs

When using Structured Outputs with user-generated input, OpenAI models may occasionally refuse to fulfill the request for safety reasons. Since a refusal does not necessarily follow the schema you have supplied in `response_format`, the API response will include a new field called `refusal` to indicate that the model refused to fulfill the request.

When the `refusal` property appears in your output object, you might present the refusal in your UI, or include conditional logic in code that consumes the response to handle the case of a refused request.

```python
class Step(BaseModel):
  explanation: str
  output: str

class MathReasoning(BaseModel):
  steps: list[Step]
  final_answer: str

completion = client.beta.chat.completions.parse(
  model="gpt-4o-2024-08-06",
  messages=[
      {"role": "system", "content": "You are a helpful math tutor. Guide the user through the solution step by step."},
      {"role": "user", "content": "how can I solve 8x + 7 = -23"}
  ],
  response_format=MathReasoning,
)

math_reasoning = completion.choices[0].message

# If the model refuses to respond, you will get a refusal message
if (math_reasoning.refusal):
  print(math_reasoning.refusal)
else:
  print(math_reasoning.parsed)
```

```javascript
const Step = z.object({
explanation: z.string(),
output: z.string(),
});

const MathReasoning = z.object({
steps: z.array(Step),
final_answer: z.string(),
});

const completion = await openai.beta.chat.completions.parse({
model: "gpt-4o-2024-08-06",
messages: [
  { role: "system", content: "You are a helpful math tutor. Guide the user through the solution step by step." },
  { role: "user", content: "how can I solve 8x + 7 = -23" },
],
response_format: zodResponseFormat(MathReasoning, "math_reasoning"),
});

const math_reasoning = completion.choices[0].message

// If the model refuses to respond, you will get a refusal message
if (math_reasoning.refusal) {
console.log(math_reasoning.refusal);
} else {
console.log(math_reasoning.parsed);
}
```

The API response from a refusal will look something like this:

```json
{
"id": "chatcmpl-9nYAG9LPNonX8DAyrkwYfemr3C8HC",
"object": "chat.completion",
"created": 1721596428,
"model": "gpt-4o-2024-08-06",
"choices": [
  {
  "index": 0,
  "message": {
          "role": "assistant",
          "refusal": "I'm sorry, I cannot assist with that request."
  },
  "logprobs": null,
  "finish_reason": "stop"
}
],
"usage": {
    "prompt_tokens": 81,
    "completion_tokens": 11,
    "total_tokens": 92,
    "completion_tokens_details": {
      "reasoning_tokens": 0,
      "accepted_prediction_tokens": 0,
      "rejected_prediction_tokens": 0
    }
},
"system_fingerprint": "fp_3407719c7f"
}
```

### 

Tips and best practices

#### Handling user-generated input

If your application is using user-generated input, make sure your prompt includes instructions on how to handle situations where the input cannot result in a valid response.

The model will always try to adhere to the provided schema, which can result in hallucinations if the input is completely unrelated to the schema.

You could include language in your prompt to specify that you want to return empty parameters, or a specific sentence, if the model detects that the input is incompatible with the task.

#### Handling mistakes

Structured Outputs can still contain mistakes. If you see mistakes, try adjusting your instructions, providing examples in the system instructions, or splitting tasks into simpler subtasks. Refer to the [prompt engineering guide](/docs/guides/prompt-engineering) for more guidance on how to tweak your inputs.

#### Avoid JSON schema divergence

To prevent your JSON Schema and corresponding types in your programming language from diverging, we strongly recommend using the native Pydantic/zod sdk support.

If you prefer to specify the JSON schema directly, you could add CI rules that flag when either the JSON schema or underlying data objects are edited, or add a CI step that auto-generates the JSON Schema from type definitions (or vice-versa).

Streaming
---------

You can use streaming to process model responses or function call arguments as they are being generated, and parse them as structured data.

That way, you don't have to wait for the entire response to complete before handling it. This is particularly useful if you would like to display JSON fields one by one, or handle function call arguments as soon as they are available.

We recommend relying on the SDKs to handle streaming with Structured Outputs. You can find an example of how to stream function call arguments without the SDK `stream` helper in the [function calling guide](/docs/guides/function-calling#advanced-usage).

Here is how you can stream a model response with the `stream` helper:

```python
from typing import List
from pydantic import BaseModel
from openai import OpenAI

class EntitiesModel(BaseModel):
  attributes: List[str]
  colors: List[str]
  animals: List[str]

client = OpenAI()

with client.beta.chat.completions.stream(
  model="gpt-4o",
  messages=[
      {"role": "system", "content": "Extract entities from the input text"},
      {
          "role": "user",
          "content": "The quick brown fox jumps over the lazy dog with piercing blue eyes",
      },
  ],
  response_format=EntitiesModel,
) as stream:
  for event in stream:
      if event.type == "content.delta":
          if event.parsed is not None:
              # Print the parsed data as JSON
              print("content.delta parsed:", event.parsed)
      elif event.type == "content.done":
          print("content.done")
      elif event.type == "error":
          print("Error in stream:", event.error)

final_completion = stream.get_final_completion()
print("Final completion:", final_completion)
```

```js
import OpenAI from "openai";
import { zodResponseFormat } from "openai/helpers/zod";
import { z } from "zod";
export const openai = new OpenAI();

const EntitiesSchema = z.object({
attributes: z.array(z.string()),
colors: z.array(z.string()),
animals: z.array(z.string()),
});

const stream = openai.beta.chat.completions
.stream({
  model: "gpt-4o",
  messages: [
    { role: "system", content: "Extract entities from the input text" },
    {
      role: "user",
      content:
        "The quick brown fox jumps over the lazy dog with piercing blue eyes",
    },
  ],
  response_format: zodResponseFormat(EntitiesSchema, "entities"),
})
.on("refusal.done", () => console.log("request refused"))
.on("content.delta", ({ snapshot, parsed }) => {
  console.log("content:", snapshot);
  console.log("parsed:", parsed);
  console.log();
})
.on("content.done", (props) => {
  console.log(props);
});

await stream.done();

const finalCompletion = await stream.finalChatCompletion();

console.log(finalCompletion);
```

You can also use the `stream` helper to parse function call arguments:

```python
from pydantic import BaseModel
import openai
from openai import OpenAI

class GetWeather(BaseModel):
  city: str
  country: str

client = OpenAI()

with client.beta.chat.completions.stream(
  model="gpt-4o",
  messages=[
      {
          "role": "user",
          "content": "What's the weather like in SF and London?",
      },
  ],
  tools=[
      openai.pydantic_function_tool(GetWeather, name="get_weather"),
  ],
  parallel_tool_calls=True,
) as stream:
  for event in stream:
      if event.type == "tool_calls.function.arguments.delta" or event.type == "tool_calls.function.arguments.done":
          print(event)

print(stream.get_final_completion())
```

```js
import { zodFunction } from "openai/helpers/zod";
import OpenAI from "openai/index";
import { z } from "zod";

const GetWeatherArgs = z.object({
city: z.string(),
country: z.string(),
});

const client = new OpenAI();

const stream = client.beta.chat.completions
.stream({
  model: "gpt-4o",
  messages: [
    {
      role: "user",
      content: "What's the weather like in SF and London?",
    },
  ],
  tools: [zodFunction({ name: "get_weather", parameters: GetWeatherArgs })],
})
.on("tool_calls.function.arguments.delta", (props) =>
  console.log("tool_calls.function.arguments.delta", props)
)
.on("tool_calls.function.arguments.done", (props) =>
  console.log("tool_calls.function.arguments.done", props)
)
.on("refusal.delta", ({ delta }) => {
  process.stdout.write(delta);
})
.on("refusal.done", () => console.log("request refused"));

const completion = await stream.finalChatCompletion();

console.log("final completion:", completion);
```

Supported schemas
-----------------

Structured Outputs supports a subset of the [JSON Schema](https://json-schema.org/docs) language.

#### Supported types

The following types are supported for Structured Outputs:

*   String
*   Number
*   Boolean
*   Integer
*   Object
*   Array
*   Enum
*   anyOf

#### Root objects must not be `anyOf`

Note that the root level object of a schema must be an object, and not use `anyOf`. A pattern that appears in Zod (as one example) is using a discriminated union, which produces an `anyOf` at the top level. So code such as the following won't work:

```javascript
import { z } from 'zod';
import { zodResponseFormat } from 'openai/helpers/zod';

const BaseResponseSchema = z.object({ /* ... */ });
const UnsuccessfulResponseSchema = z.object({ /* ... */ });

const finalSchema = z.discriminatedUnion('status', [
  BaseResponseSchema,
  UnsuccessfulResponseSchema,
]);

// Invalid JSON Schema for Structured Outputs
const json = zodResponseFormat(finalSchema, 'final_schema');
```

#### All fields must be `required`

To use Structured Outputs, all fields or function parameters must be specified as `required`.

```json
{
  "name": "get_weather",
  "description": "Fetches the weather in the given location",
  "strict": true,
  "parameters": {
      "type": "object",
      "properties": {
          "location": {
              "type": "string",
              "description": "The location to get the weather for"
          },
          "unit": {
              "type": "string",
              "description": "The unit to return the temperature in",
              "enum": ["F", "C"]
          }
      },
      "additionalProperties": false,
      "required": ["location", "unit"]
  }
}
```

Although all fields must be required (and the model will return a value for each parameter), it is possible to emulate an optional parameter by using a union type with `null`.

```json
{
  "name": "get_weather",
  "description": "Fetches the weather in the given location",
  "strict": true,
  "parameters": {
      "type": "object",
      "properties": {
          "location": {
              "type": "string",
              "description": "The location to get the weather for"
          },
          "unit": {
              "type": ["string", "null"],
              "description": "The unit to return the temperature in",
              "enum": ["F", "C"]
          }
      },
      "additionalProperties": false,
      "required": [
          "location", "unit"
      ]
  }
}
```

#### Objects have limitations on nesting depth and size

A schema may have up to 100 object properties total, with up to 5 levels of nesting.

#### Limitations on total string size

In a schema, total string length of all property names, definition names, enum values, and const values cannot exceed 15,000 characters.

#### Limitations on enum size

A schema may have up to 500 enum values across all enum properties.

For a single enum property with string values, the total string length of all enum values cannot exceed 7,500 characters when there are more than 250 enum values.

#### `additionalProperties: false` must always be set in objects

`additionalProperties` controls whether it is allowable for an object to contain additional keys / values that were not defined in the JSON Schema.

Structured Outputs only supports generating specified keys / values, so we require developers to set `additionalProperties: false` to opt into Structured Outputs.

```json
{
  "name": "get_weather",
  "description": "Fetches the weather in the given location",
  "strict": true,
  "schema": {
      "type": "object",
      "properties": {
          "location": {
              "type": "string",
              "description": "The location to get the weather for"
          },
          "unit": {
              "type": "string",
              "description": "The unit to return the temperature in",
              "enum": ["F", "C"]
          }
      },
      "additionalProperties": false,
      "required": [
          "location", "unit"
      ]
  }
}
```

#### Key ordering

When using Structured Outputs, outputs will be produced in the same order as the ordering of keys in the schema.

#### Some type-specific keywords are not yet supported

Notable keywords not supported include:

*   **For strings:** `minLength`, `maxLength`, `pattern`, `format`
*   **For numbers:** `minimum`, `maximum`, `multipleOf`
*   **For objects:** `patternProperties`, `unevaluatedProperties`, `propertyNames`, `minProperties`, `maxProperties`
*   **For arrays:** `unevaluatedItems`, `contains`, `minContains`, `maxContains`, `minItems`, `maxItems`, `uniqueItems`

If you turn on Structured Outputs by supplying `strict: true` and call the API with an unsupported JSON Schema, you will receive an error.

#### For `anyOf`, the nested schemas must each be a valid JSON Schema per this subset

Here's an example supported anyOf schema:

```json
{
  "type": "object",
  "properties": {
      "item": {
          "anyOf": [
              {
                  "type": "object",
                  "description": "The user object to insert into the database",
                  "properties": {
                      "name": {
                          "type": "string",
                          "description": "The name of the user"
                      },
                      "age": {
                          "type": "number",
                          "description": "The age of the user"
                      }
                  },
                  "additionalProperties": false,
                  "required": [
                      "name",
                      "age"
                  ]
              },
              {
                  "type": "object",
                  "description": "The address object to insert into the database",
                  "properties": {
                      "number": {
                          "type": "string",
                          "description": "The number of the address. Eg. for 123 main st, this would be 123"
                      },
                      "street": {
                          "type": "string",
                          "description": "The street name. Eg. for 123 main st, this would be main st"
                      },
                      "city": {
                          "type": "string",
                          "description": "The city of the address"
                      }
                  },
                  "additionalProperties": false,
                  "required": [
                      "number",
                      "street",
                      "city"
                  ]
              }
          ]
      }
  },
  "additionalProperties": false,
  "required": [
      "item"
  ]
}
```

#### Definitions are supported

You can use definitions to define subschemas which are referenced throughout your schema. The following is a simple example.

```json
{
  "type": "object",
  "properties": {
      "steps": {
          "type": "array",
          "items": {
              "$ref": "#/$defs/step"
          }
      },
      "final_answer": {
          "type": "string"
      }
  },
  "$defs": {
      "step": {
          "type": "object",
          "properties": {
              "explanation": {
                  "type": "string"
              },
              "output": {
                  "type": "string"
              }
          },
          "required": [
              "explanation",
              "output"
          ],
          "additionalProperties": false
      }
  },
  "required": [
      "steps",
      "final_answer"
  ],
  "additionalProperties": false
}
```

#### Recursive schemas are supported

Sample recursive schema using `#` to indicate root recursion.

```json
{
  "name": "ui",
  "description": "Dynamically generated UI",
  "strict": true,
  "schema": {
      "type": "object",
      "properties": {
          "type": {
              "type": "string",
              "description": "The type of the UI component",
              "enum": ["div", "button", "header", "section", "field", "form"]
          },
          "label": {
              "type": "string",
              "description": "The label of the UI component, used for buttons or form fields"
          },
          "children": {
              "type": "array",
              "description": "Nested UI components",
              "items": {
                  "$ref": "#"
              }
          },
          "attributes": {
              "type": "array",
              "description": "Arbitrary attributes for the UI component, suitable for any element",
              "items": {
                  "type": "object",
                  "properties": {
                      "name": {
                          "type": "string",
                          "description": "The name of the attribute, for example onClick or className"
                      },
                      "value": {
                          "type": "string",
                          "description": "The value of the attribute"
                      }
                  },
                  "additionalProperties": false,
                  "required": ["name", "value"]
              }
          }
      },
      "required": ["type", "label", "children", "attributes"],
      "additionalProperties": false
  }
}
```

Sample recursive schema using explicit recursion:

```json
{
  "type": "object",
  "properties": {
      "linked_list": {
          "$ref": "#/$defs/linked_list_node"
      }
  },
  "$defs": {
      "linked_list_node": {
          "type": "object",
          "properties": {
              "value": {
                  "type": "number"
              },
              "next": {
                  "anyOf": [
                      {
                          "$ref": "#/$defs/linked_list_node"
                      },
                      {
                          "type": "null"
                      }
                  ]
              }
          },
          "additionalProperties": false,
          "required": [
              "next",
              "value"
          ]
      }
  },
  "additionalProperties": false,
  "required": [
      "linked_list"
  ]
}
```

JSON mode
---------

JSON mode is a more basic version of the Structured Outputs feature. While JSON mode ensures that model output is valid JSON, Structured Outputs reliably matches the model's output to the schema you specify. We recommend you use Structured Outputs if it is supported for your use case.

When JSON mode is turned on, the model's output is ensured to be valid JSON, except for in some edge cases that you should detect and handle appropriately.

To turn on JSON mode with the Chat Completions or Assistants API you can set the `response_format` to `{ "type": "json_object" }`. If you are using function calling, JSON mode is always turned on.

Important notes:

*   When using JSON mode, you must always instruct the model to produce JSON via some message in the conversation, for example via your system message. If you don't include an explicit instruction to generate JSON, the model may generate an unending stream of whitespace and the request may run continually until it reaches the token limit. To help ensure you don't forget, the API will throw an error if the string "JSON" does not appear somewhere in the context.
*   JSON mode will not guarantee the output matches any specific schema, only that it is valid and parses without errors. You should use Structured Outputs to ensure it matches your schema, or if that is not possible, you should use a validation library and potentially retries to ensure that the output matches your desired schema.
*   Your application must detect and handle the edge cases that can result in the model output not being a complete JSON object (see below)

Handling edge cases

```javascript
const we_did_not_specify_stop_tokens = true;

try {
  const response = await openai.chat.completions.create({
    model: "gpt-3.5-turbo-0125",
    messages: [
      {
        role: "system",
        content: "You are a helpful assistant designed to output JSON.",
      },
      { role: "user", content: "Who won the world series in 2020? Please respond in the format {winner: ...}" },
    ],
    response_format: { type: "json_object" },
  });

  // Check if the conversation was too long for the context window, resulting in incomplete JSON 
  if (response.choices[0].message.finish_reason === "length") {
    // your code should handle this error case
  }

  // Check if the OpenAI safety system refused the request and generated a refusal instead
  if (response.choices[0].message[0].refusal) {
    // your code should handle this error case
    // In this case, the .content field will contain the explanation (if any) that the model generated for why it is refusing
    console.log(response.choices[0].message[0].refusal)
  }

  // Check if the model's output included restricted content, so the generation of JSON was halted and may be partial
  if (response.choices[0].message.finish_reason === "content_filter") {
    // your code should handle this error case
  }

  if (response.choices[0].message.finish_reason === "stop") {
    // In this case the model has either successfully finished generating the JSON object according to your schema, or the model generated one of the tokens you provided as a "stop token"

    if (we_did_not_specify_stop_tokens) {
      // If you didn't specify any stop tokens, then the generation is complete and the content key will contain the serialized JSON object
      // This will parse successfully and should now contain  {"winner": "Los Angeles Dodgers"}
      console.log(JSON.parse(response.choices[0].message.content))
    } else {
      // Check if the response.choices[0].message.content ends with one of your stop tokens and handle appropriately
    }
  }
} catch (e) {
  // Your code should handle errors here, for example a network error calling the API
  console.error(e)
}
```

```python
we_did_not_specify_stop_tokens = True

try:
    response = client.chat.completions.create(
        model="gpt-3.5-turbo-0125",
        messages=[
            {"role": "system", "content": "You are a helpful assistant designed to output JSON."},
            {"role": "user", "content": "Who won the world series in 2020? Please respond in the format {winner: ...}"}
        ],
        response_format={"type": "json_object"}
    )

    # Check if the conversation was too long for the context window, resulting in incomplete JSON 
    if response.choices[0].message.finish_reason == "length":
        # your code should handle this error case
        pass

    # Check if the OpenAI safety system refused the request and generated a refusal instead
    if response.choices[0].message[0].get("refusal"):
        # your code should handle this error case
        # In this case, the .content field will contain the explanation (if any) that the model generated for why it is refusing
        print(response.choices[0].message[0]["refusal"])

    # Check if the model's output included restricted content, so the generation of JSON was halted and may be partial
    if response.choices[0].message.finish_reason == "content_filter":
        # your code should handle this error case
        pass

    if response.choices[0].message.finish_reason == "stop":
        # In this case the model has either successfully finished generating the JSON object according to your schema, or the model generated one of the tokens you provided as a "stop token"

        if we_did_not_specify_stop_tokens:
            # If you didn't specify any stop tokens, then the generation is complete and the content key will contain the serialized JSON object
            # This will parse successfully and should now contain  "{"winner": "Los Angeles Dodgers"}"
            print(response.choices[0].message.content)
        else:
            # Check if the response.choices[0].message.content ends with one of your stop tokens and handle appropriately
            pass
except Exception as e:
    # Your code should handle errors here, for example a network error calling the API
    print(e)
```

Resources
---------

To learn more about Structured Outputs, we recommend browsing the following resources:

*   Check out our [introductory cookbook](https://cookbook.openai.com/examples/structured_outputs_intro) on Structured Outputs
*   Learn [how to build multi-agent systems](https://cookbook.openai.com/examples/structured_outputs_multi_agent) with Structured Outputs

===

Text generation
===============

Learn how to generate text from a prompt.

OpenAI provides simple APIs to use a [large language model](/docs/models) to generate text from a prompt, as you might using [ChatGPT](https://chatgpt.com). These models have been trained on vast quantities of data to understand multimedia inputs and natural language instructions. From these [prompts](/docs/guides/prompt-engineering), models can generate almost any kind of text response, like code, mathematical equations, structured JSON data, or human-like prose.

Quickstart
----------

To generate text, you can use the [chat completions endpoint](/docs/api-reference/chat/) in the REST API, as seen in the examples below. You can either use the [REST API](/docs/api-reference) from the HTTP client of your choice, or use one of OpenAI's [official SDKs](/docs/libraries) for your preferred programming language.

Generate prose

Create a human-like response to a prompt

```javascript
import OpenAI from "openai";
const openai = new OpenAI();

const completion = await openai.chat.completions.create({
    model: "gpt-4o",
    messages: [
        { role: "developer", content: "You are a helpful assistant." },
        {
            role: "user",
            content: "Write a haiku about recursion in programming.",
        },
    ],
});

console.log(completion.choices[0].message);
```

```python
from openai import OpenAI
client = OpenAI()

completion = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "developer", "content": "You are a helpful assistant."},
        {
            "role": "user",
            "content": "Write a haiku about recursion in programming."
        }
    ]
)

print(completion.choices[0].message)
```

```bash
curl "https://api.openai.com/v1/chat/completions" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $OPENAI_API_KEY" \
    -d '{
        "model": "gpt-4o",
        "messages": [
            {
                "role": "developer",
                "content": "You are a helpful assistant."
            },
            {
                "role": "user",
                "content": "Write a haiku about recursion in programming."
            }
        ]
    }'
```

Analyze an image

Describe the contents of an image

```javascript
import OpenAI from "openai";
const openai = new OpenAI();

const completion = await openai.chat.completions.create({
    model: "gpt-4o",
    messages: [
        {
            role: "user",
            content: [
                { type: "text", text: "What's in this image?" },
                {
                    type: "image_url",
                    image_url: {
                        "url": "https://upload.wikimedia.org/wikipedia/commons/thumb/d/dd/Gfp-wisconsin-madison-the-nature-boardwalk.jpg/2560px-Gfp-wisconsin-madison-the-nature-boardwalk.jpg",
                    },
                }
            ],
        },
    ],
});

console.log(completion.choices[0].message);
```

```python
from openai import OpenAI

client = OpenAI()

completion = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {
            "role": "user",
            "content": [
                {"type": "text", "text": "What's in this image?"},
                {
                    "type": "image_url",
                    "image_url": {
                        "url": "https://upload.wikimedia.org/wikipedia/commons/thumb/d/dd/Gfp-wisconsin-madison-the-nature-boardwalk.jpg/2560px-Gfp-wisconsin-madison-the-nature-boardwalk.jpg",
                    }
                },
            ],
        }
    ],
)

print(completion.choices[0].message)
```

```bash
curl https://api.openai.com/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -d '{
    "model": "gpt-4o",
    "messages": [
      {
        "role": "user",
        "content": [
          {
            "type": "text",
            "text": "What is in this image?"
          },
          {
            "type": "image_url",
            "image_url": {
              "url": "https://upload.wikimedia.org/wikipedia/commons/thumb/d/dd/Gfp-wisconsin-madison-the-nature-boardwalk.jpg/2560px-Gfp-wisconsin-madison-the-nature-boardwalk.jpg"
            }
          }
        ]
      }
    ]
  }'
```

Generate JSON data

Generate JSON data based on a JSON Schema

```javascript
import OpenAI from "openai";
const openai = new OpenAI();

const completion = await openai.chat.completions.create({
    model: "gpt-4o-2024-08-06",
    messages: [
        { role: "developer", content: "You extract email addresses into JSON data." },
        {
            role: "user",
            content: "Feeling stuck? Send a message to help@mycompany.com.",
        },
    ],
    response_format: {
        // See /docs/guides/structured-outputs
        type: "json_schema",
        json_schema: {
            name: "email_schema",
            schema: {
                type: "object",
                properties: {
                    email: {
                        description: "The email address that appears in the input",
                        type: "string"
                    }
                },
                additionalProperties: false
            }
        }
    }
});

console.log(completion.choices[0].message.content);
```

```python
from openai import OpenAI
client = OpenAI()

response = client.chat.completions.create(
    model="gpt-4o-2024-08-06",
    messages=[
        {
            "role": "developer", 
            "content": "You extract email addresses into JSON data."
        },
        {
            "role": "user", 
            "content": "Feeling stuck? Send a message to help@mycompany.com."
        }
    ],
    response_format={
        "type": "json_schema",
        "json_schema": {
            "name": "email_schema",
            "schema": {
                "type": "object",
                "properties": {
                    "email": {
                        "description": "The email address that appears in the input",
                        "type": "string"
                    },
                    "additionalProperties": False
                }
            }
        }
    }
)

print(response.choices[0].message.content);
```

```bash
curl https://api.openai.com/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -d '{
    "model": "gpt-4o-2024-08-06",
    "messages": [
      {
        "role": "developer",
        "content": "You extract email addresses into JSON data."
      },
      {
        "role": "user",
        "content": "Feeling stuck? Send a message to help@mycompany.com."
      }
    ],
    "response_format": {
      "type": "json_schema",
      "json_schema": {
        "name": "email_schema",
        "schema": {
            "type": "object",
            "properties": {
                "email": {
                    "description": "The email address that appears in the input",
                    "type": "string"
                }
            },
            "additionalProperties": false
        }
      }
    }
  }'
```

Choosing a model
----------------

When making a text generation request, the first option to configure is which [model](/docs/models) you want to generate the response. The model you choose can greatly influence the output, and impact how much each generation request [costs](https://openai.com/api/pricing/).

*   A **large model** like [`gpt-4o`](/docs/models#gpt-4o) will offer a very high level of intelligence and strong performance, while having a higher cost per token.
*   A **small model** like [`gpt-4o-mini`](/docs/models#gpt-4o-mini) offers intelligence not quite on the level of the larger model, but is faster and less expensive per token.
*   A **reasoning model** like [the `o1` family of models](/docs/models#o1) is slower to return a result, and uses more tokens to "think", but is capable of advanced reasoning, coding, and multi-step planning.

Experiment with different models [in the Playground](/playground) to see which one works best for your prompts! More information on choosing a model can [also be found here](/docs/guides/model-selection).

Building prompts
----------------

The process of crafting prompts to get the right output from a model is called **prompt engineering**. By giving the model precise instructions, examples, and necessary context information (like private or specialized information that wasn't included in the model's training data), you can improve the quality and accuracy of the model's output. Here, we'll get into some high-level guidance on building prompts, but you might also find the [prompt engineering guide](/docs/guides/prompt-engineering) helpful.

In the [chat completions](/docs/api-reference/chat/) API, you create prompts by providing an array of `messages` that contain instructions for the model. Each message can have a different `role`, which influences how the model might interpret the input.

### User messages

User messages contain instructions that request a particular type of output from the model. You can think of `user` messages as the messages you might type in to [ChatGPT](https://chaptgpt.com) as an end user.

Here's an example of a user message prompt that asks the `gpt-4o` model to generate a haiku poem based on a prompt.

```javascript
const response = await openai.chat.completions.create({
  model: "gpt-4o",
  messages: [
    {
      "role": "user",
      "content": [
        {
          "type": "text",
          "text": "Write a haiku about programming."
        }
      ]
    }
  ]
});
```

### Developer messages

Messages with the `developer` role provide instructions to the model that are prioritized ahead of user messages, as described in the [chain of command section in the model spec](https://cdn.openai.com/spec/model-spec-2024-05-08.html#follow-the-chain-of-command). They typically describe how the model should generally behave and respond. This message role used to be called the `system` prompt, but it has been renamed to more accurately describe its place in the instruction-following hierarchy.

Here's an example of a developer message that modifies the behavior of the model when generating a response to a `user` message:

```javascript
const response = await openai.chat.completions.create({
  model: "gpt-4o",
  messages: [
    {
      "role": "developer",
      "content": [
        {
          "type": "text",
          "text": `
            You are a helpful assistant that answers programming 
            questions in the style of a southern belle from the 
            southeast United States.
          `
        }
      ]
    },
    {
      "role": "user",
      "content": [
        {
          "type": "text",
          "text": "Are semicolons optional in JavaScript?"
        }
      ]
    }
  ]
});
```

This prompt returns a text output in the rhetorical style requested:

```text
Well, sugar, that's a fine question you've got there! Now, in the 
world of JavaScript, semicolons are indeed a bit like the pearls 
on a necklace – you might slip by without 'em, but you sure do look 
more polished with 'em in place. 

Technically, JavaScript has this little thing called "automatic 
semicolon insertion" where it kindly adds semicolons for you 
where it thinks they oughta go. However, it's not always perfect, 
bless its heart. Sometimes, it might get a tad confused and cause 
all sorts of unexpected behavior.
```

### Assistant messages

Messages with the `assistant` role are presumed to have been generated by the model, perhaps in a previous generation request (see the "Conversations" section below). They can also be used to provide examples to the model for how it should respond to the current request - a technique known as **few-shot learning**.

Here's an example of using an assistant message to capture the results of a previous text generation result, and making a new request based on that.

```javascript
const response = await openai.chat.completions.create({
  model: "gpt-4o",
  messages: [
    {
      "role": "user",
      "content": [{ "type": "text", "text": "knock knock." }]
    },
    {
      "role": "assistant",
      "content": [{ "type": "text", "text": "Who's there?" }]
    },
    {
      "role": "user",
      "content": [{ "type": "text", "text": "Orange." }]
    }
  ]
});
```

### Giving the model additional data to use for generation

The message types above can also be used to provide additional information to the model which may be outside its training data. You might want to include the results of a database query, a text document, or other resources to help the model generate a relevant response. This technique is often referred to as **retrieval augmented generation**, or RAG. [Learn more about RAG techniques here](https://help.openai.com/en/articles/8868588-retrieval-augmented-generation-rag-and-semantic-search-for-gpts).

Conversations and context
-------------------------

While each text generation request is independent and stateless (unless you are using [assistants](/docs/assistants/overview)), you can still implement **multi-turn conversations** by providing additional messages as parameters to your text generation request. Consider the "knock knock" joke example shown above:

```javascript
const response = await openai.chat.completions.create({
  model: "gpt-4o",
  messages: [
    {
      "role": "user",
      "content": [{ "type": "text", "text": "knock knock." }]
    },
    {
      "role": "assistant",
      "content": [{ "type": "text", "text": "Who's there?" }]
    },
    {
      "role": "user",
      "content": [{ "type": "text", "text": "Orange." }]
    }
  ]
});
```

By using alternating `user` and `assistant` messages, you can capture the previous state of a conversation in one request to the model.

### Managing context for text generation

As your inputs become more complex, or you include more and more turns in a conversation, you will need to consider both **output token** and **context window** limits. Model inputs and outputs are metered in [**tokens**](https://help.openai.com/en/articles/4936856-what-are-tokens-and-how-to-count-them), which are parsed from inputs to analyze their content and intent, and assembled to render logical outputs. Models have limits on how many tokens can be used during the lifecycle of a text generation request.

*   **Output tokens** are the tokens that are generated by a model in response to a prompt. Each model supports different limits for output tokens, [documented here](/docs/models). For example, `gpt-4o-2024-08-06` can generate a maximum of 16,384 output tokens.
*   A **context window** describes the total tokens that can be used for both input tokens and output tokens (and for some models, [reasoning tokens](/docs/guides/reasoning)), [documented here](/docs/models). For example, `gpt-4o-2024-08-06` has a total context window of 128k tokens.

If you create a very large prompt (usually by including a lot of conversation context or additional data/examples for the model), you run the risk of exceeding the allocated context window for a model, which might result in truncated outputs.

You can use the [tokenizer tool](/tokenizer) (which uses the [tiktoken library](https://github.com/openai/tiktoken)) to see how many tokens are present in a string of text.

Optimizing model outputs
------------------------

As you iterate on your prompts, you will be continually trying to improve **accuracy**, **cost**, and **latency**.

||
|Accuracy|Ensure the model produces accurate and useful responses to your prompts.|Accurate responses require that the model has all the information it needs to generate a response, and knows how to go about creating a response (from interpreting input to formatting and styling). Often, this will require a mix of prompt engineering, RAG, and model fine-tuning.Learn about optimizing for accuracy here.|
|Cost|Drive down the total cost of model usage by reducing token usage and using cheaper models when possible.|To control costs, you can try to use fewer tokens or smaller, cheaper models. Learn more about optimizing for cost here.|
|Latency|Decrease the time it takes to generate responses to your prompts.|Optimizing latency is a multi-faceted process including prompt engineering, parallelism in your own code, and more. Learn more here.|

===

Text to speech
==============

Learn how to turn text into lifelike spoken audio.

Overview
--------

The Audio API provides a [`speech`](/docs/api-reference/audio/createSpeech) endpoint based on our [TTS (text-to-speech) model](/docs/models#tts). It comes with 6 built-in voices and can be used to:

*   Narrate a written blog post
*   Produce spoken audio in multiple languages
*   Give real time audio output using streaming

Here is an example of the `alloy` voice:

Please note that our [usage policies](https://openai.com/policies/usage-policies) require you to provide a clear disclosure to end users that the TTS voice they are hearing is AI-generated and not a human voice.

Quickstart
----------

The `speech` endpoint takes in three key inputs: the [model](/docs/api-reference/audio/createSpeech#audio-createspeech-model), the [text](/docs/api-reference/audio/createSpeech#audio-createspeech-input) that should be turned into audio, and the [voice](/docs/api-reference/audio/createSpeech#audio-createspeech-voice) to be used for the audio generation. A simple request would look like the following:

Generate spoken audio from input text

```javascript
import fs from "fs";
import path from "path";
import OpenAI from "openai";

const openai = new OpenAI();
const speechFile = path.resolve("./speech.mp3");

const mp3 = await openai.audio.speech.create({
  model: "tts-1",
  voice: "alloy",
  input: "Today is a wonderful day to build something people love!",
});

const buffer = Buffer.from(await mp3.arrayBuffer());
await fs.promises.writeFile(speechFile, buffer);
```

```python
from pathlib import Path
from openai import OpenAI

client = OpenAI()
speech_file_path = Path(__file__).parent / "speech.mp3"
response = client.audio.speech.create(
    model="tts-1",
    voice="alloy",
    input="Today is a wonderful day to build something people love!",
)
response.stream_to_file(speech_file_path)
```

```bash
curl https://api.openai.com/v1/audio/speech \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "tts-1",
    "input": "Today is a wonderful day to build something people love!",
    "voice": "alloy"
  }' \
  --output speech.mp3
```

By default, the endpoint will output a MP3 file of the spoken audio but it can also be configured to output any of our [supported formats](#supported-output-formats).

### Audio quality

For real-time applications, the standard `tts-1` model provides the lowest latency but at a lower quality than the `tts-1-hd` model. Due to the way the audio is generated, `tts-1` is likely to generate content that has more static in certain situations than `tts-1-hd`. In some cases, the audio may not have noticeable differences depending on your listening device and the individual person.

### Voice options

Experiment with different voices (`alloy`, `echo`, `fable`, `onyx`, `nova`, and `shimmer`) to find one that matches your desired tone and audience. The current voices are optimized for English.

#### Alloy

#### Echo

#### Fable

#### Onyx

#### Nova

#### Shimmer

### Streaming real time audio

The Speech API provides support for real time audio streaming using [chunk transfer encoding](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Transfer-Encoding). This means that the audio is able to be played before the full file has been generated and made accessible.

```python
from openai import OpenAI

client = OpenAI()

response = client.audio.speech.create(
    model="tts-1",
    voice="alloy",
    input="Hello world! This is a streaming test.",
)

response.stream_to_file("output.mp3")
```

Supported output formats
------------------------

The default response format is "mp3", but other formats like "opus", "aac", "flac", and "pcm" are available.

*   **Opus**: For internet streaming and communication, low latency.
*   **AAC**: For digital audio compression, preferred by YouTube, Android, iOS.
*   **FLAC**: For lossless audio compression, favored by audio enthusiasts for archiving.
*   **WAV**: Uncompressed WAV audio, suitable for low-latency applications to avoid decoding overhead.
*   **PCM**: Similar to WAV but containing the raw samples in 24kHz (16-bit signed, low-endian), without the header.

Supported languages
-------------------

The TTS model generally follows the Whisper model in terms of language support. Whisper [supports the following languages](https://github.com/openai/whisper#available-models-and-languages) and performs well despite the current voices being optimized for English:

Afrikaans, Arabic, Armenian, Azerbaijani, Belarusian, Bosnian, Bulgarian, Catalan, Chinese, Croatian, Czech, Danish, Dutch, English, Estonian, Finnish, French, Galician, German, Greek, Hebrew, Hindi, Hungarian, Icelandic, Indonesian, Italian, Japanese, Kannada, Kazakh, Korean, Latvian, Lithuanian, Macedonian, Malay, Marathi, Maori, Nepali, Norwegian, Persian, Polish, Portuguese, Romanian, Russian, Serbian, Slovak, Slovenian, Spanish, Swahili, Swedish, Tagalog, Tamil, Thai, Turkish, Ukrainian, Urdu, Vietnamese, and Welsh.

You can generate spoken audio in these languages by providing the input text in the language of your choice.

FAQ
---

### How can I control the emotional range of the generated audio?

There is no direct mechanism to control the emotional output of the audio generated. Certain factors may influence the output audio like capitalization or grammar but our internal tests with these have yielded mixed results.

### Can I create a custom copy of my own voice?

No, this is not something we support.

### Do I own the outputted audio files?

Yes, like with all outputs from our API, the person who created them owns the output. You are still required to inform end users that they are hearing audio generated by AI and not a real person talking to them.

===

Vision
======

Learn how to use vision capabilities to understand images.

Many OpenAI [models](/docs/models) have vision capabilities, meaning the models can take in images and answer questions about them. Historically, language model systems have been limited by taking in a single input modality, text.

Quickstart
----------

Images are made available to the model in two main ways: by passing a link to the image or by passing the base64 encoded image directly in the request. Images can be passed in the `user` messages.

Analyze the content of an image

```javascript
import OpenAI from "openai";
const openai = new OpenAI();

const response = await openai.chat.completions.create({
  model: "gpt-4o-mini",
  messages: [
    {
      role: "user",
      content: [
        { type: "text", text: "What's in this image?" },
        {
          type: "image_url",
          image_url: {
            "url": "https://upload.wikimedia.org/wikipedia/commons/thumb/d/dd/Gfp-wisconsin-madison-the-nature-boardwalk.jpg/2560px-Gfp-wisconsin-madison-the-nature-boardwalk.jpg",
          },
        },
      ],
    },
  ],
});

console.log(response.choices[0]);
```

```python
from openai import OpenAI
client = OpenAI()

response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[
        {
            "role": "user",
            "content": [
                {"type": "text", "text": "What's in this image?"},
                {
                    "type": "image_url",
                    "image_url": {
                        "url": "https://upload.wikimedia.org/wikipedia/commons/thumb/d/dd/Gfp-wisconsin-madison-the-nature-boardwalk.jpg/2560px-Gfp-wisconsin-madison-the-nature-boardwalk.jpg",
                    },
                },
            ],
        }
    ],
    max_tokens=300,
)

print(response.choices[0])
```

```bash
curl https://api.openai.com/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [
      {
        "role": "user",
        "content": [
          {
            "type": "text",
            "text": "What is in this image?"
          },
          {
            "type": "image_url",
            "image_url": {
              "url": "https://upload.wikimedia.org/wikipedia/commons/thumb/d/dd/Gfp-wisconsin-madison-the-nature-boardwalk.jpg/2560px-Gfp-wisconsin-madison-the-nature-boardwalk.jpg"
            }
          }
        ]
      }
    ],
    "max_tokens": 300
  }'
```

The model is best at answering general questions about what is present in the images. While it does understand the relationship between objects in images, it is not yet optimized to answer detailed questions about the location of certain objects in an image. For example, you can ask it what color a car is or what some ideas for dinner might be based on what is in you fridge, but if you show it an image of a room and ask it where the chair is, it may not answer the question correctly.

It is important to keep in mind the [limitations of the model](/docs/guides/vision#limitations) as you explore what use-cases visual understanding can be applied to.

[

Video understanding with vision

Learn how to use use GPT-4 with Vision to understand videos in the OpenAI Cookbook

](https://cookbook.openai.com/examples/gpt_with_vision_for_video_understanding)

Uploading Base64 encoded images
-------------------------------

If you have an image or set of images locally, you can pass those to the model in base 64 encoded format, here is an example of this in action:

```python
import base64
from openai import OpenAI

client = OpenAI()

# Function to encode the image
def encode_image(image_path):
    with open(image_path, "rb") as image_file:
        return base64.b64encode(image_file.read()).decode("utf-8")

# Path to your image
image_path = "path_to_your_image.jpg"

# Getting the base64 string
base64_image = encode_image(image_path)

response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[
        {
            "role": "user",
            "content": [
                {
                    "type": "text",
                    "text": "What is in this image?",
                },
                {
                    "type": "image_url",
                    "image_url": {"url": f"data:image/jpeg;base64,{base64_image}"},
                },
            ],
        }
    ],
)

print(response.choices[0])
```

Multiple image inputs
---------------------

The Chat Completions API is capable of taking in and processing multiple image inputs in both base64 encoded format or as an image URL. The model will process each image and use the information from all of them to answer the question.

Multiple image inputs

```javascript
import OpenAI from "openai";
const openai = new OpenAI();

const response = await openai.chat.completions.create({
  model: "gpt-4o-mini",
  messages: [
    {
      role: "user",
      content: [
        { type: "text", text: "What are in these images? Is there any difference between them?" },
        {
          type: "image_url",
          image_url: {
            "url": "https://upload.wikimedia.org/wikipedia/commons/thumb/d/dd/Gfp-wisconsin-madison-the-nature-boardwalk.jpg/2560px-Gfp-wisconsin-madison-the-nature-boardwalk.jpg",
          },
        },
        {
          type: "image_url",
          image_url: {
            "url": "https://upload.wikimedia.org/wikipedia/commons/thumb/d/dd/Gfp-wisconsin-madison-the-nature-boardwalk.jpg/2560px-Gfp-wisconsin-madison-the-nature-boardwalk.jpg",
          },
        }
      ],
    },
  ],
});
console.log(response.choices[0]);
```

```python
from openai import OpenAI

client = OpenAI()
response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[
        {
            "role": "user",
            "content": [
                {
                    "type": "text",
                    "text": "What are in these images? Is there any difference between them?",
                },
                {
                    "type": "image_url",
                    "image_url": {
                        "url": "https://upload.wikimedia.org/wikipedia/commons/thumb/d/dd/Gfp-wisconsin-madison-the-nature-boardwalk.jpg/2560px-Gfp-wisconsin-madison-the-nature-boardwalk.jpg",
                    },
                },
                {
                    "type": "image_url",
                    "image_url": {
                        "url": "https://upload.wikimedia.org/wikipedia/commons/thumb/d/dd/Gfp-wisconsin-madison-the-nature-boardwalk.jpg/2560px-Gfp-wisconsin-madison-the-nature-boardwalk.jpg",
                    },
                },
            ],
        }
    ],
    max_tokens=300,
)
print(response.choices[0])
```

```bash
curl https://api.openai.com/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [
      {
        "role": "user",
        "content": [
          {
            "type": "text",
            "text": "What are in these images? Is there any difference between them?"
          },
          {
            "type": "image_url",
            "image_url": {
              "url": "https://upload.wikimedia.org/wikipedia/commons/thumb/d/dd/Gfp-wisconsin-madison-the-nature-boardwalk.jpg/2560px-Gfp-wisconsin-madison-the-nature-boardwalk.jpg",
            }
          },
          {
            "type": "image_url",
            "image_url": {
              "url": "https://upload.wikimedia.org/wikipedia/commons/thumb/d/dd/Gfp-wisconsin-madison-the-nature-boardwalk.jpg/2560px-Gfp-wisconsin-madison-the-nature-boardwalk.jpg",
            }
          }
        ]
      }
    ],
    "max_tokens": 300
  }'
```

Here the model is shown two copies of the same image and can answer questions about both or each of the images independently.

Low or high fidelity image understanding
----------------------------------------

By controlling the `detail` parameter, which has three options, `low`, `high`, or `auto`, you have control over how the model processes the image and generates its textual understanding. By default, the model will use the `auto` setting which will look at the image input size and decide if it should use the `low` or `high` setting.

*   `low` will enable the "low res" mode. The model will receive a low-res 512px x 512px version of the image, and represent the image with a budget of 85 tokens. This allows the API to return faster responses and consume fewer input tokens for use cases that do not require high detail.
*   `high` will enable "high res" mode, which first allows the model to first see the low res image (using 85 tokens) and then creates detailed crops using 170 tokens for each 512px x 512px tile.

Choosing the detail level

```javascript
import OpenAI from "openai";
const openai = new OpenAI();

const response = await openai.chat.completions.create({
  model: "gpt-4o-mini",
  messages: [
    {
      role: "user",
      content: [
        { type: "text", text: "What's in this image?" },
        {
          type: "image_url",
          image_url: {
            "url": "https://upload.wikimedia.org/wikipedia/commons/thumb/d/dd/Gfp-wisconsin-madison-the-nature-boardwalk.jpg/2560px-Gfp-wisconsin-madison-the-nature-boardwalk.jpg",
            "detail": "low"
          },
        },
      ],
    },
  ],
});

console.log(response.choices[0]);
```

```python
from openai import OpenAI
client = OpenAI()

response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[
        {
            "role": "user",
            "content": [
                {"type": "text", "text": "What's in this image?"},
                {
                    "type": "image_url",
                    "image_url": {
                        "url": "https://upload.wikimedia.org/wikipedia/commons/thumb/d/dd/Gfp-wisconsin-madison-the-nature-boardwalk.jpg/2560px-Gfp-wisconsin-madison-the-nature-boardwalk.jpg",
                        "detail": "high",
                    },
                },
            ],
        }
    ],
    max_tokens=300,
)

print(response.choices[0].message.content)
```

```bash
curl https://api.openai.com/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [
      {
        "role": "user",
        "content": [
          {
            "type": "text",
            "text": "What is in this image?"
          },
          {
            "type": "image_url",
            "image_url": {
              "url": "https://upload.wikimedia.org/wikipedia/commons/thumb/d/dd/Gfp-wisconsin-madison-the-nature-boardwalk.jpg/2560px-Gfp-wisconsin-madison-the-nature-boardwalk.jpg",
              "detail": "high"
            }
          }
        ]
      }
    ],
    "max_tokens": 300
  }'
```

Managing images
---------------

The Chat Completions API, unlike the Assistants API, is not stateful. That means you have to manage the messages (including images) you pass to the model yourself. If you want to pass the same image to the model multiple times, you will have to pass the image each time you make a request to the API.

For long running conversations, we suggest passing images via URL's instead of base64. The latency of the model can also be improved by downsizing your images ahead of time to be less than the maximum size they are expected them to be. For low res mode, we expect a 512px x 512px image. For high res mode, the short side of the image should be less than 768px and the long side should be less than 2,000px.

After an image has been processed by the model, it is deleted from OpenAI servers and not retained. [We do not use data uploaded via the OpenAI API to train our models](https://openai.com/enterprise-privacy).

Limitations
-----------

While GPT-4 with vision is powerful and can be used in many situations, it is important to understand the limitations of the model. Here are some of the limitations we are aware of:

*   Medical images: The model is not suitable for interpreting specialized medical images like CT scans and shouldn't be used for medical advice.
*   Non-English: The model may not perform optimally when handling images with text of non-Latin alphabets, such as Japanese or Korean.
*   Small text: Enlarge text within the image to improve readability, but avoid cropping important details.
*   Rotation: The model may misinterpret rotated / upside-down text or images.
*   Visual elements: The model may struggle to understand graphs or text where colors or styles like solid, dashed, or dotted lines vary.
*   Spatial reasoning: The model struggles with tasks requiring precise spatial localization, such as identifying chess positions.
*   Accuracy: The model may generate incorrect descriptions or captions in certain scenarios.
*   Image shape: The model struggles with panoramic and fisheye images.
*   Metadata and resizing: The model doesn't process original file names or metadata, and images are resized before analysis, affecting their original dimensions.
*   Counting: May give approximate counts for objects in images.
*   CAPTCHAS: For safety reasons, we have implemented a system to block the submission of CAPTCHAs.

Calculating costs
-----------------

Image inputs are metered and charged in tokens, just as text inputs are. The token cost of a given image is determined by two factors: its size, and the `detail` option on each image\_url block. All images with `detail: low` cost 85 tokens each. `detail: high` images are first scaled to fit within a 2048 x 2048 square, maintaining their aspect ratio. Then, they are scaled such that the shortest side of the image is 768px long. Finally, we count how many 512px squares the image consists of. Each of those squares costs **170 tokens**. Another **85 tokens** are always added to the final total.

Here are some examples demonstrating the above.

*   A 1024 x 1024 square image in `detail: high` mode costs 765 tokens
    *   1024 is less than 2048, so there is no initial resize.
    *   The shortest side is 1024, so we scale the image down to 768 x 768.
    *   4 512px square tiles are needed to represent the image, so the final token cost is `170 * 4 + 85 = 765`.
*   A 2048 x 4096 image in `detail: high` mode costs 1105 tokens
    *   We scale down the image to 1024 x 2048 to fit within the 2048 square.
    *   The shortest side is 1024, so we further scale down to 768 x 1536.
    *   6 512px tiles are needed, so the final token cost is `170 * 6 + 85 = 1105`.
*   A 4096 x 8192 image in `detail: low` most costs 85 tokens
    *   Regardless of input size, low detail images are a fixed cost.

FAQ
---

### Can I fine-tune the image capabilities in `gpt-4`?

Vision fine-tuning is available for some models - [learn more](/docs/guides/fine-tuning#vision).

### Can I use `gpt-4` to generate images?

No, you can use `dall-e-3` to generate images and `gpt-4o`, `gpt-4o-mini` or `gpt-4-turbo` to understand images.

### What type of files can I upload?

We currently support PNG (.png), JPEG (.jpeg and .jpg), WEBP (.webp), and non-animated GIF (.gif).

### Is there a limit to the size of the image I can upload?

Yes, we restrict image uploads to 20MB per image.

### Can I delete an image I uploaded?

No, we will delete the image for you automatically after it has been processed by the model.

### Where can I learn more about the considerations of GPT-4 with Vision?

You can find details about our evaluations, preparation, and mitigation work in the [GPT-4 with Vision system card](https://openai.com/contributions/gpt-4v).

We have further implemented a system to block the submission of CAPTCHAs.

### How do rate limits for GPT-4 with Vision work?

We process images at the token level, so each image we process counts towards your tokens per minute (TPM) limit. See the calculating costs section for details on the formula used to determine token count per image.

### Can GPT-4 with Vision understand image metadata?

No, the model does not receive image metadata.

### What happens if my image is unclear?

If an image is ambiguous or unclear, the model will do its best to interpret it. However, the results may be less accurate. A good rule of thumb is that if an average human cannot see the info in an image at the resolutions used in low/high res mode, then the model cannot either.


