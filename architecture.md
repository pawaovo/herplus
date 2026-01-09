**文档文件名**: `docs/architecture.md` (Part 1/2)

```markdown
# HerPlus 技术架构文档 (Technical Architecture)

| 文档属性 | 内容 |
| :--- | :--- |
| **项目名称** | HerPlus 智能戒指 APP |
| **版本号** | v1.0 (Architecture) |
| **状态** | **已定稿** |
| **基于** | PRD v1.0, UI/UX Spec, Audit Report |
| **作者** | Architect (Winston) |

---

## 1. 技术栈选型 (Tech Stack)

基于 "Borderless, Soft & Healing" 的体验需求（高性能动画）和跨平台目标，选型如下：

### 1.1 移动端 (Mobile App)
* **框架**: **Flutter** (Dart)
    * *理由*: 相比 React Native，Flutter 的 Skia 引擎能更好地还原 "多态漩涡" (Shader/WebGL) 和复杂的全屏转场动画，且 BLE 库生态成熟。
* **蓝牙通信**: `flutter_blue_plus`
* **本地存储**: **WatermelonDB** (基于 SQLite，高性能，支持离线优先)
* **状态管理**: **Riverpod**
* **动画引擎**: **Rive** 或 **Flutter Shaders** (用于实现 Vortex)

### 1.2 后端服务 (Backend)
* **Runtime**: **Node.js** (TypeScript) - *NestJS 框架*
* **API 风格**: RESTful (主) + WebSocket (用于 Guardian 实时信令)
* **AI 网关**: LangChain.js (封装豆包大模型，管理 Context 和 Prompt)

### 1.3 数据存储 (Data Persistence)
* **关系型数据库**: **PostgreSQL 15**
    * *规范*: 所有表名、字段名强制使用 `snake_case`。
* **缓存/队列**: **Redis** (用于 AI 会话状态、SOS 短期令牌、限流)
* **时序数据**: TimescaleDB (基于 PG 的插件，优化心率/体温等高频数据存储)

---

## 2. 系统架构图 (System Architecture)

graph TD
    %% --- Client Side ---
    subgraph ClientBox ["HerPlus App (Flutter)"]
        direction TB
        UI["UI Layer (Vortex/Charts)"]
        Logic["Business Logic (Riverpod)"]
        LocalDB[("WatermelonDB - Offline First")]
        BLE["BLE Manager"]
    end

    %% --- Hardware Side ---
    subgraph RingBox ["Smart Ring"]
        direction TB
        Sensor["Sensors (PPG/Temp/IMU)"]
    end

    %% --- Cloud Side ---
    subgraph CloudBox ["HerPlus Backend Cloud"]
        direction TB
        Gateway["API Gateway / Load Balancer"]
        Auth["Auth Service (JWT)"]
        Core["Core API (User/Sync/Impact)"]
        AI_Svc["AI Service (LangChain)"]
        Guardian_Svc["Guardian Socket Server"]
        
        DB[("PostgreSQL + TimescaleDB")]
        Cache[("Redis")]
    end

    %% --- External Services ---
    subgraph ExternalBox ["Third Party Services"]
        direction TB
        Doubao["豆包 LLM API"]
        SMS["SMS Provider (Twilio/Aliyun)"]
        Map["Mapbox API"]
    end

    %% --- Connections ---
    BLE <-->|Bluetooth 5.0| Sensor
    Logic <-->|Sync/Rest| Gateway
    Logic <-->|Socket| Guardian_Svc
    
    Gateway --> Auth
    Gateway --> Core
    Gateway --> AI_Svc
    
    Core --> DB
    AI_Svc --> Doubao
    AI_Svc --> Cache
    
    Guardian_Svc --> SMS
    Guardian_Svc --> Map

```

---

## 3. 数据库模型设计 (Database Schema)

**设计原则**：依据《数据接口对比说明》，全量修正命名不一致问题，统一使用 **Snake Case**。

### 3.1 用户与配置 (Users & Profile)

> **修复点**：增加 `is_onboarding_completed` (替代 is_new_user)，统一身高体重单位，增加科研/守护开关。

