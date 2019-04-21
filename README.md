# Payzero QuickPay API文档

## 技术综述

### 请求格式说明
QuickPay API整体采用RESTful API的风格，以application/json格式进行数据的传递。
本文档中所有的HTTP方法（无论其是GET/POST/DELETE/PUT或任何其他方法），其HTTP HEADER中必须加入OAuth2风格的Http header属性:

~~~
Authorization: Bearer xxxxxxxxxxx
~~~

其中xxxxxxxxxxx为Payzero用于验证调用方身份的JWT风格的token, 该token的获得是通过使用Payzero分配的username和password并调用登录接口。其有效时间为4小时，且JWT本身包含了过期的时间的信息。关于JWT token，可参考 <https://jwt.io/>。4小时到期之前，可重新调用登录接口获取token，token之间均为独立关系，获取新的token不会影响未过期的老token的使用。

### 返回格式说明

调用接口时的返回HTTP STATUS CODE遵循HTTP返回值的通用定义，常见HTTP返回值例如:

|HTTP STATUS CODE|说明|
|:--:|:--|
|200|SUCCESS, 调用成功(仅表示api调用成功，业务是否成功需查看返回的HTTP BODY)|
|400|BAD REQUEST, 通常用于请求的参数不正确|
|405|METHOD NOT ALLOWED, HTTP方法使用错误，例如错误的使用POST调用了一个GET方法|
|401|UNAUTHORIZED, access_token不正确|
|403|FORBIDDEN, 权限不足|
|404|NOT FOUND, 不存在的接口url|
|500|INTERNAL SERVER ERROR, 服务器内部错误|

调用方应先根据HTTP STATUS CODE作为判断，过滤所有HTTP返回值即不成功的情况。

所有调用在HTTP正常调用成功即返回200的情况下，返回的HTTP BODY消息体内容为统一的JSON格式内容如下：

|字段名称|参数|类型|说明|
|:--:|:--:|:--|:--|
|业务层面是否正常完成| success | Boolean | 可以此字段作为标准判定业务调用是否完成，当success为false时，可以从errorMsg读取到错误原因等，errorCode作为参考便于向Payzero反馈问题 |
|业务错误代码| errorCode | String | |
|业务错误信息| errorMsg | String | |
|返回数据 | data | JSON | 具体返回的业务数据，本文后续所有api在"response"部分所述内容，为简便均表示的是data部分的内容 |

一个典型的业务成功的返回例子如下:

~~~
{
    "success": true,
    "errorMsg": null,
    "errorCode": null,
    "data": {
        "merchantId": "13",
        "orderBatchId": "11",
    }
}
~~~

一个典型的业务失败的返回例子如下:

~~~
{
    "success": false,
    "errorMsg": "参数 xxxx 未正确设置。",
    "errorCode": "30001",
    "data": null
}
~~~


## 接口地址 ##
使用如下链接代替本文中出现的{payzero\_api\_url}字样

* 测试环境: https://dev-quickpay-api.payzero.cn
* 生产环境: https://quickpay-api.payzero.cn

## 接口介绍 ##

### 1. 登录获取token ###
* url: {payzero\_api\_url}/auth/login
* method: POST
* request body parameter 

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|用户名 | username| "abc" |  |
|密码 | password| "p@ssw0rd" |  |

* POSTMAN调用示例:
![](doc/auth.jpg)

* response:

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|用户名|username| "admin"|
|token| token| "xxxxxx" | |
~~~
{
    "success": true,
    "errorMsg": null,
    "errorCode": null,
    "data": {
        "username": "admin",
        "token": "eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJhZG1pbiIsInJvbGVzIjpbIlJPTEVfVVNFUiIsIlJPTEVfQURNSU4iXSwiaWF0IjoxNTUzMDY2OTg1LCJleHAiOjE1NTMwODEzODV9.KIfG8LOk2FoMgqXAU8DCg_cbci36OqbLUIHUdIXDsF0"
    }
}
~~~

