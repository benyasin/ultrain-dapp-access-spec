# 钱包接入

## 一、移动端钱包 UltrainOne

UltrainOne是用ReactNative实现的钱包综合APP, 其中UltrainOne的DAPP应用市场汇集了若干第三方应用，
针对html5实现的第三方web应用，UltrainOne采用Webview的形式直接打开应用并与产生数据交互。

<img width="50%" src="https://user-images.githubusercontent.com/1866848/60152198-2939d500-9812-11e9-96d9-8c4a058f0197.jpeg">

UltrainOne可以从苹果商店、谷歌商店、小米或华为应用市场以及[Ultrain官网](https://ultrain.io/)下载。

#### DAPP获取账户或用户信息

DAPP上架UltrainOne的DAPP市场时，管理员会根据DAPP是否接入钱包来配置用户进入DAPP时是否需要进行钱包状态的检查。
如果是纯展示类的DAPP,则可以不需要拦截，直接进入DAPP页面。
如果是接入了钱包的，UltrainOne会在点击DAPP入口时做钱包状态检测，钱包不可用时会被拦截到钱包账号创建或导入页面。
因此从UltrainOne的DAPP入口经WebView访问DAPP时，跳转的URL中是有含有钱包账户、用户手机等信息的。

从UltrainOne进入到DAPP时，会默认在DAPP提供的URL的后面拼接上用户ID、用户手机号和账户名，格式如：
https://dapp_xxxx?chainId=rX9r2wf&userId=Xjudda12&phoneNum=008615857169999&accountName=abcdefg12345
注意：上述四个参数都具有唯一性。

如果DAPP需要获取用户的头像、邮箱、姓名等其它信息，则需要通过单独的授权接口请求。开发者在个人中心添加DAPP时，需要选择对应的数据请求接口做授权。
相关操作流程请参考[流程规范]一章。

#### DAPP唤起UltrainOne转账

DAPP在html5中通过window.postMessage接口向UltrainOne的Webview发送数据，
UltrainOne接收到付款请求的数据后，唤起app付款界面确认，构建转账逻辑，由用户自行签名并完成付款，
付款完成后并通过webview.postMessage接口向html5发送回执消息。

DAPP通过window.postMessage(data)发送的data格式如下：

```
{
    "chainId": "HJiRph6xN",                 //[必填],链ID,从url的参数中获取后回填至此
    "contract": "benyasin1112",             //[必填],如果转账UGAS,则值为"utrio.token"，否则值为具体的发币合约的owner账号
    "action": "transfer",                   //[必填],转账业务，值为固定的值"transfer"
    "type": "transfer",                     //[必填],转账业务的固定值为"transfer"
    "bizId": "86534135672411",              //[必填],业务id,用来保证同一业务不会重复转账
    "data": {
      "receiver": "benyasin1112",           //[必填],收款账号，一般为商家的账号
      "quantity": "1.50 BGGG",              //[必填],数量及单位，如果是UGAS,则比如"100.0000 UGAS"
      "memo": "test"                        //[必填],值可以空
    }
}
```

UltrainOne通过webview.postMessage(data)发送给第三方DAPP html5的回执消息格式如下：

```
{
    "bizId": "86534135672411",              //业务id
    "success": true                         //业务执行结果
    "msg": "",                              //消息，成功时为空，失败时有具体原因
}

```

注意：如果DAPP重复发送相同bizId的请求，UltrainOne会忽略，不做处理。


#### DAPP唤起UltrainOne调用合约方法（非转账类）

UltrainOne针对非转账类的合约方法调用提供一个通用的接口。
DAPP在html5中通过window.postMessage接口向UltrainOne的Webview发送数据，
UltrainOne接收到调用合约方法的请求后，唤起app的合约方法调用界面进行确认，构建方法调用逻辑，由用户自行签名并完成方法调用，
方法调用完成后并通过webview.postMessage接口向html5发送回执消息。

DAPP通过window.postMessage(data)发送的data格式如下：

```
{
    "chainId": "HJiRph6xN",                 //[必填],链ID,从url的参数中获取后回填至此
    "contract": "benyasin1112",             //[必填],值为具体合约的owner账号, 比如"ben"
    "action": "doAction",                   //[必填],值为合约中具体的某个方法名，比如"doAction"
    "type": "contract",                     //[必填],合约调用的固定值为"contract"
    "bizId": "86534135672411",              //[必填],业务id,用来保证同一业务不会重复转账
    "data": {                               //[必填],data固定，其内容为合约中方法的具体入参，以下仅为举例
      "name": "bob",                        
      "age": 30,                            
      "msg": ""                             
    }
}
```

UltrainOne通过webview.postMessage(data)发送给第三方DAPP html5的回执消息格式如下：

```
{
    "bizId": "86534135672411",              //业务id
    "success": true                         //业务执行结果
    "msg": "",                              //消息，成功时为空，失败时有具体原因
}

```

注意：调用合约中的某个方法，用法上类型于转账，除了type参数为contract不同之外，data中的key也是不固定，
data的具体内容取决于调用的方法的入参，依次罗列即可，所有参数不能缺失，即使值为空，也要保证有这个Key。
如果DAPP重复发送相同bizId的请求，UltrainOne会忽略，不做处理。

以下给出一个Vue编写的集成UltrainOne的h5示例。
```
<template>
  <div style="text-align: center;margin: 50px 0; font-size: 26px">
    <button v-on:click="handleTransferClick">调钱包转账接口</button>
    <p style="text-align: center">收到UltrainOne发送的数据: <span id="data">{{data}}</span></p>
  </div>

  <div style="text-align: center;margin: 100px 0; font-size: 26px">
    <button v-on:click="handleContractClick">调合约某个接口</button>
    <p style="text-align: center">收到UltrainOne发送的数据: <span id="data2">{{data}}</span></p>
  </div>
</template>
<script>

  export default {
    data() {
      return {
        accountName: '',
        chainId: '',
        data: '...',
      };
    },
    mounted() {
      document.addEventListener('message', (e) => {
        this.data = e.data;
        alert(this.data);
      });

      this.accountName = this.$route.query.accountName;
      this.chainId = this.$route.query.chainId;
    },
    methods: {
      handleTransferClick() {
        this.sendData(JSON.stringify({
          'chainId': this.chainId,
          'contract': 'ultrainpoint',
          'action': 'transfer',
          'type': 'transfer',
          'bizId': Math.random() * 10000,
          'data': {
            'receiver': 'benyasin1',           //商户账号
            'quantity': '1 UPOINT',            //数量及单位
            'memo': 'test',
          },
        }));
      },

      handleContractClick() {
        this.sendData(JSON.stringify({
          'chainId': this.chainId,
          'contract': 'ben',
          'action': 'hi',
          'type': 'contract',
          'bizId': Math.random() * 10000,
          'data': {
            'name': this.accountName,
            'age': 32,
            'msg': 'hello',
          },
        }));
      },

      sendData(data) {
        if (window.postMessage) {
          console.log('sending data to webview...', data);
          window.postMessage(data);
        } else {
          throw Error('postMessage接口还未注入');
        }
      },
    },
  };
</script>



```

## 二、桌面端插件钱包 Cona

Cona是一款基于浏览器插件的超脑链轻钱包，涵盖转账、收款、账号同步、连接与授权认证等功能，可以让你在浏览器环境
中运行超脑链的DApp。

<img width="50%" src="https://user-images.githubusercontent.com/1866848/60158890-8d659480-9824-11e9-8a46-f97599600b08.jpg">

#### 检查安装

Cona会在浏览器注入window.Cona对象，检测window.Cona如果存在则表示用户有安装。

Cona在线安装地址为 https://chrome.google.com/webstore/detail/cona/joopmnkobcdaojgcmohnjhloldhfgfgk

Cona离线下载地址为 https://ultrain-cona.oss-cn-hangzhou.aliyuncs.com/cona.crx

```
window.addEventListener('load', function () {
  if (typeof window.Cona !== 'undefined') {
    console.log('Cona is enabled')
  } else {
    console.log('Cona is not installed')
  }
})

```

#### 连接Cona

DAPP接入Cona后，用户在DAPP中使用钱包时，需要先进行一个"连接"确认，确认后Cona会将钱包中默认账户的账户名与公钥信息返回给DAPP。
经用户确认后，DAPP才可以继续使用Cona进行转账与鉴权等操作。

Cona.connectRequest() 

返回信息格式为

```


```

#### 授权认证

Cona提供了一个针对账户的线下授权功能，类似于Google的第三方登录授权。经用户主动授权后，Cona会对授权账户进行签名与验签，确认账户信息合法后，
返回授权账户的账户名与公钥信息给DAPP。

Cona.authenticate()

返回信息格式为

```


```


####  发起交易

DAPP内部需要发起转账时，可唤起Cona钱包的转账界面，经用户手动转账。

Cona.send(params)

```
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

```


> params参数介绍如下所示：
> to: 接收账户，类型为字符串
> contract: 交易的合约名称，类型为字符串
> quantity: 转账金额，精度请按代币约定填写，如果超出代币的精度则将会被四舍五入，例如UGAS的精度是0.0001，如果传入10.66666，则会被四舍五入为10.6667
> symbol: 代币单位
> memo: 交易描述文字，可以不填写

返回值：方法是一个异步方法，返回一个promise对象，可以在then中获取到链上返回的交易信息。在catch中捕获异常。

注意，为确保send方法能够被正常调起，请首先安装Cona插件，选择合适的网络进行登录，并导入至少一个钱包。