```sql
CREATE TABLE users (
    user_id             UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    phone_number        VARCHAR(20) UNIQUE NOT NULL, -- E.164 格式
    created_at          TIMESTAMP DEFAULT NOW(),
    
    -- 状态标识 (对应 is_new_user 逻辑)
    is_onboarding_completed BOOLEAN DEFAULT FALSE,
    last_login_at       TIMESTAMP,

    -- 基础画像
    gender              VARCHAR(10) CHECK (gender IN ('FEMALE', 'NOT_SURE')),
    dob                 DATE,          -- 出生日期
    height_cm           NUMERIC(5,2),  -- 统一公制 cm
    weight_kg           NUMERIC(5,2),  -- 统一公制 kg
    
    -- 隐私与开关 (修复审计缺口)
    is_research_active  BOOLEAN DEFAULT FALSE, -- 科研数据开关
    is_guardian_active  BOOLEAN DEFAULT FALSE, -- 守护功能开关
    privacy_agreed_at   TIMESTAMP,             -- 隐私条款同意时间

    -- 周期模型 (统一命名为 length)
    cycle_length_days   INT DEFAULT 28, -- 对应 PRD cycle_duration
    period_length_days  INT DEFAULT 5,  -- 对应 PRD period_duration
    last_period_date    DATE            -- 上次经期开始日
);

-- 目标设定 (新增表，解决字段冗余)
CREATE TABLE user_goals (
    user_id             UUID REFERENCES users(user_id),
    daily_steps         INT DEFAULT 10000,
    daily_calories      INT DEFAULT 2500,
    stand_hours         INT DEFAULT 12,
    PRIMARY KEY (user_id)
);

```

### 3.2 每日健康汇总 (Health Daily)

> **修复点**：统一 `rhr`, `stress_avg` 命名，补全 `sleep_score`, `readiness_score`。

```sql
CREATE TABLE health_daily (
    id                  BIGSERIAL PRIMARY KEY,
    user_id             UUID REFERENCES users(user_id),
    date                DATE NOT NULL,
    
    -- 核心分数
    readiness_score     INT, -- 0-100
    sleep_score         INT, -- 0-100 (补全公式因子)
    
    -- 生理指标 (统一 Snake Case)
    rhr                 INT,             -- 静息心率 (原 restingHeartRate)
    stress_avg          INT,             -- 平均压力 (原 stressIndex)
    skin_temp_delta     NUMERIC(4,2),    -- 体温偏差 (原 skinTemperatureDelta)
    respiratory_rate    INT,             -- 呼吸率
    spo2_avg            INT,             -- 平均血氧 (补全)
    
    -- 活动数据
    daily_steps         INT,             -- 步数 (原 stepCount/steps_total)
    calories_burned     INT,             -- 活动消耗
    stand_hours         INT,             -- 站立小时
    
    -- 睡眠结构 (JSON 存储分期数据)
    -- 格式: { "deep": 45, "light": 120, "rem": 80, "awake": 10 } (单位: 分钟)
    sleep_stages_summary JSONB,
    
    UNIQUE(user_id, date)
);

```

### 3.3 AI 会话与上下文 (AI Sessions)

> **修复点**：结构化会话记录，增加 `session_type`, `round_count`。

```sql
CREATE TABLE ai_sessions (
    session_id          UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id             UUID REFERENCES users(user_id),
    
    -- 会话类型: 'HOME' (首页对话舱), 'EXPERT' (专家咨询)
    session_type        VARCHAR(20) NOT NULL, 
    
    -- 配额控制
    round_count         INT DEFAULT 0,  -- 当前轮次
    is_active           BOOLEAN DEFAULT TRUE,
    
    -- 上下文快照 (审计建议新增)
    -- 记录生成对话时的身体状态: { "stress": 85, "sleep": 60 }
    context_snapshot    JSONB, 
    
    created_at          TIMESTAMP DEFAULT NOW(),
    updated_at          TIMESTAMP DEFAULT NOW()
);

CREATE TABLE ai_messages (
    id                  BIGSERIAL PRIMARY KEY,
    session_id          UUID REFERENCES ai_sessions(session_id),
    role                VARCHAR(10), -- 'USER' or 'ASSISTANT'
    content             TEXT,
    sentiment_tag       VARCHAR(20), -- 预测标签/情绪标签
    created_at          TIMESTAMP DEFAULT NOW()
);

```

### 3.4 公益与科研 (Impact & Research)

