I need main.py to act like this route so i can change this route file to point to my main


The fastapi current route:
ai_router.py:
"""
"""
AI Router Module

This module handles all AI-related endpoints, including message processing
and graph manipulation through natural language commands.
"""

from fastapi import APIRouter, HTTPException, status
from app.schemas.schemas import MessageRequest, MessageResponse
from app.services.ai_service import generate_ai_response
from typing import List
import logging

# Configure logging
logger = logging.getLogger(__name__)

router = APIRouter(
    prefix="/ai",
    tags=["AI"],
    responses={
        404: {"description": "Not found"},
        500: {"description": "Internal server error"},
    }
)

@router.post(
    "/message",
    response_model=MessageResponse,
    summary="Process a user message",
    description="""
    Process a message from the user and return an AI response along with any graph updates.
    
    The endpoint accepts natural language commands such as:
    - "add node": Creates a new node
    - "remove node X": Removes node number X
    - "list nodes": Lists all current nodes
    
    The response includes:
    - AI's text response
    - Any new or updated nodes
    - Any new or updated edges between nodes
    """,
    responses={
        200: {
            "description": "Successful response",
            "content": {
                "application/json": {
                    "example": {
                        "ai_response": "Node 1 added successfully.",
                        "nodes": [{
                            "id": "123e4567-e89b-12d3-a456-426614174000",
                            "type": "default",
                            "data": {"label": "Node 1"},
                            "position": {"x": 250.0, "y": 150.0}
                        }],
                        "edges": []
                    }
                }
            }
        },
        400: {
            "description": "Bad request",
            "content": {
                "application/json": {
                    "example": {
                        "detail": "User message cannot be empty."
                    }
                }
            }
        }
    }
)
async def process_message(request: MessageRequest):
    """
    Process a message from the user and generate an AI response with graph updates.
    
    Args:
        request: The message request containing user message and conversation history
        
    Returns:
        MessageResponse containing AI's response and any graph updates
        
    Raises:
        HTTPException: 
            - 400 if message is empty or invalid
            - 422 if conversation history is invalid
            - 500 if AI service encounters an error
    """
    logger.info(f"Processing user message: {request.user_message}")
    try:
        # Validate user message
        if not request.user_message or not request.user_message.strip():
            raise HTTPException(
                status_code=status.HTTP_400_BAD_REQUEST,
                detail="User message cannot be empty."
            )
            
        # Validate conversation history
        if request.conversation_history:
            if not isinstance(request.conversation_history, List):
                raise HTTPException(
                    status_code=status.HTTP_422_UNPROCESSABLE_ENTITY,
                    detail="Conversation history must be a list of strings."
                )
            if any(not isinstance(msg, str) for msg in request.conversation_history):
                raise HTTPException(
                    status_code=status.HTTP_422_UNPROCESSABLE_ENTITY,
                    detail="All conversation history items must be strings."
                )
            
        # Generate response
        response = generate_ai_response(request.user_message, request.conversation_history)
        logger.info(f"Successfully generated AI response for user message: {request.user_message}")
        return response
        
    except HTTPException:
        # Re-raise HTTP exceptions
        raise
    except Exception as e:
        # Log the actual error but return a generic message
        logger.error(f"Error processing AI message: {str(e)}", exc_info=True)
        raise HTTPException(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            detail="An error occurred while processing your request. Please try again later."
        )

"""




