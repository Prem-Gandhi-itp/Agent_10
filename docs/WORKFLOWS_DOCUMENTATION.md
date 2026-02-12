# Workflows System Documentation

## Overview

The Workflows System in the NeuroStack provides a comprehensive framework for creating, managing, and executing multi-agent workflows. It enables the composition of individual agents into coordinated workflows that can execute tasks using different patterns: sequential, parallel, loop, and orchestration.

## Architecture Components

### Core Components

#### 1. **Workflow Templates**
- **Sequential Template** (`SequentialWorkflowTemplate`): Executes agents in a predefined order
- **Parallel Template** (`ParallelWorkflowTemplate`): Executes agents simultaneously
- **Loop Template** (`LoopWorkflowTemplate`): Executes agents in iterative cycles
- **Orchestration Template** (`OrchestrationWorkflowTemplate`): Uses LLM-based intelligent delegation

#### 2. **Workflow Registry** (`WorkflowRegistry`)
- Manages workflow discovery and registration
- Handles filesystem-based workflow storage
- Provides workflow metadata and configuration management

#### 3. **ADK Workflow Engine** (`ADKWorkflowEngine`)
- Executes workflows using Google ADK agents
- Manages workflow lifecycle and execution state
- Provides execution monitoring and history

#### 4. **Agent Loader** (`AgentLoader`)
- Discovers and loads existing agents for workflow creation
- Provides agent metadata and capabilities
- Manages agent-to-workflow integration

#### 5. **Workflow Execution Engine** (`WorkflowExecutionEngine`)
- Handles workflow execution with context management
- Supports parallel and sequential execution patterns
- Provides error handling and recovery

## Workflow Creation Process

### 1. Agent Discovery and Selection

The workflow creation process begins with discovering available agents:

```python
# Agent discovery
agent_loader = get_agent_loader()
available_agents = agent_loader.discover_agents()

# Agent selection for workflow
selected_agents = ["finance", "database_agent", "plotting_analysis_agent"]
```

### 2. Workflow Configuration Creation

Each workflow type creates a configuration dictionary:

```python
# Example orchestration workflow config
workflow_config = {
    "workflow_id": "unique_id",
    "name": "FinanceWorkflow",
    "type": "orchestration",
    "description": "Financial analysis workflow",
    "created_at": "2024-01-01T00:00:00",
    "orchestrator": {
        "name": "FinanceWorkflow_orchestrator",
        "capabilities": ["task_analysis", "agent_selection"],
        "config": {
            "model_name": "gemini-2.0-flash-exp",
            "instructions": "Coordinate financial analysis tasks",
            "routing_strategy": "intelligent",
            "max_delegations": 3
        }
    },
    "available_agents": [
        {
            "agent_name": "finance",
            "capabilities": ["financial_analysis", "data_processing"],
            "config": {...}
        }
    ],
    "execution_config": {
        "model_name": "gemini-2.0-flash-exp",
        "routing_strategy": "intelligent",
        "delegation_tracking": True,
        "adaptive_routing": True
    }
}
```

### 3. Workflow Directory Creation

Each workflow gets its own directory structure:

```
workflows/
├── FinanceWorkflow/
│   ├── __init__.py
│   ├── agent.py           # Generated workflow agent
│   └── workflow_config.json
```

### 4. Agent Script Generation

The system generates a complete `agent.py` file for each workflow:

```python
# Generated agent.py structure
"""
FinanceWorkflow - Orchestration Workflow
Generated automatically by OrchestrationWorkflowTemplate.
"""

import sys
from pathlib import Path
sys.path.insert(0, str(Path(__file__).parent.parent.parent.parent))

from google.adk.agents import LlmAgent
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService

# Import existing agents
from modular_agents.agents.finance.agent import root_agent as finance_agent
from modular_agents.agents.database_agent.agent import root_agent as database_agent

# Create independent workflow instances
finance_workflow_instance = LlmAgent(
    name="FinanceWorkflow_finance",
    model="gemini-2.0-flash-exp",
    tools=finance_agent.tools if hasattr(finance_agent, 'tools') else [],
    instruction=getattr(finance_agent, 'instruction', 'You are a helpful assistant.'),
    description=getattr(finance_agent, 'description', 'finance agent')
)

# Create the orchestration workflow
workflow_agent = LlmAgent(
    name="FinanceWorkflow",
    model="gemini-2.0-flash-exp",
    sub_agents=[finance_workflow_instance, database_workflow_instance],
    description="Financial analysis workflow",
    instruction="You are a FinanceWorkflow orchestrator..."
)

# For ADK compatibility
root_agent = workflow_agent
```

