# AIcarus-Message-Protocol 通信协议文档 v1.4.

## 1. 概述

本文档定义了聊天机器人核心 (Core) 与平台适配器 (Adapter) 之间，基于 AIcarus-Message-Protocol 的通信协议。该协议旨在提供一套标准化的事件结构和交互模式，以实现组件解耦、易于扩展和多平台支持。

所有通信均通过 AIcarus-Message-Protocol 定义的 `Event` 对象进行封装和传输。`event_type` 字段用于明确事件的详细类型，而 `content` 字段则承载该事件的具体数据。

**版本说明:** v1.4.0 - 此版本将通信的基本载体统一为 `Event` 对象。引入了层级化的 `event_type` (如 `message.group.normal`, `notice.conversation.member_increase`, `action.message.send`) 以取代原先的 `interaction_purpose` 和分散的类型定义。事件的具体参数和消息内容统一存放于 `content` 字段中。`ConversationInfo` 作为标准会话描述，`UserInfo` 保持不变，`Seg` 对象作为通用信息单元，用于承载各类事件的具体数据。

## 2. 核心数据结构

### 2.1. `Event` 对象

所有交互的顶层载体。

* **`event_id` (str)**: 事件包装对象的唯一标识符 (例如：由Adapter生成的UUID)，用于追踪和调试。
* **`event_type` (str)**: 描述事件类型的字符串，采用点分层级结构。
    * **前缀规范**:
        * `message.*`: 用户发送的消息 (例如: `message.private.friend`, `message.group.normal`, `message.channel.thread_reply`)。
        * `notice.*`: 平台产生的通知 (例如: `notice.conversation.member_increase`, `notice.user.profile_update`)。
        * `request.*`: 平台产生的、需要机器人响应的请求 (例如: `request.friend.add`, `request.conversation.join_application`)。
        * `action.*`: 由Core发起，指示Adapter执行的动作 (例如: `action.message.send`, `action.conversation.kick_user`)。
        * `action_response.*`: (可选) Adapter对`action.*`的执行结果反馈 (例如: `action_response.success`, `action_response.failure`)。
        * `meta.*`: 关于机器人自身或Adapter状态的元事件 (例如: `meta.lifecycle.connect`, `meta.heartbeat`)。
    * 后续层级根据具体事件类型进一步细化。
* **`time` (float)**: 事件发生的Unix毫秒时间戳。
* **`platform` (str)**: 事件来源或动作目标的平台标识符 (例如: "qq", "discord", "wechat", "core")。
* **`bot_id` (str)**: 机器人自身在该平台上的ID。
* **`user_info` (Optional[`UserInfo`])**: 与事件最直接相关的用户信息 (例如：消息发送者，被操作用户，请求发起者)。详见 2.2节。
* **`conversation_info` (Optional[`ConversationInfo`])**: 事件发生的会话上下文信息 (例如：消息所在的群聊/私聊/频道)。详见 2.3节。
* **`content` (List[`Seg`])**: 事件的具体内容，表现为一个 `Seg` 对象列表。其具体构造和内部 `Seg` 的类型与数据由 `event_type` 决定。
    * 对于 `message.*` 类型事件: `content` 是一个包含构成消息实际内容的 `Seg` 对象列表 (例如：文本、图片、at等多种 `Seg` 组合)。详见 3.1节，其可能还包含与消息本身相关的元数据（如消息ID、字体等），这些元数据可以作为列表中的一个特殊类型 `Seg` 或通过专门的 `Seg` 类型承载。(主人，此处请注意：原先 `message_id`, `font` 等是直接在 `content` 的字典中，现在如果 `content` 统一为 `List[Seg]`，这些字段需要考虑如何融入 `Seg` 序列。一个简单的做法是，`content` 的第一个 `Seg` 是一个特殊类型，比如 `type="message_meta"`，其 `data` 包含 `message_id`, `font` 等，后续 `Seg` 才是消息的实际内容。或者，如果参数不多，可以考虑将 `message_id` 等提升到 `Event` 对象的顶层，如果它们对于所有消息事件都是通用的。小猫在这里暂时保持 `content` 仍承载这些，但具体形式需主人定夺。为了演示 `List[Seg]` 的统一性，以下示例会将消息元数据也包装进 `Seg`。)
    * 对于 `notice.*`, `request.*`, `action.*` (除发送消息类), `action_response.*`, `meta.*` 类型事件: `content` 通常包含一个特定类型的 `Seg` 对象，该 `Seg` 对象的 `type` 字段指明其承载的数据性质 (例如 `notice.conversation.member_increase`)，其 `data` 字段则是一个包含该事件类型所需所有参数的字典。详见对应章节。
