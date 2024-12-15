# Operating-systems-project

### **LO03 项目 2024-2025**  
### **任务调度器编程**  

---

### **1 功能与语法**

- **pcron 命令**：用于让系统执行提前定义和计划的所有任务（命令和脚本）。这些任务可以是简单的命令，也可以是复杂的脚本，能够在固定时间执行，甚至周期性执行，并生成日志报告。

- **文件监控**：pcron 命令会定期读取位于 **/etc/pcron/** 目录和 **/etc/pcrontab** 文件中的内容，以判断是否需要执行某些任务。

- **日志记录**：每次执行 pcron 操作时，都会向 **/var/log/pcron** 文件添加一条日志记录。

- **标准输出**：默认情况下，pcron 执行的命令产生的输出会被发送到标准输出。

- **用户权限**：pcron 命令通常仅限管理员使用。不过，特定用户也可以被授权使用。授权用户列表保存在 **/etc/pcron.allow** 文件中，未授权用户列表则保存在 **/etc/pcron.deny** 文件中。  
    示例：仅用户 Jean 和 Charles 被允许使用 pcron 服务：
    ```bash
    $ cat pcron.allow  
    Jean  
    Charles  
    ```

- **pcrontab 命令**：用于为 pcron 服务编程，语法如下：  
    ```bash
    pcrontab [-u user] {-l | -r | -e}
    ```
    - `pcrontab -u toto -l`：显示用户 toto 的 pcrontab 文件，文件位于 **/etc/pcron/**。  
    - `pcrontab -u toto -r`：删除用户 toto 的 pcrontab 文件。  
    - `pcrontab -u toto -e`：创建或编辑用户 toto 的 pcrontab 文件。修改时会在 **/tmp** 目录中打开临时文件，保存后写入 **/etc/pcron/pcrontabtoto**。

- **pcrontab 文件**：每行包含 **7 个字段**，前 6 个字段表示任务的执行时间，第 7 个字段表示要执行的命令（可以是脚本）。  
    - **字段说明**：  
      1. 15 秒（0-3）  
      2. 分钟（0-59）  
      3. 小时（0-23）  
      4. 月内日期（1-31）  
      5. 月份（1-12）  
      6. 星期（0-6，0 表示周日）  

    - **字段格式**：  
        - **单一值**：例如 `15` 表示分钟字段中的第 15 分钟。  
        - **值列表**：例如 `1:3:5` 表示多个有效值（1、3 和 5）。  
        - **时间范围**：例如 `1-5` 表示从周一到周五。  
        - **星号（*）**：表示字段的所有可能值。  
        - **步长值**：例如 `*/4` 表示每隔 4 小时执行一次。  
        - **排除值**：使用 `~` 排除指定值，例如 `5-8~6~7` 等价于 `5:8`。  

- **示例任务**：  
    - 每 4 小时执行一次，日期是每月的 1 日和 15 日：  
      ```bash
      0 0 */4 1,15 * * 命令
      ```
    - 每月 1 日和 15 日凌晨 2:30:30 重启机器：  
      ```bash
      2 30 2 1,15 * * /sbin/shutdown -r
      ```
    - 每周一凌晨 3:15 执行备份脚本：  
      ```bash
      0 15 3 * * 1 /usr/bin/backup
      ```
    - 每隔 15 秒执行一次：  
      ```bash
      1 * * * * * 命令
      ```
    - 每周一到周五早上 7:30 执行：  
      ```bash
      0 30 7 * * 1-5 命令
      ```
    - 每小时的 15 分、30 分、45 分在指定时间段执行：  
      ```bash
      0 0,15,30,45 15-19 1-15 1-12~4~5 1-5 命令
      ```
    - 每月 1 日凌晨 2:00 清理 `/tmp` 目录中过期 3 天的文件：  
      ```bash
      0 0 2 1 * * find /tmp -atime 3 -exec rm -f {}\;
      ```

---

### **2 说明**  

与现有任务调度工具（如 crontab、anacron、fcron 等）或其衍生品的相似之处纯属巧合。因此，**不建议**在项目中使用这些工具完成任务。

---

### **3 扩展**  

所有与项目相关的扩展都欢迎提出。用户界面设计可由你自行决定。

---

### **4 报告与展示**  

- 项目必须由三人小组共同完成。  
- **截止日期**：2025 年 2 月 24 日（周一）前需提交一份最终报告。  
- **展示与测试**：将在 2025 年 2 月 24 日至 27 日之间的最后一周进行，持续 15 分钟。

# 具体实现
## **1. 系统架构设计**

### **1.1 主要组件**