all_scripts.txt:
"""
# parent_agent.py

import json
import re
import os
import uuid
from llm import call_parent
from constants import MAX_HISTORY_TOKENS
from estimate_tokens import estimate_tokens

async def process_user_input(user_input, session_dir):
    """
    Processes the user's input, determining whether to generate new scripts or update existing ones.
    Returns the AI's response to the user and the list of scripts.
    """
    history_file = os.path.join(session_dir, 'parent_history.txt')
    history = await read_history(history_file)

    # Append the user's input to the history
    user_message = {"role": "user", "content": user_input}
    history.append(user_message)
    history = await trim_history(history)
    await write_history(history_file, history)

    # Construct the prompt with history and context
    prompt = await construct_prompt(history, session_dir)

    parent_response = await call_parent(prompt)
    print("Parent Response:")
    print(parent_response)  # For debugging purposes

    # Append the assistant's response to the history
    assistant_message = {"role": "assistant", "content": parent_response}
    history.append(assistant_message)
    history = await trim_history(history)
    await write_history(history_file, history)

    # Extract AI response and scripts from the parent agent's reply
    ai_response = await extract_ai_response(parent_response)
    scripts = await extract_scripts(parent_response)
    return ai_response, scripts

async def construct_prompt(history, session_dir):
    """
    Constructs the prompt for the parent agent, including history and context.
    """
    # Build the conversation history into the prompt
    prompt = ""
    for message in history:
        role = "User" if message["role"] == "user" else "Assistant"
        prompt += f"{role}: {message['content']}\n"

    # Include the existing scripts with versioning
    scripts = []
    script_files = [f for f in os.listdir(session_dir) if f.endswith('.py')]
    for script_name in script_files:
        script_path = os.path.join(session_dir, script_name)
        with open(script_path, 'r') as f:
            script_content = f.read()
        version = await extract_version(script_content) or 1
        scripts.append({
            "filename": script_name,
            "content": script_content,
            "version": version
        })

    prompt += "\nHere are the existing scripts:\n"
    for script in scripts:
        prompt += f"\n# {script['filename']} (Version {script['version']})\n```python\n{script['content']}\n```\n"

    # Append instructions for the assistant
    prompt += """
You are a reusable agentic architecture designed by Yash Sharma from the EAA team inside Optum, a part of UnitedHealth Group.
Your name is Orchestra.

As an expert Python software developer assistant, you should:

- Analyze the user's request and understand the required changes.
- Decide which scripts need to be updated or if new scripts need to be created.
- Only provide the scripts that need to be updated or new scripts that need to be created.
- Do not include scripts that do not need changes.
- Maintain versioning of scripts. When updating a script, increment its version number.
- Provide a brief AI response to the user explaining what you've done or asking for clarification if needed.
- Ensure that all scripts are properly formatted and include comments at the top with the filename and version number.
- Think of the scripts you create as a pipeline, so build the minimum scripts needed and reuse functions from existing scripts.
- Aim to generate 2 to 5 scripts, adding more only if necessary.
- Ensure each script you generate uses functions from other scripts to reduce redundancy and increase reusability.
- At the end, there should always be a main file utilizing all the other files.

**Formatting Instructions:**

- Start your response with `AI Response:` followed by your message to the user (mention any updates to the blueprint visible in the UI).
- Simplify the AI response; you're communicating with users who may not be technical or developers.
- Then, provide only the updated or new scripts in one output.
- Do not provide scripts that have not been changed.
- Each script should be enclosed in triple backticks with the language identifier `python` like so:

```python
# filename.py (Version X)
# Description:
# This script is responsible for...

# Code starts here
def function():
    pass
```

- Do not include any additional text outside of the AI response and code blocks.
- Ensure that each script starts with a comment containing the filename and version number.
"""

    return prompt

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

async def extract_ai_response(response_text):
    """
    Extracts the AI's response to the user from the parent agent's response.
    """
    ai_response_pattern = r'AI Response:\s*(.*?)(?=```python|$)'
    match = re.search(ai_response_pattern, response_text, re.DOTALL)
    if match:
        return match.group(1).strip()
    else:
        return "No AI response found."

async def extract_scripts(response_text):
    """
    Extracts Python scripts from the parent agent's response.
    """
    scripts = []
    script_pattern = r'```python\s*([\s\S]*?)```'
    matches = re.findall(script_pattern, response_text)
    for match in matches:
        script_content = match.strip()
        # Extract filename and version from the script content
        header_match = re.match(r'#\s*(.*\.py)\s*\(Version\s*(\d+)\)', script_content)
        if header_match:
            filename = header_match.group(1).strip()
            version = int(header_match.group(2))
            # Remove the header from the script content
            script_body = script_content[header_match.end():].strip()
            scripts.append({
                "filename": filename,
                "version": version,
                "content": script_body
            })
        else:
            # If no header match, generate a filename
            scripts.append({
                "filename": f'script_{uuid.uuid4().hex[:8]}.py',
                "version": 1,
                "content": script_content
            })
    return scripts

async def read_history(file_path):
    """
    Reads the conversation history from the session folder.
    """
    history = []
    if os.path.exists(file_path):
        with open(file_path, 'r') as f:
            for line in f:
                history.append(json.loads(line.strip()))
    return history

async def write_history(file_path, history):
    """
    Writes the conversation history to the session folder.
    """
    with open(file_path, 'w') as f:
        for message in history:
            f.write(json.dumps(message) + '\n')

async def trim_history(history):
    """
    Trims the conversation history to stay within token limits.
    """
    total_tokens = 0
    trimmed_history = []
    for message in reversed(history):
        tokens = await estimate_tokens(message['content'])
        if total_tokens + tokens <= MAX_HISTORY_TOKENS:
            trimmed_history.insert(0, message)
            total_tokens += tokens
        else:
            break
    return trimmed_history"""


