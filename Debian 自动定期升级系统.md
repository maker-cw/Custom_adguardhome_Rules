一、脚本功能概述

这个脚本将完成以下任务：

步骤	功能
1️⃣	自动更新系统软件源索引（apt update）
2️⃣	自动升级所有可用更新（apt upgrade -y）
3️⃣	自动清理多余软件包（apt autoremove -y）
4️⃣	记录详细日志到 /var/log/auto_upgrade.log
5️⃣	每次运行时记录时间、升级前后磁盘空间对比
6️⃣	自动清理超过 30 天的旧日志文件
💻 二、编写脚本
1️⃣ 新建脚本文件

在 Debian 上执行：sudo nano /usr/local/bin/auto_upgrade.sh

2️⃣ 粘贴以下内容：

'''
#!/bin/bash
# ==========================================
# Debian 自动升级系统脚本
# 作者：ChatGPT + 你 😊
# 功能：自动更新系统 + 清理旧包 + 日志记录
# ==========================================

LOGFILE="/var/log/auto_upgrade.log"

# 将所有输出写入日志文件（含错误输出）
exec > >(tee -a "$LOGFILE") 2>&1

echo "=========================================="
echo "🚀 系统自动升级开始：$(date)"
echo "------------------------------------------"

# 记录升级前磁盘空间
echo "📊 升级前磁盘使用情况："
df -h /

# 更新APT索引
echo "👉 正在更新软件包索引..."
apt-get update -y

# 升级系统
echo "👉 正在升级系统..."
apt-get upgrade -y

# 可选：升级发行版（谨慎开启）
# echo "👉 正在执行 dist-upgrade..."
# apt-get dist-upgrade -y

# 清理多余的包
echo "👉 清理无用包和缓存..."
apt-get autoremove -y
apt-get autoclean -y

# 记录升级后磁盘空间
echo "------------------------------------------"
echo "📊 升级后磁盘使用情况："
df -h /

# 清理旧日志（30天前）
echo "👉 清理旧日志（超过30天）..."
find /var/log -type f -name "auto_upgrade.log*" -mtime +30 -delete

echo "✅ 系统升级完成：$(date)"
echo "=========================================="
echo


'''

3️⃣ 保存并退出

按：

Ctrl + O → 回车 → Ctrl + X

4️⃣ 添加执行权限
sudo chmod +x /usr/local/bin/auto_upgrade.sh

⏰ 三、设置定时任务（cron）

让系统自动执行脚本（比如每周日凌晨 4 点）：

sudo crontab -e


在文件顶部加上 PATH 环境变量（防止找不到命令）：

PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin


然后在文件末尾添加一行：

0 4 * * 0 /usr/local/bin/auto_upgrade.sh >> /var/log/auto_upgrade.log 2>&1


说明：

0 4 * * 0 → 每周日凌晨 4 点执行

>> 表示追加日志

2>&1 表示捕获错误输出

🧾 四、日志查看

查看最近日志：

sudo tail -n 30 /var/log/auto_upgrade.log


清空日志（若太大）：

sudo truncate -s 0 /var/log/auto_upgrade.log

🧰 五、立即测试运行

你可以立即手动执行一次验证：

sudo /usr/local/bin/auto_upgrade.sh


如果一切正常，你会看到：

🚀 系统自动升级开始：Sat Oct 12 06:00:00 2025
👉 正在更新软件包索引...
👉 正在升级系统...
👉 清理无用包和缓存...
✅ 系统升级完成

💡 六、建议的运行频率
使用场景	建议周期
家庭服务器（稳定型）	每周 1 次（周日凌晨）
实验环境 / Docker 构建机	每天凌晨 3 点
核心网络服务机	手动执行（防止意外升级中断服务）

