>git clone https://github.com/polashds/ai-saas-apps-fullstack-deployment.git

## Project Overview

We'll build an AI-powered application with:
- **Frontend**: Next.js (TypeScript), Tailwind CSS, deployed on Vercel
- **Backend**: FastAPI, Docker, AWS Lambda

## Checklist

### Phase 1: Project Setup
- [ ] Initialize project structure
- [ ] Set up backend (FastAPI)
- [ ] Configure development environment
- [ ] Set up frontend (Next.js)
- [ ] Configure development environment

### Phase 2: Backend Development
- [ ] Set up FastAPI routes
- [ ] Implement AI model integration
- [ ] Configure vector database
- [ ] Create authentication system

### Phase 3: Frontend Development
- [ ] Create basic pages and components
- [ ] Implement API communication
- [ ] Add state management
- [ ] Style with Tailwind CSS

### Phase 4: Integration
- [ ] Connect frontend to backend
- [ ] Implement API security
- [ ] Set up CI/CD pipelines

### Phase 5: Deployment
- [ ] Dockerize applications
- [ ] Deploy frontend to Vercel
- [ ] Deploy backend to AWS
- [ ] Configure production environment variables

### Phase 6: Monitoring & Maintenance
- [ ] Set up logging
- [ ] Implement monitoring
- [ ] Plan for scaling

## Phase 1: Project Setup

### 1.1 Initialize Project Structure

```
ai-app/
├── frontend/          # Next.js application
├── backend/           # FastAPI application
├── docker/            # Docker configurations
├── scripts/           # Deployment scripts
└── README.md
```



### 1.2 Set Up Backend (FastAPI)

```bash
mkdir backend && cd backend
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
pip install fastapi uvicorn python-dotenv
```

Create `backend/main.py`:
```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

# CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.get("/")
async def root():
    return {"message": "AI Backend is running"}
```

Create `backend/requirements.txt`:
```
fastapi==0.95.2
uvicorn==0.22.0
python-dotenv==1.0.0
```

### 1.3 Set Up Frontend (Next.js)

```bash
npx create-next-app@latest frontend --typescript --eslint --tailwind --src-dir --app --import-alias "@/*"
cd frontend
```

Configure `frontend/tsconfig.json`:
```json
{
  "compilerOptions": {
    "target": "es5",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "forceConsistentCasingInFileNames": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "node",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

### 1.4 Configure Development Environment

Create `.env` in both frontend and backend:

Frontend `.env`:
```env
NEXT_PUBLIC_API_URL=http://localhost:8000
```

Backend `.env`:
```env
APP_ENV=development
DATABASE_URL=postgresql://user:password@localhost:5432/ai_app
VECTOR_DB_URL=http://localhost:8080
OPENAI_API_KEY=your_key_here
```
## Phase 2: Backend Development

### 2.1 Set Up FastAPI Routes

Update `backend/main.py`:
```python
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import os
from typing import Optional
import openai

app = FastAPI()

# Models
class ChatRequest(BaseModel):
    prompt: str
    context: Optional[list] = None

class ChatResponse(BaseModel):
    response: str
    context: Optional[list] = None

