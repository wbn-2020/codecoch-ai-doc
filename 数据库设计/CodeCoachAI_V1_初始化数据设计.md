# CodeCoachAI V1 初始化数据设计

## 1. 文档说明

| 项目 | 内容 |
|---|---|
| 文档名称 | CodeCoachAI V1 初始化数据设计 |
| 项目阶段 | V1 |
| 用途 | 记录 V1 本地演示和开发需要的基础初始化数据 |

本文档只包含 V1 初始化数据建议，不包含 V2 / V3 的文件、搜索、任务、通知等初始化数据。

---

## 2. 角色初始化

```sql
INSERT INTO sys_role (id, role_code, role_name, description, status)
VALUES
  (1, 'USER', '普通用户', '用户端普通用户', 1),
  (2, 'ADMIN', '管理员', '管理端管理员', 1);
```

---

## 3. 管理员账号初始化

密码必须使用加密后的密文，`{encrypted_password}` 是占位符。

```sql
INSERT INTO sys_user (id, username, password, nickname, email, status)
VALUES
  (1, 'admin', '{encrypted_password}', '系统管理员', 'admin@codecoachai.local', 1);

INSERT INTO sys_user_role (user_id, role_id)
VALUES
  (1, 2);
```

---

## 4. 题目分类初始化

```sql
INSERT INTO question_category (id, parent_id, category_name, category_code, sort_order, status)
VALUES
  (1, 0, 'Java 基础', 'JAVA_BASIC', 1, 1),
  (2, 0, '集合框架', 'COLLECTION', 2, 1),
  (3, 0, '多线程与并发', 'CONCURRENCY', 3, 1),
  (4, 0, 'JVM', 'JVM', 4, 1),
  (5, 0, 'Spring', 'SPRING', 5, 1),
  (6, 0, 'Spring Boot', 'SPRING_BOOT', 6, 1),
  (7, 0, 'MyBatis', 'MYBATIS', 7, 1),
  (8, 0, 'MySQL', 'MYSQL', 8, 1),
  (9, 0, 'Redis', 'REDIS', 9, 1),
  (10, 0, 'RabbitMQ', 'RABBITMQ', 10, 1),
  (11, 0, '微服务', 'MICROSERVICE', 11, 1),
  (12, 0, '分布式', 'DISTRIBUTED', 12, 1),
  (13, 0, '设计模式', 'DESIGN_PATTERN', 13, 1),
  (14, 0, '项目场景题', 'PROJECT_SCENARIO', 14, 1);
```

---

## 5. Prompt 模板初始化

```sql
INSERT INTO prompt_template (template_name, template_code, scene, template_content, variable_description, status)
VALUES
  ('八股文提问模板', 'INTERVIEW_QUESTION_GENERATE_DEFAULT', 'INTERVIEW_QUESTION_GENERATE', '你是一名 Java 技术面试官，请基于题目、目标岗位、经验年限和当前阶段生成一个自然的面试问题。每次只问一个问题。', '题目、目标岗位、经验年限、当前阶段', 1),
  ('项目深挖提问模板', 'PROJECT_DEEP_DIVE_QUESTION_DEFAULT', 'PROJECT_DEEP_DIVE_QUESTION', '你是一名擅长项目深挖的 Java 面试官，请基于用户简历项目经历生成一个项目追问问题。每次只问一个问题。', '简历项目、技术栈、个人职责、项目难点', 1),
  ('回答评分模板', 'INTERVIEW_ANSWER_EVALUATE_DEFAULT', 'INTERVIEW_ANSWER_EVALUATE', '请根据面试问题、参考答案和用户回答进行评分、点评，并判断是否需要追问。输出应包含分数、点评、缺失点、是否建议追问和追问方向。', '当前问题、参考答案、用户回答', 1),
  ('动态追问模板', 'INTERVIEW_FOLLOW_UP_GENERATE_DEFAULT', 'INTERVIEW_FOLLOW_UP_GENERATE', '请根据用户回答质量和追问方向生成一个追问问题。追问要具体，避免一次问多个问题。', '当前问题、用户回答、追问方向、历史问答', 1),
  ('面试报告模板', 'INTERVIEW_REPORT_GENERATE_DEFAULT', 'INTERVIEW_REPORT_GENERATE', '请根据本场面试的阶段、问答、评分和点评，生成结构化面试报告，包含总评、优势、问题、薄弱点和复习建议。', '面试配置、阶段得分、问答记录、AI点评', 1);
```

---

## 6. 系统配置初始化

```sql
INSERT INTO system_config (config_key, config_value, config_name, description, config_type, editable, status)
VALUES
  ('interview.max_follow_up_count', '2', '每题最大追问次数', '每个主问题最多追问次数', 'NUMBER', 1, 1),
  ('interview.max_question_count', '12', '每场最大问题数', '单场面试最大主问题数量', 'NUMBER', 1, 1),
  ('ai.timeout_seconds', '60', 'AI调用超时时间', 'AI同步调用超时时间，单位秒', 'NUMBER', 1, 1),
  ('ai.daily_limit_per_user', '50', '用户每日AI调用上限', '普通用户每日AI调用次数限制', 'NUMBER', 1, 1);
```

---

## 7. 初始化原则

1. 初始化数据只服务 V1 演示和开发。
2. 管理员密码必须在实际 SQL 中替换为加密密文。
3. Prompt 模板内容后续可在管理端调整。
4. 分类初始化后仍允许管理员维护。
5. 初始化数据不要包含真实用户隐私信息。
