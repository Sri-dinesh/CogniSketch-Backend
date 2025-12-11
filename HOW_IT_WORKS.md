# How CogniSketch Backend Works

This document provides an in-depth explanation of how the CogniSketch backend processes hand-drawn mathematical expressions using AI and returns calculated results.

---

## Table of Contents

1. [Application Architecture](#application-architecture)
2. [Server Setup](#server-setup)
3. [API Endpoints](#api-endpoints)
4. [Image Processing Pipeline](#image-processing-pipeline)
5. [Gemini AI Integration](#gemini-ai-integration)
6. [The AI Prompt](#the-ai-prompt)
7. [Response Parsing](#response-parsing)
8. [Data Flow](#data-flow)
9. [Error Handling](#error-handling)

---

## Application Architecture

The backend is built using FastAPI, a modern Python web framework known for its speed and automatic API documentation. The architecture follows a modular structure:

```
CogniSketch-Backend/
├── main.py              # Application entry point & CORS setup
├── constants.py         # Configuration (API keys, ports)
├── schema.py            # Pydantic data models
└── apps/
    └── calculator/
        ├── route.py     # API route handlers
        └── utils.py     # Gemini AI integration logic
```

---

## Server Setup

### Entry Point (`main.py`)

The server initialization happens in `main.py`:

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI(lifespan=lifespan)

# Enable CORS for frontend access
app.add_middleware(
    CORSMiddleware,
    allow_origins=['*'],           # Allow all origins
    allow_credentials=True,
    allow_methods=["*"],           # Allow all HTTP methods
    allow_headers=["*"],           # Allow all headers
)

# Register the calculator router
app.include_router(calculator_router, prefix="/calculate", tags=["calculate"])
```

### CORS (Cross-Origin Resource Sharing)

CORS middleware is essential because the frontend (hosted on Vercel) makes requests to the backend (hosted on Render). Without CORS headers, browsers would block these cross-origin requests.

### Configuration (`constants.py`)

```python
from dotenv import load_dotenv
import os

load_dotenv()

SERVER_URL = 'localhost'
PORT = 8900
ENV = 'dev'
GEMINI_API_KEY = os.getenv('GEMINI_API_KEY')
```

The Gemini API key is loaded from environment variables for security.

---

## API Endpoints

### Health Check

```
GET /
```

Returns server status to verify the backend is running:

```python
@app.get('/')
async def root():
    return {"message": "Server is running successfully"}
```

### Calculate Expression

```
POST /calculate
```

The main endpoint that processes drawn images:

```python
@router.post('')
async def run(data: ImageData):
    # Decode base64 image
    image_data = base64.b64decode(data.image.split(",")[1])
    image_bytes = BytesIO(image_data)
    image = Image.open(image_bytes)
    
    # Analyze with AI
    responses = analyze_image(image, dict_of_vars=data.dict_of_vars)
    
    return {"message": "Image processed", "data": responses, "status": "success"}
```

---

## Image Processing Pipeline

When a request arrives, the image goes through several processing steps:

### Step 1: Receive Base64 Image

The frontend sends the canvas as a base64-encoded data URL:

```
data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAA...
```

### Step 2: Decode Base64

```python
# Split to get just the base64 part (after the comma)
image_data = base64.b64decode(data.image.split(",")[1])
```

### Step 3: Convert to PIL Image

```python
from PIL import Image
from io import BytesIO

image_bytes = BytesIO(image_data)
image = Image.open(image_bytes)
```

The image is now a PIL Image object that can be sent to the Gemini API.

### Step 4: Send to AI

```python
responses = analyze_image(image, dict_of_vars=data.dict_of_vars)
```

---

## Gemini AI Integration

The `utils.py` file contains the core AI integration logic.

### API Configuration

```python
import google.generativeai as genai
from constants import GEMINI_API_KEY

genai.configure(api_key=GEMINI_API_KEY)
```

### Model Selection

```python
model = genai.GenerativeModel(model_name="gemini-2.5-flash")
```

We use `gemini-2.5-flash` for its balance of speed and accuracy. This model is optimized for quick responses while maintaining high-quality image understanding.

### Sending the Request

```python
response = model.generate_content([prompt, img])
```

The Gemini API accepts multimodal input - both text (the prompt) and image (the drawing).

---

## The AI Prompt

The prompt is carefully crafted to guide the AI in recognizing and solving mathematical expressions. Here's a breakdown:

### Basic Instructions

```
You have been given an image with some mathematical expressions, equations, 
or graphical problems, and you need to solve them.
```

### PEMDAS Rule Explanation

```
Use the PEMDAS rule for solving mathematical expressions. 
PEMDAS stands for: Parentheses, Exponents, Multiplication and Division 
(from left to right), Addition and Subtraction (from left to right).

Example: 2 + 3 * 4 → (3 * 4) => 12, 2 + 12 = 14
```

### Five Problem Types

The prompt defines five types of problems the AI should recognize:

#### Type 1: Simple Expressions
```
Input: 2 + 2, 3 * 4, 5 / 6
Output: [{"expr": "2 + 2", "result": "4"}]
```

#### Type 2: Equations to Solve
```
Input: x^2 + 2x + 1 = 0
Output: [{"expr": "x", "result": "-1", "assign": true}]
```

#### Type 3: Variable Assignments
```
Input: x = 4
Output: [{"expr": "x", "result": "4", "assign": true}]
```

#### Type 4: Graphical Math Problems
```
Input: Drawing of a right triangle with sides labeled
Output: [{"expr": "hypotenuse", "result": "5"}]
```

#### Type 5: Abstract Concepts
```
Input: Drawing representing "love" or a historical event
Output: [{"expr": "Drawing of two hearts", "result": "love"}]
```

### Variable Context

```python
dict_of_vars_str = json.dumps(dict_of_vars, ensure_ascii=False)

# Added to prompt:
f"Here is a dictionary of user-assigned variables. If the given expression 
has any of these variables, use its actual value: {dict_of_vars_str}"
```

This allows the AI to use previously assigned variables in calculations.

### Formatting Instructions

```
DO NOT USE BACKTICKS OR MARKDOWN FORMATTING.
PROPERLY QUOTE THE KEYS AND VALUES IN THE DICTIONARY FOR EASIER PARSING.
```

These instructions ensure the response can be parsed as Python literals.

---

## Response Parsing

The AI returns a string that needs to be parsed into Python objects.

### Raw Response

```python
response = model.generate_content([prompt, img])
print(response.text)
# Output: [{"expr": "2 + 3", "result": "5"}]
```

### Parsing with ast.literal_eval

```python
import ast

try:
    answers = ast.literal_eval(response.text)
except Exception as e:
    print(f"Error in parsing response from Gemini API: {e}")
    answers = []
```

We use `ast.literal_eval` instead of `json.loads` because:
1. It's safer (doesn't execute arbitrary code)
2. It handles Python-style formatting (single quotes, etc.)

### Processing Results

```python
for answer in answers:
    if 'assign' in answer:
        answer['assign'] = True
    else:
        answer['assign'] = False
```

This normalizes the `assign` field to ensure it's always present.

---

## Data Flow

Here's the complete journey of a calculation request:

```
┌─────────────┐         ┌─────────────┐         ┌─────────────┐
│   Frontend  │         │   Backend   │         │  Gemini AI  │
│   (React)   │         │  (FastAPI)  │         │   (Google)  │
└─────────────┘         └─────────────┘         └─────────────┘
       │                       │                       │
       │  1. POST /calculate   │                       │
       │  {image, dict_of_vars}│                       │
       │──────────────────────>│                       │
       │                       │                       │
       │                       │  2. Decode base64     │
       │                       │  Convert to PIL Image │
       │                       │                       │
       │                       │  3. Send image+prompt │
       │                       │──────────────────────>│
       │                       │                       │
       │                       │                       │  4. Analyze image
       │                       │                       │  Solve expression
       │                       │                       │
       │                       │  5. Return solution   │
       │                       │<──────────────────────│
       │                       │                       │
       │                       │  6. Parse response    │
       │                       │  Format results       │
       │                       │                       │
       │  7. Return results    │                       │
       │  {data: [...]}        │                       │
       │<──────────────────────│                       │
       │                       │                       │
       │  8. Display result    │                       │
       │  as LaTeX card        │                       │
       │                       │                       │
```

### Detailed Steps

1. **Frontend sends request**: Canvas image as base64 + any stored variables
2. **Backend decodes image**: Base64 → bytes → PIL Image
3. **Backend calls Gemini**: Sends image + detailed prompt
4. **Gemini analyzes**: Uses vision model to read handwriting and solve
5. **Gemini responds**: Returns structured answer as text
6. **Backend parses**: Converts text to Python objects
7. **Backend responds**: Sends formatted JSON to frontend
8. **Frontend displays**: Renders result as draggable LaTeX card

---

## Error Handling

### API Errors

```python
try:
    response = model.generate_content([prompt, img])
    answers = ast.literal_eval(response.text)
except Exception as e:
    print(f"Error in parsing response from Gemini API: {e}")
    answers = []
```

If the AI returns unparseable text, an empty array is returned.

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| CORS Error | Backend not allowing origin | Check CORS middleware config |
| 500 Error | API key invalid/missing | Verify `GEMINI_API_KEY` env var |
| Parse Error | AI returned malformed text | Retry or improve prompt |
| Timeout | Large image or slow network | Reduce image size |

---

## Request/Response Schema

### Request Schema (`schema.py`)

```python
from pydantic import BaseModel

class ImageData(BaseModel):
    image: str          # Base64 encoded image
    dict_of_vars: dict  # Previously assigned variables
```

Pydantic automatically validates incoming requests.

### Response Structure

```json
{
  "message": "Image processed",
  "status": "success",
  "data": [
    {
      "expr": "2 + 3 * 4",
      "result": "14",
      "assign": false
    }
  ]
}
```

For variable assignments:

```json
{
  "data": [
    {
      "expr": "x",
      "result": "10",
      "assign": true
    }
  ]
}
```

---

## Summary

The CogniSketch backend:

1. **Receives** hand-drawn images as base64-encoded data
2. **Decodes** and converts them to PIL Image objects
3. **Sends** images to Gemini AI with a detailed prompt
4. **Parses** the AI's response into structured data
5. **Returns** calculation results to the frontend
6. **Maintains** variable context across multiple calculations

The AI is guided by a carefully crafted prompt that handles five types of mathematical problems, from simple arithmetic to abstract concept recognition.