* **`raw_data` (Optional[str])**: 原始事件的字符串表示 (例如：平台推送的JSON原文)，可选，主要用于调试或特殊处理。

### 2.2. `UserInfo` 对象

用于描述用户信息。其结构与之前版本（v1.3.0提案中的定义）保持一致。

```json
{
  "platform": "Optional[str]",
  "user_id": "Optional[str]",
  "user_nickname": "Optional[str]",
  "user_cardname": "Optional[str]",
  "user_titlename": "Optional[str]",
  "permission_level": "Optional[str]",
  "role": "Optional[str]",
  "level": "Optional[str]",
  "sex": "Optional[str]",
  "age": "Optional[int]",
  "area": "Optional[str]",
  "additional_data": "Optional[Dict[str, Any]]"
}
```

### 2.3. `ConversationInfo` 对象

用于描述会话信息。其结构与之前版本（v1.3.0提案中的定义）保持一致。

```json
{
  "platform": "Optional[str]",
  "conversation_id": "str",
  "type": "str",
  "name": "Optional[str]",
  "parent_id": "Optional[str]",
  "extra": "Optional[Dict[str, Any]]"
}
```

### 2.4. `Seg` 对象 (通用信息单元)

AIcarus-Message-Protocol 定义的通用信息单元，是构成所有类型事件（包括用户消息内容、平台通知详情、平台请求参数、核心动作指令、动作响应详情以及元事件数据等）具体内容的原子构建块。

* **`type` (str)**: 字符串，用于区分 `Seg` 的类型和用途。
    * 用户消息内容段: 如 `"text"`, `"image"`, `"face"`, `"at"`, `"reply"` 等。
    * 承载参数集合的Seg: 对于非消息类事件，通常会有一个主 `Seg` 用于封装该事件的所有参数，其类型可以约定为例如 `notice.conversation.member_increase`，或者其他能明确其功能的类型字符串。
    * 其他特殊用途的Seg类型可按需定义。
* **`data` (Dict[str, Any])**: 承载该 `Seg` 类型的具体数据，其结构随 `type` 变化。
    * 例如: `Seg(type="text", data={"text": "你好"})`
    * 例如: `Seg(type="at", data={"user_id": "12345", "display_name": "@某人"})`
    * 例如: `Seg(type="reply", data={"message_id": "original_msg_id_abc"})` (此 `message_id` 指被回复的消息的平台ID)
    * 例如 (用于承载通知参数): `Seg(type="notice.conversation.member_increase", data={"operator_user_info": UserInfo, "join_type": "invite"})`

## 3. 事件类型详解 (`event_type` 及 `content` 结构)

### 3.1. 用户消息 (`event_type = message.*`)

由用户发送的消息。

* **`event_type` 示例**:
    * `message.private.friend` (好友私聊)
    * `message.private.temporary` (临时会话私聊)
    * `message.group.normal` (普通群消息)
    * `message.group.anonymous` (匿名群消息)
    * `message.channel.normal` (频道消息)
* **`content` 结构 (List[`Seg`])**:
    * 列表的第一个 `Seg` 对象通常用于承载消息的元数据。
        * `Seg(type="message_metadata", data={"message_id": "str", "font": "Optional[str]", "anonymity_info": "Optional[Dict]", "sender_title": "Optional[str]", "client_info": "Optional[Dict]"})`
            * `message_id` (str): 此条消息在平台上的唯一ID。 **(核心字段)**
            * `font` (Optional[str]): 字体信息 (部分平台支持)。
            * `anonymity_info` (Optional[Dict]): 如果是匿名消息，包含匿名者信息。
            * `sender_title` (Optional[str]): (特定于群聊/频道) 发送者当时的头衔。
            * `client_info` (Optional[Dict]): (部分平台) 发送消息的客户端信息。
    * 后续的 `Seg` 对象列表则代表消息的实际内容片段。
        * 例如: `Seg(type="text", data={"text": "你好"})`, `Seg(type="image", data={"url": "...", "file_id": "..."})`

**示例: QQ群普通消息 (用户发送 "你好 @张三 [图片]")**
```json
{
  "event_id": "uuid_generated_by_adapter_1",
  "event_type": "message.group.normal",
  "time": 1678886400123.0,
  "platform": "qq",
  "bot_id": "10001",
  "user_info": {
    "platform": "qq",
    "user_id": "user_sender_456",
    "user_nickname": "李四",
    "user_cardname": "群里的李四"
  },
  "conversation_info": {
    "platform": "qq",
    "conversation_id": "group123",
    "type": "group",
    "name": "测试群"
  },
  "content": [
    {
      "type": "message_metadata",
      "data": {
        "message_id": "platform_msg_789",
        "font": "宋体"
      }
    },
    { "type": "text", "data": { "text": "你好 " } },
    { "type": "at", "data": { "user_id": "user_zhangsan_001", "display_name": "@张三" } },
    { "type": "text", "data": { "text": " " } },
    { "type": "image", "data": { "url": "http://example.com/image.jpg", "file_id": "qq_image_abc" } }
  ],
  "raw_data": "{...原始QQ事件...}"
}
```