# Routes
@app.post("/api/v1/ai/chat", response_model=ChatResponse)
async def chat_with_ai(request: ChatRequest):
    try:
        # Initialize OpenAI client
        openai.api_key = os.getenv("OPENAI_API_KEY")
        
        # Call OpenAI API
        completion = openai.ChatCompletion.create(
            model="gpt-3.5-turbo",
            messages=[
                {"role": "system", "content": "You are a helpful assistant."},
                {"role": "user", "content": request.prompt}
            ]
        )
        
        return {
            "response": completion.choices[0].message.content,
            "context": request.context
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### 2.2 Configure Vector Database (Weaviate)

Install Weaviate client:
```bash
pip install weaviate-client
```

Create `backend/vector_db.py`:
```python
import weaviate
import os

class VectorDB:
    def __init__(self):
        self.client = weaviate.Client(
            url=os.getenv("VECTOR_DB_URL"),
            additional_headers={
                "X-OpenAI-Api-Key": os.getenv("OPENAI_API_KEY")
            }
        )
        
    def create_schema(self):
        schema = {
            "classes": [
                {
                    "class": "Document",
                    "description": "A document with text content",
                    "properties": [
                        {
                            "name": "content",
                            "dataType": ["text"],
                            "description": "The content of the document",
                        }
                    ],
                    "vectorizer": "text2vec-openai"
                }
            ]
        }
        self.client.schema.create(schema)
        
    def add_document(self, text: str):
        self.client.data_object.create(
            data_object={"content": text},
            class_name="Document"
        )
        
    def search_similar(self, query: str, limit: int = 3):
        result = self.client.query.get(
            "Document",
            ["content"]
        ).with_near_text({
            "concepts": [query]
        }).with_limit(limit).do()
        
        return result.get("data", {}).get("Get", {}).get("Document", [])
```

Update `backend/main.py` to include vector search:
```python
from vector_db import VectorDB

vector_db = VectorDB()

@app.post("/api/v1/ai/chat-with-context", response_model=ChatResponse)
async def chat_with_context(request: ChatRequest):
    try:
        # Search vector DB for relevant context
        similar_docs = vector_db.search_similar(request.prompt)
        context = [doc["content"] for doc in similar_docs]
        
        # Call OpenAI API with context
        openai.api_key = os.getenv("OPENAI_API_KEY")
        
        system_message = {
            "role": "system",
            "content": "You are a helpful assistant. Use the following context if relevant:\n" + "\n".join(context)
        }
        
        completion = openai.ChatCompletion.create(
            model="gpt-3.5-turbo",
            messages=[system_message, {"role": "user", "content": request.prompt}]
        )
        
        return {
            "response": completion.choices[0].message.content,
            "context": context
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

## Phase 3: Frontend Development

### 3.1 Create Basic Pages and Components

Create `frontend/src/app/page.tsx`:
```tsx
import { Inter } from 'next/font/google'
import { useState } from 'react'

const inter = Inter({ subsets: ['latin'] })

export default function Home() {
  const [input, setInput] = useState('')
  const [response, setResponse] = useState('')
  const [isLoading, setIsLoading] = useState(false)

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()
    setIsLoading(true)
    try {
      const res = await fetch('/api/ai', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({ prompt: input }),
      })
      const data = await res.json()
      setResponse(data.result)
    } catch (error) {
      console.error(error)
      setResponse('Error processing your request')
    } finally {
      setIsLoading(false)
    }
  }

  return (
    <main className={`min-h-screen p-24 ${inter.className}`}>
      <h1 className="text-4xl font-bold mb-8">AI Application</h1>
      
      <form onSubmit={handleSubmit} className="mb-8">
        <div className="flex gap-2">
          <input
            type="text"
            value={input}
            onChange={(e) => setInput(e.target.value)}
            className="flex-1 p-2 border rounded"
            placeholder="Ask AI anything..."
          />
          <button 
            type="submit" 
            className="bg-blue-500 text-white p-2 rounded hover:bg-blue-600"
            disabled={isLoading}
          >
            {isLoading ? 'Processing...' : 'Submit'}
          </button>
        </div>
      </form>

      {response && (
        <div className="bg-gray-100 p-4 rounded">
          <h2 className="text-xl font-semibold mb-2">AI Response:</h2>
          <p>{response}</p>
        </div>
      )}
    </main>
  )
}
```

### 3.2 Implement API Communication

Create `frontend/src/app/api/ai/route.ts`:
```typescript
import { NextResponse } from 'next/server'

