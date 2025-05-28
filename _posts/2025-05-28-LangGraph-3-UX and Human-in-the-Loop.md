---
title: LangGraph-3-UX and Human-in-the-Loop
category: aigc
tags:
  - agent
  - ai
  - langgraph
  - python
---

本模块聚焦于人机协作，它基于记忆功能，使用户能够直接与图进行多样的交互。为了更好地引入人机协作，将先介绍流式处理，它可以在执行过程中以多种方式展示图的输出，如节点状态或聊天模型的标记等。

- **人机协作（human-in-the-loop）**：这是一种将人类的智慧和决策融入到自动化系统中的方法。在这个上下文中，它意味着用户能够积极参与到图的交互过程中，而不是完全依赖自动化系统。例如，用户可以根据图的实时状态做出决策，或者对图的某些部分进行手动调整。
- **基于记忆功能**：这表明人机协作的实现依赖于之前提到的记忆功能。记忆功能可能是指系统能够存储和回忆之前的信息，这使得用户在与图交互时能够基于这些记忆进行操作，从而实现更连贯、更个性化的交互体验。
- **直接与图交互**：用户可以直接对图进行操作，而不是通过复杂的界面或间接的方式。这种直接交互可以是修改图的结构、调整节点的属性，或者根据图的输出做出即时反馈。
- **流式处理（streaming）**：这是一种数据处理方式，数据以连续的流的形式被处理，而不是一次性处理所有数据。在这个场景中，流式处理用于实时展示图的输出，使用户能够看到图在执行过程中的动态变化。

# Streaming

## 流式输出

`.stream` 和 .`astream` 是用于回传结果的同步和异步方法。

LangGraph 支持几种不同的图状态流模式：

* `values`：此操作在调用每个节点之后会流式传输整个图的状态。
* `updates`：此操作在调用每个节点之后会流式传输图状态的更新。

```python
# Create a thread
config = {"configurable": {"thread_id": "1"}}

# Start conversation
for chunk in graph.stream({"messages": [HumanMessage(content="hi! I'm Lance")]}, config, stream_mode="updates"):
    print(chunk)
```

```
{'conversation': {'messages': AIMessage(content='Hi Lance! How can I assist you today?', additional_kwargs={'refusal': None}, response_metadata={'token_usage': {'completion_tokens': 10, 'prompt_tokens': 11, 'total_tokens': 21}, 'model_name': 'gpt-4o-2024-05-13', 'system_fingerprint': 'fp_25624ae3a5', 'finish_reason': 'stop', 'logprobs': None}, id='run-6d58e31e-a278-4df6-ab0a-bb51d08ca037-0', usage_metadata={'input_tokens': 11, 'output_tokens': 10, 'total_tokens': 21})}}
```

打印state的更新


```
================================== Ai Message ==================================
Hi Lance! How are you doing today?
```

可以看到，用户输入`"hi! I'm Lance"`,AI回复`Hi Lance! How can I assist you today?'`，在当前的update模式下，用户输入的信息将不会被保留，而是被更新为AI的回复，因为是`update`模式

**对比**

```python
# Start conversation, again
config = {"configurable": {"thread_id": "2"}}

# Start conversation
input_message = HumanMessage(content="hi! I'm Lance")
for event in graph.stream({"messages": [input_message]}, config, stream_mode="values"):
    for m in event['messages']:
        m.pretty_print()
    print("---"*25)
```

```python
================================ Human Message =================================

hi! I'm Lance
---------------------------------------------------------------------------
================================ Human Message =================================

hi! I'm Lance
================================== Ai Message ==================================

Hi Lance! How can I assist you today?
---------------------------------------------------------------------------
```



## Streaming tokens

我们在流式传输时往往不止希望获得图状态信息，例如对于聊天模型的通话而言，通常会实时传输生成的令牌。我们可以使用 `.astream_events` 方法来实现这一功能，该方法会在节点内部实时回传事件！

每个事件都是一个字典，其中包含几个键值对：

- event：这是正在被发出的此类事件。
- name：这是该事件的名称。
- data：这是与该事件相关联的数据。
- metadata：包含 langgraph_node，即发出该事件的节点。

```python
config = {"configurable": {"thread_id": "3"}}
input_message = HumanMessage(content="Tell me about the 49ers NFL team")
async for event in graph.astream_events({"messages": [input_message]}, config, version="v2"):
    print(f"Node: {event['metadata'].get('langgraph_node','')}. Type: {event['event']}. Name: {event['name']}")
```

```
Node: . Type: on_chain_start. Name: LangGraph
Node: __start__. Type: on_chain_start. Name: __start__
Node: __start__. Type: on_chain_end. Name: __start__
Node: conversation. Type: on_chain_start. Name: conversation
Node: conversation. Type: on_chat_model_start. Name: ChatOpenAI
Node: conversation. Type: on_chat_model_stream. Name: ChatOpenAI
Node: conversation. Type: on_chat_model_stream. Name: ChatOpenAI
Node: conversation. Type: on_chat_model_stream. Name: ChatOpenAI
Node: conversation. Type: on_chat_model_stream. Name: ChatOpenAI
Node: conversation. Type: on_chat_model_stream. Name: ChatOpenAI
Node: conversation. Type: on_chat_model_stream. Name: ChatOpenAI

..........

Node: conversation. Type: on_chat_model_stream. Name: ChatOpenAI
Node: conversation. Type: on_chat_model_stream. Name: ChatOpenAI
Node: conversation. Type: on_chat_model_end. Name: ChatOpenAI
Node: conversation. Type: on_chain_start. Name: ChannelWrite<conversation,messages,summary>
Node: conversation. Type: on_chain_end. Name: ChannelWrite<conversation,messages,summary>
Node: conversation. Type: on_chain_start. Name: should_continue
Node: conversation. Type: on_chain_end. Name: should_continue
Node: conversation. Type: on_chain_stream. Name: conversation
Node: conversation. Type: on_chain_end. Name: conversation
Node: . Type: on_chain_stream. Name: LangGraph
Node: . Type: on_chain_end. Name: LangGraph
```

核心要点在于，您图中的聊天模型生成的令牌具有 `on_chat_model_stream` 类型。
我们可以使用 `event['metadata']['langgraph_node']`来选择要进行流式传输的节点。
并且我们可以使用 `event['data']`来获取每个事件的实际数据，而在这种情况下，该数据是一个 `AIMessageChunk` 类型的对象。

```python
node_to_stream = 'conversation'
config = {"configurable": {"thread_id": "4"}}
input_message = HumanMessage(content="Tell me about the 49ers NFL team")
async for event in graph.astream_events({"messages": [input_message]}, config, version="v2"):
    # Get chat model tokens from a particular node 
    if event["event"] == "on_chat_model_stream" and event['metadata'].get('langgraph_node','') == node_to_stream:
        print(event["data"])
```

