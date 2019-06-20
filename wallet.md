# 钱包接入

## 一、移动端钱包 UltrainOne

UltrainOne是用ReactNative实现的钱包综合APP, 其中UltrainOne的DAPP应用市场模块汇集了若干商家应用，针对html5实现的第三方web应用，UltrainOne采用Webview的形式直接打开或产生数据交互。

#### DAPP获取账户或用户信息

DAPP上架DAPP市场后，管理员会根据产品需求设置进入该DAPP时是否需要保证钱包状态可用。
UltrainOne会在点击DAPP入口时做账号检测，如果用户钱包中没有任何账号，则会拦截并定向到钱包账号创建或导入的页面。
因此用户在请求第三方DAPP的URL时，一定是有账号信息的。如果设置成不需要保证钱包状态可用，则可以直接进入DAPP。

从UltrainOne进入到DAPP时，会默认在DAPP提供的URL的后面拼接上用户ID、用户手机号和账户名，格式如：
https://dapp_xxxx?chainId=rX9r2wf&userId=Xjudda12&phoneNum=008615857169999&accountName=abcdefg12345
注意：上述四个参数都具有唯一性。

如果DAPP需要获取用户的头像、邮箱、姓名等其它信息，则需要通过单独的授权接口请求。

#### DAPP唤起UltrainOne转账

DAPP在html5中通过window.postMessage接口向UltrainOne的Webview发送数据，
UltrainOne接收到付款请求的数据后，唤起app付款界面确认，构建转账逻辑，由用户自行签名并完成付款，
付款完成后并通过webview.postMessage接口向html5发送回执消息。

DAPP通过window.postMessage(data)发送的data格式如下：
{
    "chainId": "HJiRph6xN",
    "contract": "benyasin1112",
    "action": "transfer",
    "bizId": "86534135672411",              //业务id
    "data": {
      "payer": "zkxhappy4321",              //用户付款账号
      "receiver": "benyasin1112",           //商户账号
      "quantity": "1.0000 BGGG",            //数量及单位
      "memo": "test"
    }
}

UltrainOne通过webview.postMessage(data)发送给第三方DAPP html5的回执消息格式如下：
{
    "bizId": "86534135672411",              //业务id
    "success": true                         //业务执行结果
    "msg": "",                              //消息，成功时为空，失败时有具体原因
}

注意：如果DAPP重复发送相同bizId的请求，UltrainOne会忽略，不做处理。


## 二、桌面端插件钱包 Cona

Cona是一款基于浏览器插件的超脑链轻钱包，涵盖转账、收款、账号同步及授权认证等功能，可以让你在浏览器环境
中运行超脑链的DApp。

#### 检查安装

会在浏览器注入window.Cona对象，检测window.Cona如果存在则表示用户有安装。

Cona在线安装地址为 https://chrome.google.com/webstore/detail/cona/joopmnkobcdaojgcmohnjhloldhfgfgk

Cona离线下载地址为 https://ultrain-cona.oss-cn-hangzhou.aliyuncs.com/cona.crx

window.addEventListener('load', function () {
  if (typeof window.Cona !== 'undefined') {
    console.log('Cona is enabled')
  } else {
    console.log('Cona is not installed')
  }
})

####  发起交易

Cona.send(params)

 const to = 'utest1';
    const contract = 'utrio.token';
    const quantity = 10;
    const symbol = 'UGAS';
    const memo = 'transfer 10.0000 UGAS';
    window.Cona.send({ to, contract, quantity, symbol, memo }).then((trx) => {
      // trx为链上返回的交易详情，需要通过u3轮询交易来确认交易结果
      console.log(trx);
    }).catch((e) => {
      // 处理异常
      console.log(e);
});

> params参数介绍如下所示：
> to: 接收账户，类型为字符串
> contract: 交易的合约名称，类型为字符串
> quantity: 转账金额，精度请按代币约定填写，如果超出代币的精度则将会被四舍五入，例如UGAS的精度是0.0001，如果传入10.66666，则会被四舍五入为10.6667
> symbol: 代币单位
> memo: 交易描述文字，可以不填写

返回值：方法是一个异步方法，返回一个promise对象，可以在then中获取到链上返回的交易信息。在catch中捕获异常。

注意，为确保send方法能够被正常调起，请首先安装Cona插件，选择合适的网络进行登录，并导入至少一个钱包。