## Workflow Types and Templates

### 1. Sequential Workflows

**Purpose**: Execute agents in a predefined order where each agent processes the output of the previous agent.

**Creation Process**:
```python
template = SequentialWorkflowTemplate()
config = await template.create_workflow(
    workflow_name="DataProcessingSequence",
    agent_names=["data_collector", "data_processor", "data_analyzer"],
    description="Sequential data processing workflow",
    instructions="Process data through each stage sequentially"
)
```

**Generated Agent Structure**:
```python
workflow_agent = SequentialAgent(
    name="DataProcessingSequence",
    sub_agents=[collector_instance, processor_instance, analyzer_instance],
    description="Sequential data processing workflow"
)
```

### 2. Parallel Workflows

**Purpose**: Execute agents simultaneously and aggregate results.

**Creation Process**:
```python
template = ParallelWorkflowTemplate()
config = await template.create_workflow(
    workflow_name="MultiSourceAnalysis",
    agent_names=["source1_agent", "source2_agent", "source3_agent"],
    description="Parallel data source analysis",
    result_aggregation="combine"
)
```

**Generated Agent Structure**:
```python
workflow_agent = ParallelAgent(
    name="MultiSourceAnalysis",
    sub_agents=[source1_instance, source2_instance, source3_instance],
    description="Parallel data source analysis"
)
```

### 3. Loop Workflows

**Purpose**: Execute agents in iterative cycles with termination conditions.

**Creation Process**:
```python
template = LoopWorkflowTemplate()
config = await template.create_workflow(
    workflow_name="IterativeRefinement",
    agent_names=["generator", "refiner", "evaluator"],
    description="Iterative content refinement",
    max_iterations=5,
    termination_condition="quality_threshold",
    quality_threshold=0.8
)
```

**Generated Agent Structure**:
```python
workflow_agent = LoopAgent(
    name="IterativeRefinement",
    sub_agents=[generator_instance, refiner_instance, evaluator_instance],
    description="Iterative content refinement"
)
```

### 4. Orchestration Workflows

**Purpose**: Use LLM-based intelligent delegation to route tasks to appropriate agents.

**Creation Process**:
```python
template = OrchestrationWorkflowTemplate()
config = await template.create_workflow(
    workflow_name="IntelligentTaskRouter",
    agent_names=["specialist1", "specialist2", "specialist3"],
    description="Intelligent task routing workflow",
    routing_strategy="capability_based",
    max_delegations=3
)
```

**Generated Agent Structure**:
```python
workflow_agent = LlmAgent(
    name="IntelligentTaskRouter",
    model="gemini-2.0-flash-exp",
    sub_agents=[specialist1_instance, specialist2_instance, specialist3_instance],
    description="Intelligent task routing workflow",
    instruction="You are an orchestrator that intelligently routes tasks..."
)
```

## Workflow Execution

### 1. ADK-Based Execution

Workflows are executed using Google ADK agents:

```python
# Execute workflow
adk_engine = get_adk_engine()
result = adk_engine.execute_workflow(
    workflow_name="FinanceWorkflow",
    input_data={"task": "Analyze Q4 financial data"}
)
```

### 2. Web Server Execution

Workflows can be executed through ADK web servers:

```python
# Start ADK web server
subprocess.Popen(["adk", "web", "--port", "8800"], cwd=workflows_dir)

# Execute via HTTP API
response = requests.post(
    "http://localhost:8800/chat",
    json={
        "message": "Analyze financial data",
        "session_id": "session_123",
        "user_id": "user_456"
    }
)
```

### 3. Execution Context Management

Each execution maintains context and state:

