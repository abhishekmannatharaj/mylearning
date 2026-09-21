# Complete Project Report

## 1. Project Purpose

This project is a FastAPI backend for an Instagram-style application. It provides:

- User registration
- User login
- JWT-based authentication
- Password hashing with bcrypt
- PostgreSQL database storage
- CRUD-style post endpoints
- Docker-based application and database setup
- Pydantic validation and response serialization
- SQLAlchemy ORM models

The application is organized into three main layers:

```text
FastAPI routes
    ↓
Pydantic schemas and authentication dependencies
    ↓
SQLAlchemy ORM models
    ↓
PostgreSQL database
```

---

# 2. Complete Project Structure

```text
app/
├── main.py
├── onlyfastapi.py
├── requirements.txt
├── readme.md
├── Dockerfile
├── docker-compose.yml
├── .env
├── .env.example
├── .gitignore
├── .dockerignore
├── api/
│   └── v1/
│       ├── auth/
│       │   ├── __init__.py
│       │   ├── auth_router.py
│       │   ├── security.py
│       │   └── user_schemas.py
│       └── posts/
│           ├── __init__.py
│           ├── router.py
│           └── schemas.py
├── db/
│   ├── __init__.py
│   ├── database.py
│   └── models.py
├── img/
│   ├── image.png
│   └── img2.png
├── venv/
└── __pycache__/
```

The `venv` and `__pycache__` directories are generated environment/cache directories and are not application source code.

---

# 3. Application Startup Flow

The application starts from [main.py](main.py).

The main startup sequence is:

```python
from db.database import engine, Base
import db.models
```

These imports trigger the following sequence:

1. `db.database` loads environment variables.
2. It creates the SQLAlchemy database engine.
3. It creates the SQLAlchemy session factory.
4. It creates the declarative `Base`.
5. It attempts to connect to PostgreSQL.
6. `db.models` defines the `Post` and `User` tables.
7. `Base.metadata.create_all(bind=engine)` creates missing tables.
8. FastAPI application is created.
9. The post and authentication routers are registered.

The table creation happens here:

```python
Base.metadata.create_all(bind=engine)
```

This means the application automatically creates database tables when it starts.

It does not perform database migrations. If the model changes later, existing tables will not automatically be altered.

---

# 4. [main.py](main.py)

This is the main application entry point.

```python
from fastapi import FastAPI
from db.database import engine, Base
import db.models
from api.v1.posts.router import router as posts_router
from api.v1.auth.auth_router import router as auth_router
```

## Imports

### `FastAPI`

Creates the web application.

### `engine`

The SQLAlchemy connection object used to communicate with PostgreSQL.

### `Base`

The base class used by SQLAlchemy models.

### `import db.models`

This import is important even though it is not directly used.

Importing `db.models` causes SQLAlchemy to register the `User` and `Post` tables with `Base.metadata`.

Without importing the models before this line:

```python
Base.metadata.create_all(bind=engine)
```

SQLAlchemy may not know that the tables exist.

## Database table creation

```python
Base.metadata.create_all(bind=engine)
```

This checks whether the declared tables exist and creates missing tables.

The current tables are:

```text
users
posts
```

## FastAPI application definition

```python
app = FastAPI(
    title="Instagram-Style Backend API",
    version="1.0.0",
    description="Modular CRUD backend using FastAPI, SQLAlchemy, and PostgreSQL",
)
```

This metadata appears in the automatically generated Swagger documentation.

The documentation is available at:

```text
http://localhost:8000/docs
```

The ReDoc documentation is usually available at:

```text
http://localhost:8000/redoc
```

## Root endpoint

```python
@app.get("/")
def read_root():
    return {"message": "Server is up and running"}
```

A request to:

```text
GET /
```

returns:

```json
{
  "message": "Server is up and running"
}
```

## Router registration

```python
app.include_router(posts_router, prefix="/api/v1")
app.include_router(auth_router, prefix="/api/v1")
```

The routers already have their own prefixes:

```python
router = APIRouter(prefix="/posts")
```

and:

```python
router = APIRouter(prefix="/auth")
```

The final route prefixes are therefore:

```text
/api/v1/posts
/api/v1/auth
```

---

