# WuKongIM 历史数据清理方案

## 📚 目录

- [1. 问题背景](#1-问题背景)
- [2. 清理方案概述](#2-清理方案概述)
- [3. 方案一：通过API清理（推荐）](#3-方案一通过api清理推荐)
- [4. 方案二：数据库级别清理](#4-方案二数据库级别清理)
- [5. 方案三：归档历史数据](#5-方案三归档历史数据)
- [6. 方案四：定期备份+重建](#6-方案四定期备份重建)
- [7. 自动化清理脚本](#7-自动化清理脚本)
- [8. 注意事项](#8-注意事项)

---

## 1. 问题背景

### 1.1 为什么需要清理历史数据？

随着 WuKongIM 运行时间增长，`./wukongimdata` 目录会不断增大：

- 📦 **磁盘空间占用**: 消息数据持续累积
- 🐌 **查询性能下降**: 数据量过大影响查询速度
- 💰 **存储成本增加**: 云服务器磁盘费用上升
- 🔧 **备份时间变长**: 数据量大导致备份耗时

### 1.2 清理目标

清理 **180 天之前**的历史数据，包括：

- ✅ 180天前的消息记录
- ✅ 过期的会话记录
- ✅ 无效的用户设备信息
- ⚠️ 保留用户、频道等基础信息

---

## 2. 清理方案概述

### 2.1 方案对比

| 方案 | 优点 | 缺点 | 推荐度 | 停机时间 |
|------|------|------|--------|---------|
| API清理 | 安全、可控、支持增量 | 需要开发脚本 | ⭐⭐⭐⭐⭐ | 无需停机 |
| 数据库清理 | 直接、快速 | 风险高、需停机 | ⭐⭐ | 需要停机 |
| 归档方案 | 数据可恢复 | 复杂、占用空间 | ⭐⭐⭐⭐ | 无需停机 |
| 备份+重建 | 彻底清理 | 停机时间长 | ⭐⭐⭐ | 长时间停机 |

### 2.2 推荐方案

**方案一（API清理）+ 方案三（归档）** 组合使用：

1. 先归档历史数据（可选）
2. 通过API删除过期消息
3. 定期执行，保持数据量稳定

---

## 3. 方案一：通过API清理（推荐）

### 3.1 清理原理

通过 WuKongIM 提供的 HTTP API 删除指定时间之前的消息。

### 3.2 API 接口

WuKongIM 没有直接的"批量删除历史消息"接口，但可以通过以下方式实现：

#### 方式1: 删除整个频道的消息

```bash
# 删除频道（会清空该频道的所有消息）
curl -X POST http://localhost:5001/channel/delete \
  -H "Content-Type: application/json" \
  -d '{
    "channel_id": "channel123",
    "channel_type": 2
  }'
```

#### 方式2: 通过 TruncateLogTo 截断消息

```go
// 这是内部方法，需要通过自定义脚本调用
// 保留最近N条消息，删除之前的
db.TruncateLogTo(channelId, channelType, messageSeq)
```

### 3.3 清理脚本（Python）

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
WuKongIM 历史数据清理脚本
清理180天之前的消息数据
"""

import requests
import time
from datetime import datetime, timedelta

# 配置
WUKONGIM_API = "http://localhost:5001"
DAYS_TO_KEEP = 180  # 保留最近180天的数据
BATCH_SIZE = 100    # 每批处理的频道数量

def get_all_channels():
    """获取所有频道列表"""
    # 注意：WuKongIM 可能没有直接的"获取所有频道"接口
    # 需要从你的业务系统数据库中查询
    # 这里仅作示例
    
    # 示例：从MySQL查询
    import pymysql
    conn = pymysql.connect(
        host='localhost',
        user='root',
        password='password',
        database='your_business_db'
    )
    cursor = conn.cursor()
    cursor.execute("SELECT channel_id, channel_type FROM channels")
    channels = cursor.fetchall()
    conn.close()
    
    return channels

def get_channel_max_seq(channel_id, channel_type):
    """获取频道的最大消息序号"""
    url = f"{WUKONGIM_API}/channel/max_message_seq"
    params = {
        "channel_id": channel_id,
        "channel_type": channel_type
    }
    
    try:
        response = requests.get(url, params=params)
        if response.status_code == 200:
            data = response.json()
            return data.get("max_message_seq", 0)
    except Exception as e:
        print(f"获取频道 {channel_id} 最大序号失败: {e}")
    
    return 0

def calculate_cutoff_seq(channel_id, channel_type, days):
    """
    计算需要保留的消息序号
    这个方法需要根据实际情况调整
    """
    # 方法1: 假设消息均匀分布
    max_seq = get_channel_max_seq(channel_id, channel_type)
    if max_seq == 0:
        return 0
    
    # 假设频道创建至今的天数
    # 这个需要从数据库或API获取
    total_days = 365  # 示例：假设频道存在365天
    
    # 计算180天前的序号
    cutoff_seq = int(max_seq * (total_days - days) / total_days)
    
    return cutoff_seq

def clean_channel_messages(channel_id, channel_type, cutoff_timestamp):
    """
    清理频道的历史消息
    注意：WuKongIM 没有直接的批量删除API
    这里需要自定义实现
    """
    print(f"清理频道: {channel_id}, 类型: {channel_type}")
    
    # 方案1: 如果频道已废弃，直接删除整个频道
    # delete_channel(channel_id, channel_type)
    
    # 方案2: 保留频道，只删除历史消息
    # 需要通过数据库直接操作（见方案二）
    
    pass

def main():
    """主函数"""
    print("=" * 60)
    print("WuKongIM 历史数据清理脚本")
    print(f"清理 {DAYS_TO_KEEP} 天之前的数据")
    print("=" * 60)
    
    # 计算截止时间
    cutoff_date = datetime.now() - timedelta(days=DAYS_TO_KEEP)
    cutoff_timestamp = int(cutoff_date.timestamp())
    
    print(f"截止日期: {cutoff_date.strftime('%Y-%m-%d %H:%M:%S')}")
    print(f"截止时间戳: {cutoff_timestamp}")
    print()
    
    # 获取所有频道
    print("正在获取频道列表...")
    channels = get_all_channels()
    print(f"共找到 {len(channels)} 个频道")
    print()
    
    # 清理每个频道
    for i, (channel_id, channel_type) in enumerate(channels, 1):
        print(f"[{i}/{len(channels)}] 处理频道: {channel_id}")
        clean_channel_messages(channel_id, channel_type, cutoff_timestamp)
        
        # 避免请求过快
        time.sleep(0.1)
    
    print()
    print("清理完成！")

if __name__ == "__main__":
    main()
```


---

## 4. 方案二：数据库级别清理

### 4.1 清理原理

⚠️ **警告**: 此方案直接操作 Pebble 数据库文件，**风险较高**，操作前务必备份！

由于 Pebble 是嵌入式数据库，无法像 MySQL 那样直接执行 SQL 删除。需要：

1. 停止 WuKongIM 服务
2. 编写 Go 程序读取并过滤数据
3. 重新写入保留的数据
4. 重启服务

### 4.2 清理步骤

#### 步骤1: 备份数据

```bash
# 停止服务
systemctl stop wukongim

# 完整备份
tar -czf wukongimdata_backup_$(date +%Y%m%d).tar.gz ./wukongimdata

# 或使用 rsync
rsync -av ./wukongimdata/ ./wukongimdata_backup/
```

#### 步骤2: 编写清理程序

```go
// clean_old_messages.go
package main

import (
	"fmt"
	"path/filepath"
	"time"

	"github.com/WuKongIM/WuKongIM/pkg/wkdb"
	"github.com/cockroachdb/pebble"
)

func main() {
	dataDir := "./wukongimdata"
	daysToKeep := 180
	
	// 计算截止时间
	cutoffTime := time.Now().AddDate(0, 0, -daysToKeep)
	cutoffTimestamp := cutoffTime.Unix()
	
	fmt.Printf("清理 %s 之前的数据\n", cutoffTime.Format("2006-01-02 15:04:05"))
	fmt.Printf("截止时间戳: %d\n", cutoffTimestamp)
	
	// 打开数据库
	opts := wkdb.NewOptions(
		wkdb.WithDir(dataDir),
		wkdb.WithShardNum(8),
	)
	
	db := wkdb.NewWukongDB(opts)
	err := db.Open()
	if err != nil {
		panic(err)
	}
	defer db.Close()
	
	// 遍历所有频道，清理历史消息
	// 注意：这需要根据实际的数据结构实现
	// 以下是伪代码示例
	
	fmt.Println("开始清理历史消息...")
	
	// TODO: 实现具体的清理逻辑
	// 1. 遍历所有频道
	// 2. 对每个频道，查询消息时间戳
	// 3. 删除时间戳小于 cutoffTimestamp 的消息
	
	fmt.Println("清理完成！")
}
```

#### 步骤3: 执行清理

```bash
# 编译程序
go build -o clean_old_messages clean_old_messages.go

# 执行清理（确保已停止 WuKongIM）
./clean_old_messages

# 重启服务
systemctl start wukongim
```

### 4.3 方案二的问题

❌ **不推荐使用此方案**，原因：

1. **复杂度高**: 需要深入了解 Pebble 数据结构
2. **风险大**: 直接操作数据库，容易损坏数据
3. **需要停机**: 清理期间服务不可用
4. **难以实现**: WuKongIM 的数据结构复杂，难以准确定位历史数据

---

## 5. 方案三：归档历史数据（推荐）

### 5.1 归档原理

将历史数据导出到其他存储（如对象存储、冷存储），然后从 WuKongIM 中删除。

### 5.2 归档流程

```
1. 导出历史数据
   ↓
2. 上传到对象存储（OSS/S3）
   ↓
3. 验证数据完整性
   ↓
4. 从 WuKongIM 删除
   ↓
5. 定期清理归档数据
```

### 5.3 归档脚本

```bash
#!/bin/bash
# archive_old_data.sh
# 归档180天之前的数据

set -e

# 配置
DATA_DIR="./wukongimdata"
ARCHIVE_DIR="/backup/wukongim_archive"
DAYS_TO_KEEP=180
DATE_SUFFIX=$(date +%Y%m%d)

# 计算截止日期
CUTOFF_DATE=$(date -d "$DAYS_TO_KEEP days ago" +%Y-%m-%d)

echo "=========================================="
echo "WuKongIM 数据归档脚本"
echo "归档日期: $CUTOFF_DATE 之前的数据"
echo "=========================================="

# 创建归档目录
mkdir -p "$ARCHIVE_DIR"

# 步骤1: 备份当前数据
echo "步骤1: 备份当前数据..."
BACKUP_FILE="$ARCHIVE_DIR/wukongim_full_backup_$DATE_SUFFIX.tar.gz"
tar -czf "$BACKUP_FILE" "$DATA_DIR"
echo "备份完成: $BACKUP_FILE"

# 步骤2: 导出历史数据（需要自定义实现）
echo "步骤2: 导出历史数据..."
# 这里需要编写程序从 Pebble 中读取历史数据
# ./export_old_messages --cutoff-date="$CUTOFF_DATE" --output="$ARCHIVE_DIR/messages_$DATE_SUFFIX.json"

# 步骤3: 上传到对象存储
echo "步骤3: 上传到对象存储..."
# 使用阿里云 OSS CLI
# ossutil cp "$BACKUP_FILE" oss://your-bucket/wukongim/archive/

# 或使用 AWS S3
# aws s3 cp "$BACKUP_FILE" s3://your-bucket/wukongim/archive/

# 步骤4: 验证上传
echo "步骤4: 验证上传..."
# 检查文件是否成功上传

# 步骤5: 清理本地归档文件（可选）
echo "步骤5: 清理本地归档文件..."
# 保留最近30天的归档
find "$ARCHIVE_DIR" -name "wukongim_*.tar.gz" -mtime +30 -delete

echo "归档完成！"
```

### 5.4 定期执行归档

```bash
# 添加到 crontab，每月1号凌晨2点执行
0 2 1 * * /path/to/archive_old_data.sh >> /var/log/wukongim_archive.log 2>&1
```


---

## 6. 方案四：定期备份+重建（适合小规模）

### 6.1 重建原理

定期将最近 N 天的数据导出，然后重建 WuKongIM 数据库。

### 6.2 重建流程

```
1. 导出最近180天的数据
   ↓
2. 停止 WuKongIM 服务
   ↓
3. 备份旧数据目录
   ↓
4. 删除旧数据目录
   ↓
5. 启动 WuKongIM（自动创建新数据库）
   ↓
6. 导入最近180天的数据
   ↓
7. 验证数据完整性
```

### 6.3 重建脚本

```bash
#!/bin/bash
# rebuild_database.sh
# 重建数据库，只保留最近180天的数据

set -e

# 配置
DATA_DIR="./wukongimdata"
BACKUP_DIR="/backup/wukongim"
DAYS_TO_KEEP=180
DATE_SUFFIX=$(date +%Y%m%d_%H%M%S)

echo "=========================================="
echo "WuKongIM 数据库重建脚本"
echo "保留最近 $DAYS_TO_KEEP 天的数据"
echo "=========================================="

# 步骤1: 导出最近180天的数据
echo "步骤1: 导出最近180天的数据..."
# 需要自定义实现导出程序
# ./export_recent_data --days=$DAYS_TO_KEEP --output="$BACKUP_DIR/recent_data_$DATE_SUFFIX.json"

# 步骤2: 停止服务
echo "步骤2: 停止 WuKongIM 服务..."
systemctl stop wukongim
sleep 5

# 步骤3: 备份旧数据
echo "步骤3: 备份旧数据..."
if [ -d "$DATA_DIR" ]; then
    mv "$DATA_DIR" "$BACKUP_DIR/wukongimdata_old_$DATE_SUFFIX"
    echo "旧数据已移动到: $BACKUP_DIR/wukongimdata_old_$DATE_SUFFIX"
fi

# 步骤4: 启动服务（自动创建新数据库）
echo "步骤4: 启动 WuKongIM 服务..."
systemctl start wukongim
sleep 10

# 步骤5: 导入最近的数据
echo "步骤5: 导入最近的数据..."
# 需要自定义实现导入程序
# ./import_data --input="$BACKUP_DIR/recent_data_$DATE_SUFFIX.json"

# 步骤6: 验证数据
echo "步骤6: 验证数据..."
# 检查服务状态
systemctl status wukongim

# 检查数据目录大小
du -sh "$DATA_DIR"

# 步骤7: 清理旧备份（保留最近3次）
echo "步骤7: 清理旧备份..."
cd "$BACKUP_DIR"
ls -t wukongimdata_old_* | tail -n +4 | xargs -r rm -rf

echo "=========================================="
echo "数据库重建完成！"
echo "新数据目录: $DATA_DIR"
echo "旧数据备份: $BACKUP_DIR/wukongimdata_old_$DATE_SUFFIX"
echo "=========================================="
```

### 6.4 方案四的优缺点

**优点**：
- ✅ 彻底清理，数据库体积大幅减小
- ✅ 数据库性能恢复到最佳状态
- ✅ 适合定期维护

**缺点**：
- ❌ 需要停机，服务中断
- ❌ 需要导出/导入工具
- ❌ 操作复杂，风险较高

---

## 7. 自动化清理脚本

### 7.1 综合清理脚本（推荐）

结合多种方案的优点，提供一个实用的清理脚本：

```bash
#!/bin/bash
# wukongim_cleanup.sh
# WuKongIM 自动化数据清理脚本

set -e

#==================== 配置区 ====================
DATA_DIR="./wukongimdata"
BACKUP_DIR="/backup/wukongim"
ARCHIVE_DIR="/archive/wukongim"
DAYS_TO_KEEP=180
LOG_FILE="/var/log/wukongim_cleanup.log"

# 阿里云OSS配置（可选）
OSS_BUCKET="your-bucket"
OSS_PATH="wukongim/archive"

#==================== 函数定义 ====================

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

check_disk_space() {
    local required_space=$1  # GB
    local available_space=$(df -BG "$DATA_DIR" | awk 'NR==2 {print $4}' | sed 's/G//')
    
    if [ "$available_space" -lt "$required_space" ]; then
        log "错误: 磁盘空间不足！需要 ${required_space}GB，可用 ${available_space}GB"
        exit 1
    fi
    
    log "磁盘空间检查通过: 可用 ${available_space}GB"
}

backup_data() {
    log "开始备份数据..."
    
    local backup_file="$BACKUP_DIR/wukongim_backup_$(date +%Y%m%d_%H%M%S).tar.gz"
    mkdir -p "$BACKUP_DIR"
    
    tar -czf "$backup_file" "$DATA_DIR"
    
    if [ $? -eq 0 ]; then
        log "备份成功: $backup_file"
        log "备份大小: $(du -h $backup_file | cut -f1)"
    else
        log "错误: 备份失败！"
        exit 1
    fi
}

get_data_size() {
    du -sh "$DATA_DIR" | cut -f1
}

clean_old_backups() {
    log "清理旧备份文件..."
    
    # 保留最近7天的备份
    find "$BACKUP_DIR" -name "wukongim_backup_*.tar.gz" -mtime +7 -delete
    
    log "旧备份清理完成"
}

upload_to_oss() {
    local file=$1
    
    if command -v ossutil &> /dev/null; then
        log "上传到阿里云OSS..."
        ossutil cp "$file" "oss://$OSS_BUCKET/$OSS_PATH/"
        
        if [ $? -eq 0 ]; then
            log "上传成功"
        else
            log "警告: 上传失败"
        fi
    else
        log "提示: ossutil 未安装，跳过OSS上传"
    fi
}

monitor_cleanup() {
    log "监控清理效果..."
    
    local before_size=$1
    local after_size=$(get_data_size)
    
    log "清理前大小: $before_size"
    log "清理后大小: $after_size"
}

#==================== 主流程 ====================

main() {
    log "=========================================="
    log "WuKongIM 数据清理脚本开始执行"
    log "保留最近 $DAYS_TO_KEEP 天的数据"
    log "=========================================="
    
    # 1. 检查磁盘空间
    check_disk_space 50
    
    # 2. 记录清理前的数据大小
    before_size=$(get_data_size)
    log "当前数据大小: $before_size"
    
    # 3. 备份数据
    backup_data
    
    # 4. 执行清理（这里需要根据实际情况实现）
    log "开始清理历史数据..."
    
    # 方法1: 如果有清理API，调用API
    # python3 /path/to/clean_old_messages.py --days=$DAYS_TO_KEEP
    
    # 方法2: 如果使用数据库级别清理
    # systemctl stop wukongim
    # ./clean_old_messages --days=$DAYS_TO_KEEP
    # systemctl start wukongim
    
    log "清理完成"
    
    # 5. 监控清理效果
    monitor_cleanup "$before_size"
    
    # 6. 上传备份到OSS（可选）
    # upload_to_oss "$backup_file"
    
    # 7. 清理旧备份
    clean_old_backups
    
    log "=========================================="
    log "数据清理脚本执行完成"
    log "=========================================="
}

# 执行主流程
main
```

### 7.2 定时任务配置

```bash
# 编辑 crontab
crontab -e

# 添加定时任务
# 每月1号凌晨2点执行清理
0 2 1 * * /path/to/wukongim_cleanup.sh

# 或者每周日凌晨3点执行
0 3 * * 0 /path/to/wukongim_cleanup.sh
```

### 7.3 监控告警

```bash
#!/bin/bash
# monitor_disk.sh
# 监控磁盘使用率，超过阈值发送告警

THRESHOLD=80  # 告警阈值（百分比）
DATA_DIR="./wukongimdata"

usage=$(df -h "$DATA_DIR" | awk 'NR==2 {print $5}' | sed 's/%//')

if [ "$usage" -gt "$THRESHOLD" ]; then
    echo "警告: WuKongIM 数据目录使用率已达 ${usage}%"
    
    # 发送邮件告警
    # echo "磁盘使用率: ${usage}%" | mail -s "WuKongIM 磁盘告警" admin@example.com
    
    # 或发送钉钉/企业微信通知
    # curl -X POST "https://oapi.dingtalk.com/robot/send?access_token=xxx" \
    #   -H "Content-Type: application/json" \
    #   -d "{\"msgtype\":\"text\",\"text\":{\"content\":\"WuKongIM磁盘使用率: ${usage}%\"}}"
fi
```


---

## 8. 注意事项

### 8.1 清理前的准备

#### ✅ 必须做的事情

1. **完整备份**
```bash
# 备份数据目录
tar -czf wukongimdata_backup_$(date +%Y%m%d).tar.gz ./wukongimdata

# 验证备份文件
tar -tzf wukongimdata_backup_$(date +%Y%m%d).tar.gz | head -20
```

2. **测试环境验证**
- 先在测试环境执行清理
- 验证清理后服务正常
- 确认数据完整性

3. **通知用户**
- 提前通知用户可能的服务中断
- 说明历史消息的保留策略

4. **准备回滚方案**
```bash
# 如果清理失败，快速回滚
systemctl stop wukongim
rm -rf ./wukongimdata
tar -xzf wukongimdata_backup_20231224.tar.gz
systemctl start wukongim
```

#### ⚠️ 风险提示

| 风险 | 影响 | 预防措施 |
|------|------|---------|
| 数据丢失 | 严重 | 完整备份 + 验证 |
| 服务中断 | 中等 | 选择低峰期 + 快速回滚 |
| 数据损坏 | 严重 | 测试环境验证 |
| 磁盘空间不足 | 中等 | 提前检查磁盘空间 |

### 8.2 清理后的验证

#### 1. 服务状态检查

```bash
# 检查服务是否运行
systemctl status wukongim

# 检查日志是否有错误
tail -f logs/wukongim.log

# 检查端口是否监听
netstat -tlnp | grep 5001
```

#### 2. 数据完整性检查

```bash
# 检查数据目录大小
du -sh ./wukongimdata

# 检查分片是否完整
ls -lh ./wukongimdata/wukongimdb/

# 通过API检查数据
curl http://localhost:5001/varz
```

#### 3. 功能测试

```bash
# 测试发送消息
curl -X POST http://localhost:5001/message/send \
  -H "Content-Type: application/json" \
  -d '{
    "from_uid": "test_user",
    "channel_id": "test_channel",
    "channel_type": 2,
    "payload": "测试消息"
  }'

# 测试查询消息
curl http://localhost:5001/messages?channel_id=test_channel&channel_type=2
```

### 8.3 最佳实践

#### 1. 定期清理策略

```
建议清理周期：
- 小规模（<10万消息/天）: 每季度清理一次
- 中等规模（10-100万消息/天）: 每月清理一次
- 大规模（>100万消息/天）: 每周清理一次
```

#### 2. 数据保留策略

```
推荐保留时长：
- 活跃频道: 保留180天
- 普通频道: 保留90天
- 临时频道: 保留30天
- 系统消息: 保留365天
```

#### 3. 监控指标

```bash
# 监控数据目录大小
watch -n 60 'du -sh ./wukongimdata'

# 监控磁盘使用率
df -h | grep wukongimdata

# 监控数据库性能
curl http://localhost:5001/varz | jq '.db'
```

### 8.4 常见问题

#### Q1: 清理后数据库大小没有明显减小？

**原因**: Pebble 使用 LSM-tree 结构，删除数据后需要压缩（Compaction）才能释放空间。

**解决方案**:
```bash
# 重启服务触发压缩
systemctl restart wukongim

# 或等待自动压缩（可能需要几小时）
```

#### Q2: 清理过程中服务能否继续运行？

**回答**: 
- ✅ 如果使用 API 清理，服务可以继续运行
- ❌ 如果直接操作数据库文件，必须停止服务

#### Q3: 如何恢复误删的数据？

**解决方案**:
```bash
# 1. 停止服务
systemctl stop wukongim

# 2. 恢复备份
rm -rf ./wukongimdata
tar -xzf wukongimdata_backup_20231224.tar.gz

# 3. 启动服务
systemctl start wukongim
```

#### Q4: 清理会影响用户体验吗？

**影响分析**:
- ✅ 历史消息无法查看（超过保留期）
- ✅ 最近消息不受影响
- ✅ 用户列表、频道信息不受影响
- ⚠️ 如果停机清理，服务会短暂中断

#### Q5: 能否只清理某些频道的数据？

**可以**，通过脚本实现：
```python
# 只清理指定频道
channels_to_clean = [
    ("channel1", 2),
    ("channel2", 2),
]

for channel_id, channel_type in channels_to_clean:
    clean_channel_messages(channel_id, channel_type, cutoff_timestamp)
```

### 8.5 紧急情况处理

#### 场景1: 清理过程中服务崩溃

```bash
# 1. 立即停止清理脚本
kill -9 $(pgrep -f cleanup)

# 2. 检查数据完整性
ls -lh ./wukongimdata/wukongimdb/

# 3. 如果数据损坏，恢复备份
systemctl stop wukongim
rm -rf ./wukongimdata
tar -xzf wukongimdata_backup_latest.tar.gz
systemctl start wukongim
```

#### 场景2: 磁盘空间不足

```bash
# 1. 立即清理临时文件
rm -rf /tmp/wukongim_*

# 2. 清理日志文件
find ./logs -name "*.log" -mtime +7 -delete

# 3. 清理旧备份
find /backup -name "wukongim_*.tar.gz" -mtime +30 -delete

# 4. 如果还不够，扩容磁盘
# 或将数据迁移到更大的磁盘
```

#### 场景3: 数据损坏

```bash
# 1. 停止服务
systemctl stop wukongim

# 2. 尝试修复（Pebble 自带修复功能）
# 注意：这个需要编写 Go 程序调用 Pebble 的修复接口

# 3. 如果无法修复，恢复最近的备份
rm -rf ./wukongimdata
tar -xzf wukongimdata_backup_latest.tar.gz

# 4. 启动服务
systemctl start wukongim
```

---

## 9. 总结与建议

### 9.1 推荐方案

根据不同规模选择合适的方案：

| 规模 | 推荐方案 | 清理周期 | 停机时间 |
|------|---------|---------|---------|
| 小型（<1GB） | 方案四：备份+重建 | 每季度 | 10-30分钟 |
| 中型（1-10GB） | 方案三：归档+API清理 | 每月 | 无需停机 |
| 大型（>10GB） | 方案三：归档+分批清理 | 每周 | 无需停机 |

### 9.2 实施步骤

**第一次清理**：
1. 完整备份数据
2. 在测试环境验证
3. 选择低峰期执行
4. 实时监控清理过程
5. 验证数据完整性

**后续清理**：
1. 配置自动化脚本
2. 设置定时任务
3. 配置监控告警
4. 定期检查备份

### 9.3 关键提示

⚠️ **重要**：
- ✅ 清理前必须完整备份
- ✅ 先在测试环境验证
- ✅ 选择业务低峰期执行
- ✅ 准备快速回滚方案
- ✅ 清理后验证数据完整性

💡 **建议**：
- 定期清理比一次性大量清理更安全
- 归档历史数据比直接删除更保险
- 使用集群模式可以减少清理风险
- 监控磁盘使用率，提前规划清理

---

## 附录

### A. 相关命令速查

```bash
# 查看数据目录大小
du -sh ./wukongimdata

# 查看各分片大小
du -sh ./wukongimdata/wukongimdb/shard*

# 查看磁盘使用率
df -h | grep wukongimdata

# 备份数据
tar -czf backup.tar.gz ./wukongimdata

# 恢复数据
tar -xzf backup.tar.gz

# 检查服务状态
systemctl status wukongim

# 查看日志
tail -f logs/wukongim.log
```

### B. 联系支持

- **GitHub Issues**: https://github.com/WuKongIM/WuKongIM/issues
- **官方文档**: https://githubim.com
- **微信**: wukongimgo

---

**文档版本**: v1.0  
**更新日期**: 2024-12-24  
**适用版本**: WuKongIM 2.x

