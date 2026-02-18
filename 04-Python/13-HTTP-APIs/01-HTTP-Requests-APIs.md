# HTTP, Requests, and API Development

## The requests Library

### Installation and Basic Usage

```python
# pip install requests
import requests

# Basic GET request
response = requests.get('https://api.example.com/users')

# Response attributes
response.status_code      # 200
response.text             # Response body as string
response.content          # Response body as bytes
response.json()           # Parse JSON response
response.headers          # Response headers (dict-like)
response.url              # Final URL (after redirects)
response.elapsed          # Time taken for request
response.encoding         # Detected encoding
response.ok               # True if status < 400

# Check for errors
response.raise_for_status()  # Raises HTTPError if 4xx/5xx
```

### HTTP Methods

```python
import requests

# GET - Retrieve data
response = requests.get('https://api.example.com/users')

# POST - Create resource
response = requests.post(
    'https://api.example.com/users',
    json={'name': 'Alice', 'email': 'alice@example.com'}
)

# PUT - Update/replace resource
response = requests.put(
    'https://api.example.com/users/123',
    json={'name': 'Alice Updated', 'email': 'alice@example.com'}
)

# PATCH - Partial update
response = requests.patch(
    'https://api.example.com/users/123',
    json={'name': 'New Name'}
)

# DELETE - Remove resource
response = requests.delete('https://api.example.com/users/123')

# HEAD - Get headers only (no body)
response = requests.head('https://api.example.com/users')

# OPTIONS - Get allowed methods
response = requests.options('https://api.example.com/users')
```

### Query Parameters

```python
# Method 1: params argument (preferred)
response = requests.get(
    'https://api.example.com/search',
    params={
        'q': 'python',
        'page': 1,
        'limit': 10
    }
)
# URL becomes: https://api.example.com/search?q=python&page=1&limit=10

# Method 2: Multiple values for same key
response = requests.get(
    'https://api.example.com/items',
    params=[('tag', 'python'), ('tag', 'api')]
)
# URL: https://api.example.com/items?tag=python&tag=api

# Special characters are automatically encoded
response = requests.get(
    'https://api.example.com/search',
    params={'q': 'hello world'}  # Becomes q=hello%20world
)
```

### Request Headers

```python
# Custom headers
headers = {
    'Authorization': 'Bearer eyJ0eXAiOiJKV1QiLCJhbGci...',
    'Content-Type': 'application/json',
    'Accept': 'application/json',
    'User-Agent': 'MyApp/1.0'
}

response = requests.get(
    'https://api.example.com/protected',
    headers=headers
)

# Access response headers
print(response.headers['Content-Type'])
print(response.headers.get('X-RateLimit-Remaining'))
```

### Request Body

```python
# JSON data (most common for APIs)
response = requests.post(
    'https://api.example.com/users',
    json={'name': 'Alice', 'age': 30}  # Auto-sets Content-Type
)

# Form data (traditional HTML forms)
response = requests.post(
    'https://api.example.com/login',
    data={'username': 'alice', 'password': 'secret'}
)

# Raw data
response = requests.post(
    'https://api.example.com/data',
    data='raw string data',
    headers={'Content-Type': 'text/plain'}
)

# File upload
with open('document.pdf', 'rb') as f:
    response = requests.post(
        'https://api.example.com/upload',
        files={'file': f}
    )

# Multiple files with metadata
files = {
    'file1': ('report.pdf', open('report.pdf', 'rb'), 'application/pdf'),
    'file2': ('data.csv', open('data.csv', 'rb'), 'text/csv'),
}
response = requests.post('https://api.example.com/upload', files=files)

# File + form data together
response = requests.post(
    'https://api.example.com/upload',
    data={'description': 'My file'},
    files={'file': open('image.png', 'rb')}
)
```

---

## Sessions and Connection Pooling

### Why Use Sessions

```python
import requests

# Without session: New connection each time
requests.get('https://api.example.com/users')   # New connection
requests.get('https://api.example.com/posts')   # New connection
requests.get('https://api.example.com/comments')  # New connection

# With session: Connection reuse (much faster)
session = requests.Session()
session.get('https://api.example.com/users')    # New connection
session.get('https://api.example.com/posts')    # Reuses connection
session.get('https://api.example.com/comments') # Reuses connection

# Benefits:
# - Connection pooling (faster)
# - Cookie persistence
# - Default headers/params
# - Auth persistence
```