```
{'chunk': AIMessageChunk(content='', id='run-b76ec3b8-9c45-42fe-b321-4ec3a69c185c')}
{'chunk': AIMessageChunk(content='The', id='run-b76ec3b8-9c45-42fe-b321-4ec3a69c185c')}
{'chunk': AIMessageChunk(content=' San', id='run-b76ec3b8-9c45-42fe-b321-4ec3a69c185c')}
{'chunk': AIMessageChunk(content=' Francisco', id='run-b76ec3b8-9c45-42fe-b321-4ec3a69c185c')}
{'chunk': AIMessageChunk(content=' ', id='run-b76ec3b8-9c45-42fe-b321-4ec3a69c185c')}
{'chunk': AIMessageChunk(content='49', id='run-b76ec3b8-9c45-42fe-b321-4ec3a69c185c')}
{'chunk': AIMessageChunk(content='ers', id='run-b76ec3b8-9c45-42fe-b321-4ec3a69c185c')}
{'chunk': AIMessageChunk(content=' are', id='run-b76ec3b8-9c45-42fe-b321-4ec3a69c185c')}
{'chunk': AIMessageChunk(content=' a', id='run-b76ec3b8-9c45-42fe-b321-4ec3a69c185c')}
{'chunk': AIMessageChunk(content=' professional', id='run-b76ec3b8-9c45-42fe-b321-4ec3a69c185c')}
.......

```

只需使用 chunk key 来获取 AIMessageChunk 即可。

```python
config = {"configurable": {"thread_id": "5"}}
input_message = HumanMessage(content="Tell me about the 49ers NFL team")
async for event in graph.astream_events({"messages": [input_message]}, config, version="v2"):
    # Get chat model tokens from a particular node 
    if event["event"] == "on_chat_model_stream" and event['metadata'].get('langgraph_node','') == node_to_stream:
        data = event["data"]
        print(data["chunk"].content, end="|")
```

```
|The| San| Francisco| |49|ers| are| a| professional| American| football| team| based| in| the| San| Francisco| Bay| Area|.| They| compete| in| the| National| Football| League| (|NFL|)| as| a| member| of| the| league|'s| National| Football| Conference| (|N|FC|)| West| division|.| Here| are| some| key| points| about| the| team|:

|###| History|
|-| **|Founded|:**| |194|6| as| a| charter| member| of| the| All|-Amer|ica| Football| Conference| (|AA|FC|)| and| joined| the| NFL| in| |194|9| when| the| leagues| merged|.
|-| **|Team| Name|:**| The| name| "|49|ers|"| is| a| reference| to| the| prospect|ors| who| arrived| in| Northern| California| during| the| |184|9| Gold| Rush|.
```



# Breakpoints 断点

对于 `human-in-the-loop` ，我们通常希望在运行过程中查看其graph输出。

1.  `Approval` - 我们可以中断我们的代理，向用户呈现当前状态，并允许用户接受一个操作。

2) `Debugging` - 我们可以回溯图以重现或避免问题

3. `Editing` - 您可以修改状态

## 设置断点
有两个地方可以设置断点：

* 通过在编译时或运行时设置断点，在节点执行之前或之后执行。我们称这些为**静态断点**。
* 在节点内部使用NodeInterrupt异常。我们称之为**动态断点**。

本节介绍**静态断点**

## 静态断点
静态断点在节点执行之前或之后触发。你可以通过在**编译时**或**运行时**指定`interrupt_before`和`interrupt_after`来设置静态断点。

如果您想每次单步执行一个节点的图，或者想在特定节点暂停图的执行，静态断点对于调试特别有用。

编译时

```python
graph = graph_builder.compile( 
    interrupt_before=["node_a"], 
    interrupt_after=["node_b", "node_c"], 
    checkpointer=checkpointer, 
)

config = {
    "configurable": {
        "thread_id": "some_thread"
    }
}

# Run the graph until the breakpoint
graph.invoke(inputs, config=thread_config) 

# Resume the graph
graph.invoke(None, config=thread_config)
```

运行时

```python
graph.invoke( 
    inputs, 
    interrupt_before=["node_a"], 
    interrupt_after=["node_b", "node_c"] 
    config={
        "configurable": {"thread_id": "some_thread"}
    }, 
)

config = {
    "configurable": {
        "thread_id": "some_thread"
    }
}

# Run the graph until the breakpoint
graph.invoke(inputs, config=config) 

# Resume the graph
graph.invoke(None, config=config)
```

## 人工审批的断点

- 让我们重新思考一下在模块 1 中我们所研究的那个简单的智能体。
- 假设我们关注的是工具使用：我们希望批准该智能体使用其任何工具。
- 我们所需要做的就是简单地用 `interrupt_before=["tools"]` 编译图形，其中 `tools` 是我们的工具节点。
- 这意味着执行将在执行工具调用的节点 `tools` 之前中断。

module3/breakpoint.py

```python
from langchain_openai import ChatOpenAI

def multiply(a: int, b: int) -> int:
    """Multiply a and b.

    Args:
        a: first int
        b: second int
    """
    return a * b

# This will be a tool
def add(a: int, b: int) -> int:
    """Adds a and b.

    Args:
        a: first int
        b: second int
    """
    return a + b

def divide(a: int, b: int) -> float:
    """Divide a by b.

    Args:
        a: first int
        b: second int
    """
    return a / b

tools = [add, multiply, divide]
llm = ChatOpenAI(model="gpt-4o")
llm_with_tools = llm.bind_tools(tools)

from IPython.display import Image, display

from langgraph.checkpoint.memory import MemorySaver
from langgraph.graph import MessagesState
from langgraph.graph import START, StateGraph
from langgraph.prebuilt import tools_condition, ToolNode

from langchain_core.messages import AIMessage, HumanMessage, SystemMessage

# System message
sys_msg = SystemMessage(content="You are a helpful assistant tasked with performing arithmetic on a set of inputs.")

# Node
def assistant(state: MessagesState):
   return {"messages": [llm_with_tools.invoke([sys_msg] + state["messages"])]}

# Graph
builder = StateGraph(MessagesState)

# Define nodes: these do the work
builder.add_node("assistant", assistant)
builder.add_node("tools", ToolNode(tools))

# Define edges: these determine the control flow
builder.add_edge(START, "assistant")
builder.add_conditional_edges(
    "assistant",
    # If the latest message (result) from assistant is a tool call -> tools_condition routes to tools
    # If the latest message (result) from assistant is a not a tool call -> tools_condition routes to END
    tools_condition,
)
builder.add_edge("tools", "assistant")

memory = MemorySaver()
graph = builder.compile(interrupt_before=["tools"], checkpointer=memory)

# Show
display(Image(graph.get_graph(xray=True).draw_mermaid_png()))
```

![截屏2025-04-30 10.05.04.png](/assets/images/截屏2025-04-30%2010.05.04.png)

```python
# Input
initial_input = {"messages": HumanMessage(content="Multiply 2 and 3")}

# Thread
thread = {"configurable": {"thread_id": "1"}}

# Run the graph until the first interruption
for event in graph.stream(initial_input, thread, stream_mode="values"):
    event['messages'][-1].pretty_print()
```

