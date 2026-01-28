# Python instrumentation

## Initialize once

Call `Laminar.initialize()` at startup and pick the instrumentation set you need.

```python
import os
from lmnr import Laminar, Instruments

Laminar.initialize(
    project_api_key=os.environ.get("LMNR_PROJECT_API_KEY"),
    instruments={Instruments.OPENAI},  # or set() to disable all
    # For self-hosted Laminar:
    # base_url="http://localhost",
    # http_port=8000,
    # grpc_port=8001,
)
```

## Add manual spans

Use `@observe()` on functions that represent meaningful steps.

```python
from lmnr import observe

@observe(
    name="agent.run",
    session_id=session_id,
    user_id=user_id,
    tags=["agent", "search"],
    metadata={"route": "/search"},
)
def run_agent():
    retrieve_context()

@observe(name="retrieve.context", ignore_inputs=["secrets"])  # hide sensitive args
def retrieve_context():
    pass
```

## Manual span context

For ad-hoc spans, use `Laminar.start_as_current_span()`.

```python
from lmnr import Attributes, Laminar

with Laminar.start_as_current_span("llm.call", span_type="LLM", input=prompt):
    response = call_model(prompt)
    Laminar.set_span_output(response.text)
    Laminar.set_span_attributes({
        Attributes.PROVIDER: "openai",
        Attributes.REQUEST_MODEL: response.model,
        Attributes.RESPONSE_MODEL: response.model,
        Attributes.INPUT_TOKEN_COUNT: response.usage.prompt_tokens,
        Attributes.OUTPUT_TOKEN_COUNT: response.usage.completion_tokens,
    })
```

## Short-lived scripts

Call `Laminar.flush()` at the end of a script or test to export spans quickly.
