Certainly! I'll help you modify your `graph_builder.py` script to meet your requirements:

1. **Generate `return_json.json`**: Store the graph JSON response in a file named `return_json.json` in the session directory, overwriting it each time.

2. **Generate `node_descriptions.json`**: For each node in `return_json.json`, use the LLM to generate an HTML description (using Tailwind CSS classes) and store these in a JSON file named `node_descriptions.json`, ensuring the IDs match.

3. **Overwrite Existing JSON Files**: Each time the graph is generated, both JSON files should overwrite any existing ones in the session directory.

4. **Copy JSON Files to Specified Directory**: After generating the JSON files, copy both `return_json.json` and `node_descriptions.json` to `/Users/ysharm12/Documents/orchestra/orchestra/orchestra-backend/app/data/`.

---

## **Updated `graph_builder.py` Script**

Below is the modified `graph_builder.py` script incorporating all your requirements:

```python
# graph_builder.py

import os
import re
import json
import asyncio
import sys
import shutil
from llm import call_graph

async def read_scripts(session_dir):
    """
    Reads all the scripts in the session directory and returns them as a list.
    """
    scripts = []
    script_files = sorted([
        f for f in os.listdir(session_dir) if f.endswith('.py')
    ])

    for filename in script_files:
        filepath = os.path.join(session_dir, filename)
        with open(filepath, 'r') as f:
            code = f.read()
        version = await extract_version(code) or 1
        scripts.append({
            'filename': filename,
            'version': version,
            'content': code
        })
    return scripts

async def extract_version(script_content):
    """
    Extracts the version number from the script content.
    """
    version_pattern = r'#.*\(Version\s*(\d+)\)'
    match = re.search(version_pattern, script_content)
    if match:
        return int(match.group(1))
    else:
        return None

async def generate_graph(scripts, ai_response):
    """
    Generates the graph JSON including the AI response.
    """
    # Prepare the prompt for the LLM
    prompt = f"""
You are an assistant that generates a JSON response for a UI to display a graph.
The graph represents scripts and their relationships.

Here are the scripts:

"""

    for script in scripts:
        prompt += f"\n# {script['filename']} (Version {script['version']})\n```python\n{script['content']}\n```\n"

    prompt += f"""
The AI has the following response to the user:

"{ai_response}"

Your task is to analyze the scripts and generate a JSON response that represents the scripts and their relationships.

The API response must follow this structure:

{{
    "ai_response": "Text response to user",
    "nodes": [...],
    "edges": [...],
}}

Ensure that:
- The "ai_response" field contains the AI's response to the user.
- Nodes and edges accurately represent the relationships between scripts.
- Positions are assigned to nodes to make the graph visually appealing.

Each node must have these properties:
- "id": Unique string ID
- "type": Must be "custom"
- "position": An object with "x" and "y" coordinates (floats)
- "data": An object containing:
    - "label": Display name (script filename)
    - "type": One of: "input", "process", "output"
    - "description": Optional description
    - "inputs": Array of input handles, MUST use "input-0" format
    - "outputs": Array of output handles, MUST use "output-0" format
    - "color": Optional color

Each edge must have these properties:
- "id": Unique string ID (suggested format: "e{{source}}-{{target}}")
- "source": Source node ID (string)
- "target": Target node ID (string)
- "sourceHandle": Must match source node's outputs array
- "targetHandle": Must match target node's inputs array
- "animated": Optional, defaults to true

Important Rules:

1. Handle IDs:
   - Always use zero-based indices: "input-0", "output-0"
   - Multiple handles should increment: "input-0", "input-1", etc.
   - Handles in edges must match the node's inputs/outputs arrays

2. Node Types:
   - "input" nodes should only have outputs
   - "output" nodes should only have inputs
   - "process" nodes can have both inputs and outputs

Return only the JSON response without any extra text or comments. The JSON must be valid and parseable.
"""

    # Call the LLM to generate the graph
    llm_response = await call_graph(prompt)
    json_response = await extract_json(llm_response)

    if json_response:
        return json_response
    else:
        print("Failed to generate JSON response from LLM.")
        return None

async def generate_node_descriptions(scripts, graph_json):
    """
    Generates the node_descriptions.json based on the graph_json and scripts.
    """
    # Prepare the prompt for the LLM
    prompt = f"""
You are an assistant that generates HTML descriptions for nodes in a graph, using Tailwind CSS classes.

Here is the graph JSON:

{json.dumps(graph_json, indent=4)}

Here are the scripts:

"""

    for script in scripts:
        prompt += f"\n# {script['filename']} (Version {script['version']})\n```python\n{script['content']}\n```\n"

    prompt += """
Your task is to create a JSON object called node_descriptions, where each key is the node ID, and each value is an object containing:

{
    "description_html": "<HTML content using Tailwind CSS classes>"
}

Instructions:

- For each node in the graph JSON, generate an HTML description that provides an overview of the node.
- Use the script content to inform the description.
- The HTML should be well-structured and styled using Tailwind CSS classes.
- Ensure that the node IDs in node_descriptions match the IDs in the graph JSON.
- Return only the JSON object without any extra text or comments.

Example format:

{
    "node_id_1": {
        "description_html": "<div class='...'>...</div>"
    },
    "node_id_2": {
        "description_html": "<div class='...'>...</div>"
    },
    ...
}
"""

    # Call the LLM to generate the node descriptions
    llm_response = await call_graph(prompt)
    node_descriptions = await extract_json(llm_response)

    if node_descriptions:
        return node_descriptions
    else:
        print("Failed to generate node descriptions from LLM.")
        return None

async def extract_json(text):
    """
    Extracts JSON content from the LLM response.
    """
    # Try to find the first occurrence of '{' and the last occurrence of '}' and extract the JSON
    try:
        json_start = text.find('{')
        json_end = text.rfind('}') + 1
        json_text = text[json_start:json_end]
        json_response = json.loads(json_text)
        return json_response
    except (ValueError, json.JSONDecodeError) as e:
        print(f"JSON decode error: {e}")
        return None

async def main():
    if len(sys.argv) != 2:
        print("Usage: python3 graph_builder.py <session_dir>")
        return

    session_dir = sys.argv[1]

    # Read the AI response from the session folder
    ai_response_file = os.path.join(session_dir, 'ai_response.txt')
    if os.path.exists(ai_response_file):
        with open(ai_response_file, 'r') as f:
            ai_response = f.read().strip()
    else:
        ai_response = ""

    scripts = await read_scripts(session_dir)
    graph_json = await generate_graph(scripts, ai_response)

    if graph_json:
        # Save the graph JSON to return_json.json
        return_json_file = os.path.join(session_dir, 'return_json.json')
        with open(return_json_file, 'w') as f:
            json.dump(graph_json, f, indent=4)
        print(f"Saved graph JSON to {return_json_file}")

        # Generate node_descriptions.json
        node_descriptions = await generate_node_descriptions(scripts, graph_json)
        if node_descriptions:
            node_descriptions_file = os.path.join(session_dir, 'node_descriptions.json')
            with open(node_descriptions_file, 'w') as f:
                json.dump(node_descriptions, f, indent=4)
            print(f"Saved node descriptions to {node_descriptions_file}")
        else:
            print("No valid node descriptions generated.")

        # Copy JSON files to the specified directory
        target_dir = '/Users/ysharm12/Documents/orchestra/orchestra/orchestra-backend/app/data/'
        os.makedirs(target_dir, exist_ok=True)

        # Copy return_json.json
        shutil.copy(return_json_file, target_dir)
        print(f"Copied {return_json_file} to {target_dir}")

        # Copy node_descriptions.json
        shutil.copy(node_descriptions_file, target_dir)
        print(f"Copied {node_descriptions_file} to {target_dir}")

    else:
        print("No valid graph JSON generated.")

if __name__ == "__main__":
    asyncio.run(main())
```

