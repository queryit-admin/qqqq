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