all_scripts.txt:
"""
import os

def write_script_to_file(filename, script_path):
  """
  Reads the content of a Python script and appends it to a file with formatting.
  """
  try:
    with open(script_path, 'r') as f:
      script_content = f.read()
  except FileNotFoundError:
    script_content = f"** File not found: {script_path} **"
  
  with open(filename, 'a') as f:
    f.write(f"\n\n{filename}:\n")
    f.write('"""\n')
    f.write(script_content)
    f.write('"""\n')

def main():
  # Replace "your_directory" with the actual directory containing Python scripts
  script_dir = "/Users/ysharm12/Documents/orchestra"
  output_file = "all_scripts.txt"

  # Clear the output file before writing
  with open(output_file, 'w') as f:
    f.write("")

  for filename in os.listdir(script_dir):
    if filename.endswith(".py"):
      script_path = os.path.join(script_dir, filename)
      write_script_to_file(output_file, script_path)

if __name__ == "__main__":
  main()
"""


all_scripts.txt:
"""
async def estimate_tokens(text):
    """
    Estimates the number of tokens in the text.
    """
    # Simple estimation: average 4 characters per token
    return len(text) // 4"""


all_scripts.txt:
"""
MAX_HISTORY_TOKENS = 128000"""


all_scripts.txt:
"""
# call_openai.py

import openai
import os
from openai import AsyncAzureOpenAI

client = AsyncAzureOpenAI(
    azure_endpoint="https://dycsb0qcdmbjek0openai.openai.azure.com/",
    api_key="61708ba3ecb24abcbf691b8126feb905",
    api_version="2024-08-01-preview"
)


# Set OpenAI Azure Configuration
openai.api_type = "azure"
openai.azure_endpoint = "https://dycsb0qcdmbjek0openai.openai.azure.com/"
openai.api_version = "2024-08-01-preview"
openai.api_key = "61708ba3ecb24abcbf691b8126feb905"

# Model names
deployment_parent = "o1-preview"
deployment_child = "o1-mini"
embedding_model = "text-embedding-3-large"

async def call_parent(prompt):
    try:

        response = await client.chat.completions.create(
            model="o1-mini",
            messages=[{"role": "user", "content": prompt}]
        )
        result = response.choices[0].message.content.strip()
        return result
    except openai.error.Timeout as e:
        # Handle request timeout
        print(f"Request timed out: {e}")
    except openai.error.APIConnectionError as e:
        # Handle connection errors
        print(f"Connection error: {e}")
    except openai.error.InvalidRequestError as e:
        # Handle invalid requests
        print(f"Invalid request: {e}")
    except openai.error.AuthenticationError as e:
        # Handle authentication errors
        print(f"Authentication error: {e}")
    except openai.error.PermissionError as e:
        # Handle permission errors
        print(f"Permission error: {e}")
    except openai.error.RateLimitError as e:
        # Handle rate limit errors
        print(f"Rate limit exceeded: {e}")
    except openai.APIError as e:
        # Handle other API errors
        print(f"API error: {e}")
    except Exception as e:
        # Handle unexpected errors
        print(f"An unexpected error occurred: {e}")
    
async def call_graph(prompt):
    try:

        response = await client.chat.completions.create(
            model="o1-mini",
            messages=[{"role": "user", "content": prompt}]
        )
        result = response.choices[0].message.content.strip()
        return result
    except openai.error.Timeout as e:
        # Handle request timeout
        print(f"Request timed out: {e}")
    except openai.error.APIConnectionError as e:
        # Handle connection errors
        print(f"Connection error: {e}")
    except openai.error.InvalidRequestError as e:
        # Handle invalid requests
        print(f"Invalid request: {e}")
    except openai.error.AuthenticationError as e:
        # Handle authentication errors
        print(f"Authentication error: {e}")
    except openai.error.PermissionError as e:
        # Handle permission errors
        print(f"Permission error: {e}")
    except openai.error.RateLimitError as e:
        # Handle rate limit errors
        print(f"Rate limit exceeded: {e}")
    except openai.APIError as e:
        # Handle other API errors
        print(f"API error: {e}")
    except Exception as e:
        # Handle unexpected errors
        print(f"An unexpected error occurred: {e}")
    

# print(call_parent("Whats up?"))
# print(call_child("Whats up?"))
# print(call_embedding("Whats up?"))"""


