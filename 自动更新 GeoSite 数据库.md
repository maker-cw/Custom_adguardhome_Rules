既然确定了导致卡死的是 “自动更新 GeoSite 数据库”，那问题的指向就更明确了。GeoSite 的文件规模和处理逻辑与 GeoIP 有显著区别，这正是导致路由器崩溃的 “罪魁祸首”。

以下是针对 GeoSite 自动更新导致系统卡死的原因深度分析：


这个方案的核心思路是：用外部脚本在后台悄悄下载，绕过 OpenClash 插件图形化界面复杂的处理流程，从而避免系统瞬间过载卡死。

以下是详细的操作步骤：

第一步：准备脚本文件
连接路由器： 使用 SSH 工具（如 PuTTY, Termius 或 macOS 终端）连接到你的 OpenWrt。

创建脚本： 输入以下命令创建并编辑脚本文件：

Bash
'''
vi /root/update_geosite.sh

'''


粘贴代码： 按下键盘上的 i 键进入编辑模式，将以下代码完整复制进去：

'''

#!/bin/sh

# 1. 定义路径
DEST_PATH="/etc/openclash/GeoSite.dat"
TMP_PATH="/tmp/geosite_new.dat"
# 国内镜像源 (由 MetaCubeX 维护，通过 jsdelivr 加速)
URL="https://testingcf.jsdelivr.net/gh/MetaCubeX/meta-rules-dat@release/geosite.dat"

echo "[$(date)] 开始更新 GeoSite..."

# 2. 下载文件到内存临时目录 /tmp (不占用 Flash 寿命)
wget --no-check-certificate $URL -O $TMP_PATH

# 3. 检查下载是否成功
if [ $? -eq 0 ]; then
    # 获取文件大小 (字节)
    FILE_SIZE=$(ls -l $TMP_PATH | awk '{print $5}')
    
    # 检查文件是否合法 (GeoSite 通常 > 3MB)
    if [ "$FILE_SIZE" -gt 3145728 ]; then
        echo "下载成功，正在替换旧文件..."
        mv $TMP_PATH $DEST_PATH
        echo "更新完成，当前文件大小: $((FILE_SIZE / 1024 / 1024)) MB"
    else
        echo "错误：下载的文件太小 ($FILE_SIZE 字节)，可能已损坏，取消替换。"
        rm -f $TMP_PATH
    fi
else
    echo "错误：下载失败，请检查网络。"
fi

'''


保存退出： 按下 Esc 键，然后输入 :wq 并回车。

第二步：赋予执行权限并测试
在 SSH 中执行以下命令，确保脚本可以运行：

赋予权限：

Bash

chmod +x /root/update_geosite.sh
手动测试运行：

Bash

sh /root/update_geosite.sh
如果脚本运行正常且路由器没有卡死，说明此方案有效。

第三步：关闭 OpenClash 内部更新
这一步非常关键，必须停用插件自带的更新功能，防止冲突：

登录 OpenWrt 管理界面。

进入 服务 -> OpenClash -> 全局设置 -> 自动更新设置。

取消勾选 “自动更新 GeoSite 数据库”。

点击页面底部的 “应用配置”。

第四步：设置定时自动执行 (Crontab)
既然手动测试没问题，我们让系统在每周三凌晨 4 点自动执行此任务（避开网络高峰）：

在 OpenWrt 管理界面进入 系统 -> 计划任务。

在文本框的最底部添加下面这一行：

Bash

0 4 * * 3 /bin/sh /root/update_geosite.sh > /dev/null 2>&1
点击 “提交”。

专家提醒与避坑指南
为什么要存放在 /tmp？ /tmp 是内存盘。先下载到内存可以避免因 Flash 闪存读写慢导致的系统 “假死”。只有下载完成且校验通过后，才会执行 mv（移动）操作。

关于重启 OpenClash： 大多数情况下，直接替换 GeoSite.dat 后，OpenClash 会在下次规则匹配时自动读取新库。如果你的版本比较旧，可以在脚本的最后一行 fi 之前添加 /etc/init.d/openclash restart（但这会导致网络瞬间断开一下）。

存储空间排查： 请务必确认你的 /etc/openclash/ 所在分区有足够的剩余空间。在 SSH 输入 df -h 查看。如果可用空间（Available）小于 10MB，更新大体积的 GeoSite 依然会有风险。

这个方案现在已经帮你规避了 UI 界面操作带来的内存峰值。你是否需要我顺便提供一份类似的脚本，用来同时更新 GeoIP 数据库？


