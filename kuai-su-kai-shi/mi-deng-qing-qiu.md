# 幂等请求

> 幂等请求是指在一次或多次请求服务器时，对于相同的请求数据，无论发送多少次请求，都不会对服务器产生副作用，也不会改变服务器的状态。即，对于相同的请求，无论请求多少次，服务器的响应都是相同的。

* 请求header中，若有`idempotency-key`时，则会缓存接口返回结果信息，有效期：24小时
* 后续相同`idempotency-key`, 相同`bizType`, `version`, `group`, `companyId`, `token`, 有效期内，则直接获取缓存结果
* 如果并发请求，只会放行一次请求，其余请求，直接返回`请求频率过快，请稍后重试`

### 涉及接口

用于所有操作类的接口，包括但不仅限于以下接口：

* [客户注册](https://docs.hyperex.io/reference/post\_register-customer-app-customerwebapiservice-register) `customer.app.CustomerWebApiService.register`
* [下單](https://docs.hyperex.io/reference/post\_global-tradeapi-app-cfdmmorderapiservice-addmarketorder) `/global/tradeapi.app.CfdMMOrderApiService.addMarketOrder`
* [创建存款提案](https://docs.hyperex.io/reference/post\_global-fund-app-depositappdubboservice-createdepositproposal) `/global/fund.app.DepositAppDubboService.createDepositProposal`
* [创建取款提案](https://docs.hyperex.io/reference/post\_global-fund-app-withdrawappdubboservice-createwithdrawproposal) `/global/fund.app.WithdrawAppDubboService.createWithdrawProposal`
* [资产资金划转](https://docs.hyperex.io/reference/post\_global-fund-app-depositappdubboservice-capitaltransfersupportdiffcurr) `/global/fund.app.DepositAppDubboService.capitalTransferSupportDiffCurr`
* [提交KYC等级认证申请](https://docs.hyperex.io/reference/post\_global-customer-app-kycwebapiservice-kyclevelapply) `/global/customer.app.KycWebApiService.kycLevelApply`

### 开启幂等请求的方式

接口请求头headers里面的添加`idempotency-key`字段；

![idempotencyKey](https://files.readme.io/2db673f-idempotencyKey.png)

### 接入步骤 (以充值场景举例)

* 在用户进入到充值页面的时候，用uuid生成一个`idempotency-key`；

```js
let idempotencyKey = uuid();
```

* 用户提交充值的时候，在接口请求头里面增加`idempotency-key`；
* 接口响应成功之后，再更新变量`idempotencyKey`的值，留给下一次使用；

```js
request({
    url: '/global/fund.app.DepositAppDubboService.createDepositProposal',
    method: 'post',
    headers: {
        "version": '0.0.1',
        "idempotency-key": idempotencyKey
    },
    data
}).then((res)=>{
    // 更新idempotencyKey变量
    idempotencyKey = uuid();

    // GATEWAY_CODE_018 错误码是重复请求的接口返回的报错信息
    if(res.code === "GATEWAY_CODE_018") return ;

    // 这里处理返回的res数据
    // ......
})
```

**请注意：**

如果你收到`GATEWAY_CODE_018`的错误码，那是因为你的接口出现了并发请求，只会放行一次请求，其余请求，直接返回：

```json
{
    "msg": "请求频率过快，请稍后重试",
    "trace": "你的trace",
    "code": "GATEWAY_CODE_018",
    "data": "你的请求地址",
    "bizCode": "G"
}
```

这时候你可以根据你的业务场景对`GATEWAY_CODE_018`错误码进行处理，或者不做任何处理。

### 常見問題

*   問：当用户充值完成之后，下次再进行充值的时候需要用新的`idempotency-key`吗？

    答：是的。当用户真的进行下一轮充值等时候，是需要使用新的`idempotency-key`，如果继续使用上一次的key，服务端会将上一次的响应信息直接返。
*   問：我应该是在用户点击提交按钮的时候生成`idempotency-key`吗？

    答：不能这样，用户在快速点击提交按钮的时候如果每次都生成新的`idempotency-key`，就无法实现幂等请求的效果。最好的方式是在用户进入页面的时候就生成`idempotency-key`，然后在提交成功之后再次更新`idempotency-key`，给用户在下一次充值的时候使用。
*   `idempotency-key`必須用uuid生成嗎？

    答：很重要的一点是：相同`idempotency-key`, 相同`bizType`, `version`, `group`, `companyId`, `token`, 有效期内，则直接返回缓存结果。它不是必須用uuid生成，只是需要保證用戶在提交一輪新的操作的時候，`idempotency-key`需要生成新的唯一標識符。