# 5. [db/database.py](db/database.py)

This file manages the database connection and SQLAlchemy configuration.

## Environment loading

```python
BASE_DIR = Path(__file__).resolve().parent.parent
load_dotenv(dotenv_path=BASE_DIR / ".env")
```

`BASE_DIR` resolves to the project’s `app` directory.

The application then loads environment variables from:

```text
app/.env
```

The configuration values are read using:

```python
db_user = os.getenv("DB_USER", "postgres")
db_password = os.getenv("DB_PASSWORD")
db_host = os.getenv("DB_HOST", "localhost")
db_port = os.getenv("DB_PORT", "5432")
db_name = os.getenv("DB_NAME", "fastapi")
```

The defaults are:

| Variable | Default |
|---|---|
| `DB_USER` | `postgres` |
| `DB_PASSWORD` | No default |
| `DB_HOST` | `localhost` |
| `DB_PORT` | `5432` |
| `DB_NAME` | `fastapi` |

## SQLAlchemy URL

```python
SQLALCHEMY_DATABASE_URL = (
    f"postgresql://{db_user}:{db_password}@"
    f"{db_host}:{db_port}/{db_name}"
)
```

This produces a connection URL similar to:

```text
postgresql://postgres:password@localhost:5432/fastapi
```

The actual password should never be placed in source code or documentation.

## Engine

```python
engine = create_engine(SQLALCHEMY_DATABASE_URL)
```

The engine manages communication between SQLAlchemy and PostgreSQL.

It does not necessarily open a database connection immediately. Connections are usually acquired from the engine’s connection pool when required.

## Session factory

```python
SessionLocal = sessionmaker(
    autocommit=False,
    autoflush=False,
    bind=engine
)
```

`SessionLocal` is a factory for creating database sessions.

Each API request normally gets its own session.

## Declarative base

```python
Base = declarative_base()
```

All SQLAlchemy models inherit from this base.

For example:

```python
class Post(Base):
    ...
```

The base keeps track of model metadata and table definitions.

## Request database dependency

```python
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

This is a FastAPI dependency.

A route can use it like this:

```python
db: Session = Depends(get_db)
```

The lifecycle is:

```text
Request begins
    ↓
SessionLocal() creates a session
    ↓
Route uses the session
    ↓
Route completes
    ↓
finally block closes the session
```

This prevents database sessions from remaining open after the request.

## Connection retry loop

```python
while True:
    try:
        connection = psycopg2.connect(...)
        cursor = connection.cursor()
        print("Database connection was successful")
        break
    except Exception as error:
        print("Connecting to database failed")
        print("Error: ", error)
        time.sleep(2)
```

At import time, the application attempts to connect to PostgreSQL.

If the connection fails, it waits two seconds and retries indefinitely.

This is useful in Docker because PostgreSQL may take a few seconds to start.

However, this loop also means:

- A wrong password can cause an infinite retry loop.
- A wrong hostname can cause the application to appear frozen.
- The connection and cursor created by this check are not explicitly closed.
- SQLAlchemy already manages its own connection pool separately.

The `psycopg2` check and SQLAlchemy connection are currently two separate database mechanisms.

---

# 6. [db/models.py](db/models.py)

This file defines the database tables using SQLAlchemy ORM.

## Post model

```python
class Post(Base):
    __tablename__ = "posts"
```

This maps the Python `Post` class to the PostgreSQL `posts` table.

### Columns

```python
id = Column(
    Integer,
    primary_key=True,
    nullable=False,
    index=True
)
```

The `id` column is:

- An integer
- The primary key
- Required
- Indexed
- Automatically suitable for identifying posts

```python
title = Column(String, nullable=False)
content = Column(String, nullable=False)
```

Both title and content are required.

```python
published = Column(
    Boolean,
    server_default="TRUE",
    nullable=False
)
```

If the client does not provide `published`, PostgreSQL uses `TRUE`.

```python
rating = Column(Integer, nullable=True)
```

The rating is optional.

```python
created_at = Column(
    TIMESTAMP(timezone=True),
    nullable=False,
    server_default=text("now()")
)
```

PostgreSQL creates the timestamp automatically using its own clock.

The database schema is conceptually:

```text
posts
├── id INTEGER PRIMARY KEY
├── title VARCHAR NOT NULL
├── content VARCHAR NOT NULL
├── published BOOLEAN NOT NULL DEFAULT TRUE
├── rating INTEGER NULL
└── created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT now()
```

## User model

```python
class User(Base):
    __tablename__ = "users"
