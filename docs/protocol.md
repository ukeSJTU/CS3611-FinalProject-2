# 分布式即时聊天系统 - 消息协议

**版本:** 1.0
**日期:** 2025-05-15
**格式:** JSON

## 1. 基础结构 (Base Structure)

所有客户端与服务器之间的通信都遵循以下基本 JSON 结构。

### 1.1. 通用消息格式 (General Message Format)

```json
{
    "type": "MESSAGE_TYPE_STRING",
    "timestamp": "YYYY-MM-DDTHH:MM:SS.sssZ", // ISO 8601 UTC 时间戳
    "payload": {
        // 特定于 MESSAGE_TYPE 的数据对象
    },
    "token": "session_token_string_or_null", // 用户登录后分配的会话令牌，用于后续请求的认证
    "request_id": "unique_request_id_string_or_null" // [可选] 客户端用于关联请求与响应，建议使用UUID
}
```

-   `type`: 字符串，消息的类型，用于区分不同的操作。
-   `timestamp`: 字符串，消息发送的 UTC 时间，ISO 8601 格式 (例如: `2025-05-15T12:30:05.123Z`)。
-   `payload`: 对象，包含该消息类型特定的数据。如果某个请求或响应不需要额外数据，`payload`可以是一个空对象 `{}`。
-   `token`: 字符串或 null。对于需要认证的请求，客户端必须包含登录时获取的会话令牌。对于登录、注册等初始请求，此字段为 null。
-   `request_id`: 字符串或 null。客户端可以生成一个唯一 ID 来跟踪请求及其对应的响应，特别是在异步通信中。服务器应在对应的响应中回传此 ID。

### 1.2. 通用错误响应 (General Error Response)

当请求处理失败或发生错误时，服务器将返回此格式的响应。

```json
{
    "type": "ERROR_RESPONSE",
    "timestamp": "YYYY-MM-DDTHH:MM:SS.sssZ",
    "payload": {
        "original_request_type": "MESSAGE_TYPE_OF_FAILED_REQUEST_STRING_OR_NULL", // 原始请求的类型
        "error_code": "ERROR_CODE_STRING", // 预定义的错误码 (例如: "INVALID_PAYLOAD", "AUTH_FAILED", "RESOURCE_NOT_FOUND", "FEATURE_NOT_SUPPORTED", "INTERNAL_SERVER_ERROR")
        "message": "详细的错误描述信息" // 人类可读的错误信息
    },
    "token": null, // 或者原始请求的 token
    "request_id": "original_request_id_if_any" // 从原始请求中回传的 request_id
}
```

---

## 2. 用户认证与管理 (Authentication & User Management)

### 2.1. 用户注册 (User Registration)

-   **`REGISTER_REQUEST` (Client -\> Server)**
    ```json
    {
        "type": "REGISTER_REQUEST",
        "timestamp": "...",
        "payload": {
            "username": "desired_username_string", // 用户名，服务器需校验唯一性
            "password_hash": "hashed_password_string" // 客户端哈希后的密码 (例如 SHA-256)
        },
        "token": null,
        "request_id": "client_generated_uuid"
    }
    ```
-   **`REGISTER_RESPONSE` (Server -\> Client)**
    ```json
    {
        "type": "REGISTER_RESPONSE",
        "timestamp": "...",
        "payload": {
            "success": true, // 或者 false
            "message": "注册成功" // 或者 "用户名已存在", "密码不符合要求" 等
            // "user_id": "server_generated_user_id_string" // 注册成功时可选返回用户ID
        },
        "token": null,
        "request_id": "echoed_client_request_id"
    }
    ```

### 2.2. 用户登录 (User Login)

-   **`LOGIN_REQUEST` (Client -\> Server)**
    ```json
    {
        "type": "LOGIN_REQUEST",
        "timestamp": "...",
        "payload": {
            "username": "username_string",
            "password_hash": "hashed_password_string"
        },
        "token": null,
        "request_id": "client_generated_uuid"
    }
    ```
-   **`LOGIN_RESPONSE` (Server -\> Client)**
    ```json
    {
        "type": "LOGIN_RESPONSE",
        "timestamp": "...",
        "payload": {
            "success": true, // 或者 false
            "message": "登录成功", // 或者 "用户名或密码错误"
            "user_id": "user_id_string_if_success_null_otherwise",
            "username": "username_string_if_success_null_otherwise",
            "session_token": "generated_session_token_string_if_success_null_otherwise" // 登录成功后用于后续通信
        },
        "token": null, // 响应本身不携带token，而是payload中包含新的session_token
        "request_id": "echoed_client_request_id"
    }
    ```

### 2.3. 用户登出 (User Logout)