> **修复点**：补全审计报告中缺失的实体结构。

```sql
-- 公益项目 (审计新增)
CREATE TABLE charity_projects (
    project_id          VARCHAR(50) PRIMARY KEY, -- 如 'EDU_GIRLS_01'
    title               VARCHAR(200),
    org_name            VARCHAR(100),
    image_url           TEXT,
    target_amount       DECIMAL(10, 2),
    current_amount      DECIMAL(10, 2),
    status              VARCHAR(20) -- 'ACTIVE', 'COMPLETED'
);

-- 用户捐赠记录
CREATE TABLE charity_contributions (
    id                  BIGSERIAL PRIMARY KEY,
    user_id             UUID REFERENCES users(user_id),
    project_id          VARCHAR(50) REFERENCES charity_projects(project_id),
    date                DATE,
    steps_donated       INT,
    currency_value      DECIMAL(10, 2), -- 此时汇率折算的价值
    created_at          TIMESTAMP DEFAULT NOW()
);

-- 科研项目 (审计新增)
CREATE TABLE research_studies (
    study_id            VARCHAR(50) PRIMARY KEY,
    title               VARCHAR(200),
    institution         VARCHAR(100), -- 合作机构
    required_data_types TEXT[],       -- ['SLEEP', 'HRV']
    is_recruiting       BOOLEAN DEFAULT TRUE
);

-- 科研参与关系
CREATE TABLE research_enrollments (
    user_id             UUID REFERENCES users(user_id),
    study_id            VARCHAR(50) REFERENCES research_studies(study_id),
    enrolled_at         TIMESTAMP DEFAULT NOW(),
    status              VARCHAR(20), -- 'ACTIVE', 'WITHDRAWN'
    PRIMARY KEY (user_id, study_id)
);


---

## 4. API 接口定义 (API Contract)

**设计原则**：RESTful 风格，所有 JSON 字段强制使用 `snake_case`。
**鉴权方式**：Bearer Token (JWT)

### 4.1 认证与用户 (Auth & User)

* **登录/注册**
    * `POST /api/auth/login`
    * **Request**: `{ "phone": "+8613800000000", "otp": "123456" }`
    * **Response**:
        ```json
        {
          "token": "eyJhbG...",
          "user_id": "uuid...",
          "is_onboarding_completed": false, // 审计修复: 用于前端判断跳转向导或首页
          "is_new_user": true
        }
