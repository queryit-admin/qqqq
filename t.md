main.py:
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
        asyncio.run(main())
"""


graph_builder.py:
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
    asyncio.run(main())
"""


Current ai_router.py (That needs same api resonse structure but needs to be modified to cll main script:
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
