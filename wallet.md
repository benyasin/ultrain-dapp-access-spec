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

Cona离线下载地址为 https://ultrain-cona.oss-cn-hangzhou.aliyuncs.com/cona.crx.zip

离线包安装方法：

>1.解压出cona.crx文件；
>
>2.打开Chrome，在URL中输入 chrome://extensions,同时打开右上角的开发者模式；
>
>3.将cona.crx拖拽到该网页中间进行添加；

<img width="80%" src="https://user-images.githubusercontent.com/1866848/61033042-34395b80-a3f5-11e9-8b64-df6f7d65d383.png">


检查是否成功安装的方法如下：

```
window.addEventListener('load', function () {
  if (typeof window.Cona !== 'undefined') {
    console.log('Cona is enabled')
  } else {
    console.log('Cona is not installed')
  }
})

```


#### 注意事项

DAPP需要向Cona提供唯一性身份信息。名称、域名和Logo。
除域名外，名称与Logo可以在HTML的head中指定。格式如下：

```
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8"/>
  <link rel="shortcut icon" href="https://xxxx/logo.jpg">
  <title>DAPP Name</title>
  ...

```

#### 连接请求

DAPP接入Cona后，用户在DAPP中使用钱包进行交互时，需要先进行一个"连接"确认。此时，Cona会记录下该DAPP的唯一标识，类似于添加到白名单。
经用户确认后，DAPP才可以继续使用Cona进行其它一切操作。方法是一个异步方法，返回一个promise对象，可以在then中获取到链上返回的交易信息。在catch中捕获异常。

Cona.connectRequest() 


调用格式为

```

const rs = await window.Cona.connectRequest();

```


返回信息格式为

```
{
    action: "connect_request",
    inpage: true,
    sender_id: 2585,
    status: "success"
}

```

#### 授权认证

Cona提供了一个针对账户的授权认证功能，类似于Google的第三方登录授权。该认证结果不会上链，是Cona提供的线下签名。
经用户主动授权后，Cona会使用当前账户进行签名与验签，并将账户的账户名与公钥信息返回给DAPP。
方法是一个异步方法，返回一个promise对象，可以在then中获取到链上返回的交易信息。在catch中捕获异常。

Cona.sign()

调用格式为

```

const rs = await window.Cona.sign();

```

返回信息格式为

```
{
    action: "sign",
    data: {
        nonce: 4973, 
        signature: "SIG_K1_JyR7vCNDbUu4eSPaEbSdrZFJy28tBBACYXGUwz1FE1b…UrEdywWaGPsf5cm41mHyd3TAEys99rqoFv2rXgqQ3hNEi68pe", 
        account_name: "cona2", 
        public_key: "UTR5mPDnuuAgAaMd8Nfq28qZdJnvjdV2jqBGsjNvtPWqToq5wY5gE",
    },
    error: null,
    inpage: true,
    sender_id: 2585,
    status: "success"
}

```


####  发起交易

DAPP内部需要发起转账时，可唤起Cona钱包的转账界面，经用户手动转账。如果转账的是UGAS,则contract的值为固定值utrio.token，否则为具体的代币发行owner账号。
send返回交易信息，但不保证最终会被打包确认。需要用户通过轮询该交易来确定最终确认状态。
方法是一个异步方法，返回一个promise对象，可以在then中获取到链上返回的交易信息。在catch中捕获异常。

Cona.send(params)

参数为：
* **to**   收款账号
* **contract**  发币合约的owner账号,如果转UGAS，则值为utrio.token
* **quantity**  转账金额，精度请按代币约定填写，如果超出代币的精度则将会被四舍五入，例如UGAS的精度是0.0001，如果传入10.66666，则会被四舍五入为10.6667
* **memo**  备注，可空
* **symbol**  符号，如果UGAS

调用格式为

```
    const to = 'utest1';
    const contract = 'utrio.token'; //合约Owner账户，如果转账的是UGAS,则为固定值utrio.token
    const quantity = 10;
    const symbol = 'UGAS';
    const memo = 'transfer 10.0000 UGAS';
    window.Cona.send({ to, contract, quantity, symbol, memo }).then((trx) => {
      // trx为链上返回的交易详情，需要通过u3轮询交易来确认交易结果
      console.log(trx);
      
      // 以下为检查交易最终是否确认的过程
      // 首先检查交易执行状态有没有失败
      if (!trx || trx.processed.receipt.status !== "executed") {
          console.log("the transaction was failed");
          return;
      }
    
      // 然后检查在过期之前有没有置为不可回滚
      let timeout = new Date(trx.transaction.transaction.expiration + "Z") - new Date();
      let finalResult = false;
      try {
          await U3Utils.test.waitUntil(async () => {
            let tx = await u3.getTxByTxId(trx.transaction_id);
            finalResult = tx && tx.irreversible;
            if (finalResult) return true;
          }, timeout, 1000);
      } catch (e) {
          console.log(finalResult);
      }
            
    }).catch((e) => {
      // 处理异常
      console.log(e);
});

```
返回格式为