---

## **Explanation of Changes**

1. **Generating `return_json.json`:**

   - The graph JSON is now saved to `return_json.json` in the session directory, overwriting it each time.

     ```python
     return_json_file = os.path.join(session_dir, 'return_json.json')
     with open(return_json_file, 'w') as f:
         json.dump(graph_json, f, indent=4)
     ```

2. **Generating `node_descriptions.json`:**

   - Added the `generate_node_descriptions` function, which:

     - Constructs a prompt for the LLM using the `graph_json` and the scripts.
     - Asks the LLM to generate HTML descriptions for each node, using Tailwind CSS classes.
     - The IDs in `node_descriptions.json` match the node IDs in `return_json.json`.

     ```python
     node_descriptions = await generate_node_descriptions(scripts, graph_json)
     ```

   - The `node_descriptions.json` is saved in the session directory, overwriting it each time.

     ```python
     node_descriptions_file = os.path.join(session_dir, 'node_descriptions.json')
     with open(node_descriptions_file, 'w') as f:
         json.dump(node_descriptions, f, indent=4)
     ```

3. **Copying JSON Files to the Specified Directory:**

   - After generating both JSON files, they are copied to the target directory:

     ```python
     target_dir = '/Users/ysharm12/Documents/orchestra/orchestra/orchestra-backend/app/data/'
     os.makedirs(target_dir, exist_ok=True)

     shutil.copy(return_json_file, target_dir)
     shutil.copy(node_descriptions_file, target_dir)
     ```

