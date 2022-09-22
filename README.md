# Minimal async FastAPI + PostgreSQL template

- [Feauters](#features)
- [Quickstart](#quickstart)
- [About](#about)
- [Step by step example - POST and GET endpoints](#step-by-step-example---post-and-get-endpoints)
  - [1. Create SQLAlchemy model](#1-create-sqlalchemy-model)
  - [2. Create and apply alembic migration](#2-create-and-apply-alembic-migration)
  - [3. Create request and response schemas](#3-create-request-and-response-schemas)
  - [4. Create endpoint](#4-create-endpoints)
  - [5. Write tests](#5-write-tests)
- [Deployment strategies - via Docker image](#deployment-strategies---via-docker-image)


## Quickstart

`python >3.9`

<br>

```bash
cd project_name
# Poetry install (and activate environment!)
poetry install
# Setup two databases
docker-compose up -d
# Alembic migrations upgrade and initial_data.py script
bash init.sh
# And this is it:
uvicorn app.main:app --reload

# Optionally - use git init to initialize git repository
```

#### Running tests

```bash
# Note, it will use second database declared in docker-compose.yml, not default one
pytest

# collected 7 items

# app/tests/test_auth.py::test_auth_access_token PASSED                                                                       [ 14%]
# app/tests/test_auth.py::test_auth_access_token_fail_no_user PASSED                                                          [ 28%]
# app/tests/test_auth.py::test_auth_refresh_token PASSED                                                                      [ 42%]
# app/tests/test_users.py::test_read_current_user PASSED                                                                      [ 57%]
# app/tests/test_users.py::test_delete_current_user PASSED                                                                    [ 71%]
# app/tests/test_users.py::test_reset_current_user_password PASSED                                                            [ 85%]
# app/tests/test_users.py::test_register_new_user PASSED                                                                      [100%]
#
# ======================================================== 7 passed in 1.75s ========================================================
```

<br>

## Step by step example - POST and GET endpoints

I always enjoy to have some kind of an example in templates (even if I don't like it much, _some_ parts may be useful and save my time...), so let's create two example endpoints:

- `POST` endpoint `/pets/create` for creating `Pets` with relation to currently logged `User`
- `GET` endpoint `/pets/me` for fetching all user's pets.

<br>

### 1. Create SQLAlchemy model

We will add Pet model to `app/models.py`. To keep things clear, below is full result of models.py file.

```python
# app/models.py

import uuid
from dataclasses import dataclass, field

from sqlalchemy import Column, ForeignKey, Integer, String
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import registry, relationship

Base = registry()


@Base.mapped
@dataclass
class User:
    __tablename__ = "user_model"
    __sa_dataclass_metadata_key__ = "sa"

    id: uuid.UUID = field(
        init=False,
        default_factory=uuid.uuid4,
        metadata={"sa": Column(UUID(as_uuid=True), primary_key=True)},
    )
    email: str = field(
        metadata={"sa": Column(String(254), nullable=False, unique=True, index=True)}
    )
    hashed_password: str = field(metadata={"sa": Column(String(128), nullable=False)})


@Base.mapped
@dataclass
class Pet:
    __tablename__ = "pets"
    __sa_dataclass_metadata_key__ = "sa"

    id: int = field(init=False, metadata={"sa": Column(Integer, primary_key=True)})
    user_id: uuid.UUID = field(
        metadata={"sa": Column(ForeignKey("user_model.id", ondelete="CASCADE"))},
    )
    pet_name: str = field(
        metadata={"sa": Column(String(50), nullable=False)},
    )



```

Note, we are using super powerful SQLAlchemy feature here - you can read more about this fairy new syntax based on dataclasses [in this topic in the docs](https://docs.sqlalchemy.org/en/14/orm/declarative_styles.html#example-two-dataclasses-with-declarative-table).

<br>

### 2. Create and apply alembic migration

```bash
### Use below commands in root folder in virtualenv ###

# if you see FAILED: Target database is not up to date.
# first use alembic upgrade head

# Create migration with alembic revision
alembic revision --autogenerate -m "create_pet_model"


# File similar to "2022050949_create_pet_model_44b7b689ea5f.py" should appear in `/alembic/versions` folder


# Apply migration using alembic upgrade
alembic upgrade head

# (...)
# INFO  [alembic.runtime.migration] Running upgrade d1252175c146 -> 44b7b689ea5f, create_pet_model
```

PS. Note, alembic is configured in a way that it work with async setup and also detects specific column changes.

<br>

### 3. Create request and response schemas

Note, I personally lately (after seeing clear benefits at work) prefer less files than a lot of them for things like schemas.

Thats why there are only 2 files: `requests.py` and `responses.py` in `schemas` folder and I would keep it that way even for few dozen of endpoints.

```python
# app/schemas/requests.py

(...)


class PetCreateRequest(BaseRequest):
    pet_name: str

```

```python
# app/schemas/responses.py

(...)


class PetResponse(BaseResponse):
    id: int
    pet_name: str
    user_id: uuid.UUID

```

<br>

### 4. Create endpoints

```python
# /app/api/endpoints/pets.py

from fastapi import APIRouter, Depends
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.api import deps
from app.models import Pet, User
from app.schemas.requests import PetCreateRequest
from app.schemas.responses import PetResponse

router = APIRouter()


@router.post("/create", response_model=PetResponse, status_code=201)
async def create_new_pet(
    new_pet: PetCreateRequest,
    session: AsyncSession = Depends(deps.get_session),
    current_user: User = Depends(deps.get_current_user),
):
    """Creates new pet. Only for logged users."""

    pet = Pet(user_id=current_user.id, pet_name=new_pet.pet_name)

    session.add(pet)
    await session.commit()
    return pet


@router.get("/me", response_model=list[PetResponse], status_code=200)
async def get_all_my_pets(
    session: AsyncSession = Depends(deps.get_session),
    current_user: User = Depends(deps.get_current_user),
):
    """Get list of pets for currently logged user."""

    pets = await session.execute(
        select(Pet)
        .where(
            Pet.user_id == current_user.id,
        )
        .order_by(Pet.pet_name)
    )
    return pets.scalars().all()

```

Also, we need to add newly created endpoints to router.

```python
# /app/api/api.py

from fastapi import APIRouter

from app.api.endpoints import auth, pets, users

api_router = APIRouter()
api_router.include_router(auth.router, prefix="/auth", tags=["auth"])
api_router.include_router(users.router, prefix="/users", tags=["users"])
api_router.include_router(pets.router, prefix="/pets", tags=["pets"])

```

<br>

### 5. Write tests

```python
# /app/tests/test_pets.py

from httpx import AsyncClient
from sqlalchemy.ext.asyncio import AsyncSession

from app.main import app
from app.models import Pet, User


async def test_create_new_pet(
    client: AsyncClient, default_user_headers, default_user: User
):
    response = await client.post(
        app.url_path_for("create_new_pet"),
        headers=default_user_headers,
        json={"pet_name": "Tadeusz"},
    )
    assert response.status_code == 201
    result = response.json()
    assert result["user_id"] == str(default_user.id)
    assert result["pet_name"] == "Tadeusz"


async def test_get_all_my_pets(
    client: AsyncClient, default_user_headers, default_user: User, session: AsyncSession
):

    pet1 = Pet(default_user.id, "Pet_1")
    pet2 = Pet(default_user.id, "Pet_2")
    session.add(pet1)
    session.add(pet2)
    await session.commit()

    response = await client.get(
        app.url_path_for("get_all_my_pets"),
        headers=default_user_headers,
    )
    assert response.status_code == 200

    assert response.json() == [
        {
            "user_id": str(pet1.user_id),
            "pet_name": pet1.pet_name,
            "id": pet1.id,
        },
        {
            "user_id": str(pet2.user_id),
            "pet_name": pet2.pet_name,
            "id": pet2.id,
        },
    ]

```

## Deployment strategies - via Docker image

This template has by default included `Dockerfile` with [Nginx Unit](https://unit.nginx.org/) webserver, that is my prefered choice, because of direct support for FastAPI and great ease of configuration. You should be able to run container(s) (over :80 port) and then need to setup the proxy, loadbalancer, with https enbaled, so the app stays behind it.

`nginx-unit-config.json` file included in main folder has some default configuration options, runs app in single process and thread. More info about config file here https://unit.nginx.org/configuration/#python and about also read howto for FastAPI: https://unit.nginx.org/howto/fastapi/.

If you prefer other webservers for FastAPI, check out [Daphne](https://github.com/django/daphne), [Hypercorn](https://pgjones.gitlab.io/hypercorn/index.html) or [Uvicorn](https://www.uvicorn.org/).