-   **`LOGOUT_REQUEST` (Client -\> Server)**
    ```json
    {
        "type": "LOGOUT_REQUEST",
        "timestamp": "...",
        "payload": {}, // 无特定payload
        "token": "active_session_token", // 需要提供当前会话的token
        "request_id": "client_generated_uuid"
    }
    ```
-   **`LOGOUT_RESPONSE` (Server -\> Client)**
    ```json
    {
        "type": "LOGOUT_RESPONSE",
        "timestamp": "...",
        "payload": {
            "success": true, // 或者 false (例如token无效)
            "message": "登出成功"
        },
        "token": null,
        "request_id": "echoed_client_request_id"
    }
    ```

### 2.4. 功能宣告 (Feature Announcement) - 可选

-   **`CLIENT_FEATURES_ANNOUNCE_REQUEST` (Client -\> Server)** (登录后发送)
    ```json
    {
        "type": "CLIENT_FEATURES_ANNOUNCE_REQUEST",
        "timestamp": "...",
        "payload": {
            "supported_extensions": [
                "FILE_TRANSFER_V1",
                "E2EE_V1_BASIC_PROFILE",
                "MESSAGE_RECALL_V1"
            ] // 客户端支持的扩展功能列表
        },
        "token": "active_session_token",
        "request_id": "client_generated_uuid"
    }
    ```
-   **`SERVER_FEATURES_ANNOUNCE_RESPONSE` (Server -\> Client)** (作为 LOGIN_RESPONSE 的一部分或单独发送)
    ```json
    {
        "type": "SERVER_FEATURES_ANNOUNCE_RESPONSE", // 或集成在 LOGIN_RESPONSE payload 中
        "timestamp": "...",
        "payload": {
            "supported_extensions": [
                "FILE_TRANSFER_V1",
                "MESSAGE_RECALL_V1",
                "VOICE_VIDEO_V1_SIGNALING"
            ] // 服务器支持的扩展功能列表
        },
        "token": "active_session_token_if_separate_message_else_null",
        "request_id": "echoed_client_request_id_if_response_else_null"
    }
    ```

---

## 3. 核心消息传递 (Core Messaging)

### 3.1. 发送私聊消息 (Send Private Message)

-   **`SEND_PRIVATE_MESSAGE_REQUEST` (Client -\> Server)**
    ```json
    {
        "type": "SEND_PRIVATE_MESSAGE_REQUEST",
        "timestamp": "...", // 客户端发送时间
        "payload": {
            "recipient_user_id": "target_user_id_string",
            "client_message_id": "client_generated_unique_id_for_this_message", // 客户端生成的消息ID，用于去重和ACK
            "content_type": "TEXT", // "TEXT", "ENCRYPTED_TEXT_AES_GCM", "IMAGE_POINTER", "FILE_POINTER" 等
            "content": "你好，这是一条消息。" // 如果是E2EE，这里是加密后的密文；如果是文件，这里可能是文件元数据引用
        },
        "token": "active_session_token",
        "request_id": "client_generated_uuid"
    }
    ```

### 3.2. 接收私聊消息 (Receive Private Message)

-   **`PRIVATE_MESSAGE_NOTIFICATION` (Server -\> Client)**
    ```json
    {
        "type": "PRIVATE_MESSAGE_NOTIFICATION",
        "timestamp": "...", // 服务器转发时间或原始发送时间
        "payload": {
            "server_message_id": "server_generated_unique_id_for_this_message",
            "client_message_id": "original_client_message_id_if_available", // 发送方客户端生成的消息ID
            "sender_user_id": "sender_user_id_string",
            "sender_username": "sender_username_string",
            "content_type": "TEXT", // 与发送时一致
            "content": "你好，这是一条消息。",
            "sent_at_timestamp": "original_client_timestamp_string" // 原始发送时间
        },
        "token": null, // 服务器推送的通知通常不包含token
        "request_id": null
    }
    ```

### 3.3. 发送群聊消息 (Send Group Message)

-   **`SEND_GROUP_MESSAGE_REQUEST` (Client -\> Server)**
    ```json
    {
        "type": "SEND_GROUP_MESSAGE_REQUEST",
        "timestamp": "...",
        "payload": {
            "group_id": "target_group_id_string",
            "client_message_id": "client_generated_unique_id_for_this_message",
            "content_type": "TEXT",
            "content": "大家好！"
        },
        "token": "active_session_token",
        "request_id": "client_generated_uuid"
    }
    ```

### 3.4. 接收群聊消息 (Receive Group Message)

