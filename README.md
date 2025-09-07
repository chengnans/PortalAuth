# **城市热点portal认证脚本**

---

## ✅ 功能：

1. ✅ log输出写入本地日志文件
2. ✅ 保留控制台输出（方便手动调试）
3. ✅ 日志自动轮转（可选）
4. ✅ 企业微信通知

---



## ✅ 脚本

```bash
#!/usr/bin/bash

# Portal 自动认证脚本
# 作者：chengnans
# 日期：2025-09-07

# ========== 配置区域 ==========
PORTAL_URL="http://portal_ip:801/eportal/portal/login"
# HUAWEI网络连通性测试URL
TEST_URL="http://connectivitycheck.platform.hicloud.com/generate_204"
PORTAL_DOMAIN="portal服务器地址"
USER_ACCOUNT="你的账号"
USER_PASSWORD="你的密码"
JS_VERSION="4.2.1"
TERMINAL_TYPE="1"
LANG="zh-cn"
V="3914"

# 🆕 企业微信机器人 Webhook（替换成你自己的）
WX_WEBHOOK_URL="https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=你的KEY"

# 🆕 本地日志文件路径（建议使用 /var/log，需 root 权限；普通用户可用 ~/portal_auth.log）
LOG_FILE="/var/log/portal_auth.log"

# 如果日志目录不存在，自动创建（需要权限）
mkdir -p "$(dirname "$LOG_FILE")" 2>/dev/null || true

# ========== 函数定义 ==========
# 🆕 log 函数：同时输出到控制台和日志文件
log() {
    local msg="[$(date '+%Y-%m-%d %H:%M:%S')] $1"
    echo "$msg" | tee -a "$LOG_FILE"
}

# 🆕 发送企业微信通知
send_wx_msg() {
    local msg="$1"
    if [[ -z "$WX_WEBHOOK_URL" ]] || [[ "$WX_WEBHOOK_URL" == *"你的KEY"* ]]; then
        log "⚠️ 未配置企业微信机器人，跳过通知"
        return
    fi

    json_data=$(cat <<EOF
{
    "msgtype": "text",
    "text": {
        "content": "$msg"
    }
}
EOF
)

    response=$(curl -s -X POST "$WX_WEBHOOK_URL" \
        -H "Content-Type: application/json" \
        -d "$json_data" 2>&1)

    if echo "$response" | grep -q '"errcode":0'; then
        log "✅ 企业微信通知发送成功"
    else
        log "❌ 企业微信通知发送失败：$response"
    fi
}

# ✅ 检测是否在线
check_online() {
    log "🔍 检测网络认证状态：curl -Is $TEST_URL"

    response=$(curl -Is -m 3 --connect-timeout 3 "$TEST_URL" 2>/dev/null)
    http_code=$(echo "$response" | head -1 | awk '{print $2}')
    location=$(echo "$response" | grep -i "^Location:" | awk '{print $2}' | tr -d '\r')

    if [[ "$http_code" == "200" ]] || [[ "$http_code" == "204" ]]; then
        log "✅ 检测通过：HTTP $http_code → 已认证"
        return 0
    fi

    if [[ "$http_code" =~ ^30[127]$ ]] && [[ "$location" == *"${PORTAL_DOMAIN}"* ]]; then
        log "⚠️ 检测到重定向到 Portal：$location → 未认证"
        return 1
    fi

    log "⚠️ 检测异常（HTTP $http_code），可能未认证或网络故障 → 尝试登录"
    return 1
}

# 执行登录
do_login() {
    log "🔑 正在执行 Portal 认证..."

    LOGIN_FULL_URL="${PORTAL_URL}?callback=dr1003&login_method=1&user_account=${USER_ACCOUNT}&user_password=${USER_PASSWORD}&wlan_user_ip=&wlan_user_ipv6=&wlan_user_mac=000000000000&wlan_ac_ip=&wlan_ac_name=&jsVersion=${JS_VERSION}&terminal_type=${TERMINAL_TYPE}&lang=${LANG}&v=${V}&lang=${LANG}"

    response=$(curl -s --connect-timeout 5 "$LOGIN_FULL_URL" 2>&1)

    if echo "$response" | grep -q '"result":1'; then
        log "🎉 认证成功！"
        send_wx_msg "📡 Portal 自动认证成功！\n时间：$(date '+%Y-%m-%d %H:%M:%S')\n设备：Linux 自动脚本"
        return 0
    else
        log "❌ 认证失败！响应：${response:0:200}..."  # 截取前200字符，避免日志爆炸
        send_wx_msg "⚠️ Portal 认证失败！\n时间：$(date '+%Y-%m-%d %H:%M:%S')\n响应：${response:0:200}..."
        return 1
    fi
}

# ========== 主程序 ==========
main() {
    log "========== 脚本开始执行 =========="
    if check_online; then
        log "✅ 网络已认证，无需操作。"
    else
        do_login
    fi
    log "========== 脚本执行结束 ==========\n"
}

# 执行主程序
main
```

---

## 📁 日志文件路径说明

默认路径：

```bash
LOG_FILE="/var/log/portal_auth.log"
```



---

## 🧹 可选：日志轮转（防止单个文件过大）

创建日志轮转配置（需 root）：

```bash
sudo tee /etc/logrotate.d/portal_auth <<EOF
/var/log/portal_auth.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    create 644 root root
}
EOF
```

> ✅ 每天轮转，保留7天，自动压缩旧日志。



---

## 🛠️ 使用步骤

### 1. 获取企业微信机器人 Webhook

- 打开企业微信 → 选择一个群 → 添加「群机器人」→ 复制 Webhook URL
- 替换脚本中的：

```bash
WX_WEBHOOK_URL="https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=你的KEY"
```

---

### 2. 测试通知是否正常

权限问题：确保脚本有执行权限：
```bash
chmod +x /path/to/portal_auth.sh
```

你可以手动运行一次（未认证状态下）：

```bash
./portal_auth.sh
```

如果认证成功或失败，你应该在企业微信群里收到通知！

---

### 3. 设置定时任务

```bash
sudo crontab -e
```

添加：

```bash
*/2 * * * * /path/to/portal_auth.sh >/dev/null 2>&1
```

```
*    *    *    *    *    command
│    │    │    │    │
│    │    │    │    └── 星期几 (0-7, 0和7都是星期日)
│    │    │    └────── 月份 (1-12)
│    │    └─────────── 日期 (1-31)
│    └──────────────── 小时 (0-23)
└───────────────────── 分钟 (0-59)
```

###  ✅ `>/dev/null 2>&1` 含义：

**输出重定向**，用于**静默执行**（不产生任何输出或日志）：

- `>`：重定向标准输出（stdout）
- `/dev/null`：Linux 的“黑洞设备”，写入的内容会被丢弃
- `2>&1`：将标准错误（stderr，文件描述符 2）重定向到标准输出（文件描述符 1）