```
{
    "action": "send",
    "status": "success",
    "data": {
        "transaction_id": "a97ee837185036b017ed0e011b3db48b4df59015bf44ee8eba5eca541deb1531",
        "processed": {
            "id": "a97ee837185036b017ed0e011b3db48b4df59015bf44ee8eba5eca541deb1531",
            "receipt": {
                "status": "executed",
                "cpu_usage_us": 3996,
                "net_usage_words": 17
            },
            "elapsed": 3996,
            "net_usage": 136,
            "scheduled": false,
            "action_traces": [
                {
                    "receipt": {
                        "receiver": "ultrainpoint",
                        "act_digest": "5ac71c493603800aff262fc6534edf750caaa8ab1b5d744926035499c1e74c09",
                        "global_sequence": 1314168,
                        "recv_sequence": 54,
                        "auth_sequence": [
                            [
                                "cona1",
                                61
                            ]
                        ],
                        "code_sequence": 1,
                        "abi_sequence": 1
                    },
                    "act": {
                        "account": "ultrainpoint",
                        "name": "transfer",
                        "authorization": [
                            {
                                "actor": "cona1",
                                "permission": "active"
                            }
                        ],
                        "data": {
                            "from": "cona1",
                            "to": "cona2",
                            "quantity": "1 UPOINT",
                            "memo": ""
                        },
                        "hex_data": "0000000080602645000000000061264501000000000000000055504f494e540000"
                    },
                    "elapsed": 1523,
                    "cpu_usage": 0,
                    "console": "",
                    "total_cpu_usage": 0,
                    "trx_id": "a97ee837185036b017ed0e011b3db48b4df59015bf44ee8eba5eca541deb1531",
                    "return_value": "",
                    "inline_traces": [
                        {
                            "receipt": {
                                "receiver": "cona1",
                                "act_digest": "5ac71c493603800aff262fc6534edf750caaa8ab1b5d744926035499c1e74c09",
                                "global_sequence": 1314169,
                                "recv_sequence": 94,
                                "auth_sequence": [
                                    [
                                        "cona1",
                                        62
                                    ]
                                ],
                                "code_sequence": 1,
                                "abi_sequence": 1
                            },
                            "act": {
                                "account": "ultrainpoint",
                                "name": "transfer",
                                "authorization": [
                                    {
                                        "actor": "cona1",
                                        "permission": "active"
                                    }
                                ],
                                "data": {
                                    "from": "cona1",
                                    "to": "cona2",
                                    "quantity": "1 UPOINT",
                                    "memo": ""
                                },
                                "hex_data": "0000000080602645000000000061264501000000000000000055504f494e540000"
                            },
                            "elapsed": 1088,
                            "cpu_usage": 0,
                            "console": "",
                            "total_cpu_usage": 0,
                            "trx_id": "a97ee837185036b017ed0e011b3db48b4df59015bf44ee8eba5eca541deb1531",
                            "return_value": "",
                            "inline_traces": []
                        },
                        {
                            "receipt": {
                                "receiver": "cona2",
                                "act_digest": "5ac71c493603800aff262fc6534edf750caaa8ab1b5d744926035499c1e74c09",
                                "global_sequence": 1314170,
                                "recv_sequence": 78,
                                "auth_sequence": [
                                    [
                                        "cona1",
                                        63
                                    ]
                                ],
                                "code_sequence": 1,
                                "abi_sequence": 1
                            },
                            "act": {
                                "account": "ultrainpoint",
                                "name": "transfer",
                                "authorization": [
                                    {
                                        "actor": "cona1",
                                        "permission": "active"
                                    }
                                ],
                                "data": {
                                    "from": "cona1",
                                    "to": "cona2",
                                    "quantity": "1 UPOINT",
                                    "memo": ""
                                },
                                "hex_data": "0000000080602645000000000061264501000000000000000055504f494e540000"
                            },
                            "elapsed": 996,
                            "cpu_usage": 0,
                            "console": "",
                            "total_cpu_usage": 0,
                            "trx_id": "a97ee837185036b017ed0e011b3db48b4df59015bf44ee8eba5eca541deb1531",
                            "return_value": "",
                            "inline_traces": []
                        }
                    ]
                }
            ],
            "except": null,
            "ability": "Normal"
        },
        "broadcast": true,
        "transaction": {
            "compression": "none",
            "transaction": {
                "expiration": "2019-07-11T07:12:15",
                "ref_block_num": 53687,
                "ref_block_prefix": 529344676,
                "net_usage_words": 0,
                "max_cpu_usage_ms": 0,
                "delay_sec": 0,
                "context_free_actions": [],
                "actions": [
                    {
                        "account": "ultrainpoint",
                        "name": "transfer",
                        "authorization": [
                            {
                                "actor": "cona1",
                                "permission": "active"
                            }
                        ],
                        "data": "0000000080602645000000000061264501000000000000000055504f494e540000"
                    }
                ],
                "transaction_extensions": []
            },
            "signatures": [
                "SIG_K1_Ki12M9wGLdxbyZHmEY45b85imnE4L2ccZMAfWPzmR6Qz6nBtgG5bq27xP4pzykdvaXiL8BKK2Gx8XA9mcV8MdfVjZCWdiY"
            ]
        },
        "httpEndpoint": "http://pioneer.natapp1.cc",
        "httpEndpointHistory": "http://pioneer-history.natapp1.cc",
        "chainId": "20c35b993c10b5ea1007014857bb2b8832fb8ae22e9dcfdc61dacf336af4450f",
        "name": "11",
        "network": "TestNet",
        "locale": {
            "zh-CN": "测试网先锋链",
            "en": "Testnet Pioneer"
        }
    },
    "error": null,
    "sender_id": 2956,
    "inpage": true
}

```