### 2. 订单相关接口 ###
#### 2.1 订单批次上传 ####
若商户有一个批次的订单需要处理，请先调用本接口创建订单批次。创建完订单批次后，登录商户端后台可查看到订单批次。根据商户是否需要代垫资服务，订单批次的手续费状态可能为"等待支付手续费"(需垫资)或"已确认"(无需代垫资)。若为"等待支付手续费"，需通知相关人员进行财务打款并通知我方确认收款。待订单批次手续费确认后，可进行后续调用。

* url: {payzero\_api\_url}/orderBatch
* method: POST
* request body parameter 

请传入一个数组的order类型对象，order类型的结构如下

|字段名称|参数|类型|是否必填|例子|说明|
|:--|:--|:--|:--|:--|:--|
| 货币代码 | currency | String | 是 | "CNY" | 请固定为CNY |
| 需申报的电子口岸代码 | customsCode | String | 否 | "HG016" | 调用前请咨询相关技术人员。若需申报则为必填。 |
| 海关关区代码| customsAreaCode | String | 否 | "5130" | 若需申报且申报海关为广州海关时必填 |
| 检验检疫机构代码 | customsJyOrg | String | 否 | "440009" |  若需申报且申报为广州海关时必填|
| 进口类型 | customsInType | String | 否 | "1" | 若需申报且申报天津电子口岸时为必填，1-保税进口，2-直邮进口 |
| 商户订单编号 | mchtOrderNo | String| 是 | "2019032000000123" | 请确保商户订单不重复 |
| 订单下单时间 | orderDatetime | Date| 否 | "2019-03-20T06:57:29.396Z" | Date类型 |
| 订购人姓名 | payerName | String | 否 | "张三" | 若需申报则必填 |
| 订购人身份证号 | payerNumber | String | 否 | "310113198010101234" | 若需申报则必填|
| 订购人电话 | payerPhone | String | 否 | "18512001234" | 若需申报则必填 |
| 订单额度 | paymentAmount | Long | 是 | 4023 | 请务必注意单位为分 |
| 订单内主要商品信息 | subject | String | 是 | "XXXX化妆品" | |
| 订单内商品列表 | items | items类型数组 | 否 | | 仅针对需要托管179文对接接口至Payzero的合作伙伴，需要传输该字段，items的结构参见后续说明 |

items类型的结构如下:

|字段名称|参数|类型|是否必填|例子|说明|
|:--|:--|:--|:--|:--|:--|
| 商品名称 | subject | String | 是 | "XXXX口红" | 需179文对接托管客户则必填 |
| 商品链接 | itemLink | String | 是 | "http://www.baidu.com" | 需179文对接托管客户则必填 |
| 货号 | articleNum | String | 否 | "WO11111" |  |


* request example: 

~~~
[
  {
    "currency": "CNY",
    "customsCode": "HG022",
    "customsAreaCode": "5130",
    "customsJyOrg": "440009",
    "customsInType": "2",
    "items": [
      {
        "articleNum": "HH00001",
        "itemLink": "http://www.baidu.com",
        "subject": "测试商品1"
      },
       {
        "articleNum": "HH00002",
        "itemLink": "http://www.baidu.com",
        "subject": "测试商品2"
      }
    ],
    "mchtOrderNo": "F20190402123",
    "orderDatetime": "2019-04-02T11:11:46.740Z",
    "payerName": "李白",
    "payerNumber": "310327198009270027",
    "payerPhone": "13800138000",
    "paymentAmount": 35302,
    "subject": "测试商品1"
  },
  {
    "currency": "CNY",
    "customsCode": "HG022",
    "customsAreaCode": "5130",
    "customsJyOrg": "440009",
    "customsInType": "2",
    "mchtOrderNo": "F20190402124",
    "orderDatetime": "2019-04-02T11:11:46.740Z",
    "payerName": "杜甫",
    "payerNumber": "360104199010101234",
    "payerPhone": "13300000000",
    "paymentAmount": 35000,
    "subject": "测试商品xxxx"
  }
]
~~~

