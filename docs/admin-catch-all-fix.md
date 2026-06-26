# CloudMail 管理员 Catch-All 邮件路由修复

## 问题描述

管理员邮箱 (`admin@域名`) 无法接收发送到该域名下其他地址的邮件。例如：

- `admin@owaviowa.dpdns.org` **无法**收到发送给 `1213212@grok1.owaviowa.dpdns.org` 的邮件
- Cloudflare 电子邮件路由摘要显示所有未注册地址的邮件状态为 **"已删除"**
- 必须手动添加对应邮箱地址到系统中，才能接收到该地址的邮件

### 预期行为

管理员邮箱应作为 **Catch-All（全捕获）** 地址，自动接收发送到该域名（包括子域名）下所有地址的邮件，无需逐一注册。

| 收件地址 | 修复前 | Catch-All 修复后 | 子域名修复后 |
|---------|--------|-----------------|-------------|
| `admin@domain` | ✅ 正常接收 | ✅ 正常接收 | ✅ 正常接收 |
| `1@domain`（未注册） | ❌ 被拒绝 | ✅ 路由至管理员 | ✅ 路由至管理员 |
| `1@sub.domain`（未注册） | ❌ 被拒绝 | ❌ 仍被拒绝 | ✅ 路由至管理员 |
| `registered@domain`（已注册） | ✅ 正常接收 | ✅ 正常接收 | ✅ 正常接收 |

---

## 根因分析

### 邮件接收流程

```
Cloudflare Email Routing
        │
        ▼
┌─────────────────────────┐
│   Worker email handler   │  ← email.js
│                          │
│  1. 全局设置检查          │
│  2. 解析原始邮件          │
│  3. 黑名单过滤            │
│  4. 查询收件人账户  ◄────── 关键步骤
│  5. 域名权限检查          │
│  6. 保存邮件到数据库       │
│  7. 转发（TG/邮箱）       │
└─────────────────────────┘
```

### 问题代码（修复前）

**文件：`mail-worker/src/email/email.js`**

```javascript
// 第60行：按收件地址精确查询账户
const account = await accountService.selectByEmailIncludeDel(
    { env: env }, message.to
);

// 第62-65行：找不到账户 → 直接拒绝
if (!account && noRecipient === settingConst.noRecipient.CLOSE) {
    message.setReject('Recipient not found');  // ← Cloudflare 显示"已删除"
    return;
}
```

当邮件发送到 `1213212@grok1.owaviowa.dpdns.org` 时：

1. 系统在 `account` 表中查找 `1213212@grok1.owaviowa.dpdns.org`
2. 找不到匹配记录 → `account = null`
3. `noRecipient` 设置为 `CLOSE` → 调用 `message.setReject()` 拒绝邮件
4. Cloudflare 收到拒绝信号 → 标记为"已删除"

### 管理员权限的局限

代码第 73 行的管理员特殊处理：

```javascript
if (account && userRow.email !== env.admin) {
    // 非管理员才检查域名权限和封禁列表
}
```

这段代码**仅在 `account` 存在时生效**。当 `account` 为 `null` 时，根本执行不到这里。也就是说，管理员的特殊权限从来不会在"收件地址无匹配账户"的场景中发挥作用。

### 内部邮件的同样问题

**文件：`mail-worker/src/service/email-service.js`** 的 `HandleOnSiteEmail` 方法

当系统内部发送邮件（所有收件人都在同一域名下）时，如果收件地址无对应账户：

```javascript
// 第600-613行
} else {
    emailValues.userId = 0;       // 无归属用户
    emailValues.accountId = 0;    // 无归属账户
    emailValues.status = emailConst.status.NOONE;  // "无人接收"状态

    if (noRecipient === settingConst.noRecipient.CLOSE) {
        emailValues.status = emailConst.status.BOUNCED;  // 拒收
    }
}
```

邮件被标记为 `NOONE`（userId=0）或 `BOUNCED`，对所有人不可见，包括管理员。

---

## 修复方案

### 核心思路