-   **`GROUP_MESSAGE_NOTIFICATION` (Server -\> Client)**
    ```json
    {
        "type": "GROUP_MESSAGE_NOTIFICATION",
        "timestamp": "...",
        "payload": {
            "server_message_id": "server_generated_unique_id_for_this_message",
            "client_message_id": "original_client_message_id_if_available",
            "group_id": "group_id_string",
            "sender_user_id": "sender_user_id_string",
            "sender_username": "sender_username_string",
            "content_type": "TEXT",
            "content": "大家好！",
            "sent_at_timestamp": "original_client_timestamp_string"
        },
        "token": null,
        "request_id": null
    }
    ```

### 3.5. 消息回执 (Message Acknowledgement) - 可选

-   **`MESSAGE_ACK_NOTIFICATION` (Server -\> Sending Client)** (告知发送方消息已送达服务器/对方客户端)
    ```json
    {
        "type": "MESSAGE_ACK_NOTIFICATION",
        "timestamp": "...",
        "payload": {
            "client_message_id": "client_message_id_being_acked", // 客户端发送消息时使用的ID
            "server_message_id": "server_message_id_assigned_by_server", // 服务器为此消息分配的ID
            "chat_type": "PRIVATE", // "PRIVATE" or "GROUP"
            "recipient_id": "user_id_or_group_id_string", // 接收者ID
            "status": "SENT_TO_SERVER" // 或 "DELIVERED_TO_RECIPIENT_CLIENT", "READ_BY_RECIPIENT" (已读状态较复杂)
        },
        "token": null,
        "request_id": null
    }
    ```

---

## 4. 群组管理 (Group Management)

### 4.1. 创建群组 (Create Group)

-   **`CREATE_GROUP_REQUEST` (Client -\> Server)**
    ```json
    {
        "type": "CREATE_GROUP_REQUEST",
        "timestamp": "...",
        "payload": {
            "group_name": "desired_group_name_string",
            "initial_member_ids": ["user_id_1", "user_id_2"] // [可选] 初始成员列表 (不含创建者)
        },
        "token": "active_session_token",
        "request_id": "client_generated_uuid"
    }
    ```
-   **`CREATE_GROUP_RESPONSE` (Server -\> Client)**
    ```json
    {
        "type": "CREATE_GROUP_RESPONSE",
        "timestamp": "...",
        "payload": {
            "success": true,
            "message": "群组创建成功",
            "group_id": "server_generated_group_id_string",
            "group_name": "group_name_string"
            // "members": [{"user_id": "...", "username": "...", "role": "OWNER"}] // 初始成员列表
        },
        "token": null,
        "request_id": "echoed_client_request_id"
    }
    ```

### 4.2. 加入群组 (Join Group)

-   **`JOIN_GROUP_REQUEST` (Client -\> Server)**
    ```json
    {
        "type": "JOIN_GROUP_REQUEST",
        "timestamp": "...",
        "payload": {
            "group_id": "group_id_to_join_string"
            // "invite_code": "optional_invite_code_string" // 如果群组是私有的且需要邀请码
        },
        "token": "active_session_token",
        "request_id": "client_generated_uuid"
    }
    ```
-   **`JOIN_GROUP_RESPONSE` (Server -\> Client)**
    ```json
    {
        "type": "JOIN_GROUP_RESPONSE",
        "timestamp": "...",
        "payload": {
            "success": true,
            "message": "成功加入群组",
            "group_id": "group_id_string",
            "group_name": "group_name_string"
            // "members": [{"user_id": "...", "username": "...", "online_status": "ONLINE|OFFLINE"}] // 更新后的成员列表
        },
        "token": null,
        "request_id": "echoed_client_request_id"
    }
    ```

### 4.3. 退出群组 (Leave Group)

-   **`LEAVE_GROUP_REQUEST` (Client -\> Server)**
    ```json
    {
        "type": "LEAVE_GROUP_REQUEST",
        "timestamp": "...",
        "payload": {
            "group_id": "group_id_to_leave_string"
        },
        "token": "active_session_token",
        "request_id": "client_generated_uuid"
    }
    ```
-   **`LEAVE_GROUP_RESPONSE` (Server -\> Client)**
    ```json
    {
        "type": "LEAVE_GROUP_RESPONSE",
        "timestamp": "...",
        "payload": {
            "success": true,
            "message": "已成功退出群组",
            "group_id": "group_id_string"
        },
        "token": null,
        "request_id": "echoed_client_request_id"
    }
    ```

### 4.4. 获取用户所在的群组列表 (List User Groups)

-   **`LIST_USER_GROUPS_REQUEST` (Client -\> Server)**
    ```json
    {
        "type": "LIST_USER_GROUPS_REQUEST",
        "timestamp": "...",
        "payload": {}, // 无特定payload
        "token": "active_session_token",
        "request_id": "client_generated_uuid"
    }
    ```