4. **Ensuring IDs Match:**

   - By using the `graph_json` directly when generating the node descriptions, we ensure that the IDs in both JSON files match.

---

## **Additional Notes**

- **Overwriting Existing JSON Files:**

  - The script overwrites `return_json.json` and `node_descriptions.json` in the session directory each time it's run.

- **Ensuring the LLM Generates Valid JSON:**

  - The prompts instruct the LLM to return only the JSON object without extra text.
  - The `extract_json` function attempts to parse the JSON from the LLM's response.

- **Error Handling:**

  - If the LLM fails to generate valid JSON, the script prints an error message.
  - It's important to monitor the LLM's output and adjust the prompts if necessary to ensure valid JSON is returned.

- **Paths and Permissions:**

  - Ensure that the script has the necessary permissions to read and write in both the session directory and the target directory.
  - The `os.makedirs` function with `exist_ok=True` ensures that the target directory exists before copying files.

---

## **Testing the Updated Script**

1. **Run `main.py` as Before:**

   - Make sure that `main.py` invokes `run_graph_builder(session_dir)` after processing user input.

2. **Verify JSON Files in Session Directory:**

   - After running `main.py` and entering a user query, check the session directory to see if `return_json.json` and `node_descriptions.json` are generated and overwritten.

3. **Check the Target Directory:**

   - Ensure that both JSON files are copied to `/Users/ysharm12/Documents/orchestra/orchestra/orchestra-backend/app/data/` and are updated.

4. **Validate JSON Content:**

   - Open the JSON files to ensure that:

     - `return_json.json` contains the graph data with nodes and edges.
     - `node_descriptions.json` contains HTML descriptions for each node, with IDs matching those in `return_json.json`.

---

## **Example of Generated `node_descriptions.json`**

Given the LLM prompt and example, the `node_descriptions.json` might look like:

```json
{
    "1": {
        "description_html": "<div class='p-6 space-y-4'><h2 class='text-2xl font-bold text-gray-900'>Input Node</h2><div class='bg-blue-50 p-4 rounded-lg'>...</div></div>"
    },
    "2": {
        "description_html": "<div class='p-6 space-y-4'><h2 class='text-2xl font-bold text-gray-900'>Processing Node</h2><div class='bg-purple-50 p-4 rounded-lg'>...</div></div>"
    },
    "3": {
        "description_html": "<div class='p-6 space-y-4'><h2 class='text-2xl font-bold text-gray-900'>Output Node</h2><div class='bg-green-50 p-4 rounded-lg'>...</div></div>"
    }
}
```

---

## **Potential Issues and Solutions**

- **LLM Fails to Generate Valid JSON:**

  - **Issue:** The LLM might include extra text or comments in its response, causing JSON parsing to fail.
  - **Solution:** Refine the prompts to be more explicit, emphasizing that it should return only the JSON object.

- **IDs Do Not Match:**

  - **Issue:** If the LLM generates node IDs that don't match those in `return_json.json`.
  - **Solution:** Ensure the prompt instructs the LLM to use the same IDs from the `graph_json`.

- **File Permissions:**

  - **Issue:** The script may not have permission to write to the target directory.
  - **Solution:** Adjust the permissions of the directories or run the script with appropriate privileges.

---

## **Summary**

- **Modified `graph_builder.py`** to:

  - Generate `return_json.json` and `node_descriptions.json` in the session directory.
  - Overwrite these files each time the graph is generated.
  - Copy both JSON files to the specified target directory.
  - Ensure node IDs match in both JSON files.

- **Ensured that the LLM is used** to generate both the graph JSON and the node descriptions, maintaining the original functionality and integrating the new requirements.

---

Please let me know if you need any further assistance or adjustments to the script!
