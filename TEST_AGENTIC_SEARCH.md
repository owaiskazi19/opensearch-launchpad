# Testing Agentic Search Integration

## Prerequisites

1. Ensure you have Python dependencies installed:
```bash
pip install strands opensearch-py boto3
```

2. Set up AWS credentials (for Bedrock access):
```bash
export AWS_ACCESS_KEY_ID=your_access_key
export AWS_SECRET_ACCESS_KEY=your_secret_key
export AWS_REGION=us-east-1  # or your preferred region
```

3. Set up OpenSearch connection (if testing with local OpenSearch):
```bash
export OPENSEARCH_HOST=localhost
export OPENSEARCH_PORT=9200
export OPENSEARCH_USER=admin
export OPENSEARCH_PASSWORD=myStrongPassword123!
```

## Test Scenario 1: Natural Language Query Use Case

### Step 1: Start the Orchestrator
```bash
python orchestrator.py
```

### Step 2: Provide Sample Data
When prompted for sample data, use one of these options:

**Option A: Paste sample product data**
```json
{"id":"1","name":"Red Running Shoes","price":89.99,"color":"red","category":"footwear","description":"Lightweight running shoes with breathable mesh"}
{"id":"2","name":"Blue Sneakers","price":120.00,"color":"blue","category":"footwear","description":"Casual sneakers with cushioned sole"}
{"id":"3","name":"Black Leather Boots","price":199.99,"color":"black","category":"footwear","description":"Premium leather boots for all seasons"}
```

**Option B: Use built-in IMDb sample**
Select option `1` for built-in sample

### Step 3: Describe Your Use Case (KEY STEP)
When the orchestrator asks about your requirements, use language that triggers agentic search:

**Example queries that should trigger agentic search:**

1. **Multi-step questions requiring answer synthesis:**
   > "I want users to ask questions like 'What are the top-rated red shoes under $100 and why are they popular?' and get synthesized answers, not just a list of documents"

2. **Complex analytical questions:**
   > "Users need to ask multi-step questions like 'Show me budget laptops with good reviews and explain the trade-offs between them'"

3. **Conversational search with reasoning:**
   > "I want a conversational search experience where users ask questions like 'Which products are best for outdoor activities and why?' and get reasoned answers"

4. **Questions requiring query decomposition:**
   > "My users want to ask complex questions that need multiple queries to answer, like 'What are the most popular items in each category and what makes them stand out?'"

### Step 4: Verify Solution Planning
The solution planning assistant should:
1. Call `read_agentic_search_guide()` to learn about agentic search
2. Recognize that your use case needs natural language query translation
3. Recommend agentic search in the solution

**Look for in the output:**
```
Query Translation Layer: Agentic Search
Agentic Model: Bedrock Claude 4 Sonnet
Agent Type: conversational (or flow)
```

### Step 5: Confirm and Execute
When asked to confirm, say "yes" or "proceed"

The worker agent should:
1. Call `create_bedrock_agentic_model()` to register Claude model
2. Call either:
   - `create_agentic_search_conversational_agent()` for multi-turn conversations with memory
   - `create_agentic_search_flow_agent()` for stateless single-turn queries (recommended for most cases)
   - Or use `create_agentic_search_agent()` with agent_type parameter (deprecated but still works)
3. Call `create_agentic_search_pipeline()` to create and attach the pipeline

**Look for in the output:**
```
Agentic model 'us.anthropic.claude-sonnet-4-20250514-v1:0' (ID: xxx) registered and deployed successfully.
Agentic search agent 'xxx' (ID: yyy) created successfully with type 'conversational'.
Agentic search pipeline 'xxx' created and attached to index 'yyy' successfully.
```

## Test Scenario 2: Testing Individual Components

### Test the Knowledge Base Tool
```bash
python -c "
from scripts.tools import read_agentic_search_guide
content = read_agentic_search_guide()
print('Knowledge base loaded:', len(content), 'characters')
print('Contains agentic search info:', 'agentic' in content.lower())
"
```

