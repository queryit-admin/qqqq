Thank you for providing the code snippets and clarifying the issue.

---

## **Understanding the Issue**

- **Observation:**
  - **First User Input:** Everything works as expected—the graph is updated, and both `return_json.json` and `node_descriptions.json` are generated and copied to the data folder.
  - **Second User Input:** The graph is updated (UI reflects changes), but `return_json.json` and `node_descriptions.json` are **not** updated in the session directory or copied to the data folder.

- **Conclusion:** The issue lies in the generation or copying of `return_json.json` and `node_descriptions.json` during the second request.

---

## **Possible Cause**

After reviewing the code, I noticed a potential issue in the `graph_builder.py` script:

- **In the `generate_additional_json` function:**
  - **`session_dir` is not defined within the scope of the function.**
  - This could cause a `NameError` when the function tries to use `session_dir`.

- **Why It Might Work the First Time:**
  - If `session_dir` was globally defined or remained in the scope during the first execution, it might have worked by coincidence.
  - On subsequent runs, the variable may not be available, causing the function to fail silently.

---

## **Solution**

### **1. Modify `generate_additional_json` to Accept `session_dir` as a Parameter**

#### **Updated `graph_builder.py`**

```python
# ... [other imports remain unchanged]
import logging

logger = logging.getLogger(__name__)
logging.basicConfig(level=logging.INFO)

async def generate_additional_json(scripts, graph_json, session_dir):
    """
    Generates return_json.json and node_descriptions.json based on graph_json and scripts.
    """
    # Generate return_json.json
    return_json = graph_json  # Or process it differently if needed

    # Save return_json.json
    return_json_file = os.path.join(session_dir, 'return_json.json')
    with open(return_json_file, 'w') as f:
        json.dump(return_json, f, indent=4)
    logger.info(f"Saved return_json.json to {return_json_file}")

    # Generate node_descriptions.json
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
        # Save node_descriptions.json
        node_descriptions_file = os.path.join(session_dir, 'node_descriptions.json')
        with open(node_descriptions_file, 'w') as f:
            json.dump(node_descriptions, f, indent=4)
        logger.info(f"Saved node_descriptions.json to {node_descriptions_file}")
    else:
        logger.error("Failed to generate node descriptions from LLM.")

# Modify the main function to pass session_dir
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
    api_response = await generate_graph(scripts, ai_response)

    if api_response:
        # Save api_response.json as before
        output_file = os.path.join(session_dir, 'api_response.json')
        with open(output_file, 'w') as f:
            json.dump(api_response, f, indent=4)
        logger.info(f"Saved api_response.json to {output_file}")

        # Also generate additional JSON files
        await generate_additional_json(scripts, api_response, session_dir)

        # Copy the new JSON files to the specified directory
        target_dir = '/Users/ysharm12/Documents/orchestra/orchestra/orchestra-backend/app/data/'
        os.makedirs(target_dir, exist_ok=True)

        try:
            # Copy return_json.json
            return_json_file = os.path.join(session_dir, 'return_json.json')
            shutil.copy(return_json_file, target_dir)
            logger.info(f"Copied {return_json_file} to {target_dir}")
        except Exception as e:
            logger.error(f"Failed to copy return_json.json: {e}")

        try:
            # Copy node_descriptions.json
            node_descriptions_file = os.path.join(session_dir, 'node_descriptions.json')
            shutil.copy(node_descriptions_file, target_dir)
            logger.info(f"Copied {node_descriptions_file} to {target_dir}")
        except Exception as e:
            logger.error(f"Failed to copy node_descriptions.json: {e}")

    else:
        logger.error("No valid API response generated.")

if __name__ == "__main__":
    asyncio.run(main())
```

**Explanation:**

- **Pass `session_dir` as a parameter to `generate_additional_json`:**

  ```python
  async def generate_additional_json(scripts, graph_json, session_dir):
  ```

- **Update calls to `generate_additional_json`:**

  ```python
  await generate_additional_json(scripts, api_response, session_dir)
  ```

- **Ensure that `session_dir` is used within `generate_additional_json` without causing a `NameError`.**

### **2. Add Detailed Logging**

- **In `graph_builder.py` and `main.py`, add logging statements to capture any errors or exceptions that occur during execution.**

- **This will help identify if any exceptions are being raised silently.**

**Example in `graph_builder.py`:**

```python
import logging

# ... [rest of imports]

logger = logging.getLogger(__name__)
logging.basicConfig(level=logging.INFO)

# Add logging statements throughout the script
logger.info("Starting graph_builder.py")

# In functions, log key events and errors
logger.info("Generating graph JSON")
```

