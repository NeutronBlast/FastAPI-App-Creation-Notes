# Fast API App Template


## What is this?

Just my way to structure Fast API applications, making this document just to remember how to do it next time.

## Create project

On Pycharm you just click on New project and select Fast API

## Folder structure

```
FastAPI-Project/
|-- config/
|   |-- db/
|   |   |-- creates.sql
|   |   |-- drops.sql
|   
|   |   |-- inserts/
|   |   |   |-- inserts.sql
|
|   |-- __init__.py
|   |-- db.py
|
|-- middlewares/
|   |-- __init__.py
|   |-- session.py
|   |-- auth.py
|   |-- cors.py
|   |-- secret.py
|   
|-- models/
|   |-- __init__.py
|  
|-- repositories/
|   |-- __init__.py
|   |-- some_repository.py
|   
|-- routes/
|   |-- __init__.py
|   |-- some_routes.py
|   
|-- schemas/
|   |-- __init__.py
|
|-- tests/   
|   |-- __init__.py
|
|   |-- fixtures/
|   |   |-- __init__.py
|   |   |-- api_fixtures.py
|   |   |-- db_fixtures.py
|
|   |-- integration/
|   |   |-- __init__.py
|
|   |-- unit/
|   |   |-- __init__.py
|
|-- .gitattributes
|-- .gitignore
|-- main.py
|-- test_main.http
|-- README.md
|-- requirements.txt
```

## IDEs
I use Pycharm to develop in PyCharm so all the instructions will be based on PyCharm

## Installation
1. On `main.py` the code should look like this:
```python
from fastapi import FastAPI
from routes.some_routes import router as router_one

app = FastAPI()

app.include_router(router_one)
```

2. This is an example of an `auth.py` middleware
```python
def get_current_user(request: Request, session: dict = Depends(get_session)) -> dict:
    """
    Retrieve the current user based on the JWT token provided either in the session (cookie)
    or in the Authorization header of the request.

    This function prioritizes the Authorization header if a token is provided there. If not,
    it falls back to checking for the token in the session. It then decodes and validates
    the token to ensure its authenticity and expiration.

    :param request: The incoming request from which the headers are extracted.
    :type request: Request

    :param session: The session dictionary associated with the request, containing user-related data.
                    This is typically where the token would reside if the user authenticated via a web frontend.
    :type session: dict

    :return: Decoded JWT token payload containing user information.
    :rtype: dict

    :raises HTTPException: If the token is missing or if the token is invalid.
    """
    token = request.headers.get("Authorization", None)
    if token and token.startswith("Bearer "):
        token = token.split("Bearer ")[1]
        token = token.replace('"', '')

    actual_token = session.get("app_token") if session and "app_token" in session else token

    if not actual_token:
        raise HTTPException(status_code=400, detail="Token is missing.")
    payload = decode_token(actual_token)
    if not payload:
        raise HTTPException(status_code=401, detail="Invalid or expired token.")

    return payload
```

3. This is an example of a `cors.py` middleware
```python
import os

from starlette.middleware.base import BaseHTTPMiddleware
from fastapi import Request
from starlette.responses import Response


class CustomCORSMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        response: Response = await call_next(request)

        origin = request.headers.get("origin")

        if origin and origin.startswith(os.getenv("APP_URL")):
            response.headers["Access-Control-Allow-Origin"] = origin
            response.headers["Access-Control-Allow-Methods"] = "*"
            response.headers["Access-Control-Allow-Headers"] = "*"
            response.headers["Access-Control-Allow-Credentials"] = "true"

        return response

```

4. This is an example of a `secret.py` middleware in case you need an API key to make requests to certain routes
```python
import os

from fastapi import Header, HTTPException


def verify_api_key(x_api_key: str = Header(...)) -> str:
    """
    Verifies that the provided API key matches the secret API key stored in the environment variables.

    Args:
        x_api_key (str): The API key passed in the request header.

    Raises:
        HTTPException: A 403 error if the API key does not match the secret API key.

    Returns:
        str: The verified API key if it matches the secret API key.
    """
    if x_api_key != os.getenv("API_KEY"):
        raise HTTPException(status_code=403, detail="Not authorized")
    return x_api_key

```

5. This is an example of a `session.py` middleware
```python
from fastapi import Request


# Dependency to get the session
def get_session(request: Request):
    """
    Retrieves the session from the incoming HTTP request.

    :param request: The incoming HTTP request object.
    :type request: Request (from starlette.requests)

    :return: The session associated with the request.
    :rtype: dict
    """
    return request.session
```

6. This is an example of a router in `router_one.py`
```python
from fastapi import APIRouter

from repositories.some_repository import get_something

router = APIRouter()


@router.get("/something")
async def get_something_by_id(id):
    resource = get_something(id)
    return {"resource": resource}
```

## OAuth Login Base
OAuth follows the same logic no matter which service you use so I will make a guide of how I implemented
it, this case uses discord and a database table called "users" that has "id" as primary key.

With my approach I don't store tokens in the database, I don't think the extra effort of managing tokens
and expiry time is worth it and having permanent tokens doesn't seem like a good idea, besides I don't
think many OAuth services allow this if any. I use sessions to store the tokens.

1. Install dependencies
    ```shell
    pip install PyJWT
    pip install pytest
    pip install itsdangerous httpx
    ```
2. In [this repository](https://github.com/NeutronBlast/RTrader-API) I have a login implementation with OAuth and Discord. Check `user_routes` and `user_repository`

You can check the API documentation in `localhost:8000/docs`

## Libraries
1. [PyJWT](https://pypi.org/project/PyJWT/)
2. [Starlette](https://pypi.org/project/starlette/)
3. [itsdangerous](https://pypi.org/project/itsdangerous/)
4. [httpx](https://www.python-httpx.org)
