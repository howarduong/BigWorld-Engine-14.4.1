# BigWorld 服务器安装记录 / BigWorld Server Installation Log

> **项目 / Project:** BigWorld 服务器测试性安装  
> **日期 / Date:** 2026-03-31  
> **状态 / Status:** 部署完成，准备开发 / Deployment Complete, Ready for Development  

---

## 一、环境信息 / Environment Information

| 项目 / Item | 信息 / Information |
|-------------|-------------------|
| 宿主机 / Host OS | Windows 11 Pro |
| 虚拟化 / Hypervisor | Microsoft Hyper-V (Gen-1) |
| 客户机 / Guest OS | CentOS 7 Minimal |
| 系统镜像 / ISO | `CentOS-7-x86_64-Minimal-1609-99.iso` |
| 安装类型 / Type | 最小化安装 / Minimal |
| **账户体系 / Accounts** | **`root`**: 管理员最高权限 / Admin Highest Privileges (环境全局配置)<br>**`bwtools`**: 普通权限 / Standard Privileges (服务器工具必须账户)<br>**`bwserver`**: 普通权限 / Standard Privileges (服务器内容运行账户) |

---

## 二、安装步骤 / Installation Steps

### Step 0: 系统安装 / System Installation
- **操作**: 使用 `CentOS-7-x86_64-Minimal-1609-99.iso` 在 Hyper-V Gen-1 中创建虚拟机并完成最小化安装，安装完成后重启。
- **结果**: 虚拟机启动正常，系统引导成功。

### Step 1: 配置阿里云 Base 源 / Configure Aliyun Base Repo
> **背景**: CentOS 7 已于 2024-06-30 EOL，官方源已下线。
- **命令**:
  ```bash
  cat /etc/centos-release
  mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
  curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
  yum makecache
  ```
- **说明**: 替换官方 Base 源为阿里云镜像，恢复包管理能力。

### Step 1.1: 全量系统更新 / Full System Update
- **命令**: `yum update -y`
- **说明**: 将所有软件包升级至阿里云源提供的最新版本，更新后需重启使内核生效。

### Step 1.2: 重启服务器 / Reboot Server
- **命令**: `reboot`
- **说明**: 全量更新完成后重启，确保内核及所有升级包生效。

### Step 1.3: 安装 EPEL 扩展源 / Install EPEL Repo
- **命令**: `yum install -y epel-release`
- **说明**: 安装 EPEL 仓库，提供 BigWorld 所需的众多依赖（Python、网络工具等），减少手动编译。

### Step 1.4: 安装 MariaDB 数据库 / Install MariaDB Server
- **命令**: `yum install -y mariadb-server`
- **说明**: 安装 MariaDB 作为 BigWorld 数据库后端，存储账号、角色、道具等核心数据。

### Step 1.5: 重启服务器 / Reboot Server
- **命令**: `reboot`
- **说明**: MariaDB 安装完成后重启，确保服务在系统重启后能正常自启。

### Step 1.6: 启动并配置 MariaDB / Start & Configure MariaDB
- **命令**:
  ```bash
  systemctl enable mariadb
  systemctl start mariadb
  # 编辑 /etc/my.cnf，在 [mysqld] 段添加:
  default-storage-engine=InnoDB
  ```
- **说明**: BigWorld 要求表必须使用 InnoDB 引擎。设置开机自启并修改默认引擎，配置后需重启 MariaDB 生效。

### Step 1.6.1: 创建 bwtools 数据库账户 / Create bwtools DB User
- **命令**:
  ```sql
  mysql -u root
  GRANT ALL PRIVILEGES ON `bw\_stat\_log\_%`.* TO 'bwtools'@'localhost' IDENTIFIED BY 'bwtools';
  GRANT ALL PRIVILEGES ON bw_web_console.* TO 'bwtools'@'localhost' IDENTIFIED BY 'bwtools';
  CREATE DATABASE bw_web_console;
  ```
- **说明**: 为工具组件创建专用账户 `bwtools`，授权统计日志通配符表及 Web 控制台数据库。

### Step 1.6.2: 创建 bwserver 数据库账户 / Create bwserver DB Account
- **命令**:
  ```sql
  mysql -u root
  CREATE DATABASE game_db_pgw;
  GRANT SELECT, INSERT, UPDATE, DELETE, ALTER, CREATE, DROP, INDEX ON game_db_pgw.* TO 'bwserver'@'localhost' IDENTIFIED BY 'bwserver';
  GRANT SELECT, INSERT, UPDATE, DELETE, ALTER, CREATE, DROP, INDEX ON game_db_pgw.* TO 'bwserver'@'%' IDENTIFIED BY 'bwserver';
  GRANT RELOAD ON *.* TO 'bwserver'@'localhost' IDENTIFIED BY 'bwserver';
  GRANT RELOAD ON *.* TO 'bwserver'@'%' IDENTIFIED BY 'bwserver';
  ```