```

* **获取画像**
    * `GET /api/user/profile`
    * **Response**:
        ```json
        {
          "gender": "FEMALE",
          "height_cm": 165.0, // 审计修复: 明确单位
          "weight_kg": 55.0,
          "cycle_length_days": 28,
          "is_research_active": true
        }
        ```

### 4.2 数据同步 (Data Sync)

* **批量上传健康数据**
    * `POST /api/health/sync`
    * **描述**: 端侧计算好 Ready/Stress 等指标后上传存档。
    * **Request**:
        ```json
        {
          "data": [
            {
              "date": "2025-05-20",
              "rhr": 65,                // 审计修复: 统一命名
              "stress_avg": 45,         // 审计修复: 统一命名
              "skin_temp_delta": 0.3,
              "daily_steps": 8500,
              "readiness_score": 88,
              "sleep_stages_summary": { "deep": 60, "light": 200 }
            }
          ]
        }
        ```

### 4.3 公益板块 (Impact API)
*审计修复：补全 PRD 中缺失的接口定义*

* **获取项目列表**
    * `GET /api/impact/projects`
    * **Response**:
        ```json
        [
          {
            "project_id": "EDU_001",
            "title": "Support Rural Girls Education",
            "org_name": "HerFuture Foundation",
            "image_url": "[https://cdn.herplus.com/img/p1.jpg](https://cdn.herplus.com/img/p1.jpg)",
            "target_amount": 50000.00,
            "current_amount": 12500.00,
            "status": "ACTIVE"
          }
        ]
        ```

* **注入能量 (捐赠)**
    * `POST /api/impact/donate`
    * **Request**: `{ "project_id": "EDU_001", "steps": 5000 }`
    * **Logic**: 后端按 `10000步 = $1` 换算，写入 `charity_contributions` 表，并更新 Project 进度。
    * **Response**: `{ "success": true, "donated_value": 0.50 }`

### 4.4 科研众包 (Research API)
*审计修复：补全 PRD 中缺失的接口定义*

* **获取研究列表**
    * `GET /api/research/studies`
    * **Response**:
        ```json
        [
          {
            "study_id": "SLEEP_MENOPAUSE_25",
            "title": "Sleep Patterns in Early Menopause",
            "institution": "Stanford Medicine",
            "required_data_types": ["sleep_stages", "skin_temp_delta"],
            "user_status": "NOT_ENROLLED" // 或 "ACTIVE"
          }
        ]
        ```

### 4.5 AI 对话 (AI Chat)

* **发送消息**
    * `POST /api/ai/chat`
    * **Request**:
        ```json
        {
          "session_type": "HOME", // 或 EXPERT
          "message": "我感觉有点焦虑",
          "context_snapshot": { "stress_avg": 85, "rhr": 78 } // 审计修复: 携带快照
        }
        ```
    * **Response**:
        ```json
        {
          "reply": "深呼吸，我一直在。要不要试着冥想一下？",
          "suggested_tags": ["好主意", "不想动", "为什么会这样"],
          "quota_remaining": 8
        }
        ```

---

## 5. 核心业务逻辑 (Business Logic)

### 5.1 Readiness 计算策略 (Hybrid Calculation)
* **端侧 (App)**: 负责实时计算。
    * *公式*: `Readiness = 0.4 * SleepScore + 0.4 * (100 - StressIndex) + 0.2 * RHR_Factor`
    * *原因*: 即使离线也能查看首页状态 (Moment)，保证体验流畅。
* **云端 (Cloud)**: 负责存储与校验。
    * APP 在同步时上传计算结果，云端仅做记录。若发现数据异常（如步数作弊），云端标记 `is_valid=false`。

### 5.2 Guardian SOS 实时信令
由于 HTTP 轮询延迟太高，Guardian 状态机采用 **WebSocket** 实现。

1.  **Trigger**: 指环检测到 `LongPress(5s)` -> APP 唤醒。
2.  **Connect**: APP 建立 WS 连接 `wss://api.herplus.com/guardian`.
3.  **Heartbeat**: 每 3s 发送 GPS 坐标包。
4.  **Fallback**: 若网络极差 (WS 连接失败)，自动降级为 **本地 SMS 发送** (利用系统 URI Scheme)。

### 5.3 数据隐私与脱敏 (The Data Gap)
针对 **科研数据 (Research Data)** 的处理流程：

1.  **User Consent**: 用户在 APP 点击“加入项目”，数据库标记 `is_research_active = true`。
2.  **ETL Job**: 每日凌晨 2:00，后台任务扫描已授权用户的数据。
3.  **Anonymization**:
    * 移除 `user_id`, `phone_number`。
    * 将 `dob` 泛化为年龄段 (e.g., "1990-05-20" -> "30-35 Age Group")。
    * 生成随机 `research_hash_id`。
4.  **Export**: 写入独立的科研数据库 (Research DB)，供机构 API 读取。

---

## 6. 安全设计 (Security Design)

### 6.1 数据加密
* **传输层**: 全站强制 **TLS 1.3** (HTTPS/WSS)。
* **存储层 (服务端)**: 敏感字段 (`phone_number`, `gps_logs`) 使用 **AES-256** 加密存储。
* **存储层 (本地)**: WatermelonDB 启用 **SQLCipher** 加密，密钥由 Keychain/Keystore 管理。

### 6.2 紧急求助 (SOS) 令牌
* 生成的 SOS 位置短链 (e.g., `her.plus/s/Ab3d9`) 对应一个 **Redis Key**。
* **TTL**: 设置为 1 小时。过期后 Key 自动删除，链接失效，确保用户历史轨迹不被永久暴露。

### 6.3 蓝牙安全
* **Bonding**: 首次配对强制使用 **Just Works** 配对流程。
* **Filter**: APP 仅响应 Service UUID 为 `HER_PLUS_SRV` 的广播包，防止恶意连接其他设备。

---

## 7. 部署架构 (Deployment)

* **Container**: Docker 容器化部署。
* **Orchestration**: Kubernetes (K8s) 或 AWS ECS。
* **CI/CD**: GitHub Actions -> 自动构建 Flutter IPA/APK 和后端 Docker Image。