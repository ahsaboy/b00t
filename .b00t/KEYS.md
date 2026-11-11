# Bootstrap 密钥与发布说明

## 首版信息

- GitHub 仓库：`ahsaboy/b00t`
- Bootstrap 文件：`.b00t/.la_b00t.json`
- jsDelivr 地址：`https://cdn.jsdelivr.net/gh/ahsaboy/b00t@main/.b00t/.la_b00t.json`
- 签名算法：ECDSA P-256 + SHA-256
- 签名编码：ASN.1 DER（OpenSSL 默认 ECDSA 签名编码）
- Key ID：`bootstrap-signing-2026-01`
- 私钥本机路径：`%LOCALAPPDATA%\LiveAssistant\bootstrap-signing-private.pem`
- 当前 `configApi` 是 `https://config.example.com` 占位地址，必须替换成真实配置 API 后重新签名发布。

## 密钥边界

私钥只用于签署 Bootstrap，绝不可提交到 GitHub、客户端、日志、安装包或此说明文档。当前开发机生成的私钥保存在上述用户目录。备份时应放入受控的离线密钥库；丢失后需生成新密钥、更新客户端公钥并发布对应签名文件。

客户端只嵌入公钥。公钥不是秘密，可在 `App.xaml.cs` 中查看。当前开发公钥对应上述本机私钥。正式发布前建议使用正式发布密钥替换开发密钥，并确保发布流水线能安全访问私钥。

## 文件签名规则

签名输入不是完整外层 JSON，而是 `payload` 对象紧凑序列化后的 UTF-8 字节，属性顺序固定如下：

```json
{"schemaVersion":1,"configApi":["https://config.example.com"],"keyId":"bootstrap-signing-2026-01","minClientVersion":"1.0.0"}
```

外层文件结构：

```json
{
  "payload": {
    "schemaVersion": 1,
    "configApi": ["https://config.example.com"],
    "keyId": "bootstrap-signing-2026-01",
    "minClientVersion": "1.0.0"
  },
  "signature": "Base64 编码的 DER 签名"
}
```

签名时必须使用紧凑 JSON、UTF-8、上述字段顺序及相同的 `keyId`。外层 JSON 可缩进格式化，因为客户端会反序列化 payload 后按固定模型重新序列化再验签。

## 更新 Bootstrap

1. 编辑 `.b00t/.la_b00t.json` 中的 `payload`，替换服务器地址或其他字段。
2. 按上面的紧凑格式生成 payload 文件，例如 `payload.json`。不要对外层 JSON 签名。
3. 使用本机私钥生成 DER 签名并转为单行 Base64：

```powershell
openssl dgst -sha256 -sign "$env:LOCALAPPDATA\LiveAssistant\bootstrap-signing-private.pem" -out signature.der payload.json
$signature = [Convert]::ToBase64String([IO.File]::ReadAllBytes('signature.der'))
```

4. 将 `$signature` 写入外层 JSON 的 `signature` 字段，检查 JSON 格式和 HTTPS 地址。
5. 提交并推送 `.b00t/.la_b00t.json`。jsDelivr 对分支别名有缓存，更新后可能需要等待缓存或使用 jsDelivr purge 工具；不要把缓存清理当作完整性校验。

## 轮换与泄漏响应

密钥轮换时，先将新公钥加入客户端并发布新版客户端，再切换 Bootstrap 的 `keyId` 和签名私钥。若私钥泄漏，应视为 Bootstrap 可被伪造：尽快发布包含新公钥的客户端版本、替换签名并撤销旧密钥。单纯更改 GitHub 文件不能撤销已被旧公钥信任的签名。

## 首版限制

首版只拉取、验签、缓存 Bootstrap 并在网络失败时回退。配置 API 地址当前是占位值；远程配置 API 的下载、验签和业务模块尚未实现。客户端不执行远程代码，Bootstrap 也不能替代服务端授权。