-   **`LIST_USER_GROUPS_RESPONSE` (Server -\> Client)**
    ```json
    {
        "type": "LIST_USER_GROUPS_RESPONSE",
        "timestamp": "...",
        "payload": {
            "groups": [
                {
                    "group_id": "group_id_1",
                    "group_name": "Group A",
                    "unread_count": 0
                }, // unread_count 可选
                {
                    "group_id": "group_id_2",
                    "group_name": "Project B",
                    "unread_count": 5
                }
            ]
        },
        "token": null,
        "request_id": "echoed_client_request_id"
    }
    ```

### 4.5. 获取群组详细信息 (Get Group Info)

-   **`GET_GROUP_INFO_REQUEST` (Client -\> Server)**
    ```json
    {
        "type": "GET_GROUP_INFO_REQUEST",
        "timestamp": "...",
        "payload": {
            "group_id": "group_id_string"
        },
        "token": "active_session_token",
        "request_id": "client_generated_uuid"
    }
    ```
-   **`GET_GROUP_INFO_RESPONSE` (Server -\> Client)**
    ```json
    {
        "type": "GET_GROUP_INFO_RESPONSE",
        "timestamp": "...",
        "payload": {
            "group_id": "group_id_string",
            "group_name": "group_name_string",
            "owner_user_id": "owner_user_id_string",
            "created_at": "timestamp_string",
            "members": [
                {
                    "user_id": "user_id_1",
                    "username": "Alice",
                    "online_status": "ONLINE",
                    "role": "OWNER"
                },
                {
                    "user_id": "user_id_2",
                    "username": "Bob",
                    "online_status": "OFFLINE",
                    "role": "MEMBER"
                }
            ]
        },
        "token": null,
        "request_id": "echoed_client_request_id"
    }
    ```

### 4.6. 群组事件通知 (Group Event Notification)

-   **`GROUP_EVENT_NOTIFICATION` (Server -\> Client Members)** (例如用户加入/退出群组)
    ```json
    {
        "type": "GROUP_EVENT_NOTIFICATION",
        "timestamp": "...",
        "payload": {
            "group_id": "group_id_string",
            "event_type": "USER_JOINED", // "USER_LEFT", "GROUP_RENAMED", "USER_KICKED", "ROLE_CHANGED"
            "actor_user_id": "user_id_who_caused_event", // 执行操作的用户
            "actor_username": "username_of_actor",
            "target_user_id": "user_id_affected_if_any", // 被操作的用户 (例如被踢出)
            "target_username": "username_of_target_if_any",
            "details": {
                // 特定于 event_type 的额外信息
                // for USER_JOINED: {"joined_user_id": "...", "joined_username": "..."}
                // for GROUP_RENAMED: {"old_name": "...", "new_name": "..."}
            }
        },
        "token": null,
        "request_id": null
    }
    ```

---

## 5. 用户在线状态 (User Online Status)

### 5.1. 用户状态更新通知 (User Status Update Notification)

-   **`USER_STATUS_UPDATE_NOTIFICATION` (Server -\> Client)** (服务器主动推送给相关客户端，例如好友或同一群组成员)
    ```json
    {
        "type": "USER_STATUS_UPDATE_NOTIFICATION",
        "timestamp": "...",
        "payload": {
            "user_id": "user_id_whose_status_changed",
            "username": "username_string", // 可选
            "online_status": "ONLINE" // 或者 "OFFLINE"
        },
        "token": null,
        "request_id": null
    }
    ```
    _(注: 客户端登录/登出时，服务器会自动更新其状态并通知相关客户端。)_

---

## 6. 心跳机制 (Heartbeat Mechanism)

### 6.1. 客户端心跳 (Client Heartbeat Ping)

-   **`HEARTBEAT_PING_REQUEST` (Client -\> Server)** (客户端定期发送)
    ```json
    {
        "type": "HEARTBEAT_PING_REQUEST",
        "timestamp": "...",
        "payload": {
            // "client_load": 0.5 // 可选，客户端可以上报一些状态信息
        },
        "token": "active_session_token",
        "request_id": "client_generated_uuid_optional" // 通常心跳不需要 request_id
    }
    ```

### 6.2. 服务器心跳回应 (Server Heartbeat Pong) - 可选

-   **`HEARTBEAT_PONG_RESPONSE` (Server -\> Client)** (服务器可选回复)
    ```json
    {
        "type": "HEARTBEAT_PONG_RESPONSE",
        "timestamp": "...",
        "payload": {
            // "server_time": "YYYY-MM-DDTHH:MM:SS.sssZ" // 可选，用于客户端时间同步
        },
        "token": null,
        "request_id": "echoed_client_request_id_if_present"
    }
    ```
    _(服务器记录每个客户端的最后心跳时间。若超时未收到，则认为客户端离线。)_

---

## 7. 消息历史记录 (Message History)

### 7.1. 拉取消息历史 (Fetch Message History)

