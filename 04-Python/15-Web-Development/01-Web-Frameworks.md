# Web Frameworks in Python

## Flask vs FastAPI

```python
"""
Flask:
- Mature, well-documented, large ecosystem
- Synchronous by default
- Flexible, minimal opinions
- Great for: traditional web apps, prototypes, small APIs

FastAPI:
- Modern, async-first design
- Automatic OpenAPI documentation
- Type hints for validation
- Great for: APIs, high-performance services, modern Python

When to choose:
- Flask: Full web apps with templates, existing codebases
- FastAPI: REST APIs, microservices, async requirements
"""
```

---

## Flask Basics

### Installation and Hello World

```python
# pip install flask
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello():
    return 'Hello, World!'

@app.route('/greet/<name>')
def greet(name):
    return f'Hello, {name}!'

if __name__ == '__main__':
    app.run(debug=True)  # Never use debug=True in production

# Run: python app.py
# Or: flask run
```

### Routing

```python
from flask import Flask, request

app = Flask(__name__)

# Basic routes
@app.route('/')
def index():
    return 'Home'

# URL parameters
@app.route('/users/<int:user_id>')
def get_user(user_id):
    return f'User {user_id}'

# Multiple parameter types
@app.route('/posts/<int:post_id>/comments/<int:comment_id>')
def get_comment(post_id, comment_id):
    return f'Post {post_id}, Comment {comment_id}'

# Converters: string (default), int, float, path, uuid
@app.route('/files/<path:filepath>')
def get_file(filepath):
    return f'File: {filepath}'  # Matches slashes

# HTTP methods
@app.route('/api/users', methods=['GET', 'POST'])
def users():
    if request.method == 'POST':
        return 'Create user', 201
    return 'List users'

# Method-specific decorators
@app.get('/api/items')
def list_items():
    return 'List items'

@app.post('/api/items')
def create_item():
    return 'Create item', 201

@app.put('/api/items/<int:item_id>')
def update_item(item_id):
    return f'Update item {item_id}'

@app.delete('/api/items/<int:item_id>')
def delete_item(item_id):
    return '', 204
```

### Request Handling

```python
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/api/users', methods=['POST'])
def create_user():
    # JSON body
    data = request.json  # or request.get_json()
    name = data.get('name')
    email = data.get('email')

    # Query parameters
    page = request.args.get('page', 1, type=int)
    limit = request.args.get('limit', 10, type=int)

    # Form data
    username = request.form.get('username')

    # Files
    file = request.files.get('avatar')
    if file:
        file.save(f'uploads/{file.filename}')

    # Headers
    auth = request.headers.get('Authorization')
    content_type = request.content_type

    # Request info
    method = request.method
    path = request.path
    full_url = request.url
    client_ip = request.remote_addr

    return jsonify({
        'name': name,
        'email': email,
        'page': page
    }), 201
```

### Response Handling

```python
from flask import Flask, jsonify, make_response, redirect, url_for

app = Flask(__name__)

# Return string
@app.route('/text')
def text_response():
    return 'Plain text'

# Return JSON
@app.route('/json')
def json_response():
    return jsonify({'message': 'Hello', 'status': 'success'})

# Return tuple (body, status, headers)
@app.route('/custom')
def custom_response():
    return jsonify({'data': 'value'}), 201, {'X-Custom': 'header'}

# Using make_response
@app.route('/response')
def make_resp():
    response = make_response(jsonify({'data': 'value'}))
    response.status_code = 201
    response.headers['X-Custom'] = 'header'
    response.set_cookie('session', 'abc123', httponly=True)
    return response

# Redirects
@app.route('/old-path')
def redirect_example():
    return redirect(url_for('new_endpoint'))

@app.route('/new-path')
def new_endpoint():
    return 'New path'

# Error responses
@app.route('/error')
def error_example():
    from flask import abort
    abort(404)  # Raises NotFound exception
```

### Error Handling

