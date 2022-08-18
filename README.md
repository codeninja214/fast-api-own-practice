## WIP: FastAPI Best Practices
Opinionated list of best practices and conventions we have used after 1.5 years in production. 
### 1. Project Structure. Group files by module domain, not file types.
I didn't like the project structure presented by @tiangolo, 
where we separate files by their type (e.g. api, crud, models, schemas).
Structure that I find more scalable and evolvable is inspired by Netflix's [Dispatch](https://github.com/Netflix/dispatch) with some little modifications.
```
fastapi-project
├── alembic/
├── src
│   ├── auth
│   │   ├── router.py
│   │   ├── schemas.py  # pydantic models
│   │   ├── models.py  # db models
│   │   ├── dependencies.py
│   │   ├── config.py  # local configs
│   │   ├── constants.py
│   │   ├── exceptions.py
│   │   ├── service.py
│   │   └── utils.py
│   ├── aws
│   │   ├── client.py  # client model for external service communication
│   │   ├── schemas.py
│   │   ├── config.py
│   │   ├── constants.py
│   │   ├── exceptions.py
│   │   └── utils.py
│   └── posts
│   │   ├── router.py
│   │   ├── schemas.py
│   │   ├── models.py
│   │   ├── dependencies.py
│   │   ├── constants.py
│   │   ├── exceptions.py
│   │   ├── service.py
│   │   └── utils.py
│   ├── config.py  # global configs
│   ├── models.py  # global models
│   ├── exceptions.py  # global exceptions
│   ├── pagination.py  # global module e.g. pagination
│   ├── database.py  # db connection related stuff
│   └── main.py
├── tests/
│   ├── auth
│   ├── aws
│   └── posts
├── templates/
│   └── index.html
├── requirements
│   ├── base.txt
│   ├── dev.txt
│   └── prod.txt
├── .env
├── .gitignore
├── logging.ini
└── alembic.ini
```
1. Store all domain directories inside `src` folder
   1. `src/` - highest level of an app, contains common models, configs, and constants, etc.
   2. `src/main.py` - root of the project, which inits the FastAPI app
2. Each package has its own router, schemas, models, etc.
   1. `router.py` - is a core of each module with all the endpoints
   2. `schemas.py` - for pydantic models
   3. `models.py` - for db models
   4. `service.py` - module specific business logic  
   5. `dependencies.py` - router dependencies
   6. `constants.py` - module specific constants and error codes
   7. `config.py` - e.g. env vars
   8. `utils.py` - non-business logic functions, e.g. response normalization, data enrichment, etc.
   9. `exceptions` - module specific exceptions, e.g. `PostNotFound`, `InvalidUserData`
3. When package requires services or dependencies or constants from other packages - import them with explicit module name
```python
from src.auth import constants as auth_constants
from src.notifictions import service as notification_service
from src.posts.constants import ErrorCode as PostsErrorCode  # in case we have Standard ErrorCode in constants module of each package
```

### 2. Excessively use Pydantic for data validation
Pydantic has a rich set of features to validate and transform data. 

In addition to regular features like required, non-required fields and default data, 
it has built-in comprehensive data processing params like regex, enums for limited allowed options, length validation, email validation, etc.
```python3
from enum import Enum
from pydantic import AnyUrl, BaseModel, EmailStr, Field, constr

class MusicBand(str, Enum):
   AEROSMITH = "AEROSMITH"
   QUEEN = "QUEEN"
   ACDC = "AC/DC"


class UserBase(BaseModel):
    first_name: str = Field(min_length=1, max_length=128)
    username: constr(regex="^[A-Za-z0-9-_]+$", to_lower=True, strip_whitespace=True)
    email: EmailStr
    age: int = Field(ge=18, default=None)  # must be greater or equal to 18
    favorite_band: MusicBand = None  # only "AEROSMITH", "QUEEN", "AC/DC" values are allowed to be inputted
    website: AnyUrl = None

```
### 3. Use dependencies for data validation vs DB
Pydantic can only validate the values of client input. 
Use dependencies to validate data against database requirements like email already exists, user not found, etc. 
```python3
# dependencies.py
async def valid_post_id(post_id: UUID4) -> Mapping:
    post = await service.get_by_id(post_id)
    if not post:
        raise PostNotFound()

    return post


# router.py
@router.get("/posts/{post_id}", response_model=PostResponse)
async def get_post_by_id(post: Mapping = Depends(valid_post_id)):
    return post


@router.put("/posts/{post_id}", response_model=PostResponse)
async def update_post(
    update_data: PostUpdate,  
    post: Mapping = Depends(valid_post_id), 
):
    updated_post: Mapping = await service.update(id=post["id"], data=update_data)
    return updated_post


@router.get("/posts/{post_id}/reviews", response_model=list[ReviewsResponse])
async def get_post_reviews(post: Mapping = Depends(valid_post_id)):
    post_reviews: list[Mapping] = await reviews_service.get_by_post_id(post["id"])
    return post_reviews
```
If we didn't put data validation to dependency, we would have to add post_id validation
for every endpoint and write the same tests for each of them. 

