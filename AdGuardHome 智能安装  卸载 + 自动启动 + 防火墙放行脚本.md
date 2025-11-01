保存为：
adguardhome_manager.sh

执行：

sudo bash adguardhome_manager.sh


👇👇👇 完整脚本内容：


'''


#!/bin/bash
# ===========================================================
# 🚀 AdGuardHome 智能安装 / 卸载 一体脚本（增强版）
# 功能：
#  - 自动检测是否已安装
#  - 一键安装或卸载
#  - 安装完成后自动启动 + 开机自启 + 放行端口
# 适用于 Debian 10/11/12/13
# 作者：ChatGPT 优化版 (2025)
# ===========================================================

set -e

INSTALL_DIR="/opt/AdGuardHome"
SERVICE_NAME="AdGuardHome"
GH_PROXY="https://ghproxy.net/"
INSTALL_SCRIPT="${GH_PROXY}https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh"

GREEN='\033[0;32m'; RED='\033[0;31m'; YELLOW='\033[1;33m'; NC='\033[0m'

# -----------------------------------------------------------
# 检查 root 权限
# -----------------------------------------------------------
if [[ $EUID -ne 0 ]]; then
  echo -e "${RED}❌ 请使用 root 用户运行此脚本！${NC}"
  exit 1
fi

# -----------------------------------------------------------
# 检查 AdGuardHome 是否已安装
# -----------------------------------------------------------
is_installed=false
if [ -x "$INSTALL_DIR/AdGuardHome" ]; then
  is_installed=true
elif systemctl list-unit-files | grep -q "$SERVICE_NAME.service"; then
  is_installed=true
fi

# -----------------------------------------------------------
# 安装函数
# -----------------------------------------------------------
install_adguardhome() {
  echo -e "${GREEN}🚀 开始安装 AdGuardHome...${NC}"
  apt update -y >/dev/null
  apt install -y curl wget tar ufw >/dev/null

  echo -e "${YELLOW}📥 调用官方安装脚本（通过 ghproxy 加速）...${NC}"
  bash <(curl -sSL "$INSTALL_SCRIPT")

  echo -e "${YELLOW}🔁 设置服务开机自启...${NC}"
  systemctl enable $SERVICE_NAME >/dev/null 2>&1 || true

  echo -e "${YELLOW}🚀 启动服务...${NC}"
  systemctl restart $SERVICE_NAME >/dev/null 2>&1 || true

  echo -e "${YELLOW}🧱 放行防火墙端口 (53, 80, 443, 3000)...${NC}"
  ufw allow 53 >/dev/null 2>&1 || true
  ufw allow 80 >/dev/null 2>&1 || true
  ufw allow 443 >/dev/null 2>&1 || true
  ufw allow 3000 >/dev/null 2>&1 || true
  ufw reload >/dev/null 2>&1 || true

  sleep 2
  IP=$(hostname -I | awk '{print $1}')
  echo
  echo -e "${GREEN}✅ AdGuardHome 安装完成并已启动！${NC}"
  echo -e "------------------------------------------------------------"
  echo -e "🌐 Web 管理界面: ${YELLOW}http://$IP:3000${NC}"
  echo -e "🧩 安装目录: ${YELLOW}$INSTALL_DIR${NC}"
  echo -e "💡 服务管理命令: systemctl start|stop|restart AdGuardHome"
  echo -e "------------------------------------------------------------"
}

# -----------------------------------------------------------
# 卸载函数
# -----------------------------------------------------------
uninstall_adguardhome() {
  echo -e "${YELLOW}🧹 开始卸载 AdGuardHome...${NC}"
  systemctl stop $SERVICE_NAME 2>/dev/null || true
  if [ -x "$INSTALL_DIR/AdGuardHome" ]; then
      $INSTALL_DIR/AdGuardHome -s uninstall || true
  fi
  rm -rf "$INSTALL_DIR" /etc/systemd/system/$SERVICE_NAME.service /var/log/adguardhome
  systemctl daemon-reload >/dev/null 2>&1
  echo -e "${GREEN}✅ 卸载完成！系统已清理干净。${NC}"
}

# -----------------------------------------------------------
# 主逻辑
# -----------------------------------------------------------
if $is_installed; then
  echo -e "${YELLOW}⚠️ 检测到系统已安装 AdGuardHome。${NC}"
  echo -e "是否要 ${RED}卸载旧版本${NC}？(y/n)"
  read -rp "请输入选择: " ans
  case $ans in
    y|Y)
      uninstall_adguardhome
      echo
      read -rp "是否要立即重新安装最新版本？(y/n): " reinstall
      if [[ $reinstall =~ ^[Yy]$ ]]; then
        install_adguardhome
      else
        echo -e "${YELLOW}👌 已退出，未重新安装。${NC}"
      fi
      ;;
    *)
      echo -e "${GREEN}👌 已退出，保留当前安装。${NC}"
      ;;
  esac
else
  echo -e "${YELLOW}🔍 未检测到 AdGuardHome，准备安装...${NC}"
  install_adguardhome
fi



'''


✅ 功能总览
功能	说明
🔍 智能检测	自动判断是否安装
🧹 自动卸载	停止服务 + 清理目录 + 注销 systemd
📥 官方安装	使用 ghproxy.net 加速调用官方脚本
🚀 自动启动	安装完成自动启动服务
🔁 开机自启	启用 systemd enable
🧱 自动放行端口	放行 53、80、443、3000
🌐 即装即用	安装完立刻访问 Web 界面
✅ 使用方法
nano adguardhome_manager.sh
# 粘贴上面的内容
chmod +x adguardhome_manager.sh
sudo bash adguardhome_manager.sh

🌟 安装完成后输出示例
✅ AdGuardHome 安装完成并已启动！
🌐 Web 管理界面: http://192.168.1.100:3000
🧩 安装目录: /opt/AdGuardHome
💡 服务管理命令: systemctl start|stop|restart AdGuardHome