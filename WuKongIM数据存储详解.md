# WuKongIM 数据存储详解

## 📚 目录

- [1. 存储架构概述](#1-存储架构概述)
- [2. 存储目录结构](#2-存储目录结构)
- [3. 存储的数据类型](#3-存储的数据类型)
- [4. Pebble KV 数据库](#4-pebble-kv-数据库)
- [5. 数据分片机制](#5-数据分片机制)
- [6. 消息持久化机制](#6-消息持久化机制)
- [7. 文件存储方案](#7-文件存储方案)
- [8. 数据库配置](#8-数据库配置)
- [9. 数据备份与迁移](#9-数据备份与迁移)
- [10. 常见问题](#10-常见问题)

---

## 1. 存储架构概述

### 1.1 核心存储引擎

WuKongIM 使用**自研的分布式存储系统**，底层基于 **Pebble KV 数据库**：

- **存储引擎**: Pebble（类似 RocksDB 的 LSM-tree 键值存储）
- **存储位置**: 本地文件系统
- **默认路径**: `./wukongimdata`（可通过 `rootDir` 配置修改）
- **数据分片**: 支持多分片（默认 8 个），提升并发性能

### 1.2 架构特点

✅ **高性能**: 基于 LSM-tree，优化高并发写入  
✅ **永久存储**: 消息默认永久保存，支持消息漫游  
✅ **自动分片**: 数据自动分散到多个分片，提升性能  
✅ **无依赖**: 不依赖 MySQL、Redis 等外部数据库  
✅ **分布式**: 支持集群模式，数据自动备份

---

## 2. 存储目录结构

### 2.1 完整目录树

```
wukongimdata/                    # rootDir 配置的根目录
├── wukongimdb/                  # Pebble KV 数据库目录
│   ├── shard000/                # 分片0 - Pebble数据库实例
│   │   ├── 000001.log          # WAL日志文件
│   │   ├── 000002.sst          # SSTable文件（存储实际数据）
│   │   ├── 000003.sst
│   │   ├── MANIFEST-000001     # 元数据清单
│   │   ├── CURRENT             # 当前MANIFEST指针
│   │   └── OPTIONS-000003      # 数据库配置
│   ├── shard001/                # 分片1
│   ├── shard002/                # 分片2
│   ├── shard003/                # 分片3
│   ├── shard004/                # 分片4
│   ├── shard005/                # 分片5
│   ├── shard006/                # 分片6
│   └── shard007/                # 分片7（默认8个分片）
├── diskqueue/                   # Webhook消息队列
│   └── wk_webhook_q.*.dat      # 磁盘队列文件
├── conversationv2/              # 最近会话备份
│   └── conversations.json      # 会话数据备份
└── cluster/                     # 集群相关数据（集群模式）
    ├── config/
    │   └── remote.json         # 集群配置
    ├── logdb/                  # 集群日志数据库
    └── cfglogdb/               # 配置日志数据库
```

### 2.2 Pebble 数据库文件说明

每个 `shardXXX` 目录都是一个独立的 **Pebble KV 数据库实例**：

| 文件类型 | 说明 | 作用 |
|---------|------|------|
| `.sst` 文件 | SSTable（Sorted String Table） | 存储实际的键值对数据，不可变 |
| `.log` 文件 | WAL（Write-Ahead Log） | 预写日志，保证数据持久性 |
| `MANIFEST-*` | 元数据清单 | 记录所有SSTable文件的状态和版本 |
| `CURRENT` | 当前指针 | 指向当前使用的MANIFEST文件 |
| `OPTIONS-*` | 配置文件 | 记录数据库的配置选项 |


---

## 3. 存储的数据类型

### 3.1 持久化存储的数据

WuKongIM 在 Pebble KV 数据库中**永久存储**以下数据：

#### ✅ 1. 消息数据（Message）

```go
type Message struct {
    MessageID    int64   // 消息ID（全局唯一）
    MessageSeq   uint32  // 消息序号（频道内递增）
    ClientMsgNo  string  // 客户端消息编号
    StreamNo     string  // 流式消息编号
    Timestamp    int32   // 时间戳
    FromUID      string  // 发送者UID
    ChannelID    string  // 频道ID
    ChannelType  uint8   // 频道类型（1=个人 2=群组）
    Topic        string  // 主题
    Payload      []byte  // 消息内容（JSON格式）
    Setting      uint8   // 消息设置
    Expire       uint32  // 过期时间
    Term         uint64  // Raft任期
}
```

**存储内容**：
- ✅ 消息的完整内容（文本、JSON等）
- ✅ 消息元数据（发送者、接收者、时间等）
- ✅ 消息状态（已读、未读等）
- ❌ **不存储文件本身**（图片、视频、文档等二进制文件）

**Payload 示例**：
```json
// 文本消息
{
  "type": "text",
  "content": "你好，世界！"
}

// 图片消息（只存储URL）
{
  "type": "image",
  "url": "https://your-oss.com/images/photo.jpg",
  "width": 1920,
  "height": 1080,
  "size": 204800
}

// 文件消息（只存储URL）
{
  "type": "file",
  "url": "https://your-oss.com/files/document.pdf",
  "name": "项目文档.pdf",
  "size": 1048576
}
```

#### ✅ 2. 用户信息（User）

```go
type User struct {
    Uid               string  // 用户UID
    DeviceCount       uint32  // 设备数量
    OnlineDeviceCount uint32  // 在线设备数量
    ConnCount         uint32  // 连接数量
    SendMsgCount      uint64  // 发送消息数量
    RecvMsgCount      uint64  // 接收消息数量
    SendMsgBytes      uint64  // 发送字节数
    RecvMsgBytes      uint64  // 接收字节数
    PluginNo          string  // 插件编号
    CreatedAt         *time.Time
    UpdatedAt         *time.Time
}
```


#### ✅ 3. 设备信息（Device）

```go
type Device struct {
    Uid          string  // 用户UID
    Token        string  // 设备Token
    DeviceFlag   uint64  // 设备标识
    DeviceLevel  uint8   // 设备等级
    ConnCount    uint32  // 连接数
    SendMsgCount uint64  // 发送消息数
    RecvMsgCount uint64  // 接收消息数
    CreatedAt    *time.Time
    UpdatedAt    *time.Time
}
```

#### ✅ 4. 频道信息（ChannelInfo）

```go
type ChannelInfo struct {
    ChannelId       string  // 频道ID
    ChannelType     uint8   // 频道类型（1=个人 2=群组）
    Ban             bool    // 是否封禁
    Large           bool    // 是否超大群
    Disband         bool    // 是否解散
    SubscriberCount int     // 订阅者数量
    DenylistCount   int     // 黑名单数量
    AllowlistCount  int     // 白名单数量
    LastMsgSeq      uint64  // 最新消息序号
    LastMsgTime     uint64  // 最后消息时间
    Webhook         string  // Webhook地址
    SendBan         bool    // 是否禁止发消息
    AllowStranger   bool    // 是否允许陌生人
}
```

#### ✅ 5. 频道成员关系

- **订阅者列表（Subscribers）**: 谁订阅了这个频道
- **黑名单（Denylist）**: 被禁止的用户
- **白名单（Allowlist）**: 允许的用户

```go
type Member struct {
    Uid       string
    CreatedAt *time.Time
    UpdatedAt *time.Time
}
```

#### ✅ 6. 最近会话（Conversation）

```go
type Conversation struct {
    Uid             string  // 用户UID
    ChannelId       string  // 频道ID
    ChannelType     uint8   // 频道类型
    UnreadCount     uint32  // 未读数量
    ReadToMsgSeq    uint64  // 已读到的消息序号
    DeletedAtMsgSeq uint64  // 删除位置
    CreatedAt       *time.Time
    UpdatedAt       *time.Time
}
```

#### ✅ 7. 分布式集群配置（ChannelClusterConfig）

```go
type ChannelClusterConfig struct {
    ChannelId       string
    ChannelType     uint8
    ReplicaMaxCount uint16   // 副本数量
    Replicas        []uint64 // 副本节点ID
    Learners        []uint64 // 学习者节点
    LeaderId        uint64   // 领导者ID
    Term            uint32   // 任期
    Status          ChannelClusterStatus
}
```

#### ✅ 8. 流式消息（Stream）

用于支持类似 ChatGPT 的流式输出。


### 3.2 不存储的数据

WuKongIM **不存储**以下内容：

| 数据类型 | 是否存储 | 推荐存储位置 |
|---------|---------|------------|
| 图片文件 | ❌ 否 | 阿里云OSS / 七牛云 / AWS S3 |
| 视频文件 | ❌ 否 | 阿里云OSS / 七牛云 / AWS S3 |
| 音频文件 | ❌ 否 | 阿里云OSS / 七牛云 / AWS S3 |
| 文档文件 | ❌ 否 | 阿里云OSS / 七牛云 / AWS S3 |
| 用户详细资料 | ❌ 否 | 你的业务系统（MySQL） |
| 群组详细信息 | ❌ 否 | 你的业务系统（MySQL） |
| 好友关系 | ❌ 否 | 你的业务系统（MySQL） |
| 业务逻辑数据 | ❌ 否 | 你的业务系统（MySQL） |

---

## 4. Pebble KV 数据库

### 4.1 什么是 Pebble？

Pebble 是一个高性能的 **LSM-tree（Log-Structured Merge-tree）键值存储引擎**：

- 🚀 **高性能**: 优化了高并发写入场景
- 📦 **嵌入式**: 直接嵌入到应用程序中，无需独立部署
- 🔄 **自动压缩**: 自动进行数据压缩（Compaction）
- 💾 **持久化**: 数据持久化到磁盘，支持崩溃恢复
- 🎯 **兼容性**: API 类似 RocksDB/LevelDB

### 4.2 LSM-tree 存储原理

```
写入流程：
1. 写入 WAL 日志（保证持久性）
2. 写入 MemTable（内存表）
3. MemTable 满后刷到磁盘（SSTable）
4. 后台自动压缩（Compaction）

读取流程：
1. 先查 MemTable
2. 再查 SSTable（从新到旧）
3. 使用 Bloom Filter 加速查找
```

### 4.3 数据存储格式

Pebble 使用 **Key-Value** 格式存储数据：

```
Key 格式示例：
- 消息主键:     msg:channel_id:channel_type:message_seq
- 消息索引:     msg_idx:message_id
- 用户信息:     user:uid:column_name
- 频道信息:     channel:channel_id:channel_type:column_name
- 最近会话:     conversation:uid:id:column_name

Value:
- 二进制编码的数据（使用自定义编码协议）
```

### 4.4 性能特点

| 操作类型 | 性能 | 说明 |
|---------|------|------|
| 顺序写入 | ⭐⭐⭐⭐⭐ | 非常快，直接写入 WAL 和 MemTable |
| 随机写入 | ⭐⭐⭐⭐ | 快，LSM-tree 优化了写入 |
| 顺序读取 | ⭐⭐⭐⭐ | 快，利用缓存和预读 |
| 随机读取 | ⭐⭐⭐ | 中等，需要查找多个 SSTable |
| 范围查询 | ⭐⭐⭐⭐ | 快，SSTable 有序存储 |


---

## 5. 数据分片机制

### 5.1 为什么需要分片？

- 🚀 **提升并发性能**: 多个分片可以并行读写
- 📦 **避免单库过大**: 数据分散存储，单个数据库不会太大
- 🔄 **负载均衡**: 数据均匀分布到各个分片

### 5.2 分片策略

```go
// 默认配置：8个分片
ShardNum: 8

// 数据如何分配到分片
func shardId(key string) uint32 {
    h := fnv.New32()           // FNV-1a 哈希算法
    h.Write([]byte(key))
    return h.Sum32() % shardNum  // 哈希取模
}
```

**分片规则**：
- **用户数据**: 根据 `uid` 哈希分片
- **频道数据**: 根据 `channelId` 哈希分片
- **消息数据**: 根据 `channelId` 哈希分片（同一频道的消息在同一分片）

### 5.3 分片示例

```
用户 "user001" → hash("user001") % 8 = 3 → shard003
用户 "user002" → hash("user002") % 8 = 7 → shard007
频道 "group123" → hash("group123") % 8 = 1 → shard001
```

### 5.4 分片配置

```yaml
# 在代码中配置，不能动态修改
ShardNum: 8  # 默认8个分片

# ⚠️ 注意：一旦设置就不能修改！
# 修改分片数会导致数据无法读取
```

---

## 6. 消息持久化机制

### 6.1 消息存储策略

✅ **永久存储**: 所有消息默认永久保存  
✅ **消息漫游**: 换设备登录，消息不丢失  
✅ **按频道分片**: 同一频道的消息存储在同一分片  
✅ **多重索引**: 支持按 MessageID、MessageSeq、FromUID 等查询

### 6.2 消息写入流程

```
1. 客户端发送消息
   ↓
2. WuKongIM 接收消息
   ↓
3. 生成 MessageID（全局唯一）
   ↓
4. 生成 MessageSeq（频道内递增）
   ↓
5. 写入 Pebble 数据库
   ├─ 写入 WAL 日志
   ├─ 写入 MemTable
   └─ 创建索引
   ↓
6. 返回 ACK 给客户端
   ↓
7. 推送给接收者
```

### 6.3 消息查询接口

```go
// 加载最近N条消息
LoadLastMsgs(channelId, channelType, limit)

// 加载指定范围的消息（向上翻页）
LoadPrevRangeMsgs(channelId, channelType, startSeq, endSeq, limit)

// 加载指定范围的消息（向下翻页）
LoadNextRangeMsgs(channelId, channelType, startSeq, endSeq, limit)

// 根据MessageID查询
GetMessage(messageId)

// 搜索消息
SearchMessages(req)
```


### 6.4 消息索引

WuKongIM 为消息创建了多个索引，加速查询：

| 索引类型 | Key 格式 | 用途 |
|---------|---------|------|
| 主键索引 | `msg:channelId:channelType:messageSeq` | 按序号查询消息 |
| MessageID索引 | `msg_idx:messageId` | 按全局ID查询 |
| FromUID索引 | `msg_from:fromUid:primaryKey` | 查询某人发送的消息 |
| ClientMsgNo索引 | `msg_client:clientMsgNo:primaryKey` | 按客户端编号查询 |

### 6.5 消息过期

```yaml
# 消息可以设置过期时间
expire: 86400  # 24小时后过期（秒）

# 过期后：
# - 消息仍然存储在数据库中
# - 但客户端拉取时会被过滤掉
# - 可以通过API手动删除过期消息
```

---

## 7. 文件存储方案

### 7.1 WuKongIM 不存储文件

⚠️ **重要**: WuKongIM **只存储消息内容**，不存储文件本身！

### 7.2 正确的文件存储架构

```
┌─────────────────────────────────────────────────────┐
│                    客户端                            │
└──────────┬──────────────────────────┬───────────────┘
           │                          │
           │ 1. 上传文件              │ 3. 发送消息
           ▼                          ▼
    ┌─────────────┐           ┌──────────────┐
    │  阿里云OSS   │           │  WuKongIM    │
    │  (文件存储)  │           │  (消息传输)  │
    └─────────────┘           └──────┬───────┘
           │                          │
           │ 2. 返回URL               │ 4. Webhook通知
           │                          ▼
           │                   ┌──────────────┐
           └──────────────────▶│  业务系统     │
                               │  (MySQL)     │
                               └──────────────┘
```

### 7.3 发送图片消息示例

#### 步骤1: 上传图片到阿里云OSS

```javascript
// 前端代码
import OSS from 'ali-oss';

const client = new OSS({
  region: 'oss-cn-hangzhou',
  accessKeyId: 'your-access-key-id',
  accessKeySecret: 'your-access-key-secret',
  bucket: 'your-bucket-name'
});

// 上传图片
async function uploadImage(file) {
  const result = await client.put(`images/${Date.now()}_${file.name}`, file);
  return result.url;  // 返回图片URL
}
```

#### 步骤2: 发送包含URL的消息

```javascript
// 上传图片
const imageUrl = await uploadImage(imageFile);

// 发送消息
im.send(targetUid, WKIMChannelType.Person, {
  type: "image",
  url: imageUrl,           // OSS返回的URL
  width: 1920,
  height: 1080,
  size: 204800,
  thumbnail: thumbnailUrl  // 缩略图URL（可选）
});
```

#### 步骤3: WuKongIM 存储的消息

```json
{
  "message_id": 123456789,
  "message_seq": 100,
  "from_uid": "user001",
  "channel_id": "user002",
  "channel_type": 1,
  "payload": "{\"type\":\"image\",\"url\":\"https://your-oss.com/images/photo.jpg\",\"width\":1920,\"height\":1080,\"size\":204800}",
  "timestamp": 1703001234
}
```


### 7.4 阿里云OSS配置示例

```javascript
// Node.js 后端示例
const OSS = require('ali-oss');

const client = new OSS({
  region: 'oss-cn-hangzhou',
  accessKeyId: process.env.OSS_ACCESS_KEY_ID,
  accessKeySecret: process.env.OSS_ACCESS_KEY_SECRET,
  bucket: 'your-bucket-name'
});

// 上传文件
async function uploadFile(file, filename) {
  try {
    const result = await client.put(filename, file);
    return {
      url: result.url,
      name: result.name
    };
  } catch (e) {
    console.error(e);
    throw e;
  }
}

// 生成签名URL（临时访问）
function generateSignedUrl(filename, expires = 3600) {
  return client.signatureUrl(filename, {
    expires: expires  // 过期时间（秒）
  });
}
```

### 7.5 其他云存储方案

| 云服务商 | SDK | 文档 |
|---------|-----|------|
| 阿里云OSS | `ali-oss` | https://help.aliyun.com/product/31815.html |
| 腾讯云COS | `cos-nodejs-sdk-v5` | https://cloud.tencent.com/document/product/436 |
| 七牛云 | `qiniu` | https://developer.qiniu.com/kodo |
| AWS S3 | `aws-sdk` | https://aws.amazon.com/cn/s3/ |
| MinIO | `minio` | https://min.io/ (自建) |

---

## 8. 数据库配置

### 8.1 基础配置

```yaml
# config/wk.yaml
rootDir: "./wukongimdata"  # 数据存储根目录

# 实际存储路径：
# ./wukongimdata/wukongimdb/shard000/
# ./wukongimdata/wukongimdb/shard001/
# ...
```

### 8.2 高级配置（代码级别）

```go
// pkg/wkdb/options.go
type Options struct {
    DataDir           string  // 数据目录
    ShardNum          int     // 分片数量（默认8，不可修改）
    MemTableSize      int     // 内存表大小（默认16MB）
    ConversationLimit int     // 最近会话查询限制（默认10000）
    SlotCount         int     // 槽位数量（默认128）
    EnableCost        bool    // 是否启用耗时统计
    BatchPerSize      int     // 批量操作大小（默认10240）
}

// 默认配置
ShardNum:          8
MemTableSize:      16 * 1024 * 1024  // 16MB
ConversationLimit: 10000
SlotCount:         128
BatchPerSize:      10240
```

### 8.3 Pebble 数据库配置

```go
// pkg/wkdb/wukongdb.go
func defaultPebbleOptions() *pebble.Options {
    return &pebble.Options{
        // 内存表大小
        MemTableSize: 16 * 1024 * 1024,  // 16MB
        
        // 内存表停止写入阈值
        MemTableStopWritesThreshold: 4,
        
        // MANIFEST文件最大大小
        MaxManifestFileSize: 128 * 1024 * 1024,  // 128MB
        
        // Level 0 最大字节数
        LBaseMaxBytes: 4 * 1024 * 1024 * 1024,  // 4GB
        
        // Level 0 压缩文件阈值
        L0CompactionFileThreshold: 8,
        
        // Level 0 停止写入阈值
        L0StopWritesThreshold: 24,
    }
}
```


### 8.4 存储空间估算

| 数据类型 | 单条大小 | 10万条 | 100万条 | 1000万条 |
|---------|---------|--------|---------|----------|
| 文本消息 | ~500B | 50MB | 500MB | 5GB |
| 图片消息（URL） | ~800B | 80MB | 800MB | 8GB |
| 用户信息 | ~200B | 20MB | 200MB | 2GB |
| 频道信息 | ~300B | 30MB | 300MB | 3GB |
| 最近会话 | ~150B | 15MB | 150MB | 1.5GB |

**实际占用**会因为以下因素变化：
- Pebble 的压缩率（通常 30-50%）
- WAL 日志大小
- 索引开销
- MANIFEST 文件大小

**建议**：
- 预留磁盘空间至少为预估数据量的 **2-3 倍**
- 定期监控磁盘使用情况
- 考虑使用 SSD 提升性能

---

## 9. 数据备份与迁移

### 9.1 数据备份

#### 方法1: 直接复制目录（推荐）

```bash
# 停止 WuKongIM 服务
systemctl stop wukongim

# 备份数据目录
cp -r ./wukongimdata ./wukongimdata_backup_$(date +%Y%m%d)

# 或使用 tar 打包
tar -czf wukongim_backup_$(date +%Y%m%d).tar.gz ./wukongimdata

# 启动服务
systemctl start wukongim
```

#### 方法2: 使用 rsync 增量备份

```bash
# 增量备份到远程服务器
rsync -avz --delete ./wukongimdata/ user@backup-server:/backup/wukongim/

# 增量备份到本地目录
rsync -avz --delete ./wukongimdata/ /backup/wukongim/
```

#### 方法3: 使用文件系统快照

```bash
# LVM 快照
lvcreate -L 10G -s -n wukongim_snap /dev/vg0/wukongim_lv

# ZFS 快照
zfs snapshot tank/wukongim@backup_$(date +%Y%m%d)

# Btrfs 快照
btrfs subvolume snapshot ./wukongimdata ./wukongimdata_snap
```

### 9.2 自动备份脚本

```bash
#!/bin/bash
# backup_wukongim.sh

BACKUP_DIR="/backup/wukongim"
DATA_DIR="./wukongimdata"
RETENTION_DAYS=7

# 创建备份目录
mkdir -p $BACKUP_DIR

# 备份文件名
BACKUP_FILE="wukongim_backup_$(date +%Y%m%d_%H%M%S).tar.gz"

# 打包备份
tar -czf $BACKUP_DIR/$BACKUP_FILE $DATA_DIR

# 删除7天前的备份
find $BACKUP_DIR -name "wukongim_backup_*.tar.gz" -mtime +$RETENTION_DAYS -delete

echo "Backup completed: $BACKUP_FILE"
```

```bash
# 添加到 crontab，每天凌晨2点备份
0 2 * * * /path/to/backup_wukongim.sh
```

### 9.3 数据迁移

#### 场景1: 迁移到新服务器

```bash
# 在旧服务器上
# 1. 停止服务
systemctl stop wukongim

# 2. 打包数据
tar -czf wukongim_data.tar.gz ./wukongimdata

# 3. 传输到新服务器
scp wukongim_data.tar.gz user@new-server:/opt/wukongim/

# 在新服务器上
# 4. 解压数据
cd /opt/wukongim
tar -xzf wukongim_data.tar.gz

# 5. 修改配置
vim config/wk.yaml
# rootDir: "/opt/wukongim/wukongimdata"

# 6. 启动服务
systemctl start wukongim
```

#### 场景2: 更换存储路径

```bash
# 1. 停止服务
systemctl stop wukongim

# 2. 移动数据目录
mv ./wukongimdata /new/path/wukongimdata

# 3. 修改配置
vim config/wk.yaml
# rootDir: "/new/path/wukongimdata"

# 4. 启动服务
systemctl start wukongim
```


### 9.4 数据恢复

```bash
# 1. 停止服务
systemctl stop wukongim

# 2. 删除旧数据（可选，建议先备份）
mv ./wukongimdata ./wukongimdata_old

# 3. 恢复备份数据
tar -xzf wukongim_backup_20231224.tar.gz

# 4. 启动服务
systemctl start wukongim

# 5. 验证数据
# 检查日志，确认服务正常启动
tail -f logs/wukongim.log
```

### 9.5 集群模式备份

在集群模式下，数据会自动在多个节点之间备份：

```yaml
# config/wk.yaml
cluster:
  nodeId: 1001
  slotReplicaCount: 3      # 槽位副本数量（默认3）
  channelReplicaCount: 3   # 频道副本数量（默认3）
```

**优势**：
- ✅ 自动数据备份到多个节点
- ✅ 节点故障自动转移
- ✅ 无需手动备份（但仍建议定期备份）

---

## 10. 常见问题

### 10.1 能否使用 MySQL 替代 Pebble？

**❌ 不能直接替代**

原因：
- WuKongIM 的存储层深度集成了 Pebble
- 针对 IM 场景做了大量优化
- MySQL 不适合高并发写入场景

**✅ 替代方案**：使用 Datasource 和 Webhook 机制

```yaml
# 通过 Datasource 从你的 MySQL 获取数据
datasource:
  addr: "http://your-api.com/datasource"
  channelInfoOn: true

# 通过 Webhook 将消息推送到你的系统
webhook:
  httpAddr: "http://your-api.com/webhook"
```

### 10.2 数据库文件可以手动编辑吗？

**❌ 不能**

- Pebble 数据库文件是二进制格式
- 手动修改会导致数据损坏
- 只能通过 WuKongIM 的 API 操作数据

### 10.3 如何清空所有数据？

```bash
# ⚠️ 警告：此操作不可恢复！

# 方法1: 删除数据目录
systemctl stop wukongim
rm -rf ./wukongimdata
systemctl start wukongim

# 方法2: 只删除数据库
systemctl stop wukongim
rm -rf ./wukongimdata/wukongimdb
systemctl start wukongim
```

### 10.4 数据库文件越来越大怎么办？

Pebble 会自动进行压缩（Compaction），但你也可以：

1. **监控磁盘使用**：
```bash
du -sh ./wukongimdata
```

2. **检查是否有大量过期消息**：
通过 API 删除过期消息

3. **考虑使用集群模式**：
数据分散到多个节点

4. **定期归档历史数据**：
将旧数据导出到其他存储

### 10.5 如何查看数据库状态？

WuKongIM 提供了监控接口：

```bash
# 访问监控页面
http://your-server:5300/web

# 或通过 API 查询
curl http://your-server:5001/varz
```

监控指标包括：
- 消息总数
- 用户总数
- 频道总数
- 磁盘使用情况
- 数据库性能指标


### 10.6 分片数量可以修改吗？

**❌ 不能修改**

```go
// 分片数量在初始化时设置，之后不能修改
ShardNum: 8  // 默认8个分片
```

**原因**：
- 数据分片基于哈希取模
- 修改分片数会导致数据无法正确读取
- 如需修改，只能重新导入数据

**建议**：
- 初始化时根据预期规模设置合适的分片数
- 小规模：4-8 个分片
- 中等规模：8-16 个分片
- 大规模：16-32 个分片

### 10.7 如何监控存储性能？

#### 1. 通过 Prometheus 监控

WuKongIM 内置了 Prometheus 指标：

```yaml
# config/wk.yaml
trace:
  prometheusApiUrl: "http://prometheus:9090"
```

**关键指标**：
- `wukongim_db_message_append_total`: 消息写入总数
- `wukongim_db_message_get_total`: 消息读取总数
- `wukongim_db_compact_count`: 压缩次数
- `wukongim_db_wal_bytes`: WAL 日志大小
- `wukongim_db_disk_space_usage`: 磁盘使用量

#### 2. 通过日志监控

```bash
# 查看慢查询日志
tail -f logs/wukongim.log | grep "cost"

# 示例输出：
# appendMessages done cost=1.2s channelId=group123 msgCount=100
```

#### 3. 通过系统工具

```bash
# 监控磁盘 I/O
iostat -x 1

# 监控磁盘使用
df -h

# 监控进程资源
top -p $(pgrep wukongim)
```

### 10.8 数据安全性如何保证？

#### 1. 数据持久性

- ✅ **WAL 日志**: 写入前先写 WAL，保证崩溃恢复
- ✅ **同步写入**: 关键操作使用同步写入
- ✅ **数据校验**: Pebble 内置数据校验机制

#### 2. 数据备份

- ✅ **定期备份**: 建议每天备份
- ✅ **多地备份**: 备份到多个位置
- ✅ **集群模式**: 数据自动多副本

#### 3. 数据加密

```yaml
# WuKongIM 支持通信加密
wssConfig:
  certFile: "/path/to/cert.pem"
  keyFile: "/path/to/key.pem"
```

**建议**：
- 使用 HTTPS/WSS 加密通信
- 敏感数据在应用层加密
- 定期更新证书

### 10.9 如何优化存储性能？

#### 1. 使用 SSD

```bash
# SSD 比 HDD 快 10-100 倍
# 建议使用 NVMe SSD
```

#### 2. 调整配置

```go
// 增加内存表大小（适合写入密集场景）
MemTableSize: 32 * 1024 * 1024  // 32MB

// 增加分片数（适合高并发场景）
ShardNum: 16
```

#### 3. 系统优化

```bash
# 增加文件描述符限制
ulimit -n 65535

# 禁用 swap
swapoff -a

# 使用 deadline 或 noop I/O 调度器
echo deadline > /sys/block/sda/queue/scheduler
```

#### 4. 定期维护

- 监控磁盘使用情况
- 清理过期数据
- 定期重启服务（可选）


### 10.10 数据存储总结对比

| 特性 | WuKongIM (Pebble) | MySQL | Redis | MongoDB |
|------|------------------|-------|-------|---------|
| 存储类型 | KV 存储 | 关系型 | 内存KV | 文档型 |
| 写入性能 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| 读取性能 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| 持久化 | ✅ 是 | ✅ 是 | ⚠️ 可选 | ✅ 是 |
| 部署复杂度 | ⭐ 简单 | ⭐⭐⭐ 中等 | ⭐⭐ 简单 | ⭐⭐⭐ 中等 |
| 运维成本 | ⭐ 低 | ⭐⭐⭐ 高 | ⭐⭐ 中等 | ⭐⭐⭐ 高 |
| 适合场景 | IM消息存储 | 业务数据 | 缓存 | 日志/文档 |

---

## 附录

### A. 配置文件完整示例

```yaml
# config/wk.yaml

# 基础配置
mode: "release"                    # 运行模式: debug/release/bench
rootDir: "./wukongimdata"          # 数据存储根目录
addr: "tcp://0.0.0.0:5100"        # TCP监听地址
httpAddr: "0.0.0.0:5001"          # HTTP API监听地址
wsAddr: "ws://0.0.0.0:5200"       # WebSocket监听地址

# 外网配置
external:
  ip: "your-public-ip"            # 外网IP

# 日志配置
logger:
  level: 2                         # 日志级别: 1=debug 2=info 3=warn 4=error
  dir: "./logs"                    # 日志目录

# 管理后台
manager:
  on: true                         # 是否开启
  addr: "0.0.0.0:5300"            # 监听地址

# Demo
demo:
  on: true                         # 是否开启
  addr: "0.0.0.0:5172"            # 监听地址

# 最近会话
conversation:
  on: true                         # 是否开启
  cacheExpire: 1d                 # 缓存过期时间
  syncInterval: 5m                # 保存间隔
  userMaxCount: 1000              # 用户最大会话数

# Webhook配置
webhook:
  httpAddr: "http://your-api.com/webhook"
  msgNotifyEventPushInterval: 500ms
  msgNotifyEventRetryMaxCount: 5
  msgNotifyEventCountPerPush: 100

# Datasource配置
datasource:
  addr: "http://your-api.com/datasource"
  channelInfoOn: true

# 认证配置
auth:
  on: true
  kind: 'jwt'
  users:
    - "admin:password:*"

# JWT配置
jwt:
  secret: "your-secret-key"       # JWT密钥（必须修改）
  expire: 30d                     # 过期时间

# 集群配置（可选）
cluster:
  nodeId: 1001
  addr: "tcp://0.0.0.0:11110"
  slotCount: 64
  slotReplicaCount: 3
  channelReplicaCount: 3
  initNodes:
    - "1001@192.168.1.10:11110"
    - "1002@192.168.1.11:11110"
    - "1003@192.168.1.12:11110"
```

### B. 相关资源

- **官方网站**: https://githubim.com
- **GitHub**: https://github.com/WuKongIM/WuKongIM
- **文档**: https://githubim.com
- **架构文档**: https://deepwiki.com/WuKongIM/WuKongIM
- **Pebble 文档**: https://github.com/cockroachdb/pebble

### C. 技术支持

- **GitHub Issues**: https://github.com/WuKongIM/WuKongIM/issues
- **微信**: wukongimgo（备注进群）

---

## 总结

### 核心要点

1. ✅ **`./wukongimdata` 就是 Pebble KV 数据库的数据目录**
2. ✅ **WuKongIM 存储消息内容，但不存储文件本身**
3. ✅ **文件（图片、视频）需要上传到 OSS，只存储 URL**
4. ✅ **数据永久存储，支持消息漫游**
5. ✅ **使用分片机制提升并发性能**
6. ✅ **定期备份数据，保证数据安全**
7. ✅ **通过 Webhook 和 Datasource 对接业务系统**

### 推荐架构

```
客户端
  ├─ 文件上传 → 阿里云OSS
  └─ 消息发送 → WuKongIM (Pebble存储)
                    ↓
                 Webhook
                    ↓
              业务系统 (MySQL)
```

---

**文档版本**: v1.0  
**更新日期**: 2024-12-24  
**适用版本**: WuKongIM 2.x