在收件地址无匹配账户时，增加 **管理员 Catch-All 回退逻辑**：

```
收到邮件 → 查找收件地址账户
                │
                ├── 找到 → 正常流程（不变）
                │
                └── 未找到 → 查找管理员账户
                                │
                                ├── 找到 → 路由至管理员
                                │
                                └── 未找到 → 原有逻辑（拒绝/NOONE）
```

### 修复 1：外部邮件处理器

**文件：`mail-worker/src/email/email.js`**

```diff
- const account = await accountService.selectByEmailIncludeDel(
-     { env: env }, message.to
- );
+ let account = await accountService.selectByEmailIncludeDel(
+     { env: env }, message.to
+ );
+
+ // 管理员Catch-All：当收件地址无对应账户时，将邮件路由至管理员账户
+ if (!account && env.admin) {
+     const adminAccount = await accountService.selectByEmailIncludeDel(
+         { env: env }, env.admin
+     );
+     if (adminAccount) {
+         account = adminAccount;
+     }
+ }

  if (!account && noRecipient === settingConst.noRecipient.CLOSE) {
      message.setReject('Recipient not found');
      return;
  }
```

**关键点：**

- `const` → `let`：允许变量重新赋值
- 在 `noRecipient` 拒绝检查**之前**执行 Catch-All
- 管理员账户存在时，`account` 被赋值为管理员账户
- 后续的拒绝检查不再触发（`account` 已非 `null`）

### 修复 2：内部邮件处理器

**文件：`mail-worker/src/service/email-service.js`**

在 `HandleOnSiteEmail` 方法中：

```diff
  // 查询所有收件人权限身份
  const userIds = accountList.map(accountRow => accountRow.userId);
  let roleList = await roleService.selectByUserIds(c, userIds);

+ // 管理员Catch-All：预加载管理员账户，用于路由无匹配收件人的邮件
+ let adminAccount = null;
+ if (c.env.admin) {
+     adminAccount = await accountService.selectByEmailIncludeDel(
+         c, c.env.admin
+     );
+ }
```

在无匹配账户的分支中：

```diff
  } else {
-     // 设置无收件人邮件信息
-     emailValues.userId = 0;
-     emailValues.accountId = 0;
-     emailValues.type = emailConst.type.RECEIVE;
-     emailValues.status = emailConst.status.NOONE;
-
-     // 如果无人收件关闭改为拒收
-     if (noRecipient === settingConst.noRecipient.CLOSE) {
-         emailValues.status = emailConst.status.BOUNCED;
-         emailValues.message = `Recipient not found: <${email}>`;
+     // 管理员Catch-All：无匹配账户时路由至管理员
+     if (adminAccount) {
+         emailValues.userId = adminAccount.userId;
+         emailValues.accountId = adminAccount.accountId;
+         emailValues.type = emailConst.type.RECEIVE;
+         emailValues.status = emailConst.status.RECEIVE;
+     } else {
+         // 设置无收件人邮件信息
+         emailValues.userId = 0;
+         emailValues.accountId = 0;
+         emailValues.type = emailConst.type.RECEIVE;
+         emailValues.status = emailConst.status.NOONE;
+
+         // 如果无人收件关闭改为拒收
+         if (noRecipient === settingConst.noRecipient.CLOSE) {
+             emailValues.status = emailConst.status.BOUNCED;
+             emailValues.message = `Recipient not found: <${email}>`;
+         }
      }
  }
```

---

## 修复后的完整邮件处理流程

### 外部邮件（email.js）

