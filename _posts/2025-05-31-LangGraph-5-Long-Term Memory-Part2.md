---
title: LangGraph-5-Long-Term Memory-Part2
category: aigc
tags:
  - agent
  - ai
  - langgraph
  - python
---

# Chatbot with Collection Schema 

## Defining a collection schema

我们将创建一个灵活的集合模式来存储关于用户交互的记忆，而不是将用户信息存储在固定的配置文件结构中。

每个记忆将被存储为一个单独的条目，其中只有一个内容字段，用于存储我们想要记住的主要信息

这种方法允许我们建立一个开放式的记忆集合，当我们对用户了解更多时，这些记忆可以增长和改变。

我们可以将集合模式定义为Pydantic对象。

```python
from pydantic import BaseModel, Field

class Memory(BaseModel):
    content: str = Field(description="The main content of the memory. For example: User expressed interest in learning about French.")

class MemoryCollection(BaseModel):
    memories: list[Memory] = Field(description="A list of memories about the user.")
    
from langchain_core.messages import HumanMessage
from langchain_openai import ChatOpenAI

# Initialize the model
model = ChatOpenAI(model="gpt-4o", temperature=0)

# Bind schema to model
model_with_structure = model.with_structured_output(MemoryCollection)

# Invoke the model to produce structured output that matches the schema
memory_collection = model_with_structure.invoke([HumanMessage("My name is Lance. I like to bike.")])
memory_collection.memories
```

我们可以使用`model_dump（）`将Pydantic模型实例序列化为Python字典。

```
memory_collection.memories[0].model_dump()
```

将每个记忆的字典表示保存到store中。

```python
import uuid
from langgraph.store.memory import InMemoryStore

# Initialize the in-memory store
in_memory_store = InMemoryStore()

# Namespace for the memory to save
user_id = "1"
namespace_for_memory = (user_id, "memories")

# Save a memory to namespace as key and value
key = str(uuid.uuid4())
value = memory_collection.memories[0].model_dump()
in_memory_store.put(namespace_for_memory, key, value)

key = str(uuid.uuid4())
value = memory_collection.memories[1].model_dump()
in_memory_store.put(namespace_for_memory, key, value)
```

Search for memories in the store. 

```python
# Search 
for m in in_memory_store.search(namespace_for_memory):
    print(m.dict())
```

```
{'value': {'content': "User's name is Lance."}, 'key': 'e1c4e5ab-ab0f-4cbb-822d-f29240a983af', 'namespace': ['1', 'memories'], 'created_at': '2024-10-30T21:43:26.893775+00:00', 'updated_at': '2024-10-30T21:43:26.893779+00:00'}
{'value': {'content': 'Lance likes to bike.'}, 'key': 'e132a1ea-6202-43ac-a9a6-3ecf2c1780a8', 'namespace': ['1', 'memories'], 'created_at': '2024-10-30T21:43:26.893833+00:00', 'updated_at': '2024-10-30T21:43:26.893834+00:00'}
```

完整可运行代码module5/memoryschema_collection.py