```

The `User` class maps to the `users` table.

```python
id = Column(
    Integer,
    primary_key=True,
    nullable=False,
    index=True
)
```

```python
email = Column(
    String,
    nullable=False,
    unique=True,
    index=True
)
```

The email must be:

- Present
- Unique
- Indexed

```python
password = Column(String, nullable=False)
```

This stores the hashed password, not the original plain-text password.

```python
created_at = Column(
    TIMESTAMP(timezone=True),
    nullable=False,
    server_default=text("now()")
)
```

The database creates the user timestamp.

The user table is conceptually:

```text
users
├── id INTEGER PRIMARY KEY
├── email VARCHAR NOT NULL UNIQUE
├── password VARCHAR NOT NULL
└── created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT now()
```

## Current data-model limitation

The `Post` table does not contain a user relationship.

There is no:

```python
user_id = Column(...)
```

Therefore:

- Posts are not associated with their creator.
- Any authenticated user can create a post.
- The application cannot list posts by user.
- The application cannot restrict editing or deletion to the owner.
- There is no foreign key from `posts` to `users`.

---

# 7. [api/v1/posts/schemas.py](api/v1/posts/schemas.py)

This file defines Pydantic models for post input and output.

## `PostBase`

```python
class PostBase(BaseModel):
    title: str
    content: str
    published: bool = True
    rating: Optional[int] = None
```

This defines the shared post fields.

A valid request body might be:

```json
{
  "title": "My first post",
  "content": "Hello from FastAPI",
  "published": true,
  "rating": 5
}
```

If `published` is omitted, it defaults to `true`.

If `rating` is omitted, it defaults to `null`.

## `PostCreate`

```python
class PostCreate(PostBase):
    pass
```

Currently, `PostCreate` does not add anything to `PostBase`.

It exists as a separate schema so that the input contract can later evolve independently.

## `PostResponse`

```python
class PostResponse(PostBase):
    id: int
    created_at: datetime

    model_config = ConfigDict(from_attributes=True)
```

This is the output schema.

It contains all input fields plus:

- `id`
- `created_at`

The `from_attributes=True` option allows Pydantic to serialize a SQLAlchemy object directly.

For example, FastAPI can return a `Post` object and Pydantic converts it into JSON.

---

# 8. [api/v1/posts/router.py](api/v1/posts/router.py)

This file defines post API endpoints.

## Router definition

```python
router = APIRouter(
    prefix="/posts",
    tags=["Posts"]
)
```

The final prefix becomes:

```text
/api/v1/posts
```

## Read all posts

```python
@router.get("/", response_model=List[PostResponse])
async def read_posts(db: Session = Depends(get_db)):
    posts = db.query(Post).all()
    return posts
```

Endpoint:

```text
GET /api/v1/posts/
```

Flow:

```text
Client request
    ↓
FastAPI opens database session
    ↓
SQLAlchemy runs SELECT on posts
    ↓
Pydantic converts records to PostResponse
    ↓
JSON array is returned
    ↓
Database session closes
```

Example response:

```json
[
  {
    "title": "My first post",
    "content": "Hello from FastAPI",
    "published": true,
    "rating": 5,
    "id": 1,
    "created_at": "2026-09-21T10:30:00Z"
  }
]
```

This endpoint is currently public because it does not use `get_current_user`.

## Create a post

```python
@router.post(
    "/",
    status_code=status.HTTP_201_CREATED,
    response_model=PostResponse
)
async def create_posts(
    post: PostCreate,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user),
):
```

Endpoint:

```text
POST /api/v1/posts/
```

It requires a valid Bearer token.

The request body is validated against `PostCreate`.

The authenticated user is loaded through:

```python
current_user: User = Depends(get_current_user)
```

The post is constructed here:

```python
new_post = Post(**post.model_dump())
```

`model_dump()` converts the Pydantic object into a dictionary:

```python
{
    "title": "...",
    "content": "...",
    "published": True,
    "rating": 5
}
```

Then SQLAlchemy persists it:

```python
db.add(new_post)
db.commit()
db.refresh(new_post)
return new_post
```

The purpose of `refresh()` is to retrieve database-generated values such as:

- `id`
- `created_at`
- server defaults

Important: `current_user` is authenticated but unused. The post is not linked to that user.

## Get latest post

```python
@router.get("/latest", response_model=PostResponse)
async def get_latest_post(db: Session = Depends(get_db)):
    post = db.query(Post).order_by(Post.id.desc()).first()