```python
class WorkflowExecutionContext:
    def __init__(self, execution_id: str, workflow: Workflow):
        self.execution_id = execution_id
        self.workflow = workflow
        self.global_context: Dict[str, Any] = {}
        self.node_outputs: Dict[str, Any] = {}
        self.execution_path: List[str] = []
        self.active_nodes: Set[str] = set()
```

## API Endpoints

### Workflow Management

#### Create Workflow with Existing Agents
```http
POST /workflows/adk/create-with-agents
Content-Type: application/json

{
  "workflow_name": "FinanceAnalysis",
  "workflow_type": "orchestration",
  "agent_names": ["finance", "database_agent", "plotting_analysis_agent"],
  "description": "Financial analysis workflow",
  "instructions": "Coordinate financial analysis tasks",
  "model": "gemini-2.0-flash-exp"
}
```

#### Execute Workflow
```http
POST /workflows/adk/execute
Content-Type: application/json

{
  "workflow_name": "FinanceAnalysis",
  "input_data": "Analyze Q4 financial performance",
  "session_id": "session_123"
}
```

#### Get Workflow Status
```http
GET /workflows/adk/status/{execution_id}
```

#### List Available Workflows
```http
GET /workflows/adk/list
```

### Workflow Templates

#### Get Available Templates
```http
GET /workflows/templates
```

#### Get Template Configuration
```http
GET /workflows/templates/{template_type}/config
```

#### Validate Workflow Configuration
```http
POST /workflows/adk/validate
Content-Type: application/json

{
  "workflow_config": {
    "name": "TestWorkflow",
    "workflow_type": "sequential",
    "config": {...}
  }
}
```

### Workflow Chat Interface

#### Chat with Workflow
```http
POST /workflows/chat
Content-Type: application/json

{
  "workflow_name": "FinanceAnalysis",
  "message": "What was the revenue growth in Q4?",
  "session_id": "session_123"
}
```

## Workflow Registry

### Registration Process

Workflows are automatically registered when created:

```python
workflow_info = WorkflowInfo(
    name="FinanceWorkflow",
    workflow_type="orchestration",
    description="Financial analysis workflow",
    version="1.0.0",
    config_path="/path/to/workflow_config.json",
    template_path="/path/to/workflow/directory",
    created_at=datetime.now(),
    last_modified=datetime.now(),
    status="active",
    metadata={
        "agent_names": ["finance", "database_agent"],
        "agent_count": 2,
        "template_used": "orchestration"
    }
)

workflow_registry.update_workflow(workflow_info)
```

### Discovery and Management

```python
# List all workflows
workflows = workflow_registry.list_workflows()

# Get specific workflow
workflow = workflow_registry.get_workflow("FinanceWorkflow")

# Update workflow status
workflow_registry.update_workflow_status("FinanceWorkflow", "active")
```

## Integration with Agents

### Agent Loading and Preparation

```python
class AgentLoader:
    def load_agent(self, agent_name: str) -> Any:
        """Load an existing agent for workflow integration"""
        try:
            agent_path = self.agents_dir / agent_name / "agent.py"
            spec = importlib.util.spec_from_file_location("agent", agent_path)
            module = importlib.util.module_from_spec(spec)
            spec.loader.exec_module(module)
            
            if hasattr(module, 'root_agent'):
                return module.root_agent
            return None
        except Exception as e:
            logger.error(f"Failed to load agent {agent_name}: {e}")
            return None
```

### Agent Instance Creation

For each workflow, independent agent instances are created:

```python
def create_agent_instance(original_agent, workflow_name: str, model: str):
    """Create independent agent instance for workflow"""
    return LlmAgent(
        name=f"{workflow_name}_{original_agent.name}",
        model=model,
        tools=original_agent.tools if hasattr(original_agent, 'tools') else [],
        instruction=getattr(original_agent, 'instruction', 'You are a helpful assistant.'),
        description=getattr(original_agent, 'description', f'{original_agent.name} agent')
    )
```

## Workflow Patterns

### 1. Data Processing Pipeline (Sequential)
```python
# Create sequential workflow for data processing
agents = ["data_collector", "data_cleaner", "data_analyzer", "report_generator"]
workflow = await sequential_template.create_workflow(
    workflow_name="DataPipeline",
    agent_names=agents,
    description="Complete data processing pipeline"
)
```