```python
from flask import Flask, jsonify

app = Flask(__name__)

# Custom error handlers
@app.errorhandler(404)
def not_found(error):
    return jsonify({'error': 'Not found'}), 404

@app.errorhandler(500)
def server_error(error):
    return jsonify({'error': 'Internal server error'}), 500

# Handle specific exceptions
class ValidationError(Exception):
    def __init__(self, message):
        self.message = message

@app.errorhandler(ValidationError)
def handle_validation_error(error):
    return jsonify({'error': error.message}), 400

# Usage
@app.route('/validate')
def validate():
    raise ValidationError('Invalid input')
```

### Blueprints (Modular Apps)

```python
# blueprints/users.py
from flask import Blueprint, jsonify

users_bp = Blueprint('users', __name__, url_prefix='/api/users')

@users_bp.route('/')
def list_users():
    return jsonify({'users': []})

@users_bp.route('/<int:user_id>')
def get_user(user_id):
    return jsonify({'id': user_id})

@users_bp.route('/', methods=['POST'])
def create_user():
    return jsonify({'id': 1}), 201


# app.py
from flask import Flask
from blueprints.users import users_bp

app = Flask(__name__)
app.register_blueprint(users_bp)

# Routes available:
# GET /api/users/
# GET /api/users/<id>
# POST /api/users/
```

### Middleware (Before/After Request)

```python
from flask import Flask, request, g
import time

app = Flask(__name__)

@app.before_request
def before_request():
    # Runs before each request
    g.start_time = time.time()

    # Check authentication
    auth = request.headers.get('Authorization')
    if auth:
        g.user = decode_token(auth)  # Your auth logic

@app.after_request
def after_request(response):
    # Runs after each request
    duration = time.time() - g.start_time
    response.headers['X-Response-Time'] = str(duration)
    return response

@app.teardown_request
def teardown_request(exception):
    # Runs after request, even if exception occurred
    # Good for cleanup (closing DB connections, etc.)
    pass
```

---

## FastAPI Basics

### Installation and Hello World

```python
# pip install fastapi uvicorn
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def read_root():
    return {"Hello": "World"}

@app.get("/items/{item_id}")
def read_item(item_id: int):
    return {"item_id": item_id}

# Run: uvicorn main:app --reload

# Automatic docs at:
# http://localhost:8000/docs (Swagger UI)
# http://localhost:8000/redoc (ReDoc)
```

### Path Parameters

```python
from fastapi import FastAPI, Path
from enum import Enum

app = FastAPI()

# Basic path parameter
@app.get("/users/{user_id}")
def get_user(user_id: int):  # Type automatically validated
    return {"user_id": user_id}

# With validation
@app.get("/items/{item_id}")
def get_item(
    item_id: int = Path(..., gt=0, le=1000, description="Item ID")
):
    return {"item_id": item_id}

# Enum for fixed values
class Status(str, Enum):
    active = "active"
    pending = "pending"
    deleted = "deleted"

@app.get("/status/{status}")
def get_by_status(status: Status):
    return {"status": status}
```

### Query Parameters

```python
from fastapi import FastAPI, Query
from typing import Optional, List

app = FastAPI()

# Basic query params
@app.get("/items")
def list_items(
    skip: int = 0,
    limit: int = 10
):
    # GET /items?skip=0&limit=10
    return {"skip": skip, "limit": limit}

# Optional parameters
@app.get("/search")
def search(q: Optional[str] = None):
    if q:
        return {"results": f"Searching for {q}"}
    return {"results": "No query"}

# With validation
@app.get("/items")
def list_items(
    skip: int = Query(0, ge=0),
    limit: int = Query(10, ge=1, le=100),
    search: Optional[str] = Query(None, min_length=3, max_length=50)
):
    return {"skip": skip, "limit": limit, "search": search}

# Multiple values
@app.get("/items")
def list_items(tags: List[str] = Query([])):
    # GET /items?tags=foo&tags=bar
    return {"tags": tags}
```

### Request Body with Pydantic

