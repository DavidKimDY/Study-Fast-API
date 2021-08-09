# Study-Fast-API

# Parameter

- parameter는 qurery형과 path 형으로 존재한다.
- path형은 데코레이터에 `@app.get("/items/{item_id}")` 과 같은 형태로 있으며 함수의 인자를 `item_id` 로 받게 된다.
- query형은 함수의 인자로 받게되고 url에 `?` 와 `&` 으로 추가된다.
- path형과 query형 모두 사용이 가능하다.

# Request body

- Client가 API 서버로 보내는 data다
- API가 Client에게는 Response body를 보낸다.
- pydantic의 BaseModel을 이용해서 Request body의 형태를 지정해 줄 수 있다

```python
class Item(BaseModel):
    name: str
    description: Optional[str] = None
    price: float
    tax: Optional[float] = None

@app.post('/items/')
async def create_item(item: Item):
	return item
```

- 이렇게 type을 지정해주면 자동완성기능(Editor support)과 오류감지(dectecting error) 기능을 제공해준다.
- If the parameter is declared to be of the type of a Pydantic model, it will be interpreted as a request body.
- `import Query` 를 이용해서 입력값의 길이를 제한할 수 있다.

    ```python
    from typing import Optional

    from fastapi import FastAPI, Query

    app = FastAPI()

    @app.get("/items/")
    async def read_items(q: Optional[str] = Query(None, max_length=50)):
        results = {"items": [{"item_id": "Foo"}, {"item_id": "Bar"}]}
        if q:
            results.update({"q": q})
        return results
    ```

- Optional과 List로 request body에 리스트를 받아 올 수 있다.

    ```python
    from typing import Optional, List

    from fastapi import FastAPI, Query

    app = FastAPI()

    @app.get("/items/")
    async def read_items(q: Optional[List[str]] = Query(None, max_length=50)):
        results = {"items": [{"item_id": "Foo"}, {"item_id": "Bar"}]}
        if q:
            results.update({"q": q})
        return results

    ```

# Optional은 왜 쓰는것일까?

- `Optional[...]` 은 `Union[..., None]` 의 축약형으로 type checker에게 구체적인 타입의 오브젝트 또는 None이 필요하다고 알리는 일을 한다.
- `...` 은 유효한 타입 힌트를 의미하며 복잡한 합성 타입이나 `Union[]` 를 포함한다.
- `None` 값을 디폴트로 갖는Keyword argument에는 `Optional` 을 사용하는 것이 좋다.

# Query 와 Request body

- 둘 다 함수의 인수를 통해서 들어온다.
- Request body는 직접 만든 class를 인수의 타입으로 적용해 받을 수 있다.
- 단일 인수같은 경우 Body를 default로 넣으면 Request body에 적용된다.

# Response Model

- 어떤 path operation에서라도 응답에 사용되는 모델을 `response_model` 파라미터와 함께 선언할 수 있다.
- <**참고**>

    `response_model` 은 데코레이터의 파라미터이다. 일반 파라미터나 body와 같이 경로 연산 함수의 것이 아니다.

- response_model은 당신이 Pydantic model arribute로 지정해 놓은 타입으로 받는다. 그래서 response_model은 Pydantic model이 될 수도 있고 List[Item]이 될 수도 있다.
- Fast API는 response_model을 이렇게 쓸 것이다
    1. output data를 response_model의 선언 타입으로 변환하기 위해
    2. data의 유요성 검증
    3. OpenAPI 경로연산의 응답에 Json Schema를 추가해주기 위해
    4. automatic documentation을 위해

    하지만 가장 중요한 것은

    - output data를 response_model로 제한하는 것

    이다.

- **<기술적인 상세>**

    response model은 type annotation을 반환하는 함수대신에 선언되었다. 왜냐하면 경로 함수는 response model을 반환하지 않고 딕셔너리나 데이터베이스 객체나 다른 모델들을 반환하며 response_model을 field 제한이나 직렬화에 사용하기 때문이다.

### Return the same input data

- 사용자의 패스워드를 갖고 있는 UserIn라는 모델을 살펴보자

```python
class UserIn(BaseModel):
    username: str
    password: str
    email: EmailStr
    full_name: Optional[str] = None

# Don't do this in production!
@app.post("/user/", response_model=UserIn)
async def create_user(user: UserIn):
    return user
```