-   **`Workspace_HISTORY_REQUEST` (Client -\> Server)**
    ```json
    {
        "type": "FETCH_HISTORY_REQUEST",
        "timestamp": "...",
        "payload": {
            "chat_type": "PRIVATE", // "PRIVATE" or "GROUP"
            "chat_id": "user_id_or_group_id_string", // 对方用户ID或群组ID
            "before_message_id": "message_id_string_or_null", // [分页] 拉取此消息之前的记录，为null则从最新开始
            "before_timestamp": "timestamp_string_or_null", // [分页] 或者用时间戳
            "limit": 20 // 期望拉取的数量
        },
        "token": "active_session_token",
        "request_id": "client_generated_uuid"
    }
    ```
-   **`Workspace_HISTORY_RESPONSE` (Server -\> Client)**
    ```json
    {
        "type": "FETCH_HISTORY_RESPONSE",
        "timestamp": "...",
        "payload": {
            "chat_type": "PRIVATE",
            "chat_id": "user_id_or_group_id_string",
            "messages": [
                // 按时间倒序排列 (新的在前)
                {
                    "server_message_id": "msg_id_1",
                    "client_message_id": "client_msg_id_1_optional",
                    "sender_user_id": "user_id_string",
                    "sender_username": "username_string",
                    "content_type": "TEXT",
                    "content": "历史消息1",
                    "sent_at_timestamp": "message_timestamp_string"
                    // "recalled": true // 如果消息已被撤回
                }
                // ... more messages
            ],
            "has_more_older": true // 指示是否还有更早的历史记录可供拉取
        },
        "token": null,
        "request_id": "echoed_client_request_id"
    }
    ```

---

## 8. 扩展功能 (Extended Features)

### 8.1. 文件传输 (File Transfer - with Resume)

#### 8.1.1. 发起文件传输提议 (Offer File Transfer)

-   **`FILE_TRANSFER_OFFER_REQUEST` (Sender Client -\> Server)**
    ```json
    {
        "type": "FILE_TRANSFER_OFFER_REQUEST",
        "timestamp": "...",
        "payload": {
            "recipient_user_id": "target_user_id_string",
            "file_name": "example.zip",
            "file_size": 10485760, // Bytes
            "file_type": "application/zip", // MIME Type
            "file_hash_sha256": "hex_encoded_sha256_hash_of_file_optional" // 用于完整性校验
        },
        "token": "active_session_token",
        "request_id": "client_generated_uuid_for_offer"
    }
    ```
-   **`FILE_TRANSFER_OFFER_NOTIFICATION` (Server -\> Receiver Client)**
    ```json
    {
        "type": "FILE_TRANSFER_OFFER_NOTIFICATION",
        "timestamp": "...",
        "payload": {
            "transfer_id": "server_generated_unique_transfer_id", // 用于后续追踪此传输会话
            "sender_user_id": "sender_user_id_string",
            "sender_username": "sender_username_string",
            "file_name": "example.zip",
            "file_size": 10485760,
            "file_type": "application/zip",
            "file_hash_sha256": "hex_encoded_sha256_hash_of_file_optional"
        },
        "token": null,
        "request_id": null
    }
    ```

#### 8.1.2. 响应提议 (Respond to Offer)

-   **`FILE_TRANSFER_RESPONSE_REQUEST` (Receiver Client -\> Server)**
    ```json
    {
        "type": "FILE_TRANSFER_RESPONSE_REQUEST",
        "timestamp": "...",
        "payload": {
            "transfer_id": "transfer_id_from_notification",
            "accepted": true //或者 false
            // "reason_if_rejected": "string_optional"
        },
        "token": "active_session_token",
        "request_id": "client_generated_uuid"
    }
    ```
-   **`FILE_TRANSFER_STATUS_NOTIFICATION` (Server -\> Sender Client & Receiver Client if accepted)**
    ```json
    {
        "type": "FILE_TRANSFER_STATUS_NOTIFICATION",
        "timestamp": "...",
        "payload": {
            "transfer_id": "transfer_id_string",
            "status": "ACCEPTED", // "REJECTED", "READY_TO_UPLOAD", "READY_TO_DOWNLOAD", "TRANSFERRING", "COMPLETED", "FAILED", "CANCELLED"
            "message": "接收方已接受文件传输，请准备上传。" // 或其他状态信息
            // "upload_url": "temp_url_if_server_proxies_upload_optional", // 如果服务器充当代理
            // "download_url": "temp_url_if_server_proxies_download_optional",
            // "direct_connection_info": { ... } // 如果尝试P2P
        },
        "token": null,
        "request_id": null // 或与 FILE_TRANSFER_RESPONSE_REQUEST 关联
    }
    ```