### 5. Chain dependencies
Dependencies can use other dependencies and avoid code repetition for similar logic.
```python3
# dependencies.py
from fastapi.security import OAuth2PasswordBearer
from jose import JWTError, jwt

async def valid_post_id(post_id: UUID4) -> Mapping:
    post = await service.get_by_id(post_id)
    if not post:
        raise PostNotFound()

    return post


async def parse_jwt_data(
    token: str = Depends(OAuth2PasswordBearer(tokenUrl="/auth/token"))
) -> dict:
    try:
        payload = jwt.decode(token, "JWT_SECRET", algorithms=["HS256"])
    except JWTError:
        raise InvalidCredentials()

    return {"user_id": payload["id"]}


async def valid_owned_post(
    post: Mapping = Depends(valid_post_id), 
    token_data: dict = Depends(parse_jwt_data),
) -> Mapping:
    if post["creator_id"] != token_data["user_id"]:
        raise UserNotOwner()

    return post

# router.py
@router.get("/users/{user_id}/posts/{post_id}", response_model=PostResponse)
async def get_user_post(post: Mapping = Depends(valid_owned_post)):
    """Get post that belong the user."""
    return post

```
### 6. Decouple & Reuse dependencies. Dependency calls are cached.
Dependencies can be reused multiple times, and they won't be recalculated - FastAPI caches their result by default,
e.g. if we have a dependency which calls service `get_post_by_id`, we won't be visiting DB each time we call this dependency - only the first function call.

Knowing this, we can easily decouple dependencies onto multiple smaller functions that operate on a smaller scope and are easier to reuse in other routes.
For example, in the code below we are using `parse_jwt_data` three times:
1. `valid_owned_post`
2. `valid_active_creator`
3. `get_user_post`,

but `parse_jwt_data` is called only once, in the very first call.

```python3
# dependencies.py
from fastapi import BackgroundTasks
from fastapi.security import OAuth2PasswordBearer
from jose import JWTError, jwt

async def valid_post_id(post_id: UUID4) -> Mapping:
    post = await service.get_by_id(post_id)
    if not post:
        raise PostNotFound()

    return post


async def parse_jwt_data(
    token: str = Depends(OAuth2PasswordBearer(tokenUrl="/auth/token"))
) -> dict:
    try:
        payload = jwt.decode(token, "JWT_SECRET", algorithms=["HS256"])
    except JWTError:
        raise InvalidCredentials()

    return {"user_id": payload["id"]}


async def valid_owned_post(
    post: Mapping = Depends(valid_post_id), 
    token_data: dict = Depends(parse_jwt_data),
) -> Mapping:
    if post["creator_id"] != token_data["user_id"]:
        raise UserNotOwner()

    return post


async def valid_active_creator(
    token_data: dict = Depends(parse_jwt_data),
):
    user = await users_service.get_by_id(token_data["user_id"])
    if not user["is_active"]:
        raise UserIsBanned()
    
    return user
        

# router.py
@router.get("/users/{user_id}/posts/{post_id}", response_model=PostResponse)
async def get_user_post(
    worker: BackgroundTasks,
    post: Mapping = Depends(valid_owned_post),
    user: Mapping = Depends(valid_active_creator),
):
    """Get post that belong the active user."""
    worker.add_task(notifications_service.send_email, user["id"])
    return post

```