```

Endpoint:

```text
GET /api/v1/posts/latest
```

This sorts posts by descending ID and returns the first record.

If there are no posts:

```python
raise HTTPException(
    status_code=status.HTTP_404_NOT_FOUND,
    detail="No posts found"
)
```

returns:

```json
{
  "detail": "No posts found"
}
```

## Get one post

```python
@router.get("/{id}", response_model=PostResponse)
async def get_post(
    id: int,
    response: Response,
    db: Session = Depends(get_db)
):
```

Endpoint:

```text
GET /api/v1/posts/1
```

The path parameter is automatically converted to an integer.

The query is:

```python
post = db.query(Post).filter(Post.id == id).first()
```

If the post does not exist, it returns HTTP 404.

The `response: Response` parameter is currently unused.

## Missing operations

The README claims that these endpoints exist:

```text
PUT /api/v1/posts/{id}
DELETE /api/v1/posts/{id}
```

However, they are not implemented in the current router.

The current post endpoints are only:

| Method | Route | Authentication |
|---|---|---|
| `GET` | `/api/v1/posts/` | Public |
| `POST` | `/api/v1/posts/` | Required |
| `GET` | `/api/v1/posts/latest` | Public |
| `GET` | `/api/v1/posts/{id}` | Public |

---

# 9. [api/v1/auth/user_schemas.py](api/v1/auth/user_schemas.py)

This file defines authentication-related Pydantic schemas.

## `UserCreate`

```python
class UserCreate(BaseModel):
    email: EmailStr
    password: str
```

`EmailStr` validates that the input has a valid email format.

Example:

```json
{
  "email": "user@example.com",
  "password": "mypassword"
}
```

## `UserResponse`

```python
class UserResponse(BaseModel):
    id: int
    email: EmailStr
    created_at: datetime

    model_config = ConfigDict(from_attributes=True)
```

This is the public user response.

Notice that it does not include `password`.

That prevents the password hash from being returned to clients.

## `Token`

```python
class Token(BaseModel):
    access_token: str
    token_type: str
```

The login endpoint returns this shape:

```json
{
  "access_token": "JWT_TOKEN_HERE",
  "token_type": "bearer"
}
```

## `TokenData`

```python
class TokenData(BaseModel):
    id: Optional[int] = None
```

This stores the user ID extracted from the JWT.

---

# 10. [api/v1/auth/security.py](api/v1/auth/security.py)

This file contains password hashing, JWT creation, and current-user authentication.

## Configuration

```python
SECRET_KEY = os.getenv(
    "SECRET_KEY",
    "fallback-secret-key"
)
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 60
```

The JWT uses:

- HS256 signing
- A 60-minute expiration
- A secret key from the environment

The fallback secret is a security risk in production. A production deployment should require `SECRET_KEY` to be explicitly configured.

## OAuth2 bearer scheme

```python
oauth2_scheme = OAuth2PasswordBearer(
    tokenUrl="/api/v1/auth/login"
)
```

This tells FastAPI that protected routes expect:

```text
Authorization: Bearer <token>
```

It also informs Swagger UI where users should log in.

## Password hashing

```python
def hash_password(password: str) -> str:
    pwd_bytes = password.encode("utf-8")
    salt = bcrypt.gensalt()
    return bcrypt.hashpw(
        pwd_bytes,
        salt
    ).decode("utf-8")
```

The process is:

```text
Plain password
    ↓
Convert to bytes
    ↓
Generate random bcrypt salt
    ↓
Hash password with bcrypt
    ↓
