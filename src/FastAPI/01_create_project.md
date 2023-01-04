#　1. プロジェクトを作成する
FastAPIに必要なライブラリをインストールします

```sh
pip install fastapi "uvicorn[standard]"
```

サンプルコードで試しにFastAPIを起動してみます
app/main.pyを作成します

{% label %}test.js{% endlabel %}
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

サーバーを立ち上げます
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