```
Cloudflare Email Routing
        │
        ▼
┌──────────────────────────────────────┐
│         email.js 处理流程             │
│                                      │
│  1. 全局设置检查（receive 开关）       │
│  2. 解析原始邮件（PostalMime）        │
│  3. 黑名单过滤                        │
│                                      │
│  4. 查询收件人账户                     │
│     ├── 精确匹配 → 使用该账户         │
│     └── 无匹配 → 查找管理员账户       │
│         ├── 找到 → 使用管理员账户      │
│         └── 未找到 → 拒绝/NOONE       │
│                                      │
│  5. 域名权限检查                       │
│     └── 管理员跳过此检查              │
│                                      │
│  6. 保存邮件到数据库                   │
│     ├── toEmail = 原始收件地址         │
│     ├── userId = 管理员的 userId       │
│     └── accountId = 管理员的 accountId │
│                                      │
│  7. 转发逻辑                          │
│     ├── 规则检查                      │
│     ├── Telegram 转发                 │
│     └── 外部邮箱转发                   │
└──────────────────────────────────────┘
```

### 内部邮件（email-service.js HandleOnSiteEmail）

```
用户发送站内邮件
        │
        ▼
┌──────────────────────────────────────┐
│  判断是否全部为站内邮箱                │
│  （支持子域名匹配）                    │
│                                      │
│  domainList = ["@maindomain.com"]    │
│                                      │
│  收件人域名匹配规则：                  │
│  ├── maindomain.com       → ✅ 精确   │
│  ├── sub.maindomain.com   → ✅ 子域名  │
│  ├── a.b.maindomain.com   → ✅ 子域名  │
│  ├── evil-maindomain.com  → ❌ 不匹配  │
│  └── otherdomain.com      → ❌ 不匹配  │
│                                      │
│  全部站内 → 走 HandleOnSiteEmail      │
│  存在外站 → 走 Resend/Cloudflare 发送 │
└──────────────────────────────────────┘
```

### 数据库存储示例

当 `1213212@grok1.owaviowa.dpdns.org` 收到邮件时，数据库记录：

| 字段 | 值 | 说明 |
|------|-----|------|
| `toEmail` | `1213212@grok1.owaviowa.dpdns.org` | 保留原始收件地址 |
| `toName` | `1213212` | 原始收件人名称 |
| `userId` | 管理员的 userId | 邮件归属管理员 |
| `accountId` | 管理员的 accountId | 邮件归属管理员账户 |
| `sendEmail` | 实际发件人 | 不变 |
| `status` | `0`（RECEIVE） | 正常接收状态 |

管理员在收件箱中可以看到这封邮件，并通过 `toEmail` 字段知道邮件原本是发给谁的。

---

## 管理员权限体系（参考）

修复后，管理员在以下场景中拥有完整权限：

| 场景 | 权限 | 代码位置 |
|------|------|----------|
| 接收未注册地址的邮件 | ✅ Catch-All | `email.js:62-68` |
| 子域名邮件站内识别 | ✅ 子域名匹配 | `email-service.js:179-186` |
| 子域名权限检查 | ✅ 子域名匹配 | `role-service.js:157-172` |
| 跳过域名权限检查 | ✅ 绕过 | `email.js:81` |
| 跳过发件次数限制 | ✅ 无限制 | `email-service.js:185,200` |
| 跳过域名发件权限 | ✅ 绕过 | `email-service.js:224` |
| 跳过角色封禁检查 | ✅ 绕过 | `email.js:81` |
| API 端点完全访问 | ✅ 绕过 | `security.js:148` |
| 角色固定为 admin | ✅ 固定 | `user-service.js:48-51` |

---

## 部署步骤

```bash
# 1. 确认 wrangler.toml 中配置了 admin 变量
# [vars]
# admin = "admin@yourdomain.com"

# 2. 部署 Worker
cd mail-worker
npx wrangler deploy

# 3. 验证
# - 在 Cloudflare Dashboard > Email Routing 中确认 Catch-All 路由已启用
# - 发送测试邮件到任意未注册地址
# - 登录管理员账户确认邮件已收到
```

---

## 子域名邮件修复（补充）

### 问题描述

主域名的 Catch-All 修复后，子域名的邮件仍然无法被管理员接收。例如：

- `admin@maindomain.com` ✅ 能收到 `1@maindomain.com` 的邮件
- `admin@maindomain.com` ❌ 收不到 `1@subdomain.maindomain.com` 的邮件