### 7. Follow the REST
Developing RESTful API makes it easier to reuse dependencies in routes like these:
   1. `GET /courses/:course_id`
   2. `GET /courses/:course_id/chapters/:chapter_id/lessons`
   3. `GET /chapters/:chapter_id`

The only caveat is to use the same variable names in the path:
- If you have two endpoints `GET /profiles/:profile_id` and `GET /creators/:creator_id`
that both validate whether the given profile_id exists,  but `GET /creators/:creator_id`
also checks if the profile is creator, then it's better to rename `creator_id` path variable to `profile_id` and chain those two dependencies.
```python3
# src.profiles.dependencies
async def valid_profile_id(profile_id: UUID4) -> Mapping:
    profile = await service.get_by_id(post_id)
    if not profile:
        raise ProfileNotFound()

    return profile

# src.creators.dependencies
async def valid_creator_id(profile: Mapping = Depends(valid_profile_id)) -> Mapping:
    if not profile["is_creator"]:
       raise ProfileNotCreator()

    return profile

# src.profiles.router.py
@router.get("/profiles/{profile_id}", response_model=ProfileResponse)
async def get_user_profile_by_id(profile: Mapping = Depends(valid_profile_id)):
    """Get profile by id."""
    return profile

# src.creators.router.py
@router.get("/profiles/{profile_id}", response_model=ProfileResponse)
async def get_user_profile_by_id(
     creator_profile: Mapping = Depends(valid_creator_id)
):
    """Get profile by id."""
    return creator_profile

```

Use /me endpoints for users own resources (e.g. `GET /profiles/me`, `GET /users/me/posts`)
   1. No need to validate that user id exists - it's already checked via auth method
   2. No need to check whether the user id belongs to the requester