### Session Configuration

```python
import requests

session = requests.Session()

# Set default headers for all requests
session.headers.update({
    'Authorization': 'Bearer my-token',
    'User-Agent': 'MyApp/1.0',
    'Accept': 'application/json'
})

# Set default parameters
session.params = {'api_key': 'abc123'}

# Now all requests include these
response = session.get('https://api.example.com/data')
# Includes Authorization header and api_key param automatically

# Override per-request
response = session.get(
    'https://api.example.com/other',
    headers={'Authorization': 'Bearer different-token'}
)
```

### Session as Context Manager

```python
import requests

# Ensures session is closed properly
with requests.Session() as session:
    session.headers['Authorization'] = 'Bearer token'

    response = session.get('https://api.example.com/users')
    users = response.json()

    for user in users:
        details = session.get(f'https://api.example.com/users/{user["id"]}')
        # Connection reused for each request
```

### Cookies

```python
import requests

session = requests.Session()

# Cookies are automatically stored and sent
response = session.get('https://example.com/login')
# Server sets cookies, session stores them

response = session.get('https://example.com/dashboard')
# Cookies automatically included

# Manually set cookies
session.cookies.set('session_id', 'abc123', domain='example.com')

# View cookies
print(session.cookies.get_dict())

# Clear cookies
session.cookies.clear()

# Cookies from response
response = requests.get('https://example.com')
print(response.cookies['session_id'])
```

---

## Authentication

### Basic Authentication

```python
import requests
from requests.auth import HTTPBasicAuth

# Method 1: Using auth parameter
response = requests.get(
    'https://api.example.com/private',
    auth=HTTPBasicAuth('username', 'password')
)

# Method 2: Shorthand tuple
response = requests.get(
    'https://api.example.com/private',
    auth=('username', 'password')
)
```

### Bearer Token (JWT)

```python
import requests

# Most common for modern APIs
headers = {
    'Authorization': 'Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...'
}

response = requests.get(
    'https://api.example.com/protected',
    headers=headers
)

# With session (recommended)
session = requests.Session()
session.headers['Authorization'] = 'Bearer my-jwt-token'

# All subsequent requests include the token
response = session.get('https://api.example.com/protected')
```

### API Key Authentication

```python
import requests

# In header
response = requests.get(
    'https://api.example.com/data',
    headers={'X-API-Key': 'your-api-key'}
)

# In query parameter
response = requests.get(
    'https://api.example.com/data',
    params={'api_key': 'your-api-key'}
)
```

### OAuth 2.0 Example

```python
import requests

# Step 1: Get authorization code (usually via browser redirect)
# User authorizes your app, you receive a code

# Step 2: Exchange code for access token
token_response = requests.post(
    'https://oauth.example.com/token',
    data={
        'grant_type': 'authorization_code',
        'code': 'authorization_code_from_callback',
        'redirect_uri': 'https://yourapp.com/callback',
        'client_id': 'your-client-id',
        'client_secret': 'your-client-secret'
    }
)

tokens = token_response.json()
access_token = tokens['access_token']
refresh_token = tokens['refresh_token']

# Step 3: Use access token
response = requests.get(
    'https://api.example.com/user',
    headers={'Authorization': f'Bearer {access_token}'}
)

# Step 4: Refresh token when expired
refresh_response = requests.post(
    'https://oauth.example.com/token',
    data={
        'grant_type': 'refresh_token',
        'refresh_token': refresh_token,
        'client_id': 'your-client-id',
        'client_secret': 'your-client-secret'
    }
)
new_tokens = refresh_response.json()
```

### Custom Authentication Class