**Example in `main.py`:**

```python
import logging

logger = logging.getLogger(__name__)
logging.basicConfig(level=logging.INFO)

async def main():
    # ... [existing code]
    logger.info("Processing user input")

    ai_response, scripts = await process_user_input(user_input, session_dir)

    if scripts:
        await save_scripts(scripts, session_dir)
        logger.info(f"Saved {len(scripts)} scripts")
    else:
        logger.info("No scripts were generated or updated.")

    await save_ai_response(ai_response, session_dir)
    logger.info("Saved AI response")

    await run_graph_builder(session_dir)
    logger.info("Ran graph builder")
```

### **3. Ensure Exceptions Are Not Silently Ignored**

- **In both `main.py` and `graph_builder.py`, ensure that exceptions are logged and not ignored silently.**

- **In `run_graph_builder` in `main.py`, check for errors:**

  ```python
  async def run_graph_builder(session_dir):
      # Path to graph_builder.py
      graph_builder_path = os.path.join('/Users/ysharm12/Documents/orchestra/AI', 'graph_builder.py')

      try:
          result = subprocess.run(
              ['python3', graph_builder_path, session_dir],
              capture_output=True,
              text=True
          )
          if result.returncode != 0:
              logger.error(f"Graph builder failed with return code {result.returncode}")
              logger.error(f"stderr: {result.stderr}")
              logger.error(f"stdout: {result.stdout}")
          else:
              logger.info("Graph builder ran successfully")
      except Exception as e:
          logger.error(f"An error occurred while running graph_builder.py: {e}")
  ```

### **4. Check File Permissions and Overwriting**

- **Ensure that the scripts have permission to overwrite the files in both the session directory and the target data folder.**

- **Confirm that `shutil.copy` is overwriting existing files:**

  - By default, `shutil.copy` will overwrite the destination file if it exists.

- **However, to be safe, you can remove the existing files before copying:**

  ```python
  # Remove existing files in the target directory before copying
  target_return_json_file = os.path.join(target_dir, 'return_json.json')
  if os.path.exists(target_return_json_file):
      os.remove(target_return_json_file)
  shutil.copy(return_json_file, target_dir)
  logger.info(f"Copied {return_json_file} to {target_dir}")

  # Do the same for node_descriptions.json
  target_node_descriptions_file = os.path.join(target_dir, 'node_descriptions.json')
  if os.path.exists(target_node_descriptions_file):
      os.remove(target_node_descriptions_file)
  shutil.copy(node_descriptions_file, target_dir)
  logger.info(f"Copied {node_descriptions_file} to {target_dir}")
  ```

### **5. Verify that the AI Assistant Generates New Content**

- **Ensure that the AI assistant is generating new or updated scripts based on the second user input.**

- **If no new scripts are generated, the graph might not change, and the `graph_builder.py` might not produce updated JSON files.**

- **Check the content of the scripts and the AI response to confirm changes.**

### **6. Test the Changes**

- **After making these modifications, test the entire flow:**

  - **Send the first message and verify that all files are generated and copied correctly.**
  - **Send the second message and check if `return_json.json` and `node_descriptions.json` are updated in the session directory and copied to the data folder.**

- **Check the timestamps of the files to confirm they are updated.**

---

## **Conclusion**

- **The issue was likely caused by `session_dir` not being defined within the `generate_additional_json` function, leading to a `NameError`.**

- **By passing `session_dir` as a parameter and adding detailed logging, we can ensure that the files are generated and copied correctly, and any errors are captured.**

---

## **Additional Recommendations**

### **Ensure Consistent Session Management**

- **Since you're using a static session ID, ensure that concurrent requests do not interfere with each other.**

- **Consider using a unique session ID per user or per session if concurrency becomes an issue.**

### **Monitor Logs for Errors**

- **Regularly check the logs generated by `main.py` and `graph_builder.py` to identify any recurring issues.**

- **This will help you proactively fix problems before they affect the user experience.**

### **Validate LLM Responses**

- **Ensure that the responses from the language model (LLM) are valid and correctly formatted.**

- **If the LLM returns invalid JSON, handle the exception and log the error for debugging.**

---

## **Let Me Know If This Resolves the Issue**

- **Please implement these changes and test the system again.**

- **If the issue persists or if you encounter any new errors, feel free to share the error messages or logs, and I'll be happy to help you further.**

---

**I'm here to assist you in ensuring your AI assistant works smoothly and meets your requirements.**
