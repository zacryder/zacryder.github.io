````
#!/bin/bash

set -euo pipefail

# ----------------------------
# 全局变量定义
# ----------------------------
LOG_FILE="/var/log/security_hardening_$(date +%Y%m%d).log"
BACKUP_DIR="/root/security_backup_$(date +%Y%m%d%H%M%S)"
SCRIPT_NAME=$(basename "$0")

# 网络白名单（可外部修改）
SSH_ALLOWED_NETWORKS=(
    "10.18.10.0/255.255.255.0"
    "219.144.190.16/255.255.255.252"
    "192.168.101.0/255.255.255.0"
)

# 日志函数
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

error() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] ERROR: $1" >&2 | tee -a "$LOG_FILE"
    exit 1
}

warn() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] WARNING: $1" | tee -a "$LOG_FILE"
}

# ----------------------------
# 系统权限与兼容性检查
# ----------------------------
if [[ $EUID -ne 0 ]]; then
    error "此脚本必须以 root 权限运行"
fi

# 检查是否为 Ubuntu 22.04
if [[ ! -f /etc/os-release ]]; then
    error "/etc/os-release 不存在，无法识别系统"
fi

source /etc/os-release
if [[ "$ID" != "ubuntu" ]] || [[ "$VERSION_ID" != "22.04" ]]; then
    warn "当前系统为 $ID $VERSION_ID，仅测试于 Ubuntu 22.04，可能存在兼容性问题"
fi

log "开始安全加固流程"

# 创建备份目录
mkdir -p "$BACKUP_DIR"
log "备份目录已创建: $BACKUP_DIR"

# ----------------------------
# 基础依赖安装
# ----------------------------
log "=== 安装必要依赖 ==="
apt update -y >> "$LOG_FILE" 2>&1
apt install -y auditd libpam-pwquality rsyslog >> "$LOG_FILE" 2>&1

# ----------------------------
# 安全审计配置 (auditd)
# ----------------------------
log "=== 配置安全审计 (auditd) ==="

for service in auditd rsyslog; do
    if systemctl is-active --quiet "$service"; then
        systemctl reload-or-restart "$service" >> "$LOG_FILE" 2>&1
    else
        systemctl start "$service" >> "$LOG_FILE" 2>&1
    fi
    systemctl enable "$service" >> "$LOG_FILE" 2>&1
done

AUDIT_RULES_FILE="/etc/audit/rules.d/10-security.rules"
if [[ -f "$AUDIT_RULES_FILE" ]]; then
    cp -p "$AUDIT_RULES_FILE" "$BACKUP_DIR/"
    log "已备份 $AUDIT_RULES_FILE"
fi

cat << 'EOF' > "$AUDIT_RULES_FILE"
# 文件删除操作审计
-a always,exit -F arch=b64 -S unlink,unlinkat,rename,renameat -F auid>=1000 -F auid!=4294967295 -k file_delete
-a always,exit -F arch=b32 -S unlink,unlinkat,rename,renameat -F auid>=1000 -F auid!=4294967295 -k file_delete

# 身份文件审计
-w /etc/group -p wa -k identity
-w /etc/passwd -p wa -k identity
-w /etc/gshadow -p wa -k identity
-w /etc/shadow -p wa -k identity
-w /etc/security/opasswd -p wa -k identity

# 权限变更审计
-w /etc/sudoers -p wa -k privilege_change
-w /etc/sudoers.d/ -p wa -k privilege_change
EOF

# 确保 auditd 正在运行再加载规则
systemctl is-active auditd || systemctl start auditd
augenrules --load >> "$LOG_FILE" 2>&1
systemctl reload-or-restart auditd >> "$LOG_FILE" 2>&1

# ----------------------------
# 访问控制与权限修复
# ----------------------------
log "=== 配置访问控制与权限修复 ==="

# 配置 su：仅允许 sudo 组
SU_PAM_FILE="/etc/pam.d/su"
if [[ -f "$SU_PAM_FILE" ]]; then
    cp -p "$SU_PAM_FILE" "$BACKUP_DIR/"
    if ! grep -q "pam_group.so" "$SU_PAM_FILE"; then
        sed -i '/pam_rootok.so/a auth       sufficient   pam_group.so trust use_uid' "$SU_PAM_FILE"
        log "已启用 pam_group.so 限制 su 权限"
    fi