```python
from fastapi import FastAPI
from pydantic import BaseModel, Field, EmailStr
from typing import Optional
from datetime import datetime

app = FastAPI()

# Request/response model
class UserCreate(BaseModel):
    name: str = Field(..., min_length=1, max_length=100)
    email: EmailStr
    age: Optional[int] = Field(None, ge=0, le=150)

class UserResponse(BaseModel):
    id: int
    name: str
    email: str
    created_at: datetime

    class Config:
        from_attributes = True  # Enable ORM mode

# Create user
@app.post("/users", response_model=UserResponse, status_code=201)
def create_user(user: UserCreate):
    # user is validated Pydantic model
    db_user = save_to_db(user)
    return db_user

# Multiple body parameters
class Item(BaseModel):
    name: str
    price: float

class Order(BaseModel):
    item: Item
    quantity: int

@app.post("/orders")
def create_order(order: Order):
    return order
```

### Response Models

```python
from fastapi import FastAPI
from pydantic import BaseModel
from typing import List

app = FastAPI()

class User(BaseModel):
    id: int
    name: str
    email: str
    password: str  # Should not be in response

class UserPublic(BaseModel):
    id: int
    name: str
    email: str

# Filter response fields
@app.get("/users/{user_id}", response_model=UserPublic)
def get_user(user_id: int):
    user = get_user_from_db(user_id)
    return user  # password field excluded

# List response
@app.get("/users", response_model=List[UserPublic])
def list_users():
    return get_all_users()

# Exclude unset fields
@app.get("/users/{user_id}", response_model=UserPublic, response_model_exclude_unset=True)
def get_user(user_id: int):
    return {"id": user_id, "name": "Alice"}  # email excluded if not set
```

### Dependency Injection

```python
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy.orm import Session
from typing import Generator

app = FastAPI()

# Database dependency
def get_db() -> Generator[Session, None, None]:
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# Auth dependency
def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: Session = Depends(get_db)
) -> User:
    user = verify_token(token, db)
    if not user:
        raise HTTPException(status_code=401, detail="Invalid token")
    return user

# Use dependencies
@app.get("/users/me")
def get_me(current_user: User = Depends(get_current_user)):
    return current_user

@app.get("/items")
def list_items(
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    return db.query(Item).filter(Item.owner_id == current_user.id).all()

# Class-based dependency
class Pagination:
    def __init__(self, skip: int = 0, limit: int = 10):
        self.skip = skip
        self.limit = limit

@app.get("/items")
def list_items(pagination: Pagination = Depends()):
    return get_items(pagination.skip, pagination.limit)
```

### Error Handling

```python
from fastapi import FastAPI, HTTPException
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError

app = FastAPI()

# Raise HTTP exceptions
@app.get("/items/{item_id}")
def get_item(item_id: int):
    item = get_item_from_db(item_id)
    if not item:
        raise HTTPException(
            status_code=404,
            detail="Item not found",
            headers={"X-Error": "Item missing"}
        )
    return item

# Custom exception class
class ItemNotFoundError(Exception):
    def __init__(self, item_id: int):
        self.item_id = item_id

@app.exception_handler(ItemNotFoundError)
async def item_not_found_handler(request, exc):
    return JSONResponse(
        status_code=404,
        content={"message": f"Item {exc.item_id} not found"}
    )

# Handle validation errors
@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request, exc):
    return JSONResponse(
        status_code=422,
        content={
            "message": "Validation error",
            "details": exc.errors()
        }
    )
```

### Middleware

```python
from fastapi import FastAPI, Request
from fastapi.middleware.cors import CORSMiddleware
import time

app = FastAPI()

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Custom middleware
@app.middleware("http")
async def add_process_time(request: Request, call_next):
    start_time = time.time()
    response = await call_next(request)
    process_time = time.time() - start_time
    response.headers["X-Process-Time"] = str(process_time)
    return response

# Request logging middleware
@app.middleware("http")
async def log_requests(request: Request, call_next):
    print(f"{request.method} {request.url}")
    response = await call_next(request)
    print(f"Status: {response.status_code}")
    return response
```

### Async Support