1. **pcron 守护进程**：后台运行的 Bash 脚本，负责定期检查 `/etc/pcron/` 目录和 `/etc/pcrontab` 文件，解析任务并在指定时间执行。
2. **pcrontab 命令**：用于管理用户的 `pcrontab` 文件，包括添加、编辑、删除和查看任务。
3. **用户权限管理**：通过 `/etc/pcron.allow` 和 `/etc/pcron.deny` 文件控制哪些用户可以使用 `pcron` 服务。
4. **日志记录**：将 `pcron` 的运行日志记录到 `/var/log/pcron` 文件中。

---

## **2. 详细实现方案**

### **2.1 目录结构和文件准备**

在开始编写脚本之前，我们需要创建必要的目录和文件。

```bash
# 创建 pcron 相关目录
sudo mkdir -p /etc/pcron
sudo touch /etc/pcrontab

# 创建日志文件
sudo mkdir -p /var/log
sudo touch /var/log/pcron
sudo chmod 600 /var/log/pcron

# 创建权限文件
sudo touch /etc/pcron.allow
sudo touch /etc/pcron.deny
```

**注意**：确保 `/etc/pcron.allow` 和 `/etc/pcron.deny` 不同时存在，以避免权限冲突。通常优先使用 `pcron.allow`。

### **2.2 用户权限管理**

权限管理通过 `/etc/pcron.allow` 和 `/etc/pcron.deny` 实现。

- **/etc/pcron.allow**：列出被允许使用 `pcron` 的用户。
- **/etc/pcron.deny**：列出被拒绝使用 `pcron` 的用户。

**示例 `/etc/pcron.allow`**：

```bash
Jean
Charles
```

**示例 `/etc/pcron.deny`**：

```bash
# 如果使用 deny 文件，列出被拒绝的用户
guest
```

**权限检查逻辑**：

- 如果存在 `pcron.allow` 文件，只有文件中列出的用户被允许使用 `pcron`，其他用户被拒绝。
- 如果不存在 `pcron.allow`，但存在 `pcron.deny` 文件，则文件中列出的用户被拒绝，其他用户被允许。
- 如果两个文件都不存在，默认拒绝所有用户。

### **2.3 pcrontab 命令实现**

`pcrontab` 是用户接口，用于管理个人的 `pcrontab` 文件

代码部分

**脚本说明**：

- **权限检查**：根据 `/etc/pcron.allow` 和 `/etc/pcron.deny` 文件判断当前用户是否有权限使用 `pcrontab`。
- **列出任务** (`-l`)：显示用户的 `pcrontab` 文件内容。
- **编辑任务** (`-e`)：使用 Vim 编辑用户的 `pcrontab` 文件。
- **删除任务** (`-r`)：删除用户的 `pcrontab` 文件。

### **2.4 pcron 守护进程实现**

`pcron` 守护进程负责定期检查任务并执行。以下是 `pcron` 守护进程的 Bash 脚本实现。

**创建 `pcron` 守护进程脚本**：

使用 Vim 创建并编辑 `pcron` 脚本。

```bash
sudo vim /usr/local/bin/pcron
```

**插入以下内容**：

```bash
#!/bin/bash

# pcron - 任务调度器守护进程

PCRON_DIR="/etc/pcron"
PCRONTAB_FILE="$PCRON_DIR/pcrontab"
LOG_FILE="/var/log/pcron"

```

**脚本说明**：

- **日志记录**：使用 `log` 函数将日志信息写入 `/var/log/pcron`。
- **权限检查**：确保 `pcron` 以 root 身份运行。
- **时间匹配**：通过 `check_time` 函数解析时间字段，判断是否需要执行任务。
- **任务执行**：使用后台进程执行命令，并记录执行结果。
- **主循环**：每秒检查一次任务，确保秒级调度。

### **2.5 配置 systemd 服务**

为了让 `pcron` 守护进程在系统启动时自动运行，我们需要创建一个 systemd 服务文件。

**创建 systemd 服务文件**：

```bash
sudo vim /etc/systemd/system/pcron.service
```

**插入以下内容**：

