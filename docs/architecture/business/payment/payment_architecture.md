# Payment 支付业务

![payment_system_core_application_architecture](./images/payment_system_core_application_architecture.png)

**Payment Engine**: 支付引擎（支付核心服务）

- 负责支付订单的创建，完成支付流程流转；

- 与渠道路由服务交互，获取本次交易的支付渠道信息；

- 与风控服务交互，完成交易风控处理；

- 与账户系统交互，获取支付系统用户信息；完成系统内部的资金信息流转；

- 与营销服务交互，验证卡券、计算实际支付金额

- 与通知服务交互，下游通知

**Member Account**: 会员&账户服务（支付核心服务）

- 维护支付平台个人用户全部信息、商户的用户信息；以及用户对应的体系内的账户与余额信息

- 根据上游交易场景，完成用户账户间的资金信息流流转（分账）

- 与会计系统交互，记录交易的会计分录信息供财务使用

**Clean Settlement**: 清结算服务（支付核心服务）

- 维护交易会计信息，完成交易对账、清分（手续费计算）以及结算出款（资金流流转）

- 基于账户系统的单边交易流水，根据会计规则生成复式会计规则，供财务使用（总账）

- 收集支付订单，根据清分规则完成手续费扣减与退还（结算前置）

- 结算系统，根据商户支付订单信息，完成商户资金结算；并通过出款系统，将信息流最终流转为资金流

- 对账系统，保障系统资金与交易正确完整的保障

**Risk**: 风控服务（支付核心服务）

**Security**: 安全服务

- 维护用户支付密码等核心敏感安全信息

- 通用加解密

- 商户秘钥信息

- 渠道秘钥信息