```
================================ Human Message =================================

Multiply 2 and 3
================================== Ai Message ==================================
Tool Calls:
  multiply (call_oFkGpnO8CuwW9A1rk49nqBpY)
 Call ID: call_oFkGpnO8CuwW9A1rk49nqBpY
  Args:
    a: 2
    b: 3
```

我们可以获取状态并查看下一个节点来调用。这是一种很好的方式来表明图已被中断。

```python
state = graph.get_state(thread)
state.next
```

```
('tools',)
```

**现在，我们要来介绍一个不错的窍门。当我们使用 None 来调用该图时，它将从上一次的状态检查点继续运行！**


![截屏2025-04-30 10.10.09.png](/assets/images/截屏2025-04-30%2010.10.09.png)

为了清晰起见，LangGraph 将重新发送当前状态，其中包含带有工具调用的 AIMessage。
然后它将按照图中的步骤执行下去，这些步骤始于工具节点（tools）。
我们看到，工具节点是通过这个工具调用指令来运行的，并且它会被传递回聊天模型，以供我们得出最终答案。

```python
for event in graph.stream(None, thread, stream_mode="values"):
    event['messages'][-1].pretty_print()
```

```
================================== Ai Message ==================================
Tool Calls:
  multiply (call_oFkGpnO8CuwW9A1rk49nqBpY)
 Call ID: call_oFkGpnO8CuwW9A1rk49nqBpY
  Args:
    a: 2
    b: 3
================================= Tool Message =================================
Name: multiply

6
================================== Ai Message ==================================

The result of multiplying 2 and 3 is 6.
```

**现在，让我们将这些与接受用户输入的特定用户批准步骤结合在一起。**

```python
# Input
initial_input = {"messages": HumanMessage(content="Multiply 2 and 3")}

# Thread
thread = {"configurable": {"thread_id": "1"}}

# Run the graph until the first interruption
for event in graph.stream(initial_input, thread, stream_mode="values"):
    message = event['messages'][-1]
    print(f"Message type: {type(message)}")
    print(f"Message content: {message.content}")
    message.pretty_print()

# Get user feedback
user_approval = input("Do you want to call the tool? (yes/no): ")

# Check approval
if user_approval.lower() == "yes":
    # If approved, continue the graph execution
    for event in graph.stream(None, thread, stream_mode="values"):
        event['messages'][-1].pretty_print()
else:
    print("Operation cancelled by user.")    
```

* 当graph执行到需要调用工具时（比如这里执行乘法操作），它会暂停执行（通过interrupt_before=["tools"]实现）
* 此时，代码会通过input()函数获取用户的确认：`user_approval = input("Do you want to call the tool? (yes/no): ")`

* 查看input函数的定义，可以看到它将会从标准输入获得信息

  ```python
  (function) def input(
      prompt: object = "",
      /
  ) -> str
  Read a string from standard input. The trailing newline is stripped.
  
  The prompt string, if given, is printed to standard output without a trailing newline before reading input.
  
  If the user hits EOF (*nix: Ctrl-D, Windows: Ctrl-Z+Return), raise EOFError. On *nix systems, readline is used if available.
  ```

运行时，输出

```
Message type: <class 'langchain_core.messages.human.HumanMessage'>
Message content: Multiply 2 and 3
================================ Human Message =================================

Multiply 2 and 3
Message type: <class 'langchain_core.messages.ai.AIMessage'>
Message content: 
================================== Ai Message ==================================
Tool Calls:
  multiply (call_cd69029ef57f43f2b149f7)
 Call ID: call_cd69029ef57f43f2b149f7
  Args:
    a: 2
    b: 3
Do you want to call the tool? (yes/no): 
```

此时，我们可以输入yes，no，影响程序运行结果

```
Do you want to call the tool? (yes/no): yes
================================== Ai Message ==================================
Tool Calls:
  multiply (call_cd69029ef57f43f2b149f7)
 Call ID: call_cd69029ef57f43f2b149f7
  Args:
    a: 2
    b: 3
================================= Tool Message =================================
Name: multiply

6
================================== Ai Message ==================================

The product of 2 and 3 is 6.
```



# Editing graph state

## 状态编辑

