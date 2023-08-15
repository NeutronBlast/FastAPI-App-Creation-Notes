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

You can check the API documentation in `localhost:8000/docs`

## Libraries
1. [Nebular](https://akveo.github.io/nebular/)