```python
from fastapi import FastAPI
import asyncio
import httpx

app = FastAPI()

# Async endpoint
@app.get("/async-items")
async def get_items():
    await asyncio.sleep(1)  # Simulate async operation
    return {"items": []}

# Async with external API
@app.get("/external-data")
async def get_external_data():
    async with httpx.AsyncClient() as client:
        response = await client.get("https://api.example.com/data")
        return response.json()

# Parallel async calls
@app.get("/dashboard")
async def get_dashboard():
    async with httpx.AsyncClient() as client:
        users, posts, comments = await asyncio.gather(
            client.get("https://api.example.com/users"),
            client.get("https://api.example.com/posts"),
            client.get("https://api.example.com/comments")
        )
        return {
            "users": users.json(),
            "posts": posts.json(),
            "comments": comments.json()
        }
```

### APIRouter (Modular Apps)

```python
# routers/users.py
from fastapi import APIRouter, Depends
from sqlalchemy.orm import Session

router = APIRouter(
    prefix="/users",
    tags=["users"],
    responses={404: {"description": "Not found"}}
)

@router.get("/")
def list_users(db: Session = Depends(get_db)):
    return db.query(User).all()

@router.get("/{user_id}")
def get_user(user_id: int, db: Session = Depends(get_db)):
    return db.query(User).filter(User.id == user_id).first()

@router.post("/", status_code=201)
def create_user(user: UserCreate, db: Session = Depends(get_db)):
    db_user = User(**user.dict())
    db.add(db_user)
    db.commit()
    return db_user


# main.py
from fastapi import FastAPI
from routers import users, items

app = FastAPI()

app.include_router(users.router, prefix="/api")
app.include_router(items.router, prefix="/api")

# Routes available:
# GET /api/users/
# GET /api/users/{id}
# POST /api/users/
```

---

## Authentication Patterns

### JWT Authentication (FastAPI)

```python
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from jose import JWTError, jwt
from passlib.context import CryptContext
from datetime import datetime, timedelta
from pydantic import BaseModel

app = FastAPI()

# Configuration
SECRET_KEY = "your-secret-key"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

pwd_context = CryptContext(schemes=["bcrypt"])
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

class Token(BaseModel):
    access_token: str
    token_type: str

class User(BaseModel):
    username: str
    email: str

def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)

def create_access_token(data: dict) -> str:
    to_encode = data.copy()
    expire = datetime.utcnow() + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

async def get_current_user(token: str = Depends(oauth2_scheme)) -> User:
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username: str = payload.get("sub")
        if username is None:
            raise credentials_exception
    except JWTError:
        raise credentials_exception

    user = get_user_from_db(username)
    if user is None:
        raise credentials_exception
    return user

@app.post("/token", response_model=Token)
async def login(form_data: OAuth2PasswordRequestForm = Depends()):
    user = authenticate_user(form_data.username, form_data.password)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect username or password"
        )
    access_token = create_access_token(data={"sub": user.username})
    return {"access_token": access_token, "token_type": "bearer"}

@app.get("/users/me", response_model=User)
async def read_users_me(current_user: User = Depends(get_current_user)):
    return current_user
```

### Session Authentication (Flask)

```python
from flask import Flask, session, redirect, url_for, request
from functools import wraps

app = Flask(__name__)
app.secret_key = 'your-secret-key'

def login_required(f):
    @wraps(f)
    def decorated_function(*args, **kwargs):
        if 'user_id' not in session:
            return redirect(url_for('login'))
        return f(*args, **kwargs)
    return decorated_function

@app.route('/login', methods=['POST'])
def login():
    username = request.form['username']
    password = request.form['password']

    user = authenticate_user(username, password)
    if user:
        session['user_id'] = user.id
        session['username'] = user.username
        return redirect(url_for('dashboard'))

    return 'Invalid credentials', 401

@app.route('/logout')
def logout():
    session.clear()
    return redirect(url_for('login'))

@app.route('/dashboard')
@login_required
def dashboard():
    return f'Hello, {session["username"]}'
```

---

## Testing Web Applications

### Testing Flask

```python
import pytest
from flask import Flask

def create_app():
    app = Flask(__name__)

    @app.route('/api/users')
    def list_users():
        return {'users': []}

    return app

@pytest.fixture
def client():
    app = create_app()
    app.config['TESTING'] = True
    with app.test_client() as client:
        yield client

def test_list_users(client):
    response = client.get('/api/users')
    assert response.status_code == 200
    assert response.json == {'users': []}

def test_create_user(client):
    response = client.post(
        '/api/users',
        json={'name': 'Alice', 'email': 'alice@example.com'}
    )
    assert response.status_code == 201
```