####  获取链列表

DAPP在调用合约之前，需要指定该合约所在的链，该方法提供了获取可用的链列表。如果不指定参数，默认取的用户Cona钱包当前的网络设置对应的链列表，否则返回
指定网络（主网或测试网）的链列表。
方法是一个异步方法，返回一个promise对象，可以在then中获取到链上返回的交易信息。在catch中捕获异常。

Cona.getCurrentNetworkChains()

参数为：

* **network** 网络类型[可选]，格式为字符串，值为""、"MainNet"、"TestNet"中的任意一个

调用格式为：

```
let result = Cona.getCurrentNetworkChains('MainNet');
if(result.success){
   console.log(result.data)
}

```


返回格式为：

```
{
    data: [],
    inpage: true,
    msg: "Cona is not available, please install it and import an account at least",
    success: false
}

或

{
    data: [],
    inpage: true,
    msg: "Please change Cona's network setting to MainNet",
    success: false
}

或

{
    "success": true,
    "msg": "",
    "data": [
        {
            "locale": {
                "zh-CN": "测试网主链",
                "en": "Testnet MainChain"
            },
            "httpEndpoint": "http://ultrain.natapp1.cc",
            "httpEndpointHistory": "http://ultrain-history.natapp1.cc",
            "network": "TestNet",
            "isSideChain": false,
            "_id": "psnW5_1sQ",
            "name": "ultrainio",
            "chainId": "1f1155433d9097e0f67de63a48369916da91f19cb1feff6ba8eca2e5d978a2b2"
        },
        {
            "locale": {
                "zh-CN": "测试网先锋链",
                "en": "Testnet Pioneer"
            },
            "httpEndpoint": "http://pioneer.natapp1.cc",
            "httpEndpointHistory": "http://pioneer-history.natapp1.cc",
            "network": "TestNet",
            "isSideChain": true,
            "_id": "M2WL3lbih",
            "name": "11",
            "chainId": "20c35b993c10b5ea1007014857bb2b8832fb8ae22e9dcfdc61dacf336af4450f"
        },
        {
            "locale": {
                "zh-CN": "测试网动力链",
                "en": "Testnet Power"
            },
            "httpEndpoint": "http://power.natapp1.cc",
            "httpEndpointHistory": "http://power-history.natapp1.cc",
            "network": "TestNet",
            "isSideChain": true,
            "_id": "2hNhi3NqT",
            "name": "12",
            "chainId": "0120d06d4a73b60357a5ed24a9145c967308738d70397c25eeedcbb736166ccf"
        }
    ],
    "inpage": true
}

```

####  调用合约

DAPP对智能合约的调用，可以使用以下方法来完成，主要是针对非转账性操作。交易的确认过程与转账是一致的。

Cona.callContract(params)

参数为：
* **contract**  合约owner的账号
* **action**    合约中的action
* **params**    合约中action的入参
* **chainId**   该合约部署所在的链id

调用格式为

```

    const contract = 'cona1';
    const action = 'hi';
    const params = ['ben', 32, 'test'];
    const result = await window.Cona.getCurrentNetworkChains();
    if(result.success){
        //...决定该合约所在链的id
        const tx = await window.Cona.callContract({ contract, action, params, chainId: result.data[1]._id });
        console.log(tx)
    }else{
        console.log(result.msg)
    }

```
返回格式为

