# Database Patterns in Python

## SQLite Basics

### Why SQLite

```python
"""
SQLite - Embedded relational database:
- No separate server process
- Stores entire database in single file
- Zero configuration
- Perfect for: prototyping, testing, small apps, embedded systems
- Built into Python (sqlite3 module)

Limitations:
- Single writer at a time (readers don't block)
- Not suitable for high-concurrency web apps
- No user management/access control
"""
```

### Basic Operations

```python
import sqlite3

# Connect to database (creates file if doesn't exist)
conn = sqlite3.connect('myapp.db')

# For in-memory database (testing)
conn = sqlite3.connect(':memory:')

# Create cursor for executing SQL
cursor = conn.cursor()

# Create table
cursor.execute('''
    CREATE TABLE IF NOT EXISTS users (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        email TEXT UNIQUE NOT NULL,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    )
''')

# Insert data (use parameterized queries!)
cursor.execute(
    'INSERT INTO users (name, email) VALUES (?, ?)',
    ('Alice', 'alice@example.com')
)

# Insert multiple rows
users = [
    ('Bob', 'bob@example.com'),
    ('Charlie', 'charlie@example.com'),
]
cursor.executemany(
    'INSERT INTO users (name, email) VALUES (?, ?)',
    users
)

# Commit changes (required for writes!)
conn.commit()

# Query data
cursor.execute('SELECT * FROM users WHERE name = ?', ('Alice',))
row = cursor.fetchone()  # Returns tuple or None
print(row)  # (1, 'Alice', 'alice@example.com', '2024-01-15 10:30:00')

# Fetch all rows
cursor.execute('SELECT * FROM users')
all_users = cursor.fetchall()  # List of tuples

# Iterate over results
cursor.execute('SELECT * FROM users')
for row in cursor:
    print(f"User: {row[0]}, {row[1]}")

# Close connection when done
conn.close()
```

### Row Factories

```python
import sqlite3

conn = sqlite3.connect('myapp.db')

# Access columns by name instead of index
conn.row_factory = sqlite3.Row

cursor = conn.cursor()
cursor.execute('SELECT * FROM users WHERE id = ?', (1,))
user = cursor.fetchone()

# Now access by name
print(user['name'])      # 'Alice'
print(user['email'])     # 'alice@example.com'
print(dict(user))        # Convert to dict

# Custom row factory for named tuples
from collections import namedtuple

def namedtuple_factory(cursor, row):
    fields = [column[0] for column in cursor.description]
    Row = namedtuple('Row', fields)
    return Row(*row)

conn.row_factory = namedtuple_factory
cursor = conn.cursor()
cursor.execute('SELECT * FROM users')
user = cursor.fetchone()
print(user.name)  # 'Alice'
```

### Context Manager

```python
import sqlite3

# Automatically commits or rolls back
with sqlite3.connect('myapp.db') as conn:
    cursor = conn.cursor()
    cursor.execute('INSERT INTO users (name, email) VALUES (?, ?)',
                   ('Dave', 'dave@example.com'))
    # Auto-commits on success, rolls back on exception

# Connection is NOT closed automatically!
# Use this pattern for both:
def get_connection():
    conn = sqlite3.connect('myapp.db')
    conn.row_factory = sqlite3.Row
    return conn

# Usage
with get_connection() as conn:
    try:
        cursor = conn.cursor()
        # ... operations
    finally:
        conn.close()
```

---

## SQLAlchemy Core

### Engine and Connection

```python
from sqlalchemy import create_engine, text

# Create engine (connection pool)
# Format: dialect+driver://username:password@host:port/database

# SQLite
engine = create_engine('sqlite:///myapp.db')
engine = create_engine('sqlite:///:memory:')  # In-memory

# PostgreSQL
engine = create_engine('postgresql://user:pass@localhost:5432/mydb')

# MySQL
engine = create_engine('mysql+pymysql://user:pass@localhost:3306/mydb')

# Connection options
engine = create_engine(
    'postgresql://user:pass@localhost/mydb',
    pool_size=5,           # Connections in pool
    max_overflow=10,       # Extra connections allowed
    pool_timeout=30,       # Wait time for connection
    pool_recycle=1800,     # Recycle connections after 30 min
    echo=True              # Log SQL statements
)

# Execute raw SQL
with engine.connect() as conn:
    result = conn.execute(text("SELECT * FROM users WHERE id = :id"), {"id": 1})
    for row in result:
        print(row)

    # For writes, commit explicitly
    conn.execute(text("INSERT INTO users (name) VALUES (:name)"), {"name": "Eve"})
    conn.commit()
```

