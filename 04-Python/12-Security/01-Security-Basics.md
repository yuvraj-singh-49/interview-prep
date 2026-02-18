# Security in Python Applications

## Input Validation

### Why Input Validation Matters

```python
"""
Never trust user input!

All external input is potentially malicious:
- HTTP request parameters
- File uploads
- Database records from external sources
- API responses from third parties
- Environment variables (in some contexts)
"""
```

### String Validation

```python
import re

# Length validation
def validate_username(username: str) -> bool:
    if not username:
        raise ValueError("Username required")
    if len(username) < 3 or len(username) > 20:
        raise ValueError("Username must be 3-20 characters")
    if not re.match(r'^[a-zA-Z0-9_]+$', username):
        raise ValueError("Username can only contain letters, numbers, underscore")
    return True

# Email validation (basic)
def validate_email(email: str) -> bool:
    pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    if not re.match(pattern, email):
        raise ValueError("Invalid email format")
    return True

# For production, use email-validator library
# pip install email-validator
from email_validator import validate_email, EmailNotValidError

def validate_email_proper(email: str) -> str:
    try:
        valid = validate_email(email)
        return valid.email  # Normalized email
    except EmailNotValidError as e:
        raise ValueError(str(e))
```

### Numeric Validation

```python
def validate_age(age: int) -> bool:
    if not isinstance(age, int):
        raise TypeError("Age must be integer")
    if age < 0 or age > 150:
        raise ValueError("Age must be between 0 and 150")
    return True

def validate_price(price: float) -> bool:
    if price < 0:
        raise ValueError("Price cannot be negative")
    if price > 1000000:
        raise ValueError("Price exceeds maximum")
    # Check decimal places
    if round(price, 2) != price:
        raise ValueError("Price can only have 2 decimal places")
    return True
```

### Using Pydantic (Recommended)

```python
from pydantic import BaseModel, Field, field_validator, EmailStr
from typing import Optional

class UserCreate(BaseModel):
    username: str = Field(..., min_length=3, max_length=20, pattern=r'^[a-zA-Z0-9_]+$')
    email: EmailStr
    age: int = Field(..., ge=0, le=150)
    password: str = Field(..., min_length=8)

    @field_validator('password')
    @classmethod
    def password_strength(cls, v):
        if not any(c.isupper() for c in v):
            raise ValueError('Password must contain uppercase letter')
        if not any(c.isdigit() for c in v):
            raise ValueError('Password must contain digit')
        return v

# Validation happens automatically
try:
    user = UserCreate(
        username="alice123",
        email="alice@example.com",
        age=25,
        password="SecurePass123"
    )
except ValidationError as e:
    print(e.errors())
```

---

## SQL Injection Prevention

### The Vulnerability

```python
# DANGEROUS - SQL Injection vulnerable!
def get_user_bad(username):
    query = f"SELECT * FROM users WHERE username = '{username}'"
    cursor.execute(query)
    # If username = "' OR '1'='1"
    # Query becomes: SELECT * FROM users WHERE username = '' OR '1'='1'
    # Returns ALL users!

    # Even worse: username = "'; DROP TABLE users; --"
    # Can delete entire table!
```

### Prevention with Parameterized Queries

```python
import sqlite3

# SAFE - Parameterized query
def get_user_safe(username):
    query = "SELECT * FROM users WHERE username = ?"
    cursor.execute(query, (username,))
    return cursor.fetchone()

# With named parameters
def get_user_safe(username):
    query = "SELECT * FROM users WHERE username = :username"
    cursor.execute(query, {"username": username})
    return cursor.fetchone()

# PostgreSQL with psycopg2
import psycopg2

def get_user_postgres(username):
    query = "SELECT * FROM users WHERE username = %s"
    cursor.execute(query, (username,))
    return cursor.fetchone()

# SQLAlchemy ORM (automatically parameterized)
from sqlalchemy import select
from models import User

def get_user_sqlalchemy(session, username):
    stmt = select(User).where(User.username == username)
    return session.execute(stmt).scalar_one_or_none()
```

### SQLAlchemy Best Practices