### 2. Multi-Source Analysis (Parallel)
```python
# Create parallel workflow for multi-source analysis
agents = ["news_analyzer", "social_media_analyzer", "financial_data_analyzer"]
workflow = await parallel_template.create_workflow(
    workflow_name="MultiSourceAnalysis",
    agent_names=agents,
    description="Parallel analysis of multiple data sources"
)
```

### 3. Iterative Improvement (Loop)
```python
# Create loop workflow for iterative improvement
agents = ["content_generator", "content_refiner", "quality_evaluator"]
workflow = await loop_template.create_workflow(
    workflow_name="ContentRefinement",
    agent_names=agents,
    description="Iterative content refinement process",
    max_iterations=5
)
```

### 4. Intelligent Task Routing (Orchestration)
```python
# Create orchestration workflow for intelligent routing
agents = ["financial_expert", "technical_analyst", "market_researcher"]
workflow = await orchestration_template.create_workflow(
    workflow_name="InvestmentAnalysis",
    agent_names=agents,
    description="Intelligent investment analysis workflow"
)
```

## Error Handling and Recovery

### Workflow Execution Errors

```python
try:
    result = adk_engine.execute_workflow(workflow_name, input_data)
except Exception as e:
    logger.error(f"Workflow execution failed: {e}")
    return {"error": str(e), "status": "failed"}
```

### Agent Failure Handling

```python
# Sequential workflow error handling
for i, agent in enumerate(agents):
    try:
        result = agent.execute(current_context)
        results.append(result)
    except Exception as e:
        logger.error(f"Agent {i} failed: {e}")
        if error_handling == "stop_on_error":
            break
        elif error_handling == "continue_on_error":
            results.append({"error": str(e)})
            continue
```

## Performance Considerations

### 1. Agent Instance Management
- Independent agent instances prevent conflicts
- Memory-efficient agent loading
- Proper cleanup after execution

### 2. Parallel Execution Optimization
- Concurrent agent execution
- Result aggregation strategies
- Resource management

### 3. Workflow Caching
- Configuration caching
- Agent metadata caching
- Execution result caching

## Best Practices

### 1. Workflow Design
- Choose appropriate workflow type for use case
- Design clear agent responsibilities
- Plan for error scenarios

### 2. Agent Selection
- Select complementary agents
- Ensure agent compatibility
- Consider execution dependencies

### 3. Configuration Management
- Use descriptive workflow names
- Document workflow purposes
- Version control configurations

### 4. Monitoring and Debugging
- Use execution logging
- Monitor workflow performance
- Track agent interactions

## Troubleshooting

### Common Issues

1. **Agent Not Found**
   - Verify agent exists in `/agents` directory
   - Check agent has `root_agent` export
   - Ensure agent is properly registered

2. **Workflow Creation Fails**
   - Check agent compatibility
   - Verify workflow configuration
   - Ensure ADK dependencies are installed

3. **Execution Errors**
   - Check agent tool availability
   - Verify model access
   - Review execution logs

### Debug Commands

```python
# Check available agents
available_agents = discover_available_agents()

# Validate workflow configuration
result = workflow_registry.validate_workflow_config(config)

# Check workflow status
status = adk_engine.get_workflow_status(execution_id)
```

## Future Enhancements

### Planned Features
1. **Dynamic Workflow Modification**: Runtime workflow updates
2. **Workflow Versioning**: Version control for workflows
3. **Advanced Monitoring**: Real-time execution monitoring
4. **Workflow Marketplace**: Shared workflow templates
5. **Performance Analytics**: Detailed execution metrics

### Integration Opportunities
1. **External Services**: Integration with cloud services
2. **Database Workflows**: Specialized database operations
3. **API Workflows**: REST/GraphQL API orchestration
4. **Streaming Workflows**: Real-time data processing

The Workflows System provides a powerful framework for creating sophisticated multi-agent workflows that can handle complex tasks through coordinated agent collaboration. By building on the existing agents infrastructure, it enables the creation of scalable, maintainable, and efficient workflow solutions. 