#### 8.1.3. 文件块传输 (File Chunk Transfer)

_(这部分通常不直接封装在 JSON 中，而是通过 HTTP PUT/POST 或单独的 TCP/UDP 流进行。以下仅为信令示意。实际数据块通过其他方式传输)_

-   **`FILE_CHUNK_UPLOAD_REQUEST` (Sender Client -\> Server or P2P Receiver)** (信令，非实际数据)
    ```json
    {
        "type": "FILE_CHUNK_UPLOAD_REQUEST", // 信令
        "timestamp": "...",
        "payload": {
            "transfer_id": "transfer_id_string",
            "chunk_offset": 0, // 块的起始偏移量 (bytes)
            "chunk_size": 65536, // 块的大小 (bytes)
            "chunk_hash_md5": "md5_of_this_chunk_optional" // 块的哈希
            // 实际数据通过 HTTP PUT 或其他流发送
        },
        "token": "active_session_token",
        "request_id": "client_generated_uuid"
    }
    ```
-   **`FILE_CHUNK_ACK_NOTIFICATION` (Server/P2P Receiver -\> Sender Client)**
    ```json
    {
        "type": "FILE_CHUNK_ACK_NOTIFICATION",
        "timestamp": "...",
        "payload": {
            "transfer_id": "transfer_id_string",
            "chunk_offset": 0,
            "received_size": 65536,
            "status": "SUCCESS" // "RETRY_CHUNK", "FAILED"
        },
        "token": null,
        "request_id": null // 或与 UPLOAD_REQUEST 关联
    }
    ```

#### 8.1.4. 传输完成/失败 (Transfer Complete/Fail)

-   **`FILE_TRANSFER_COMPLETE_REQUEST` (Sender/Receiver Client -\> Server)**
    ```json
    {
        "type": "FILE_TRANSFER_COMPLETE_REQUEST", // 发送方上传完毕，或接收方下载完毕
        "timestamp": "...",
        "payload": {
            "transfer_id": "transfer_id_string",
            "status": "COMPLETED_UPLOAD", // "COMPLETED_DOWNLOAD", "FAILED_UPLOAD", "FAILED_DOWNLOAD", "CANCELLED_BY_SENDER", "CANCELLED_BY_RECEIVER"
            "final_file_hash_sha256": "hex_hash_if_completed_successfully_optional" // 用于最终校验
        },
        "token": "active_session_token",
        "request_id": "client_generated_uuid"
    }
    ```
    _(服务器收到后，可更新 FILE_TRANSFER_STATUS_NOTIFICATION 给另一方)_

#### 8.1.5. 断点续传请求 (Resume File Transfer)

-   **`FILE_TRANSFER_RESUME_REQUEST` (Client -\> Server)**
    ```json
    {
        "type": "FILE_TRANSFER_RESUME_REQUEST",
        "timestamp": "...",
        "payload": {
            "transfer_id": "original_transfer_id_string",
            "last_successful_offset": 5242880 // 已成功发送/接收的字节偏移量
        },
        "token": "active_session_token",
        "request_id": "client_generated_uuid"
    }
    ```
-   **`FILE_TRANSFER_RESUME_RESPONSE` (Server -\> Client)**
    ```json
    {
        "type": "FILE_TRANSFER_RESUME_RESPONSE",
        "timestamp": "...",
        "payload": {
            "transfer_id": "transfer_id_string",
            "can_resume": true,
            "next_expected_offset": 5242880, // 服务器期望的下一个字节偏移量
            "message": "可以从指定偏移量继续传输"
            // "new_upload_url_if_needed": "..."
        },
        "token": null,
        "request_id": "echoed_client_request_id"
    }
    ```

### 8.2. 端到端加密通信 (End-to-End Encryption - Signaling Support)

_(E2EE 的核心加密/解密在客户端进行，服务器仅辅助信令或密钥材料的安全传递，如公钥。以下是示例，具体实现依赖选择的 E2EE 协议，如 Signal Protocol 变体。)_

-   **`E2EE_INITIATE_SESSION_REQUEST` (Client A -\> Server -\> Client B)**
    _(用于交换预共享密钥、身份密钥等初始材料)_
    ```json
    {
        "type": "E2EE_INITIATE_SESSION_REQUEST",
        "timestamp": "...",
        "payload": {
            "recipient_user_id": "client_B_user_id",
            "sender_identity_public_key": "base64_encoded_key",
            "sender_prekey_bundle": {
                /* OMEMO/Signal-like prekey bundle */
            }
        },
        "token": "active_session_token",
        "request_id": "client_A_generated_uuid"
    }
    ```