```python
import sys
import os
sys.path.append(os.path.dirname(os.path.dirname(os.path.abspath(__file__))))

from pydantic import BaseModel, Field

class Memory(BaseModel):
    content: str = Field(description="The main content of the memory. For example: User expressed interest in learning about French.")

class MemoryCollection(BaseModel):
    memories: list[Memory] = Field(description="A list of memories about the user.")
    
from langchain_core.messages import HumanMessage, SystemMessage
from langchain_openai import ChatOpenAI

# Initialize the model
import common.customModel
model = common.customModel.volcengine_doubao15pro32k()

# Bind schema to model
model_with_structure = model.with_structured_output(MemoryCollection)

# Create a system message to guide the model
system_message = SystemMessage(content="""You are a memory collection system. 
Your task is to extract and store memories about the user in a structured format.
Please output the memories in the following JSON format:
{
    "memories": [
        {
            "content": "memory content here"
        }
    ]
}""")

# Invoke the model to produce structured output that matches the schema
memory_collection = model_with_structure.invoke([
    system_message,
    HumanMessage("My name is Lance. I like to bike.")
])
memories = memory_collection.memories

from common.log import debug,info,error
debug("memories:{}\n".format(memories))

model_dump = memory_collection.memories[0].model_dump()
debug(f"model_dump:{model_dump}")

import uuid
from langgraph.store.memory import InMemoryStore
from typing import List, Dict

# Initialize the in-memory store
in_memory_store = InMemoryStore()

# Namespace for the memory to save
user_id = "1"
namespace_for_memory = (user_id, "memories")

def batch_store_memories(memories: List[Memory], namespace: tuple) -> None:
    """
    批量存储记忆到内存存储中
    
    Args:
        memories: Memory对象列表
        namespace: 存储的命名空间
    """
    # 为每个记忆创建唯一的键并存储
    for memory in memories:
        key = str(uuid.uuid4())
        value = memory.model_dump()
        in_memory_store.put(namespace, key, value)

# 使用批量存储方法
batch_store_memories(memory_collection.memories, namespace_for_memory)

# 验证存储结果
# Search 
for m in in_memory_store.search(namespace_for_memory):
    print(m.dict())
```

输出

```
[2025-05-20 14:47:14] [DEBUG] [/Users/kaijin/Documents/AI_ML/workspace/langgraph/langchain-academy/lc-academy-env/common/log.py:debug] memories:[Memory(content="The user's name is Lance."), Memory(content='The user likes to bike.')]

[2025-05-20 14:47:14] [DEBUG] [/Users/kaijin/Documents/AI_ML/workspace/langgraph/langchain-academy/lc-academy-env/common/log.py:debug] model_dump:{'content': "The user's name is Lance."}
{'namespace': ['1', 'memories'], 'key': '0f8fae54-8d1a-4413-a051-b1994a626ec7', 'value': {'content': "The user's name is Lance."}, 'created_at': '2025-05-20T06:47:14.836827+00:00', 'updated_at': '2025-05-20T06:47:14.836836+00:00', 'score': None}
{'namespace': ['1', 'memories'], 'key': '062a7d64-fb0e-4c3d-89cf-2baa911e14b9', 'value': {'content': 'The user likes to bike.'}, 'created_at': '2025-05-20T06:47:14.836930+00:00', 'updated_at': '2025-05-20T06:47:14.836933+00:00', 'score': None}
```

## Updating collection schema

上一课我们讨论了更新概要模式的挑战。集合也是如此！

我们希望能够用新的内存更新集合，也希望能够更新集合中现有的内存。