```python
from requests.auth import AuthBase
import time
import hmac
import hashlib

class HMACAuth(AuthBase):
    """Custom HMAC authentication."""

    def __init__(self, api_key: str, api_secret: str):
        self.api_key = api_key
        self.api_secret = api_secret

    def __call__(self, request):
        timestamp = str(int(time.time()))
        message = f'{timestamp}{request.method}{request.path_url}'

        signature = hmac.new(
            self.api_secret.encode(),
            message.encode(),
            hashlib.sha256
        ).hexdigest()

        request.headers['X-API-Key'] = self.api_key
        request.headers['X-Timestamp'] = timestamp
        request.headers['X-Signature'] = signature

        return request

# Usage
auth = HMACAuth('my-api-key', 'my-secret')
response = requests.get('https://api.example.com/data', auth=auth)
```

---

## Timeouts and Retries

### Timeouts

```python
import requests
from requests.exceptions import Timeout

# ALWAYS set timeouts in production!
# Without timeout, request can hang indefinitely

# Single timeout (both connect and read)
response = requests.get(
    'https://api.example.com/data',
    timeout=10  # 10 seconds total
)

# Separate connect and read timeouts
response = requests.get(
    'https://api.example.com/data',
    timeout=(3.05, 27)  # (connect_timeout, read_timeout)
)

# Handle timeout
try:
    response = requests.get('https://api.example.com/slow', timeout=5)
except Timeout:
    print("Request timed out")
```

### Retry with urllib3

```python
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

# Configure retry strategy
retry_strategy = Retry(
    total=3,                    # Total retries
    backoff_factor=1,           # Wait 1, 2, 4 seconds between retries
    status_forcelist=[429, 500, 502, 503, 504],  # Retry on these status codes
    allowed_methods=["HEAD", "GET", "OPTIONS", "POST"],  # Methods to retry
    raise_on_status=False       # Don't raise exception on retry exhaustion
)

# Create adapter with retry
adapter = HTTPAdapter(max_retries=retry_strategy)

# Mount to session
session = requests.Session()
session.mount("https://", adapter)
session.mount("http://", adapter)

# Now requests automatically retry
response = session.get('https://api.example.com/data', timeout=10)
```

### Custom Retry Logic

```python
import requests
import time
from typing import Optional

def request_with_retry(
    method: str,
    url: str,
    max_retries: int = 3,
    backoff_factor: float = 1.0,
    **kwargs
) -> Optional[requests.Response]:
    """Make request with exponential backoff retry."""

    for attempt in range(max_retries):
        try:
            response = requests.request(method, url, timeout=10, **kwargs)
            response.raise_for_status()
            return response

        except requests.exceptions.RequestException as e:
            if attempt == max_retries - 1:
                raise

            wait_time = backoff_factor * (2 ** attempt)
            print(f"Attempt {attempt + 1} failed: {e}. Retrying in {wait_time}s")
            time.sleep(wait_time)

    return None

# Usage
response = request_with_retry('GET', 'https://api.example.com/data')
```

---

## Error Handling

### Exception Hierarchy

```python
import requests
from requests.exceptions import (
    RequestException,      # Base exception
    ConnectionError,       # Network problem
    HTTPError,            # HTTP error response (4xx/5xx)
    URLRequired,          # Invalid URL
    TooManyRedirects,     # Too many redirects
    ConnectTimeout,       # Connection timeout
    ReadTimeout,          # Read timeout
    Timeout,              # Either timeout
    JSONDecodeError       # Invalid JSON response
)

# Comprehensive error handling
def make_request(url: str) -> dict:
    try:
        response = requests.get(url, timeout=10)
        response.raise_for_status()
        return response.json()

    except requests.exceptions.ConnectionError:
        print("Failed to connect to server")
        raise

    except requests.exceptions.Timeout:
        print("Request timed out")
        raise

    except requests.exceptions.HTTPError as e:
        status = e.response.status_code
        if status == 401:
            print("Authentication required")
        elif status == 403:
            print("Access forbidden")
        elif status == 404:
            print("Resource not found")
        elif status >= 500:
            print("Server error")
        raise

    except requests.exceptions.JSONDecodeError:
        print("Invalid JSON response")
        raise

    except requests.exceptions.RequestException as e:
        print(f"Request failed: {e}")
        raise
```

### Status Code Handling

