🌟 新增功能（v7.0）

🕒 每次系统启动时，自动检测 DNS 是否可用；
如果主 DNS 不通，自动切换到备用公共 DNS（223.5.5.5 / 8.8.8.8）。

⚙️ 实现逻辑（简单说就是自动自愈机制）：

1️⃣ 系统启动后执行一个检测脚本 /usr/local/bin/dns_health_check.sh
2️⃣ 它尝试 ping 当前 /etc/resolv.conf 里的 DNS；
3️⃣ 如果失败，就自动写入公共 DNS 并重启网络；
4️⃣ 这样即使 RouterOS / OpenWrt 临时异常，Debian 也会自动恢复外网连接。

🚀 Debian v7.0 智能 DNS 自愈 + 启动检测版脚本

📄 路径：/usr/local/bin/fix_static_ip.sh
（直接覆盖原文件即可）


"""

#!/bin/bash
# ============================================================
# Debian 12/13 一键静态 IP + 智能 DNS 自愈 + 启动检测 + AdGuardHome 自动修复
# 版本：v7.0（2025-11-01）
# 作者：ChatGPT 助手（教师特制稳定版）
# ============================================================

set -e
BACKUP_DIR="/root/network_backup"
mkdir -p "$BACKUP_DIR"

echo "🚀 Debian v7.0 智能网络修复脚本启动..."

# 🧠 必须 root
if [ "$EUID" -ne 0 ]; then
  echo "❌ 请使用 root 运行本脚本。"
  exit 1
fi

# 🧭 检测主网卡
iface=$(ip -o link show | awk -F': ' '{print $2}' | grep -v lo | head -n1)
echo "➡️ 检测到网卡：$iface"

# 💾 自动备份旧配置
timestamp=$(date +%Y%m%d_%H%M%S)
backup_path="${BACKUP_DIR}/backup_${timestamp}"
mkdir -p "$backup_path"
[ -f /etc/resolv.conf ] && cp /etc/resolv.conf "$backup_path"/resolv.conf
[ -d /etc/systemd/network ] && cp /etc/systemd/network/*.network "$backup_path"/ 2>/dev/null || true
echo "💾 已备份旧配置到：$backup_path"

# 🧩 自动检测参数
current_ip=$(ip -4 addr show $iface | grep -oP '(?<=inet\s)\d+(\.\d+){3}' | head -n1)
current_prefix=$(ip -4 addr show $iface | grep -oP '(?<=/)\d+' | head -n1)
current_gw=$(ip route | grep default | awk '{print $3}' | head -n1)
router_ip=${current_gw:-"192.168.1.1"}

echo "🧠 当前检测："
echo "  IP: $current_ip/$current_prefix"
echo "  网关: $current_gw"
echo

# 🧮 用户输入
read -p "请输入要固定的 IP（默认：$current_ip）: " ipaddr
ipaddr=${ipaddr:-$current_ip}
read -p "请输入子网掩码前缀（默认：$current_prefix）: " prefix
prefix=${prefix:-$current_prefix}
read -p "请输入默认网关（默认：$current_gw）: " gateway
gateway=${gateway:-$current_gw}
read -p "请输入首选 DNS（默认：自动检测）: " dns_input
dns_input=${dns_input:-"auto"}

# 🧠 智能 DNS 检测
echo
if [ "$dns_input" == "auto" ]; then
  echo "🔍 正在检测网络外联能力..."
  ping -c 1 -W 1 223.5.5.5 >/dev/null 2>&1 && dns_choice="223.5.5.5 8.8.8.8" || dns_choice="$router_ip"
  echo "✅ 已自动选择 DNS：$dns_choice"
else
  dns_choice="$dns_input"
  echo "✅ 使用用户指定 DNS：$dns_choice"
fi

# 📝 写入静态配置文件
conf_file="/etc/systemd/network/10-static.network"
echo "📝 写入静态 IP 配置：$conf_file"
cat <<EOF > $conf_file
[Match]
Name=$iface

[Network]
Address=$ipaddr/$prefix
Gateway=$gateway
DHCP=no
EOF

for d in $dns_choice; do
  echo "DNS=$d" >> $conf_file
done

echo "✅ 静态网络配置文件已生成。"

# 🚫 禁用 DHCP 客户端 & systemd-resolved
for svc in dhcpcd systemd-resolved NetworkManager networking; do
  if systemctl list-unit-files | grep -q "$svc"; then
    echo "🧹 禁用服务：$svc"
    systemctl stop $svc 2>/dev/null || true
    systemctl disable $svc 2>/dev/null || true
    systemctl mask $svc 2>/dev/null || true
  fi
done

# 🧾 写 resolv.conf
echo "🧾 写入新的 /etc/resolv.conf ..."
rm -f /etc/resolv.conf
for d in $dns_choice; do
  echo "nameserver $d" >> /etc/resolv.conf
done
chmod 644 /etc/resolv.conf

# 🔧 启用 networkd
systemctl enable systemd-networkd
systemctl enable systemd-networkd-wait-online
systemctl restart systemd-networkd

# 🌐 测试网络
echo
echo "🌐 检测网络连通性..."
ping -c 2 8.8.8.8 >/dev/null 2>&1 && net_ok=true || net_ok=false
if [ "$net_ok" = true ]; then
    echo "✅ 网络正常，可访问外网。"
else
    echo "⚠️ 网络不通，尝试修复..."
    systemctl restart systemd-networkd
    sleep 5
    ping -c 2 8.8.8.8 >/dev/null 2>&1 && echo "✅ 修复成功！" || echo "❌ 请检查网关或 IP。"
fi

# 🧠 检测并重启 AdGuardHome
echo
echo "🧠 检查 AdGuardHome 状态..."
if systemctl list-unit-files | grep -q AdGuardHome.service; then
  if systemctl is-active --quiet AdGuardHome; then
    echo "♻️ 重启 AdGuardHome..."
    systemctl restart AdGuardHome
    sleep 3
    systemctl is-active --quiet AdGuardHome && echo "✅ AdGuardHome 已成功重启。" || echo "⚠️ 重启失败。"
  else
    echo "⚙️ 尝试启动 AdGuardHome..."
    systemctl start AdGuardHome
    sleep 3
    systemctl is-active --quiet AdGuardHome && echo "✅ AdGuardHome 已启动！" || echo "⚠️ 启动失败。"
  fi
else
  echo "ℹ️ 未检测到 AdGuardHome 服务。"
fi

# 🌙 创建开机 DNS 检测脚本
echo "🛠️ 创建 DNS 自动检测守护脚本..."
cat <<'EOF' > /usr/local/bin/dns_health_check.sh
#!/bin/bash
# 检查 DNS 可用性并自动修复 resolv.conf
PING_TARGET="223.5.5.5"
RESOLV_FILE="/etc/resolv.conf"
ROUTER_IP=$(ip route | grep default | awk '{print $3}' | head -n1)

if ! ping -c 1 -W 1 $PING_TARGET >/dev/null 2>&1; then
  echo "⚠️ DNS 不可用，切换为公共 DNS..."
  echo "nameserver 223.5.5.5" > $RESOLV_FILE
  echo "nameserver 8.8.8.8" >> $RESOLV_FILE
  systemctl restart systemd-networkd
else
  echo "✅ DNS 正常。"
fi
EOF

chmod +x /usr/local/bin/dns_health_check.sh

# ⏰ 注册 systemd 定时任务（开机执行）
cat <<EOF > /etc/systemd/system/dns-healthcheck.service
[Unit]
Description=Check and fix DNS at boot
After=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/dns_health_check.sh

[Install]
WantedBy=multi-user.target
EOF

systemctl enable dns-healthcheck.service

echo
echo "✅ 已启用 DNS 开机检测服务！"
echo "（每次开机将自动检测并修复 DNS）"

echo
echo "🔍 当前 IP："
ip addr show $iface | grep "inet "
echo "🔍 当前 DNS："
cat /etc/resolv.conf

echo
echo "🎯 静态 IP + 智能 DNS + 启动自愈 已设置完成！"
echo "建议执行 reboot 后验证："
echo "    ping -c 3 www.baidu.com"
echo "    systemctl status dns-healthcheck.service"




"""

🧰 使用方法

1️⃣ 保存为脚本

nano /usr/local/bin/fix_static_ip.sh


2️⃣ 添加执行权限

chmod +x /usr/local/bin/fix_static_ip.sh


3️⃣ 执行配置

/usr/local/bin/fix_static_ip.sh


4️⃣ 如果配置出错或网络断了，也别慌👇

/usr/local/bin/fix_static_ip.sh restore


它会自动还原上一次的配置，网络立即恢复。