# Agent Communication Patterns

## Overview

The AI agent architecture employs several communication patterns for agent-to-agent interactions. These patterns differ based on the complexity of the workflow, synchronicity requirements, and composition models.

## Direct Invocation Pattern

The most common pattern is direct invocation, where multiagents directly invoke individual agents:

```python
def invoke_data_wrangling_agent(state: PrimaryState):
    response = data_wrangling_agent.invoke({
        "user_instructions": state["user_instructions_data_wrangling"],
        "data_raw": state["data_raw"],
        "max_retries": state["max_retries"],
        "retry_count": state["retry_count"]
    })
    
    return {
        "data_wrangled": response.get("data_wrangled", {}),
        "data_wrangler_function": response.get("data_wrangler_function", "")
    }
```

This pattern has these characteristics:
- Synchronous execution (blocks until completion)
- Direct access to component agent's response
- State transformation between parent and child agent formats
- No intermediate communication during child agent execution

## Asynchronous Invocation Pattern

For non-blocking operations, the architecture uses asynchronous invocation:

```python
async def ainvoke_agent(self, user_instructions, data_raw: pd.DataFrame, max_retries=3, retry_count=0, **kwargs):
    response = await self._compiled_graph.ainvoke({
        "user_instructions": user_instructions,
        "data_raw": self._convert_data_input(data_raw),
        "max_retries": max_retries,
        "retry_count": retry_count,
    }, **kwargs)
    
    if response.get("messages"):
        response["messages"] = remove_consecutive_duplicates(response["messages"])
    
    self.response = response
```

This pattern enables:
- Non-blocking operations for concurrent agent execution
- Awaiting multiple agent responses
- Continuation-based processing of results

## Message-Based State Transfer

State between agents is transferred as structured data in message format:

```python
# From multiagent to component agent
input_state = {
    "user_instructions": state["user_instructions_data_visualization"],
    "data": state["data_wrangled"],
    "max_retries": state["max_retries"],
    "retry_count": state["retry_count"]
}

# From component agent back to multiagent
output_state = {
    "plotly_graph": response.get("plotly_graph", {}),
    "data_visualization_function": response.get("data_visualization_function", ""),
    "plotly_error": response.get("plotly_error", "")
}
```

Key aspects:
- Structured data objects for state transfer
- Explicit mapping between parent and child agent state schemas
- Selective state transfer (only relevant portions)
- Error information propagation

## Routing and Orchestration

Multiagents implement routing logic to determine which component agents to invoke:

```python
def router_chart_or_table(state: PrimaryState):
    if state["routing_preprocessor_decision"] == "chart":
        return "invoke_data_visualization_agent"
    else:
        return "route_printer"
```

This enables:
- Conditional execution paths
- Dynamic determination of next agent
- Workflow orchestration based on state and user instructions

## Response Aggregation

Multiagents aggregate responses from component agents to create a unified response:

```python
def route_printer(state: PrimaryState):
    messages = []
    
    # Create a single message with all the outputs
    return {
        "messages": messages
    }
```

This pattern:
- Combines results from multiple agents
- Creates a unified view for the user
- Formats heterogeneous results consistently

## Stream-Based Communication

For real-time updates, the architecture supports streaming:

```python
def stream(self, input, config=None, stream_mode=None, **kwargs):
    self.response = self._compiled_graph.stream(
        input=input, 
        config=config, 
        stream_mode=stream_mode, 
        **kwargs
    )
    
    return self.response
```

This enables:
- Real-time updates during long-running processes
- Partial result delivery
- Progress monitoring

## uAgent Implementation Approaches

The existing communication patterns can be reimplemented using Fetch.ai uAgents as follows:

### 1. Protocol-Based Communication

Replace direct invocation with protocol-based communication:

```python
# Component agent implementation
@protocol_handler("process_data")
async def process_data(self, message):
    # Process data
    # Send response back to parent agent
    await self.context.send(
        message.sender, 
        Message(
            protocol="process_data_response",
            performative="inform",
            content={
                "data_wrangled": processed_data,
                "data_wrangler_function": function_code
            }
        )
    )

# Parent agent implementation
@protocol_handler("process_data_response")
async def handle_data_processing_response(self, message):
    # Update state with response
    self.state.update({
        "data_wrangled": message.content["data_wrangled"],
        "data_wrangler_function": message.content["data_wrangler_function"]
    })
    
    # Continue workflow
    await self._continue_workflow()
```

### 2. Message Queue for Asynchronous Operations

For non-blocking operations, use a message queue pattern:

```python
@protocol_handler("request_processing")
async def handle_processing_request(self, message):
    # Enqueue request for processing
    task_id = self._generate_task_id()
    self.task_queue[task_id] = {
        "status": "pending",
        "requester": message.sender,
        "data": message.content["data"]
    }
    
    # Process in background
    asyncio.create_task(self._process_task(task_id))
    
    # Respond with task ID
    await self.context.send(
        message.sender,
        Message(
            protocol="task_created",
            performative="inform",
            content={"task_id": task_id}
        )
    )
    
async def _process_task(self, task_id):
    # Process task
    # Update task status
    # Send completion notification
```

### 3. Publish-Subscribe for Multicast Communication

For broadcasting updates to multiple interested agents:

```python
# Publisher agent
async def publish_update(self, topic, data):
    await self.context.send(
        Address("agent-directory"),
        Message(
            protocol="publish",
            performative="inform",
            content={
                "topic": topic,
                "data": data
            }
        )
    )

# Subscriber agent
@protocol_handler("subscription_update")
async def handle_update(self, message):
    # Process update from subscribed topic
    topic = message.content["topic"]
    data = message.content["data"]
    # Update internal state
```

### 4. Request-Response for Synchronous Operations

For simple request-response patterns:

```python
# Requester agent
async def request_data(self, target_agent, query):
    response = await self.context.request(
        target_agent,
        Message(
            protocol="data_query",
            performative="request",
            content={"query": query}
        )
    )
    return response.content["result"]

# Responder agent
@protocol_handler("data_query")
async def handle_query(self, message):
    query = message.content["query"]
    result = self._process_query(query)
    return {"result": result}
```

These communication patterns provide a foundation for implementing the existing agent architecture using Fetch.ai uAgents, allowing for flexible, scalable, and robust agent-to-agent communication. 