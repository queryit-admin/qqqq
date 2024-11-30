"""
Orchestra Backend API

This is the main FastAPI application for the Orchestra Backend.
It provides endpoints for AI interaction and pipeline export functionality.
"""

from fastapi import FastAPI, Request, status
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError
from app.routers import ai_router, export_router, node_router
import logging
from logging.handlers import RotatingFileHandler
import os
import uuid
from typing import Union, Dict, Any

# Configure logging
log_directory = "logs"
if not os.path.exists(log_directory):
    os.makedirs(log_directory)

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        RotatingFileHandler(
            os.path.join(log_directory, "orchestra.log"),
            maxBytes=10485760,  # 10MB
            backupCount=5
        ),
        logging.StreamHandler()
    ]
)

logger = logging.getLogger(__name__)

app = FastAPI(
    title="Orchestra Backend",
    description="""
    Orchestra Backend API provides endpoints for:
    - AI-powered graph manipulation through natural language
    - Pipeline generation and export
    - Project management
    
    Use the interactive API documentation below to explore the available endpoints.
    """,
    version="1.0.0",
    docs_url="/docs",
    redoc_url="/redoc",
    openapi_url="/openapi.json",
    contact={
        "name": "Orchestra Team",
        "url": "https://github.com/yourusername/orchestra",
        "email": "your.email@example.com",
    },
    license_info={
        "name": "MIT",
        "url": "https://opensource.org/licenses/MIT",
    }
)

# Configure CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # Update this in production
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.middleware("http")
async def log_requests(request: Request, call_next):
    """Log incoming requests and outgoing responses"""
    logger.info(f"Incoming {request.method} request to {request.url.path}")
    response = await call_next(request)
    logger.info(f"Completed {request.method} request to {request.url.path} with status {response.status_code}")
    return response

@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request: Request, exc: RequestValidationError):
    """Handle validation errors in a user-friendly way"""
    error_messages = []
    for error in exc.errors():
        field = error.get("loc", ["unknown"])[-1]
        message = error.get("msg", "Invalid value")
        error_messages.append(f"{field}: {message}")
    
    logger.warning(f"Validation error for {request.url.path}: {error_messages}")
    return JSONResponse(
        status_code=status.HTTP_422_UNPROCESSABLE_ENTITY,
        content={"detail": error_messages}
    )

@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    """Global exception handler for unhandled exceptions"""
    error_id = str(uuid.uuid4())
    logger.error(f"Unhandled exception [{error_id}] for {request.url.path}: {str(exc)}", exc_info=True)
    
    return JSONResponse(
        status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
        content={
            "detail": "An unexpected error occurred. Please try again later.",
            "error_id": error_id
        }
    )

# Include routers
app.include_router(
    ai_router.router,
    prefix="/api",
    tags=["AI"]
)
app.include_router(
    export_router.router,
    tags=["Export"]
)
app.include_router(
    node_router.router,
    tags=["Nodes"]
)

@app.get(
    "/",
    summary="Health check",
    description="Returns a simple message to verify the API is running",
    response_description="A message indicating the API is running",
    responses={
        200: {
            "description": "API is running",
            "content": {
                "application/json": {
                    "example": {"message": "Orchestra Backend is running"}
                }
            }
        }
    }
)
async def read_root():
    """Health check endpoint"""
    return {"message": "Orchestra Backend is running"}
