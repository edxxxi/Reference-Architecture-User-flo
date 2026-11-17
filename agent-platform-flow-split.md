# Agentic Platform 前端流程圖（拆分版）
虛線框的雙邊節點 `[[ ]]` 代表跨圖連結：「→ 圖 N」為出口，「來自 圖 N」為入口。

## 圖 0：Session 狀態機總覽

```mermaid
stateDiagram-v2
    state "Draft 草稿" as Draft
    state "Creating 建立中" as Creating
    state "Loading 載入歷史" as Loading
    state "Idle 待輸入" as Idle
    state "Streaming 串流中" as Streaming
    state "AwaitingApproval 等待授權" as AwaitingApproval
    state "ReadOnly 唯讀" as ReadOnly

    [*] --> Draft: 開新對話
    [*] --> Loading: 開啟 /chat/:id 或點擊歷史
    Draft --> Draft: 切換 Agent
    Draft --> Creating: 送出首句
    Creating --> Draft: 送出失敗 / 403
    Creating --> Streaming: HTTP 200 + session_created
    Loading --> Draft: 404 / 403
    Loading --> Idle: 後端 idle
    Loading --> Streaming: 後端 running，GET 接回
    Loading --> AwaitingApproval: 後端 pending_approval
    Loading --> ReadOnly: Agent 權限已拔除
    Idle --> Streaming: 送出訊息 / Retry
    Idle --> ReadOnly: 送出時 403
    Streaming --> Streaming: 事件處理 / 斷線重連
    Streaming --> AwaitingApproval: tool_approval_required
    Streaming --> Idle: message_end / error / 停止
    Streaming --> ReadOnly: PERMISSION_REVOKED
    AwaitingApproval --> Streaming: 授權 API 200，新 SSE
    AwaitingApproval --> Idle: 逾時 10 分鐘 / 停止
    AwaitingApproval --> ReadOnly: 授權 API 403
    ReadOnly --> [*]: 透過側欄離開
```

## 圖 1：登入、初始化與 Session 載入

```mermaid
flowchart TD
    classDef auth fill:#f9f0ff,stroke:#d3adf7,stroke-width:2px;
    classDef ui fill:#e6f7ff,stroke:#91d5ff,stroke-width:2px;
    classDef state fill:#feffe6,stroke:#fffb8f,stroke-width:2px;
    classDef stream fill:#f6ffed,stroke:#b7eb8f,stroke-width:2px;
    classDef warn fill:#fff2e8,stroke:#ffbb96,stroke-width:2px;
    classDef dev fill:#fffbe6,stroke:#ffe58f,stroke-width:2px;
    classDef ref fill:#f5f5f5,stroke:#8c8c8c,stroke-width:1px,stroke-dasharray:4 3;

    Start(["開啟平台<br/>或直接開啟 /chat/:id 網址"]) --> TokenCheck{"檢查本地 Token"}:::auth
    InLogin[["來自 圖 3：Refresh 失敗"]]:::ref -.-> Login
    TokenCheck -->|無效或過期| Refresh["嘗試 Refresh Token"]:::auth
    Refresh -->|成功| TokenCheck
    Refresh -->|失敗| Login["導向 SSO 或帳密登入<br/>登入後帶回原網址"]:::auth
    Login --> TokenCheck
    TokenCheck -->|有效| InitFetch["並行初始化請求：<br/>1. GET /user/permissions<br/>2. GET /agents（後端依角色過濾）<br/>3. GET /sessions<br/>全部完成前顯示 Skeleton"]:::auth
    InitFetch --> RouteCheck{"網址是否帶 session_id?"}:::ui
    RouteCheck -->|否| ToDraft[["→ 圖 2：Draft 狀態"]]:::ref
    RouteCheck -->|是| LoadHistory

    InLoad[["來自 圖 2：側欄切換 Session<br/>來自 圖 4：授權已失效"]]:::ref -.-> LoadHistory
    LoadHistory["GET /sessions/:id<br/>Hydration：還原訊息、Event、Tool 紀錄"]:::ui
    LoadHistory --> LoadResult{"載入結果"}:::auth
    LoadResult -->|404 或 403| NotFound["提示 Session 不存在或無權存取"]:::warn
    NotFound --> ToDraft
    LoadResult -->|200| LockSelector["Agent 選單鎖定為唯讀 Badge"]:::state
    LockSelector --> CheckAuth{"綁定的 agent_id<br/>仍在授權清單內?"}:::auth
    CheckAuth -->|否| ToRevoke[["→ 圖 5：權限降級"]]:::ref
    CheckAuth -->|是| RunStatus{"Session 後端執行狀態"}:::state
    RunStatus -->|idle| ToInput[["→ 圖 3：輸入框就緒"]]:::ref
    RunStatus -->|running：GET stream 接回| ToStream[["→ 圖 3：串流狀態"]]:::ref
    RunStatus -->|pending_approval| ToHITL[["→ 圖 4：HITL 授權"]]:::ref
```