**示例: QQ群消息，是对另一条消息的回复**
```json
{
  "event_id": "uuid_generated_by_adapter_2",
  "event_type": "message.group.normal",
  "time": 1678886400888.0,
  "platform": "qq",
  "bot_id": "10001",
  "conversation_info": {
    "platform": "qq",
    "conversation_id": "group123",
    "type": "group",
    "name": "主人的秘密花园"
  },
  "user_info": {
      "platform": "qq",
      "user_id": "sender_user_id_111",
      "user_nickname": "回复者小可爱"
  },
  "content": [
    {
      "type": "message_metadata",
      "data": {
        "message_id": "current_message_id_xyz"
      }
    },
    {"type": "reply", "data": {"message_id": "replied_to_message_id_abc"}},
    {"type": "text", "data": {"text": "是的呢！"}}
  ]
}
```

### 3.2. 平台通知 (`event_type = notice.*`)

由平台产生的状态变更或信息通知。

* **`event_type` 示例**:
    * `notice.conversation.member_increase` (会话成员增加)
    * `notice.conversation.member_decrease` (会话成员减少)
    * `notice.message.recalled` (消息被撤回)
* **`content` 结构 (List[`Seg`])**: 通常包含一个 `Seg` 对象，其 `type` 类似于 `notice.conversation.member_increase`，其 `data` 字段是一个包含该通知类型所需具体参数的字典。

**示例: QQ群成员增加通知**
```json
{
  "event_id": "uuid_notice_1",
  "event_type": "notice.conversation.member_increase",
  "time": 1678886400300.0,
  "platform": "qq",
  "bot_id": "10001",
  "conversation_info": {
    "platform": "qq",
    "conversation_id": "group123",
    "type": "group",
    "name": "测试群"
  },
  "user_info": {
    "platform": "qq",
    "user_id": "new_member_789",
    "user_nickname": "萌新小王"
  },
  "content": [
    {
      "type": "notice.conversation.member_increase",
      "data": {
        "operator_user_info": {
          "platform": "qq",
          "user_id": "admin_user_007",
          "user_nickname": "管理员张三"
        },
        "join_type": "invite"
      }
    }
  ]
}
```

### 3.3. 平台请求 (`event_type = request.*`)

由平台产生的、需要机器人明确响应的请求。

* **`event_type` 示例**:
    * `request.friend.add` (收到好友添加请求)
    * `request.conversation.join_application` (用户申请加入会话)
* **`content` 结构 (List[`Seg`])**: 通常包含一个 `Seg` 对象，其 `type` 字段直接表示事件类型，例如: `request.friend.add`，其 `data` 字段是一个包含该请求类型所需具体参数的字典。

**示例: 收到QQ加好友请求**
```json
{
  "event_id": "uuid_request_1",
  "event_type": "request.friend.add",
  "time": 1678886400400.0,
  "platform": "qq",
  "bot_id": "10001",
  "user_info": {
    "platform": "qq",
    "user_id": "potential_friend_007",
    "user_nickname": "想加好友的小明"
  },
  "conversation_info": null,
  "content": [
    {
      "type": "request.friend.add",
      "data": {
        "comment": "你好，我是小明，想加你好友！",
        "request_flag": "flag_for_responding_to_friend_request_abc"
      }
    }
  ]
}
```

### 3.4. 核心动作 (`event_type = action.*`)

由Core发起，指示Adapter执行的动作。

* **`event_type` 示例**:
    * `action.message.send` (发送消息)
    * `action.message.recall` (撤回消息)
    * `action.conversation.kick_user` (将会话成员踢出)
* **`content` 结构 (List[`Seg`])**:
    * 对于 `action.message.send` 类型: `content` 是一个包含构成待发送消息实际内容的 `Seg` 对象列表 (与 `message.*` 事件中的消息内容部分类似，但不包含 `message_metadata` Seg，因为发送时消息ID等由平台生成)。
    * 对于其他 `action.*` 类型: `content` 通常包含一个 `Seg` 对象，其 `type` 类似于 `action.message.recall`，其 `data` 字段是一个包含执行该动作所需具体参数的字典。