```python
from sqlalchemy import text

# DANGEROUS - String formatting in raw SQL
def search_bad(search_term):
    query = text(f"SELECT * FROM products WHERE name LIKE '%{search_term}%'")
    return session.execute(query).fetchall()

# SAFE - Parameterized raw SQL
def search_safe(search_term):
    query = text("SELECT * FROM products WHERE name LIKE :term")
    return session.execute(query, {"term": f"%{search_term}%"}).fetchall()

# BEST - Use ORM
def search_best(search_term):
    return session.query(Product).filter(
        Product.name.ilike(f"%{search_term}%")
    ).all()
```

---

## Command Injection Prevention

### The Vulnerability

```python
import os
import subprocess

# DANGEROUS - Command injection!
def ping_host_bad(host):
    os.system(f"ping -c 1 {host}")
    # If host = "google.com; rm -rf /"
    # Executes: ping -c 1 google.com; rm -rf /

# Also dangerous with subprocess shell=True
def ping_host_still_bad(host):
    subprocess.run(f"ping -c 1 {host}", shell=True)
```

### Prevention

```python
import subprocess
import shlex

# SAFE - Pass arguments as list (no shell)
def ping_host_safe(host):
    # Validate input first
    if not re.match(r'^[a-zA-Z0-9.-]+$', host):
        raise ValueError("Invalid hostname")

    result = subprocess.run(
        ['ping', '-c', '1', host],  # Arguments as list
        capture_output=True,
        text=True,
        timeout=10
    )
    return result.stdout

# If you MUST use shell (avoid if possible)
def run_command_safer(command):
    # Escape shell metacharacters
    safe_command = shlex.quote(command)
    subprocess.run(f"echo {safe_command}", shell=True)

# Best: Avoid shell entirely, use Python libraries
# Instead of: subprocess.run("ls -la /path", shell=True)
# Use: os.listdir("/path") or pathlib.Path("/path").iterdir()
```

---

## Password Security

### Password Hashing

```python
# NEVER store plaintext passwords!

# Use bcrypt (recommended)
# pip install bcrypt
import bcrypt

def hash_password(password: str) -> bytes:
    # Generate salt and hash
    salt = bcrypt.gensalt()
    return bcrypt.hashpw(password.encode('utf-8'), salt)

def verify_password(password: str, hashed: bytes) -> bool:
    return bcrypt.checkpw(password.encode('utf-8'), hashed)

# Usage
hashed = hash_password("mypassword")
# b'$2b$12$...' (includes salt, algorithm info)

verify_password("mypassword", hashed)  # True
verify_password("wrongpass", hashed)   # False

# Alternative: argon2 (winner of Password Hashing Competition)
# pip install argon2-cffi
from argon2 import PasswordHasher

ph = PasswordHasher()

hashed = ph.hash("mypassword")
ph.verify(hashed, "mypassword")  # Returns True or raises exception

# NEVER use these for passwords:
# - MD5, SHA1, SHA256 (too fast, no salt)
# - Plain hashing without salt
# - Reversible encryption
```

### Password Validation

```python
import re

def validate_password_strength(password: str) -> list[str]:
    """Return list of issues with password."""
    issues = []

    if len(password) < 8:
        issues.append("Password must be at least 8 characters")
    if len(password) > 128:
        issues.append("Password too long")
    if not re.search(r'[a-z]', password):
        issues.append("Password must contain lowercase letter")
    if not re.search(r'[A-Z]', password):
        issues.append("Password must contain uppercase letter")
    if not re.search(r'\d', password):
        issues.append("Password must contain digit")
    if not re.search(r'[!@#$%^&*(),.?":{}|<>]', password):
        issues.append("Password must contain special character")

    # Check for common passwords
    common_passwords = {'password', '123456', 'password123', 'admin'}
    if password.lower() in common_passwords:
        issues.append("Password is too common")

    return issues
```

---

## Secrets Management

### Environment Variables

```python
import os

# Load secrets from environment
API_KEY = os.environ.get('API_KEY')
DATABASE_URL = os.environ.get('DATABASE_URL')

# NEVER do this:
API_KEY = "sk-1234567890"  # Hardcoded secret!

# Check at startup
if not API_KEY:
    raise RuntimeError("API_KEY environment variable required")

# Using python-dotenv for local development
from dotenv import load_dotenv

load_dotenv()  # Load from .env file (add .env to .gitignore!)
```

### Secrets in Code