* response:

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|订单批次id| orderBatchId |  59 |
|订单批次号| dispBatchNum | "20190320162344842"|
|批次内订单总额(分)| orderTotalAmount | 70302 | 单位为分 |
|创建时间| createdTime | 1553070224475  | |

~~~
{
    "success": true,
    "errorMsg": null,
    "errorCode": null,
    "data": {
        "orderBatchId": 59,
        "merchantId": 1,
        "dispBatchNum": "20190320162344842",
        "isReconed": false,
        "feeAmount": null,
        "orderTotalAmount": 70302,
        "objectId": null,
        "feedbackObjectId": null,
        "createdTime": 1553070224475,
        "updatedTime": 1553070224817
    }
}
~~~


订单批次上传完成之后，在商户端可以查看到该订单批次信息:

![](doc/ob_screenshot.png)

#### 2.2 订单批次触发支付 ####
调用本接口对本批次内的订单进行支付与推关。（若为垫资模式，需先线下支付手续费）本接口同步返回成功只表明系统接收到开始支付的指令，支付完成时间取决于协商的合作模式等其他因素。在发送支付指令成功之后，可以进行20分钟/次的订单批量批次支付&推送结果轮询。

* url: {payzero\_api\_url}/orderBatch/{orderBatchId}/pay
* method: GET
* request Path Variable: url路径参数为创建该订单批次时返回的{orderBatchId}

* response: success参数返回为true时即表示系统已成功安排本批次支付任务

~~~
{
    "success": true,
    "errorMsg": null,
    "errorCode": null,
    "data": {
       “分配任务完成”
    }
}
~~~

* 错误代码: 若success为false，可能出现的errorCode和其对应解释如下

|errorCode|errorMsg| 备注|
|:--|:--|:--|
|50001| 未找到支持的银行卡和协议，可能与垫资模式的设置不正确有关 | 联系技术支持
|50010| 未找到已签约的银行卡进行支付 | 登录商户后台进行绑卡 |
|50054| 没有需要被分配执行的订单 | 
|50056| 系统记录的银行卡当前额度可能无法支持支付本批次订单，请增加系统内银行卡额度并确保实际资金与额度大致匹配 | 在商户后台中录入的银行卡“当前余额“总额不足以支付当前订单批次，请确保银行卡真是余额足够支付整个订单批次，并在系统中填写入真实余额


#### 2.3 订单批次支付&推送结果汇总查询 ####
用于查询订单批次的支付及推单结果。若支付失败，可重新调用订单批次支付接口，系统将只重新支付当前支付失败的订单，多次反复支付失败，需联系技术人员排查支付失败原因。若推单失败，请运营人员自行登录商户后台修改申报的订购人身份信息，编辑并点击重新推单。

* url: {payzero\_api\_url}/orderBatch/{orderBatchId}/summary
* method: GET
* request Path Variable: url路径参数为创建该订单批次时返回的{orderBatchId}

* response: 

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|批次内订单总数| total |  159 |
|支付成功总数| paySuccessCount |150 | 单位为分 |
|支付失败总数|payFailedCount | 2|
|其他支付中间状态总数|payOtherCount |7  | |
|申报成功总数|declareSuccessCount |130  | |
|申报失败总数|declareFailedCount |7  | |
|其他申报中间状态总数|declareOtherCount | 22  | |
|总已支付金额(分)|totalPaymentAmount | 10489449  | |

~~~
{
    "data": {
        "declareFailedCount": 7,
        "declareOtherCount": 22,
        "declareSuccessCount": 130,
        "payFailedCount": 2,
        "payOtherCount": 7,
        "paySuccessCount": 150,
        "total": 159,
        "totalPaymentAmount": 10489449
    },
    "errorCode": null,
    "errorMsg": null,
    "success": true
}
~~~

#### 2.4 单笔订单回执查询 
用于查询单笔订单的详细状态及所有支付原始信息，供商家进行后续申报

* url: {payzero\_api\_url}/order/feedback?mchtOrderNo={mchtOrderNo}
* method: GET
* request: Request Param {mchtOrderNo}为商家订单号


