# 第5周：JWT 认证与 RBAC 权限控制

> 基于 BME 教育平台实际项目代码讲解  
> 技术栈：Python Flask + Flask-JWT-Extended + Vue3 + Vuex + Element Plus

---

## 目录

1. [JWT 是什么](#1-jwt-是什么)
2. [JWT 的三个组成部分](#2-jwt-的三个组成部分)
3. [项目中的 JWT 配置与令牌生成](#3-项目中的-jwt-配置与令牌生成)
4. [为什么需要 RBAC](#4-为什么需要-rbac)
5. [项目中的 RBAC 数据模型](#5-项目中的-rbac-数据模型)
6. [后端权限校验装饰器](#6-后端权限校验装饰器)
7. [权限管理 API](#7-权限管理-api)
8. [前端 Token 流转机制](#8-前端-token-流转机制)
9. [完整链路：从登录到权限校验](#9-完整链路从登录到权限校验)
10. [学生端 vs 管理员后台对比](#10-学生端-vs-管理员后台对比)
11. [总结与最佳实践](#11-总结与最佳实践)

---

## 1. JWT 是什么

### 1.1 传统 Session 认证的问题

在传统的 Web 应用中，用户登录后服务器会创建一个 Session，把 Session ID 存在 Cookie 里，服务器内存中保存完整的用户登录状态：

```
浏览器                        服务器
  │                              │
  │──── POST /login ────────────>│  服务器生成 Session
  │<─── Set-Cookie: session_id ──│  内存中存储 user 对象
  │                              │
  │──── GET /api/users ─────────>│  读取 Session → 查内存
  │     Cookie: session_id       │
```

这种方式在**前后端分离**架构中会有几个问题：
- **服务器有状态**：Session 存在内存中，多台服务器需要共享 Session
- **跨域受限**：Cookie 在同源策略下很难跨域传递
- **不适合移动端**：原生 App 不用 Cookie

### 1.2 JWT 的核心思想

**JWT（JSON Web Token）** 的思路相反：**服务器不发 Session，而是发一张"令牌"，用户每次请求带着这张令牌就行。**

```
浏览器                        服务器
  │                              │
  │──── POST /login ────────────>│  验证密码
  │<─── { token: "eyJhbG..." } ──│  签发 JWT（服务器不存任何东西）
  │                              │
  │──── GET /api/users ─────────>│  验证 JWT 签名
  │     Authorization: Bearer    │  从 Token 里读出用户是谁
```

关键特性：
- **无状态**：服务器不需要存储任何会话数据
- **自包含**：Token 本身包含用户身份信息
- **跨域友好**：Token 放在 HTTP Header 里，不受同源策略限制

---

## 2. JWT 的三个组成部分

JWT 由三部分组成，用 `.` 分隔：

```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ1c2VyQGV4YW1wbGUuY29tIn0.abc123def456
│                      │                                │
│   Header（头部）      │   Payload（载荷）               │   Signature（签名）
```

### 2.1 Header（头部）

声明类型和签名算法：

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

### 2.2 Payload（载荷）

存放"声明"（claims）——也就是你想传递的数据：

```json
{
  "sub": "user@example.com",     // 用户标识
  "iat": 1715692800,             // 签发时间
  "exp": 1716297600              // 过期时间（7天后）
}
```

> **重要**：Payload 只是 Base64 编码，不是加密！任何人都能解码看到内容。所以**绝不能把密码放在 JWT 里**。

### 2.3 Signature（签名）

签名是 JWT 安全性的核心：

```
HMAC-SHA256(
    Base64(Header) + "." + Base64(Payload),
    SecretKey（只有服务器知道）
)
```

- 服务器用密钥对前两部分做哈希
- 如果有人在中间修改了 Payload（比如把自己改成管理员），签名就会对不上
- 服务器验证时发现签名不对 → 拒绝请求

**一句话总结：JWT 不加密数据，但保证数据不可篡改。**

---

## 3. 项目中的 JWT 配置与令牌生成

> 对应文件：[BME_platform_flask/config.py](../../Education_platform/BME_platform_flask/config.py)  
> 对应文件：[BME_platform_flask/app.py](../../Education_platform/BME_platform_flask/app.py)

### 3.1 JWT 配置

在 [config.py](../../Education_platform/BME_platform_flask/config.py#L25-L26) 中：

```python
JWT_SECRET_KEY = os.getenv("JWT_SECRET") or "your-secret-key-change-in-production"
JWT_ACCESS_TOKEN_EXPIRES = timedelta(days=7)
```

| 配置项 | 值 | 含义 |
|--------|-----|------|
| `JWT_SECRET_KEY` | 从环境变量读取 | JWT 签名用的密钥 — **绝不能泄露** |
| `JWT_ACCESS_TOKEN_EXPIRES` | 7 天 | Token 过期时间，过期后需重新登录 |

在 [app.py](../../Education_platform/BME_platform_flask/app.py#L29) 中初始化：

```python
jwt = JWTManager(app)
```

### 3.2 登录时生成 Token

在 [blueprints/auth.py](../../Education_platform/BME_platform_flask/blueprints/auth.py#L91-L142) 的登录接口中：

```python
@bp.route("/login", methods=["POST"])
def login():
    # ... 验证邮箱和密码 ...
    if user.password == password:
        # 关键：用邮箱作为 JWT 的 identity（身份标识）
        token = create_access_token(identity=email)
        return jsonify({
            "code": 200,
            "token": token,
            "User_Name": user.username,
            "User_Mode": user.user_mode,  # 角色信息也在响应中返回
        })
```

**这里有一个重要设计选择**：JWT 的 `identity` 存的是**用户邮箱**，而不是用户 ID。这意味着后续每个需要身份验证的接口，都要通过邮箱去查数据库才能拿到用户的角色和权限。

### 3.3 注册时也生成 Token

在 [blueprints/auth.py](../../Education_platform/BME_platform_flask/blueprints/auth.py#L30-L82) 的注册接口中：

```python
@bp.route("/register", methods=["POST"])
def register():
    # ... 验证表单和验证码 ...
    user = UserModel(email=email, password=password, username=username)
    db.session.add(user)
    db.session.commit()
    # 注册成功直接签发 JWT，实现"注册即登录"
    token = create_access_token(identity=email)
    return jsonify({"code": 200, "token": token})
```

### 3.4 后端如何获取当前用户

在任何受保护的接口中，通过 `get_jwt_identity()` 获取身份：

```python
from flask_jwt_extended import jwt_required, get_jwt_identity

@bp.route("/some-protected-api")
@jwt_required()  # ← 这个装饰器验证 Token 有效性
def protected():
    email = get_jwt_identity()  # ← 取出 Token 中的邮箱
    user = UserModel.query.filter_by(email=email).first()  # 查数据库拿完整用户信息
```

---

## 4. 为什么需要 RBAC

### 4.1 简单二元角色的问题

如果系统只有"管理员"和"普通用户"两种角色，权限判断很简单：

```python
if user.user_mode == 'admin':
    # 什么都能做
else:
    # 受限制
```

BME 项目中的 `user_mode` 字段就是这种模式：

> 代码位置：[models.py](../../Education_platform/BME_platform_flask/models.py#L22)

```python
user_mode = db.Column(db.String(20), default='user')  # 'user' 或 'admin'
```

在前端组件中也是这样判断的：

> 代码位置：[MenuComponent.vue](../../Education_platform/BME_frontend/src/components/MenuComponent.vue#L288-L289)

```html
<div v-if="$store.state.user.User_Mode == 'admin'" class="user-type-instructor">导师</div>
<div v-else class="user-type-student">学生</div>
```

**这种方式的问题**：当角色变多时（如"课程编辑"、"勋章管理员"、"审计查看者"），代码会变成到处都是 `if`，无法维护。

### 4.2 RBAC 的核心理念

**RBAC = Role-Based Access Control = 基于角色的访问控制**

核心模型是三张表：

```
┌──────────┐         ┌──────────────────────┐         ┌──────────────┐
│  用户     │  多对多  │  用户-权限关联表      │  多对多  │  权限         │
│  User    │────────>│  UserPermission      │<────────│  Permission   │
└──────────┘         └──────────────────────┘         └──────────────┘
```

权限不直接分配给用户，而是通过**权限标识**来桥接：

```
用户"李四" → 拥有权限 "course_management" → 可以管理课程
用户"王五" → 拥有权限 "article_management" → 可以管理文章
用户"赵六" → 拥有权限 "course_management", "article_management" → 全能编辑
```

### 4.3 BME 项目中的实际权限

从权限管理 API 的实际使用中，可以看到系统定义了这些权限：

| 权限名称                 | 用途                | 使用位置                                         |
| -------------------- | ----------------- | -------------------------------------------- |
| `system_management`  | 系统管理（分配权限、查看审计日志） | `/permissions/assign`, `/auth/audit_records` |
| `user_management`    | 用户管理（查看用户列表、管理小组） | `/permissions/user/<id>`, `/user/user_list`  |
| `course_management`  | 课程管理（创建/编辑/删除课程）  | `/course/edit`, `/course/delete`             |
| `article_management` | 文章管理（发布/编辑/删除文章）  | `/article/edit`, `/article/delete`           |
| `medal_management`   | 勋章管理（创建/编辑勋章）     | `/medal/medal_create`                        |

---

## 5. 项目中的 RBAC 数据模型

> 对应文件：[models.py](../../Education_platform/BME_platform_flask/models.py#L697-L715)

### 5.1 三张核心表

**权限表 — 定义系统中有哪些权限**：

```python
class PermissionModel(db.Model):
    __tablename__ = 'permission'
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(50), nullable=False, unique=True)   # 权限名称
    description = db.Column(db.String(200))                         # 权限描述
```

**用户表 — 使用已有的 UserModel**：

```python
class UserModel(db.Model):
    __tablename__ = 'user'
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(100), nullable=False)
    email = db.Column(db.String(100), nullable=False, unique=True)
    # ...
    user_mode = db.Column(db.String(20), default='user')  # 保留：向后兼容
```

**用户-权限关联表 — 多对多的桥接表**：

```python
class UserPermissionModel(db.Model):
    __tablename__ = 'user_permission'
    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)
    permission_id = db.Column(db.Integer, db.ForeignKey('permission.id'), nullable=False)

    # 保证同一个权限不会重复分配给同一个用户
    __table_args__ = (db.UniqueConstraint('user_id', 'permission_id'),)

    # 双向关联，方便查询
    user = db.relationship('UserModel', backref=db.backref('user_permissions', lazy=True))
    permission = db.relationship('PermissionModel', backref=db.backref('user_permissions', lazy=True))
```

### 5.2 数据关系图示

```
数据库中的数据示例：

permission 表                          user_permission 表
┌────┬────────────────────┐           ┌────┬──────────┬───────────────┐
│ id │ name               │           │ id │ user_id  │ permission_id │
├────┼────────────────────┤           ├────┼──────────┼───────────────┤
│ 1  │ system_management  │           │ 1  │ 2 (李四) │ 2 (user_mgmt) │
│ 2  │ user_management    │           │ 2  │ 2 (李四) │ 4 (article)   │
│ 3  │ course_management  │           │ 3  │ 3 (王五) │ 3 (course)    │
│ 4  │ article_management │           │ 4  │ 1 (张三) │ 1 (system)    │
│ 5  │ medal_management   │           └────┴──────────┴───────────────┘
└────┴────────────────────┘
```

- **张三** → `system_management` → 可以管理权限和审计日志
- **李四** → `user_management` + `article_management` → 可以管理用户和文章
- **王五** → `course_management` → 只能管理课程

---

## 6. 后端权限校验装饰器

> 对应文件：[blueprints/\_\_init\_\_.py](../../Education_platform/BME_platform_flask/blueprints/__init__.py)

这是整个 RBAC 系统的核心。

### 6.1 单权限检查

```python
def check_permission(permission_name):
    """
    权限检查装饰器
    用法: @check_permission('course_management')
    """
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            # ① 从 JWT 中获取当前用户邮箱
            user_email = get_jwt_identity()
            user = UserModel.query.filter_by(email=user_email).first()

            if not user:
                return jsonify({"code": 401, "message": "用户未认证"}), 401

            # ② admin 模式直接放行（向后兼容）
            if user.user_mode == 'admin':
                return func(*args, **kwargs)

            # ③ 从数据库查权限记录
            permission = PermissionModel.query.filter_by(name=permission_name).first()
            if not permission:
                return jsonify({"code": 403, "message": f"权限 '{permission_name}' 不存在"}), 403

            # ④ 检查用户是否拥有该权限
            user_permission = UserPermissionModel.query.filter_by(
                user_id=user.id,
                permission_id=permission.id
            ).first()

            if not user_permission:
                return jsonify({"code": 403, "message": "用户权限不足"}), 403

            # ⑤ 权限通过，执行原函数
            return func(*args, **kwargs)
        return wrapper
    return decorator
```

**执行流程**：

```
请求进入
  │
  ├─ ① @jwt_required()       → Token 有没有？有没有过期？
  │                              失败 → 401
  │
  ├─ ② get_jwt_identity()    → 从 Token 拿出邮箱，查出用户
  │
  ├─ ③ user_mode == 'admin'? → 是 → 直接放行（超级管理员绕过）
  │
  └─ ④ 查 user_permission 表  → 有没有这个权限？
                                    有 → 放行
                                    没有 → 403
```

### 6.2 多权限检查（AND / OR）

```python
def check_multiple_permissions(permission_names, require_all=False):
    """
    多权限检查装饰器
    require_all=True  → 需要全部权限（AND）
    require_all=False → 只需要其中一个（OR）
    """
```

**使用示例**：

```python
# 要求同时拥有系统管理和用户管理权限
@check_multiple_permissions(['system_management', 'user_management'], require_all=True)

# 要求拥有课程管理或文章管理之一即可
@check_multiple_permissions(['course_management', 'article_management'], require_all=False)
```

### 6.3 实际使用案例

以审计日志接口为例：

> 代码位置：[blueprints/auth.py](../../Education_platform/BME_platform_flask/blueprints/auth.py#L340-L342)

```python
@bp.route("/audit_records", methods=["GET"])
@jwt_required()                          # ① 先验证 JWT
@check_permission('system_management')     # ② 再检查是否有 system_management 权限
def get_admin_audit_logs():
    # 只有通过了上面两道关卡，才会执行到这里
    ...
```

以权限分配接口为例：

> 代码位置：[blueprints/permissionACL.py](../../Education_platform/BME_platform_flask/blueprints/permissionACL.py#L34-L38)

```python
@bp.route("/assign", methods=["POST"])
@jwt_required()
@check_permission('system_management')     # 只有系统管理员才能给别人分配权限
@audit_log(operation="分配用户权限")        # 记录审计日志
def assign_permission():
    ...
```

### 6.4 装饰器的执行顺序

Flask/Python 装饰器是**从下往上**执行的：

```python
@bp.route("/audit_records", methods=["GET"])    # ④ 最后：路由注册
@jwt_required()                                  # ③ 然后：JWT 验证
@check_permission('system_management')            # ② 接着：权限检查
def get_admin_audit_logs():                       # ① 最先：原始函数
```

等价于执行顺序：

```
请求 → 匹配路由 → jwt_required() → check_permission() → 原始函数
```

---

## 7. 权限管理 API

> 对应文件：[blueprints/permissionACL.py](../../Education_platform/BME_platform_flask/blueprints/permissionACL.py)

系统提供了完整的权限 CRUD 管理接口：

### 7.1 API 全览

| 接口 | 方法 | 需要权限 | 功能 |
|------|------|---------|------|
| `/permissions/list` | GET | 仅需登录 | 查看系统中所有权限列表 |
| `/permissions/assign` | POST | `system_management` | 给用户分配权限 |
| `/permissions/revoke` | POST | `system_management` | 撤销用户的权限 |
| `/permissions/user/self` | GET | 仅需登录 | 查看自己的权限 |
| `/permissions/user/<id>` | GET | `user_management` | 查看任意用户的权限 |

### 7.2 分配权限

```python
@bp.route("/assign", methods=["POST"])
@jwt_required()
@check_permission('system_management')
def assign_permission():
    data = request.get_json()
    user_id = data.get('user_id')
    permission_name = data.get('permission_name')   # 支持通过名称指定，更友好
    
    # 查权限对象
    permission = PermissionModel.query.filter_by(name=permission_name).first()
    
    # 检查是否重复分配
    existing = UserPermissionModel.query.filter_by(
        user_id=user_id, permission_id=permission.id
    ).first()
    
    if existing:
        return jsonify({"code": 400, "message": "用户已经拥有该权限"}), 400
    
    # 创建关联
    user_permission = UserPermissionModel(user_id=user_id, permission_id=permission.id)
    db.session.add(user_permission)
    db.session.commit()
```

### 7.3 撤销权限

```python
@bp.route("/revoke", methods=["POST"])
@jwt_required()
@check_permission('system_management')
def revoke_permission():
    # 查找并删除用户-权限关联记录
    user_permission = UserPermissionModel.query.filter_by(
        user_id=user_id, permission_id=permission.id
    ).first()
    
    if not user_permission:
        return jsonify({"code": 400, "message": "用户没有该权限"}), 400
    
    db.session.delete(user_permission)
    db.session.commit()
```

---

## 8. 前端 Token 流转机制

前端的 JWT 处理涉及三个核心环节：**登录存储 → 请求携带 → 过期处理**。

### 8.1 登录：获取并存储 Token

**学生端** [LoginComponent.vue](../../Education_platform/BME_frontend/src/components/Auth/LoginComponent.vue#L154-L158)：

```javascript
const res = await api.post('/auth/login', {
    User_Email: loginForm.value.email,
    User_Password: md5(loginForm.value.password),  // 前端 MD5 哈希
})

if (res.data.code === 200) {
    localStorage.setItem('token', res.data.token)   // ← 存到 localStorage
    store.commit('setUser', res.data)               // ← 用户信息存 Vuex
    router.push('/home')                            // ← 跳转首页
}
```

**管理员后台** [LoginComponent.vue](../../Education_platform/BME_backend/src/components/LoginComponent.vue#L59-L69)：

```javascript
// 管理员用单独的登录接口
const res = await api.post('/auth/admin_login', {
    User_Email: this.loginForm.email,
    User_Password: md5(this.loginForm.password),
})

if (res.data.code == 200) {
    localStorage.setItem("token", res.data.token)
    this.store.commit('setUser', res.data)
    this.$router.push('/')
}
```

### 8.2 每次请求自动带 Token — Axios 拦截器

**学生端** [api.js](../../Education_platform/BME_frontend/src/api.js#L15-L23)：

```javascript
// 请求拦截器：每次请求自动附加 Token
api.interceptors.request.use(config => {
    const token = localStorage.getItem('token')
    if (token) {
        config.headers['Authorization'] = `Bearer ${token}`
    }
    return config
}, error => {
    return Promise.reject(error)
})
```

**管理员后台** [api.js](../../Education_platform/BME_backend/src/api.js#L21-L29)：

```javascript
// 一模一样的方式 —— 代码完全类似
api.interceptors.request.use(config => {
    const token = localStorage.getItem('token')
    if (token) {
        config.headers['Authorization'] = `Bearer ${token}`
    }
    return config
})
```

### 8.3 Token 过期处理 — 响应拦截器

**学生端** [api.js](../../Education_platform/BME_frontend/src/api.js#L25-L47)（有完整处理）：

```javascript
// 响应拦截器：捕获 401 错误
api.interceptors.response.use(
    response => response,
    error => {
        if (error.response && error.response.status === 401) {
            if (!window.__hasShownLoginExpire) {
                window.__hasShownLoginExpire = true  // 防止重复弹窗
                ElMessage.error('登录失效，请重新登录')
                localStorage.removeItem('token')
                setTimeout(() => {
                    if (window.location.pathname !== '/login') {
                        window.location.href = '/login'
                    }
                    window.__hasShownLoginExpire = false
                }, 1000)
            }
        }
        return Promise.reject(error)
    }
)
```

**要点解读**：
- `window.__hasShownLoginExpire` 是一个简易的防抖机制——多个接口同时返回 401 时只弹一次提示
- 先清除 `localStorage` 中的 token，再跳转到登录页
- 延迟 1 秒跳转，保证用户能看到提示信息

**管理员后台** [api.js](../../Education_platform/BME_backend/src/api.js)——**没有响应拦截器处理 401！** 这是一个值得改进的点（见第 10 节）。

### 8.4 路由守卫：未登录不能进

**学生端** [router.js](../../Education_platform/BME_frontend/src/router.js#L214-L226)（有路由守卫）：

```javascript
router.beforeEach((to, from, next) => {
    const token = localStorage.getItem('token')

    if (to.meta.requiresAuth && !token) {
        // 未登录 → 重定向到登录页，附带 redirect 参数
        next({ name: 'login', query: { redirect: to.fullPath } })
    } else if (token && (to.name === 'login' || to.name === 'register')) {
        // 已登录 → 不允许再访问登录/注册页
        next({ name: 'home' })
    } else {
        next()
    }
})
```

路由配置中通过 `meta: { requiresAuth: true }` 标记需要登录的页面：

```javascript
{
    path: '/home',
    name: 'home',
    component: HomeView,
    meta: { requiresAuth: true }   // ← 标记需要登录
},
{
    path: '/login',
    name: 'login',
    component: LoginView
    // 没有 meta.requiresAuth → 公开页面
}
```

**管理员后台** [router.js](../../Education_platform/BME_backend/src/router.js)——**没有路由守卫！** 任何人可以直接在地址栏输入管理后台路径（见第 10 节）。

### 8.5 退出登录

**学生端** [MenuComponent.vue](../../Education_platform/BME_frontend/src/components/MenuComponent.vue#L190-L194)：

```javascript
const logOut = () => {
    store.dispatch('logout')             // 清除 Vuex 中的用户数据
    localStorage.removeItem('token')     // 清除 Token
    window.location.reload()             // 刷新页面（清除所有状态）
}
```

### 8.6 Vuex Store 中的认证状态

**学生端** [store.js](../../Education_platform/BME_frontend/src/store.js)：

```javascript
export default new Vuex.Store({
    state: {
        user: null,
        token: localStorage.getItem('token') || null,  // 初始化时从 localStorage 读取
    },
    getters: {
        isLogin: (state) => !!state.token,  // 有 token 就认为已登录
    },
    plugins: [
        VuexPersist({                          // 持久化插件
            key: 'my-app',
            storage: window.localStorage,      // 整个 Vuex 状态存 localStorage
        })
    ]
})
```

---

## 9. 完整链路：从登录到权限校验

这里我们用 BME 项目的真实代码，完整追踪一次"管理员查看审计日志"的请求。

### 9.1 第一步：管理员登录

**管理员后台**发起 POST `/auth/admin_login`：

```
POST /auth/admin_login
Content-Type: application/json

{
    "User_Email": "admin@example.com",
    "User_Password": "md5哈希后的密码"
}
```

**后端** [auth.py](../../Education_platform/BME_platform_flask/blueprints/auth.py#L193-L247) 处理：

```python
def admin_login():
    admin = UserModel.query.filter_by(email=email).first()

    # 额外检查：管理员必须拥有权限记录或 user_mode='admin'
    user_permission = UserPermissionModel.query.filter_by(user_id=admin.id).first()
    if not user_permission and admin.user_mode != 'admin':
        return jsonify({"code": 401, 'message': "用户权限不够"}), 401

    # 签发 JWT
    token = create_access_token(identity=email)
    return jsonify({'code': 200, 'token': token})
```

### 9.2 第二步：前端存 Token

```javascript
// 管理员后台 LoginComponent.vue
localStorage.setItem("token", res.data.token)
this.store.commit('setUser', res.data)
this.$router.push('/')    // 进入管理后台首页
```

### 9.3 第三步：发起受保护的请求

管理员在后台点击"审计日志"，前端发起请求：

```
GET /auth/audit_records?page=1&per_page=10
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...  ← 拦截器自动从 localStorage 读取并附加
```

### 9.4 第四步：后端两层验证

```python
@bp.route("/audit_records", methods=["GET"])
@jwt_required()                          # 第一关：验证 JWT
@check_permission('system_management')     # 第二关：检查权限
def get_admin_audit_logs():
    # 到这里说明两关都过了
    logs = AuditLog.query.order_by(...).paginate(...)
    return jsonify({"code": 200, "data": {...}})
```

**第一关 `@jwt_required()`** 做什么：
1. 从 `Authorization` 头提取 Token
2. 用 `JWT_SECRET_KEY` 验证签名 → 防止篡改
3. 检查 `exp` 是否过期 → 7 天内有效
4. 全部通过后，`get_jwt_identity()` 可获取邮箱

**第二关 `@check_permission('system_management')`** 做什么：
1. 从 Token 取出邮箱 → 查出 User 对象
2. 检查 `user_mode == 'admin'`？是则直接放行
3. 查 `permission` 表 → 找到 `system_management` 的记录
4. 查 `user_permission` 表 → 检查该用户是否拥有此权限
5. 有 → 放行，没有 → 返回 403

### 9.5 场景：如果被拒绝

假设一个普通学生尝试查看审计日志：

```
GET /auth/audit_records
Authorization: Bearer <学生Token>

后端响应：
HTTP 403 Forbidden
{
    "code": 403,
    "message": "用户权限不足"
}
```

前端 403 处理流程：
1. 后端返回 403（不是 401），所以**不会触发**登录过期逻辑
2. 前端组件收到 403，通常显示"无权限"提示
3. 用户仍然是登录状态，只是不能执行这个操作

**这里有个重要区别**：
- **401 Unauthorized** = "你是谁？我没认出来"（Token 无效/过期）
- **403 Forbidden** = "我知道你是谁，但你不能做这件事"（没有权限）

---

## 10. 学生端 vs 管理员后台对比

> 学生端项目：[BME_frontend](../../Education_platform/BME_frontend/)  
> 管理员后台项目：[BME_backend](../../Education_platform/BME_backend/)

### 10.1 对比表

| 能力 | 学生端 | 管理员后台 |
|------|--------|----------|
| Axios 请求拦截器（自动带 Token） | ✅ 有 | ✅ 有 |
| 响应拦截器（401 处理） | ✅ 有（防抖、延迟跳转） | ❌ 没有 |
| 路由守卫（beforeEach） | ✅ 有 | ❌ **没有！** |
| `meta.requiresAuth` 标记 | ✅ 有 | ❌ 没有 |
| 角色 UI 控制 | ✅ `v-if="User_Mode=='admin'"` | ❌ 没有 |
| 登录接口 | `/auth/login` | `/auth/admin_login`（额外验证权限记录） |

### 10.2 管理员后台缺少路由守卫

**问题**：管理员后台 [router.js](../../Education_platform/BME_backend/src/router.js) 中没有 `beforeEach` 守卫。

这意味着：
- 任何人直接在浏览器输入 `/admin/user-manage/users` 就能看到管理后台的**布局框架**
- 虽然 API 请求会因为没 Token 而失败，但页面结构会暴露
- 用户体验也不好——会看到空白页面而不是被引导去登录

**改进建议**（可在教学中讨论）：

```javascript
// 应该在管理员后台也添加路由守卫
router.beforeEach((to, from, next) => {
    const token = localStorage.getItem('token')
    if (to.path !== '/login' && to.path !== '/register' && !token) {
        next({ name: 'login' })
    } else if (token && to.name === 'login') {
        next({ name: 'home' })
    } else {
        next()
    }
})
```

### 10.3 两个前端共用同一套后端权限体系

关键认知：**前端只是 UI 层面的控制，真正的安全边界在后端**。

```
                         ┌────────────────────┐
                         │   Flask 后端        │
                         │   @jwt_required()   │
              ┌─────────>│   @check_permission │
              │          └────────────────────┘
              │                    │
              │                    │
     ┌────────┴──────┐    ┌───────┴─────────┐
     │ 学生端前台     │    │ 管理员后台       │
     │ (BME_frontend) │    │ (BME_backend)   │
     │                │    │                 │
     │ /auth/login    │    │ /auth/admin_    │
     │                │    │     login       │
     │ 有路由守卫 ✅   │    │ 无路由守卫 ❌    │
     └───────────────┘    └─────────────────┘
```

前端做权限控制是为了**用户体验**（不该点的按钮不显示），后端做权限控制是为了**安全**（不该做的操作做不了）。

---

## 11. 总结与最佳实践

### 11.1 核心知识点回顾

| 层级 | 技术 | 在本项目中的位置 |
|------|------|----------------|
| JWT 配置 | `JWT_SECRET_KEY`, 7天过期 | [config.py](../../Education_platform/BME_platform_flask/config.py#L25-L26) |
| JWT 签发 | `create_access_token(identity=email)` | [auth.py](../../Education_platform/BME_platform_flask/blueprints/auth.py#L121) |
| JWT 验证 | `@jwt_required()` + `get_jwt_identity()` | 所有受保护接口 |
| RBAC 模型 | `PermissionModel` + `UserPermissionModel` | [models.py](../../Education_platform/BME_platform_flask/models.py#L697-L715) |
| 权限检查 | `@check_permission('xxx')` | [blueprints/\_\_init\_\_.py](../../Education_platform/BME_platform_flask/blueprints/__init__.py#L175-L218) |
| 多权限检查 | `@check_multiple_permissions([...])` | [blueprints/\_\_init\_\_.py](../../Education_platform/BME_platform_flask/blueprints/__init__.py#L221-L282) |
| 前端 Token 存储 | `localStorage.setItem('token', ...)` | [LoginComponent.vue](../../Education_platform/BME_frontend/src/components/Auth/LoginComponent.vue#L155) |
| 前端 Token 携带 | Axios 请求拦截器 `Authorization: Bearer` | [api.js](../../Education_platform/BME_frontend/src/api.js#L15-L23) |
| 前端 Token 过期 | Axios 响应拦截器 401 处理 | [api.js](../../Education_platform/BME_frontend/src/api.js#L25-L47) |
| 前端路由守卫 | `router.beforeEach` + `meta.requiresAuth` | [router.js](../../Education_platform/BME_frontend/src/router.js#L214-L226) |
| 审计日志 | `@audit_log` 装饰器 | [blueprints/\_\_init\_\_.py](../../Education_platform/BME_platform_flask/blueprints/__init__.py#L11-L118) |

### 11.2 设计精华点

1. **admin 模式兜底**：`user_mode == 'admin'` 自动绕过所有权限检查，保证超级管理员始终可用
2. **权限用字符串名而非数字 ID**：`check_permission('course_management')` 比 `check_permission(3)` 可读性好得多
3. **支持 AND/OR 权限组合**：`require_all=True/False` 灵活应对复杂场景
4. **完整的审计链路**：每个敏感操作都记录操作人、IP、时间、结果
5. **前端防抖 401**：`__hasShownLoginExpire` 标志防止同时多个接口返回 401 时重复弹窗
6. **redirect 参数**：登录后自动跳回原来的目标页面

### 11.3 值得改进的点（教学中可作为思考题）

- **密码安全问题**：密码在前端用 MD5 哈希后传给后端，后端明文存储 — 应该用 bcrypt 在后端加盐哈希
- **JWT 无自定义 claims**：Token 中只存了邮箱，每次请求都要查 3 次数据库（user → permission → user_permission）— 可以把角色信息编码到 JWT 的 claims 中
- **管理员后台缺少路由守卫**：第 10.2 节已分析
- **Token 存储在 localStorage**：容易受 XSS 攻击 — 更安全的做法是 HttpOnly Cookie，但会增加复杂性

---

> **下节课预告**：第6周 — Web 安全（SQL 注入、XSS、CSRF）与 pytest 自动化测试