## 圖 2：Draft 狀態與側欄操作

```mermaid
flowchart TD
    classDef auth fill:#f9f0ff,stroke:#d3adf7,stroke-width:2px;
    classDef ui fill:#e6f7ff,stroke:#91d5ff,stroke-width:2px;
    classDef state fill:#feffe6,stroke:#fffb8f,stroke-width:2px;
    classDef stream fill:#f6ffed,stroke:#b7eb8f,stroke-width:2px;
    classDef warn fill:#fff2e8,stroke:#ffbb96,stroke-width:2px;
    classDef dev fill:#fffbe6,stroke:#ffe58f,stroke-width:2px;
    classDef ref fill:#f5f5f5,stroke:#8c8c8c,stroke-width:1px,stroke-dasharray:4 3;

    subgraph SG_Draft["Draft 與 Agent 選擇"]
        InDraft[["來自 圖 1 / 圖 5 / 本圖側欄"]]:::ref -.-> DraftMode
        DraftMode["Draft 狀態：<br/>頂部 Agent 選單可自由切換"]:::state
        DraftMode --> HasAgents{"是否有任何授權 Agent?"}:::auth
        HasAgents -->|否| NoAgent["空狀態：目前無可用 Agent<br/>輸入框停用"]:::warn
        HasAgents -->|是| SelectAgent["選擇或切換授權的 Agent"]:::ui
        SelectAgent --> SyncCap["依 capabilities 更新 UI：<br/>1. 切換 Starter Prompts<br/>2. 啟閉檔案上傳與格式限制<br/>3. 清除不相容的已選檔案"]:::state
        SyncCap --> ToInput[["→ 圖 3：輸入框就緒<br/>送出前可再切換 Agent"]]:::ref
        ToInput -.->|再次切換| SelectAgent
    end

    subgraph SG_Nav["側欄操作：任何狀態下皆可觸發"]
        Nav(["使用者操作左側欄"]):::ui --> NavType{"操作類型"}:::ui
        NavType -->|釘選 或 重新命名| PinRename["樂觀更新列表<br/>失敗則還原<br/>不影響目前畫面"]:::ui
        NavType -->|刪除| DeleteConfirm{"二次確認刪除?"}:::warn
        DeleteConfirm -->|是| DeleteAPI["DELETE /sessions/:id"]:::ui
        DeleteAPI --> IsCurrent{"刪除的是目前開啟的 Session?"}:::state
        IsCurrent -->|否| DeleteDone["從列表移除"]:::ui
        NavType -->|新對話 或 切換 Session| HasStream{"目前有進行中的串流<br/>或等待授權?"}:::state
        IsCurrent -->|是| HasStream
        HasStream -->|是| Detach["直接中止：<br/>1. 若連線存在，AbortController 斷開 SSE<br/>2. POST /api/chat/cancel<br/>3. 未完成步驟與待授權項目標記為已中斷"]:::warn
        HasStream -->|否| Dest{"目的地"}:::ui
        Detach --> Dest
    end

    Dest -->|新對話| DraftMode
    Dest -->|歷史 Session| ToLoad[["→ 圖 1：載入 Session"]]:::ref
```

## 圖 3：送出訊息、串流解析與結束