Store resulting hash
```

A bcrypt hash contains the salt and configuration information, so the salt does not need to be stored separately.

## Password verification

```python
def verify_password(
    plain_password: str,
    hashed_password: str
) -> bool:
    return bcrypt.checkpw(
        plain_password.encode("utf-8"),
        hashed_password.encode("utf-8")
    )
```

The original password is never decrypted. Instead, bcrypt hashes the supplied password and compares it to the stored hash.

## JWT creation

```python
def create_access_token(
    data: dict,
    expires_delta: Optional[timedelta] = None
) -> str:
    to_encode = data.copy()
    expire = datetime.now(timezone.utc) + (
        expires_delta
        or timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    )
    to_encode.update({"exp": expire})
    return jwt.encode(
        to_encode,
        SECRET_KEY,
        algorithm=ALGORITHM
    )
```

A login token contains:

```json
{
  "user_id": 1,
  "exp": "expiration timestamp"
}
```

The token is signed, so the server can detect whether it has been modified.

## Retrieving the current user

```python
def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: Session = Depends(get_db)
) -> User:
```

This function is used as a dependency on protected routes.

The authentication flow is:

```text
Read Authorization header
    ↓
Extract Bearer token
    ↓
Decode JWT signature
    ↓
Read user_id
    ↓
Query user from database
    ↓
Return User object
```

Token decoding:

```python
payload = jwt.decode(
    token,
    SECRET_KEY,
    algorithms=[ALGORITHM]
)
```

The expiration field is checked by the JWT library.

If the token is invalid:

```python
except JWTError:
    raise credentials_exception
```

The client receives:

```http
401 Unauthorized
```

with:

```json
{
  "detail": "Could not validate credentials"
}
```

---

# 11. [api/v1/auth/auth_router.py](api/v1/auth/auth_router.py)

This file contains registration and login endpoints.

## Router

```python
router = APIRouter(
    prefix="/auth",
    tags=["Authentication & Users"]
)
```

The final prefix is:

```text
/api/v1/auth
```

## Register user

```python
@router.post(
    "/register",
    status_code=status.HTTP_201_CREATED,
    response_model=UserResponse
)
def register_user(
    user_in: UserCreate,
    db: Session = Depends(get_db)
):
```

Endpoint:

```text
POST /api/v1/auth/register
```

Example request:

```json
{
  "email": "alice@example.com",
  "password": "strong-password"
}
```

The application checks whether the email already exists:

```python
existing_user = db.query(User).filter(
    User.email == user_in.email
).first()
```

If it exists:

```python
raise HTTPException(
    status_code=status.HTTP_400_BAD_REQUEST,
    detail="Email is already registered."
)
```

The password is hashed:

```python
hashed_pwd = hash_password(user_in.password)
```

The database record is created:

```python
new_user = User(
    email=user_in.email,
    password=hashed_pwd
)
db.add(new_user)
db.commit()
db.refresh(new_user)
return new_user
```

Because the response model is `UserResponse`, the password field is not returned.

## Login

```python
@router.post("/login", response_model=Token)
def login(
    user_credentials: OAuth2PasswordRequestForm = Depends(),
    db: Session = Depends(get_db)
):
```

Endpoint:

```text
POST /api/v1/auth/login
```

Unlike registration, login expects form-encoded data rather than JSON.

Example:

```text
username=alice@example.com
password=strong-password
```

Although the field is called `username`, the application treats it as an email:

```python
user = db.query(User).filter(
    User.email == user_credentials.username
).first()
```

The password is verified:

```python
if not user or not verify_password(
    user_credentials.password,
    user.password
):
```

If invalid:

```http
403 Forbidden
```

If valid, a token is created:

```python
access_token = create_access_token(
    data={"user_id": user.id}
)
```

The result is:

```json
{
  "access_token": "...",
  "token_type": "bearer"
}
```

---

# 12. Authentication Request Flow

## Registration

```text
Client
  ↓
POST /api/v1/auth/register
  ↓
Validate email and password with UserCreate
  ↓
Check whether email already exists
  ↓
Hash password with bcrypt
  ↓
Insert User into PostgreSQL
  ↓
Return UserResponse
```

## Login

```text
Client
  ↓
POST /api/v1/auth/login
  ↓
Read form username and password
  ↓
