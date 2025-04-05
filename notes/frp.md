# FRP Configuration
# FRP 设置 SSH 内网穿透教程（v0.58.0+）
## 一、版本与安装
### 1. 最新版本
• **当前稳定版**: v0.58.0 ([GitHub Release](https://github.com/fatedier/frp/releases))
• **更新检查**:
  ```bash
  curl -s https://api.github.com/repos/fatedier/frp/releases/latest | grep tag_name
  ```
### 2. 一键安装脚本
```bash
# 自动下载最新版（服务端/客户端通用）
wget -O frp.tar.gz $(curl -s https://api.github.com/repos/fatedier/frp/releases/latest | grep browser_download_url | grep linux_amd64 | cut -d '"' -f 4)
tar -zxvf frp.tar.gz && cd frp_*/
```
---

## 二、服务端配置（frps）
### 1. 基础配置 `frps.toml`
```toml
bindPort = 7000

# 安全增强
auth.method = "token"
auth.token = "your_strong_password"
transport.tls.force = true

# 仪表盘（建议修改默认密码）
webServer.addr = "0.0.0.0"
webServer.port = 7500
webServer.user = "admin"
webServer.password = "new_dashboard_password"

# 资源限制（防滥用）
maxPortsPerClient = 5
allowPorts = ["6000-6010"]
```

### 2. 启动与管理
```bash
# 启动（后台运行）
nohup ./frps -c frps.toml &> frps.log &

# 系统服务配置（推荐）
sudo tee /etc/systemd/system/frps.service <<EOF
[Unit]
Description=FRP Server
After=network.target

[Service]
ExecStart=$(pwd)/frps -c $(pwd)/frps.toml
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl enable --now frps
```

---

## 三、客户端配置（frpc）
### 1. 最小化SSH穿透配置 `frpc.toml`
```toml
serverAddr = "x.x.x.x"  # 替换为服务器IP
serverPort = 7000
auth.token = "your_strong_password"

[[proxies]]
name = "ssh-tunnel"
type = "tcp"
localIP = "127.0.0.1"
localPort = 22
remotePort = 6000  # 外网访问端口

# 性能优化
transport.tcpMux = true
healthCheck.type = "tcp"
```

### 2. 客户端管理
```bash
# 调试模式启动（前台运行）
./frpc -c frpc.toml

# 生产环境使用systemd（参考服务端配置）
```

---

## 四、高级特性
### 1. QUIC协议（实验性）
```toml
# 服务端启用
transport.protocol = "quic"

# 客户端需同步配置
transport.protocol = "quic"
```

### 2. 多路复用优化
```toml
transport.multiplexer = "h2mux"
transport.poolCount = 3  # 根据客户端数量调整
```

---

## 五、安全实践
1. **防火墙规则**:
   ```bash
   sudo ufw allow 7000,7500/tcp
   sudo ufw limit 6000/tcp  # 限制SSH穿透端口的连接频率
   ```

2. **TLS证书强化**:
   ```toml
   transport.tls.trustedCaFile = "/path/to/ca.crt"
   ```

3. **日志审计**:
   ```bash
   journalctl -u frps --since "1 hour ago" -f
   ```

---

## 六、连接测试
```bash
# 测试SSH穿透（替换为你的远程端口）
ssh -p 6000 username@server_ip -v

# 端口连通性检查
nc -zv server_ip 6000
```

---

## 七、维护命令
| 功能 | 命令 |
|-------|------|
| 查看运行状态 | `systemctl status frps` |
| 实时日志 | `tail -f frps.log` |
| 版本升级 | `systemctl stop frps && cp frps /usr/local/bin/ && systemctl start frps` |
| 连接数统计 | `ss -tnp | grep frps` |

---

> **提示**：建议定期检查 [官方文档](https://gofrp.org/docs/) 获取安全更新和配置变更。遇到问题时，可通过 `-log.level=debug` 参数获取详细日志。

---
