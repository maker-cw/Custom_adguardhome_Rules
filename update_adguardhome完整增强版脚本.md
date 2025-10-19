#!/bin/bash

# ============================
# AdGuard Home 自动更新脚本
# 适用系统：Debian 13+
# 作者：ChatGPT 优化版
# ============================

# === 基本配置 ===
INSTALL_DIR="/opt/AdGuardHome"         # AdGuardHome 安装目录（可修改）
BACKUP_DIR="$INSTALL_DIR/backup"       # 备份目录
LOG_DIR="/var/log/adguardhome"         # 日志目录
LOG_FILE="$LOG_DIR/update_$(date +'%Y-%m-%d').log"
DOWNLOAD_URL="https://github.com/AdguardTeam/AdGuardHome/releases/latest/download/AdGuardHome_linux_amd64.tar.gz"

# === 确保日志目录存在 ===
mkdir -p "$LOG_DIR"
mkdir -p "$BACKUP_DIR"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

log "开始检测 AdGuard Home 更新..."

# === 获取当前版本 ===
if [ -x "$INSTALL_DIR/AdGuardHome" ]; then
    current_version=$("$INSTALL_DIR/AdGuardHome" --version 2>/dev/null | grep -oE "v[0-9]+\.[0-9]+\.[0-9]+")
elif command -v AdGuardHome >/dev/null 2>&1; then
    current_version=$(AdGuardHome --version 2>/dev/null | grep -oE "v[0-9]+\.[0-9]+\.[0-9]+")
else
    current_version="未知（未检测到 AdGuardHome 程序）"
fi
if [ -z "$current_version" ]; then
    current_version="无法识别版本号"
fi
log "当前版本：$current_version"

# === 获取最新版本号 ===
latest_version=$(curl -s https://api.github.com/repos/AdguardTeam/AdGuardHome/releases/latest | grep '"tag_name":' | sed -E 's/.*"v([^"]+)".*/v\1/')
if [ -z "$latest_version" ]; then
    log "❌ 获取最新版本号失败，请检查网络或 GitHub 连接。"
    exit 1
fi
log "最新版本：$latest_version"

# === 比较版本 ===
if [ "$current_version" = "$latest_version" ]; then
    log "✅ 已是最新版本，无需更新。"
    exit 0
fi

# === 下载新版本 ===
TMP_DIR=$(mktemp -d)
cd "$TMP_DIR" || exit 1
log "🔄 检测到新版本，开始下载..."

if ! curl -L -o AdGuardHome_linux_amd64.tar.gz "$DOWNLOAD_URL"; then
    log "❌ 下载失败，请检查网络连接！"
    exit 1
fi

# === 解压与更新 ===
log "📦 正在解压..."
if ! tar -xzf AdGuardHome_linux_amd64.tar.gz; then
    log "❌ 解压失败，文件可能损坏。"
    exit 1
fi

# === 停止当前服务 ===
log "🛑 停止 AdGuard Home..."
if systemctl is-active --quiet AdGuardHome; then
    systemctl stop AdGuardHome
else
    pkill -f AdGuardHome 2>/dev/null
fi

# === 备份旧版本 ===
BACKUP_NAME="AdGuardHome_$(date +'%Y%m%d_%H%M%S')"
log "📁 备份旧版本至 $BACKUP_DIR/$BACKUP_NAME"
mkdir -p "$BACKUP_DIR/$BACKUP_NAME"
cp -r "$INSTALL_DIR"/* "$BACKUP_DIR/$BACKUP_NAME"/

# === 覆盖安装 ===
log "🚀 安装新版本..."
cp -f AdGuardHome/AdGuardHome "$INSTALL_DIR/AdGuardHome"
chmod +x "$INSTALL_DIR/AdGuardHome"

# === 重启服务 ===
log "🔁 启动 AdGuard Home..."
if systemctl list-units --type=service | grep -q "AdGuardHome.service"; then
    systemctl start AdGuardHome
else
    nohup "$INSTALL_DIR/AdGuardHome" &>/dev/null &
fi

log "✅ AdGuard Home 已成功更新至版本：$latest_version"

# === 清理临时文件 ===
rm -rf "$TMP_DIR"

# === 清理 30 天前日志 ===
find "$LOG_DIR" -type f -mtime +30 -name "*.log" -exec rm -f {} \;
log "🧹 已清理超过 30 天的旧日志。"

exit 0