else
    warn "$SU_PAM_FILE 不存在，无法限制 su"
fi

# 设置全局 umask
if ! grep -q "umask 027" /etc/profile; then
    echo "umask 027" >> /etc/profile
    log "已设置 umask 027"
fi

# 修复用户家目录权限
find /home -mindepth 1 -maxdepth 1 -type d | while read -r homedir; do
    user=$(basename "$homedir")
    if id "$user" &>/dev/null && [[ "$user" != "root" ]]; then
        chown "$user:$user" "$homedir"
        chmod 750 "$homedir"
        log "已修复用户 $user 家目录权限: 750"
    fi
done

# SSH 安全配置
SSH_CONFIG="/etc/ssh/sshd_config"
if [[ -f "$SSH_CONFIG" ]]; then
    cp -p "$SSH_CONFIG" "$BACKUP_DIR/"
    
    # 使用正则精确替换或添加
    sed -i -E 's/^#?(PermitRootLogin)\s+.*/\1 no/' "$SSH_CONFIG"
    sed -i -E 's/^#?(MaxAuthTries)\s+.*/\1 5/' "$SSH_CONFIG"
    sed -i '/^AllowGroups/d' "$SSH_CONFIG"
    echo "AllowGroups sudo" >> "$SSH_CONFIG"
    
    # 重载 SSH 服务
    systemctl reload sshd >> "$LOG_FILE" 2>&1
    log "SSH 配置已更新"
else
    error "$SSH_CONFIG 不存在，无法配置 SSH"
fi

# 锁定无关系统账户
for user in shutdown halt; do
    if id "$user" &>/dev/null; then
        usermod -L "$user" 2>/dev/null || true
        passwd -l "$user" 2>/dev/null || true
        log "已锁定用户: $user"
    fi
done

# 关键文件权限
chmod 644 /etc/passwd /etc/group
chmod 600 /etc/shadow /etc/gshadow
chmod 600 /etc/ssh/sshd_config
chown root:root /etc/ssh/sshd_config
log "关键文件权限已修复"

# ----------------------------
# 入侵防范：TCP Wrappers
# ----------------------------
log "=== 配置网络访问控制 (TCP Wrappers) ==="

# 生成 hosts.allow
> /etc/hosts.allow
for net in "${SSH_ALLOWED_NETWORKS[@]}"; do
    echo "sshd : $net" >> /etc/hosts.allow
done
echo "ALL : ALL" > /etc/hosts.deny

log "TCP Wrappers 已配置，允许 SSH 来源: ${SSH_ALLOWED_NETWORKS[*]}"

# ----------------------------
# 身份鉴别：PAM 与密码策略
# ----------------------------
log "=== 配置身份鉴别策略 ==="

PAM_AUTH="/etc/pam.d/common-auth"

if [[ -f "$PAM_AUTH" ]]; then
    cp -p "$PAM_AUTH" "$BACKUP_DIR/"

    # 确保配置顺序正确
    cat > "$PAM_AUTH" <<EOF
auth    required    pam_faillock.so preauth silent deny=5 unlock_time=900
auth    optional    pam_faillock.so authsucc
auth    [success=1 default=ignore]      pam_unix.so nullok
auth    required    pam_faillock.so authfail deny=5 unlock_time=900
EOF

    log "账户锁定策略已配置 (5次失败，锁定15分钟)"
else
    warn "$PAM_AUTH 不存在，跳过账户锁定配置"
fi

# 密码复杂度 (pwquality)
PAM_PASS="/etc/pam.d/common-password"
BACKUP_DIR="/path/to/backup"
LOG_FILE="/var/log/pam_config.log"

