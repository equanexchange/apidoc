# equan一级平台对接api文档

- [接入说明](#接入说明)
  - [对接流程](#对接流程)
  - [签名方式](#签名方式)
  - [请求格式](#请求格式)
  - [时间戳](#时间戳)
  
- [业务API](#业务API)
  - [寄售](#寄售)
    - [变量说明](#变量说明)
    - [用户验证](#用户验证)
    - [藏品列表](#藏品列表)
    - [藏品详情](#藏品详情)
    - [上下架询问](#上下架询问)  
    - [验证地址](#验证地址)
    - [藏品转赠](#藏品转赠)
    - [转赠结果](#转赠结果)
  - [授权登录](#授权登录)
    - [授权流程](#授权流程)
    - [跳转参数](#跳转参数)
    - [用户信息](#用户信息)
    - [藏品信息](#藏品信息)

- [错误码](#错误码)

## 接入说明

### 对接流程
- 一级平台需根据本文档的要求来开发接口
- equan服务端会调用一级平台接口以实现功能
- 接口通过签名和时间戳校验来保证安全性，用于签名的密钥对由equan分发

### 签名方式
1. equan会分发一对密钥用于签名，分别是api key 和secret key
2. 所有业务参数都会格式化为`foo=bar`这样，并以`&`间隔按字典顺序拼接，拼接成`foo=bar&foo1=bar1&foo2=bar2`这样的字符串，最后将时间戳参数`timestamp`追加到末尾，得到`foo=bar&foo1=bar1&foo2=bar2&timestamp=timestamp`。
3. 签名使用HMAC SHA256算法，使用secret key作为密钥，将上一步得到的字符串作为输入进行计算，并将输出转成大写十六进制，这一步得到的签名将会通过form参数`signature`传给一级平台进行校验
4. 若校验签名不通过，请返回http status code 401
5. 示例:  假设secret key是`kV4iqX9jGZKvC5o6ZHJhjBEr`，签名计算如下
  ```sh
$ echo -n 'foo=bar&foo1=bar1&foo2=bar2&timestamp=timestamp' | \
openssl dgst -sha256 -hmac 'kV4iqX9jGZKvC5o6ZHJhjBEr' | \
awk '{print toupper($0)}'
(stdin) = D3322A315096C278028AC3374847AD2B91BD4AAFB5FB07E0E50B8EC9F327C2ED
  ```

### 请求格式
- 所有请求方法都是POST

- 请求的`Content-Type`是`application/x-www-form-urlencoded`

- 请求都会包括`signature`和`timestamp`参数，`timestamp`会参与签名生成

- api key将通过header传递，字段名`X-EQUAN-APIKEY`

- 因本文档只约定了接口地址的最末端路径，请统一所有接口的路径前缀

- 若签名和时间戳校验通过，body请按如下固定参数返回:
  
  | 参数名    | 类型      | 是否必填  | 说明             |
  | --------- | ------------ | ------------------ | -------- |
  | **code** | number   | 是        | 若无报错返回0，报错返回错误码 |
  | **message**  | string     | 是         | 若无报错可为空，报错返回错误说明 |
  | **data** | object | 是 | 返回各接口所需的数据 |
  
- 若签名校验失败，请返回http status code 401

- 若时间戳校验失败，请返回http status code 400

- 若http status code 不返回200，可自行定义返回的body内容

### 时间戳
若equan请求一级平台接口时传递的`timestamp`早于当前时间过多，建议不处理该请求，直接返回http status code 400，是否判断该参数以及具体时间差阈值请自行斟酌

## 业务API

### 寄售

#### 变量说明
对下面文档中会出现的参数的一些说明
- `api.partner.com`: 假设一级平台方的请求域名
- `equan/v1`: 假设一级平台方本文档内的接口地址前缀
- `2xeBreQOWzW7aGh26a1T1cewlwXKg0gz`: 用于示例的api key
- `hF7IDqvXh2qNz8pWWJdfdW3EfPnN45ES`: 用于示例的secret key
- `1661356800000`: 用于示例的时间戳

#### 用户验证
用于向一级平台验证用户信息，一级平台存在该用户且信息核对通过后，返回一级平台内标记该用户的唯一id

- 接口地址: **/user**

- 请求方法: **POST**

- 请求参数: 

  | 参数名    | 说明         | 示例值             |
  | --------- | ------------ | ------------------ |
  | **phone** | 用户手机号   | 18888888888        |
  | **name**  | 用户姓名     | 王建国             |
  | **id_no** | 用户身份证号 | 400100197001012222 |

- 响应参数:

  | 参数名     | 类型     |  是否必填  |  说明  |
  | ---- | ---- | ---- | ---- |
  | **valid**     |  boolean    | 是 | 若一级平台存在该用户且信息核对无误，则返回true，若不存在或信息核对有误，则返回false |
  | **user_id**     |   string   | 否 | 该用户在一级平台内的唯一标识，若valid返回true，该字段必填 |
  | **fail_reason** |   string   | 否 | 失败原因 |

- cURL示例:
```sh
curl -X POST "https://api.partner.com/equan/v1/user" \
    -H "X-EQUAN-APIKEY: 2xeBreQOWzW7aGh26a1T1cewlwXKg0gz" \
    -H "Content-Type: application/x-www-form-urlencoded; charset=utf-8" \
    --data-raw "phone"="18888888888" \
    --data-raw "name"="王建国" \
    --data-raw "id_no"="400100197001012222" \
    --data-raw "timestamp"="1661356800000" \
    --data-raw "signature"="E0CB29683B70A7B1558C28AE3B20DC83B2E83B29C7FA3A817B1F7C60F72EB511"
```

- 响应示例：
```json
{
    "code": 0,
    "message": "",
    "data": {
        "valid": true,
        "user_id": "71cb45b9-4caa-4074-b249-0b7f66c8cb7a",
        "fail_reason": ""
    }
}
```

#### 藏品列表
获取用户在一级平台内的藏品列表

- 接口地址: **/collection**

- 请求方法: **POST**

- 请求参数: 

  | 参数名    | 说明         | 示例值             |
  | --------- | ------------ | ------------------ |
  | **user_id** | 用户在一级平台的id   | xxxx-xxxx-xxxx        |
  | **type** | 藏品类型，传DESTROY时，请返回可销毁的藏品，默认不传   | DESTROY       |
  | **offset**  | 记录偏移量，用于分页     | 0             |
  | **limit** | 需要返回的记录数量，用于分页 | 10 |

- 响应参数:

  | 参数名     | 类型     |  是否必填  |  说明  |
  | ---- | ---- | ---- | ---- |
  | **total** | number | 是 | 用户藏品总数量，若不支持分页可返回0 |
  | **records** | array | 是 | 藏品列表，以下为数组内对象字段 |
  | **collection_id**     |  string    | 是 | 藏品或藏品库存唯一标识，假如藏品有多个库存，是能定位到具体某个库存的id |
  | **sku_id**     |  string    | 是 | 藏品类型或藏品链接唯一标识，这个id并不需要区分到具体库存，只需能区分不同藏品或藏品类型即可 |
  | **name**     |   string   | 是 | 藏品名称 |
  | **image**     |   string   | 是 | 藏品图片，值为链接或base64编码 |
  | **image_type**     |   string   | 是 | 藏品图片类型，如果是链接返回URL，如果是base64编码返回BASE64 |
  | **price**     |   string   | 是 | 藏品的发行价格 |
  | **created_time**     |   number   | 否 | 藏品发行时间，秒级时间戳 |
  | **status**     |   string   | 是 | 藏品状态，可以转赠返回OK，无法转赠返回NOT_OK |
  
- cURL示例:
```sh
curl -X POST "https://api.partner.com/equan/v1/collection" \
    -H "X-EQUAN-APIKEY: 2xeBreQOWzW7aGh26a1T1cewlwXKg0gz" \
    -H "Content-Type: application/x-www-form-urlencoded; charset=utf-8" \
    --data-raw "user_id"="xxxx-xxxx-xxxx" \
    --data-raw "offset"="0" \
    --data-raw "limit"="10" \
    --data-raw "timestamp"="1661356800000" \
    --data-raw "signature"="789F8E99EE5DC67BDD0191D8F2F126098B829BEB9922EFEF4C714B5C081E510D"
```

- 响应示例：
```json
{
    "code": 0,
    "message": "",
    "data": {
        "records": [
            {
                "collection_id": "e6a3bc28-ab36-407b-97ca-5e61f45799c6",
                "name": "秦始皇陵内部结构图",
                "image": "https://image.example.com/xxxxxx.jpeg",
                "image_type": "URL",
                "price": "999",
                "created_time": 1661356800,
                "status": "OK"
            }
        ],
        "total": 1
    }
}
```

#### 藏品详情
获取用户在一级平台内的藏品详情，以判断藏品是否可以买入

- 接口地址: **/collection/detail**

- 请求方法: **POST**

- 请求参数: 

  | 参数名    | 说明         | 示例值             |
  | --------- | ------------ | ------------------ |
  | **user_id** | 用户id   | xxxx-xxxx-xxxx        |
  | **collection_id**  | 藏品id     | aaaa-bbbb-cccc             |

- 响应参数:

  | 参数名     | 类型     |  是否必填  |  说明  |
  | ---- | ---- | ---- | ---- |
  | **collection_id**     |  string    | 是 | 藏品或藏品库存唯一标识，假如藏品有多个库存，是能定位到具体某个库存的id |
  | **sku_id**     |  string    | 是 | 藏品类型或藏品链接唯一标识，这个id并不需要区分到具体库存，只需能区分不同藏品或藏品类型即可 |
  | **name**     |   string   | 是 | 藏品名称 |
  | **image**     |   string   | 是 | 藏品图片，值为链接或base64编码 |
  | **image_type**     |   string   | 是 | 藏品图片类型，如果是链接返回URL，如果是base64编码返回BASE64 |
  | **price**     |   string   | 是 | 藏品的发行价格 |
  | **created_time**     |   number   | 否 | 藏品发行时间，秒级时间戳 |
  | **status**     |   string   | 是 | 藏品状态，可以转赠返回OK，无法转赠返回NOT_OK |
  | **total_amount**     |   number   | 否 | 藏品发行数量 |
  | **number**     |   string   | 否 | 藏品编号 |
  | **introduce**     |   string   | 否 | 藏品介绍，最长1000字符 |
  | **tech_support**     |   string   | 否 | 藏品技术支持 |
  | **hash**     |   string   | 否 | 交易哈希 |

- cURL示例:
```sh
curl -X POST "https://api.partner.com/equan/v1/collection/detail" \
    -H "X-EQUAN-APIKEY: 2xeBreQOWzW7aGh26a1T1cewlwXKg0gz" \
    -H "Content-Type: application/x-www-form-urlencoded; charset=utf-8" \
    --data-raw "user_id"="xxxx-xxxx-xxxx" \
    --data-raw "collection_id"="aaaa-bbbb-cccc" \
    --data-raw "timestamp"="1661356800000" \
    --data-raw "signature"="925D03EA011BDC7A52E253EF4C45D96A5DE2A1A9619D9E04923626A6070011E5"
```

- 响应示例：
```json
{
    "code": 0,
    "message": "",
    "data": {
        "collection_id": "e6a3bc28-ab36-407b-97ca-5e61f45799c6",
        "name": "秦始皇陵内部结构图",
        "image": "https://image.example.com/xxxxxx.jpeg",
        "image_type": "URL",
        "price": "999",
        "created_time": 1661356800,
        "total_amount": 100,
        "status": "OK",
        "number": "#001",
        "introduce": "我秦始皇打钱",
        "tech_support": "牛逼老哥",
        "hash": "0xff000000001"
    }
}
```

#### 上下架询问
用户在 equan **上架或下架**一级平台的藏品时，equan 将调用此接口进行询问（需配置，默认不询问），当一级平台返回失败时，上下架将失败。无此需求的平台该接口可不对接。

- 接口地址: **/collection/ask**

- 请求方法: **POST**

- 请求参数:

  | 参数名    | 说明         | 示例值             |
  | --------- | ------------ | ------------------ |
  | **user_id** | 用户id   | xxxx-xxxx-xxxx        |
  | **collection_id**  | 藏品id     | aaaa-bbbb-cccc             |
  | **type**  | 操作类型，ON_SHELF 上架，OFF_SHELF 下架 | ON_SHELF / OFF_SHELF |

- 响应参数:

  | 参数名     | 类型     |  是否必填  |  说明  |
  | ---- | ---- | ---- | ---- |
  | **valid**     |   boolean   | 是 | 当 status 传 ON_SHELF 时，若允许上架请返回 true，否则返回 false 将上架失败；当 status 传 OFF_SHELF 时，若允许下架请返回 true，否则返回 false 将下架失败 |

- cURL示例:
```sh
curl -X GET "http://api.example.com/collection/ask" \
    -H "X-EQUAN-APIKEY: 2xeBreQOWzW7aGh26a1T1cewlwXKg0gz" \
    -H "Content-Type: application/x-www-form-urlencoded; charset=utf-8" \
    --data-raw "user_id"="xxxx-xxxx-xxxx" \
    --data-raw "collection_id"="aaaa-bbbb-cccc" \
    --data-raw "type"="ON_SHELF" \
    --data-raw "timestamp"="1661356800000" \
    --data-raw "signature"="E2D99AC73D79C173CD6D1DC645855058D138DB393A19E09721007372A0906833"
```

- 响应示例：
```json
{
    "code": 0,
    "message": "",
    "data": {
        "valid": true
    }
}
```

#### 验证地址
验证买家的地址是否存在于一级平台内

- 接口地址: **/address/verify**

- 请求方法: **POST**

- 请求参数: 

  | 参数名    | 说明         | 示例值             |
  | --------- | ------------ | ------------------ |
  | **collection_id**  | 藏品id，可用于判断藏品是否可转赠给目标地址     | aaaa-bbbb-cccc             |
  | **address** | 目标地址，判断一级平台内是否存在该地址，或是否可接收藏品的转赠   | 0xff00000001        |

- 响应参数:

  | 参数名     | 类型     |  是否必填  |  说明  |
  | ---- | ---- | ---- | ---- |
  | **valid**     |   boolean   | 是 | 地址是否存在，或是否可接收藏品的转赠，是返回true，若不存在或不可接受转赠则返回 false |
  | **fail_reason**     |   string   | 否 | 失败原因 |

- cURL示例:
```sh
curl -X GET "http://api.example.com/address/verify" \
    -H "X-EQUAN-APIKEY: 2xeBreQOWzW7aGh26a1T1cewlwXKg0gz" \
    -H "Content-Type: application/x-www-form-urlencoded; charset=utf-8" \
    --data-raw "address"="0xff00000001" \
    --data-raw "collection_id"="aaaa-bbbb-cccc" \
    --data-raw "timestamp"="1661356800000" \
    --data-raw "signature"="5E9E53E4F9AA29C82DB60CCC61751DE61CF7C6EF2ED3B00ED04CA837946FFC9A"
```

- 响应示例：
```json
{
    "code": 0,
    "message": "",
    "data": {
        "valid": true,
        "fail_reason": ""
    }
}
```

#### 藏品转赠
用户将一级平台内的藏品转赠给指定的地址

- 接口地址: **/transfer**

- 请求方法: **POST**

- 请求参数: 

  | 参数名    | 说明         | 示例值             |
  | --------- | ------------ | ------------------ |
  | **user_id** | 用户id   | xxxx-xxxx-xxxx        |
  | **collection_id**  | 藏品id     | aaaa-bbbb-cccc             |
  | **type** | 操作类型，DESTROY表示销毁藏品，当销毁时不会传address，默认不传   | DESTROY       |
  | **address** | 转赠目标地址   | 0xff00000001        |

- 响应参数:

  | 参数名     | 类型     |  是否必填  |  说明  |
  | ---- | ---- | ---- | ---- |
  | **transfer_id**     |  string    | 是 | 转赠id，或者是 hash，标记唯一一笔转赠 |
  | **status**     |   string   | 是 | 转赠状态，PENDING 转赠中，SUCCESS 转赠成功，FAILED 失败不可重发，FAILED_REPEAT 失败可重发，若因一些暂时的故障返回 失败可重发，equan会继续调用转赠接口，直到返回其他状态 |
  | **fail_reason**     |   string   | 否 | 失败原因 |

- cURL示例:
```sh
curl -X POST "https://api.partner.com/equan/v1/transfer" \
    -H "X-EQUAN-APIKEY: 2xeBreQOWzW7aGh26a1T1cewlwXKg0gz" \
    -H "Content-Type: application/x-www-form-urlencoded; charset=utf-8" \
    --data-raw "user_id"="xxxx-xxxx-xxxx" \
    --data-raw "collection_id"="aaaa-bbbb-cccc" \
    --data-raw "address"="0xff00000001" \
    --data-raw "timestamp"="1661356800000" \
    --data-raw "signature"="DFDA0494CA273DA87C3F328496810E4135D581D8476C51770B4BC049166F2B5C"
```

- 响应示例：
```json
{
    "code": 0,
    "message": "",
    "data": {
        "transfer_id": "9400bf96ad3973ba2dfc3102b5c515aa",
        "status": "PENDING",
        "fail_reason": ""
    }
} 
```

#### 转赠结果
用于查询转赠的结果，若转赠接口能立即返回最终状态，则无需对接此接口

- 接口地址: **/transfer/result**

- 请求方法: **POST**

- 请求参数: 

  | 参数名    | 说明         | 示例值             |
  | --------- | ------------ | ------------------ |
  | **transfer_id** | 转赠id   | 9400bf96ad3973ba2dfc3102b5c515aa        |

- 响应参数:

  | 参数名     | 类型     |  是否必填  |  说明  |
  | ---- | ---- | ---- | ---- |
  | **transfer_id**     |  string    | 是 | 转赠id，或者是 hash，标记唯一一笔转赠 |
  | **status**     |   string   | 是 | 转赠状态，PENDING 转赠中，SUCCESS 转赠成功，FAILED 失败不可重发，FAILED_REPEAT 失败可重发 |
  | **fail_reason**     |   string   | 否 | 失败原因 |
  
- cURL示例:
```sh
curl -X POST "https://api.partner.com/equan/v1/transfer/result" \
    -H "X-EQUAN-APIKEY: 2xeBreQOWzW7aGh26a1T1cewlwXKg0gz" \
    -H "Content-Type: application/x-www-form-urlencoded; charset=utf-8" \
    --data-raw "transfer_id"="9400bf96ad3973ba2dfc3102b5c515aa" \
    --data-raw "timestamp"="1661356800000" \
    --data-raw "signature"="AE170CB6D20688113B3D0EE914FA5C81C77319AE5F14B809DE1A5943BF5BF665"
```

- 响应示例：
```json
{
    "code": 0,
    "message": "",
    "data": {
        "transfer_id": "9400bf96ad3973ba2dfc3102b5c515aa",
        "status": "SUCCESS",
        "fail_reason": ""
    }
}
```

### 授权登录

#### 授权流程

- 一级平台跳转到 equan 的授权页面，进行用户登录授权
  
- 授权页登录后，equan 会同步（销毁）其他一级平台藏品，该过程仍处于授权进行中，藏品同步完成后，才算授权成功

- 授权成功后，equan 授权页会跳转至一级平台的页面，并通过跳转链接传递 access token、token 有效期等字段给一级平台

- 一级平台调用需授权的接口时，header 必须携带 access token，格式为 `Authorization: Bearer access token`

#### 跳转参数

- 页面地址: https://www.equan.com/mobile/auth

- 跳转参数: 跳转到授权页面时，需在链接上携带的参数

  | 参数名     | 是否必填     |  说明  |  示例值  |
  | ---- | ---- | ---- | ---- |
  | **api_key**     |  是    | equan分发的密钥，与寄售api key相同 | xxxxx-xxxxx-xxxxx |

- 回调参数: 授权成功后，跳转到一级平台页面会携带的参数

  | 参数名     | 是否必填     |  说明  |  示例值  |
  | ---- | ---- | ---- | ---- |
  | **access_token**     |  是    | 授权成功后生成的 access token，访问需要授权的接口时需携带 | xxxxx-xxxxx-xxxxx |
  | **expiry**     |  是    | access token 的过期时间，秒级时间戳 | 1663862400 |

#### 用户信息

获取用户在 equan 的注册信息

- 接口地址: **/api/v1/external/user**

- 请求方法: **GET**

- 需要授权: **是**

- 请求参数: 无

- 响应参数:

  | 参数名     | 类型     |  是否必填  |  说明  |
  | ---- | ---- | ---- | ---- |
  | **phone**     |  string    | 是 | 用户手机号 |
  | **name**     |   string   | 是 | 用户姓名 |
  | **id_no**     |   string   | 是 | 用户身份证号 |

- cURL示例:
```sh
curl -X GET 'https://www.equan.com/api/v1/external/user' \
-H 'Authorization: Bearer access token'
```

- 响应示例:
```json
{
  "code": 0,
  "message": "操作成功",
  "data": {
    "phone": "18888888888",
    "name": "王建国",
    "id_no": "400100197001012222"
  }
}
```

#### 藏品信息

获取用户在其他一级平台已经同步（销毁）的藏品，该接口返回同步藏品的历史数据，**非每次最新同步的藏品**，故请对接方自己维护一份数据，如想获取最新的增量数据，请将本地数据的最大id传过来。

- 接口地址: **/api/v1/external/collection**

- 请求方法: **GET**

- 需要授权: **是**

- 请求参数:

  | 参数名     | 是否必填     |  说明  |  示例值  |
  | ---- | ---- | ---- | ---- |
  | **id**     |  否    | 记录id，若传值将只返回id大于该值的记录 | 100，只返回id>100的记录 |
  | **offset**     |   否   | 记录偏移量，**非页码**，传几则跳过几条记录 | 0 |
  | **limit** |   是   | 记录数量，最大200 | 10 |

- 响应参数:

  | 参数名     | 类型     |  是否必填  |  说明  |
  | ---- | ---- | ---- | ---- |
  | **total**     |  number    | 是 | 记录的总数量 |
  | **records**     |   array   | 是 | 已同步藏品记录，以下为数组内对象字段|
  | **id**     |  number    | 是 | 记录id|
  | **platform_id**     |  string    | 是 | 藏品在同步或销毁前，所在的一级平台id|
  | **collection_id**     |  string    | 是 | 藏品或藏品库存唯一标识，假如藏品有多个库存，是能定位到具体某个库存的id |
  | **sku_id**     |  string    | 是 | 藏品类型或藏品唯一标识，这个id并不需要区分到具体库存，只需能区分不同藏品或藏品类型即可 |
  | **name**     |   string   | 是 | 藏品名称 |
  | **image**     |   string   | 是 | 藏品图片，值为链接或base64编码 |
  | **image_type**     |   string   | 是 | 藏品图片类型，如果是链接返回URL，如果是base64编码返回BASE64 |
  | **price**     |   string   | 是 | 藏品的发行价格 |
  | **created_time**     |   number   | 是 | 藏品发行时间，秒级时间戳 |

- cURL示例: 
```sh
curl -X GET 'https://www.equan.com/api/v1/external/collection?limit=10&id=888&offset=0' \
-H 'Authorization: Bearer access token'
```

- 响应示例:
```json
{
    "code": 0,
    "message": "操作成功",
    "data": {
        "total": 1,
        "records": [
            {
                "id": 1,
                "platform_id": "88",
                "collection_id": "xxx-xxx-xxx-123",
                "sku_id": "xxx-xxx-xxx",
                "name": "银河系韦伯望远镜首拍",
                "image": "https://image.example.com/xxxxxx.jpeg",
                "image_type": "URL",
                "price": "999",
                "created_time": 1661356800
            }
        ]
    }
}
```

## 错误码

| 错误码     | 类型             | 说明                              |
| ---------- | ---------------- | --------------------------------- |
| 400        | http status code | 时间戳错误或其他传参错误          |
| 401        | http status code | 签名校验不通过                    |
| 其他错误码 | 具体业务上的错误 | 请在body里的code和message字段返回 |