```mermaid
flowchart TD
    classDef auth fill:#f9f0ff,stroke:#d3adf7,stroke-width:2px;
    classDef ui fill:#e6f7ff,stroke:#91d5ff,stroke-width:2px;
    classDef state fill:#feffe6,stroke:#fffb8f,stroke-width:2px;
    classDef stream fill:#f6ffed,stroke:#b7eb8f,stroke-width:2px;
    classDef warn fill:#fff2e8,stroke:#ffbb96,stroke-width:2px;
    classDef dev fill:#fffbe6,stroke:#ffe58f,stroke-width:2px;
    classDef ref fill:#f5f5f5,stroke:#8c8c8c,stroke-width:1px,stroke-dasharray:4 3;

    subgraph SG_Send["發送訊息與 HTTP 攔截"]
        InInput[["來自 圖 1 / 圖 2"]]:::ref -.-> InputReady
        InputReady["底部輸入框就緒"]:::ui
        InputReady --> SendPrompt["送出 Prompt<br/>產生 client_message_id<br/>Draft 進入 creating 暫態，選單暫時停用"]:::ui
        SendPrompt --> APICall["POST /api/chat/stream"]:::stream
        InRetry[["來自 圖 5：Retry 請求"]]:::ref -.-> HTTPStatus
        APICall --> HTTPStatus{"檢查 HTTP 狀態碼"}:::auth
        HTTPStatus -->|401| Refresh401["Refresh Token 後<br/>以相同 client_message_id 重送"]:::auth
        Refresh401 --> APICall
        Refresh401 -.->|Refresh 失敗| ToLogin[["→ 圖 1：登入"]]:::ref
        HTTPStatus -->|5xx 或網路錯誤| SendFail["提示送出失敗，保留輸入內容<br/>Draft 時選單恢復可切換"]:::warn
        SendFail --> InputReady
    end

    subgraph SG_Stream["串流解析"]
        InStream[["來自 圖 1：running 接回<br/>來自 圖 4：授權成功，新 SSE"]]:::ref -.-> StreamingState
        StreamingState["串流狀態：送出鈕變為停止"]:::stream
        StreamingState --> EventRouter{"解析 SSE Event 類型"}:::stream
        EventRouter -->|session_created| BindSession["Draft 轉 Active：<br/>1. 選單鎖定為唯讀 Badge<br/>2. router.replace 至 /chat/:id<br/>3. 左側列表新增項目"]:::state
        EventRouter -->|agent_thought| UpdateThought["更新 Thinking 步驟條"]:::stream
        EventRouter -->|tool_call_start / end| UpdateToolUI["更新行內 Tool 狀態燈號"]:::stream
        EventRouter -->|message_delta| RenderMarkdown["累積片段，批次渲染 Markdown"]:::stream
        BindSession --> Wait["等待下一個事件<br/>記錄 lastSeq"]:::stream
        UpdateThought --> Wait
        UpdateToolUI --> Wait
        RenderMarkdown --> Wait
        Wait --> EventRouter
        Wait -->|連線中斷且未收到 message_end| Reconnect{"以 after=lastSeq 重連<br/>是否成功且未超過重試上限?"}:::warn
        Reconnect -->|是| EventRouter
        EventRouter -->|error| ErrType{"錯誤碼"}:::warn
        ErrType -->|其他錯誤| RenderStreamErr["渲染錯誤區塊<br/>未完成步驟標記為失敗"]:::warn
        Reconnect -->|否| RenderStreamErr
        EventRouter -->|tool_approval_required| ToHITL[["→ 圖 4：HITL 授權"]]:::ref
    end

    subgraph SG_Stop["主動中斷"]
        InStop[["來自 圖 4：等待授權時停止"]]:::ref -.-> StopBtn
        StopBtn(["使用者點擊停止"]):::warn
        StopBtn --> CancelReq["1. 若連線存在，AbortController 斷開 SSE<br/>2. POST /api/chat/cancel<br/>3. 未完成步驟與待授權項目標記為已中斷<br/>4. 丟棄之後到達的事件"]:::warn
    end

    subgraph SG_End["串流結束與自動命名"]
        InEnd[["來自 圖 4：授權逾時"]]:::ref -.-> StreamEnd
        StreamEnd(["串流結束：按鈕恢復為送出"])
        StreamEnd --> FirstTurnCheck{"是否為首輪且正常完成?"}:::state
        FirstTurnCheck -->|是| AutoTitle["背景生成 Session 標題<br/>若使用者已手動改名則不覆寫"]:::ui
    end

    StreamingState -.->|任何時刻| StopBtn
    CancelReq --> StreamEnd
    EventRouter -->|message_end| StreamEnd
    RenderStreamErr --> StreamEnd
    FirstTurnCheck -->|否| InputReady
    AutoTitle --> InputReady
    HTTPStatus -->|200 建立 SSE| StreamingState
    HTTPStatus -->|403| ToRevoke[["→ 圖 5：權限降級"]]:::ref
    ErrType -->|PERMISSION_REVOKED| ToRevoke
```

## 圖 4：人工介入授權 HITL