我们利用它们来中断图形流程，并在执行下一个节点之前等待用户批准。同时，断点也是[修改图形状态的机会](https://langchain-ai.github.io/langgraph/how-tos/human_in_the_loop/edit-graph-state/)。
让我们在辅助节点之前给我们的代理设置一个断点。

module3/editgraphstate.py

```python
from langchain_openai import ChatOpenAI

def multiply(a: int, b: int) -> int:
    """Multiply a and b.

    Args:
        a: first int
        b: second int
    """
    return a * b

# This will be a tool
def add(a: int, b: int) -> int:
    """Adds a and b.

    Args:
        a: first int
        b: second int
    """
    return a + b

def divide(a: int, b: int) -> float:
    """Divide a by b.

    Args:
        a: first int
        b: second int
    """
    return a / b

tools = [add, multiply, divide]
llm = ChatOpenAI(model="gpt-4o")
llm_with_tools = llm.bind_tools(tools)

from IPython.display import Image, display

from langgraph.checkpoint.memory import MemorySaver
from langgraph.graph import MessagesState
from langgraph.graph import START, StateGraph
from langgraph.prebuilt import tools_condition, ToolNode

from langchain_core.messages import HumanMessage, SystemMessage

# System message
sys_msg = SystemMessage(content="You are a helpful assistant tasked with performing arithmetic on a set of inputs.")

# Node
def assistant(state: MessagesState):
   return {"messages": [llm_with_tools.invoke([sys_msg] + state["messages"])]}

# Graph
builder = StateGraph(MessagesState)

# Define nodes: these do the work
builder.add_node("assistant", assistant)
builder.add_node("tools", ToolNode(tools))

# Define edges: these determine the control flow
builder.add_edge(START, "assistant")
builder.add_conditional_edges(
    "assistant",
    # If the latest message (result) from assistant is a tool call -> tools_condition routes to tools
    # If the latest message (result) from assistant is a not a tool call -> tools_condition routes to END
    tools_condition,
)
builder.add_edge("tools", "assistant")

memory = MemorySaver()
graph = builder.compile(interrupt_before=["assistant"], checkpointer=memory)

# Show
display(Image(graph.get_graph(xray=True).draw_mermaid_png()))
```

![截屏2025-04-30 10.52.36.png](/assets/images/截屏2025-04-30%2010.52.36.png)

我们可以看到，在聊天模型作出回应之前，图就已经中断了。

```python
# Input
initial_input = {"messages": "Multiply 2 and 3"}

# Thread
thread = {"configurable": {"thread_id": "1"}}

# Run the graph until the first interruption
for event in graph.stream(initial_input, thread, stream_mode="values"):
    event['messages'][-1].pretty_print()
```

```
================================ Human Message =================================

Multiply 2 and 3
```

```
state = graph.get_state(thread)
state
```

```python
StateSnapshot(values={'messages': [HumanMessage(content='Multiply 2 and 3', id='e7edcaba-bfed-4113-a85b-25cc39d6b5a7')]}, next=('assistant',), config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef6a412-5b2d-601a-8000-4af760ea1d0d'}}, metadata={'source': 'loop', 'writes': None, 'step': 0, 'parents': {}}, created_at='2024-09-03T22:09:10.966883+00:00', parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef6a412-5b28-6ace-bfff-55d7a2c719ae'}}, tasks=(PregelTask(id='dbee122a-db69-51a7-b05b-a21fab160696', name='assistant', error=None, interrupts=(), state=None),))
```

- 现在，我们可以直接执行状态更新操作。

- 请记住，对消息键的更新将会使用 `add_messages` reducer：
- 如果我们想要覆盖现有的消息内容，那么我们可以提供消息 ID。
- 如果我们只是想在我们的消息列表中追加一条新消息，那么我们可以传递一条未指定消息 ID 的消息，如下所示。

```python
graph.update_state(
    thread,
    {"messages": [HumanMessage(content="No, actually multiply 3 and 3!")]},
)
```

```
{'configurable': {'thread_id': '1',
  'checkpoint_ns': '',
  'checkpoint_id': '1ef6a414-f419-6182-8001-b9e899eca7e5'}}
```

我们调用了 update_state 函数，并传入了一条新的消息。
add_messages 这个 reducer 将其附加到我们的状态键 messages 上。

```python
new_state = graph.get_state(thread).values
for m in new_state['messages']:
    m.pretty_print()
```

```python
================================ Human Message =================================

Multiply 2 and 3
================================ Human Message =================================

No, actually multiply 3 and 3!
```

现在，让我们继续使用我们的代理程序，只需传递 None 并允许其从当前状态继续执行即可。
我们发出当前节点的信息，然后继续执行其余节点。

```python
for event in graph.stream(None, thread, stream_mode="values"):
    event['messages'][-1].pretty_print()
```

```
================================ Human Message =================================

No, actually multiply 3 and 3!
================================== Ai Message ==================================
Tool Calls:
  multiply (call_Mbu8MfA0krQh8rkZZALYiQMk)
 Call ID: call_Mbu8MfA0krQh8rkZZALYiQMk
  Args:
    a: 3
    b: 3
================================= Tool Message =================================
Name: multiply

9
```

现在，我们又回到了`assistant`程序那里，它设置了我们的断点。
我们可以再次传入 None 来继续执行。

```python
for event in graph.stream(None, thread, stream_mode="values"):
    event['messages'][-1].pretty_print()
```

```
================================= Tool Message =================================
Name: multiply

9
================================== Ai Message ==================================

3 multiplied by 3 equals 9.
```

## 等待用户输入

很明显，在遇到断点时我们可以编辑我们的代理状态。那么，如果我们想要允许人为反馈来执行这种状态更新的话，该怎么办呢？

* 我们将添加一个node节点，用作我们代理系统中[接收人类反馈的暂存位置](https://langchain-ai.github.io/langgraph/how-tos/human_in_the_loop/wait-user-input/#setup)。用户可以通过这个节点直接对代理的状态进行修改或调整，而不是通过其他间接的方式。这个`human_feedback`节点允许用户直接向状态添加反馈。
* 我们在`human_feedback`节点的`interrupt_before`中断点设置中断条件。我们设置了一个`checkpointer`，用于保存从该节点之前到此节点为止的图的状态。这里的意思是，在人类反馈节点之前，保存整个系统（图）的状态，以便在需要时可以恢复到这个状态，或者对状态进行分析和修改。

module3/editgraphstate2.py

```python
# System message
sys_msg = SystemMessage(content="You are a helpful assistant tasked with performing arithmetic on a set of inputs.")

# no-op node that should be interrupted on
def human_feedback(state: MessagesState):
    pass

# Assistant node
def assistant(state: MessagesState):
   return {"messages": [llm_with_tools.invoke([sys_msg] + state["messages"])]}

# Graph
builder = StateGraph(MessagesState)

# Define nodes: these do the work
builder.add_node("assistant", assistant)
builder.add_node("tools", ToolNode(tools))
builder.add_node("human_feedback", human_feedback)

# Define edges: these determine the control flow
builder.add_edge(START, "human_feedback")
builder.add_edge("human_feedback", "assistant")
builder.add_conditional_edges(
    "assistant",
    # If the latest message (result) from assistant is a tool call -> tools_condition routes to tools
    # If the latest message (result) from assistant is a not a tool call -> tools_condition routes to END
    tools_condition,
)
builder.add_edge("tools", "human_feedback")

memory = MemorySaver()
graph = builder.compile(interrupt_before=["human_feedback"], checkpointer=memory)
display(Image(graph.get_graph().draw_mermaid_png()))
```

![截屏2025-04-30 11.38.59.png](/assets/images/截屏2025-04-30%2011.38.59.png)

**human_feedback 节点的作用**：

- human_feedback 是一个中断点，用于暂停图的执行，等待用户输入。
- 它本身是一个 pass 节点，不会对状态进行任何实际的处理，只是作为一个标记点

**graph.update_state 的作用**：

- 当调用 graph.update_state 时，你实际上是在手动更新图的状态。
- 通过指定 as_node="human_feedback"，你告诉系统这次状态更新是通过 human_feedback 节点进行的。
- 这个操作会将用户输入的内容（user_input）插入到图的状态中，并且标记为 human_feedback 节点的输出。

```py
# Input
initial_input = {"messages": "Multiply 2 and 3"}

# Thread
thread = {"configurable": {"thread_id": "5"}}

# Run the graph until the first interruption
for event in graph.stream(initial_input, thread, stream_mode="values"):
    event["messages"][-1].pretty_print()
    
# Get user input
user_input = input("Tell me how you want to update the state: ")

# We now update the state as if we are the human_feedback node
graph.update_state(thread, {"messages": user_input}, as_node="human_feedback")

# Continue the graph execution
for event in graph.stream(None, thread, stream_mode="values"):
    event["messages"][-1].pretty_print()
```

```
# 输入：no, multiply 3 and 3
================================ Human Message =================================

Multiply 2 and 3
Tell me how you want to update the state: no, multiply 3 and 3
================================ Human Message =================================

no, multiply 3 and 3
================================== Ai Message ==================================
Tool Calls:
  multiply (call_864833cafb1c4cd9b48f2d)
 Call ID: call_864833cafb1c4cd9b48f2d
  Args:
    a: 3
    b: 3
================================= Tool Message =================================
Name: multiply

9
```

继续

```python
# Continue the graph execution
for event in graph.stream(None, thread, stream_mode="values"):
    event["messages"][-1].pretty_print()
```

```
================================= Tool Message =================================
Name: multiply

9
================================== Ai Message ==================================

The result of multiplying 3 and 3 is 9.
```

这里提个疑问，为什么第二次执行不会触发中断？
- 在第一次执行 graph.stream 时，图在 human_feedback 节点处中断，等待用户输入。
- 当你调用 graph.update_state 并指定 as_node="human_feedback" 时，你实际上是在模拟用户输入，并将这个输入作为 human_feedback 节点的输出。
- 因此，当第二次执行 graph.stream 时，图会从 human_feedback 节点的输出开始继续执行，而不会再次中断等待用户输入，因为 human_feedback 节点已经被“处理”过了。

# 动态断点

在图形编译过程中，开发人员会在特定节点处设置断点，这属于静态断点。但是，有时候允许图形动态地自行中断是有帮助的，这就是要介绍的动态断点`interrupt()`。

> 注意：LangGraph 0.2.57版本起推荐使用 `interrupt()` 函数来实现人机交互断点，老版本可以通过[调用 NodeInterrupt 来实现](https://langchain-ai.github.io/langgraph/how-tos/human_in_the_loop/dynamic_breakpoints/#run-the-graph-with-dynamic-interrupt)。

## interrupt

LangGraph中的`interrupt`函数通过将图暂停在特定节点，将信息呈现给人，然后根据人的输入（Command对象）恢复图来实现人在回路工作流。它对于审批、编辑或收集额外的上下文等任务非常有用。

`interrupt`属于动态断点：

1. **运行时决定**：它在节点执行过程中根据程序逻辑动态调用，而不是预先配置
2. **条件触发**：可以基于状态或其他条件来决定是否调用 `interrupt`
3. **灵活控制**：可以在同一个节点中多次调用，实现复杂的交互逻辑

### `interrupt`函数定义

`interrupt`函数定义如下所示，函数接受一个 `value: Any` 参数，这意味着您可以传递任何 JSON 可序列化的值。键名可以随意设定，可以传递复杂的嵌套结构，也可以是一个普通的string。

```python
def interrupt(value: Any) -> Any:
    """Interrupt the graph with a resumable exception from within a node.

    The `interrupt` function enables human-in-the-loop workflows by pausing graph
    execution and surfacing a value to the client. This value can communicate context
    or request input required to resume execution.

    In a given node, the first invocation of this function raises a `GraphInterrupt`
    exception, halting execution. The provided `value` is included with the exception
    and sent to the client executing the graph.

    A client resuming the graph must use the [`Command`][langgraph.types.Command]
    primitive to specify a value for the interrupt and continue execution.
    The graph resumes from the start of the node, **re-executing** all logic.

    If a node contains multiple `interrupt` calls, LangGraph matches resume values
    to interrupts based on their order in the node. This list of resume values
    is scoped to the specific task executing the node and is not shared across tasks.

    To use an `interrupt`, you must enable a checkpointer, as the feature relies
    on persisting the graph state.
```

### interrupt() 与 input() 的区别

interrupt() 是 LangGraph 中用于实现 **人机交互** 的一个函数。它的主要作用是 **暂停图的执行**，等待用户输入，然后将用户输入作为反馈继续执行图。

在传统的编程中，input() 是一个阻塞式函数，它会直接等待用户输入，直到用户按下回车键后才继续执行代码。然而，interrupt() 的设计并不是直接阻塞程序，而是通过 **中断图的执行** 来等待用户输入。

### `interrupt`与`Command`结合使用

接下来通过一个完整示例解释`interrupt`与`Command`结合使用完成中断设计。

module3/interrupt.py

```python
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

from langgraph.types import Command, interrupt
from langgraph.checkpoint.memory import MemorySaver
from IPython.display import Image, display

class State(TypedDict):
    input: str
    user_feedback: str

def step_1(state):
    print("---Step 1---")
    pass

def human_feedback(state):
    print("---human_feedback---")
    feedback = interrupt("Please provide feedback:")
    return {"user_feedback": feedback}

def step_3(state):
    print("---Step 3---")
    pass

builder = StateGraph(State)
builder.add_node("step_1", step_1)
builder.add_node("human_feedback", human_feedback)
builder.add_node("step_3", step_3)
builder.add_edge(START, "step_1")
builder.add_edge("step_1", "human_feedback")
builder.add_edge("human_feedback", "step_3")
builder.add_edge("step_3", END)

# Set up memory
memory = MemorySaver()

# Add
graph = builder.compile(checkpointer=memory)

# Input
initial_input = {"input": "hello world"}

# Thread
thread = {"configurable": {"thread_id": "1"}}

# Run the graph until the first interruption
for event in graph.stream(initial_input, thread, stream_mode="updates"):
    print(event)
    print("\n")

# Continue the graph execution
for event in graph.stream(
    Command(resume="go to step 3!"),
    thread,
    stream_mode="updates",
):
    print(event)
    print("\n")

graph.get_state(thread).values
```

![bc6f0bebebd09f4698372814bf62a134.png](/assets/images/bc6f0bebebd09f4698372814bf62a134.png)

- **中断执行**：当 interrupt() 被调用时，它会 **暂停当前节点的执行**，并返回一个中断信号（__interrupt__）。这个中断信号包含了一个提示信息（如“Please provide feedback:”），告诉用户需要提供输入。
- **保存状态**：
  - 在中断执行时，LangGraph 会通过 **检查点机制**（如 MemorySaver）保存当前图的状态。
  - 这样可以确保在用户输入后，图可以从上次中断的地方继续执行。
- **恢复执行**：
  - 用户输入后，通过调用 Command(resume=<user_input>)，将用户输入传递给图。
  - 图会恢复执行，并将用户输入作为反馈传递给中断的节点。

关键代码

```python
def human_feedback(state):
    print("---human_feedback---")
    feedback = interrupt("Please provide feedback:")
    return {"user_feedback": feedback}
```

- 在 human_feedback 函数中，interrupt() 被调用，打印提示信息“Please provide feedback:”。
- 此时，图的执行被暂停，等待用户输入。
- 用户输入后，通过 Command(resume=<user_input>) 恢复执行，并将输入值赋给 user_feedback。

### interrupt() 的核心思想

interrupt() 的核心思想是 **将用户输入的等待过程从图的执行中解耦**。它并不是直接阻塞程序等待用户输入，而是通过以下步骤实现交互：

1. **中断执行**：interrupt() 被调用时，图的执行被暂停，返回一个中断信号，提示需要用户输入。
2. 等待用户输入：这个步骤并不是由interrupt()直接完成的，而是由外部机制（如前端界面、API 调用等）来实现。用户输入可以来自任何渠道，比如：
   - 一个网页表单的提交。
   - 一个移动应用的用户交互。
   - 一个 API 接口接收的用户输入。
   - 甚至可以是程序设计者模拟的输入。
3. **恢复执行**：通过调用 Command(resume=<user_input>)，将用户输入传递给图，恢复执行。

### 关键点

- **灵活性**：interrupt() 并不直接处理用户输入，而是通过外部机制获取用户输入。这意味着开发者可以根据实际需求选择最适合的用户输入方式。
- **解耦**：用户输入的获取和图的执行是解耦的。图的执行可以暂停等待用户输入，而用户输入的获取可以在任何地方、任何时间完成。
- **模拟输入**：Command(resume=<user_input>) 允许开发者模拟用户输入，这意味着可以在测试环境中轻松地测试图的行为，而不需要真正的用户交互。



## NodeInterrupt（不推荐使用）

让我们创建一个图，在该图中，会根据输入的长度抛出 `NodeInterrupt` 异常。

module3/NodeInterrupt.py

```python
from IPython.display import Image, display

from typing_extensions import TypedDict
from langgraph.checkpoint.memory import MemorySaver
from langgraph.errors import NodeInterrupt
from langgraph.graph import START, END, StateGraph

class State(TypedDict):
    input: str

def step_1(state: State) -> State:
    print("---Step 1---")
    return state

def step_2(state: State) -> State:
    # Let's optionally raise a NodeInterrupt if the length of the input is longer than 5 characters
    if len(state['input']) > 5:
        raise NodeInterrupt(f"Received input that is longer than 5 characters: {state['input']}")
    
    print("---Step 2---")
    return state

def step_3(state: State) -> State:
    print("---Step 3---")
    return state

builder = StateGraph(State)
builder.add_node("step_1", step_1)
builder.add_node("step_2", step_2)
builder.add_node("step_3", step_3)
builder.add_edge(START, "step_1")
builder.add_edge("step_1", "step_2")
builder.add_edge("step_2", "step_3")
builder.add_edge("step_3", END)

# Set up memory
memory = MemorySaver()

# Compile the graph with memory
graph = builder.compile(checkpointer=memory)

# View
display(Image(graph.get_graph().draw_mermaid_png()))
```

![截屏2025-04-30 14.37.06.png](/assets/images/截屏2025-04-30%2014.37.06.png)

让我们用一个超过 5 个字符的输入来运行这个图。

```python
initial_input = {"input": "hello world"}
thread_config = {"configurable": {"thread_id": "1"}}

# Run the graph until the first interruption
for event in graph.stream(initial_input, thread_config, stream_mode="values"):
    print(event)
```

```python
{'input': 'hello world'}
---Step 1---
{'input': 'hello world'}
```

如果我们此时查看图的状态，我们就会得到接下来要执行的节点集合（step_2）。

```python
state = graph.get_state(thread_config)
print(state.next)
```

```
('step_2',)
```

我们可以看到，`Interrupt` 被记录到了状态信息中。

```python
print(state.tasks)
```

```
(PregelTask(id='6eb3910d-e231-5ba2-b25e-28ad575690bd', name='step_2', error=None, interrupts=(Interrupt(value='Received input that is longer than 5 characters: hello world', when='during'),), state=None),)
```

我们可以尝试从断点处重新绘制该图。但是，这不过是在重复使用同一个节点！除非改变状态，否则我们就会被困在这里。

```python
for event in graph.stream(None, thread_config, stream_mode="values"):
    print(event)
```

```
{'input': 'hello world'}
```

```
state = graph.get_state(thread_config)
print(state.next)
```

```
('step_2',)
```

现在，我们可以更新状态了。

```python
graph.update_state( thread_config, {"input": "hi"},)
```

```
{'configurable': {'thread_id': '1',
  'checkpoint_ns': '',
  'checkpoint_id': '1ef6a434-06cf-6f1e-8002-0ea6dc69e075'}}
```

```python
for event in graph.stream(None, thread_config, stream_mode="values"):
    print(event)
```

```python
{'input': 'hi'}
---Step 2---
{'input': 'hi'}
---Step 3---
{'input': 'hi'}
```

在某些需要抛出异常来中断执行的场景中，`NodeInterrupt` 作为异常类型可能更符合传统的错误处理模式。

# Time travel

在使用基于模型做出决策的非确定性系统（例如，由 LLMs 驱动的代理）时，详细检查其决策过程可能会很有用

1. 🤔 **理解推理**：分析导致成功结果的步骤。
2. 🐞 **调试错误**：识别错误发生的位置和原因。
3. 🔍 **探索替代方案**：测试不同的路径以发现更好的解决方案。

现在，让我们展示一下 LangGraph 如何支持调试功能，包括查看历史、回放以及从过往状态进行分支操作等操作。我们把这种现象称为时间旅行。

```python
from langchain_openai import ChatOpenAI

def multiply(a: int, b: int) -> int:
    """Multiply a and b.

    Args:
        a: first int
        b: second int
    """
    return a * b

# This will be a tool
def add(a: int, b: int) -> int:
    """Adds a and b.

    Args:
        a: first int
        b: second int
    """
    return a + b

def divide(a: int, b: int) -> float:
    """Divide a by b.

    Args:
        a: first int
        b: second int
    """
    return a / b

tools = [add, multiply, divide]
llm = ChatOpenAI(model="gpt-4o")
llm_with_tools = llm.bind_tools(tools)

from IPython.display import Image, display

from langgraph.checkpoint.memory import MemorySaver
from langgraph.graph import MessagesState
from langgraph.graph import START, END, StateGraph
from langgraph.prebuilt import tools_condition, ToolNode

from langchain_core.messages import AIMessage, HumanMessage, SystemMessage

# System message
sys_msg = SystemMessage(content="You are a helpful assistant tasked with performing arithmetic on a set of inputs.")

# Node
def assistant(state: MessagesState):
   return {"messages": [llm_with_tools.invoke([sys_msg] + state["messages"])]}

# Graph
builder = StateGraph(MessagesState)

# Define nodes: these do the work
builder.add_node("assistant", assistant)
builder.add_node("tools", ToolNode(tools))

# Define edges: these determine the control flow
builder.add_edge(START, "assistant")
builder.add_conditional_edges(
    "assistant",
    # If the latest message (result) from assistant is a tool call -> tools_condition routes to tools
    # If the latest message (result) from assistant is a not a tool call -> tools_condition routes to END
    tools_condition,
)
builder.add_edge("tools", "assistant")

memory = MemorySaver()
graph = builder.compile(checkpointer=MemorySaver())

# Show
display(Image(graph.get_graph(xray=True).draw_mermaid_png()))
```



```py
# Input
initial_input = {"messages": HumanMessage(content="Multiply 2 and 3")}

# Thread
thread = {"configurable": {"thread_id": "1"}}

# Run the graph until the first interruption
for event in graph.stream(initial_input, thread, stream_mode="values"):
    event['messages'][-1].pretty_print()
```

```
================================ Human Message =================================

Multiply 2 and 3
================================== Ai Message ==================================
Tool Calls:
  multiply (call_ikJxMpb777bKMYgmM3d9mYjW)
 Call ID: call_ikJxMpb777bKMYgmM3d9mYjW
  Args:
    a: 2
    b: 3
================================= Tool Message =================================
Name: multiply

6
================================== Ai Message ==================================

The result of multiplying 2 and 3 is 6.
```

## Browsing History浏览历史

我们可以使用 get_state 函数来查看给定线程 ID 对应的图的当前状态！

```py
graph.get_state({'configurable': {'thread_id': '1'}})
```

```python
StateSnapshot(values={'messages': [HumanMessage(content='Multiply 2 and 3', id='4ee8c440-0e4a-47d7-852f-06e2a6c4f84d'), AIMessage(content='', additional_kwargs={'tool_calls': [{'id': 'call_ikJxMpb777bKMYgmM3d9mYjW', 'function': {'arguments': '{"a":2,"b":3}', 'name': 'multiply'}, 'type': 'function'}], 'refusal': None}, response_metadata={'token_usage': {'completion_tokens': 17, 'prompt_tokens': 131, 'total_tokens': 148}, 'model_name': 'gpt-4o-2024-05-13', 'system_fingerprint': 'fp_157b3831f5', 'finish_reason': 'tool_calls', 'logprobs': None}, id='run-bc24d334-8013-4f85-826f-e1ed69c86df0-0', tool_calls=[{'name': 'multiply', 'args': {'a': 2, 'b': 3}, 'id': 'call_ikJxMpb777bKMYgmM3d9mYjW', 'type': 'tool_call'}], usage_metadata={'input_tokens': 131, 'output_tokens': 17, 'total_tokens': 148}), ToolMessage(content='6', name='multiply', id='1012611a-30c5-4732-b789-8c455580c7b4', tool_call_id='call_ikJxMpb777bKMYgmM3d9mYjW'), AIMessage(content='The result of multiplying 2 and 3 is 6.', additional_kwargs={'refusal': None}, response_metadata={'token_usage': {'completion_tokens': 14, 'prompt_tokens': 156, 'total_tokens': 170}, 'model_name': 'gpt-4o-2024-05-13', 'system_fingerprint': 'fp_157b3831f5', 'finish_reason': 'stop', 'logprobs': None}, id='run-b46f3fed-ca3b-4e09-83f4-77ea5071e9bf-0', usage_metadata={'input_tokens': 156, 'output_tokens': 14, 'total_tokens': 170})]}, next=(), config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef6a440-ac9e-6024-8003-6fd8435c1d3b'}}, metadata={'source': 'loop', 'writes': {'assistant': {'messages': [AIMessage(content='The result of multiplying 2 and 3 is 6.', additional_kwargs={'refusal': None}, response_metadata={'token_usage': {'completion_tokens': 14, 'prompt_tokens': 156, 'total_tokens': 170}, 'model_name': 'gpt-4o-2024-05-13', 'system_fingerprint': 'fp_157b3831f5', 'finish_reason': 'stop', 'logprobs': None}, id='run-b46f3fed-ca3b-4e09-83f4-77ea5071e9bf-0', usage_metadata={'input_tokens': 156, 'output_tokens': 14, 'total_tokens': 170})]}}, 'step': 3, 'parents': {}}, created_at='2024-09-03T22:29:54.309727+00:00', parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef6a440-a759-6d02-8002-f1da6393e1ab'}}, tasks=())
```

StateSnapshot是LangGraph持久性架构的一个基本组件。它与checkpointers携手工作，以支持高级工作流功能，如中断、状态更新和分支执行路径。快照包含了理解图中到目前为止发生了什么以及接下来会发生什么所需的所有信息，这使得它对于调试、监控和控制图的执行流程至关重要。

> StateSnapshot被实现为一个包含7个关键字段的命名元组：
>
> `values`：包含图中所有状态通道的当前值
> `next`：计划在下一步中执行的节点名称的元组
> `config`：用于获取此快照的RunnableConfig
> `metadata`：与检查点相关的元数据，包括步骤信息和写
> `created_at`：快照创建时间戳
> `parent_config`：用于获取父快照的配置（如果有的话）。
> `tasks`（任务）：PregelTask对象的元组，表示这一步要执行的任务，如果之前尝试过，可能会包含错误信息



我们还可以浏览代理的状态历史记录。`get_state_history` 方法让我们能够获取所有先前步骤的状态。

```py
all_states = [s for s in graph.get_state_history(thread)]
len(all_states)
```

5

第一个要素就是当前状态，正如我们从 `get_state` 函数中获取到的那样。

```
all_states[-2]
```

```python
StateSnapshot(values={'messages': [HumanMessage(content='Multiply 2 and 3', id='4ee8c440-0e4a-47d7-852f-06e2a6c4f84d')]}, next=('assistant',), config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef6a440-a003-6c74-8000-8a2d82b0d126'}}, metadata={'source': 'loop', 'writes': None, 'step': 0, 'parents': {}}, created_at='2024-09-03T22:29:52.988265+00:00', parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef6a440-9ffe-6512-bfff-9e6d8dc24bba'}}, tasks=(PregelTask(id='ca669906-0c4f-5165-840d-7a6a3fce9fb9', name='assistant', error=None, interrupts=(), state=None),))
```

上述所有内容我们都能够可视化出来：

![fdb72c07ddb6fb0570fc7c6de4c36e95.png](/assets/images/fdb72c07ddb6fb0570fc7c6de4c36e95.png)

## Replaying回放

我们可以从之前的任何步骤重新启动我们的代理程序。

![de6f010423cb95c57143d3658ce8e833.png](/assets/images/de6f010423cb95c57143d3658ce8e833.png)

让我们回顾一下接收人类输入的那一环节！

```python
to_replay = all_states[-2]
```

```
StateSnapshot(values={'messages': [HumanMessage(content='Multiply 2 and 3', id='4ee8c440-0e4a-47d7-852f-06e2a6c4f84d')]}, next=('assistant',), config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef6a440-a003-6c74-8000-8a2d82b0d126'}}, metadata={'source': 'loop', 'writes': None, 'step': 0, 'parents': {}}, created_at='2024-09-03T22:29:52.988265+00:00', parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef6a440-9ffe-6512-bfff-9e6d8dc24bba'}}, tasks=(PregelTask(id='ca669906-0c4f-5165-840d-7a6a3fce9fb9', name='assistant', error=None, interrupts=(), state=None),))
```

```python
to_replay.values
```

```
{'messages': [HumanMessage(content='Multiply 2 and 3', id='4ee8c440-0e4a-47d7-852f-06e2a6c4f84d')]}
```

下一个调用node


```python
to_replay.next
```


    ('assistant',)

我们还获取到了配置信息，其中包含了 `checkpoint_id` 以及 `thread_id` 这两个参数。


```python
to_replay.config
```


    {'configurable': {'thread_id': '1',
      'checkpoint_ns': '',
      'checkpoint_id': '1ef6a440-a003-6c74-8000-8a2d82b0d126'}}

要从这里重新播放，我们只需将配置信息传递给代理即可！该图知晓此检查点已执行过。它就从这个检查点重新开始播放！


```python
for event in graph.stream(None, to_replay.config, stream_mode="values"):
    event['messages'][-1].pretty_print()
```

    ================================ Human Message =================================
    
    Multiply 2 and 3
    ================================== Ai Message ==================================
    Tool Calls:
      multiply (call_SABfB57CnDkMu9HJeUE0mvJ9)
     Call ID: call_SABfB57CnDkMu9HJeUE0mvJ9
      Args:
        a: 2
        b: 3
    ================================= Tool Message =================================
    Name: multiply
    
    6
    ================================== Ai Message ==================================
    
    The result of multiplying 2 and 3 is 6.


现在，我们可以看到在代理重新运行之后我们的当前状况。

## Forking 分支
要是我们想从同一个步骤开始操作，但这次使用不同的输入数据会怎样呢？

![0110f4bfea74e82b64cce58f38906133.png](/assets/images/0110f4bfea74e82b64cce58f38906133.png)

```python
to_fork = all_states[-2]
to_fork.values["messages"]
```


    [HumanMessage(content='Multiply 2 and 3', id='4ee8c440-0e4a-47d7-852f-06e2a6c4f84d')]

查看config


```python
to_fork.config
```


    {'configurable': {'thread_id': '1',
      'checkpoint_ns': '',
      'checkpoint_id': '1ef6a440-a003-6c74-8000-8a2d82b0d126'}}

让我们在这个检查点对状态进行修改。
我们可以直接使用传入的 `checkpoint_id` 来运行 `update_state` 函数。
还记得我们在 `messages` 上面的reducer是如何运作的吗：

- 如果我们不提供消息 ID，它就会追加（内容）。
- 我们提供消息 ID 是为了覆盖该消息内容，而不是对其进行追加！（以更新状态）！

所以，要覆盖这条消息，我们只需提供消息 ID，即我们有 `to_fork.values["messages"].id` 这个值。


```python
fork_config = graph.update_state(
    to_fork.config,
    {"messages": [HumanMessage(content='Multiply 5 and 3', 
                               id=to_fork.values["messages"][0].id)]},
)
fork_config
```


```python
{'configurable': {'thread_id': '1',
  'checkpoint_ns': '',
  'checkpoint_id': '1ef6a442-3661-62f6-8001-d3c01b96f98b'}}
```

这会创建一个新的、分支式的检查点。
但是，元数据——比如接下来该去哪里——是得以保留下来的！
我们可以看到，我们的代理程序的当前状态已通过我们的分支进行了更新。


```python
all_states = [state for state in graph.get_state_history(thread) ]
all_states[0].values["messages"]
```


    [HumanMessage(content='Multiply 5 and 3', id='4ee8c440-0e4a-47d7-852f-06e2a6c4f84d')]


```python
graph.get_state({'configurable': {'thread_id': '1'}})
```


```python
StateSnapshot(values={'messages': [HumanMessage(content='Multiply 5 and 3', id='4ee8c440-0e4a-47d7-852f-06e2a6c4f84d')]}, next=('assistant',), config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef6a442-3661-62f6-8001-d3c01b96f98b'}}, metadata={'source': 'update', 'step': 1, 'writes': {'__start__': {'messages': [HumanMessage(content='Multiply 5 and 3', id='4ee8c440-0e4a-47d7-852f-06e2a6c4f84d')]}}, 'parents': {}}, created_at='2024-09-03T22:30:35.598707+00:00', parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef6a440-a003-6c74-8000-8a2d82b0d126'}}, tasks=(PregelTask(id='f8990132-a8d3-5ddd-8d9e-1efbfc220da1', name='assistant', error=None, interrupts=(), state=None),))
```

现在，在我们进行流式传输时，该图就会知道这个检查点从未被执行过。
所以，这个过程是持续进行的，而非只是简单地重复播放。


```python
for event in graph.stream(None, fork_config, stream_mode="values"):
    event['messages'][-1].pretty_print()
```

    ================================ Human Message =================================
    
    Multiply 5 and 3
    ================================== Ai Message ==================================
    Tool Calls:
      multiply (call_KP2CVNMMUKMJAQuFmamHB21r)
     Call ID: call_KP2CVNMMUKMJAQuFmamHB21r
      Args:
        a: 5
        b: 3
    ================================= Tool Message =================================
    Name: multiply
    
    15
    ================================== Ai Message ==================================
    
    The result of multiplying 5 and 3 is 15.


现在，我们可以看到当前状态是我们的代理程序运行的结束阶段。


```python
graph.get_state({'configurable': {'thread_id': '1'}})
```


```python
StateSnapshot(values={'messages': [HumanMessage(content='Multiply 5 and 3', id='4ee8c440-0e4a-47d7-852f-06e2a6c4f84d'), AIMessage(content='', additional_kwargs={'tool_calls': [{'id': 'call_KP2CVNMMUKMJAQuFmamHB21r', 'function': {'arguments': '{"a":5,"b":3}', 'name': 'multiply'}, 'type': 'function'}], 'refusal': None}, response_metadata={'token_usage': {'completion_tokens': 17, 'prompt_tokens': 131, 'total_tokens': 148}, 'model_name': 'gpt-4o-2024-05-13', 'system_fingerprint': 'fp_157b3831f5', 'finish_reason': 'tool_calls', 'logprobs': None}, id='run-bc420009-d1f6-49b8-bea7-dfc9fca7eb79-0', tool_calls=[{'name': 'multiply', 'args': {'a': 5, 'b': 3}, 'id': 'call_KP2CVNMMUKMJAQuFmamHB21r', 'type': 'tool_call'}], usage_metadata={'input_tokens': 131, 'output_tokens': 17, 'total_tokens': 148}), ToolMessage(content='15', name='multiply', id='9232e653-816d-471a-9002-9a1ecd453364', tool_call_id='call_KP2CVNMMUKMJAQuFmamHB21r'), AIMessage(content='The result of multiplying 5 and 3 is 15.', additional_kwargs={'refusal': None}, response_metadata={'token_usage': {'completion_tokens': 14, 'prompt_tokens': 156, 'total_tokens': 170}, 'model_name': 'gpt-4o-2024-05-13', 'system_fingerprint': 'fp_157b3831f5', 'finish_reason': 'stop', 'logprobs': None}, id='run-86c21888-d832-47c5-9e76-0aa2676116dc-0', usage_metadata={'input_tokens': 156, 'output_tokens': 14, 'total_tokens': 170})]}, next=(), config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef6a442-a2e2-6e98-8004-4a0b75537950'}}, metadata={'source': 'loop', 'writes': {'assistant': {'messages': [AIMessage(content='The result of multiplying 5 and 3 is 15.', additional_kwargs={'refusal': None}, response_metadata={'token_usage': {'completion_tokens': 14, 'prompt_tokens': 156, 'total_tokens': 170}, 'model_name': 'gpt-4o-2024-05-13', 'system_fingerprint': 'fp_157b3831f5', 'finish_reason': 'stop', 'logprobs': None}, id='run-86c21888-d832-47c5-9e76-0aa2676116dc-0', usage_metadata={'input_tokens': 156, 'output_tokens': 14, 'total_tokens': 170})]}}, 'step': 4, 'parents': {}}, created_at='2024-09-03T22:30:46.976463+00:00', parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef6a442-9db0-6056-8003-7304cab7bed8'}}, tasks=())
```