### Table Definition (Core)

```python
from sqlalchemy import (
    MetaData, Table, Column, Integer, String, DateTime,
    ForeignKey, create_engine, select, insert, update, delete
)
from datetime import datetime

metadata = MetaData()

# Define tables
users = Table(
    'users', metadata,
    Column('id', Integer, primary_key=True),
    Column('name', String(100), nullable=False),
    Column('email', String(255), unique=True, nullable=False),
    Column('created_at', DateTime, default=datetime.utcnow)
)

posts = Table(
    'posts', metadata,
    Column('id', Integer, primary_key=True),
    Column('title', String(200), nullable=False),
    Column('content', String),
    Column('user_id', Integer, ForeignKey('users.id'), nullable=False)
)

# Create tables
engine = create_engine('sqlite:///myapp.db')
metadata.create_all(engine)

# Insert
with engine.connect() as conn:
    result = conn.execute(
        insert(users).values(name='Alice', email='alice@example.com')
    )
    user_id = result.inserted_primary_key[0]
    conn.commit()

# Select
with engine.connect() as conn:
    result = conn.execute(select(users).where(users.c.id == 1))
    user = result.fetchone()
    print(user.name)

# Update
with engine.connect() as conn:
    conn.execute(
        update(users).where(users.c.id == 1).values(name='Alice Smith')
    )
    conn.commit()

# Delete
with engine.connect() as conn:
    conn.execute(delete(users).where(users.c.id == 1))
    conn.commit()
```

---

## SQLAlchemy ORM

### Model Definition

```python
from sqlalchemy import create_engine, Column, Integer, String, DateTime, ForeignKey
from sqlalchemy.orm import declarative_base, relationship, sessionmaker
from datetime import datetime

Base = declarative_base()

class User(Base):
    __tablename__ = 'users'

    id = Column(Integer, primary_key=True)
    name = Column(String(100), nullable=False)
    email = Column(String(255), unique=True, nullable=False)
    created_at = Column(DateTime, default=datetime.utcnow)

    # Relationships
    posts = relationship('Post', back_populates='author', cascade='all, delete-orphan')

    def __repr__(self):
        return f"<User(id={self.id}, name='{self.name}')>"


class Post(Base):
    __tablename__ = 'posts'

    id = Column(Integer, primary_key=True)
    title = Column(String(200), nullable=False)
    content = Column(String)
    user_id = Column(Integer, ForeignKey('users.id'), nullable=False)
    created_at = Column(DateTime, default=datetime.utcnow)

    # Relationship
    author = relationship('User', back_populates='posts')

    def __repr__(self):
        return f"<Post(id={self.id}, title='{self.title}')>"


# Create engine and tables
engine = create_engine('sqlite:///myapp.db', echo=True)
Base.metadata.create_all(engine)

# Create session factory
Session = sessionmaker(bind=engine)
```

### SQLAlchemy 2.0 Style (Modern)

```python
from sqlalchemy import create_engine, String, ForeignKey
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship, Session
from datetime import datetime
from typing import List, Optional

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = 'users'

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    email: Mapped[str] = mapped_column(String(255), unique=True)
    created_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)

    posts: Mapped[List["Post"]] = relationship(back_populates="author")


class Post(Base):
    __tablename__ = 'posts'

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    content: Mapped[Optional[str]]
    user_id: Mapped[int] = mapped_column(ForeignKey('users.id'))

    author: Mapped["User"] = relationship(back_populates="posts")
```

### Session Operations (CRUD)

```python
from sqlalchemy.orm import Session

engine = create_engine('sqlite:///myapp.db')

# Create
with Session(engine) as session:
    user = User(name='Alice', email='alice@example.com')
    session.add(user)
    session.commit()
    print(f"Created user with id: {user.id}")

# Read
with Session(engine) as session:
    # Get by primary key
    user = session.get(User, 1)

    # Query with filter
    user = session.query(User).filter(User.email == 'alice@example.com').first()

    # SQLAlchemy 2.0 style
    from sqlalchemy import select
    stmt = select(User).where(User.email == 'alice@example.com')
    user = session.execute(stmt).scalar_one_or_none()

    # Multiple results
    all_users = session.query(User).all()

    # With conditions
    active_users = session.query(User).filter(
        User.created_at >= datetime(2024, 1, 1)
    ).order_by(User.name).limit(10).all()

# Update
with Session(engine) as session:
    user = session.get(User, 1)
    if user:
        user.name = 'Alice Smith'
        session.commit()

    # Bulk update
    session.query(User).filter(User.name.like('Test%')).update(
        {'name': 'Updated'},
        synchronize_session='fetch'
    )
    session.commit()

# Delete
with Session(engine) as session:
    user = session.get(User, 1)
    if user:
        session.delete(user)
        session.commit()

    # Bulk delete
    session.query(User).filter(User.id > 100).delete()
    session.commit()
```