- **说明**: 为服务器进程创建 `bwserver` 账户，授权本地/远程访问及 RELOAD 权限，创建主游戏库 `game_db_pgw`。

### Step 1.7: 关闭防火墙 / Disable Firewall
- **命令**:
  ```bash
  sudo systemctl stop firewalld
  sudo systemctl disable firewalld
  sudo systemctl mask --now firewalld
  ```
- **说明**: 关闭并屏蔽 FirewallD，确保 BigWorld 组件间网络通信不受限，`mask` 防止服务被其他进程唤醒。

### Step 2: 安装 BigWorld RPM 包 / Install BigWorld RPMs
- **命令** (root 账户):
  ```bash
  yum install --nogpgcheck bigworld-bwmachined-14.4.1.el7.x86_64.rpm
  yum install --nogpgcheck bigworld-server-14.4.1.el7.x86_64.rpm
  yum install --nogpgcheck bigworld-tools-14.4.1.el7.x86_64.rpm
  ```
- **说明**: 安装 BigWorld 14.4.1 核心组件：`bwmachined` (节点通信)、`server` (BaseApp/CellApp)、`tools` (管理控制台)。

### Step 5: 配置 StatLogger 并重启系统 / Configure StatLogger & Reboot
- **操作**: 安装 RPM 包后，编辑 `/etc/bigworld/stat_logger.xml`，配置 `<database>` 段：
  ```xml
  <database>
    <enable>true</enable>
    <host>localhost</host>
    <port>3306</port>
    <user>bwtools</user>
    <password>bwtools_passwd</password>
    <prefix>bw_stat_log_data</prefix>
  </database>
  ```
- **关键说明**:
  - **共用配置**: 此处的数据库账户信息为**全组件共用**，`bwtools`、`bw_web_console` 等均通过此文件连接 MariaDB，密码必须与 Step 1.6.1 一致。
  - **prefix 作用**: 用于定义 StatLogger 数据存储的数据库表名前缀。
  - **必须重启 Linux 系统**: 配置完成后执行 `reboot` 重启整机。此步骤可避免线程冲突和服务启动失败问题，确保所有 BigWorld 组件正确加载数据库配置并初始化运行环境。
- **重启命令**: `reboot`

### Step 6: 服务启动验证与排查 / Service Startup & Troubleshooting
- **操作**: 系统重启后，检查 `bw_stat_logger`、`bw_message_logger`、`bw_web_console` 状态。
- **现象**:
  - `bw_stat_logger`: systemd 报 `failed (Result: protocol)`，PID 归属异常。
  - `bw_message_logger`: systemd 报 `failed (Result: exit-code)`。
  - `bw_web_console`: 依赖失败 `Dependency failed`。
- **排查结论**:
  - **bw_stat_logger**: 生产环境通常不需要开启，**决定跳过**。
  - **bw_message_logger & bw_web_console**: BigWorld 14.x 组件属老式 System V daemon，systemd PID 追踪机制无法正确识别导致误报。通过 `ps aux | grep` 确认进程实际存在且运行正常。WebConsole 可正常访问，无需修复。
- **状态**: ✅ 核心服务运行正常，辅助组件按预期工作。

### Step 7: 创建 BigWorld 专用运行账户 / Create BigWorld Runtime User
- **命令**:
  ```bash
  adduser bwserver
  passwd bwserver
  ```
- **说明**: 为 BigWorld 服务器进程创建专用运行账户 `bwserver`，避免使用 root 权限运行服务，提高系统安全性。该账户将用于后续启动和管理 BigWorld 核心组件。

### Step 8: 创建 BigWorld 新项目 / Create BigWorld Project
- **操作**: 切换至 `bwserver` 用户，执行项目初始化脚本。
- **命令**:
  ```bash
  su - bwserver
  bw_configure bigworld_first_run
  ```
- **说明**: `bw_configure` 是安装时附带的配置辅助脚本。以 `bwserver` 用户运行该脚本并指定项目名称（如 `bigworld_first_run`），会自动完成以下操作：
  - 在用户目录下创建新项目目录结构（包含 BigWorld Tutorial 资源）
  - 生成用于启动服务器的配置文件
  - 配置用户环境以支持 BigWorld 服务器运行
- **配置文件 `.bwmachined.conf` 说明**:
  - 该文件位于用户主目录（`~/.bwmachined.conf`），为单行配置文件，供 `bwmachined` 进程读取以确定如何启动服务器进程。
  - **格式**: `<server_binary_directory>;<game_res_directory>[:<secondary_res_directory>]`
  - **路径说明**:
    - `server_binary_directory`: 自动填充 BigWorld 服务器二进制文件安装路径。
    - `game_res_directory`: 通常为 `/home/<username>/<project_name>/res`，是游戏实现、添加和修改的主要目录。
    - `secondary_res_directory`: 自动填充 BigWorld 资源目录，包含默认配置文件和服务器进程使用的 Python 库，所有游戏通常都需要此目录。

