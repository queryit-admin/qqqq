Certainly! Based on your requirements, you want to:

- **Keep `main.py` unchanged** or with minimal modifications.
- **Use the existing functions** in `main.py` and other modules to process user input.
- **Modify `ai_router.py`** to import and use these functions in the `/message` endpoint.
- **Ensure the AI response and graph output** are returned in the expected JSON format.

---

## Solution Overview

To achieve this, we'll:

1. **Import necessary functions** from `main.py` and other modules into `ai_router.py`.
2. **Reuse the existing logic** in `main.py` within the `/message` endpoint of `ai_router.py`.
3. **Minimize modifications to `main.py`** by ensuring all functions are importable.
4. **Ensure the graph output** matches the expected structure for the API response.

---

## Step-by-Step Implementation

### 1. Ensure Functions in `main.py` are Importable

Since you want to keep `main.py` mostly untouched, we'll make sure the functions `save_scripts`, `save_ai_response`, and `run_graph_builder` are defined at the module level and can be imported.

In your existing `main.py`, these functions are already defined at the module level, so they can be imported without modifying `main.py`.

### 2. Modify `ai_router.py` to Use Functions from `main.py`

We'll import the necessary functions into `ai_router.py` and use them in the `/message` endpoint.

---

### Updated `ai_router.py`

```python
# ai_router.py

"""
AI Router Module

This module handles all AI-related endpoints, including message processing
and graph manipulation through natural language commands.
"""

from fastapi import APIRouter, HTTPException, status
from app.schemas.schemas import MessageRequest, MessageResponse
from typing import List
import logging
import os
import json

# Import necessary functions
from parent_agent import process_user_input
from main import save_scripts, save_ai_response, run_graph_builder

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
                            "type": "custom",
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
        
        # Use a static session ID (you can modify this to handle dynamic sessions)
        session_id = 'static_session'
        session_dir = os.path.join('sessions', session_id)
        os.makedirs(session_dir, exist_ok=True)
        
        # Process the user input using the existing logic
        ai_response, scripts = await process_user_input(request.user_message, session_dir)
        
        if scripts:
            # Save scripts to files
            await save_scripts(scripts, session_dir)
        
        # Save the AI response
        await save_ai_response(ai_response, session_dir)
        
        # Update the graph
        await run_graph_builder(session_dir)
        
        # Read the graph output
        api_response_file = os.path.join(session_dir, 'api_response.json')
        if os.path.exists(api_response_file):
            with open(api_response_file, 'r') as f:
                api_response = json.load(f)
        else:
            # If no graph output, create a default response
            api_response = {
                "ai_response": ai_response,
                "nodes": [],
                "edges": []
            }
        
        # Construct the MessageResponse object
        response = MessageResponse(
            ai_response=api_response.get('ai_response', ai_response),
            nodes=api_response.get('nodes', []),
            edges=api_response.get('edges', [])
        )
        
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
```

**Explanation:**

- **Imported Functions:**
  - `process_user_input` from `parent_agent.py`
  - `save_scripts`, `save_ai_response`, and `run_graph_builder` from `main.py`

- **Session Management:**
  - Used a static `session_id`. You can modify this to handle per-user sessions if needed.

- **Processing Logic:**
  - Reused the logic from `main.py` within the `/message` endpoint.
  - After processing, read the graph output from `api_response.json`.

- **Constructing the Response:**
  - Created a `MessageResponse` object using the AI response and graph data.

- **Error Handling:**
  - Added exception handling to return appropriate HTTP responses.

---

### 3. Ensure Models and Imports are Correct

Make sure that the `MessageRequest` and `MessageResponse` models in `app.schemas.schemas` are compatible with the data being returned.

```python
# app/schemas/schemas.py

from pydantic import BaseModel
from typing import List, Dict, Any

class MessageRequest(BaseModel):
    user_message: str
    conversation_history: List[str] = []

class MessageResponse(BaseModel):
    ai_response: str
    nodes: List[Dict[str, Any]] = []
    edges: List[Dict[str, Any]] = []
```

---

### 4. No Changes to `main.py`

As per your request, we've made no modifications to `main.py`. All functions are imported as is.

---

## Final Notes

- **Function Importing:**
  - Since the functions `save_scripts`, `save_ai_response`, and `run_graph_builder` are defined at the module level in `main.py`, they can be imported directly without modifying `main.py`.

- **Asynchronous Functions:**
  - All functions are asynchronous (`async def`), so remember to use `await` when calling them.

- **Session Management:**
  - Currently uses a static session. For multiple users, consider implementing dynamic session IDs.

- **Logging:**
  - Ensure logging is properly configured to capture logs from both `main.py` and `ai_router.py`.

- **Error Handling:**
  - Adjust exception handling as needed to provide appropriate responses and logs.

- **Testing:**
  - Test the API endpoint thoroughly to ensure it behaves as expected.

---

## Conclusion

By importing the necessary functions from `main.py` into `ai_router.py` and reusing the existing logic, we've met your requirements:

- **`main.py` remains unchanged.**
- **Processing logic is reused within the `/message` endpoint.**
- **The AI response and graph data are returned in the expected JSON format.**

This approach allows you to integrate the functionality of `main.py` into your FastAPI application with minimal modifications.

---

**Let me know if this meets your requirements or if you need any further assistance!**