```
{
    "action": "call_contract",
    "status": "success",
    "data": {
        "transaction_id": "4cf67a231a80b7cb96d618796dcd04260445b0842fa4d33924f33df4b3800229",
        "processed": {
            "id": "4cf67a231a80b7cb96d618796dcd04260445b0842fa4d33924f33df4b3800229",
            "receipt": {
                "status": "executed",
                "cpu_usage_us": 2437,
                "net_usage_words": 15
            },
            "elapsed": 2437,
            "net_usage": 120,
            "scheduled": false,
            "action_traces": [
                {
                    "receipt": {
                        "receiver": "cona1",
                        "act_digest": "cb607cbeb9d5bbd2ea7d7cb8a6907f6cbdda50218275f4201a524089b61a3897",
                        "global_sequence": 1314281,
                        "recv_sequence": 95,
                        "auth_sequence": [
                            [
                                "cona1",
                                64
                            ]
                        ],
                        "code_sequence": 4,
                        "abi_sequence": 4
                    },
                    "act": {
                        "account": "cona1",
                        "name": "hi",
                        "authorization": [
                            {
                                "actor": "cona1",
                                "permission": "active"
                            }
                        ],
                        "data": {
                            "name": "test22222224",
                            "age": 32,
                            "msg": "memo"
                        },
                        "hex_data": "408410420891b1ca20000000046d656d6f"
                    },
                    "elapsed": 2054,
                    "cpu_usage": 0,
                    "console": "hello, name = test22222224 age = 32 msg = memo",
                    "total_cpu_usage": 0,
                    "trx_id": "4cf67a231a80b7cb96d618796dcd04260445b0842fa4d33924f33df4b3800229",
                    "return_value": "call hi() succeed.",
                    "inline_traces": []
                }
            ],
            "except": null,
            "ability": "Normal"
        },
        "broadcast": true,
        "transaction": {
            "compression": "none",
            "transaction": {
                "expiration": "2019-07-11T07:19:35",
                "ref_block_num": 53731,
                "ref_block_prefix": 1480805400,
                "net_usage_words": 0,
                "max_cpu_usage_ms": 0,
                "delay_sec": 0,
                "context_free_actions": [],
                "actions": [
                    {
                        "account": "cona1",
                        "name": "hi",
                        "authorization": [
                            {
                                "actor": "cona1",
                                "permission": "active"
                            }
                        ],
                        "data": "408410420891b1ca20000000046d656d6f"
                    }
                ],
                "transaction_extensions": []
            },
            "signatures": [
                "SIG_K1_Kd8nEw7hwEH5w6sFAwgXjtkMxzntVhMcp2GGjnT8evdzQw2cGUGsHrtvdyGB1ksKRqsq4YJVB2fAhAc9KhCE2rf9YdcWeG"
            ]
        },
        "name": "TestNet",
        "httpEndpoint": "http://ultrain.natapp1.cc",
        "httpEndpointHistory": "http://ultrain-history.natapp1.cc",
        "chainId": "1f1155433d9097e0f67de63a48369916da91f19cb1feff6ba8eca2e5d978a2b2",
        "symbol": "UGAS",
        "locale": {
            "zh-CN": "测试网",
            "en": "TestNet"
        },
        "createUser": true,
        "chains": [
            {
                "locale": {
                    "zh-CN": "测试网主链",
                    "en": "Testnet MainChain"
                },
                "httpEndpoint": "http://ultrain.natapp1.cc",
                "httpEndpointHistory": "http://ultrain-history.natapp1.cc",
                "network": "TestNet",
                "isSideChain": false,
                "_id": "psnW5_1sQ",
                "name": "ultrainio",
                "chainId": "1f1155433d9097e0f67de63a48369916da91f19cb1feff6ba8eca2e5d978a2b2"
            },
            {
                "locale": {
                    "zh-CN": "测试网先锋链",
                    "en": "Testnet Pioneer"
                },
                "httpEndpoint": "http://pioneer.natapp1.cc",
                "httpEndpointHistory": "http://pioneer-history.natapp1.cc",
                "network": "TestNet",
                "isSideChain": true,
                "_id": "M2WL3lbih",
                "name": "11",
                "chainId": "20c35b993c10b5ea1007014857bb2b8832fb8ae22e9dcfdc61dacf336af4450f"
            },
            {
                "locale": {
                    "zh-CN": "测试网动力链",
                    "en": "Testnet Power"
                },
                "httpEndpoint": "http://power.natapp1.cc",
                "httpEndpointHistory": "http://power-history.natapp1.cc",
                "network": "TestNet",
                "isSideChain": true,
                "_id": "2hNhi3NqT",
                "name": "12",
                "chainId": "0120d06d4a73b60357a5ed24a9145c967308738d70397c25eeedcbb736166ccf"
            }
        ]
    },
    "error": null,
    "sender_id": 2956,
    "inpage": true
}

```