### 根因分析

**文件：`mail-worker/src/service/email-service.js`** 的 `HandleOnSiteEmail` 方法

站内邮件判断使用精确匹配：

```javascript
// 修复前
const allInternal = receiveEmail.every(email => {
    const domain = '@' + emailUtils.getDomain(email);
    return domainList.includes(domain);  // 精确匹配！
});
```

`domainList` = `["@maindomain.com"]`，当发件人是 `user@subdomain.maindomain.com` 时：
- `domain` = `@subdomain.maindomain.com`
- `domainList.includes("@subdomain.maindomain.com")` → **false**
- 邮件被当作**外部邮件**处理 → 没有子域名的 Resend token → 抛出 `noSendProvider` 错误

**文件：`mail-worker/src/service/role-service.js`** 的 `hasAvailDomainPerm` 方法

域名权限检查同样使用精确匹配，导致子域名用户无法通过权限验证。

### 修复 3：子域名站内邮件判断

**文件：`mail-worker/src/service/email-service.js`**

```diff
- //判断接收方是不是全部为站内邮箱
- const allInternal = receiveEmail.every(email => {
-     const domain = '@' + emailUtils.getDomain(email);
-     return domainList.includes(domain);
- });
+ //判断接收方是不是全部为站内邮箱（支持子域名匹配）
+ const allInternal = receiveEmail.every(e => {
+     const domain = emailUtils.getDomain(e).toLowerCase();
+     return domainList.some(d => {
+         const baseDomain = d.slice(1).toLowerCase();
+         return domain === baseDomain || domain.endsWith('.' + baseDomain);
+     });
+ });
```

**关键点：**
- `domain === baseDomain`：精确匹配主域名
- `domain.endsWith('.' + baseDomain)`：匹配所有子域名（`sub.maindomain.com`、`a.b.maindomain.com` 等）
- 前缀 `.` 确保不会误匹配（如 `evil-maindomain.com` 不会匹配 `maindomain.com`）

### 修复 4：子域名权限检查

**文件：`mail-worker/src/service/role-service.js`**

```diff
  hasAvailDomainPerm(availDomain, email) {
      availDomain = availDomain.split(',').filter(item => item !== '');
      if (availDomain.length === 0) {
          return true
      }
+     const domain = emailUtils.getDomain(email.toLowerCase());
      const availIndex = availDomain.findIndex(item => {
-         const domain = emailUtils.getDomain(email.toLowerCase());
          const availDomainItem = item.toLowerCase();
-         return domain === availDomainItem
+         return domain === availDomainItem || domain.endsWith('.' + availDomainItem)
      })
      return availIndex > -1
  },
```

### 子域名匹配逻辑

```
配置域名：maindomain.com

匹配结果：
├── maindomain.com           → ✅ 精确匹配
├── sub.maindomain.com       → ✅ 子域名匹配
├── a.b.maindomain.com       → ✅ 深层子域名匹配
├── evil-maindomain.com      → ❌ 不匹配（前缀无 `.`）
├── maindomain.com.evil.com  → ❌ 不匹配
└── otherdomain.com          → ❌ 不匹配
```

---

## 注意事项

1. **管理员账户必须存在**：`env.admin` 配置的邮箱必须在系统中已注册为账户，否则 Catch-All 不会生效
2. **Cloudflare 路由配置**：确保 Cloudflare Email Routing 中已为相关域名（包括子域名）配置了路由规则，将邮件发送到 Worker
3. **`toEmail` 保留原始地址**：管理员可以通过邮件详情中的收件人字段识别邮件原本发送给哪个地址
4. **转发规则**：Catch-All 邮件的 Telegram/邮箱转发遵循管理员账户的转发设置
5. **`noRecipient` 设置**：Catch-All 逻辑在 `noRecipient` 检查之前执行，因此无论该设置如何，只要管理员账户存在，邮件都会被接收
6. **子域名自动支持**：配置主域名后，所有子域名自动获得支持，无需在 `env.domain` 中逐一添加子域名