### Step 9: 生成并配置 RSA 密钥对 / Generate & Configure RSA Keypairs
- **操作**: 使用 OpenSSL 生成自定义 RSA 密钥对，替换默认密钥，并配置到项目资源目录。
- **命令**:
  ```bash
  # 切换到 bwserver 用户的项目资源目录
  cd /home/bwserver/bigworld_first_run/res

  # 1. 生成 2048 位 RSA 私钥
  openssl genrsa 2048 > loginapp.privkey

  # 2. 提取对应的公钥
  openssl rsa -pubout < loginapp.privkey > loginapp.pubkey

  # 3. 创建 server 目录并移动私钥
  mkdir -p server
  mv loginapp.privkey server/

  # 4. 放置 bigworld.key (需从源码或指定位置获取)
  # cp /path/to/bigworld.key server/
  ```
- **说明**:
  - **LoginApp Keypair**: 用于加密客户端与服务器的初始通信（用户名/密码），防止被拦截。强烈建议从项目开始就使用自定义密钥，替换安装包附带的默认密钥。
    - **私钥 (`loginapp.privkey`)**: 存放在 `/home/bwserver/bigworld_first_run/res/server`。
    - **公钥 (`loginapp.pubkey`)**: 存放在 `/home/bwserver/bigworld_first_run/res` (客户端资源也需此文件)。
  - **BigWorld Server Key**: 文件 `bigworld.key` 需放置在 `res/server` 目录。
  - **安全警告**: 私钥 (`loginapp.privkey`) 绝不能随游戏客户端资源一起分发。

### Step 10: 配置 Web Console 用户与权限 / Configure Web Console Users & Permissions
- **操作**:
  1.  **访问地址**: 浏览器打开 `http://<服务器 IP>:8080`。
  2.  **管理员登录**: 使用默认管理员账户 `admin` 登录。
  3.  **新增成员**: 进入成员管理页面，添加新用户 `bwserver`。
  4.  **设置密码**: 为该用户设置登录密码。
  5.  **权限分配**: 将 `bwserver` 用户的权限设定为 `modify all`（完全控制）。
- **说明**: Web Console 是 BigWorld 的可视化管理工具。通过添加 `bwserver` 用户（对应 Linux 系统账户名）并赋予 `modify all` 权限，允许该用户在 Web 端进行服务器配置、启停服务等全权操作。

### Step 10: 配置 Web Console 用户与权限 / Configure Web Console Users & Permissions
- **操作**:
  1.  **访问地址**: 浏览器打开 `http://<服务器 IP>:8080`。
  2.  **管理员登录**: 使用默认管理员账户 `admin` 登录。
  3.  **新增成员**: 进入成员管理页面，添加新用户 `bwserver`。
  4.  **设置密码**: 为该用户设置登录密码。
  5.  **权限分配**: 将 `bwserver` 用户的权限设定为 `modify all`（完全控制）。
- **说明**: Web Console 是 BigWorld 的可视化管理工具。通过添加 `bwserver` 用户（对应 Linux 系统账户名）并赋予 `modify all` 权限，允许该用户在 Web 端进行服务器配置、启停服务等全权操作。

### Step 11: 启动 BigWorld 服务器 / Start BigWorld Server
- **操作**:
  1.  **登录**: 使用 `bwserver` 账户登录 Web Console (`http://<服务器 IP>:8080`)。
  2.  **管理**: 导航至“服务器管理” (Server Management) 页面。
  3.  **启动**: 点击“启动本机服务器” (Start Local Server)。
- **结果**: 服务器启动成功，各项服务（BaseApp, CellApp 等）开始运行。
- **说明**: 此步骤验证了之前的所有配置（数据库连接、RSA 密钥、用户权限、资源路径）均正确无误。服务器正式进入运行状态，可以接受客户端连接。

### Step 12: 部署 Fantasy Demo 资源 / Deploy Fantasy Demo Resources
- **操作**:
  1.  **上传**: 将本地打包好的 `fantasydemo.7z` 上传至服务器 `bwserver` 用户目录（例如 `/home/bwserver/`）。
      *   命令示例 (本地执行): `scp fantasydemo.7z bwserver@<服务器 IP>:/home/bwserver/`
  2.  **解压**: 登录服务器，切换至 `bwserver` 用户，解压资源包。
      *   命令示例 (服务器执行):
        ```bash
        su - bwserver
        # 若未安装 7z 需先安装: yum install -y p7zip
        7z x fantasydemo.7z -o/home/bwserver/bigworld_first_run/res/
        ```
- **说明**: 上传并解压 Fantasy Demo 资源包，用于测试服务器加载外部游戏资源的能力。解压后的资源通常包含模型、贴图、脚本等，将替换或补充到项目的 `res` 目录中。

---

## 三、文档信息 / Document Information

| 项目 / Item | 内容 / Content |
|-------------|---------------|
| 创建日期 / Created | 2026-03-31 |
| 最后更新 / Last Updated | 2026-03-31 |
| 版本 / Version | 2.2 |
| 状态 / Status | 部署完成，准备开发 / Deployment Complete, Ready for Development |