```python
import requests

response = requests.get('https://api.example.com/users')

# Check specific codes
if response.status_code == 200:
    users = response.json()
elif response.status_code == 404:
    print("No users found")
elif response.status_code == 401:
    print("Please authenticate")
elif response.status_code >= 500:
    print("Server error, try again later")

# Check categories
if response.ok:  # 200-399
    print("Success")

# Using response.raise_for_status()
try:
    response.raise_for_status()
    data = response.json()
except requests.exceptions.HTTPError:
    if response.status_code == 404:
        data = None  # Handle gracefully
    else:
        raise
```

---

## Building API Clients

### Basic API Client Pattern

```python
import requests
from typing import Optional, Dict, Any, List

class APIClient:
    """Base API client with common functionality."""

    def __init__(
        self,
        base_url: str,
        api_key: str,
        timeout: int = 30
    ):
        self.base_url = base_url.rstrip('/')
        self.timeout = timeout

        self.session = requests.Session()
        self.session.headers.update({
            'Authorization': f'Bearer {api_key}',
            'Content-Type': 'application/json',
            'Accept': 'application/json'
        })

    def _request(
        self,
        method: str,
        endpoint: str,
        **kwargs
    ) -> requests.Response:
        """Make HTTP request with error handling."""
        url = f'{self.base_url}/{endpoint.lstrip("/")}'

        try:
            response = self.session.request(
                method,
                url,
                timeout=self.timeout,
                **kwargs
            )
            response.raise_for_status()
            return response

        except requests.exceptions.HTTPError as e:
            # Log error, maybe transform to custom exception
            raise APIError(f"HTTP {e.response.status_code}: {e.response.text}")
        except requests.exceptions.RequestException as e:
            raise APIError(f"Request failed: {e}")

    def get(self, endpoint: str, params: Dict = None) -> Dict:
        response = self._request('GET', endpoint, params=params)
        return response.json()

    def post(self, endpoint: str, data: Dict) -> Dict:
        response = self._request('POST', endpoint, json=data)
        return response.json()

    def put(self, endpoint: str, data: Dict) -> Dict:
        response = self._request('PUT', endpoint, json=data)
        return response.json()

    def delete(self, endpoint: str) -> None:
        self._request('DELETE', endpoint)

    def close(self):
        self.session.close()

    def __enter__(self):
        return self

    def __exit__(self, *args):
        self.close()


class APIError(Exception):
    """Custom API exception."""
    pass
```

### Resource-Specific Client

```python
from dataclasses import dataclass
from typing import List, Optional

@dataclass
class User:
    id: int
    name: str
    email: str

    @classmethod
    def from_dict(cls, data: dict) -> 'User':
        return cls(
            id=data['id'],
            name=data['name'],
            email=data['email']
        )


class UserClient(APIClient):
    """Client for User API endpoints."""

    def list_users(self, page: int = 1, limit: int = 20) -> List[User]:
        """Get list of users."""
        data = self.get('/users', params={'page': page, 'limit': limit})
        return [User.from_dict(u) for u in data['users']]

    def get_user(self, user_id: int) -> User:
        """Get single user by ID."""
        data = self.get(f'/users/{user_id}')
        return User.from_dict(data)

    def create_user(self, name: str, email: str) -> User:
        """Create new user."""
        data = self.post('/users', {'name': name, 'email': email})
        return User.from_dict(data)

    def update_user(self, user_id: int, **updates) -> User:
        """Update user fields."""
        data = self.put(f'/users/{user_id}', updates)
        return User.from_dict(data)

    def delete_user(self, user_id: int) -> None:
        """Delete user."""
        self.delete(f'/users/{user_id}')


# Usage
with UserClient('https://api.example.com', 'my-api-key') as client:
    # List users
    users = client.list_users(page=1, limit=10)

    # Create user
    new_user = client.create_user('Alice', 'alice@example.com')

    # Update user
    updated = client.update_user(new_user.id, name='Alice Smith')

    # Delete user
    client.delete_user(new_user.id)
```

### Async API Client (aiohttp)

