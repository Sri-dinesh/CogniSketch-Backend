# CogniSketch - Backend

A FastAPI backend that powers the CogniSketch AI drawing calculator. This server processes hand-drawn mathematical expressions using Google's Gemini AI and returns calculated results.

## ✨ Features

- **AI-Powered Image Analysis** - Uses Gemini 2.5 Flash model for fast, accurate recognition
- **Mathematical Expression Solving** - Handles arithmetic, algebra, and graphical problems
- **Variable Support** - Maintains context of assigned variables across calculations
- **PEMDAS Compliance** - Follows proper order of operations
- **RESTful API** - Clean, simple API endpoints
- **CORS Enabled** - Ready for cross-origin requests from frontend

## 🛠️ Tech Stack

- **FastAPI** - Modern, fast Python web framework
- **Google Generative AI** - Gemini 2.5 Flash model
- **Pillow** - Image processing
- **Uvicorn** - ASGI server
- **Pydantic** - Data validation

## 📦 Installation

1. **Clone the repository**
   ```bash
   git clone https://github.com/yourusername/cognisketch.git
   cd cognisketch/CogniSketch-Backend
   ```

2. **Create a virtual environment**
   ```bash
   python -m venv venv
   
   # Windows
   venv\Scripts\activate
   
   # macOS/Linux
   source venv/bin/activate
   ```

3. **Install dependencies**
   ```bash
   pip install -r requirements.txt
   ```

4. **Set up environment variables**
   
   Create a `.env` file in the root directory:
   ```env
   GEMINI_API_KEY=your_gemini_api_key_here
   ```

   Get your API key from [Google AI Studio](https://aistudio.google.com/app/apikey)

5. **Start the development server**
   ```bash
   python main.py
   ```

   The server will be available at `http://localhost:8900`



## 🧮 Supported Problem Types

The AI can recognize and solve:

1. **Simple Expressions** - `2 + 2`, `3 * 4`, `5 / 6`
2. **Algebraic Equations** - `x^2 + 2x + 1 = 0`
3. **Variable Assignments** - `x = 4`, `y = 5`
4. **Graphical Math Problems** - Word problems as drawings
5. **Abstract Concepts** - Drawings representing ideas


## 🔗 Related

- [CogniSketch Frontend](../CogniSketch) - The React frontend for this application
