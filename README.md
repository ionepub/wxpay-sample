# 微信扫码支付（单商户）

微信支付包括APP支付/js微信应用内支付/扫码支付等方式，官方提供了demo：[https://pay.weixin.qq.com/wiki/doc/api/jsapi.php?chapter=11_1](https://pay.weixin.qq.com/wiki/doc/api/jsapi.php?chapter=11_1)

这里只使用其中的扫码支付，其他的支付方式暂不提，不过使用方法大同小异。

扫码支付顾名思义，就是在页面上显示一个二维码，用户打开微信或者其他扫码工具（比如支付宝或者淘宝，我试过，也可以调起微信支付）进行扫描并支付。

微信提供了两种方式：

### 方法一
1. 调用统一下单接口，获取到微信支付的url（如：weixin://wxpay/bizpayurl?pr=XNY7eDw），使用第三方工具生成二维码显示在页面上
2. 用户扫描二维码，进行支付
3. 支付完成之后，微信服务器会通知支付成功
4. 在支付成功通知中需要查单确认是否真正支付成功

### 方法二
1. 组装包含支付信息的url，生成二维码
2. 用户扫描二维码，进行支付
3. 确定支付之后，微信服务器会回调预先配置的回调地址，在【微信开放平台-微信支付-支付配置】中进行配置
4. 在接到回调通知之后，用户进行统一下单支付，并返回支付信息以完成支付
5. 支付完成之后，微信服务器会通知支付成功
6. 在支付成功通知中需要查单确认是否真正支付成功

能看出来，方法二比方法一要复杂一些，而且需要在微信开放平台做设置，相对比较麻烦。实际上，微信官方也提倡使用第一种方法。（demo中方法一和方法二是反过来的）

![两种模式生成的二维码对比](https://dn-shimo-image.qbox.me/v5NoETZThqQ8xE40.png!thumbnail)

对比上面两个二维码（模式一对应方法二）也可以知道，方法一生成的二维码要简单地多，这也意味着扫码更加容易。

说一下方法一怎么使用。因为有demo，所以使用起来还是很方便的。

首先需要在`./lib/Wxpay.Config.php` 中修改相应的参数，将自己的商户信息填写进去。

    说明一下，这里只使用扫码支付，所以只需要使用APPID，MCHID和KEY即可，其他参数用不到（暂不涉及退款，所以证书也不需要）

```php
<?php

  const APPID = 'wx426b3015555a46be'; //APPID：绑定支付的APPID（必须配置，开户邮件中可查看）
	const MCHID = '1225312702';         //MCHID：商户号（必须配置，开户邮件中可查看）
	const KEY = 'e10adc3949ba59abbe56e057f20f883e';  //KEY：商户支付密钥，参考开户邮件设置（必须配置，登录商户平台自行设置），设置地址：https://pay.weixin.qq.com/index.php/account/api_cert
	const APPSECRET = '';     //APPSECRET：公众帐号secert（仅JSAPI支付的时候需要配置， 登录公众平台，进入开发者中心可设置），
	
	const SSLCERT_PATH = '';
	const SSLKEY_PATH = '';

?>
```

微信统一下单文档：https://pay.weixin.qq.com/wiki/doc/api/jsapi.php?chapter=9_1

在example/native.php中可以找到：

```php
<?php
ini_set('date.timezone','Asia/Shanghai');

require_once "../lib/WxPay.Api.php";
require_once "WxPay.NativePay.php";

require_once 'log.php';  //日志记录文件，个人建议使用比较好，主要是方便查询，如果某个订单出了问题，可以从中得到一些帮助

$input = new WxPayUnifiedOrder();  //开始统一下单
$input->SetBody("test");    //商品或支付单简要描述
$input->SetAttach("test");    //附加数据，会原样返回
$input->SetOut_trade_no(WxPayConfig::MCHID.date("YmdHis"));    //订单号，商家自己的
$input->SetTotal_fee("1");             //付款金额，以分做单位的，1就是1分。100就是一块钱
$input->SetTime_start(date("YmdHis"));            //订单生成时间，必须是这种时间格式的
$input->SetTime_expire(date("YmdHis", time() + 600));   //订单失效时间，也必须是这种格式的，而且不能比订单生成时间少600秒（5分钟）
//$input->SetGoods_tag("test");   //代金券或立减优惠功能的参数，暂时用不到
$input->SetNotify_url("http://paysdk.weixin.qq.com/example/notify.php");   //支付完成的回调地址，这个是异步的回调，地址中不能有参数，如?mod=wxpay
$input->SetTrade_type("NATIVE");    //交易类型，固定填NATIVE
$input->SetProduct_id("123456789");   //商品id，必须要传递给微信的参数

$notify = new NativePay();
$result = $notify->GetPayUrl($input);
$url = $result["code_url"];  //这个就是获取到的待生成二维码的url

?>
```

    需要注意的不多，一个是订单的失效时间可以不传，但是如果传了，就需要比订单的生成时间至少多600秒，另外就是支付的回调地址url中不能带参数（pathinfo类型的应该没有问题），还有就是商品id记得传

    关于这个接口的调用频率文档里貌似没有提到，不过如果需要的话，也可以在cookie或者session里存放生成的url，一定程度上可以防止用户恶意刷新

### 微信支付回调

由于是扫码支付，支付的回调只能通过异步来完成

微信的回调示例代码在 example/notify.php里

```php
        //重写回调处理函数
	public function NotifyProcess($data, &$msg)
	{
		Log::DEBUG("call back:" . json_encode($data));
		$notfiyOutput = array();
		
		if(!array_key_exists("transaction_id", $data)){
			$msg = "输入参数不正确";
			return false;
		}
		//查询订单，判断订单真实性
		if(!$this->Queryorder($data["transaction_id"])){
			$msg = "订单查询失败";
			return false;
		}
		//可以在这里写自己的支付完成逻辑
		return true;
	}
```

仅仅是扫码支付，所以相对而已会简单很多，我自己在项目中是这样处理的：

1. 使用curl将订单信息post给native.php，native.php接收到数据之后，向微信发起统一下单，从而获得微信支付链接；
2. 使用js二维码生成库将链接转换成二维码（微信官方提供的是phpqrcode这个库，对于我这个项目来说，这个库太大了，光文件就几十上百个，而js的话就只是一个文件而已）
3. 用户支付完成之后回调，在NotifyProcess中同样通过curl将订单支付信息post到项目中，之后对订单进行付款处理

		这样做有一个好处，这样的代码相当于对外有接口，只要用约定的方法调用，可以直接把整个文件夹复制到多个项目重用。


