

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

`pcrontab` 是用户接口，用于管理个人的 `pcrontab` 文件。以下是 `pcrontab` 脚本的实现。

**创建 `pcrontab` 脚本**：

使用 Vim 创建并编辑 `pcrontab` 脚本。

```bash
sudo vim /usr/local/bin/pcrontab
```

**插入以下内容**：

```bash
#!/bin/bash

# pcrontab - 管理 pcron 任务调度表

PCRON_DIR="/etc/pcron"
PCRONTAB_FILE="$PCRON_DIR/pcrontab$USER"
ALLOW_FILE="/etc/pcron.allow"
DENY_FILE="/etc/pcron.deny"

# 检查权限
check_permission() {
    if [ -f "$ALLOW_FILE" ]; then
        if ! grep -qw "$USER" "$ALLOW_FILE"; then
            echo "用户 $USER 没有权限使用 pcrontab。"
            exit 1
        fi
    elif [ -f "$DENY_FILE" ]; then
        if grep -qw "$USER" "$DENY_FILE"; then
            echo "用户 $USER 被拒绝使用 pcrontab。"
            exit 1
        fi
    else
        echo "未配置权限文件，默认拒绝所有用户。"
        exit 1
    fi
}

# 列出任务
list_crontab() {
    if [ -f "$PCRONTAB_FILE" ]; then
        cat "$PCRONTAB_FILE"
    else
        echo "没有找到 pcrontab 文件。"
    fi
}

# 编辑任务
edit_crontab() {
    TEMP_FILE="/tmp/pcrontab_temp"
    cp "$PCRONTAB_FILE" "$TEMP_FILE" 2>/dev/null
    if [ $? -ne 0 ]; then
        touch "$TEMP_FILE"
    fi
    # 使用 Vim 进行编辑
    vim "$TEMP_FILE"
    # 保存修改
    cp "$TEMP_FILE" "$PCRONTAB_FILE"
    chmod 600 "$PCRONTAB_FILE"
    rm "$TEMP_FILE"
    echo "pcrontab 已更新。"
}

# 删除任务
remove_crontab() {
    if [ -f "$PCRONTAB_FILE" ]; then
        rm "$PCRONTAB_FILE"
        echo "pcrontab 文件已删除。"
    else
        echo "没有找到 pcrontab 文件。"
    fi
}

# 解析命令行参数
if [ "$EUID" -eq 0 ]; then
    TARGET_USER="$2"
else
    TARGET_USER="$USER"
fi

while getopts ":u:ler" opt; do
    case $opt in
        u)
            TARGET_USER="$OPTARG"
            ;;
        l)
            ACTION="list"
            ;;
        e)
            ACTION="edit"
            ;;
        r)
            ACTION="remove"
            ;;
        \?)
            echo "无效的选项: -$OPTARG" >&2
            exit 1
            ;;
    esac
done

# 切换到目标用户
if [ "$TARGET_USER" != "$USER" ]; then
    if [ "$EUID" -ne 0 ]; then
        echo "只有 root 用户可以指定其他用户。"
        exit 1
    fi
    # 使用 sudo 切换用户环境
    USER_HOME=$(eval echo "~$TARGET_USER")
    PCRONTAB_FILE="$PCRON_DIR/pcrontab$TARGET_USER"
    # 设置环境变量
    export USER="$TARGET_USER"
fi

# 检查权限
check_permission

# 执行动作
case $ACTION in
    list)
        list_crontab
        ;;
    edit)
        edit_crontab
        ;;
    remove)
        remove_crontab
        ;;
    *)
        echo "使用方法: pcrontab [-u 用户] {-l | -e | -r}"
        exit 1
        ;;
esac

exit 0
```

**保存并退出 Vim**：按 `Esc`，输入 `:wq`，然后按 `Enter`。

**赋予可执行权限**：

```bash
sudo chmod +x /usr/local/bin/pcrontab
```

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

# 函数：记录日志
log() {
    local level="$1"
    local message="$2"
    echo "$(date '+%Y-%m-%d %H:%M:%S') $level: $message" >> "$LOG_FILE"
}

# 函数：检查权限（仅 root 运行）
check_root() {
    if [ "$EUID" -ne 0 ]; then
        echo "pcron 必须以 root 身份运行。"
        exit 1
    fi
}

