# FastAPI
## FastAPIとは
[FastAPI](https://github.com/tiangolo/fastapi)はPythonのWebフレームワークです
FastAPIは以下の特徴があります

- Pythonフレームワークの中で最も高速
- 自動でAPIドキュメントを出力できる
- Pythonの型定義を利用してデータを検証できる
- 公式ドキュメントがわかりやすくて充実している

このドキュメントでは、FastAPIを使ってAPIを開発する手順をまとめます
使用するFastAPIのバージョンは **0.88.0** です

このドキュメントで使用したソースコードは以下のリポジトリで確認することができます
https://github.com/youichiro/fastapi-todo-app-demo

## 作るもの
FastAPIでシンプルなTODOアプリのAPIを作ります
以下のようなデータモデルを想定します

```mermaid
erDiagram
    users {
        id int PK
        name string
        created_at datetime
    }
    tasks {
        id int PK
        user_id int FK
        title string
        done boolean
        created_at datetime
        updated_at datetime
    }
```

以下のCRUDを用意します
- create a user
- get a user
- get all users
- create a task
- get all tasks for user


## やること
1. プロジェクトを作成する
1. データベースを用意する
1. データベースと接続する設定ファイルを作成する
1. データモデルを定義する
1. Pydanticスキーマを定義する
1. CRUDの処理を用意する
1. ルーティングを定義する
1. マイグレーションを実行する
1. 動作確認する

## 最終的なディレクトリ構成
```
.
├── app
│   ├── crud.py
│   ├── database.py
│   ├── main.py
│   ├── models.py
│   └── schemas.py
├── db
│   ├── alembic.ini
│   ├── development
│   │   ├── README
│   │   ├── env.py
│   │   ├── script.py.mako
│   │   └── versions
│   │       └── 9965d78849e3_first_migration.py
│   └── init
│       └── 1_create_databases.sql
├── docker-compose.yaml
└── requirements.txt
```

## 1. プロジェクトを作成する
FastAPIに必要なライブラリをインストールします
```sh
pip install fastapi "uvicorn[standard]"
```

サンプルコードで試しにFastAPIを起動してみます
app/main.pyを作成します

```python:app/main.py
from fastapi import FastAPI

app = FastAPI()


@app.get("/")
def read_root():
    return {"Hello": "World"}


@app.get("/items/{item_id}")
def read_item(item_id: int, q: str = None):
    return {"item_id": item_id, "q": q}
```

サーバーを立ち上げる
```sh
uvicorn app.main:app --reload
```

curlでレスポンスを確認します

```sh
$ curl http://127.0.0.1:8000/items/5?q=somequery

{"item_id":5,"q":"somequery"}
```

FastAPIではルーティングの定義に従って、自動でOpenAPIを生成してくれます
http://127.0.0.1:8000/docs を開くと、[Swagger UI](https://swagger.io/tools/swagger-ui/)でOpenAPIドキュメントを表示してくれる

![](https://storage.googleapis.com/zenn-user-upload/69055f7f9341-20230104.png)

また、http://127.0.0.1:8000/redoc を開くと [ReDoc](https://github.com/Redocly/redoc) でもOpenAPIドキュメントを表示してくれます

![](https://storage.googleapis.com/zenn-user-upload/0769a3531e86-20230104.png)

## 2. データベースを用意する
docker composeでPostgreSQLを起動します

まず、データベースを作成するためのSQLファイルを用意します
```sql:db/init/1_create_databases.sql
CREATE DATABASE fastapi_todo_app_development;
```

次に、docker-compose.yamlを作成します
```yaml:docker-compose.yaml
version: '3'

services:
  db:
    image: postgres:15
    ports:
      - 5432:5432
    volumes:
      - db-store:/var/lib/postgresql/data
      - ./db/init:/docker-entrypoint-initdb.d
    environment:
      - POSTGRES_PASSWORD=password
volumes:
  db-store:
```

最後に、dockerコンテナを起動します
```sh
docker compose up -d
```

## 3. データベースと接続する設定ファイルを作成する
データベースとの接続には[SQLAlchemy](https://www.sqlalchemy.org)を使用するので、ライブラリをインストールします
```sh
pip install sqlalchemy
```

appディレクトリにdatabase.pyを作成します
```python:app/database.py
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

SQLALCHEMY_DATABASE_URL = "postgresql://postgres:password@0.0.0.0:5432/fastapi_todo_app_development"

engine = create_engine(SQLALCHEMY_DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

Base = declarative_base()
```

- `SQLALCHEMY_DATABASE_URL`にはPostgreSQLの情報を記述する

## 4. データモデルを定義する
appディレクトリにmodels.pyを作成します

```python:app/models.py
from datetime import datetime

from sqlalchemy import Boolean, Column, ForeignKey, Integer, String, DateTime
from sqlalchemy.orm import relationship

from .database import Base


class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    name = Column(String, unique=True, index=True)
    created_at = Column(DateTime, default=datetime.now(), nullable=False)

    tasks = relationship("Task", back_populates="user")


class Task(Base):
    __tablename__ = "tasks"

    id = Column(Integer, primary_key=True, index=True)
    title = Column(String, index=True)
    done = Column(Boolean, default=False, index=True)
    created_at = Column(DateTime, default=datetime.now(), nullable=False)
    updated_at = Column(DateTime, default=datetime.now(), onupdate=datetime.now(), nullable=False)
    user_id = Column(Integer, ForeignKey("users.id"))

    user = relationship("User", back_populates="tasks")
```

- 先ほど作成したdatabase.pyの`Base`をimportし、各エンティティのモデルクラスで継承する
- 各カラムの型は[SQLAlchemyの型](https://docs.sqlalchemy.org/en/14/core/type_basics.html)で指定する
- 関連付けは`relationship`で表現する

## 5. Pydanticスキーマを定義する
models.pyで定義したSQLAlchemyモデルに対して、作成時・読み込み時のモデルの属性や型、制約などをPydanticモデルで定義します (Pydanticスキーマ)

appディレクトリにschemas.pyを作成します

```python:app/schemas.py
from datetime import datetime

from pydantic import BaseModel


class TaskBase(BaseModel):
    title: str
    done: bool = False


class TaskCreate(TaskBase):
    pass


class Task(TaskBase):
    id: int
    created_at: datetime
    updated_at: datetime
    user_id: int

    class Config:
        orm_mode = True


class UserBase(BaseModel):
    name: str


class UserCreate(UserBase):
    pass


class User(UserBase):
    id: int
    created_at: datetime
    tasks: list[Task] = []

    class Config:
        orm_mode = True
```

- `TaskBase`、`UserBase`はそのモデルの作成時・読み込み時での共通の属性情報を定義している
- `TaskCreate`、`UserCreate`は、そのモデルの作成時に必要な属性情報を定義している (今回はBaseと差分がないのでpassしている)
- `Task`、`User`は、そのモデルの読み込み時に必要な属性情報を定義している
    - idやcreated_atなどは実際にレコードが作成された後に付与される値なので、作成時は指定しないはず
- `orm_mode = True`を設定することで、Pydanticモデルの属性を `id = data["id"]` だけでなく `id = data.id` でも参照できるようになる


## 6. CRUDの処理を用意する
データベースから特定のレコードを取得・挿入する処理を実行する関数を用意します

appディレクトリにcrud.pyを作成します

```python:app/crud.py
from sqlalchemy.orm import Session

from . import models, schemas


def get_user(db: Session, user_id: int):
    return db.query(models.User).filter(models.User.id == user_id).first()


def get_user_by_name(db: Session, name: str):
    return db.query(models.User).filter(models.User.name == name).first()


def get_users(db: Session, skip: int = 0, limit: int = 100):
    return db.query(models.User).offset(skip).limit(limit).all()


def create_user(db: Session, user: schemas.UserCreate):
    new_user = models.User(**user.dict())
    db.add(new_user)
    db.commit()
    db.refresh(new_user)
    return new_user


def get_tasks(db: Session, user_id: int, skip: int = 0, limit: int = 100):
    return db.query(models.Task).filter(models.Task.user_id == user_id).offset(skip).limit(limit).all()


def create_user_task(db: Session, task: schemas.TaskCreate, user_id: int):
    new_task = models.Task(**task.dict(), user_id=user_id)
    db.add(new_task)
    db.commit()
    db.refresh(new_task)
    return new_task
```

- 各関数はsqlalchemy.orm.Session型の`db`インスタンスを引数に受け取り、このインスタンスを通じてデータベースを操作する
- Pydanticモデルのインスタンスは`task.dict()`で属性を辞書に変換できる

## 7. ルーティングを定義する
app/main.pyを変更し、ルーティングを定義する

```python:app/main.py
from fastapi import Depends, FastAPI, HTTPException
from sqlalchemy.orm import Session

from . import crud, models, schemas
from .database import SessionLocal, engine

models.Base.metadata.create_all(bind=engine)

app = FastAPI()


def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()


@app.post("/users/", response_model=schemas.User)
def create_user(user: schemas.UserCreate, db: Session = Depends(get_db)):
    db_user = crud.get_user_by_name(db, name=user.name)
    if db_user:
        raise HTTPException(status_code=400, detail=f"User name: {user.name} already exists.")
    return crud.create_user(db=db, user=user)


@app.get("/users/", response_model=list[schemas.User])
def get_users(skip: int = 0, limit: int = 100, db: Session = Depends(get_db)):
    users = crud.get_users(db, skip=skip, limit=limit)
    return users


@app.get("/users/{user_id}", response_model=schemas.User)
def get_user(user_id: int, db: Session = Depends(get_db)):
    user = crud.get_user(db, user_id=user_id)
    if user:
        raise HTTPException(status_code=404, detail=f"User ID: {user_id} not found")
    return user

@app.post("/users/{user_id}/tasks/", response_model=schemas.Task)
def create_task_for_user(user_id: int, task: schemas.TaskCreate, db: Session = Depends(get_db)):
    task = crud.create_user_task(db=db, task=task, user_id=user_id)
    return task


@app.get("/users/{user_id}/tasks/", response_model=list[schemas.Task])
def get_tasks_for_user(user_id: int, skip: int = 0, limit: int = 100, db: Session = Depends(get_db)):
    tasks = crud.get_tasks(db=db, user_id=user_id, skip=skip, limit=limit)
    return tasks
```

- リクエストごとにデータベースのセッションを発行→クローズする
    - そのため、全てのパス関数 (`@app.xxx`デコレータを付けてる関数のこと) でsqlalchemy.orm.Session型の`db`インスタンスを渡している
    - デフォルト値に`Depends(get_db)`を指定することで、パス関数を実行する前に`get_db`を実行し、dbインスタンスを取得する
        - もし取得できなかった場合は失敗する
        - つまり、パス関数はget_dbの返り値を要求する (依存させる)
        - 参考: https://fastapi.tiangolo.com/features/#dependency-injection
- エラーレスポンスを返したいときは `HTTPException` をraiseする
    - カスタム例外ハンドラーも定義できる
    - 参考: https://fastapi.tiangolo.com/tutorial/handling-errors/
- パスパラメータ・クエリパラメータ・ボディパラメータのバリデーションもここで定義することができる
    - 例えばクエリパラメータの最大文字数を制限したい場合は `def read_items(q: str = Query(max_length=50))` のように書ける
        - 参考: https://fastapi.tiangolo.com/ja/tutorial/query-params-str-validations/
    - また、models.pyの各モデルクラスでもバリデーションを定義できる
        - 参考: https://fastapi.tiangolo.com/ja/tutorial/schema-extra-example/

## 8. マイグレーションを実行する
SQLAlchemyのモデル定義をもとにデータベースにテーブルを作成するために、[Alembic](https://alembic.sqlalchemy.org/en/latest/index.html) を使用します

まず、alembicと、今回はデータベースにPostgreSQLを使うので psycopg2 をインストールします

```sh
pip install alembic psycopg2
```

dbディレクトリで `alembic init` コマンドを実行し、テンプレートを生成します

```sh
cd db
alembic init development # staging環境やproduction環境でDBの状態を分けることを想定してdevelopmentにしてる
```

```sh
$ tree .
.
├── alembic.ini
├── development
│   ├── README
│   ├── env.py
│   ├── script.py.mako
│   └── versions
└── init
    └── 1_create_databases.sql
```

db/alembic.iniのデータベースの指定を修正します

```diff:db/alembic.ini
- sqlalchemy.url = driver://user:pass@localhost/dbname
+ sqlalchemy.url = postgresql://postgres:password@0.0.0.0:5432/fastapi_todo_app_development
```

また、db/development/env.py のmetadataの指定を修正します

```diff:db/development/env.py
+ import os
+ import sys

# ...

 # add your model's MetaData object here
 # for 'autogenerate' support
+ sys.path.append(os.path.join(os.path.dirname(__file__), '../..'))
+ from app.models import Base
+ target_metadata = Base.metadata
- # from myapp import mymodel
- # target_metadata = mymodel.Base.metadata
- target_metadata = None

# ...
```

`alembic revision`コマンドで、マイグレーションファイルを生成します
このとき、[autogenerate](https://alembic.sqlalchemy.org/en/latest/autogenerate.html)オプションを指定することで、上で指定したSQLAlchemyのmetadataを参照してテーブル定義を生成することができます

```sh
alembic revision --autogenerate -m "first migration"
```

db/development/versions配下にmigrationファイルが生成されます

```python:db/development/versions/160c8582fb23_first_migration.py
"""first migration

Revision ID: 65ffef639936
Revises: 
Create Date: 2023-01-03 15:40:18.802548

"""
from alembic import op
import sqlalchemy as sa


# revision identifiers, used by Alembic.
revision = '65ffef639936'
down_revision = None
branch_labels = None
depends_on = None


def upgrade() -> None:
    # ### commands auto generated by Alembic - please adjust! ###
    op.create_table('users',
    sa.Column('id', sa.Integer(), nullable=False),
    sa.Column('name', sa.String(), nullable=True),
    sa.Column('created_at', sa.DateTime(), nullable=False),
    sa.PrimaryKeyConstraint('id')
    )
    op.create_index(op.f('ix_users_id'), 'users', ['id'], unique=False)
    op.create_index(op.f('ix_users_name'), 'users', ['name'], unique=True)
    op.create_table('tasks',
    sa.Column('id', sa.Integer(), nullable=False),
    sa.Column('title', sa.String(), nullable=True),
    sa.Column('done', sa.Boolean(), nullable=True),
    sa.Column('created_at', sa.DateTime(), nullable=False),
    sa.Column('updated_at', sa.DateTime(), nullable=False),
    sa.Column('user_id', sa.Integer(), nullable=True),
    sa.ForeignKeyConstraint(['user_id'], ['users.id'], ),
    sa.PrimaryKeyConstraint('id')
    )
    op.create_index(op.f('ix_tasks_done'), 'tasks', ['done'], unique=False)
    op.create_index(op.f('ix_tasks_id'), 'tasks', ['id'], unique=False)
    op.create_index(op.f('ix_tasks_title'), 'tasks', ['title'], unique=False)
    # ### end Alembic commands ###


def downgrade() -> None:
    # ### commands auto generated by Alembic - please adjust! ###
    op.drop_index(op.f('ix_tasks_title'), table_name='tasks')
    op.drop_index(op.f('ix_tasks_id'), table_name='tasks')
    op.drop_index(op.f('ix_tasks_done'), table_name='tasks')
    op.drop_table('tasks')
    op.drop_index(op.f('ix_users_name'), table_name='users')
    op.drop_index(op.f('ix_users_id'), table_name='users')
    op.drop_table('users')
    # ### end Alembic commands ###
```

`alembic upgrade`コマンドで、実際にマイグレーションを実行します
```sh
$ alembic upgrade head
```

これでテーブルが作成されます

## 9. 動作確認する
最後に、サーバーを起動して動作確認します

```sh
# プロジェクトディレクトリで実行する
uvicorn app.main:app --reload
```

OpenAPIが自動で定義されています

![](https://storage.googleapis.com/zenn-user-upload/702ba9c77e48-20230104.png)

curlでリクエストを送信してみます
```sh
$ curl localhost:8000/users/ \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{
    "name": "hoge"
  }'

{"name":"hoge","id":1,"created_at":"2023-01-03T20:02:18.375334","tasks":[]}%
```
