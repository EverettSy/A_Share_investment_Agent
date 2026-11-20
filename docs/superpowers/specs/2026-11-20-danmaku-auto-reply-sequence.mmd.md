# 弹幕自动回复 · 最终版时序图（代理商身份）

> 依据：[弹幕 HTTP 对接文档](https://docs.apipost.net/docs/detail/732230de0472000?locale=zh-cn&target_id=32229200b12005)  
> 身份：**代理商** = 监听服务；**平台** = 文档「我方服务器」；**源直播间** = `live_url`（如抖音）

## 文档既有对接流程（无自动回复）

```mermaid
sequenceDiagram
    autonumber
    participant U as 用户/主播
    participant P as 平台服务器
    participant G as 代理商监听服务
    participant S as 源直播间

    U->>P: 开播
    P->>G: POST /api/open/tasks<br/>live_auth_code, live_url
    G->>S: 建立监听
    G-->>P: code=1 监听成功

    loop 收到弹幕
        S-->>G: 观众弹幕/事件
        G->>P: POST /api/partner/danmaku/comments<br/>comments[]（msg/come/…）
        P-->>G: code=1 已接收
    end

    opt 监听异常可重连
        G->>P: comments msg_type=sys status=-2
        P-->>G: code=1
        P->>G: POST /api/open/tasks/reconnect<br/>live_auth_code, request_id
        G->>S: 重新建立监听
        G-->>P: code=1 重连成功
    end

    alt 源站已下播
        G->>P: comments msg_type=sys status=-1
        P-->>G: code=1
    else 用户关播
        U->>P: 关播
        P->>G: POST /api/open/tasks/stop
        G->>S: 停止订阅并清理
        G-->>P: code=1 停止成功
    end
```

## 叠加自动回复（推荐：代理商侧匹配；出站待补齐）

```mermaid
sequenceDiagram
    autonumber
    participant P as 平台服务器
    participant G as 代理商（含 AutoReplyEngine）
    participant S as 源直播间

    P->>G: POST /api/open/tasks
    G->>S: 建立监听
    G-->>P: code=1

    S-->>G: 观众弹幕 message
    G->>G: 精确匹配关键词 + 冷却/QPS
    G->>P: POST comments（上报观众弹幕）

    alt 命中规则
        Note over G,S: 【文档缺口】向源直播间发送回复话术<br/>需另有发弹幕能力
        G--xS: ⑤' 发送自动回复（待确认）
        opt 若需让平台可见回复
            G->>P: POST comments（bot 回复？文档未定义）
        end
    else 未命中/冷却
        G->>G: 审计 skipped
    end
```

## 步骤对照

| 步骤 | 方向 | 接口/动作 | 自动回复相关 |
|------|------|-----------|--------------|
| 开播任务 | 平台→代理商 | `POST /api/open/tasks` | 会话键 `live_auth_code` |
| 监听采集 | 源站→代理商 | 代理商自建 | **入站在此发生** |
| 上报弹幕 | 代理商→平台 | `POST /api/partner/danmaku/comments` | 上报；非回复出站 |
| 匹配回复 | 代理商内部 | AutoReplyEngine | MVP：精确关键词 |
| 回源发送 | 代理商→源站 | **文档未提供** | 闭环阻塞点 |
| 停止/重连 | 双向 | `/tasks/stop` · `/tasks/reconnect` · `sys±1/±2` | 任务生命周期 |
