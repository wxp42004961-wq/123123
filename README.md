# 驾校科二自动分车管理系统（完整开发方案）

## 1. 数据库设计

### 1.1 用户表 users

```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(50) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL,
    role ENUM('admin','staff','coach') DEFAULT 'staff',
    nickname VARCHAR(50),
    status TINYINT DEFAULT 1,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

### 1.2 教练表 coaches

```sql
CREATE TABLE coaches (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(50) NOT NULL,
    phone VARCHAR(20),
    annual_quota INT DEFAULT 0,
    current_assigned INT DEFAULT 0,
    remain_slots INT DEFAULT 0,
    ratio DECIMAL(10,4) DEFAULT 0,
    enabled TINYINT DEFAULT 1,
    remark TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

### 1.3 学员表 students

```sql
CREATE TABLE students (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(50) NOT NULL,
    alias_name VARCHAR(50),
    phone VARCHAR(20),
    gender ENUM('男','女'),
    age INT,
    class_type VARCHAR(50),
    referrer VARCHAR(50),
    student_type ENUM('VIP','WORKER','NORMAL') DEFAULT 'NORMAL',
    fixed_coach_id BIGINT,
    appointed_coach_id BIGINT,
    assigned_coach_id BIGINT,
    lock_status TINYINT DEFAULT 0,
    no_adjust TINYINT DEFAULT 0,
    blacklist_status TINYINT DEFAULT 0,
    note TEXT,
    import_batch VARCHAR(100),
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

### 1.4 禁忌规则表 forbidden_rules

```sql
CREATE TABLE forbidden_rules (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    rule_type ENUM('REFERRER','STUDENT_TYPE','SPECIAL','BLACKLIST'),
    referrer_name VARCHAR(50),
    student_type VARCHAR(50),
    coach_id BIGINT,
    student_id BIGINT,
    enabled TINYINT DEFAULT 1,
    remark TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

### 1.5 分车记录表 assignment_records

```sql
CREATE TABLE assignment_records (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    batch_no VARCHAR(100),
    student_id BIGINT,
    coach_id BIGINT,
    assign_reason VARCHAR(255),
    pool_type VARCHAR(50),
    operator_id BIGINT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

### 1.6 分车日志表 assignment_logs

```sql
CREATE TABLE assignment_logs (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    batch_no VARCHAR(100),
    operation_type VARCHAR(50),
    operation_content TEXT,
    operator_id BIGINT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

### 1.7 导出记录表 export_records

```sql
CREATE TABLE export_records (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    export_type VARCHAR(50),
    file_url VARCHAR(255),
    operator_id BIGINT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

### 1.8 错别字别名映射表 name_aliases

```sql
CREATE TABLE name_aliases (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    wrong_name VARCHAR(50),
    correct_name VARCHAR(50),
    similarity DECIMAL(5,2),
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

## 2. ER图

```text
users
  └── assignment_logs

coaches
  ├── students
  ├── forbidden_rules
  └── assignment_records

students
  ├── assignment_records
  └── forbidden_rules

assignment_records
  └── assignment_logs
```

## 3. API设计

### 3.1 登录模块

- `POST /api/auth/login`

### 3.2 教练模块

- `GET /api/coaches`
- `POST /api/coaches`
- `PUT /api/coaches/:id`
- `DELETE /api/coaches/:id`

### 3.3 学员模块

- `GET /api/students`
- `POST /api/students/import`
- `GET /api/students/export`
- `PUT /api/students/:id`

### 3.4 自动分车模块

- `POST /api/assignments/auto`
- `POST /api/assignments/manual`
- `POST /api/assignments/revoke`
- `GET /api/assignments/result`

### 3.5 禁忌规则模块

- `POST /api/rules`
- `GET /api/rules`

### 3.6 导出模块

- `GET /api/export/summary`
- `GET /api/export/report`

## 4. 分车算法设计

### 4.1 核心算法流程

```text
加载教练数据
    ↓
计算招生占比
    ↓
加载学员池
    ↓
固定学员优先分配
    ↓
指定教练学员分配
    ↓
VIP池按比例分配
    ↓
上班族池按比例分配
    ↓
普通池补齐
    ↓
人数平衡修正
    ↓
禁忌规则校验
    ↓
重复遗漏校验
    ↓
生成结果
```

### 4.2 招生占比算法

```js
ratio = coach.annual_quota / totalQuota
targetCount = Math.round(totalStudents * ratio)
```

### 4.3 分层池算法

```js
const vipPool = students.filter(s => s.student_type === 'VIP')
const workerPool = students.filter(s => s.student_type === 'WORKER')
const normalPool = students.filter(s => s.student_type === 'NORMAL')
```

### 4.4 禁忌规则过滤

```js
function isForbidden(student, coach, rules) {
  return rules.some(rule => {
    return (
      rule.referrer_name === student.referrer &&
      rule.coach_id === coach.id
    )
  })
}
```

### 4.5 自动平衡算法

```js
while (maxCoach.count - minCoach.count > 1) {
  moveStudent(maxCoach, minCoach)
}
```

### 4.6 自动兜底算法

```js
if (!assigned) {
  assignToLeastLoadedCoach(student)
}
```

### 4.7 错别字匹配

推荐：Levenshtein Distance、拼音模糊匹配、中文同音词库。

## 5. 页面原型设计

- 登录页
- 数据看板
- 教练管理
- 学员管理
- 自动分车页面
- 分车结果页
- 规则配置页
- 日志中心
- 导出中心

## 6. 后端代码结构

```text
server
├── app.js
├── routes
├── controllers
├── services
├── models
├── middlewares
├── utils
├── config
└── uploads
```

## 7. 前端代码结构

```text
src
├── api
├── views
├── components
├── layouts
├── router
├── store
├── utils
└── styles
```

## 8. Docker部署方案

- Node 20 构建后端镜像
- Node 20 + Nginx 构建前端镜像
- docker-compose 编排 frontend/backend/mysql

## 9. 推荐优化方案

- 微信小程序
- 多驾校 SaaS 版
- AI 智能推荐教练
- 自动语音通知
- 微信通知
- 人脸签到
- AI 风险规则检测

## 10. 推荐技术栈增强

**前端**：Pinia、Axios、ECharts、VueUse  
**后端**：Prisma ORM、Redis、BullMQ、Winston  
**部署**：Nginx、PM2、Jenkins、GitHub Actions

## 11. 核心算法建议（生产级）

建议采用：**固定优先 + 动态权重 + 分层池 + 禁忌过滤 + 负载均衡**。

## 12. MVP开发顺序（建议）

**第一阶段**：登录、教练管理、学员导入、自动分车、结果展示。  
**第二阶段**：禁忌规则、手动调人、导出、校验报告。  
**第三阶段**：SaaS 多校区、微信小程序、AI 智能优化。