```ini
[Unit]
Description=pcron Task Scheduler
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/pcron
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

**重新加载 systemd 配置**：

```bash
sudo systemctl daemon-reload
```

**启动并启用 pcron 服务**：

```bash
sudo systemctl start pcron
sudo systemctl enable pcron
```

**检查服务状态**：

```bash
sudo systemctl status pcron
```

您应该看到 `pcron` 服务正在运行。

### **2.6 pcrontab 文件格式**

每行包含 **7 个字段**，前 6 个字段表示任务的执行时间，第 7 个字段表示要执行的命令。

**字段说明**：

1. 秒（0-59）
2. 分钟（0-59）
3. 小时（0-23）
4. 月内日期（1-31）
5. 月份（1-12）
6. 星期（0-6，0 表示周日）
7. 命令（可以是脚本）

**字段格式**：

- `*`：所有可能值
- 单一值：例如 `15`
- 值列表：例如 `1:3:5`
- 时间范围：例如 `1-5`
- 步长值：例如 `*/4`
- 排除值：例如 `5-8~6~7`

**示例 pcrontab 文件**：

```bash
# 全局 pcrontab 文件 /etc/pcrontab
0 0 */4 1,15 * * /path/to/command
2 30 2 1,15 * * /sbin/shutdown -r
0 15 3 * * 1 /usr/bin/backup
1 * * * * * /path/to/another_command
0 30 7 * * 1-5 /usr/bin/morning_task
0 0,15,30,45 15-19 1-15 1-12~4~5 1-5 /path/to/scheduled_task
0 0 2 1 * * find /tmp -atime 3 -exec rm -f {} \;
```

**用户 pcrontab 文件**：位于 `/etc/pcron/pcrontab<username>`，例如 `/etc/pcron/pcrontabJean`

---

## **3. 系统安装与配置**

### **3.1 安装 pcron 守护进程和 pcrontab 命令**

**步骤**：

1. **创建必要的目录和文件**（已在前述部分完成）。
2. **复制并赋予脚本可执行权限**（已在前述部分完成）。
3. **配置 systemd 服务**（已在前述部分完成）。

### **3.2 配置用户权限**

1. **编辑 `/etc/pcron.allow`**（如果需要）：

    ```bash
    sudo vim /etc/pcron.allow
    ```

    添加允许使用 `pcron` 的用户，例如：

    ```
    Jean
    Charles
    ```

2. **编辑 `/etc/pcron.deny`**（如果需要）：

    ```bash
    sudo vim /etc/pcron.deny
    ```

    添加被拒绝使用 `pcron` 的用户，例如：

    ```
    guest
    ```

**注意**：确保 `pcron.allow` 和 `pcron.deny` 不同时存在，以避免权限冲突。

---

## **4. 测试与验证**

### **4.1 测试 pcrontab 命令**

1. **列出任务**：

    ```bash
    pcrontab -l
    ```

    如果没有任务，您会看到提示“没有找到 pcrontab 文件。”

2. **编辑任务**：

    ```bash
    pcrontab -e
    ```

    在 Vim 中添加以下任务作为测试：

    ```
    0 0 * * * * echo "Hello, World!" >> /tmp/pcron_test.log
    ```

    **保存并退出 Vim**：按 `Esc`，输入 `:wq`，然后按 `Enter`。

3. **列出任务**：

    再次运行 `pcrontab -l`，确认任务已添加。

### **4.2 验证 pcron 守护进程执行任务**

1. **查看日志文件**：

    ```bash
    sudo tail -f /var/log/pcron
    ```

    您应该看到类似以下的日志输出：

    ```
    2024-04-27 14:30:00 INFO:用户 Jean 执行命令: echo "Hello, World!" >> /tmp/pcron_test.log 成功。输出:
    ```

2. **检查任务执行结果**：

    ```bash
    cat /tmp/pcron_test.log
    ```

    应该看到 "Hello, World!" 被写入。

3. **测试错误处理**：

    编辑 `pcrontab` 添加一个无效命令：

    ```bash
    pcrontab -e
    ```

    添加以下行：

    ```
    0 0 * * * * /invalid/command
    ```

    保存并退出。

    查看日志文件，确认错误信息被记录：

    ```
    2024-04-27 14:31:00 ERROR:用户 Jean 执行命令: /invalid/command 失败。输出: /bin/bash: /invalid/command: 没有那个文件或目录
    ```

### **4.3 用户权限测试**

1. **以被允许的用户运行 pcrontab 命令**：

    切换到被允许的用户，例如 Jean：

    ```bash
    sudo -u Jean pcrontab -l
    ```

    确认可以列出任务。

2. **以被拒绝的用户运行 pcrontab 命令**：

    切换到被拒绝的用户，例如 guest：

    ```bash
    sudo -u guest pcrontab -l
    ```

    应该看到权限被拒绝的提示。

---

## **5. 待实现扩展功能**

1. **邮件通知**：
    - 在任务执行失败或成功时发送邮件通知用户。
    - 使用 `mail` 命令发送邮件。

2. **图形用户界面 (GUI)**：
    - 使用 Kdialog 提供简单的图形界面，方便用户管理任务。

3. **任务分类**：
    - 支持任务分组或分类，便于管理大量任务。

4. **权限细化**：
    - 支持更细粒度的权限控制，例如按任务类型授权。

5. **任务依赖管理**：
    - 允许任务之间设置依赖关系，确保任务按顺序执行。

6. **安全性增强**：
    - 对任务命令进行沙箱化，防止恶意命令执行。

7. **Web API**：
    - 提供 RESTful API，允许远程管理 `pcron` 任务。