```python
# WRONG: Secrets in code
DATABASE_PASSWORD = "supersecret"  # Gets committed to git!

# WRONG: Secrets in comments
# password: supersecret123  # Still visible in history!

# RIGHT: Reference environment/vault
DATABASE_PASSWORD = os.environ['DATABASE_PASSWORD']

# Check for accidental secrets in code
# Use tools like:
# - git-secrets
# - detect-secrets
# - truffleHog
# - Gitleaks
```

### Using Secret Managers

```python
# AWS Secrets Manager
import boto3
import json

def get_secret(secret_name: str) -> dict:
    client = boto3.client('secretsmanager')
    response = client.get_secret_value(SecretId=secret_name)
    return json.loads(response['SecretString'])

# HashiCorp Vault
# pip install hvac
import hvac

client = hvac.Client(url='http://localhost:8200')
client.token = os.environ['VAULT_TOKEN']
secret = client.secrets.kv.v2.read_secret_version(path='myapp/database')
password = secret['data']['data']['password']
```

---

## Common Vulnerabilities (OWASP Top 10)

### Path Traversal

```python
# VULNERABLE
def read_file_bad(filename):
    with open(f"/app/data/{filename}") as f:
        return f.read()
    # If filename = "../../../etc/passwd"
    # Reads /etc/passwd!

# SAFE
from pathlib import Path

def read_file_safe(filename):
    base_path = Path("/app/data").resolve()
    file_path = (base_path / filename).resolve()

    # Check that resolved path is under base_path
    if not file_path.is_relative_to(base_path):
        raise ValueError("Invalid file path")

    with open(file_path) as f:
        return f.read()
```

### XML External Entity (XXE)

```python
# VULNERABLE - XXE attack possible
import xml.etree.ElementTree as ET

def parse_xml_bad(xml_string):
    return ET.fromstring(xml_string)

# SAFE - Disable external entities
from defusedxml import ElementTree

def parse_xml_safe(xml_string):
    return ElementTree.fromstring(xml_string)

# Or configure standard library
import xml.etree.ElementTree as ET

def parse_xml_safe(xml_string):
    parser = ET.XMLParser()
    parser.entity = {}  # Disable entity expansion
    return ET.fromstring(xml_string, parser=parser)

# Best: Use defusedxml library
# pip install defusedxml
```

### Insecure Deserialization

```python
# DANGEROUS - Never unpickle untrusted data!
import pickle

def load_data_bad(data):
    return pickle.loads(data)  # Can execute arbitrary code!

# Attackers can craft pickle that runs any command:
# pickle.dumps(os.system("rm -rf /"))

# SAFE alternatives:
# 1. Use JSON for data exchange
import json
data = json.loads(json_string)

# 2. If you must use pickle, sign it
import hmac
import hashlib

SECRET_KEY = os.environ['SECRET_KEY'].encode()

def safe_pickle_dumps(obj):
    data = pickle.dumps(obj)
    signature = hmac.new(SECRET_KEY, data, hashlib.sha256).hexdigest()
    return signature + '|' + data.hex()

def safe_pickle_loads(signed_data):
    signature, data_hex = signed_data.split('|')
    data = bytes.fromhex(data_hex)
    expected_sig = hmac.new(SECRET_KEY, data, hashlib.sha256).hexdigest()
    if not hmac.compare_digest(signature, expected_sig):
        raise ValueError("Invalid signature")
    return pickle.loads(data)
```

### Server-Side Request Forgery (SSRF)

```python
# VULNERABLE - User controls URL
def fetch_url_bad(url):
    return requests.get(url).text
    # User could request: http://169.254.169.254/latest/meta-data/
    # (AWS metadata endpoint - leaks credentials!)

# SAFE - Validate and restrict URLs
from urllib.parse import urlparse
import ipaddress

ALLOWED_HOSTS = {'api.example.com', 'cdn.example.com'}

def fetch_url_safe(url):
    parsed = urlparse(url)

    # Only allow HTTPS
    if parsed.scheme != 'https':
        raise ValueError("Only HTTPS allowed")

    # Whitelist hosts
    if parsed.hostname not in ALLOWED_HOSTS:
        raise ValueError("Host not allowed")

    # Block internal IPs
    try:
        ip = ipaddress.ip_address(parsed.hostname)
        if ip.is_private or ip.is_loopback or ip.is_link_local:
            raise ValueError("Internal IP not allowed")
    except ValueError:
        pass  # Not an IP, it's a hostname

    return requests.get(url, timeout=10).text
```