```mermaid
flowchart TD
    classDef auth fill:#f9f0ff,stroke:#d3adf7,stroke-width:2px;
    classDef ui fill:#e6f7ff,stroke:#91d5ff,stroke-width:2px;
    classDef state fill:#feffe6,stroke:#fffb8f,stroke-width:2px;
    classDef stream fill:#f6ffed,stroke:#b7eb8f,stroke-width:2px;
    classDef warn fill:#fff2e8,stroke:#ffbb96,stroke-width:2px;
    classDef dev fill:#fffbe6,stroke:#ffe58f,stroke-width:2px;
    classDef ref fill:#f5f5f5,stroke:#8c8c8c,stroke-width:1px,stroke-dasharray:4 3;

    InHITL[["來自 圖 3：tool_approval_required<br/>來自 圖 1：重整還原 pending_approval"]]:::ref -.-> RenderHITL
    RenderHITL["後端結束本次串流<br/>Session 狀態記為 pending_approval<br/>氣泡內渲染 允許執行 / 拒絕 按鈕與參數預覽<br/>以 approval_id 對應<br/>顯示到期時間（10 分鐘，以後端為準）"]:::warn
    RenderHITL --> UserDecision{"使用者決策"}:::warn
    UserDecision -->|允許 或 拒絕| SendApproval["POST /api/chat/tool-approval<br/>按鈕 loading 防止連點<br/>成功時回傳新的 SSE 串流"]:::stream
    UserDecision -->|點擊停止| ToStop[["→ 圖 3：主動中斷"]]:::ref
    UserDecision -->|超過 10 分鐘未決策| ApprovalExpired["授權逾時：<br/>按鈕停用，步驟標記為已逾時<br/>後端終止該次執行"]:::warn
    SendApproval --> ApprovalResult{"API 回應"}:::auth
    ApprovalResult -->|200 建立新 SSE| ToStream[["→ 圖 3：串流狀態"]]:::ref
    ApprovalResult -->|403| ToRevoke[["→ 圖 5：權限降級"]]:::ref
    ApprovalResult -->|409 已過期或已處理| ApprovalStale["提示授權已失效<br/>重新同步 Session 狀態"]:::warn
    ApprovalStale --> ToLoad[["→ 圖 1：載入 Session"]]:::ref
    ApprovalExpired --> ToEnd[["→ 圖 3：串流結束"]]:::ref
```

## 圖 5：權限降級與可觀測性

```mermaid
flowchart TD
    classDef auth fill:#f9f0ff,stroke:#d3adf7,stroke-width:2px;
    classDef ui fill:#e6f7ff,stroke:#91d5ff,stroke-width:2px;
    classDef state fill:#feffe6,stroke:#fffb8f,stroke-width:2px;
    classDef stream fill:#f6ffed,stroke:#b7eb8f,stroke-width:2px;
    classDef warn fill:#fff2e8,stroke:#ffbb96,stroke-width:2px;
    classDef dev fill:#fffbe6,stroke:#ffe58f,stroke-width:2px;
    classDef ref fill:#f5f5f5,stroke:#8c8c8c,stroke-width:1px,stroke-dasharray:4 3;

    subgraph SG_Revoke["權限異動與唯讀降級"]
        InRevoke[["來自 圖 1：Agent 權限已拔除<br/>來自 圖 3：HTTP 403 / PERMISSION_REVOKED<br/>來自 圖 4：授權 API 403"]]:::ref -.-> ToastRevoke
        ToastRevoke["彈出權限變更提示"]:::warn
        ToastRevoke --> HasSessionId{"是否已取得 session_id?"}:::state
        HasSessionId -->|否，仍為 Draft| ReloadAgents["重新取得 Agent 清單<br/>保留輸入內容"]:::warn
        ReloadAgents --> ToDraft[["→ 圖 2：Draft 狀態"]]:::ref
        HasSessionId -->|是| ReadOnlyUI["唯讀模式：<br/>可瀏覽對話與展開舊 Event<br/>鎖定輸入框、HITL 按鈕與 Retry"]:::warn
        ReadOnlyUI -.->|離開需透過側欄| ToNav[["→ 圖 2：側欄操作"]]:::ref
    end

    subgraph SG_Obs["可觀測性：依角色決定渲染內容"]
        ObsEntry(["點擊任一 Event 或 Tool 節點<br/>串流中或歷史紀錄皆可"]):::ui
        ObsEntry --> RoleBranch{"使用者角色"}:::auth
        RoleBranch -->|Standard| BasicObserve["展開行內摺疊匣<br/>僅工具名稱與狀態燈號<br/>後端不下發 Raw 資料"]:::ui
        RoleBranch -->|Developer| DevObserve["滑出 Trace Inspector<br/>動態載入元件<br/>按需 GET trace 詳細資料"]:::dev
        DevObserve --> DevActions["Raw JSON / Waterfall / 耗時 / Token 數<br/>複製 Trace ID / 匯出 Session Log"]:::dev
        DevActions -->|點擊 Retry| RetryCheck{"Session 可寫入<br/>且未在串流中?"}:::auth
        RetryCheck -->|否| RetryDisabled["Retry 按鈕停用並顯示原因"]:::dev
        RetryCheck -->|是| RetryCall["POST /api/chat/retry<br/>帶 from_node_id"]:::dev
        RetryCall --> ToHTTP[["→ 圖 3：檢查 HTTP 狀態碼"]]:::ref
    end
```