all_scripts.txt:
"""
# graph_builder.py

import os
import re
import json
import asyncio
import sys
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
    api_response = await generate_graph(scripts, ai_response)

    if api_response:
        # Output the API response
        print(json.dumps(api_response, indent=4))
        # Optionally, save the response to a file
        output_file = os.path.join(session_dir, 'api_response.json')
        with open(output_file, 'w') as f:
            json.dump(api_response, f, indent=4)
    else:
        print("No valid API response generated.")

if __name__ == "__main__":
    asyncio.run(main())"""


all_scripts.txt:
"""
# main.py

import asyncio
import os
import uuid
import re
import subprocess
from parent_agent import process_user_input
from llm import call_parent

async def main():
    # Use a static session ID
    session_id = 'static_session'

    # Create a session directory
    session_dir = os.path.join('sessions', session_id)
    os.makedirs(session_dir, exist_ok=True)

    while True:
        user_input = input("\nEnter your query or modification request: ")

        ai_response, scripts = await process_user_input(user_input, session_dir)

        if not scripts:
            print("No scripts were generated or updated.")
            continue

        # Save scripts to files
        await save_scripts(scripts, session_dir)

        # Save the AI response
        await save_ai_response(ai_response, session_dir)

        # Display the AI response
        print("\nAI Response:")
        print(ai_response)

        # Update the graph
        await run_graph_builder(session_dir)

        # Continue the loop without prompting

async def save_scripts(scripts, session_dir):
    """
    Saves the scripts to files in the session directory.
    Removes older versions when a script is updated.
    """
    for script in scripts:
        filename = script["filename"]
        version = script["version"]
        content = script["content"]

        # Remove older versions of the script
        existing_files = [f for f in os.listdir(session_dir) if f == filename]
        for existing_file in existing_files:
            os.remove(os.path.join(session_dir, existing_file))

        # Save the new script
        script_path = os.path.join(session_dir, filename)
        with open(script_path, 'w') as f:
            # Add the header with filename and version
            header = f"# {filename} (Version {version})\n"
            f.write(header + '\n' + content)

        print(f"Saved script: {filename} (Version {version})")

async def save_ai_response(ai_response, session_dir):
    """
    Saves the AI's response to the user in the session directory.
    """
    ai_response_file = os.path.join(session_dir, 'ai_response.txt')
    with open(ai_response_file, 'w') as f:
        f.write(ai_response)

async def run_graph_builder(session_dir):
    """
    Runs the graph_builder.py script to generate the API response.
    """
    try:
        result = subprocess.run(
            ['python3', 'graph_builder.py', session_dir],
            capture_output=True,
            text=True
        )
        if result.returncode == 0:
            api_response = result.stdout
            print("\nUpdated API Response:")
            print(api_response)
        else:
            print("Graph builder failed:")
            print(result.stderr)
    except Exception as e:
        print(f"An error occurred while running graph_builder.py: {e}")

if __name__ == "__main__":
        asyncio.run(main())"""
