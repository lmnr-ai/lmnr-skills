# Python quickstart demo (no external LLM calls)

## Steps

1. Install the SDK:
   - `pip install lmnr`
2. Export `LMNR_PROJECT_API_KEY`.
3. Create `quickstart.py` with the code below.
4. Run `python quickstart.py`.
5. Open the Laminar UI -> Traces and filter by tag `quickstart` or metadata `run_id`.

## quickstart.py

```python
import os
import time
from lmnr import Laminar, observe

Laminar.initialize(
    project_api_key=os.environ["LMNR_PROJECT_API_KEY"],
    instruments=set(),
    disable_batch=True,
    # For self-hosted Laminar:
    # base_url="http://localhost",
    # http_port=8000,
    # grpc_port=8001,
)

run_id = f"quickstart-{int(time.time())}"

@observe(name="quickstart.root", tags=["quickstart", run_id], metadata={"run_id": run_id})
def main():
    @observe(name="quickstart.step")
    def step():
        return {"answer": 42}

    return {"step": step()}

if __name__ == "__main__":
    result = main()
    print("Laminar quickstart run id:", run_id)
    print("Result:", result)
    Laminar.flush()  # IMPORTANT: required in short-lived scripts
```

## Notes

- If you prefer batching, remove `disable_batch=True` and keep `Laminar.flush()`.
