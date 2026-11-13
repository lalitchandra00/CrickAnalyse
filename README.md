# CrickAnalyse

A beginner-friendly LangGraph project for analyzing cricket batting
performance with parallel calculations and an LLM-generated summary.

The complete implementation is in [`llm.ipynb`](./llm.ipynb).

## What the workflow does

The notebook calculates three batting metrics in parallel and then sends the
results to Google Gemini to create a concise performance summary:

```text
             +------------------+
             |  Calculate SR    |
             +------------------+
                      |
  ------------------ START --------------
  |                   |                 |
  v                   v                 v
Calculate SR    Calculate BPB    Calculate boundary %
  |                   |                 |
  +-------------------+-----------------+
                      |
                      v
             Generate summary with Gemini
                      |
                     END
```

The workflow demonstrates LangGraph state management, parallel node execution,
and integration with `ChatGoogleGenerativeAI`.

## Project structure

| File | Purpose |
| --- | --- |
| [`llm.ipynb`](./llm.ipynb) | Defines and runs the cricket analysis workflow |
| [`workflow.png`](./workflow.png) | Visual representation of the workflow |
| [`requirements.txt`](./requirements.txt) | Python dependencies |
| `.env` | Stores the Google API key locally; do not commit this file |

## How it works, step by step

### 1. Import the libraries

The notebook imports:

- `StateGraph`, `START`, and `END` from LangGraph.
- `ChatGoogleGenerativeAI` to generate the batting summary with Gemini.
- `TypedDict` to define the workflow state.
- `load_dotenv` to load environment variables from `.env`.

### 2. Configure Gemini

`load_dotenv()` loads the API key from `.env`. The notebook then creates a
Gemini model using:

- Model: `gemini-3.8-flash`
- API key: `GOOGLE_API_KEY`
- Temperature: `1.5`

Create a `.env` file in the project directory:

```env
GOOGLE_API_KEY=your_google_api_key
```

Keep the key private and never add `.env` to source control.

### 3. Define the shared workflow state

`BatsmanState` describes the input data and calculated output:

```python
class BatsmanState(TypedDict):
    runs: int
    balls: int
    fours: int
    sixes: int
    sr: float
    bpb: float
    boundary_percent: float
    summary: str
```

The input values are runs, balls faced, fours, and sixes. The remaining fields
are populated by the workflow.

### 4. Calculate the batting metrics

The workflow calculates:

- **Strike rate (`sr`)**: `(runs / balls) * 100`
- **Balls per boundary (`bpb`)**: `balls / (fours + sixes)`
- **Boundary percentage**: runs scored through fours and sixes as a percentage
  of total runs

These calculations start from `START` independently, allowing LangGraph to
execute them in parallel.

### 5. Generate the summary

After all three calculations complete, the `summary` node builds a prompt with
the calculated statistics and invokes Gemini:

```python
summary = model.invoke(prompt).content
```

The generated text is stored in `state["summary"]`.

### 6. Build and compile the graph

The notebook registers the calculation and summary functions as graph nodes:

```python
graph.add_node("calculate_sr", calculate_sr)
graph.add_node("calculate_bpb", calculate_bpb)
graph.add_node("calculate_boundary_percent", calculate_boundary_percent)
graph.add_node("summary", summary)
```

The calculation nodes connect directly from `START`. Each then connects to the
summary node, which connects to `END`:

```python
graph.add_edge(START, "calculate_sr")
graph.add_edge(START, "calculate_bpb")
graph.add_edge(START, "calculate_boundary_percent")

graph.add_edge("calculate_sr", "summary")
graph.add_edge("calculate_bpb", "summary")
graph.add_edge("calculate_boundary_percent", "summary")
graph.add_edge("summary", END)
```

`graph.compile()` converts the graph definition into an executable workflow.

### 7. Run the workflow

The workflow starts with a batsman’s match statistics:

```python
initial_state = {
    "runs": 100,
    "balls": 50,
    "fours": 6,
    "sixes": 4,
}

final_state = workflow.invoke(initial_state)
```

After execution, `final_state` contains the calculated metrics and the Gemini
summary.

## Setup and run

1. Create or activate a Python environment.
2. Install the dependencies:

   ```bash
   pip install -r requirements.txt
   ```

3. Create a `.env` file with your `GOOGLE_API_KEY`.
4. Open [`llm.ipynb`](./llm.ipynb) in Jupyter or VS Code.
5. Run the notebook cells from top to bottom.

## Notes

- A valid Google Gemini API key is required for the summary node.
- The notebook must be run sequentially so the model and graph are defined
  before invoking the workflow.
- Do not commit `.env` or expose your API key.