export async function POST(request: Request) {
  try {
    const { prompt } = await request.json()
    
    // Forward the request to our FastAPI backend
    const backendResponse = await fetch(
      `${process.env.NEXT_PUBLIC_API_URL}/api/v1/ai/chat`,
      {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({ prompt }),
      }
    )

    if (!backendResponse.ok) {
      throw new Error('Backend request failed')
    }

    const data = await backendResponse.json()
    return NextResponse.json({ result: data.response })
  } catch (error) {
    console.error('AI API error:', error)
    return NextResponse.json(
      { error: 'Error processing your request' },
      { status: 500 }
    )
  }
}
```



## Phase 4: Integration

### 4.1 Connect Frontend to Backend

Update the frontend API route to use the context-aware endpoint:

```typescript
// frontend/src/app/api/ai/route.ts
export async function POST(request: Request) {
  try {
    const { prompt, context } = await request.json()
    
    const backendResponse = await fetch(
      `${process.env.NEXT_PUBLIC_API_URL}/api/v1/ai/chat-with-context`,
      {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({ prompt, context }),
      }
    )

    // ... rest of the code
  }
}
```

### 4.2 Implement API Security

Create `backend/auth.py`:
```python
from fastapi import Depends, HTTPException, status
from fastapi.security import APIKeyHeader
from typing import Annotated

API_KEY_NAME = "X-API-KEY"
api_key_header = APIKeyHeader(name=API_KEY_NAME, auto_error=False)

async def get_api_key(api_key: str = Depends(api_key_header)):
    if api_key != os.getenv("API_KEY"):
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Invalid API Key"
        )
    return api_key
```

Update routes to require authentication:
```python
from auth import get_api_key

@app.post("/api/v1/ai/chat-with-context", dependencies=[Depends(get_api_key)])
async def chat_with_context(request: ChatRequest):
    # ... existing code
```

## Phase 5: Deployment

### 5.1 Dockerize Applications

Create `backend/Dockerfile`:
```dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Create `docker-compose.yml`:
```yaml
version: '3.8'

services:
  backend:
    build: ./backend
    ports:
      - "8000:8000"
    env_file:
      - .env
    depends_on:
      - weaviate

  weaviate:
    image: semitechnologies/weaviate:1.19.6
    ports:
      - "8080:8080"
    environment:
      QUERY_DEFAULTS_LIMIT: 20
      AUTHENTICATION_ANONYMOUS_ACCESS_ENABLED: 'true'
      PERSISTENCE_DATA_PATH: '/var/lib/weaviate'
      DEFAULT_VECTORIZER_MODULE: 'text2vec-openai'
      ENABLE_MODULES: 'text2vec-openai'
      OPENAI_APIKEY: ${OPENAI_API_KEY}
      CLUSTER_HOSTNAME: 'node1'
```

### 5.2 Deploy Frontend to Vercel

1. Push your code to a GitHub repository
2. Go to Vercel.com and import your project
3. Configure environment variables in Vercel dashboard
4. Deploy!

### 5.3 Deploy Backend to AWS

Create `backend/aws/Dockerfile.prod`:
```dockerfile
FROM python:3.9-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt gunicorn

# Copy application
COPY . .

# Run gunicorn
CMD ["gunicorn", "-w", "4", "-k", "uvicorn.workers.UvicornWorker", "--bind", "0.0.0.0:8000", "main:app"]
```

Create ECS task definition and deploy to AWS ECS or EKS.

## Phase 6: Monitoring & Maintenance

### 6.1 Set Up Logging

Update `backend/main.py`:
```python
import logging
from fastapi.logger import logger

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s"
)
logger = logging.getLogger(__name__)

# Add logging to your endpoints
@app.post("/api/v1/ai/chat-with-context")
async def chat_with_context(request: ChatRequest):
    logger.info(f"Received chat request with prompt: {request.prompt[:50]}...")
    # ... rest of the code
```

### 6.2 Implement Monitoring

Set up AWS CloudWatch or Datadog for monitoring your backend services.

## Final Checklist

Before going to production:
- [ ] Test all API endpoints
- [ ] Implement rate limiting
- [ ] Set up proper error handling
- [ ] Configure proper CORS settings
- [ ] Implement input validation
- [ ] Set up proper logging
- [ ] Configure monitoring and alerts
- [ ] Implement backup strategy for your vector database
- [ ] Set up proper security headers for frontend
- [ ] Implement CI/CD pipelines
- [ ] Configure proper scaling for your backend services

This guide provides a comprehensive foundation for your full-stack AI application. You can extend it with additional features like user authentication, more complex AI workflows, or specialized vector database operations as needed.
