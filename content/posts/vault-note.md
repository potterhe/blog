---
title: "Vault 学习笔记"
date: 2023-04-14T10:40:31+08:00
draft: true
---

- [官网](https://www.vaultproject.io/)
- [Vault 中文手册](https://lonegunmanb.github.io/essential-vault/)
- [MFA Login with Vault TOTP](https://docmoa.github.io/04-HashiCorp/06-Vault/03-Auth_Method/mfa-login.html) 这是一个可以工作的样例，Vault版本：1.13.1


在Vault中，您可以使用`identity/mfa/method/totp/admin-generate`命令生成一个包含TOTP密钥的base64-encoded barcode和OTP url JSON对象。您可以使用此数据来配置任何TOTP生成器应用程序，例如Google Authenticator、Authy等。

要使用此数据，可以按照以下步骤执行：

1. 解码base64-encoded barcode - 获取包含base64-encoded barcode的JSON对象，然后使用base64解码器将其解码为图像文件并在图像应用程序中查看要扫描的条形码。 例如，您可以使用以下命令来执行此操作：

   ```
   $ echo "<BASE64 BARCODE DATA>" | base64 -d > barcode.png
   ```

2. 获取OTP url并输入生成的OTP code - 取得OTP url，通常它具有以下格式：

   ```
   otpauth://totp/<VAULT_APP_LABEL>?secret=<TOTP_SECRET>&issuer=<VAULT_APP_NAME>
   ```

   在此，`<VAULT_APP_LABEL>`是应用程序的标签，`<TOTP_SECRET>`是由Vault生成的TOTP密钥，`<VAULT_APP_NAME>`是应用程序的名称。

3. 打开TOTP生成器应用程序并选择“添加帐户”。选择手动添加，然后输入以下信息：

   - “帐户名称”或“标签” - <VAULT_APP_LABEL>（从URL中获取）
   - “密钥”或“验证码” - <TOTP_SECRET>（从URL或Vault JSON数据中获取）
   - “发行者”或“厂商” - <VAULT_APP_NAME>（从URL中获取）

4. 验证JWT密码 - 获取JWT密码，解码后将其输入到Vault中来验证TOTP身份验证。

总之，您可以使用上述步骤配置TOTP生成器应用程序并使用OTP url和OTP代码来验证Vault身份验证。

Vault UI 内置 wizard 在1.12.4 版本中删除 [GH-19220](https://github.com/hashicorp/vault/pull/19220)
Vault UI未提供二维码相关功能