-   **`E2EE_SESSION_DATA_EXCHANGE_MESSAGE` (Client A \<-\> Server \<-\> Client B)**
    _(用于在会话建立过程中或会话中交换加密的密钥材料或控制消息)_
    ```json
    {
        "type": "E2EE_SESSION_DATA_EXCHANGE_MESSAGE",
        "timestamp": "...",
        "payload": {
            "target_user_id": "other_party_user_id",
            "encrypted_e2ee_payload": "base64_encoded_encrypted_data" // 包含密钥或控制信息，由对方客户端的E2EE模块解密
        },
        "token": "active_session_token",
        "request_id": "client_generated_uuid"
    }
    ```
    _(加密后的文本消息通过 `SEND_PRIVATE_MESSAGE_REQUEST` 发送，`content_type` 设为 `ENCRYPTED_TEXT_...`)\_

### 8.3. 语音/视频聊天 (Voice/Video Chat - Signaling via TCP)

_(实际音视频数据通过 UDP 传输，这里定义 TCP 信令)_

#### 8.3.1. 发起呼叫 (Initiate Call)

-   **`MEDIA_CALL_REQUEST` (Caller Client -\> Server)**
    ```json
    {
        "type": "MEDIA_CALL_REQUEST",
        "timestamp": "...",
        "payload": {
            "recipient_user_id": "callee_user_id_string",
            "media_type": "VOICE", // "VOICE" or "VIDEO"
            "sdp_offer": {
                /* SDP offer object or string, for WebRTC-like NAT traversal */
            } // 可选
        },
        "token": "active_session_token",
        "request_id": "caller_generated_uuid"
    }
    ```
-   **`MEDIA_CALL_INVITATION_NOTIFICATION` (Server -\> Callee Client)**
    ```json
    {
        "type": "MEDIA_CALL_INVITATION_NOTIFICATION",
        "timestamp": "...",
        "payload": {
            "call_id": "server_generated_unique_call_id",
            "caller_user_id": "caller_user_id_string",
            "caller_username": "caller_username_string",
            "media_type": "VOICE",
            "sdp_offer": {
                /* from caller */
            } // 可选
        },
        "token": null,
        "request_id": null
    }
    ```

#### 8.3.2. 响应呼叫 (Respond to Call)

-   **`MEDIA_CALL_RESPONSE_REQUEST` (Callee Client -\> Server)**
    ```json
    {
        "type": "MEDIA_CALL_RESPONSE_REQUEST",
        "timestamp": "...",
        "payload": {
            "call_id": "call_id_from_invitation",
            "accepted": true, // false if rejected
            "sdp_answer": {
                /* SDP answer object or string, if accepted and using SDP */
            } // 可选
        },
        "token": "active_session_token",
        "request_id": "callee_generated_uuid"
    }
    ```
-   **`MEDIA_CALL_STATUS_NOTIFICATION` (Server -\> Caller Client & Callee Client if accepted)**
    ```json
    {
      "type": "MEDIA_CALL_STATUS_NOTIFICATION",
      "timestamp": "...",
      "payload": {
        "call_id": "call_id_string",
        "status": "ACCEPTED" , // "REJECTED_BY_USER", "REJECTED_UNAVAILABLE", "CONNECTING", "CONNECTED", "ENDED"
        "message": "对方已接受呼叫" // 或其他状态信息
        "sdp_answer": { /* from callee, if accepted */ } // 可选, 发给主叫方
        // "udp_relay_info": {"ip": "...", "port": "..."} // 如果服务器提供TURN/Relay服务
      },
      "token": null,
      "request_id": null // 或与 MEDIA_CALL_RESPONSE_REQUEST 关联
    }
    ```

#### 8.3.3. ICE 候选交换 (ICE Candidate Exchange - for NAT traversal)

-   **`MEDIA_ICE_CANDIDATE_MESSAGE` (Client -\> Server -\> Other Client in call)**
    ```json
    {
        "type": "MEDIA_ICE_CANDIDATE_MESSAGE",
        "timestamp": "...",
        "payload": {
            "call_id": "call_id_string",
            "target_user_id": "other_party_user_id_in_call",
            "candidate": {
                /* ICE candidate object or string */
            }
        },
        "token": "active_session_token",
        "request_id": "client_generated_uuid_optional"
    }
    ```

#### 8.3.4. 结束呼叫 (End Call)

-   **`MEDIA_CALL_END_REQUEST` (Client in call -\> Server)**
    ```json
    {
        "type": "MEDIA_CALL_END_REQUEST",
        "timestamp": "...",
        "payload": {
            "call_id": "call_id_string",
            "reason": "USER_HUNG_UP" // "CONNECTION_LOST", "NO_ANSWER_TIMEOUT"
        },
        "token": "active_session_token",
        "request_id": "client_generated_uuid"
    }
    ```
    _(服务器收到后，会向通话中的另一方发送 `MEDIA_CALL_STATUS_NOTIFICATION`，状态为 `ENDED`)_