```python
import aiohttp
import asyncio
from typing import Dict, List

class AsyncAPIClient:
    """Async API client using aiohttp."""

    def __init__(self, base_url: str, api_key: str):
        self.base_url = base_url.rstrip('/')
        self.headers = {
            'Authorization': f'Bearer {api_key}',
            'Content-Type': 'application/json'
        }
        self._session: aiohttp.ClientSession = None

    async def __aenter__(self):
        self._session = aiohttp.ClientSession(headers=self.headers)
        return self

    async def __aexit__(self, *args):
        await self._session.close()

    async def get(self, endpoint: str, params: Dict = None) -> Dict:
        url = f'{self.base_url}/{endpoint.lstrip("/")}'
        async with self._session.get(url, params=params) as response:
            response.raise_for_status()
            return await response.json()

    async def post(self, endpoint: str, data: Dict) -> Dict:
        url = f'{self.base_url}/{endpoint.lstrip("/")}'
        async with self._session.post(url, json=data) as response:
            response.raise_for_status()
            return await response.json()


# Usage
async def main():
    async with AsyncAPIClient('https://api.example.com', 'key') as client:
        # Concurrent requests
        users, posts = await asyncio.gather(
            client.get('/users'),
            client.get('/posts')
        )
        print(f"Got {len(users)} users and {len(posts)} posts")

asyncio.run(main())
```

---

## Rate Limiting

### Client-Side Rate Limiting

```python
import time
import requests
from threading import Lock

class RateLimitedClient:
    """Client with rate limiting."""

    def __init__(
        self,
        base_url: str,
        requests_per_second: float = 10.0
    ):
        self.base_url = base_url
        self.min_interval = 1.0 / requests_per_second
        self.last_request = 0.0
        self._lock = Lock()
        self.session = requests.Session()

    def _wait_for_rate_limit(self):
        """Wait if necessary to respect rate limit."""
        with self._lock:
            elapsed = time.time() - self.last_request
            if elapsed < self.min_interval:
                time.sleep(self.min_interval - elapsed)
            self.last_request = time.time()

    def get(self, endpoint: str, **kwargs) -> requests.Response:
        self._wait_for_rate_limit()
        return self.session.get(f'{self.base_url}{endpoint}', **kwargs)


# Token bucket for burst handling
class TokenBucket:
    """Token bucket rate limiter."""

    def __init__(self, rate: float, capacity: int):
        self.rate = rate          # Tokens per second
        self.capacity = capacity  # Max tokens
        self.tokens = capacity
        self.last_update = time.time()
        self._lock = Lock()

    def acquire(self, tokens: int = 1) -> bool:
        """Try to acquire tokens. Returns True if successful."""
        with self._lock:
            now = time.time()
            # Add tokens based on time passed
            elapsed = now - self.last_update
            self.tokens = min(self.capacity, self.tokens + elapsed * self.rate)
            self.last_update = now

            if self.tokens >= tokens:
                self.tokens -= tokens
                return True
            return False

    def wait_for_token(self, tokens: int = 1):
        """Wait until token is available."""
        while not self.acquire(tokens):
            time.sleep(0.1)
```

### Handling Server Rate Limits

```python
import requests
import time

def request_with_rate_limit_handling(
    url: str,
    max_retries: int = 3
) -> requests.Response:
    """Handle 429 Too Many Requests responses."""

    for attempt in range(max_retries):
        response = requests.get(url)

        if response.status_code == 429:
            # Check Retry-After header
            retry_after = response.headers.get('Retry-After')

            if retry_after:
                # Could be seconds or HTTP date
                try:
                    wait_time = int(retry_after)
                except ValueError:
                    # Parse HTTP date format
                    wait_time = 60  # Default
            else:
                # Exponential backoff
                wait_time = 2 ** attempt

            print(f"Rate limited. Waiting {wait_time} seconds...")
            time.sleep(wait_time)
            continue

        return response

    raise Exception("Max retries exceeded due to rate limiting")
```

---

## Testing HTTP Code

### Using responses Library

```python
import responses
import requests

# Mock GET request
@responses.activate
def test_get_users():
    responses.add(
        responses.GET,
        'https://api.example.com/users',
        json={'users': [{'id': 1, 'name': 'Alice'}]},
        status=200
    )

    response = requests.get('https://api.example.com/users')
    assert response.status_code == 200
    assert len(response.json()['users']) == 1

# Mock POST request
@responses.activate
def test_create_user():
    responses.add(
        responses.POST,
        'https://api.example.com/users',
        json={'id': 1, 'name': 'Alice'},
        status=201
    )

    response = requests.post(
        'https://api.example.com/users',
        json={'name': 'Alice'}
    )
    assert response.status_code == 201

# Mock error response
@responses.activate
def test_not_found():
    responses.add(
        responses.GET,
        'https://api.example.com/users/999',
        json={'error': 'Not found'},
        status=404
    )

    response = requests.get('https://api.example.com/users/999')
    assert response.status_code == 404
```