if [[ -f "$PAM_PASS" ]]; then
    # 创建备份目录（如果不存在）
    mkdir -p "$BACKUP_DIR"
    if ! cp -p "$PAM_PASS" "$BACKUP_DIR/$(basename $PAM_PASS).$(date +%Y%m%d%H%M%S)"; then
        echo "[$(date)] 备份失败: $PAM_PASS" >> "$LOG_FILE"
        exit 1
    fi

    # 删除旧的 pam_pwquality.so 配置
    if ! sed -i '/^password.*pam_pwquality.so/d' "$PAM_PASS"; then
        echo "[$(date)] 删除旧配置失败" >> "$LOG_FILE"
        exit 1
    fi

    # 插入新配置到 pam_unix.so 前
    if grep -q '^password.*pam_unix.so' "$PAM_PASS"; then
        if ! sed -i '/^password.*pam_unix\.so/i password    requisite     pam_pwquality.so try_first_pass retry=5 minlen=11 minclass=3 enforce_for_root' "$PAM_PASS"; then
            echo "[$(date)] 插入新配置失败" >> "$LOG_FILE"
            exit 1
        fi
        echo "[$(date)] 密码复杂度策略已设置 (minlen=11, minclass=3)" >> "$LOG_FILE"
    else
        echo "[$(date)] 未找到 pam_unix.so 行，跳过插入" >> "$LOG_FILE"
    fi
else
    echo "[$(date)] $PAM_PASS 不存在，跳过密码复杂度配置" >> "$LOG_FILE"
fi



# 全局密码策略
LOGIN_DEFS="/etc/login.defs"
for param in PASS_MAX_DAYS PASS_MIN_DAYS PASS_WARN_AGE; do
    sed -i "/^$param/d" "$LOGIN_DEFS" 2>/dev/null || true
done

{
    echo "PASS_MAX_DAYS   90"
    echo "PASS_MIN_DAYS   7"
    echo "PASS_WARN_AGE   7"
} >> "$LOGIN_DEFS"

# 应用到 guangxigw 用户
if id guangxigw &>/dev/null; then
    chage --maxdays 90 --mindays 7 --warndays 7 guangxigw
    log "已为用户 guangxigw 应用密码策略"
fi

# 会话超时
TIMEOUT_SCRIPT="/etc/profile.d/99-timeout.sh"
cat > "$TIMEOUT_SCRIPT" << 'EOF'
readonly TMOUT=900
export TMOUT
EOF
chmod +x "$TIMEOUT_SCRIPT"
log "会话超时已设置: 900秒"

# ----------------------------
# 清理空密码账户
# ----------------------------
log "=== 清理空密码账户 ==="
awk -F: '($2 == "" && $1 != "root") {print $1}' /etc/shadow | while read -r user; do
    if [[ -n "$user" ]]; then
        passwd -l "$user"
        log "已锁定空密码用户: $user"
    fi
done

# ----------------------------
# 配置验证与报告
# ----------------------------
log "=== 配置验证报告 ==="

echo -e "\n【服务状态】" | tee -a "$LOG_FILE"
for svc in auditd rsyslog sshd; do
    if systemctl is-active "$svc" >/dev/null; then
        echo "✅ $svc 正在运行" | tee -a "$LOG_FILE"
    else
        echo "❌ $svc 未运行" | tee -a "$LOG_FILE"
    fi
done

echo -e "\n【SSH 配置】" | tee -a "$LOG_FILE"
grep -E "PermitRootLogin|MaxAuthTries|AllowGroups" /etc/ssh/sshd_config | sed 's/^/   /' | tee -a "$LOG_FILE"

echo -e "\n【密码策略】" | tee -a "$LOG_FILE"
grep -E "^PASS_MAX_DAYS|^PASS_MIN_DAYS|^PASS_WARN_AGE" /etc/login.defs | sed 's/^/   /' | tee -a "$LOG_FILE"

echo -e "\n【审计规则】" | tee -a "$LOG_FILE"
rule_count=$(auditctl -l 2>/dev/null | wc -l || echo 0)
echo "   加载规则数: $rule_count 条" | tee -a "$LOG_FILE"

echo -e "\n【关键文件权限】" | tee -a "$LOG_FILE"
ls -l /etc/passwd /etc/shadow /etc/group /etc/gshadow /etc/ssh/sshd_config 2>/dev/null | sed 's/^/   /' | tee -a "$LOG_FILE"

# ----------------------------
# 完成提示
# ----------------------------
log "✅ 安全加固完成"
cat << EOF

==================================================
✅ 安全加固已完成！
📁 备份路径: $BACKUP_DIR
📄 日志文件: $LOG_FILE
💡 建议：
   1. 保留当前会话，测试新 SSH 登录是否正常
   2. 验证 sudo 用户能否 su 切换
   3. 检查 guangxigw 用户密码策略是否生效
   4. 生产环境建议择机重启以加载审计规则
   5. 根据实际网络环境调整 /etc/hosts.allow
==================================================
EOF`
````