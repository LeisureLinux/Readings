# 从 Junk 文件夹里救回正常邮件：rspamd 误判治理实录

> 一篇面向自建邮件服务器管理员的排障记录：合法新闻通讯被 rspamd 当成垃圾邮件丢进 Junk 文件夹，如何通过白名单域名 + 组合规则（composite）把分数压回负值，并重新过滤 Junk 文件夹。

## 背景

我的邮件服务器（Postfix + Dovecot + rspamd，作为本域 MX）跑了一段日子后，发现本地客户端（Thunderbird）的 Junk 文件夹里积压了很多**明明是正常订阅**的新闻通讯：

- Debian 邮件列表
- McKinsey 邮件
- LinkedIn 邮件
- dev.to 邮件
- RFI 邮件
- 以及 Amazon/AWS 的营销邮件

它们的共同点：都通过了 SPF/DKIM，属于高信誉机构，但 rspamd 仍给了高分，被判定为垃圾。于是开始排障。

## 根本原因分析

用 `rspamc` 对几封误报邮件重新打分，定位到三个"帮凶"符号：

| 符号 | 默认分值 | 触发原因 |
|------|---------|---------|
| `FORGED_SENDER` | +5.0 | **envelope sender ≠ From 头**。ESP 常用子域做弹回地址，如 `bounce@email.xxx`、`em4984.xxx`、`edt.xxx`，与 `From:` 的域名不一致 |
| `FORGED_RECIPIENTS` | +2.0 | **收件人 `To:` 是列表地址**而非个人邮箱，常见于 Debian 等邮件列表 |
| `BLACKLIST_DMARC` | +6.0 | 供应商经 SES 等代发平台**代表主域发送**（envelope 是 `amazonses.com`，From 是 `amazon.com`），DMARC 对齐检查把域标记为黑名单 |

所有误报邮件 **SPF 均通过**（`R_SPF_ALLOW`），这是判为合法的关键依据——只要发件域在白名单内且通过 SPF/DKIM，就应视为可信。

## 规则调整

### 1. 扩展白名单域名

在 `local.d/maps.d/whitelist_domains.inc` 中追加了已订阅的新闻通讯/厂商域名：

```
mckinsey.com
email.mckinsey.com
dev.to
mail.dev.to
em4984.dev.to
rfi.nlfrancemm.com
nlfrancemm.com
linkedin.com
computerworld.com
edt.computerworld.com
cncf.io
amazon.com
amazonaws.com
```

> 说明：`WHITELIST_DOMAINS_FROM` 是 rspamd 内置的动态映射符号，基于上述文件匹配 `From:` 头域名。

### 2. 新增三条组合规则（composites.conf）

追加在 `local.d/composites.conf`，针对每个误报原因精准对冲：

```
# 对冲 FORGED_SENDER(+5.0)：白名单域且通过 SPF 或 DKIM
TRUSTED_NEWSLETTER_PASS {
	expression = "WHITELIST_DOMAINS_FROM & (R_SPF_ALLOW | R_DKIM_ALLOW)";
	score = -5.0;
}

# 对冲 FORGED_RECIPIENTS(+2.0)：白名单域 + 列表收件人 + 通过 SPF/DKIM
TRUSTED_NEWSLETTER_RECIPIENTS {
	expression = "WHITELIST_DOMAINS_FROM & FORGED_RECIPIENTS & (R_SPF_ALLOW | R_DKIM_ALLOW)";
	score = -2.0;
}

# 对冲 BLACKLIST_DMARC(+6.0)：白名单域 + 命中 BLACKLIST_DMARC + 通过 SPF/DKIM
BLACKLIST_DMARC_TRUSTED {
	expression = "WHITELIST_DOMAINS_FROM & BLACKLIST_DMARC & (R_SPF_ALLOW | R_DKIM_ALLOW)";
	score = -7.0;
}
```

设计要点：
- **限定白名单域**：普通垃圾邮件哪怕带了 `List-Unsubscribe` 头，只要不在白名单内，就绝不会吃到负分。
- **要求强鉴权**：必须 `R_SPF_ALLOW` **或** `R_DKIM_ALLOW` 至少一个通过，杜绝伪装白名单域的钓鱼（伪造域通常 SPF/DKIM 双双失败，另由 `WHITELIST_SPOOFED` 等规则 +10 分击穿）。
- **负分只对冲、不给加成**：每条规则分值恰好抵消对应误判符号（-5↔+5、-2↔+2、-7↔+6），不会把垃圾邮件误放行到 Inbox。

### 3. 重新加载并验证

```bash
sudo systemctl reload rspamd
```

语法检查无警告，服务状态 `active`。随后用 `rspamc` 对几封原误报邮件重新打分：

| 邮件 | 调整前 | 调整后 |
|------|--------|--------|
| McKinsey | +4.80 | **-3.00** |
| dev.to | +4.80 | **-3.00** |
| RFI | +4.80 | **-1.80** |
| Debian 列表 | +6.20 | **-23.20** |
| LinkedIn | +4.80 | **-4.50** |
| AWS | +7.90 | **-9.90** |

所有误报分数都从正分压到负值，远低于 add_header（6 分）和 reject（15 分）阈值，不会再进 Junk。

## 重新过滤 Junk 文件夹

已经进 Junk 的老邮件，分数不会自动重算。手动处理流程如下。

### 1. 备份与 dry-run 扫描

```bash
# 备份将被移动的文件
mkdir -p /path/to/backup

# 对 Junk 中每封邮件重新打分，看是否有符号变成负分
sudo sh -c "cat <mailfile> | rspamc -h 127.0.0.1:11333"
```

逐个确认哪些是"清白"邮件后，再做移动。

### 2. 改写头部并移动

对确认清白的邮件：
- 用文本工具重写头部：`X-Spam: Yes` → `X-Spam: No`，移除旧的 `X-Spamd-*` / `X-Rspamd-*` 头，避免客户端仍按旧状态显示；
- 原始文件先备份（副本留档，便于回滚）；
- 将文件从 `Maildir/.Junk/cur/` 移动到 `Maildir/cur/`。

### 3. 同步 Dovecot 索引

移动文件后，让 Dovecot 重建索引，否则客户端看不到变化：

```bash
sudo /usr/bin/doveadm force-resync -u USERNAME INBOX
sudo /usr/bin/doveadm force-resync -u USERNAME Junk
```

### 4. 结果

本轮从 Junk 移回 Inbox 19 封（含后来单独处理的 AWS 那封）。Junk 中剩余 4 封经复核确为垃圾（诈骗/健康广告/疑似群发），予以保留。

## 备份与回滚

所有改动都已带时间戳备份，可一键回退：

```bash
# 备份文件（示例）
/etc/rspamd/local.d/composites.conf.backup.<TS>
/etc/rspamd/local.d/maps.d/whitelist_domains.inc.backup.<TS>

# 回滚 = 用备份覆盖 + reload
sudo cp <backup> <原文件>
sudo systemctl reload rspamd
```

## 小结

- **根因**：合法 ESP 用子域做 envelope sender / 列表 `To:` 地址 / 供应商代发，触发了 rspamd 的 `FORGED_*` 与 `BLACKLIST_DMARC` 误判符号。
- **解法**：白名单域名 + 三条"白名单域 & 强鉴权"的组合规则，精准对冲误判分，且不给垃圾邮件留口子。
- **数据支撑**：SPF/DKIM 通过是判断合法性的关键信号，宁可多设白名单，也不要放松对伪造域的惩罚。