### 8.4. 消息撤回 (Message Recall)

#### 8.4.1. 请求撤回消息 (Request Message Recall)

-   **`RECALL_MESSAGE_REQUEST` (Client -\> Server)**
    ```json
    {
        "type": "RECALL_MESSAGE_REQUEST",
        "timestamp": "...",
        "payload": {
            "server_message_id": "server_message_id_to_recall", // 要撤回的消息的服务器ID
            "client_message_id": "client_message_id_to_recall_optional", // 或客户端ID
            "chat_type": "PRIVATE", // "PRIVATE" or "GROUP"
            "chat_id": "user_id_or_group_id_string" // 撤回消息所在的聊天会话
        },
        "token": "active_session_token",
        "request_id": "client_generated_uuid"
    }
    ```
-   **`RECALL_MESSAGE_RESPONSE` (Server -\> Requesting Client)**
    ```json
    {
        "type": "RECALL_MESSAGE_RESPONSE",
        "timestamp": "...",
        "payload": {
            "success": true, // false 如果超过时限、消息不存在或无权限
            "server_message_id": "server_message_id_string",
            "client_message_id": "client_message_id_string_optional",
            "message": "消息已成功申请撤回" // 或 "撤回失败：超过可撤回时限"
        },
        "token": null,
        "request_id": "echoed_client_request_id"
    }
    ```

#### 8.4.2. 消息已撤回通知 (Message Recalled Notification)

-   **`MESSAGE_RECALLED_NOTIFICATION` (Server -\> Relevant Clients in the chat)**
    ```json
    {
        "type": "MESSAGE_RECALLED_NOTIFICATION",
        "timestamp": "...", // 撤回操作执行的时间
        "payload": {
            "server_message_id": "recalled_server_message_id_string",
            "client_message_id": "recalled_client_message_id_optional",
            "chat_type": "PRIVATE",
            "chat_id": "user_id_or_group_id_string",
            "recaller_user_id": "user_id_who_recalled_the_message",
            "recaller_username": "username_of_recaller",
            "placeholder_text": "用户 [username] 撤回了一条消息" // 客户端应用此文本替换原消息内容
        },
        "token": null,
        "request_id": null
    }
    ```

---

## 9. 附录 (Appendix)

### 9.1. 错误码示例 (Example Error Codes)

-   `INVALID_REQUEST_FORMAT`: 请求 JSON 格式错误。
-   `INVALID_PAYLOAD`: 请求的`payload`内容不符合规范或缺少必要字段。
-   `MISSING_TOKEN`: 需要认证的接口未提供`token`。
-   `INVALID_TOKEN`: `token`无效或已过期。
-   `AUTH_FAILED`: 用户名或密码错误。
-   `USERNAME_TAKEN`: 注册时用户名已存在。
-   `USER_NOT_FOUND`: 操作的用户不存在。
-   `GROUP_NOT_FOUND`: 操作的群组不存在。
-   `NOT_GROUP_MEMBER`: 用户不是该群组成员，无权操作。
-   `ALREADY_GROUP_MEMBER`: 用户已是该群组成员。
-   `PERMISSION_DENIED`: 用户无权执行此操作。
-   `RESOURCE_NOT_FOUND`: 请求的资源未找到 (通用)。
-   `MESSAGE_NOT_FOUND`: 要操作的消息未找到。
-   `RECALL_TIMEOUT`: 消息超过可撤回时限。
-   `FEATURE_NOT_SUPPORTED`: 请求的功能服务器不支持。
-   `TRANSFER_ID_INVALID`: 文件传输 ID 无效。
-   `CALL_ID_INVALID`: 呼叫 ID 无效。
-   `INTERNAL_SERVER_ERROR`: 服务器内部错误。
-   `RATE_LIMIT_EXCEEDED`: 请求频率过高。

### 9.2. ID 生成策略 (ID Generation Strategy)

-   **`user_id`**: 服务器在用户注册时生成，全局唯一 (例如 UUID)。
-   **`group_id`**: 服务器在群组创建时生成，全局唯一 (例如 UUID)。
-   **`session_token`**: 服务器在用户登录时生成，具有时效性 (例如 JWT 或安全随机字符串)。
-   **`client_message_id`**: 客户端发送消息前生成 (例如 UUID)，用于去重和 ACK。
-   **`server_message_id`**: 服务器收到消息后为其分配，用于持久化和历史记录查询 (可以是自增 ID 或 UUID)。
-   **`transfer_id`**, **`call_id`**: 服务器为文件传输会话或音视频呼叫会话生成，全局唯一。
-   **`request_id`**: 客户端为每个请求生成 (例如 UUID)，用于跟踪。