* response: orderResultDto

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|商家订单号| mchtOrderNo |  2019041000011 |
|支付金额(分)| paymentAmount | 1 | 单位为分 |
|申报支付人姓名| payerName | 周Xx |
|申报支付人身份证号| payerNumber | 610327198509271234  | |
|申报支付人手机号| payerPhone | 13800138000  | |
|交易摘要| subject |YYYYYYYY-口红  | |
|支付状态| payStatus | PAY_SUCCEED  |	参见附录[A.1](#a1-支付状态)|
|申报状态| declareStatus | DECLARE_FAILED  | 参见附录[A.2](#a2-申报状态)|
|申报失败原因| declareFailReason | 支付人姓名和证件号不匹配 | |
|支付单号| paymentOrderNo | 111906650000561884 | |
|核验机构| verDept | null | 验核机构 1-银联 2-网联 3-其他 |
|支付类型| payType | 2 | 用户支付类型 1-APP 2-PC 3-扫码 4-其他 |
|支付原始请求| initRequest | ... | |
|支付原始返回| initResponse | ... | |
|支付完成时间| paymentDatetime | 20190411213822 | 格式为yyyyMMddHHmmss |
|支付公司海关备案名称| customsPayCompanyName | 通联支付网络服务股份有限公司 | |
|支付公司海关备案号| customsPayCompanyCode |  312228034T | |

~~~
{
  "success": true,
  "errorMsg": null,
  "errorCode": null,
  "data": {
    "mchtOrderNo": "2019041000011",
    "paymentAmount": 1,
    "payerName": "周Xx",
    "payerNumber": "610327198509271234",
    "payerPhone": "13800138000",
    "subject": "YYYYYYYY-口红",
    "payStatus": "PAY_SUCCEED",
    "declareStatus": "DECLARE_FAILED",
    "declareFailReason": "支付人姓名和证件号不匹配",
    "paymentOrderNo": "111906650000561884",
    "verDept": null,
    "payType": "2",
    "initRequest": "https://vsp.allinpay.com/apiweb/qpay/payapplyagree[data:{agreeid=201902221443545123, amount=1, appid=00152305, currency=CNY, cusid=55152104816ZLVW, notifyurl=https://dev-quickpay-api.payzero.cn/quickpay_notify/ALLINPAY/pengma, orderid=2019041000011, randomstr=1554989889665, subject=YYYYYYYY-口红, version=11}]",
    "initResponse": "{\"retcode\":\"SUCCESS\",\"retmsg\":null,\"randomstr\":\"917005450382\",\"sign\":\"B74D60CEB7141B3CB26FD4D7A5BD7B29\",\"orderid\":\"2019041000011\",\"trxstatus\":\"1999\",\"errmsg\":\"请输入短信验证码\",\"trxid\":null,\"chnltrxid\":null,\"fintime\":null,\"thpinfo\":\"{\\\"sign\\\":\\\"\\\",\\\"tphtrxcrtime\\\":\\\"\\\",\\\"tphtrxid\\\":0,\\\"trxflag\\\":\\\"trx\\\",\\\"trxsn\\\":\\\"\\\"}\"}",
    "paymentDatetime": "20190411213822",
    "customsPayCompanyName": "通联支付网络服务股份有限公司",
    "customsPayCompanyCode": "312228034T"
  }
}

~~~

#### 2.5 订单批次回执查询

用于批量查询一个订单批次中所有订单的详细状态及所有支付原始信息，供商家进行后续申报。请求需传递每页数据条数和当前的页数，每次最多请求100条数据。数据是按照原订单入库本系统的时间倒序排列。

* url: {payzero\_api\_url}/orderBatch/{orderBatchId}/feedback?pagesize={pagesize}&page={page}
* method: GET
* request: Path Variable url路径参数为创建该订单批次时返回的{orderBatchId}

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|pagesize| 每页数据条数 | 10| 每页数据条数最大不能超过100条 |
|page| 当前为全部数据的第几页| 1 | 数据从第1页开始|


一个请求例子

~~~
curl -X GET "http://127.0.0.1:9040/orderBatch/53/feedback?page=1&pagesize=10" -H "accept: */*" -H "Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJwZW5nbWEiLCJyb2xlcyI6WyJST0xFX1VTRVIiXSwiaWF0IjoxNTU1MTY3MTY5LCJleHAiOjE1NTUxODE1Njl9.EgRhAzPXVryIXMR_OnM-51gus5m0cm1zdjwds764ajk"
~~~

response: 

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|rows| 订单回执数据| ... | 为一个数组的orderResultDto，orderResultDto可参考[2.4](#24-单笔订单回执查询)的返回结果 |
|page| 当前为全部数据的第几页| 1 | 数据从第1页开始|
|pagesize| 每页数据条数 | 10| 每页数据条数最大不能超过100条 |
|total| 总数据条数 | 465 | |

一个每页3条数据，请求第二页的返回例子:

~~~
{
  "success": true,
  "errorMsg": null,
  "errorCode": null,
  "data": {
    "rows": [
      {
        "mchtOrderNo": "BA1812-100937",
        "paymentAmount": 58130,
        "payerName": "赵XX",
        "payerNumber": "21080219930XXXX3X",
        "payerPhone": "1864170XXXX",
        "subject": "Svelty分解酵母120粒",
        "payStatus": "PAY_CANCELLED",
        "declareStatus": "NOT_DECLARED",
        "declareFailReason": null,
        "paymentOrderNo": null,
        "verDept": null,
        "payType": "2",
        "initRequest": null,
        "initResponse": null,
        "paymentDatetime": 20190313124233,
        "customsPayCompanyName": "通联支付网络服务股份有限公司",
        "customsPayCompanyCode": "312228034T"
      },
      {
        "mchtOrderNo": "BA1812-100937-1",
        "paymentAmount": 17162,
        "payerName": "赵XX",
        "payerNumber": "2108021993083XXX3X",
        "payerPhone": "1864170XXXX",
        "subject": "CLAYGE洗发水*1",
        "payStatus": "PAY_CANCELLED",
        "declareStatus": "NOT_DECLARED",
        "declareFailReason": null,
        "paymentOrderNo": null,
        "verDept": null,
        "payType": "2",
        "initRequest": null,
        "initResponse": null,
        "paymentDatetime": 20190313124233,
        "customsPayCompanyName": "通联支付网络服务股份有限公司",
        "customsPayCompanyCode": "312228034T"
      },
      {
        "mchtOrderNo": "BA1812-100953",
        "paymentAmount": 123616,
        "payerName": "龚XX",
        "payerNumber": "33068319870110XXXX",
        "payerPhone": "177173XXXX",
        "subject": "F.O.Online 儿童袜裤",
        "payStatus": "PAY_SUCCEED",
        "declareStatus": "DECLARED",
        "declareFailReason": null,
        "paymentOrderNo": "111906650000530919",
        "verDept": "1",
        "payType": "2",
        "initRequest": null,
        "initResponse": null,
        "paymentDatetime": "20190313124233",
        "customsPayCompanyName": "通联支付网络服务股份有限公司",
        "customsPayCompanyCode": "312228034T"
      }
    ],
    "page": 2,
    "pagesize": 3,
    "total": 465
  }
}
~~~


## 附录

### A.1 支付状态
|状态代码|状态说明|
|:--|:--|
|INIT\_FEE\_PENDING|初始化订单，未进行手续费对账|
|NOT\_PAYED|未支付|
|PAY\_APPLIED|已申请支付|
|PAY\_SMS\_CONFIRMED|支付中间状态|
|PAY\_SUCCEED|支付成功|
|PAY\_FAILED|支付失败|

### A.2 申报状态
|状态代码|状态说明|
|:--|:--|
| NOT\_DECLARED | 未申报|
| PENDING_DECLARE | 待申报 |
| DECLARING | 申报中 |
| DECLARED | 已申报 |
| DECLARE_FAILED | 申报失败 |
	