### Relationships and Eager Loading

```python
from sqlalchemy.orm import joinedload, selectinload

# Problem: N+1 queries
with Session(engine) as session:
    users = session.query(User).all()
    for user in users:
        print(user.posts)  # Each access = separate query!

# Solution: Eager loading

# joinedload - Single query with JOIN
with Session(engine) as session:
    users = session.query(User).options(
        joinedload(User.posts)
    ).all()
    # One query: SELECT users JOIN posts

# selectinload - Two queries (better for many-to-many)
with Session(engine) as session:
    users = session.query(User).options(
        selectinload(User.posts)
    ).all()
    # Query 1: SELECT users
    # Query 2: SELECT posts WHERE user_id IN (...)

# SQLAlchemy 2.0 style
from sqlalchemy import select

stmt = select(User).options(joinedload(User.posts))
users = session.execute(stmt).unique().scalars().all()

# Nested eager loading
stmt = select(User).options(
    joinedload(User.posts).joinedload(Post.comments)
)
```

---

## Transactions

### Basic Transaction Handling

```python
from sqlalchemy.orm import Session
from sqlalchemy.exc import IntegrityError

engine = create_engine('sqlite:///myapp.db')

# Automatic transaction with context manager
with Session(engine) as session:
    try:
        user = User(name='Alice', email='alice@example.com')
        session.add(user)

        post = Post(title='First Post', content='Hello', author=user)
        session.add(post)

        session.commit()  # Commits all changes

    except IntegrityError:
        session.rollback()  # Undo all changes
        raise

# Transaction is automatically rolled back if exception occurs
# before commit
```

### Nested Transactions (Savepoints)

```python
from sqlalchemy.orm import Session

with Session(engine) as session:
    user = User(name='Alice', email='alice@example.com')
    session.add(user)

    # Create savepoint
    savepoint = session.begin_nested()
    try:
        # Risky operation
        post = Post(title='Draft', author=user)
        session.add(post)
        session.flush()  # Execute SQL but don't commit

        # Something goes wrong
        raise ValueError("Changed my mind")

    except ValueError:
        savepoint.rollback()  # Only rolls back to savepoint
        # User is still in session, just not the post

    session.commit()  # User is saved
```

### Manual Transaction Control

```python
from sqlalchemy.orm import Session

with Session(engine) as session:
    # Start transaction explicitly
    with session.begin():
        user = User(name='Alice', email='alice@example.com')
        session.add(user)
        # Auto-commits when exiting block
        # Auto-rolls back on exception

# Without context manager
session = Session(engine)
try:
    user = User(name='Bob', email='bob@example.com')
    session.add(user)
    session.commit()
except:
    session.rollback()
    raise
finally:
    session.close()
```

### Transaction Isolation Levels

```python
from sqlalchemy import create_engine

# Set isolation level on engine
engine = create_engine(
    'postgresql://user:pass@localhost/mydb',
    isolation_level='REPEATABLE READ'
)

# Available levels (database dependent):
# - READ UNCOMMITTED
# - READ COMMITTED (default for most)
# - REPEATABLE READ
# - SERIALIZABLE

# Per-connection isolation
with engine.connect().execution_options(
    isolation_level='SERIALIZABLE'
) as conn:
    # This connection uses SERIALIZABLE
    pass
```

---

## Connection Patterns

### Connection Pool

```python
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool, NullPool

# Default pool (QueuePool)
engine = create_engine(
    'postgresql://user:pass@localhost/mydb',
    pool_size=5,        # Maintained connections
    max_overflow=10,    # Additional connections allowed
    pool_timeout=30,    # Seconds to wait for connection
    pool_recycle=1800,  # Recycle connections after 30 min
    pool_pre_ping=True  # Check connection health before using
)

# Disable pooling (for testing or specific use cases)
engine = create_engine(
    'postgresql://user:pass@localhost/mydb',
    poolclass=NullPool
)

# Check pool status
print(engine.pool.status())
```

### Session Patterns