- 위 모델은 input을 선언하고 같은 모델을 output으로 선언하기 위해 사용된다.
- 위 코드는 브라우저가 user를 password와 함께 생상할 때마다 API는 같은 password를 응답으로 반환한다.
- 이같은 경우는 문제가 되지 않을 수도 있다. 왜냐면 유저 자신이 그 password를 직접 보냈기 때문이다.
- 하지만 우리가 같은 모델을 다른 경로 연산에 사용한다면 이 유저의 비밀번호를 모든 client들에게 보내게 될 수도 있다.
- **<위험>**

    절대 비밀번호를 그대로 저장하거나 응답으로 보내서는 안된다.

### Add an output model

- 우리는 대신에 문자 그대로의 비밀번호를 Input model로 받고 output model에는 비밀번호를 저장하지 않을 수 있다

```python
class UserIn(BaseModel):
    username: str
    password: str
    email: EmailStr
    full_name: Optional[str] = None

class UserOut(BaseModel):
    username: str
    email: EmailStr
    full_name: Optional[str] = None

@app.post("/user/", response_model=UserOut)
async def create_user(user: UserIn):
    return user
```

- 경로 연산 함수가 비록 똑같은 input user를 반환하더라도 우리는 `response_model=UserOut` 으로 선언했기 때문에 비밀번호는 포함되지 않는다.
- 그래서 FastAPI는 output model에서 선언되지 않은 data들은 모두 필터링할 것이다. (Pydantic을 사용해서)

### Response Model encoding parameters

- response model은 default 값을 가질 수 있다.
- `response_model_exclude_unset=True` 을 설정하면 default 값이 응답에 포함되지 않을 것이다.

# Extra Models

- **<위험>**

    사용자의 비밀번호를 그대로 저장해서는 안된다. `secure hash` 로 저장하고 확인해야한다.

### Multiple models

```python
class UserIn(BaseModel):
    username: str
    password: str
    email: EmailStr
    full_name: Optional[str] = None

class UserOut(BaseModel):
    username: str
    email: EmailStr
    full_name: Optional[str] = None

class UserInDB(BaseModel):
    username: str
    hashed_password: str
    email: EmailStr
    full_name: Optional[str] = None

def fake_password_hasher(raw_password: str):
    return hash(raw_password)

def fake_save_user(user_in: UserIn):
    hashed_password = fake_password_hasher(user_in.password)
    user_in_db = UserInDB(**user_in.dict(), hashed_password=hashed_password)
    print("User Saved")
    return user_in_db

@app.post('/user/', response_model=UserOut)
async def create_user(user_in: UserIn):
    user_saved = fake_save_user(user_in)
    return user_saved
```

- 사용자의 비밀번호와 그것들의 위치에 관한 모델들의 일반적인 아이디어가 위의 코드이다.

### Pydantic's .dict()

- `user_in` 은 UserIn 클래스의 Pydantic 모델이다.
- Pydantic 모델은  `.dict()` method를 갖고 있다. 이 메소드는 모델의 데이터를 dict로 반환해준다.
- 그래서 만약 Pydantic의 객체인 user_in을 다음과 같이 생성한다면 `user_in = UserIn(username="john", password="secret", [email="john.doe@example.com](mailto:email=%22john.doe@example.com)")` user_dict에는 `user_dict = user_in.dict()` 가 들어가게 된다.

### Reduce duplication

- 중복 코드를 없애는 것은 FastAPI의 핵심아이디어중 하나다.
- 코드 중복은 많은 문제점을 발생시킨다.
- 위 모델들은 많은 데이터를 공유하고 중복된 속성과 타입을 갖고 있다.
- 기본적인 것들을 다른 모델들에게 제공하는 `UserBase` 를 선언할 수 있다. 그리고서 이 모델을 상속하는 subclass들을 만들 수 있다.
- 모든 데이터의 변환, 증명, 문서화 등은 여전히 잘 동작할 것이다.

```python
class UserBase(BaseModel):
    username: str
    email: EmailStr
    full_name: Optional[str] = None

class UserIn(UserBase):
    password: str

class UserOut(UserBase):
    pass

class UserInDB(UserBase):
    hashed_password: str
```