Find user by email
  ↓
Verify bcrypt password
  ↓
Create signed JWT
  ↓
Return access token
```

## Protected post creation

```text
Client sends:
Authorization: Bearer JWT_TOKEN
  ↓
OAuth2PasswordBearer extracts token
  ↓
get_current_user decodes JWT
  ↓
User ID is read from token
  ↓
User is loaded from PostgreSQL
  ↓
Post request is accepted
```

---

# 13. [Dockerfile](Dockerfile)

The Dockerfile creates the API container.

```dockerfile
FROM python:3.11-slim
```

The application uses Python 3.11 on a lightweight Linux image.

```dockerfile
WORKDIR /app
```

All following commands execute from `/app`.

```dockerfile
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
```

Dependencies are installed before the application source is copied.

This improves Docker layer caching because dependency installation does not need to repeat when only source files change.

```dockerfile
COPY . .
```

Copies the project into the container.

```dockerfile
EXPOSE 8000
```

Documents that the application listens on port 8000.

```dockerfile
CMD [
  "uvicorn",
  "main:app",
  "--host",
  "0.0.0.0",
  "--port",
  "8000"
]
```

This starts the FastAPI application.

The `main:app` notation means:

```text
main.py → variable named app
```

---

# 14. [docker-compose.yml](docker-compose.yml)

This file defines two services.

## API service

```yaml
api:
  build: .
  container_name: fastapi_app
  ports:
    - "8000:8000"
```

The API container is built using the Dockerfile.

Port mapping:

```text
Host port 8000 → Container port 8000
```

## Environment configuration

```yaml
env_file:
  - .env
environment:
  - DB_HOST=db
```

The container loads variables from `.env`.

The important Docker-specific override is:

```text
DB_HOST=db
```

Inside Docker Compose, `db` is the hostname of the PostgreSQL service.

The application must not use `localhost` to reach PostgreSQL from inside the API container. `localhost` would refer to the API container itself.

## Dependency order

```yaml
depends_on:
  - db
```

This starts the database service before the API service.

However, it does not necessarily wait until PostgreSQL is ready to accept connections. The retry loop in `database.py` compensates for this.

## Volume

```yaml
volumes:
  - .:/app
```

The local project directory is mounted into the container.

This is useful during development because local source changes are visible inside the container.

## PostgreSQL service

```yaml
db:
  image: postgres:15
```

The database uses PostgreSQL 15.

```yaml
environment:
  POSTGRES_USER: ${DB_USER:-postgres}
  POSTGRES_PASSWORD: ${DB_PASSWORD:-postgres}
  POSTGRES_DB: ${DB_NAME:-fastapi}
```

The `${VARIABLE:-default}` syntax means:

- Use the environment variable if defined.
- Otherwise use the fallback.

## Database persistence

```yaml
volumes:
  - postgres_data:/var/lib/postgresql/data
```

The named volume keeps database data even if the container is recreated.

---

# 15. [requirements.txt](requirements.txt)

Important dependencies include:

```text
fastapi[standard]
uvicorn[standard]
```

These provide the FastAPI framework and ASGI server.

```text
pydantic
```

Used for validation and serialization.

```text
psycopg2-binary
```

PostgreSQL driver used by the explicit connection test in `database.py`.

```text
python-dotenv
```

Loads `.env` configuration values.

```text
python-multipart
```

Required for `OAuth2PasswordRequestForm`.

```text
SQLAlchemy==2.0.32
```

ORM and database abstraction layer.

```text
python-jose[cryptography]
```

Used to encode and decode JWT tokens.

```text
bcrypt==4.3.0
```

Used for secure password hashing.

`pandas` is installed but is not used by the current application code.

---

# 16. [onlyfastapi.py](onlyfastapi.py)

This is an earlier, simpler version of the application.

It stores posts in memory:

```python
mypost = [
    {
        "title": "Gladiator",
        "content": "thala ajith",
        "id": 1
    },
    {
        "title": "GBU",
        "content": "Ak-arakan",
        "id": 2
    }
]
```

There is no PostgreSQL or SQLAlchemy usage for actual CRUD operations.

## Post schema

```python
class Post(BaseModel):
    title: str
    content: str
    published: bool = True
    rating: Optional[int] = None