```python
from sqlalchemy.orm import sessionmaker, scoped_session
from contextlib import contextmanager

# Session factory
Session = sessionmaker(bind=engine)

# Pattern 1: Manual session management
def create_user(name: str, email: str) -> User:
    session = Session()
    try:
        user = User(name=name, email=email)
        session.add(user)
        session.commit()
        return user
    except:
        session.rollback()
        raise
    finally:
        session.close()

# Pattern 2: Context manager
@contextmanager
def get_session():
    session = Session()
    try:
        yield session
        session.commit()
    except:
        session.rollback()
        raise
    finally:
        session.close()

def create_user(name: str, email: str) -> User:
    with get_session() as session:
        user = User(name=name, email=email)
        session.add(user)
        return user

# Pattern 3: Scoped session (thread-local)
# Good for web applications
session_factory = sessionmaker(bind=engine)
ScopedSession = scoped_session(session_factory)

def get_user(user_id: int) -> User:
    return ScopedSession.query(User).get(user_id)

# Remove session when done (e.g., end of request)
@app.teardown_appcontext
def shutdown_session(exception=None):
    ScopedSession.remove()
```

### Dependency Injection (FastAPI Pattern)

```python
from fastapi import FastAPI, Depends
from sqlalchemy.orm import Session
from typing import Generator

app = FastAPI()

def get_db() -> Generator[Session, None, None]:
    """Dependency that provides database session."""
    db = Session(engine)
    try:
        yield db
    finally:
        db.close()

@app.get("/users/{user_id}")
def get_user(user_id: int, db: Session = Depends(get_db)):
    user = db.query(User).filter(User.id == user_id).first()
    if not user:
        raise HTTPException(status_code=404)
    return user

@app.post("/users")
def create_user(name: str, email: str, db: Session = Depends(get_db)):
    user = User(name=name, email=email)
    db.add(user)
    db.commit()
    db.refresh(user)
    return user
```

---

## Query Patterns

### Complex Queries

```python
from sqlalchemy import select, func, and_, or_, desc
from sqlalchemy.orm import Session

with Session(engine) as session:
    # Multiple conditions
    stmt = select(User).where(
        and_(
            User.created_at >= datetime(2024, 1, 1),
            User.name.like('A%')
        )
    )

    # OR conditions
    stmt = select(User).where(
        or_(
            User.email.endswith('@gmail.com'),
            User.email.endswith('@yahoo.com')
        )
    )

    # Ordering
    stmt = select(User).order_by(desc(User.created_at))

    # Limit and offset
    stmt = select(User).limit(10).offset(20)

    # Aggregations
    stmt = select(func.count(User.id))
    count = session.execute(stmt).scalar()

    # Group by
    stmt = select(
        User.name,
        func.count(Post.id).label('post_count')
    ).join(Post).group_by(User.id)

    # Having
    stmt = select(
        User.name,
        func.count(Post.id).label('post_count')
    ).join(Post).group_by(User.id).having(func.count(Post.id) > 5)

    # Subqueries
    subq = select(func.max(Post.created_at)).where(
        Post.user_id == User.id
    ).scalar_subquery()

    stmt = select(User, subq.label('latest_post'))
```

### Raw SQL with ORM

```python
from sqlalchemy import text

with Session(engine) as session:
    # Raw SQL returning model instances
    result = session.execute(
        text("SELECT * FROM users WHERE name = :name"),
        {"name": "Alice"}
    )
    for row in result:
        print(row)

    # Map to model
    users = session.query(User).from_statement(
        text("SELECT * FROM users WHERE id > :id")
    ).params(id=10).all()
```

---

## Migrations with Alembic

### Setup

```bash
# Install alembic
pip install alembic

# Initialize alembic in project
alembic init alembic

# Configure alembic.ini and env.py
```

```python
# alembic/env.py
from myapp.models import Base
target_metadata = Base.metadata
```

### Creating Migrations

```bash
# Auto-generate migration from model changes
alembic revision --autogenerate -m "Add users table"

# Create empty migration
alembic revision -m "Custom migration"
```

```python
# alembic/versions/xxxx_add_users_table.py
"""Add users table

Revision ID: abc123
"""
from alembic import op
import sqlalchemy as sa

def upgrade():
    op.create_table(
        'users',
        sa.Column('id', sa.Integer(), nullable=False),
        sa.Column('name', sa.String(100), nullable=False),
        sa.Column('email', sa.String(255), nullable=False),
        sa.PrimaryKeyConstraint('id'),
        sa.UniqueConstraint('email')
    )

def downgrade():
    op.drop_table('users')
```

### Running Migrations

```bash
# Apply all migrations
alembic upgrade head

# Apply specific revision
alembic upgrade abc123

# Rollback one step
alembic downgrade -1

# Rollback to beginning
alembic downgrade base

# Show current revision
alembic current

# Show history
alembic history
```

