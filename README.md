please think about if you changed your fucking net and contributed to a fucking Muti-DNS-Cashe errorssssssssssssss!

多层网络DNS缓存冲突排查指南
---
多层网络环境 DNS 解析缓存冲突排查与根治指南
1. 适用场景
适用于所有多层网络嵌套开发环境，切换网络（WiFi/网线）后出现外网域名解析失败问题：
- 物理主机 → 虚拟机（VMware/Hyper-V）→ Docker 容器
- Windows 主机 → WSL2 → Docker 容器
- 路由局域网 → 终端 → 虚拟化/容器服务
触发条件：更换 WiFi、切换网络、重启网卡后，服务突然无法调用外网接口。
2. 问题本质（核心原理）
主机、虚拟机、容器三层均存在独立DNS解析缓存。外层网络切换后，外层DNS已更新，但内层缓存不会自动刷新，产生层级错位：
外层能上网、内层解析失败，属于典型的「多层DNS缓存冲突」，非服务BUG、非网络断连。
3. 故障识别特征（精准判断）
- 换网前一切正常，换网后立刻报错
- 仅外网域名解析报错：Failed to resolve、连接超时
- 内网服务、数据库、本地功能完全正常
- 清空缓存/重启容器即可临时恢复，反复换网会复现
4. 由外到内分层排查法（快速定位问题层）
遵循 物理机 → 虚拟机 → 容器 逐层测试，哪一层解析失败，问题就在哪一层。
1）物理主机测试
浏览器访问目标域名 或 执行：nslookup 目标域名
正常 → 外网通畅，问题在内层
2）虚拟机/WSL 中层测试
虚拟机内执行：nslookup 目标域名
物理机通、虚拟机不通 → 虚拟机DNS缓存过期
3）Docker 容器内层测试
执行：docker exec -it 容器名 curl -I https://目标域名
虚拟机通、容器不通 → 容器未同步最新DNS
5. 分层根治方案
5.1 物理机层
管理员CMD清空系统DNS缓存：
ipconfig /flushdns
5.2 虚拟机层（关键根治）
1. 清空虚拟机缓存：
ipconfig /flushdns
2. 永久解决：手动固定公共DNS，不再跟随虚拟网关动态DNS（换网必炸根源）
- 首选DNS：114.114.114.114
- 备用DNS：223.5.5.5 / 8.8.8.8
5.3 容器层
1. 重启容器同步DNS：
docker compose down && docker compose up -d
2. 永久解决：项目.env 强制写入容器DNS，彻底隔离宿主机网络波动
DNS_SERVERS=114.114.114.114,223.5.5.5
6. 换网后标准预防流程（杜绝复发）
每次切换WiFi/网络后，按顺序执行：
1. 物理机清DNS缓存
2. 确认虚拟机外网可通
3. 重启Docker容器服务
7. 速查对照表
现象
根因
解决方案
主机通、虚拟机不通
虚拟机DNS缓存过期/网关DNS不稳定
清缓存 + 固定公共DNS
虚拟机能通、容器不通
容器未同步新网络DNS
重启容器 + 配置.env固定DNS
全层级超时
当前网络外网故障
检查物理机网络连通性
---
---
Multi-Layer Network DNS Cache Conflict Troubleshooting & Permanent Fix Guide
1. Applicable Scenarios
For all multi-layer nested network development environments where external domain resolution fails after network switching (WiFi/Ethernet):
- Physical Host → VMware/Hyper-V Virtual Machine → Docker Container
- Windows Host → WSL2 → Docker Container
- LAN Router → Terminal → Virtualization/Container Service
Trigger Condition: External API resolution failure after switching WiFi, network adapter reset, or network environment change.
2. Root Cause
Hosts, virtual machines, and containers maintain independent DNS cache. After outer network switching, outer DNS updates normally, but inner cache cannot refresh automatically.
Result: Outer network works, inner layer domain resolution fails. This is a multi-layer DNS cache conflict, not application bug or network disconnection.
3. Typical Fault Features
- All services work normally before network switching
- Only external domain errors occur: Failed to resolve, connection timeout
- Intranet services, databases and local functions are unaffected
- Temporary recovery by clearing cache or restarting containers; error recurs after network switching
4. Layer-by-Layer Troubleshooting
Check in order: Physical Host → Virtual Machine → Container. The faulty layer is where resolution fails.
1) Physical Host Test
Browser access or command: nslookup [target-domain]
Normal result = external network is stable, fault exists in inner layers
2) Virtual Machine / WSL Test
Run in VM: nslookup [target-domain]
Host works but VM fails = VM DNS cache outdated
3) Docker Container Test
Command: docker exec -it [container-name] curl -I https://[target-domain]
VM works but container fails = container DNS not synchronized
5. Layer-by-Layer Permanent Solutions
5.1 Physical Host Layer
Flush system DNS cache via administrator CMD:
ipconfig /flushdns
5.2 Virtual Machine Layer (Core Fix)
1. Flush VM DNS cache:
ipconfig /flushdns
2. Permanent solution: Set static public DNS, abandon unstable gateway dynamic DNS:
- Preferred DNS: 114.114.114.114
- Alternate DNS: 223.5.5.5 / 8.8.8.8
5.3 Container Layer
1. Restart containers to synchronize latest DNS:
docker compose down && docker compose up -d
2. Permanent solution: Force container DNS in .env to avoid network fluctuation impact:
DNS_SERVERS=114.114.114.114,223.5.5.5
6. Standard Prevention Workflow
Execute after every network switch to avoid recurrence:
1. Flush DNS cache on physical host
2. Verify VM external network connectivity
3. Restart Docker container services
7. Quick Reference Table
Phenomenon
Root Cause
Solution
Host OK, VM failed
Outdated VM DNS / unstable gateway DNS
Flush cache + static public DNS
VM OK, Container failed
Container DNS not synchronized
Restart container + fixed DNS in .env
All layers timeout
Physical network failure
Check host network connectivity