```

This validates incoming post data.

## Find helper

```python
def find_post(id):
    for p in mypost:
        if p["id"] == id:
            return p
```

This performs a linear search through the in-memory list.

## Create endpoint

```python
@app.post("/posts")
async def create_posts(post: Post):
    post_dict = post.dict()
    post_dict["id"] = randrange(0, 1000000)
    mypost.append(post_dict)
    return {"data": post_dict}
```

The post is added to the Python list.

This data disappears when the process restarts.

## Difference from the main application

`onlyfastapi.py` is a learning or prototype implementation.

The production-style application in `main.py` uses:

- PostgreSQL
- SQLAlchemy
- Authentication
- Modular routers
- Response models
- Database persistence

`onlyfastapi.py` is not imported by `main.py`.

---

# 17. [readme.md](readme.md)

The README documents:

- How to create a virtual environment
- How to install dependencies
- How to start Docker Compose
- The intended API endpoints
- The authentication design
- The project architecture

Example startup commands:

```powershell
py -3 -m venv venv
.\\venv\\Scripts\\Activate.ps1
pip install -r requirements.txt
docker compose up --build -d
```

The README correctly describes concepts such as:

- FastAPI dependency injection
- Pydantic schemas
- SQLAlchemy models
- JWT authentication
- bcrypt password hashing
- Database-generated timestamps

However, it advertises endpoints not currently implemented:

```text
PUT /api/v1/posts/{id}
DELETE /api/v1/posts/{id}
```

The README should eventually be updated or the missing routes should be implemented.

---

# 18. [.env.example](.env.example)

This provides a template for local configuration:

```text
DB_HOST=localhost
DB_NAME=fastapi
DB_USER=postgres
DB_PASSWORD=your_password_here
DB_PORT=5432
```

For local execution outside Docker, `DB_HOST=localhost` is correct.

For Docker Compose, the application container overrides the host to:

```text
DB_HOST=db
```

The actual `.env` file contains sensitive credentials and should not be committed.

---

# 19. [.env](.env)

This contains the real local database configuration.

It is correctly excluded by [.gitignore](.gitignore) and [.dockerignore](.dockerignore).

The password should not be exposed in reports, source code, logs, or commits.

The JWT `SECRET_KEY` should also be placed in `.env` rather than relying on the fallback value in `security.py`.

---

# 20. [.gitignore](.gitignore)

This prevents generated and sensitive files from being committed.

Important exclusions:

```text
venv/
.venv/
__pycache__/
.env
.env.*
```

This is appropriate because virtual environments, Python caches, and secrets should not be committed.

---

# 21. [.dockerignore](.dockerignore)

This prevents unnecessary files from being copied into the Docker image.

It excludes:

```text
venv/
.venv/
__pycache__/
*.pyc
.env
.git/
```

Excluding `.env` is especially important because Docker should not bake local secrets into the image.

The `.env` file is still provided at runtime through Docker Compose:

```yaml
env_file:
  - .env
```

---

# 22. Empty `__init__.py` Files

These files are empty:

- [db/__init__.py](db/__init__.py)
- [api/v1/auth/__init__.py](api/v1/auth/__init__.py)
- [api/v1/posts/__init__.py](api/v1/posts/__init__.py)

Their purpose is to mark directories as Python packages and allow imports such as:

```python
from db.database import engine
from api.v1.posts.router import router
```

The `api/v1` directory itself does not currently contain an `__init__.py`, but modern Python supports namespace packages, so these imports can still work.

---

# 23. Current API Endpoint Map

## General

| Method | Route | Purpose |
|---|---|---|
| `GET` | `/` | Server health message |

## Authentication

| Method | Route | Body | Authentication |
|---|---|---|---|
| `POST` | `/api/v1/auth/register` | JSON | Public |
| `POST` | `/api/v1/auth/login` | Form data | Public |

## Posts

| Method | Route | Body | Authentication |
|---|---|---|---|
| `GET` | `/api/v1/posts/` | None | Public |
| `POST` | `/api/v1/posts/` | JSON | Required |
| `GET` | `/api/v1/posts/latest` | None | Public |
| `GET` | `/api/v1/posts/{id}` | None | Public |

## Documented but missing

| Method | Route | Current status |
|---|---|---|
| `PUT` | `/api/v1/posts/{id}` | Not implemented |
| `DELETE` | `/api/v1/posts/{id}` | Not implemented |

---

# 24. Example Complete User Journey

## Step 1: Start the application

```text
docker compose up --build -d
```

Docker starts:

```text
PostgreSQL container
    ↓
