3️⃣ （可选）在 RouterOS 上排除这个 IP

为了防止 RouterOS DHCP 再次干扰，
在 RouterOS > IP > DHCP Server > Leases 中添加一个静态保留：

MAC 地址：00:0c:29:17:b2:a0
IP 地址：192.168.1.10


并设置为 static，这样 RouterOS 永远不会重新分配这个 IP。