### Test the QA Assistant
```bash
python -c "
from opensearch_qa_assistant import opensearch_qa_assistant
response = opensearch_qa_assistant('What is agentic search and when should I use it?')
print(response)
"
```

### Test Worker Tools Directly (requires OpenSearch + AWS setup)
```python
from scripts.opensearch_ops_tools import (
    create_bedrock_agentic_model,
    create_agentic_search_agent,
    create_agentic_search_pipeline
)

# Step 1: Create model
model_id = create_bedrock_agentic_model(
    model_name="us.anthropic.claude-sonnet-4-20250514-v1:0",
    role_arn="arn:aws:iam::123456789012:role/BedrockAccessRole"  # or leave empty for env credentials
)
print(f"Model ID: {model_id}")

# Step 2: Create agent
agent_id = create_agentic_search_agent(
    agent_name="test-agent",
    model_id=model_id,
    agent_type="conversational"
)
print(f"Agent ID: {agent_id}")

# Step 3: Create pipeline
result = create_agentic_search_pipeline(
    pipeline_name="test-pipeline",
    agent_id=agent_id,
    index_name="test-index"
)
print(result)
```

## Expected Behavior

### ✅ Agentic Search SHOULD be recommended when:
- User mentions "natural language queries"
- User mentions "multi-step questions"
- User says "users don't know Query DSL"
- User wants "conversational search"
- User describes complex filtering in natural language

### ❌ Agentic Search should NOT be recommended when:
- Simple keyword search only
- Simple semantic similarity search
- User explicitly mentions latency-critical requirements
- User explicitly mentions cost-sensitive requirements
- Query patterns are known and predictable

## Troubleshooting

### Issue: Solution planner doesn't recommend agentic search
**Solution:** Make sure your use case description explicitly mentions natural language queries or multi-step questions. Try using the example queries above.

### Issue: Worker fails to create model
**Possible causes:**
1. AWS credentials not set or invalid
2. Bedrock not available in your region
3. IAM role doesn't have Bedrock permissions

**Check:**
```bash
aws bedrock list-foundation-models --region us-east-1
```

### Issue: Worker fails to create agent
**Possible causes:**
1. Model ID is invalid or model not deployed
2. OpenSearch version < 3.2
3. ML Commons plugin not installed

**Check:**
```bash
curl -X GET "https://localhost:9200/_cat/plugins?v" -u admin:password --insecure
```

### Issue: Pipeline creation fails
**Possible causes:**
1. Agent ID is invalid
2. Index doesn't exist
3. OpenSearch version < 3.2

## Verification Checklist

- [ ] Knowledge base file exists and contains agentic search info
- [ ] `read_agentic_search_guide()` returns content
- [ ] QA assistant can answer questions about agentic search
- [ ] Solution planner mentions agentic search in system prompt
- [ ] Solution planner recommends agentic search for natural language use cases
- [ ] Worker has all 3 agentic functions imported
- [ ] Worker has all 3 functions in tools list
- [ ] Worker can execute model creation
- [ ] Worker can execute agent creation
- [ ] Worker can execute pipeline creation
- [ ] MCP server exports all agentic tools

## Quick Verification Commands

```bash
# Check knowledge base exists
ls -la scripts/knowledge/agentic_search_guide.md

# Check tool function exists
grep -n "def read_agentic_search_guide" scripts/tools.py

# Check solution planner mentions agentic
grep -n "Agentic Search" solution_planning_assistant.py

# Check worker imports
grep -n "create_bedrock_agentic_model" worker.py
grep -n "create_agentic_search_agent" worker.py
grep -n "create_agentic_search_pipeline" worker.py

# Check MCP server exports
grep -n "create_bedrock_agentic_model" mcp_server.py
```

All checks should return results. If any grep returns nothing, that component is missing.