FastAPI container
```

The API waits until PostgreSQL becomes available.

## Step 2: Register

```http
POST /api/v1/auth/register
Content-Type: application/json
```

```json
{
  "email": "alice@example.com",
  "password": "secret-password"
}
```

The application:

1. Validates the email.
2. Checks for duplicates.
3. Hashes the password.
4. Inserts the user.
5. Returns the user without the password.

## Step 3: Login

```http
POST /api/v1/auth/login
Content-Type: application/x-www-form-urlencoded
```

```text
username=alice@example.com&password=secret-password
```

The application returns a JWT.

## Step 4: Create a post

```http
POST /api/v1/posts/
Authorization: Bearer JWT_TOKEN
Content-Type: application/json
```

```json
{
  "title": "My first post",
  "content": "This is my first post.",
  "published": true,
  "rating": 5
}
```

The JWT is validated before the post is inserted.

## Step 5: Read posts

```http
GET /api/v1/posts/
```

Currently, this endpoint is public.

---

# 25. Important Implementation Findings

## 1. README and implementation disagree

The README claims full CRUD, but only create and read operations currently exist.

Missing:

```text
PUT /api/v1/posts/{id}
DELETE /api/v1/posts/{id}
```

## 2. Posts are not connected to users

Authentication confirms that a user exists, but no user ID is saved on the post.

A future model would likely need:

```python
user_id = Column(
    Integer,
    ForeignKey("users.id"),
    nullable=False
)
```

This would allow ownership checks.

## 3. The JWT fallback secret is unsafe

The security module contains a fallback secret.

Production systems should fail startup if `SECRET_KEY` is missing rather than using a known default.

## 4. Database startup retry is infinite

This is helpful when PostgreSQL is slow to start, but configuration errors can cause an endless retry loop.

## 5. A synchronous SQLAlchemy session is used inside `async def` routes

The route functions are declared as asynchronous:

```python
async def read_posts(...)
```

But the database calls are synchronous:

```python
db.query(Post).all()
```

This can block the event loop under load. A consistent design would use either:

- Normal synchronous route functions with synchronous SQLAlchemy, or
- SQLAlchemy’s async engine and `AsyncSession`.

## 6. The explicit psycopg2 connection is not closed

The startup connection check creates:

```python
connection
cursor
```

but does not close them after success.

It would be cleaner to use a context manager or remove the separate check and let SQLAlchemy perform connection validation.

## 7. No migrations

`Base.metadata.create_all()` creates missing tables but does not safely manage schema changes.

For a growing project, Alembic migrations would be more appropriate.

## 8. No automated tests are present

There are currently no visible test files.

Important test areas would include:

- User registration
- Duplicate email handling
- Invalid login
- Successful login
- Expired JWT
- Protected route without a token
- Post creation
- Missing post lookup
- Database session cleanup

---

# 26. Overall Architecture Summary

The project currently follows this architecture:

```text
Client
  │
  ├── GET /
  │
  ├── Authentication routes
  │     ├── POST /api/v1/auth/register
  │     └── POST /api/v1/auth/login
  │
  └── Post routes
        ├── GET /api/v1/posts/
        ├── POST /api/v1/posts/
        ├── GET /api/v1/posts/latest
        └── GET /api/v1/posts/{id}
                    │
                    ▼
             FastAPI dependency injection
                    │
                    ├── Pydantic validation
                    ├── JWT validation
                    └── SQLAlchemy session
                              │
                              ▼
                         PostgreSQL
```

The application is already split into useful modules:

```text
main.py
    Application entry point

api/
    HTTP routes, request schemas, authentication

db/
    Database connection and ORM models

Docker files
    Runtime and infrastructure configuration
```

The main next architectural step would be connecting posts to users and implementing ownership-aware update and delete operations.