**示例: Core指示Adapter发送一条QQ群消息 (回复)**
```json
{
  "event_id": "core_action_uuid_1",
  "event_type": "action.message.send",
  "time": 1678886400500.0,
  "platform": "qq",
  "bot_id": "10001",
  "conversation_info": {
    "platform": "qq",
    "conversation_id": "target_group_456",
    "type": "group"
  },
  "user_info": null,
  "content": [
    { "type": "text", "data": { "text": "收到主人的命令！" } }
    // 若要作为回复，则应包含如：
    // { "type": "reply", "data": { "message_id": "target_replied_message_id" } },
    // { "type": "text", "data": { "text": "收到主人的命令！" } }
  ]
}
```

**示例: Core指示Adapter撤回某条消息**
```json
{
  "event_id": "core_action_uuid_2",
  "event_type": "action.message.recall",
  "time": 1678886400600.0,
  "platform": "qq",
  "bot_id": "10001",
  "conversation_info": {
    "platform": "qq",
    "conversation_id": "group_where_message_exists_789",
    "type": "group"
  },
  "user_info": null,
  "content": [
    {
      "type": "action.message.recall",
      "data": {
        "target_message_id": "platform_message_id_to_be_recalled_xyz"
      }
    }
  ]
}
```

### 3.5. 动作响应 (`event_type = action_response.*`) (可选)

Adapter对Core发起的`action.*`的执行结果进行反馈。

* **`event_type` 示例**:
    * `action_response.success`
    * `action_response.failure`
* **`content` 结构 (List[`Seg`])**: 通常包含一个 `Seg` 对象，其 `type` 类似于 `action_response.success`，其 `data` 字段是一个包含响应所需具体参数的字典。
    * `original_event_id` (str): 对应Core发起的`action.*`事件的 `event_id`，用于追踪。
    * `original_action_type` (str): 原始动作的 `event_type` (例如: `"action.message.send"`)。
    * `status_code` (Optional[int]): 平台或Adapter定义的执行状态码/错误码。
    * `message` (Optional[str]): 成功或失败的描述信息。
    * `data` (Optional[Dict]): 成功时可能返回的额外数据 (例如：发送消息成功后返回的新消息ID `{"sent_message_id": "platform_new_msg_id"}`)。

> **注**: 动作响应的记录不仅可以用于调试和追踪，还可以为学习和记忆系统提供数据支持，帮助总结规律和优化未来的决策。

**示例: 动作执行成功响应 (假设)**
```json
{
  "event_id": "adapter_response_uuid_1",
  "event_type": "action_response.success",
  "time": 1678886400700.0,
  "platform": "qq",
  "bot_id": "10001",
  "content": [
    {
      "type": "action_response.success",
      "data": {
        "original_event_id": "core_action_uuid_1",
        "original_action_type": "action.message.send",
        "status_code": 200,
        "message": "消息已成功发送。",
        "data": {
          "sent_message_id": "platform_new_msg_id_abc123"
        }
      }
    }
  ]
}
```

**示例: 动作执行失败响应 (假设)**
```json
{
  "event_id": "adapter_response_uuid_1",
  "event_type": "action_response.failure",
  "time": 1678886400700.0,
  "platform": "qq",
  "bot_id": "10001",
  "content": [
    {
      "type": "action_response.failure",
      "data": {
        "original_event_id": "core_action_uuid_1",
        "original_action_type": "action.message.send",
        "status_code": 403,
        "message": "网络连接错误，无法发送消息。",
        "data": {}
      }
    }
  ]
}
```

### 3.6. 平台元事件 (`event_type = meta.*`)

关于机器人自身或 Adapter 状态的元事件。

* **`event_type` 示例**:
    * `meta.lifecycle.connect` (Adapter连接到平台成功)
    * `meta.lifecycle.disconnect` (Adapter与平台断开)
    * `meta.heartbeat` (心跳)
* **`content` 结构 (List[`Seg`])**: 通常包含一个 `Seg` 对象，其 `type` 类似于 `meta.lifecycle.connect`，其 `data` 字段是一个包含该元事件相关的状态信息的字典。

**示例: Adapter连接成功元事件**
```json
{
  "event_id": "adapter_meta_uuid_1",
  "event_type": "meta.lifecycle.connect",
  "time": 1678886400800.0,
  "platform": "qq",
  "bot_id": "10001",
  "content": [
    {
      "type": "meta.lifecycle.connect",
      "data": {
        "adapter_version": "1.0.0",
        "platform_api_version": "v11_custom"
      }
    }
  ]
}
```

## 4. 传输

所有 `Event` 对象在序列化为 JSON 字符串后，通过自定义实现的通信组件 (如基于 WebSocket、TCP、HTTP 或消息队列) 进行传输。

## 5. 版本控制

本协议当前版本为 **v1.4.0**。后续如有不兼容的重大变更，将递增主版本号或次版本号。小的兼容性修正或补充可递增修订版本号。