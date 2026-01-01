明白你的需求。既然 GeoASN 的 UI 显示同步比较麻烦，且你不想通过脚本更新它了，那我们就回归到最稳健的 GeoIP + GeoSite 双更新脚本。

我会去掉 ASN 相关部分，同时保留强制刷新 UI 显示时间的逻辑，确保你运行脚本后，OpenClash 面板上的 GeoIP 和 GeoSite 时间会同步更新。

最终优化版：GeoIP + GeoSite 双更新脚本
请使用 vi /root/update_clash_all.sh 命令，将内容全部替换为以下代码：

'''
#!/bin/sh

# === 配置路径 ===
OC_DIR="/etc/openclash"
IP_DEST="$OC_DIR/Country.mmdb"
SITE_DEST="$OC_DIR/GeoSite.dat"

IP_TMP="/tmp/Country.mmdb"
SITE_TMP="/tmp/geosite_new.dat"

# === 验证过的镜像源 (国内 CDN 加速) ===
# 1. GeoIP (MMDB 格式，最稳健的源)
IP_MMDB_URL="https://testingcf.jsdelivr.net/gh/Loyalsoldier/geoip@release/Country.mmdb"
# 2. GeoSite
SITE_URL="https://testingcf.jsdelivr.net/gh/MetaCubeX/meta-rules-dat@release/geosite.dat"

echo "[$(date)] 开始同步 GeoIP 和 GeoSite 数据库..."

# --- 1. 更新 GeoIP ---
echo "正在下载 GeoIP..."
wget --no-check-certificate $IP_MMDB_URL -O $IP_TMP
if [ $? -eq 0 ] && [ $(ls -l $IP_TMP | awk '{print $5}') -gt 3145728 ]; then
    mv $IP_TMP $IP_DEST
    # 同步更新 UI 显示的时间戳
    echo "$(date "+%Y-%m-%d %H:%M:%S")" > /usr/share/openclash/res/geoip_version
    echo "GeoIP 更新成功并已同步 UI 时间。"
else
    echo "GeoIP 下载失败或文件异常。"
    rm -f $IP_TMP
fi

# --- 2. 更新 GeoSite ---
echo "正在下载 GeoSite..."
wget --no-check-certificate $SITE_URL -O $SITE_TMP
if [ $? -eq 0 ] && [ $(ls -l $SITE_TMP | awk '{print $5}') -gt 3145728 ]; then
    mv $SITE_TMP $SITE_DEST
    # 同步更新 UI 显示的时间戳
    echo "$(date "+%Y-%m-%d %H:%M:%S")" > /usr/share/openclash/res/geosite_version
    echo "GeoSite 更新成功并已同步 UI 时间。"
else
    echo "GeoSite 下载失败。"
    rm -f $SITE_TMP
fi

# --- 3. 强制刷新 OpenClash 运行状态 ---
# 这行命令会通知 OpenClash 重新读取文件，确保新数据库生效
/etc/init.d/openclash reload

echo "[$(date)] 数据库更新任务已完成。"



'''


操作说明：
修改脚本： 执行 vi /root/update_clash_all.sh，删除旧内容，粘贴上面的新代码，保存退出（:wq）。


第二步：赋予执行权限并测试
赋予权限：

Bash

chmod +x /root/update_clash_all.sh

手动测试： 执行 /bin/sh /root/update_clash_all.sh。


第四步：设置 Cron 定时任务
建议将此全量更新设置在每周的低峰时段。

进入 OpenWrt 系统 -> 计划任务。

在末尾添加这一行（设定为每周一凌晨 4:30 运行）：

Bash

30 4 * * 1 /bin/sh /root/update_clash_all.sh > /dev/null 2>&1

检查 UI： 刷新 OpenClash 页面。

GeoIP 和 GeoSite 的更新时间现在应该显示为刚才执行脚本的时间。

GeoASN 因为不在脚本内更新，它会保持原样（你可以手动在插件界面点一次 ASN 更新，之后就不用管它了，ASN 数据库通常变动极小）。

为什么这次 UI 会显示？
我在代码中加入了 echo "$(date ...)" > /usr/share/openclash/res/geoip_version。这是 OpenClash 插件读取 “上次更新时间” 的原始文件。通过手动改写它，插件就会认为 “自动更新” 已经成功完成，从而在网页上显示正确的时间。

现在的配置既解决了系统卡死断网的问题，又兼顾了 UI 界面美观，建议保持这个方案长期运行。