---

## Testing Database Code

### Using In-Memory SQLite

```python
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import Session

@pytest.fixture
def db_session():
    """Create clean database for each test."""
    engine = create_engine('sqlite:///:memory:')
    Base.metadata.create_all(engine)

    session = Session(engine)
    yield session

    session.close()

def test_create_user(db_session):
    user = User(name='Alice', email='alice@test.com')
    db_session.add(user)
    db_session.commit()

    assert user.id is not None
    assert db_session.query(User).count() == 1

def test_user_posts(db_session):
    user = User(name='Bob', email='bob@test.com')
    post = Post(title='Test', content='Content', author=user)

    db_session.add(user)
    db_session.commit()

    assert len(user.posts) == 1
    assert post.author == user
```

### Using Transactions for Test Isolation

```python
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import Session

@pytest.fixture(scope='module')
def engine():
    """Create test database once per module."""
    engine = create_engine('postgresql://test:test@localhost/testdb')
    Base.metadata.create_all(engine)
    yield engine
    Base.metadata.drop_all(engine)

@pytest.fixture
def db_session(engine):
    """Create session with transaction rollback."""
    connection = engine.connect()
    transaction = connection.begin()
    session = Session(bind=connection)

    yield session

    session.close()
    transaction.rollback()
    connection.close()

# Each test runs in rolled-back transaction
# Fast and isolated
```

---

## Best Practices

```python
"""
Database Best Practices:

1. Always Use Parameterized Queries
   - Never concatenate user input into SQL
   - cursor.execute("SELECT * FROM users WHERE id = ?", (user_id,))

2. Use Connection Pooling
   - Don't create new connections for each request
   - Use SQLAlchemy engine with proper pool settings

3. Handle Transactions Properly
   - Commit explicitly or use context managers
   - Always handle rollback on errors
   - Keep transactions short

4. Use Migrations
   - Track schema changes with Alembic
   - Never modify production schema manually
   - Version control your migrations

5. Optimize Queries
   - Use eager loading to avoid N+1 problems
   - Add indexes for frequently queried columns
   - Use EXPLAIN to analyze slow queries

6. Close Connections
   - Use context managers
   - Implement proper cleanup in web apps
   - Don't leak connections

7. Separate Models from Business Logic
   - Models define schema and relationships
   - Services/repositories handle business logic
   - Easier to test and maintain
"""
```

---

## Interview Questions

```python
# Q1: What is the N+1 query problem and how do you solve it?
"""
N+1 Problem:
- Loading N related records requires N+1 queries
- 1 query to load parent, N queries to load children

Example:
users = session.query(User).all()  # 1 query
for user in users:
    print(user.posts)  # N queries!

Solutions:
1. joinedload: Single JOIN query
2. selectinload: Two queries (IN clause)
3. subqueryload: Two queries (subquery)

session.query(User).options(joinedload(User.posts)).all()
"""

# Q2: Explain SQLAlchemy session lifecycle
"""
Session lifecycle:
1. Create session from factory
2. Add/query objects (objects become "pending" or "persistent")
3. flush() - sends SQL to database (within transaction)
4. commit() - commits transaction
5. rollback() - undoes changes
6. close() - releases connection

Object states:
- Transient: Not attached to session
- Pending: Added but not flushed
- Persistent: Flushed/queried, tracked
- Detached: Was persistent, session closed
"""

# Q3: How do you handle database transactions?
"""
1. Use context managers:
   with Session(engine) as session:
       session.add(obj)
       session.commit()  # or auto-rollback on exception

2. Explicit transaction:
   with session.begin():
       session.add(obj)
       # auto-commits at end

3. Savepoints for partial rollback:
   savepoint = session.begin_nested()
   try:
       # risky operation
   except:
       savepoint.rollback()
"""

# Q4: What's the difference between session.flush() and session.commit()?
"""
flush():
- Sends pending SQL to database
- Stays within current transaction
- Changes visible within session
- Can be rolled back

commit():
- Calls flush() first
- Commits the transaction
- Changes become permanent
- Starts new transaction

Use flush when:
- Need generated IDs before commit
- Testing constraints within transaction
- Batch operations
"""

# Q5: How do you test database code?
"""
1. In-memory SQLite:
   - Fast, isolated
   - Create fresh DB per test

2. Transaction rollback:
   - Real database
   - Wrap each test in transaction
   - Rollback after test

3. Test fixtures:
   - Create known test data
   - Use factories (factory_boy)
   - Clean up in teardown

Best practice: Use real database type for tests
(SQLite behavior differs from PostgreSQL)
"""
```