### Union of anyOf

- 응답을 두가지 타입의 `Union` 으로 선언할 수 있다.
- 즉, 응답이 두가지중 하나가 되게 할 수 있다.
- OpenAPI 와 `anyOf` 안에서 정의될 것이다.
- 이렇게 하기 위해서 `typing.Union` 을 사용한다.
- **<참고>**

    Union을 정의할 때는, 가장 구체적인 타입을 먼저 포함하고 그 다음 덜 구체적인 타입을 포함해라.

    다음의 예제에서는 `Union[PlaneItem, CarItem]` 과 같이 더 구체적인 타입인 `PlaneItem` 이 `CarItem` 보다 먼저 포함되는 것을 볼 수 있다. (안그러면 PlaneItem이 갖고 있는 specific한 attribute(`size: int` )가 response_model에 적용되지 않을 것이다.)

```python
class BaseItem(BaseModel):
    description: str
    type: str

class CarItem(BaseItem):
    type = "car"

class PlaneItem(BaseItem):
    type = "plane"
    size: int

items = {
    "item1": {"description": "All my friends drive a low rider", "type": "car"},
    "item2": {
        "description": "Music is my aeroplane, it's my aeroplane",
        "type": "plane",
        "size": 5,
    },
}

@app.get("/items/{item_id}", response_model=Union[PlaneItem, CarItem])
async def read_item(item_id):
    return items[item_id
```

### Response with arbitrary dict

- 있는 그대로의 독단적인 딕셔너리 또한 응답으로 선언할 수 있다.
- Pydantic 모델을 사용하지 않고 key 타입과 value 타입을 선언해 줌으로 가능하다.
- field나 attribute의 이름을 정확히 알지 못할 경우에 유용하게 사용할 수 있다. (Pydantic 모델은 정확히 알아야 하니께)

```python
from typing import Dict

@app.get("/keyword-weights/", response_model=Dict[str, float])
async def read_keyword_weights():
    return {"foo": 2.3, "bar": 3.4}
```

# Response Status Code

- 경로 연산 안에 `status_code` 를 지정해 줌으로서 상태 코드를 지정할 수 있다.
- **<참고>**

    `status_code` 는 데코레이터 메소드의 파라미터다. 경로 연산 함수의 것이 아니다.

### shortcut to remember the name

- fastapi의 `status` 를 이용하면 기억하기 쉽다.
- `status.HTTP_201_CREATED` 와 같이 IDE에서 자동완성으로 사용하면 수월하다.
- **<기술적인 상세>**
    - `from starlette import status` 를 사용해도 된다. 왜냐하면 fastapi의 `status` 는 starlette의 `status` 를 그대로 사용한 것이기 때문이다.

# Form Data

- Json 대신에 form field를 받아야 할때 `Form` 을 써주면 된다.
- **<알림>**

    form을 쓰기 위해서는 `python-multipart` 를 설치해주어야 한다

### Define Form parameters

- Body나 Query를 사용하는 방법과 동일하게 Form을 사용할 수 있다.
- 예를 들어서, 여러 방법중에 하나로 OAuth2 specification을 사용할 수 있는데 이는 `username`과  `password`를 form field로써 보내기를 요구한다.
- The spec(OAuth2 를 의미하는 것으로 추정)은  field의 이름이 정확히 `username`과 `password` 가 되는 것을 요구하며 Json 형태가 아닌 form field로 보내지는 것을 요구한다.

    ### HTML FORM

    HTML Form은 interative controls를 사용하는 웹서버에서의 유저들의 정보를 저장하는 document다. HTML Form은 username, password, contact number, email id 등을 담고 있다. HTML form이 사용되는 요소는 체크박스, 인풋박스, 라디오 버튼, 제출 버튼 등이 있다.  이런 요소를 사용함으로 사용자들의 정보다 웹 서버로 제출된다.

- `Form` 을 사용함으로써 Body나 Query를 다루었던 것럼 메타데이터와 검증을 선언할 수 있다.
- **<정보>**

    `Form`은 `Body` 를 직접 상속받는다.

- **<팁>**

    form body를 선언할 때는 `Form` 을 꼭 써줘야 한다. 그렇지 않으면 파라미터들은 query 파라미터나 혹은 body(JSON) 파라미터로 해석된다
