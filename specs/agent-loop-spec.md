# Spec: `run_agent()`

**File:** `agent.py`
**Status:** Partially pre-filled — complete the two blank fields before implementing

---

## Purpose

Orchestrate a single conversational turn for the Plant Advisor agent. Given a user message and the conversation history, call the LLM with available tools, execute any tool calls the LLM requests, and return the final text response.

This is the core of what makes Plant Advisor an *agent* rather than a simple chatbot: the ability to decide which tools to call, use their results to inform its response, and loop until it has everything it needs.

---

## Input / Output Contract

**Inputs:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `user_message` | `str` | The user's current message |
| `history` | `list` | Gradio conversation history — list of `[user_msg, assistant_msg]` pairs |

**Output:** `str`

The agent's final text response for this turn. Should never be empty — if something goes wrong, return a user-readable fallback message.

---

## Design Decisions

*Read `specs/system-design.md` (especially the "How the Groq Tool Calling API Works" section) before reviewing these. Complete the two blank fields before writing any code.*

---

### Messages list structure

The messages list must start with the system prompt, then replay the conversation
history, then add the new user message. Gradio history is a list of `[user, assistant]`
pairs — convert each pair to two API-format dicts:

```python
messages = [{"role": "system", "content": SYSTEM_PROMPT}]

for user_msg, assistant_msg in history:
    messages.append({"role": "user", "content": user_msg})
    if assistant_msg:
        messages.append({"role": "assistant", "content": assistant_msg})

messages.append({"role": "user", "content": user_message})
```

---

### Initial LLM call

Pass the model, the messages list, the tool definitions, and `tool_choice="auto"`
so the LLM can decide whether to call a tool or respond directly:

```python
response = client.chat.completions.create(
    model=LLM_MODEL,
    messages=messages,
    tools=TOOL_DEFINITIONS,
    tool_choice="auto",
)
```

---

### Detecting tool calls in the response

The response object has a `choices` list. Index 0 gives the assistant message.
Check its `tool_calls` attribute — if it's truthy, the LLM wants to call tools:

```python
assistant_message = response.choices[0].message

if not assistant_message.tool_calls:
    # No tool calls — LLM has a final answer
    ...
```

---

### Appending the assistant message

When there are tool calls, append the full assistant message object to `messages`
**before** appending any tool results. The API requires this ordering — a tool
result message must immediately follow the assistant message that requested it:

```python
messages.append(assistant_message)  # must come first
```

---

### Executing and appending tool results

For each tool call, extract the name and arguments, call `dispatch_tool()`, and
append the result as a `"tool"` role message. The `tool_call_id` links this result
back to the specific tool call that requested it:

```python
for tool_call in assistant_message.tool_calls:
    tool_name = tool_call.function.name
    tool_args = json.loads(tool_call.function.arguments)
    tool_result = dispatch_tool(tool_name, tool_args)

    messages.append({
        "role": "tool",
        "tool_call_id": tool_call.id,
        "content": tool_result,
    })
```

---

### Loop termination conditions

*The loop should stop when: (a) the LLM returns a response with no tool calls, OR (b) the MAX_TOOL_ROUNDS limit is reached. Describe how you will detect each condition and what you will return in each case.*

```
[your answer here]
pseudocode:

craete message list structure # per 'Messages List structure' above
rounds = 0
while rounds < MAX_TOOL_ROUNDS
    call tool  # per 'Initial LLM call' above
    assistant_message = response.choices[0].message
    if not assistant_message.tool_calls: # No tool calls — LLM has a final answer
        return assistant_message.content
    else:
        Executing and appending tool results # per 'Executing and appending tool results'
    rounds += 1
return f"Unable to complete user request as {MAX_TOOL_ROUNDS} rounds of tool usage has been exceeded" 
```

---

### Extracting the final text response

*Once the loop exits because there are no more tool calls, how do you extract the text content from the response object? What field holds the string you should return?*

```
[your answer here]
response.choices[0].message.content
```

---

## Implementation Notes

*Fill this in after implementing and testing.*

**Trace of a working agent turn (what tools were called and in what order):**

```
Query: "How should I care for my calathea?"
  → Tool call: lookup_plant({'plant_name': 'calathea'})
  ← Result: {"found": true, "plant": {"display_name": "Calathea", "scientific_name": "Calathea ornata", "aliases": ["prayer plant", ...

Final response: [brief description]
According to the care data for your calathea, this plant prefers low to medium indirect light, consistently moist soil, and high humidity (50%+). It's also sensitive to cold drafts and temperatures below 55°F. To care for your calathea, make sure to water it every 1-2 weeks with filtered, distilled, or rainwater, and fertilize it monthly during the growing season. Also, keep it in an area with high humidity, and avoid direct sunlight, which can damage the plant's leaves. If you experience any issues, such as brown leaf edges or leaf curling, adjust your care routine accordingly.
```

**What happens when you ask about a plant that isn't in the database?**

```
[describe the behavior you observed]
I'm sorry, but 'bok choy' was not found in the plant database. I only have information on a limited set of houseplants and it seems that 'bok choy' is not one of them. However, I can tell you that bok choy is a common vegetable that can be grown indoors. It's a type of Chinese cabbage that prefers partial shade, consistent moisture, and cooler temperatures. If you have any specific questions about its care or would like general advice, I'd be happy to help.
```

**One thing about the tool call API that surprised you:**

```
[your answer here]
```