# 函数：解析时间字段
# 返回 0 表示匹配当前时间，1 表示不匹配
check_time() {
    local now_sec="$1"
    local now_min="$2"
    local now_hour="$3"
    local now_day="$4"
    local now_month="$5"
    local now_week="$6"

    local sec_field="$7"
    local min_field="$8"
    local hour_field="$9"
    local day_field="${10}"
    local month_field="${11}"
    local week_field="${12}"

    # 函数：匹配单个字段
    match_field() {
        local value="$1"
        local field="$2"

        if [ "$field" = "*" ]; then
            return 0
        elif [[ "$field" =~ ^\*\/([0-9]+)$ ]]; then
            step="${BASH_REMATCH[1]}"
            if (( value % step == 0 )); then
                return 0
            else
                return 1
            fi
        elif [[ "$field" =~ ^([0-9]+)-([0-9]+)$ ]]; then
            start="${BASH_REMATCH[1]}"
            end="${BASH_REMATCH[2]}"
            if (( value >= start && value <= end )); then
                return 0
            else
                return 1
            fi
        elif [[ "$field" =~ ^([0-9]+):([0-9]+)(:[0-9]+)*$ ]]; then
            IFS=':' read -ra ADDR <<< "$field"
            for val in "${ADDR[@]}"; do
                if (( value == val )); then
                    return 0
                fi
            done
            return 1
        elif [[ "$field" =~ ^([0-9]+)-([0-9]+)(~[0-9]+)*$ ]]; then
            include_range="${BASH_REMATCH[1]}-${BASH_REMATCH[2]}"
            exclude_values=$(echo "$field" | grep -o "~[0-9]\+")
            IFS='-' read -ra RANGE <<< "$include_range"
            if (( value >= RANGE[0] && value <= RANGE[1] )); then
                if [ -n "$exclude_values" ]; then
                    for ex in $(echo "$exclude_values" | tr '~' ' '); do
                        if (( value == ex )); then
                            return 1
                        fi
                    done
                fi
                return 0
            else
                return 1
            fi
        else
            if (( value == field )); then
                return 0
            else
                return 1
            fi
        fi
    }

    # 检查各个时间字段
    match_field "$now_sec" "$sec_field" || return 1
    match_field "$now_min" "$min_field" || return 1
    match_field "$now_hour" "$hour_field" || return 1
    match_field "$now_day" "$day_field" || return 1
    match_field "$now_month" "$month_field" || return 1
    match_field "$now_week" "$week_field" || return 1

    return 0
}

# 函数：执行命令
execute_command() {
    local command="$1"
    local user="$2"

    # 执行命令并捕获输出
    output=$(bash -c "$command" 2>&1)
    status=$?

    if [ $status -eq 0 ]; then
        log "INFO" "用户 $user 执行命令: $command 成功。输出: $output"
    else
        log "ERROR" "用户 $user 执行命令: $command 失败。输出: $output"
    fi
}

# 函数：加载所有任务
load_tasks() {
    local tasks=()

    # 读取全局 pcrontab 文件
    if [ -f "$PCRONTAB_FILE" ]; then
        while IFS= read -r line; do
            # 忽略空行和注释
            [[ -z "$line" || "$line" =~ ^# ]] && continue
            tasks+=("$line")
        done < "$PCRONTAB_FILE"
    fi

    # 读取 /etc/pcron/ 目录下的所有用户 pcrontab 文件
    for user_crontab in "$PCRON_DIR"/pcrontab*; do
        [ -f "$user_crontab" ] || continue
        while IFS= read -r line; do
            [[ -z "$line" || "$line" =~ ^# ]] && continue
            tasks+=("$line")
        done < "$user_crontab"
    done

    echo "${tasks[@]}"
}

# 主循环
main_loop() {
    while true; do
        # 获取当前时间
        now_sec=$(date +%S)
        now_min=$(date +%M)
        now_hour=$(date +%H)
        now_day=$(date +%d)
        now_month=$(date +%m)
        now_week=$(date +%w) # 0-6, 0 表示周日

        # 加载所有任务
        tasks=$(load_tasks)

        # 遍历所有任务
        for task in $tasks; do
            # 解析任务字段
            IFS=' ' read -r sec min hour day month week command <<< "$task"

            # 检查时间匹配
            if check_time "$now_sec" "$now_min" "$now_hour" "$now_day" "$now_month" "$now_week" "$sec" "$min" "$hour" "$day" "$month" "$week"; then
                # 获取任务所属用户（假设 pcrontab 文件名为 pcrontab<username>)
                if [[ "$task" =~ ^/etc/pcron/pcrontab(.+)$ ]]; then
                    task_user="${BASH_REMATCH[1]}"
                else
                    task_user="$USER"
                fi
                # 执行命令
                execute_command "$command" "$task_user" &
            fi
        done

        # 等待 1 秒
        sleep 1
    done
}

# 启动守护进程
check_root
log "INFO" "pcron 守护进程启动。"
main_loop
```

**保存并退出 Vim**：按 `Esc`，输入 `:wq`，然后按 `Enter`。

**赋予可执行权限**：

```bash
sudo chmod +x /usr/local/bin/pcron
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

**保存并退出 Vim**：按 `Esc`，输入 `:wq`，然后按 `Enter`。

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

# 5. 扩展功能建议
虽然项目要求使用 Bash 脚本实现基本功能，以下是一些可选的扩展功能，以提升 pcron 的实用性和用户体验：

邮件通知：

在任务执行失败或成功时发送邮件通知用户。
使用 mail 命令发送邮件。
图形用户界面 (GUI)：

使用 Kdialog 提供简单的图形界面，方便用户管理任务。
任务分类：

支持任务分组或分类，便于管理大量任务。
权限细化：

支持更细粒度的权限控制，例如按任务类型授权。
任务依赖管理：

允许任务之间设置依赖关系，确保任务按顺序执行。
安全性增强：

对任务命令进行沙箱化，防止恶意命令执行。
Web API：

提供 RESTful API，允许远程管理 pcron 任务。




---