### 8. Don't make your routes async, if you have only blocking I/O operations
Under the hood, FastAPI can [effectively handle](https://fastapi.tiangolo.com/async/#path-operation-functions) both async and sync I/O operations. 
- FastAPI calls sync routes in the [threadpool](https://en.wikipedia.org/wiki/Thread_pool) 
and blocking I/O operations won't stop [event loop](https://docs.python.org/3/library/asyncio-eventloop.html) 
from executing the tasks. 
- Otherwise, if the route is defined as `async` then it's called regularly via `await` 
and FastAPI trusts you to do only non-blocking I/O operations.

The caveat is if you fail that trust and execute blocking operations within async routes, 
event loop will not be able to run the next tasks until that blocking operation is done.
```python
import asyncio
import time

@router.get("/terrible-ping")
async def terrible_catastrophic_ping():
    time.sleep(10) # I/O blocking operation for 10 seconds
    pong = service.get_pong()  # I/O blocking operation to get pong from DB
    
    return {"pong": pong}

@router.get("/good-ping")
def good_ping():
    time.sleep(10) # I/O blocking operation for 10 seconds, but in another thread
    pong = service.get_pong()  # I/O blocking operation to get pong from DB, but in another thread
    
    return {"pong": pong}

@router.get("/perfect-ping")
async def perfect_ping():
    await asyncio.sleep(10) # non I/O blocking operation
    pong = await service.async_get_pong()  # non I/O blocking db call

    return {"pong": pong}

```
**What happens when we call:**
1. `GET /terrible-ping`
   1. FastAPI server receives a request and starts handling it 
   2. Server's event loop and all the tasks in the queue will be waiting until `time.sleep()` is finished
      1. Server thinks `time.sleep()` is not an I/O task, so it waits until it is finished
      2. Server won't accept any new requests while waiting
   3. Then, event loop and all the tasks in the queue will be waiting until `service.get_pong` is finished
      1. Server thinks `service.get_pong()` is not an I/O task, so it waits until it is finished
      2. Server won't accept any new requests while waiting
   4. Server returns the response. 
      1. After a response, server starts accepting new requests
2. `GET /good-ping`
   1. FastAPI server receives a request and starts handling it
   2. FastAPI sends the whole route `good_ping` to the threadpool, where a worker thread will run the function
   3. While `good_ping` is being executed, event loop selects next tasks from the queue and works on them (e.g. accept new request, call db)
      - Independently of main thread (i.e. our FastAPI app), 
        worker thread will be waiting for `time.sleep` to finish and then for `service.get_pong` to finish
   4. When `good_ping` finishes its work, server returns a response to the client
3. `GET /perfect-ping`
   1. FastAPI server receives a request and starts handling it
   2. FastAPI awaits `asyncio.sleep(10)`
   3. Event loop selects next tasks from the queue and works on them (e.g. accept new request, call db)
   4. When `asyncio.sleep(10)` is done, servers goes to the next lines and awaits `service.async_get_pong`
   5. Event loop selects next tasks from the queue and works on them (e.g. accept new request, call db)
   6. When `service.async_get_pong` is done, server returns a response to the client

The caveat is that operations that are non-blocking awaitables or sent to thread pool must be I/O intensive tasks (e.g. open file, db call, external API call).
- Awaiting CPU intensive tasks (e.g. heavy calculations, data processing, video transcoding) is worthless, since CPU has to work to finish the tasks, 
while I/O operations are external and server does nothing while waiting for that operations to finish, thus it can go to the next tasks.
- Running CPU intensive tasks in other threads also isn't effective, because of [GIL](https://realpython.com/python-gil/). 
In short, GIL allows only one thread to work at a time, which makes it useless for CPU tasks. 
- If you want to optimize CPU intensive tasks you should send them to workers in another process.

**Related StackOverflow questions of confused users**
1. https://stackoverflow.com/questions/62976648/architecture-flask-vs-fastapi/70309597#70309597
   - Here you can also check [my answer](https://stackoverflow.com/a/70309597/6927498)
2. https://stackoverflow.com/questions/65342833/fastapi-uploadfile-is-slow-compared-to-flask
3. https://stackoverflow.com/questions/71516140/fastapi-runs-api-calls-in-serial-instead-of-parallel-fashion

### 9. Custom base model from day 0.
Having a controllable global pydantic base model allows us to customize all the models within the app.
For example, we could have a standard datetime format or add a super method for all subclasses of the base model.
```python
from datetime import datetime
from zoneinfo import ZoneInfo

import orjson
from fastapi.encoders import jsonable_encoder
from pydantic import BaseModel, root_validator


def orjson_dumps(v, *, default):
    # orjson.dumps returns bytes, to match standard json.dumps we need to decode
    return orjson.dumps(v, default=default).decode()


def convert_datetime_to_gmt(dt: datetime) -> str:
    if not dt.tzinfo:
        dt = dt.replace(tzinfo=ZoneInfo("UTC"))

    return dt.strftime("%Y-%m-%dT%H:%M:%S%z")


class ORJSONModel(BaseModel):
    class Config:
        json_loads = orjson.loads
        json_dumps = orjson_dumps
        json_encoders = {datetime: convert_datetime_to_gmt}  # method for customer JSON encoding of datetime fields

    @root_validator()
    def set_null_microseconds(cls, data: dict) -> dict:
       """Drops microseconds in all the datetime field values."""
        datetime_fields = {
            k: v.replace(microsecond=0)
            for k, v in data.items()
            if isinstance(k, datetime)
        }

        return {**data, **datetime_fields}

    def serializable_dict(self, **kwargs):
       """Return a dict which contains only serializable fields."""
        default_dict = super().dict(**kwargs)

        return jsonable_encoder(default_dict)
```
In the example above we have decided to make a global base model which: 
- uses [orjson](https://github.com/ijl/orjson) to serialize data
- drops microseconds to 0 in all date formats
- serializes all datetime fields to standard format with explicit timezone 
### 10. Docs
1. Unless your API is public, hide docs by default. Show it explicitly on the selected envs only.
```python
from fastapi import FastAPI
from starlette.config import Config

config = Config(".env")  # parse .env file for env variables

ENVIRONMENT = config("ENVIRONMENT")  # get current env name
SHOW_DOCS_ENVIRONMENT = ("local", "staging")  # explicit list of allowed envs

app_configs = {"title": "My Cool API"}
if ENVIRONMENT not in SHOW_DOCS_ENVIRONMENT:
   app_configs["openapi_url"] = None  # set url for docs as null

app = FastAPI(**app_configs)
```
2. Help FastAPI to generate an easy-to-understand docs
   1. Set `response_model`, `status_code`, `description`, etc.
   2. If models and statuses vary, use `responses` route attribute to add docs for different responses
```python
from fastapi import APIRouter, status

router = APIRouter()

@router.post(
    "/endpoints",
    response_model=DefaultResponseModel,  # default response pydantic model 
    status_code=status.HTTP_201_CREATED,  # default status code
    description="Description of the well documented endpoint",
    tags=["Endpoint Category"],
    summary="Summary of the Endpoint",
    responses={
        status.HTTP_200_OK: {
            "model": OkResponse, # custom pydantic model for 200 response
            "description": "Ok Response",
        },
        status.HTTP_201_CREATED: {
            "model": CreatedResponse,  # custom pydantic model for 201 response
            "description": "Creates something from user request ",
        },
        status.HTTP_202_ACCEPTED: {
            "model": AcceptedResponse,  # custom pydantic model for 202 response
            "description": "Accepts request and handles it later",
        },
    },
)
async def documented_route():
    pass
```
Will generate docs like this:
![FastAPI Generated Custom Response Docs](images/custom_responses.png "Custom Response Docs")

### 11. Use Starlette's Config object
It's decent enough not to use 3rd party ones.
```python
from starlette.config import Config

config = Config(".env")

DATABASE_URL = config("DATABASE_URL")
IS_GOOD_ENV = config("IS_GOOD_ENV", cast=bool, default=True)
ALLOWED_CORS_ORIGINS = config(
    "CORS_ORIGINS",
    cast=lambda x: x.split(","),  # if no env, it first sets default value, only then it casts it with this lambda
    default="https://mysite.com,https://mysite.org",
)
```
### 12. SQLAlchemy: Set DB keys naming convention from day 0
```python
from sqlalchemy import MetaData

POSTGRES_INDEXES_NAMING_CONVENTION = {
    "ix": "%(column_0_label)s_idx",
    "uq": "%(table_name)s_%(column_0_name)s_key",
    "ck": "%(table_name)s_%(constraint_name)s_check",
    "fk": "%(table_name)s_%(column_0_name)s_fkey",
    "pk": "%(table_name)s_pkey",
}
metadata = MetaData(naming_convention=POSTGRES_INDEXES_NAMING_CONVENTION)
```
### 13. Set DB table naming convention immediately from day 0
### 14. Set UUIDs within the app
Setting them in database makes it harder to write integration tests.
### 15. Set tests client async from day 0
Writing integration tests with DB will most likely lead to messed up event loop errors in the future.
Set the async test client immediately, e.g. [asyn_asgi_testclient](https://github.com/vinissimus/async-asgi-testclient) or [httpx](https://github.com/encode/starlette/issues/652)
```python
import pytest
from async_asgi_testclient import TestClient

from src.main import app  # inited FastAPI app


@pytest.fixture
async def client():
    host, port = "127.0.0.1", "5555"
    scope = {"client": (host, port)}

    async with TestClient(
        app, scope=scope, headers={"X-User-Fingerprint": "Test"}
    ) as client:
        yield client
```
Unless you have sync db connection (excuse me?) or aren't planning to write integration tests.
### 16. Set postgres identity from day 0
### 17. Use BackgroundTasks
They are stable enough for async (delayed) tasks
```python
from fastapi import BackgroundTasks


# router.py
@router.get("/users/{user_id}/posts/{post_id}")
async def get_user_post(
    worker: BackgroundTasks,
):
    """Get post that belong the active user."""
    worker.add_task(notifications_service.send_email, user["id"])
    return {"status": "ok"}
```
### 19. Typing is important
FastAPI, Pydantic, and modern IDEs encourage to take use of type hints.

**Without Type Hints**

<img src="images/type_hintsless.png" width="400" height="auto">

**With Type Hints**

<img src="images/type_hints.png" width="400" height="auto">

### 20. Don't hope your clients will send small BLOBs. Save files in chunk.
```python
import aiofiles
from fastapi import UploadFile

DEFAULT_CHUNK_SIZE = 1024 * 1024 * 50  # 50 megabytes

async def save_video(video_file: UploadFile):
   async with aiofiles.open("/file/path/name.mp4", "wb") as f:
     while chunk := await video_file.read(DEFAULT_CHUNK_SIZE):
         await f.write(chunk)
```
### 21. Be careful with dynamic pydantic fields
If you have a pydantic field that can accept multiple types, be sure validator explicitly knows the difference between those types.
```python
from pydantic import BaseModel


class Article(BaseModel):
   text: str | None
   extra: str | None


class Video(BaseModel):
   video_id: int
   text: str | None
   extra: str | None


   
class Post(BaseModel):
   content: Article | Video

   
post = Post(content={"video_id": 1, "text": "text"})
print(type(post.content))
# OUTPUT: Article
# Because Article is very inclusive and all fields are optional
```
**Solutions:**
1. Not so bad solution. Order field types properly: from the most strict ones to loose ones.
```python
class Post(BaseModel):
   content: Video | Article
```
2. Not so bad solution. Validate input has only valid fields 
```python
from pydantic import BaseModel, root_validator

class Article(BaseModel):
   text: str | None
   extra: str | None
   
   @root_validator(pre=True)  # validate all values before pydantic
   def has_only_article_fields(cls, data: dict):
      """Silly and ugly solution to validate data has only article fields."""
      fields = set(data.keys())
      if fields != {"text", "extra"}:
         raise ValueError("invalid fields")

      return data
       

class Video(BaseModel):
   video_id: int
   text: str | None
   extra: str | None
   
   @root_validator(pre=True)
   def has_only_video_fields(cls, data: dict):
      """Silly and ugly solution to validate data has only article fields."""
      fields = set(data.keys())
      if fields != {"text", "extra", "video_id"}:
         raise ValueError("invalid fields")

      return data

   
class Post(BaseModel):
   content: Article | Video
```
3. Good solution. Use Pydantic's Smart Union (>v1.9)
```python
from pydantic import BaseModel

class Post(BaseModel):
   content: Article | Video

   class Config:
      smart_union = True
```
### 22. SQL-first, Pydantic-second
### 23. Validate file formats
### 24. Validate url source (if users are able to send files)
```python
from pydantic import AnyUrl

ALLOWED_MEDIA_URLS = {"mysite.com", "mysite.org"}

class CompanyMediaUrl(AnyUrl):
    @classmethod
    def validate_host(cls, parts: dict) -> tuple[str, str | None, str, bool]:
        host, tld, host_type, rebuild = super().validate_host(parts)
        if host not in ALLOWED_MEDIA_URLS:
            raise ValueError(
                "Forbidden host url. Upload files only to internal services."
            )

        return host, tld, host_type, rebuild
```
### 25. root_validator to use multiple columns during validation
```python
from pydantic import BaseModel, root_validator


class Profile(BaseModel):
    username: str | None
    first_name: str
    last_name: str

    @root_validator()
    def set_username(cls, data: dict) -> dict:
        if not data.get("username"):
            data["username"] = f'{data["first_name"]}_{data["last_name"]}'
        
        return data
```
### 26. pre if data need to be pre-handled before validation
### 27. you can just raise a ValueError in pydantic schemas, if schemas faces http client 
it will return a nice response
### 28. don't forget that fastapi converts response Model to Dict then to Model then to JSON
it may lead to bugs like model can parse only raw data (e.g. forced data aggregation for raw data)
### 29. if no async lib, and poor documentation, then use starlette's run_in_threadpool or asgiref
### 30. use linters (black, isort, autoflake)
### 31. set logs from day 0