### Using httpretty

```python
import httpretty
import requests

@httpretty.activate
def test_api_call():
    httpretty.register_uri(
        httpretty.GET,
        'https://api.example.com/data',
        body='{"result": "success"}',
        content_type='application/json'
    )

    response = requests.get('https://api.example.com/data')
    assert response.json()['result'] == 'success'
```

### Using requests-mock with pytest

```python
import pytest
import requests

def test_api(requests_mock):
    requests_mock.get(
        'https://api.example.com/users',
        json={'users': []}
    )

    response = requests.get('https://api.example.com/users')
    assert response.json() == {'users': []}
```

---

## Best Practices

```python
"""
HTTP Client Best Practices:

1. Always Set Timeouts
   - Never make requests without timeout
   - Use separate connect/read timeouts
   - requests.get(url, timeout=(3.05, 27))

2. Use Sessions for Multiple Requests
   - Connection pooling improves performance
   - Cookies and headers persist
   - Use context manager for cleanup

3. Handle Errors Gracefully
   - Check status codes or use raise_for_status()
   - Handle specific exceptions
   - Implement retry logic for transient failures

4. Implement Retries with Backoff
   - Use exponential backoff
   - Only retry idempotent operations
   - Respect Retry-After headers

5. Respect Rate Limits
   - Implement client-side rate limiting
   - Handle 429 responses gracefully
   - Use token bucket for burst handling

6. Secure Sensitive Data
   - Never log full requests with auth headers
   - Use environment variables for API keys
   - Validate SSL certificates (verify=True)

7. Use Proper Content Types
   - json= for JSON data (auto-sets Content-Type)
   - data= for form data
   - files= for file uploads

8. Log Appropriately
   - Log request method and URL
   - Log response status and time
   - Redact sensitive headers in logs
"""
```

---

## Interview Questions

```python
# Q1: What's the difference between json= and data= in requests?
"""
json=:
- Serializes dict to JSON string
- Sets Content-Type: application/json
- Use for API requests

data=:
- Sends form-encoded data
- Sets Content-Type: application/x-www-form-urlencoded
- Use for HTML forms

Example:
requests.post(url, json={'key': 'value'})  # JSON body
requests.post(url, data={'key': 'value'})  # Form data
"""

# Q2: How do you handle authentication in requests?
"""
Multiple methods:
1. Basic Auth: auth=('user', 'pass')
2. Bearer Token: headers={'Authorization': 'Bearer token'}
3. API Key: headers={'X-API-Key': 'key'} or params={'api_key': 'key'}
4. Custom: Create class extending requests.auth.AuthBase

Best practice: Use sessions for persistent auth headers.
"""

# Q3: Why use sessions instead of individual requests?
"""
Benefits of sessions:
1. Connection pooling - reuses TCP connections
2. Cookie persistence - automatic cookie handling
3. Default configuration - shared headers/params
4. Performance - avoids connection overhead

Always use sessions when making multiple requests
to the same host.
"""

# Q4: How do you implement retry logic?
"""
Use urllib3 Retry with HTTPAdapter:

from urllib3.util.retry import Retry
from requests.adapters import HTTPAdapter

retry = Retry(total=3, backoff_factor=1, status_forcelist=[500, 502, 503])
adapter = HTTPAdapter(max_retries=retry)
session.mount('https://', adapter)

Key considerations:
- Only retry idempotent operations
- Use exponential backoff
- Set maximum retry limit
"""

# Q5: How do you test code that makes HTTP requests?
"""
Use mocking libraries:
1. responses - decorator-based, simple
2. httpretty - protocol-level mocking
3. requests-mock - pytest integration

Example with responses:
@responses.activate
def test_api():
    responses.add(responses.GET, 'url', json={'data': 'value'})
    response = requests.get('url')
    assert response.json()['data'] == 'value'
"""
```
