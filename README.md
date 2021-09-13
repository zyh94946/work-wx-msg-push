
# 基于腾讯云Serverless实现的企业微信应用消息推送服务

Serverless 云函数目前每月有免费资源使用量40万GBs、免费调用次数100万次

API网关目前开通即送时长12个月100万次免费额度

个人或者低频率使用完全够了，可以通过 GET、POST 方式调用发消息。

对于有服务器、域名资源，~~通过简单修改~~也可以直接部署到服务器上。

**独立部署版见 [wx-msg-push](https://github.com/zyh94946/wx-msg-push)**

## 消息效果

<details>
<summary>点击展开</summary>
<img src="https://raw.githubusercontent.com/zyh94946/wx-msg-push-tencent/main/demo/demo.gif" width="50%" /><img src="https://raw.githubusercontent.com/zyh94946/wx-msg-push-tencent/main/demo/demo_text.png" width="50%" />
</details>

不用安装企业微信App，直接通过微信App关注微信插件即可实现在微信App中接收应用消息，还可以选择消息免打扰。

## 消息限制

目前支持发送的应用消息类型为：

1. 图文消息(mpnews)，消息内容支持html标签，不超过666K个字节，会自动生成摘要替换br标签为换行符，过滤html标签。
2. 文本消息(text)，消息内容最长不超过2048个字节，超过将截断。
3. markdown消息(markdown)，markdown内容，最长不超过2048个字节，必须是utf8编码。([支持的markdown语法](https://work.weixin.qq.com/api/doc/90000/90135/90236#%E6%94%AF%E6%8C%81%E7%9A%84markdown%E8%AF%AD%E6%B3%95))

每企业消息发送不可超过帐号上限数*30人次/天（注：若调用api一次发给1000人，算1000人次；若企业帐号上限是500人，则每天可发送15000人次的消息）

每应用对同一个成员不可超过30条/分，超过部分会被丢弃不下发

默认已启用重复消息推送检查5分钟内同样内容的消息，不会重复收到，可修改 `EnableDuplicateCheck` `DuplicateCheckInterval` 调整是否开启与时间间隔。

## 部署方式

首先注册[企业微信](https://work.weixin.qq.com/)

### 创建应用

登录[企业微信web管理](https://work.weixin.qq.com/)

进入 `应用管理` ， `创建应用` ，完成后复制下 `AgentId` `Secret` 。

(如果仅使用文本消息可跳过此步) 进入 `管理工具` ， `素材库` ， `图片` ， `添加图片` （这个图片是图文消息的展示图），上传成功后在图片下载按钮上复制下载地址

<img src="https://raw.githubusercontent.com/zyh94946/wx-msg-push-tencent/main/demo/media.png" />

(如果仅使用文本消息可跳过此步) 把url的 `media_id` 值复制下备用

进入 `我的企业` ，把 `企业ID` 复制下，进入 `微信插件` ，用微信APP扫 `邀请关注` 的二维码码即可在微信App中查看企业微信消息。

<img src="https://raw.githubusercontent.com/zyh94946/wx-msg-push-tencent/main/demo/info.png" />

### 注册腾讯云账号

[腾讯云](https://cloud.tencent.com/)

### 新建云函数

[云函数](https://console.cloud.tencent.com/scf/index)

如果想要绑定域名的话可以选择香港地区，免备案。

从 releases 中下载 `linux-amd64` 的预编译 zip 包。

选择 `自定义创建` ， `运行环境` `Go1` ， `提交方法` 选择 `本地上传zip包` 选择刚才的 zip 包。

<img src="https://raw.githubusercontent.com/zyh94946/wx-msg-push-tencent/main/demo/cf1.png" />

`高级配置` 中增加 `环境变量`

- CORP_ID 企业微信 企业id
- CORP_SECRET 企业微信 应用Secret
- AGENT_ID 企业微信 应用AgentId
- MEDIA_ID 企业微信 图片素材的media_id(如果仅使用文本消息可随意填写)

`触发器` 配置选择 `自定义配置` ，触发方式选择 `API网关触发`

点击 `完成` 开始创建

### 设置API网关

[Api网关](https://console.cloud.tencent.com/apigateway/service)

进入 `API网关` 服务列表，选择 `配置管理` ，然后 `管理API` 点 `编辑`

<img src="https://raw.githubusercontent.com/zyh94946/wx-msg-push-tencent/main/demo/api1.png" />

增加 `参数配置` ，参数名 `SECRET` ，参数位置 `path`，类型 `string`

路径修改为 `/你的云函数名称/{SECRET}`

然后点 `立即完成` 发布服务

在 `基础配置` 中复制 `公网访问地址` ，想要绑定域名可以在 `自定义域名` 中绑定，CNAME指到API网关的二级域名，自定义路径映射 `/` 路径 到 `发布` 环境。

## 使用方法

消息类型值：`text` 代表文本消息，`mpnews` 代表图文消息，`markdown` 代表 markdown 消息。为兼容旧版本，不传默认为图文消息。

支持推送消息至指定的 `touser`, `toparty`, `totag`。不传默认设置 `touser=@all` 

GET方式

`https://你的Api网关域名/你的云函数名称/CORP_SECRET?title=消息标题&content=消息内容&type=消息类型`

POST方式

```bash
$ curl --location --request POST 'https://你的Api网关域名/你的云函数名称/CORP_SECRET' \
--header 'Content-Type: application/json;charset=utf-8' \
--data-raw '{"title":"消息标题","content":"消息内容","type":"消息类型"}'
```

发送成功状态码返回200，`"Content-Type":"application/json"` body `{"errorCode":0,"errorMessage":""}` 。

## 其它

发送失败问题排查：

- 请检查云函数环境变量是否正确
- 请检查发送url中CORP_SECRET是否正确
- 请检查Api网关SECRET参数是否设置
- 进入云函数后台查看请求日志的具体错误原因

## 更新记录

- 2021-04-11 支持文本消息，优化代码结构方便新增其它消息类型。
- 2021-04-29 支持推送消息至指定的`touser`, `toparty`, `totag`。
- 2021-09-13 支持markdown消息类型。