---

## Cryptography

### Encryption Best Practices

```python
# Use cryptography library
# pip install cryptography
from cryptography.fernet import Fernet

# Generate key (do this once, store securely)
key = Fernet.generate_key()
# b'...' - Store in secrets manager!

# Encrypt/decrypt
cipher = Fernet(key)
encrypted = cipher.encrypt(b"secret data")
decrypted = cipher.decrypt(encrypted)

# For passwords, use hashing (bcrypt/argon2), NOT encryption!
# Encryption is for data that needs to be retrieved.
```

### Secure Random Numbers

```python
import secrets
import os

# For security-sensitive operations, use secrets module
# NOT random module (which is predictable)

# Generate secure token
token = secrets.token_hex(32)      # 64 hex characters
token = secrets.token_urlsafe(32)  # URL-safe base64

# Generate random bytes
random_bytes = secrets.token_bytes(32)

# Generate random integer
random_int = secrets.randbelow(100)  # 0 to 99

# Secure comparison (timing-attack safe)
secrets.compare_digest(a, b)

# For cryptographic operations
random_bytes = os.urandom(32)

# NEVER use for security:
import random  # Predictable!
random.randint(0, 100)  # NOT secure
```

---

## Security Headers (Web Applications)

```python
# Flask example
from flask import Flask, Response

app = Flask(__name__)

@app.after_request
def add_security_headers(response: Response) -> Response:
    # Prevent clickjacking
    response.headers['X-Frame-Options'] = 'DENY'

    # Prevent MIME type sniffing
    response.headers['X-Content-Type-Options'] = 'nosniff'

    # Enable XSS filter
    response.headers['X-XSS-Protection'] = '1; mode=block'

    # Content Security Policy
    response.headers['Content-Security-Policy'] = "default-src 'self'"

    # Force HTTPS
    response.headers['Strict-Transport-Security'] = 'max-age=31536000; includeSubDomains'

    return response

# Or use Flask-Talisman
# pip install flask-talisman
from flask_talisman import Talisman
Talisman(app)
```

---

## Security Checklist

```python
"""
Security Checklist for Python Applications:

Input Validation:
☐ Validate all input (length, format, type)
☐ Use allowlists, not blocklists
☐ Use parameterized queries for SQL
☐ Escape output (HTML, JavaScript)

Authentication:
☐ Use strong password hashing (bcrypt/argon2)
☐ Implement rate limiting on login
☐ Use secure session management
☐ Implement MFA where possible

Secrets:
☐ No hardcoded secrets in code
☐ Use environment variables or secret managers
☐ Rotate secrets regularly
☐ Use separate secrets per environment

Dependencies:
☐ Keep dependencies updated
☐ Use pip-audit or safety for vulnerability scanning
☐ Pin dependency versions
☐ Review new dependencies before adding

Infrastructure:
☐ Use HTTPS everywhere
☐ Set security headers
☐ Enable logging and monitoring
☐ Implement proper error handling (don't leak info)

Code:
☐ Use defusedxml for XML parsing
☐ Avoid pickle with untrusted data
☐ Validate file paths to prevent traversal
☐ Use subprocess with shell=False
"""
```

---

## Interview Questions

```python
# Q1: How do you prevent SQL injection?
"""
Use parameterized queries / prepared statements:
cursor.execute("SELECT * FROM users WHERE id = ?", (user_id,))

Never concatenate user input into SQL strings.
Use ORMs like SQLAlchemy for additional safety.
"""

# Q2: How should passwords be stored?
"""
1. Hash passwords with bcrypt or argon2
2. Never store plaintext
3. Never use MD5/SHA for passwords
4. Salt is included automatically in bcrypt
5. Hash on server, not client
"""

# Q3: What is the principle of least privilege?
"""
Give users/processes only the minimum permissions
needed to perform their tasks.

Examples:
- Database users with read-only access
- API keys with limited scope
- File permissions restricted appropriately
"""

# Q4: How do you handle secrets in applications?
"""
1. Never hardcode in source
2. Use environment variables for simple cases
3. Use secret managers (AWS Secrets Manager, Vault) for production
4. Rotate secrets regularly
5. Different secrets per environment
6. Add .env to .gitignore
"""
```
