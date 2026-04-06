## 1. 安装依赖

使用 `PyJWT` 库（推荐）：

- pip install PyJWT
- pip install cryptography

本文以最常用的 HS256（对称加密） 为例。

---

## 2. 生成 JWT（签发 Token）

```
import jwt
import datetime
# 密钥（生产环境应使用安全的密钥，并存储在环境变量中）
SECRET_KEY = "your-secret-key-here"
def generate_jwt_token(user_id: int, username: str, expires_in_hours: int = 1):
    """
     生成 JWT Token
     :param user_id: 用户ID
     :param username: 用户名
     :param expires_in_hours: Token 有效期（小时）
     :return: 编码后的 JWT 字符串
     """
    payload = {
        "user_id": user_id,
        "username": username,
        # 标准声明（Registered Claims）
        "exp": datetime.datetime.utcnow() + datetime.timedelta(hours=expires_in_hours), # 过期时间
        "iat": datetime.datetime.utcnow(), # 签发时间
        "iss": "my-app", # 签发者
    }
    token = jwt.encode(payload, SECRET_KEY, algorithm="HS256")
    return token
```

---

## 3. 验证 JWT（解析并验证）

```
def verify_jwt_token(token: str):
    """
     验证并解码 JWT Token
     :param token: JWT 字符串
     :return: payload 字典（如果验证成功）
     :raises: jwt.ExpiredSignatureError, jwt.InvalidTokenError 等
     """
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
        return payload
    except jwt.ExpiredSignatureError:
        raise Exception("Token 已过期")
    except jwt.InvalidTokenError:
        raise Exception("无效的 Token")
```

---

## 4. 完整示例：登录 → 获取 Token → 验证 Token

```
# 模拟用户登录成功
 user_id = 123
 username = "alice"
# 1. 生成 Token（有效期1小时）
 token = generate_jwt_token(user_id, username, expires_in_hours=1)
 print("✅ 生成的 Token:", token)
# 2. 验证 Token
try:
    payload = verify_jwt_token(token)
    print("✅ Token 验证成功，用户信息:", payload)
except Exception as e:
    print("❌ Token 验证失败:", str(e))
# 3. 模拟使用过期 Token（等待1秒后伪造过期）
import time
fake_expired_token = jwt.encode({
    "user_id": 123,
    "username": "alice",
    "exp": datetime.datetime.utcnow() - datetime.timedelta(seconds=10), # 已过期
    "iat": datetime.datetime.utcnow() - datetime.timedelta(hours=1),
     }, SECRET_KEY, algorithm="HS256")
try:
    verify_jwt_token(fake_expired_token)
except Exception as e:
    print("❌ 过期 Token 测试:", str(e))
```

---

## 5. 自定义声明（Custom Claims）

除了标准声明（如 `exp`, `iat`, `iss`），你还可以添加任意自定义字段：

注意：避免在 Token 中存储敏感信息（如密码），因为 JWT 只是 Base64 编码，不是加密，可被解码查看。

---

## 6. 刷新 Token（Refresh Token 机制）

JWT 本身无状态，无法主动失效。常见做法是：

- Access Token：短期有效（如 15 分钟）
- Refresh Token：长期有效（如 7 天），用于换取新的 Access Token

Refresh Token 通常存储在数据库或 Redis 中，以便在用户登出时将其作废。

---

## 7. 安全建议

- 密钥管理：`SECRET_KEY` 必须保密，不要硬编码，使用环境变量。
- 使用 HTTPS：防止 Token 被窃听。
- 设置合理过期时间：减少被盗用风险。
- 不要存储敏感数据：JWT 可被解码。
- 考虑使用黑名单：对于提前登出的 Token，可用 Redis 存储已失效的 Token（需配合 `jti` 声明）。

---

## 8. 扩展：使用 `jti`（JWT ID）防止重放攻击

```
import uuid
payload = {
"user_id": 123,
"jti": str(uuid.uuid4()), # 唯一标识此 Token
"exp": datetime.datetime.utcnow() + datetime.timedelta(hours=1)
}
```

服务端可记录已使用的 `jti`，防止重复使用。

---

## 总结

JWT 完整流程：

1. 用户登录 → 服务端验证凭据 → 生成 JWT
2. 客户端保存 Token（如 localStorage / Cookie）
3. 后续请求携带 Token（如 `Authorization: Bearer <token>`）
4. 服务端解析并验证 Token → 允许/拒绝请求
5. Token 过期后，使用 Refresh Token 获取新 Access Token（可选）