现在我们将展示[Trustcall](https://github.com/hinthornw/trustcall)也可以用于更新集合。这既可以添加新的记忆，也可以[更新集合中现有的记忆](https://github.com/hinthornw/trustcall?tab=readme-ov-file#simultanous-updates--insertions).

让我们用`Trustcall`定义一个新的提取器。和之前一样，我们为每个内存提供模式` memory `。

但是，我们可以提供`enable_inserts=True`来允许`extractor`向集合中插入新的内存。

module5/schema_collection_update.py

```python
from trustcall import create_extractor

# Create the extractor
trustcall_extractor = create_extractor(
    model,
    tools=[Memory],
    tool_choice="Memory",
    enable_inserts=True,
)

from langchain_core.messages import HumanMessage, SystemMessage, AIMessage

# Instruction
instruction = """Extract memories from the following conversation:"""

# Conversation
conversation = [HumanMessage(content="Hi, I'm Lance."), 
                AIMessage(content="Nice to meet you, Lance."), 
                HumanMessage(content="This morning I had a nice bike ride in San Francisco.")]

# Invoke the extractor
result = trustcall_extractor.invoke({"messages": [SystemMessage(content=instruction)] + conversation})

# Messages contain the tool calls
for m in result["messages"]:
    m.pretty_print()
```

```
================================== Ai Message ==================================
Tool Calls:
  Memory (call_Pj4kctFlpg9TgcMBfMH33N30)
 Call ID: call_Pj4kctFlpg9TgcMBfMH33N30
  Args:
    content: Lance had a nice bike ride in San Francisco this morning.
```

```python
# Responses contain the memories that adhere to the schema
for m in result["responses"]: 
    print(m)
```

```
content='Lance had a nice bike ride in San Francisco this morning.'
```

```python
# Metadata contains the tool call  
for m in result["response_metadata"]: 
    print(m)
```

```
{'id': 'call_Pj4kctFlpg9TgcMBfMH33N30'}
```

```python
# Update the conversation
updated_conversation = [AIMessage(content="That's great, did you do after?"), 
                        HumanMessage(content="I went to Tartine and ate a croissant."),                        
                        AIMessage(content="What else is on your mind?"),
                        HumanMessage(content="I was thinking about my Japan, and going back this winter!"),]

# Update the instruction
system_msg = """Update existing memories and create new ones based on the following conversation:"""

# We'll save existing memories, giving them an ID, key (tool name), and value
tool_name = "Memory"
existing_memories = [(str(i), tool_name, memory.model_dump()) for i, memory in enumerate(result["responses"])] if result["responses"] else None
existing_memories
```

```
[('0',
  'Memory',
  {'content': 'Lance had a nice bike ride in San Francisco this morning.'})]
```

```python
# Invoke the extractor with our updated conversation and existing memories
result = trustcall_extractor.invoke({"messages": updated_conversation, 
                                     "existing": existing_memories})
```

```
================================== Ai Message ==================================
Tool Calls:
  Memory (call_vxks0YH1hwUxkghv4f5zdkTr)
 Call ID: call_vxks0YH1hwUxkghv4f5zdkTr
  Args:
    content: Lance had a nice bike ride in San Francisco this morning. He went to Tartine and ate a croissant. He was thinking about his trip to Japan and going back this winter!
  Memory (call_Y4S3poQgFmDfPy2ExPaMRk8g)
 Call ID: call_Y4S3poQgFmDfPy2ExPaMRk8g
  Args:
    content: Lance went to Tartine and ate a croissant. He was thinking about his trip to Japan and going back this winter!
```

```python
# Responses contain the memories that adhere to the schema
for m in result["responses"]: 
    print(m)
```

```
content='Lance had a nice bike ride in San Francisco this morning. He went to Tartine and ate a croissant. He was thinking about his trip to Japan and going back this winter!'
content='Lance went to Tartine and ate a croissant. He was thinking about his trip to Japan and going back this winter!'
```

这告诉我们，我们通过指定`json_doc_id`更新了集合中的第一个内存。

```python
# Metadata contains the tool call  
for m in result["response_metadata"]: 
    print(m)
```

```
{'id': 'call_vxks0YH1hwUxkghv4f5zdkTr', 'json_doc_id': '0'}
{'id': 'call_Y4S3poQgFmDfPy2ExPaMRk8g'}
```

###  `enable_inserts=True` 参数的作用

`enable_inserts=True` 参数在 `trustcall_extractor` 中的作用是允许在更新记忆时插入新的记忆。

1. 当 `enable_inserts=True` 时：

   - 系统可以在更新现有记忆的同时，创建新的记忆

   - 从代码输出可以看到，系统能够同时处理多个记忆，比如：

     ```
     content='User Lance said he had a nice bike ride in San Francisco this morning.'
     content=' User Lance also said he was thinking about Japan and going back this winter!'
     ```

   - 每个新的记忆都会被赋予一个唯一的 ID（如 `call_agg8qbxx4p46pulnduumsbrz`）

2. 当 `enable_inserts=False` 时（默认值）：

   - 系统只能更新现有的记忆
   - 不能创建新的记忆
   - 所有的信息都会被合并到现有的记忆中

这个参数的主要用途是在处理对话记忆时，允许系统：

1. 保持对话的连续性
2. 分别存储不同的记忆片段
3. 在需要时能够独立访问和更新特定的记忆

在示例代码中，我们可以看到它被用于：

- 首先提取用户骑自行车的记忆
- 然后添加用户想去日本的记忆
- 每个记忆都被单独存储和管理

这种设计使得记忆系统更加灵活和可管理，特别是在处理长期对话或需要分别追踪不同主题的场景中。

### 使用 `enable_inserts=True` 的场景：

1. **多主题对话场景**

   - 当用户在一个对话中讨论多个不同的主题时，例如：

     ```
     用户：我喜欢骑自行车
     用户：我计划去日本旅行
     ```

   - 这种情况下，应该分别创建两个记忆，因为这是两个独立的主题

2. **时间跨度较大的信息**

   - 当用户提供的信息跨越不同的时间段，例如：

     ```
     用户：我昨天去了健身房
     用户：我下周要去参加一个会议
     ```

   - 这些信息应该分开存储，因为它们属于不同的时间点

3. **需要独立追踪的信息**

   - 当某些信息需要单独查询或更新时，例如：

     ```
     用户：我的工作地址是XX
     用户：我的家庭住址是YY
     ```

   - 这些地址信息可能需要分别更新，所以应该分开存储

4. **情感分析或用户偏好**

   - 当需要分别追踪用户的不同偏好或情感表达，例如：

     ```
     用户：我喜欢吃甜食
     用户：我不喜欢运动
     ```

   - 这些偏好应该分开存储，以便于后续的个性化推荐

### 在代码中自动区分多主题对话

#### 1. 基于规则的主题分类方案

```python
from enum import Enum
from typing import List, Optional
from pydantic import BaseModel, Field

class TopicType(Enum):
    TRAVEL = "travel"
    FOOD = "food"
    SPORTS = "sports"
    WORK = "work"
    PERSONAL = "personal"
    OTHER = "other"

class Memory(BaseModel):
    content: str = Field(description="The main content of the memory")
    topic: TopicType = Field(description="The topic category of the memory")
    confidence: float = Field(description="Confidence score of topic classification", default=0.0)
    
    @classmethod
    def should_create_new_memory(cls, current_memory: Optional['Memory'], new_content: str) -> bool:
        if not current_memory:
            return True
            
        # 如果主题不同，创建新记忆
        if current_memory.topic != cls.classify_topic(new_content):
            return True
            
        # 如果主题相同但时间跨度大，创建新记忆
        if cls.has_time_gap(current_memory.content, new_content):
            return True
            
        return False
    
    @staticmethod
    def classify_topic(content: str) -> TopicType:
        # 使用关键词匹配进行主题分类
        travel_keywords = ["travel", "trip", "visit", "go to", "flight"]
        food_keywords = ["eat", "food", "restaurant", "cook", "meal"]
        sports_keywords = ["sport", "exercise", "workout", "gym", "run"]
        
        content_lower = content.lower()
        
        if any(keyword in content_lower for keyword in travel_keywords):
            return TopicType.TRAVEL
        if any(keyword in content_lower for keyword in food_keywords):
            return TopicType.FOOD
        if any(keyword in content_lower for keyword in sports_keywords):
            return TopicType.SPORTS
            
        return TopicType.OTHER
    
    @staticmethod
    def has_time_gap(content1: str, content2: str) -> bool:
        # 简单的时间词检测
        time_words = ["yesterday", "today", "tomorrow", "next week", "last week"]
        content1_has_time = any(word in content1.lower() for word in time_words)
        content2_has_time = any(word in content2.lower() for word in time_words)
        
        return content1_has_time and content2_has_time
```

#### 2. 使用 LLM 进行主题分类

```python
from langchain_openai import ChatOpenAI
from pydantic import BaseModel, Field

class Memory(BaseModel):
    content: str = Field(description="The main content of the memory")
    topic: str = Field(description="The topic category of the memory")
    confidence: float = Field(description="Confidence score of topic classification", default=0.0)

class MemoryManager:
    def __init__(self):
        self.llm = ChatOpenAI()
        self.memories: List[Memory] = []
    
    def should_create_new_memory(self, new_content: str) -> bool:
        if not self.memories:
            return True
            
        # 使用 LLM 判断是否应该创建新记忆
        prompt = f"""
        判断以下新内容是否应该创建新的记忆，还是应该更新现有记忆。
        
        现有记忆:
        {[m.content for m in self.memories]}
        
        新内容:
        {new_content}
        
        请回答 'new' 或 'update'，并给出理由。
        """
        
        response = self.llm.invoke(prompt)
        return "new" in response.lower()
    
    def classify_topic(self, content: str) -> str:
        prompt = f"""
        请将以下内容分类到一个主题中：
        {content}
        
        请只返回主题名称，不要其他解释。
        """
        
        return self.llm.invoke(prompt).strip()
```

#### 3. 混合方案（推荐）

```python
from typing import List, Optional, Tuple
from pydantic import BaseModel, Field
from langchain_openai import ChatOpenAI
import numpy as np
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.metrics.pairwise import cosine_similarity

class Memory(BaseModel):
    content: str = Field(description="The main content of the memory")
    topic: str = Field(description="The topic category of the memory")
    embedding: List[float] = Field(description="Content embedding vector")
    timestamp: float = Field(description="Creation timestamp")
    
class MemoryManager:
    def __init__(self):
        self.llm = ChatOpenAI()
        self.memories: List[Memory] = []
        self.vectorizer = TfidfVectorizer()
        
    def should_create_new_memory(self, new_content: str) -> Tuple[bool, float]:
        if not self.memories:
            return True, 1.0
            
        # 1. 计算语义相似度
        similarity_scores = self._calculate_similarity(new_content)
        max_similarity = max(similarity_scores)
        
        # 2. 使用规则进行初步判断
        if self._has_time_gap(new_content):
            return True, 1.0
            
        # 3. 如果相似度在中间范围，使用 LLM 进行判断
        if 0.3 <= max_similarity <= 0.7:
            return self._llm_judge(new_content), max_similarity
            
        # 4. 根据相似度阈值决定
        return max_similarity < 0.3, max_similarity
    
    def _calculate_similarity(self, new_content: str) -> List[float]:
        # 使用 TF-IDF 计算相似度
        all_contents = [m.content for m in self.memories] + [new_content]
        tfidf_matrix = self.vectorizer.fit_transform(all_contents)
        similarities = cosine_similarity(tfidf_matrix[-1:], tfidf_matrix[:-1])[0]
        return similarities.tolist()
    
    def _has_time_gap(self, content: str) -> bool:
        time_words = ["yesterday", "today", "tomorrow", "next week", "last week"]
        return any(word in content.lower() for word in time_words)
    
    def _llm_judge(self, new_content: str) -> bool:
        prompt = f"""
        判断以下新内容是否应该创建新的记忆，还是应该更新现有记忆。
        
        现有记忆:
        {[m.content for m in self.memories]}
        
        新内容:
        {new_content}
        
        请回答 'new' 或 'update'，并给出理由。
        """
        
        response = self.llm.invoke(prompt)
        return "new" in response.lower()
    
    def add_memory(self, content: str):
        should_create, confidence = self.should_create_new_memory(content)
        
        if should_create:
            # 创建新记忆
            new_memory = Memory(
                content=content,
                topic=self._classify_topic(content),
                embedding=self._get_embedding(content),
                timestamp=time.time()
            )
            self.memories.append(new_memory)
        else:
            # 更新最相似的记忆
            similarity_scores = self._calculate_similarity(content)
            most_similar_idx = np.argmax(similarity_scores)
            self.memories[most_similar_idx].content += f"\n{content}"
```

#### 使用建议：

1. **选择合适的方案**：
   - 对于简单场景，使用基于规则的方案
   - 对于复杂场景，使用混合方案
   - 对于高精度要求，使用纯 LLM 方案

2. **设置合理的阈值**：
   - 相似度阈值（如 0.3）
   - 时间间隔阈值
   - 主题分类的置信度阈值

3. **优化性能**：
   - 使用缓存存储 embedding
   - 批量处理相似度计算
   - 定期清理过期记忆

4. **监控和调整**：
   - 记录分类准确率
   - 收集错误案例
   - 定期更新规则和阈值

5. **处理边界情况**：
   - 处理模糊的主题
   - 处理时间相关的表达
   - 处理多主题混合的情况

这个混合方案结合了规则、语义相似度和 LLM 的优点，能够更准确地判断是否需要创建新记忆。同时，它也具有较好的可扩展性和可维护性。

## Chatbot with collection schema updating

现在，让我们将Trustcall引入到我们的聊天机器人中，以创建和更新内存集合。

```python
from IPython.display import Image, display

import uuid

from langgraph.graph import StateGraph, MessagesState, START, END
from langgraph.store.memory import InMemoryStore
from langchain_core.messages import merge_message_runs
from langchain_core.messages import HumanMessage, SystemMessage
from langchain_core.runnables.config import RunnableConfig
from langgraph.checkpoint.memory import MemorySaver
from langgraph.store.base import BaseStore

# Initialize the model
model = ChatOpenAI(model="gpt-4o", temperature=0)

# Memory schema
class Memory(BaseModel):
    content: str = Field(description="The main content of the memory. For example: User expressed interest in learning about French.")

# Create the Trustcall extractor
trustcall_extractor = create_extractor(
    model,
    tools=[Memory],
    tool_choice="Memory",
    # This allows the extractor to insert new memories
    enable_inserts=True,
)

# Chatbot instruction
MODEL_SYSTEM_MESSAGE = """You are a helpful chatbot. You are designed to be a companion to a user. 

You have a long term memory which keeps track of information you learn about the user over time.

Current Memory (may include updated memories from this conversation): 

{memory}"""

# Trustcall instruction
TRUSTCALL_INSTRUCTION = """Reflect on following interaction. 

Use the provided tools to retain any necessary memories about the user. 

Use parallel tool calling to handle updates and insertions simultaneously:"""

def call_model(state: MessagesState, config: RunnableConfig, store: BaseStore):

    """Load memories from the store and use them to personalize the chatbot's response."""
    
    # Get the user ID from the config
    user_id = config["configurable"]["user_id"]

    # Retrieve memory from the store
    namespace = ("memories", user_id)
    memories = store.search(namespace)

    # Format the memories for the system prompt
    info = "\n".join(f"- {mem.value['content']}" for mem in memories)
    system_msg = MODEL_SYSTEM_MESSAGE.format(memory=info)

    # Respond using memory as well as the chat history
    response = model.invoke([SystemMessage(content=system_msg)]+state["messages"])

    return {"messages": response}

def write_memory(state: MessagesState, config: RunnableConfig, store: BaseStore):

    """Reflect on the chat history and update the memory collection."""
    
    # Get the user ID from the config
    user_id = config["configurable"]["user_id"]

    # Define the namespace for the memories
    namespace = ("memories", user_id)

    # Retrieve the most recent memories for context
    existing_items = store.search(namespace)

    # Format the existing memories for the Trustcall extractor
    tool_name = "Memory"
    existing_memories = ([(existing_item.key, tool_name, existing_item.value)
                          for existing_item in existing_items]
                          if existing_items
                          else None
                        )

    # Merge the chat history and the instruction
    updated_messages=list(merge_message_runs(messages=[SystemMessage(content=TRUSTCALL_INSTRUCTION)] + state["messages"]))

    # Invoke the extractor
    result = trustcall_extractor.invoke({"messages": updated_messages, 
                                        "existing": existing_memories})

    # Save the memories from Trustcall to the store
    for r, rmeta in zip(result["responses"], result["response_metadata"]):
        store.put(namespace,
                  rmeta.get("json_doc_id", str(uuid.uuid4())),
                  r.model_dump(mode="json"),
            )

# Define the graph
builder = StateGraph(MessagesState)
builder.add_node("call_model", call_model)
builder.add_node("write_memory", write_memory)
builder.add_edge(START, "call_model")
builder.add_edge("call_model", "write_memory")
builder.add_edge("write_memory", END)

# Store for long-term (across-thread) memory
across_thread_memory = InMemoryStore()

# Checkpointer for short-term (within-thread) memory
within_thread_memory = MemorySaver()

# Compile the graph with the checkpointer fir and store
graph = builder.compile(checkpointer=within_thread_memory, store=across_thread_memory)

# View
display(Image(graph.get_graph(xray=1).draw_mermaid_png()))

# We supply a thread ID for short-term (within-thread) memory
# We supply a user ID for long-term (across-thread) memory 
config = {"configurable": {"thread_id": "1", "user_id": "1"}}

# User input 
input_messages = [HumanMessage(content="Hi, my name is Lance")]

# Run the graph
for chunk in graph.stream({"messages": input_messages}, config, stream_mode="values"):
    chunk["messages"][-1].pretty_print()
```

```
================================ Human Message =================================

Hi, my name is Lance
================================== Ai Message ==================================

Hi Lance! It's great to meet you. How can I assist you today?
```

```python
# User input 
input_messages = [HumanMessage(content="I like to bike around San Francisco")]

# Run the graph
for chunk in graph.stream({"messages": input_messages}, config, stream_mode="values"):
    chunk["messages"][-1].pretty_print()
```

```
================================ Human Message =================================

I like to bike around San Francisco
================================== Ai Message ==================================

That sounds like a lot of fun! San Francisco has some beautiful routes for biking. Do you have a favorite trail or area you like to explore?
```

```py
# Namespace for the memory to save
user_id = "1"
namespace = ("memories", user_id)
memories = across_thread_memory.search(namespace)
for m in memories:
    print(m.dict())
```

```
{'value': {'content': "User's name is Lance."}, 'key': 'dee65880-dd7d-4184-8ca1-1f7400f7596b', 'namespace': ['memories', '1'], 'created_at': '2024-10-30T22:18:52.413283+00:00', 'updated_at': '2024-10-30T22:18:52.413284+00:00'}
{'value': {'content': 'User likes to bike around San Francisco.'}, 'key': '662195fc-8ea4-4f64-a6b6-6b86d9cb85c0', 'namespace': ['memories', '1'], 'created_at': '2024-10-30T22:18:56.597813+00:00', 'updated_at': '2024-10-30T22:18:56.597814+00:00'}
```

```python
# User input 
input_messages = [HumanMessage(content="I also enjoy going to bakeries")]

# Run the graph
for chunk in graph.stream({"messages": input_messages}, config, stream_mode="values"):
    chunk["messages"][-1].pretty_print()
```

```
================================ Human Message =================================

I also enjoy going to bakeries
================================== Ai Message ==================================

Biking and bakeries make a great combination! Do you have a favorite bakery in San Francisco, or are you on the hunt for new ones to try?
```

用新的话题继续对话。

```python
# We supply a thread ID for short-term (within-thread) memory
# We supply a user ID for long-term (across-thread) memory 
config = {"configurable": {"thread_id": "2", "user_id": "1"}}

# User input 
input_messages = [HumanMessage(content="What bakeries do you recommend for me?")]

# Run the graph
for chunk in graph.stream({"messages": input_messages}, config, stream_mode="values"):
    chunk["messages"][-1].pretty_print()
```

```
================================ Human Message =================================

What bakeries do you recommend for me?
================================== Ai Message ==================================

Since you enjoy biking around San Francisco, you might like to check out some of these bakeries that are both delicious and located in areas that are great for a bike ride:

1. **Tartine Bakery** - Located in the Mission District, it's famous for its bread and pastries. The area is vibrant and perfect for a leisurely ride.

2. **Arsicault Bakery** - Known for its incredible croissants, it's in the Richmond District, which offers a nice ride through Golden Gate Park.

3. **B. Patisserie** - Situated in Lower Pacific Heights, this bakery is renowned for its kouign-amann and other French pastries. The neighborhood is charming and bike-friendly.

4. **Mr. Holmes Bakehouse** - Famous for its cruffins, it's located in the Tenderloin, which is a bit more urban but still accessible by bike.

5. **Noe Valley Bakery** - A cozy spot in Noe Valley, perfect for a stop after exploring the hilly streets of the area.

Do any of these sound like a good fit for your next biking adventure?
```

# Memory Agent

- 我们将把我们学过的东西拼凑起来，构建一个具有长期记忆的[智能体](https://langchain-ai.github.io/langgraph/concepts/agentic_concepts/)。

- 我们的代理`task_mAIstro`将帮助我们管理待办事项列表！

- 我们之前构建的聊天机器人**总是**会反思对话并节省内存。

- `task_mAIstro`将决定**何时**保存内存（待办事项列表中的待办事项）。
- 我们之前构建的聊天机器人总是保存一种类型的内存，一个配置文件或集合。

- `task_mAIstro`可以决定保存到用户配置文件或待办事项的集合。

- 除了语义内存，`task_mAIstro`还将管理过程内存。

- 这允许用户更新创建ToDo项的首选项。

## Visibility into Trustcall updates

Trustcall创建和更新JSON模式。如果我们想看到Trustcall所做的 *特定更改*，该怎么办？

例如，我们之前看到Trustcall有一些自己的工具：

* 从验证失败中自我纠正——[参见跟踪示例](https://smith.langchain.com/public/5cd23009-3e05-4b00-99f0-c66ee3edd06e/r/9684db76-2003-443b-9aa2-9a9dbc5498b7)
* 更新现有文档——[参见跟踪示例](https://smith.langchain.com/public/f45bdaf0-6963-4c19-8ec9-f4b7fe0f68ad/r/760f90e1-a5dc-48f1-8c34-79d6a3414ac3)

这些工具的可见性对于我们将要构建的代理非常有用。

下面，我们将展示如何做到这一点！

```python
from pydantic import BaseModel, Field

class Memory(BaseModel):
    content: str = Field(description="The main content of the memory. For example: User expressed interest in learning about French.")

class MemoryCollection(BaseModel):
    memories: list[Memory] = Field(description="A list of memories about the user.")
```

我们可以添加一个[listener](https://python.langchain.com/docs/how_to/lcel_cheatsheet/#add-lifecycle-listeners)到Trustcall提取器。

这将把运行从extractor的执行传递到我们将要定义的类`Spy`。

我们的`Spy`类将提取Trustcall进行了哪些工具调用的信息。

```python
from trustcall import create_extractor
from langchain_openai import ChatOpenAI

# Inspect the tool calls made by Trustcall
class Spy:
    def __init__(self):
        self.called_tools = []

    def __call__(self, run):
        # Collect information about the tool calls made by the extractor.
        q = [run]
        while q:
            r = q.pop()
            if r.child_runs:
                q.extend(r.child_runs)
            if r.run_type == "chat_model":
                self.called_tools.append(
                    r.outputs["generations"][0][0]["message"]["kwargs"]["tool_calls"]
                )

# Initialize the spy
spy = Spy()

# Initialize the model
model = ChatOpenAI(model="gpt-4o", temperature=0)

# Create the extractor
trustcall_extractor = create_extractor(
    model,
    tools=[Memory],
    tool_choice="Memory",
    enable_inserts=True,
)

# Add the spy as a listener
trustcall_extractor_see_all_tool_calls = trustcall_extractor.with_listeners(on_end=spy)
```

```py
from langchain_core.messages import HumanMessage, SystemMessage, AIMessage

# Instruction
instruction = """Extract memories from the following conversation:"""

# Conversation
conversation = [HumanMessage(content="Hi, I'm Lance."), 
                AIMessage(content="Nice to meet you, Lance."), 
                HumanMessage(content="This morning I had a nice bike ride in San Francisco.")]

# Invoke the extractor
result = trustcall_extractor.invoke({"messages": [SystemMessage(content=instruction)] + conversation})
```

```python
# Messages contain the tool calls
for m in result["messages"]:
    m.pretty_print()
```

```
================================== Ai Message ==================================
Tool Calls:
  Memory (call_NkjwwJGjrgxHzTb7KwD8lTaH)
 Call ID: call_NkjwwJGjrgxHzTb7KwD8lTaH
  Args:
    content: Lance had a nice bike ride in San Francisco this morning.
```