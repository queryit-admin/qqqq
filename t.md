"""
Node Router Module

This module handles endpoints related to node information and descriptions.
"""

import json
import os
from fastapi import APIRouter, HTTPException, status
from typing import Dict, Any

# Configure router
router = APIRouter(
    prefix="/api/nodes",
    tags=["Nodes"],
    responses={
        404: {"description": "Node not found"},
        500: {"description": "Internal server error"},
    }
)

# Load node descriptions from JSON file
def load_node_descriptions() -> Dict[str, Any]:
    """Load node descriptions from the JSON file."""
    try:
        file_path = os.path.join(os.path.dirname(__file__), "../data/node_descriptions.json")
        with open(file_path, "r") as f:
            return json.load(f)
    except Exception as e:
        print(f"Error loading node descriptions: {str(e)}")
        return {}

@router.get(
    "/{node_id}/description",
    response_model=Dict[str, str],
    summary="Get node description",
    description="Get the HTML description for a specific node",
    responses={
        200: {
            "description": "Successful response",
            "content": {
                "application/json": {
                    "example": {
                        "description_html": "<div class='p-4'>...</div>"
                    }
                }
            }
        },
        404: {
            "description": "Node not found",
            "content": {
                "application/json": {
                    "example": {
                        "detail": "Node not found"
                    }
                }
            }
        }
    }
)
async def get_node_description(node_id: str) -> Dict[str, str]:
    """
    Get the HTML description for a specific node.
    
    Args:
        node_id: ID of the node
        
    Returns:
        Dict containing HTML description
        
    Raises:
        HTTPException: 
            - 404 if node not found
            - 500 if there's an error loading descriptions
    """
    descriptions = load_node_descriptions()
    
    if not descriptions:
        raise HTTPException(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            detail="Error loading node descriptions"
        )
    
    if node_id not in descriptions:
        # Return a default description for unknown nodes
        return {
            "description_html": f"""
            <div class='p-6 space-y-4'>
                <h2 class='text-2xl font-bold text-gray-900'>Node {node_id}</h2>
                <div class='bg-gray-50 p-4 rounded-lg'>
                    <h3 class='text-lg font-semibold text-gray-900 mb-2'>Overview</h3>
                    <p class='text-gray-700'>This is a custom node in your pipeline.</p>
                </div>
            </div>
            """
        }
    
    return descriptions[node_id]