### Testing FastAPI

```python
import pytest
from fastapi.testclient import TestClient
from main import app

@pytest.fixture
def client():
    return TestClient(app)

def test_read_root(client):
    response = client.get("/")
    assert response.status_code == 200
    assert response.json() == {"Hello": "World"}

def test_create_item(client):
    response = client.post(
        "/items",
        json={"name": "Test Item", "price": 9.99}
    )
    assert response.status_code == 201
    assert response.json()["name"] == "Test Item"

def test_invalid_item(client):
    response = client.post(
        "/items",
        json={"name": ""}  # Invalid: empty name
    )
    assert response.status_code == 422

# Async testing
import pytest
from httpx import AsyncClient

@pytest.mark.asyncio
async def test_async_endpoint():
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/async-items")
        assert response.status_code == 200
```

---

## Best Practices

```python
"""
Web Framework Best Practices:

1. Project Structure
   myapp/
   ├── app/
   │   ├── __init__.py
   │   ├── main.py
   │   ├── config.py
   │   ├── models/
   │   ├── routers/
   │   ├── services/
   │   └── schemas/
   ├── tests/
   └── requirements.txt

2. Use Environment Variables
   - Never hardcode secrets
   - Use pydantic Settings for FastAPI
   - Use python-dotenv for Flask

3. Validate All Input
   - Use Pydantic models (FastAPI)
   - Use Flask-Marshmallow or WTForms

4. Handle Errors Gracefully
   - Custom exception handlers
   - Consistent error response format
   - Don't expose internal errors

5. Use Dependency Injection
   - Database sessions
   - Authentication
   - Configuration

6. Implement Proper Logging
   - Request/response logging
   - Error logging
   - Use structured logging (JSON)

7. Security
   - CORS configuration
   - Rate limiting
   - Input sanitization
   - HTTPS in production
"""
```

---

## Interview Questions

```python
# Q1: Flask vs FastAPI - when to use which?
"""
Flask:
- Full web apps with templates
- Existing codebases, mature ecosystem
- Simpler learning curve
- Synchronous workloads

FastAPI:
- REST APIs, microservices
- Async/high-performance requirements
- Auto-generated documentation needed
- Type-safe development

FastAPI is generally preferred for new APIs due to
performance and modern features.
"""

# Q2: How do you handle authentication in web APIs?
"""
Common approaches:

1. JWT (JSON Web Tokens):
   - Stateless, scalable
   - Token in Authorization header
   - Good for APIs/SPAs

2. Session-based:
   - Server stores session
   - Cookie-based
   - Good for traditional web apps

3. OAuth2:
   - Third-party auth (Google, GitHub)
   - Token-based
   - Good for integrations

FastAPI provides OAuth2PasswordBearer for JWT.
Flask-Login for session-based auth.
"""

# Q3: How do you structure a large Flask/FastAPI app?
"""
Use modular patterns:

Flask: Blueprints
- Group related routes
- Separate by feature/domain
- Register in app factory

FastAPI: APIRouter
- Similar to blueprints
- Include in main app
- Organize by resource

Both: Application Factory Pattern
- Create app in function
- Configure based on environment
- Easier testing
"""

# Q4: How do you handle database connections in web apps?
"""
Use connection pooling and proper lifecycle:

FastAPI:
- Create engine at startup
- Use Depends() for session per request
- Close session after request

Flask:
- Use Flask-SQLAlchemy
- Session scoped to request
- Teardown cleanup

Key: One session per request, proper cleanup.
"""

# Q5: How do you test web applications?
"""
Flask:
- app.test_client() for HTTP testing
- pytest fixtures for app setup
- Test database with transactions

FastAPI:
- TestClient from fastapi.testclient
- Async testing with httpx.AsyncClient
- Override dependencies for mocking

Both:
- Test happy path and error cases
- Use test database (rollback transactions)
- Mock external services
"""
```
