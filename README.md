# Fast API App Template


## What is this?

Just my way to structure Fast API applications, making this document just to remember how to do it next time.

## Folder structure (Nebular)

```
FastAPI-Project/
|-- config/
|   |-- __init__.py
|   |-- db.py
|
|-- middlewares/
|   |-- __init__.py
|   |-- session.py
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
1. Run `pip install`
2. On `main.py` the code should look like this:
```python
from fastapi import FastAPI
from routes.some_routes import router as router_one

app = FastAPI()

app.include_router(router_one)
```

3. This is an example of a router in `router_one.py`
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
2. We are going to use a SessionMiddleware from Starlette on our `main.py` script. 
This is in order to allow the `session` object to be tied to a client-side cookie
    ```python
    import os

    from dotenv import load_dotenv
    from fastapi import FastAPI
    from starlette.middleware.sessions import SessionMiddleware
    
    from routes.user_routes import router as auth_routes
    
    app = FastAPI()
    
    # Add middleware for session handling
    app.add_middleware(SessionMiddleware, secret_key=os.getenv("SECRET_KEY"))
    
    app.include_router(auth_routes)
    ```
3. Now, we are going to set the get_session dependency function in `middlewares/session.py`
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
4. These are our login and logout routes, includes the `create_token` function for JWT auth
    ```python
    import os
    
    from fastapi import HTTPException, APIRouter, Depends
    import requests
    import jwt
    import datetime
    from sqlalchemy.orm import Session
    
    from config.db import get_db
    from middlewares.session import get_session
    from repositories.user_repository import UserRepository
    
    router = APIRouter()
    
    
    def create_token(user_id):
        """
        Generates a JWT token for the given user ID.
    
        :param user_id: The user's ID for whom the token is being generated.
        :type user_id: str or int
    
        :return: A JWT token representing the user.
        :rtype: str
        """
        # Define the payload with user information and expiration time
        payload = {
            "user_id": user_id,
            "exp": datetime.datetime.utcnow() + datetime.timedelta(days=1)
        }
    
        # Secret key to sign the token (should be kept secure and private)
        secret_key = os.getenv("SECRET_KEY")
    
        # Generate the JWT token with HS256 algorithm
        token = jwt.encode(payload, secret_key, algorithm="HS256")
    
        return token
    
    
    @router.get("/login/callback")
    async def login_callback(code: str, session: dict = Depends(get_session), db: Session = Depends(get_db)):
        """
        Handles the OAuth2 callback from Discord, authenticates the user, and associates the obtained token
        with the user in the database.
    
        :param code: The authorization code provided by Discord as part of the OAuth2 flow.
        :type code: str
        :param session: The session dictionary associated with the request.
        :type session: dict
        :param db: The database session used to interact with the application's database.
        :type db: Session (from SQLAlchemy or your chosen ORM)
    
        :return: A JSON object containing the application token for the authenticated user.
        :rtype: dict
    
        :raises HTTPException: If the code is missing or if authentication with Discord fails.
        """
        if not code:
            raise HTTPException(status_code=400, detail="Code is required")
    
        data = {
            "client_id": os.getenv("CLIENT_ID"),
            "client_secret": os.getenv("CLIENT_SECRET"),
            "grant_type": "authorization_code",
            "code": code,
            "redirect_uri": os.getenv("API_URL") + "login/callback",
        }
        headers = {"Content-Type": "application/x-www-form-urlencoded"}
    
        try:
            response = requests.post("https://discord.com/api/oauth2/token", data=data, headers=headers)
            response.raise_for_status()  # Raises an HTTPError if the HTTP request returned an unsuccessful status code
            token = response.json()
    
            # Check if token contains the required keys
            if 'access_token' not in token:
                raise ValueError("Access token missing in response")
    
            # Retrieve the user's information from Discord
            headers = {"Authorization": f"Bearer {token['access_token']}"}
            user_response = requests.get(os.getenv("DISCORD_API_URL") + "users/@me", headers=headers)
            discord_user = user_response.json()
    
            # Check if the user exists in your database
            user_repository = UserRepository(db)
            user = user_repository.get_user_by_id(discord_user['id'])
    
            if not user:
                user = user_repository.add_user(discord_user['id'])
    
            # Save tokens in the session instead of the database
            session["access_token"] = token["access_token"]
    
            # Create an application token or session
            app_token = create_token(user.id)
    
            return {"token": app_token}
    
        except requests.RequestException as e:
            raise HTTPException(status_code=503, detail="Service unavailable") from e
        except ValueError as e:
            raise HTTPException(status_code=500, detail="Unexpected response format") from e
    
    
    @router.post("/logout")
    async def logout(session: dict = Depends(get_session)):
        """
        Logs the user out by clearing the stored access and refresh tokens in the session.
    
        :param session: The session dictionary associated with the request.
        :type session: dict
    
        :return: A success message indicating that the user has been logged out.
        :rtype: dict
        """
        session.pop("access_token", None)
        session.pop("refresh_token", None)
    
        return {"message": "Successfully logged out"}
    
    ```
5. This is the `UserRepository` class stored in `repostories/user_repository.py`
    ```python
    from sqlalchemy.orm import Session
    
    from models.user import User
    
    
    class UserRepository:
        """
        Repository class for managing user-related database operations.
    
        Attributes:
            db (Session): The database session used to interact with the database.
        """
    
        def __init__(self, db: Session):
            """
            Initializes the UserRepository with a given database session.
    
            Args:
                db (Session): The database session.
            """
            self.db = db
    
        def get_user_by_id(self, discord_id: str) -> User:
            """
            Retrieves a user by its discord id
    
            Args:
                discord_id (str): The unique identifier of the user.
    
            Returns:
                User: The user object if found, otherwise None.
            """
            return self.db.query(User).filter(User.discord_id == discord_id).first()
    
        def add_user(self, discord_id: str) -> User:
            """
            Adds a new user to the database.
    
            Args:
                discord_id (str): The unique identifier of the user.
            """
            user = User(discord_id=discord_id)  # Fill in other fields as needed
            self.db.add(user)
            self.db.commit()
            return user
    ```
6. Finally, this is the model used for this example
    ```python
    from sqlalchemy import Column, BigInteger, String
    
    from sqlalchemy.ext.declarative import declarative_base
    
    Base = declarative_base()
    
    
    class User(Base):
        """
        Represents a user in the application, with a unique identifier and an associated Discord ID.
    
        Attributes:
            id (int): The unique identifier for the user. This is the primary key in the database and is auto-incremented.
            discord_id (str): The unique Discord ID for the user, represented as a string. This value must be unique
            across all users and cannot be null.
    
        Example:
            user = User(discord_id="190270795174510593")
        """
    
        __tablename__ = 'users'
    
        id = Column(BigInteger, primary_key=True, autoincrement=True)
        discord_id = Column(String(100), unique=True, nullable=False)
    
    ```
7. In case you want to test these functions. I made some integration tests in `tests/integration/test_auth.py`
    ```python
    import requests
    from fastapi.testclient import TestClient
    import pytest
    from unittest.mock import patch
    
    from tests.fixtures.api_fixtures import api_client
    
    
    # This will mock the requests.post method to mimic the response from Discord
    def mock_post(*args, **kwargs):
        """
        Mocks the requests.post method to simulate the response from Discord's OAuth2 callback.
    
        :param args: Positional arguments passed to the post request
        :param kwargs: Keyword arguments passed to the post request
        :return: MockResponse object containing the JSON data and status code
        :rtype: MockResponse
        """
    
        class MockResponse:
            def __init__(self, json_data, status_code):
                self.json_data = json_data
                self.status_code = status_code
    
            def json(self):
                return self.json_data
    
            # Add this method to simulate raise_for_status
            def raise_for_status(self):
                if self.status_code != 200:
                    raise requests.HTTPError(response=self)
    
        return MockResponse({
            "token_type": "Bearer",
            "access_token": VALID_TOKEN_HERE 
        }, 200)
    
    
    def test_login_callback(api_client):
        """
        Tests the login_callback endpoint to ensure it properly handles the OAuth2 callback from Discord.
    
        This test includes checks for:
        - Successful authentication with a valid authorization code
        - Handling of a missing authorization code
        - (Optionally) Handling of an error response from Discord
    
        :param api_client: A TestClient instance used to send HTTP requests to the FastAPI application.
        :type api_client: TestClient
        """
        with patch('requests.post', mock_post):
            # Test with valid code
            response = api_client.get("/login/callback", params={"code": "valid_code"})
            assert response.status_code == 200
            assert response.json()["token"] is not None
    
            # Test with missing code
            response = api_client.get("/login/callback", params={"code": ""})
            assert response.status_code == 400
    
    
    def mock_login(api_client):
        """
        Mocks the login process by calling the login_callback endpoint with a valid code.
        Uses the mock_post function to simulate the response from Discord's OAuth2 callback.
    
        :param api_client: A TestClient instance used to send HTTP requests to the FastAPI application.
        :type api_client: TestClient
    
        :return: Cookies containing the session data, including access and refresh tokens.
        :rtype: dict
        """
        with patch('requests.post', mock_post):
            response = api_client.get("/login/callback", params={"code": "valid_code"})
            assert response.status_code == 200
            assert response.json()["token"] is not None
            return response.cookies
    
    
    def test_logout(api_client):
        """
        Tests the logout endpoint to ensure it clears the access and refresh tokens from the session.
    
        This test includes checks for:
        - Successful logout
        - Proper clearing of the tokens
    
        :param api_client: A TestClient instance used to send HTTP requests to the FastAPI application.
        :type api_client: TestClient
        """
    
        # First, log in to set the session variables
        cookies = mock_login(api_client)
    
        # Then, call the logout endpoint
        response = api_client.post("/logout", cookies=cookies)
        assert response.status_code == 200
        assert response.json()["message"] == "Successfully logged out"
    
        # Verify that the session tokens have been cleared
        assert "access_token" not in response.cookies.get("session", {})
        assert "refresh_token" not in response.cookies.get("session", {})
    ```

You can check the API documentation in `localhost:8000/docs`

## Libraries
1. [PyJWT](https://pypi.org/project/PyJWT/)
2. [Starlette](https://pypi.org/project/starlette/)
3. [itsdangerous](https://pypi.org/project/itsdangerous/)
4. [httpx](https://www.python-httpx.org)
