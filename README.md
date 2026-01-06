
## Project context

I am building a deliberately small backend authentication service as a practice kata to improve:
    - backend design fundamentals
    - production-grade coding habits
    - code review skills

This project is not a full product.
The goal is to practice clean separation of concerns and testable design.

---

### Technical constraints

- Language: Python
- Framework: FastAPI
- No database (use in-memory repository only)
- No JWT, no refresh token, no Redis (yet)
- Focus on a single endpoint only

---

### Current scope

Implement POST /login with the following requirements:

- Accept username and password
- Verify credentials via an authentication service (not inside the router)
- Passwords must be hashed (do not use plaintext comparison)
- On success:
    - Return HTTP 200
    - Return { "access_token": "...", "token_type": "bearer" }
- On invalid credentials:
    - Return HTTP 401
    - Do not reveal whether the user exists

---

### Design rules

- Router handles only HTTP I/O
- Business logic lives in a service layer
- Data access goes through a repository abstraction
- Code should be easy to unit test

---

### Testing

- Include basic tests for:
    - successful login
    - invalid credentials returning 401

---

> [!IMPORTANT]
> Do NOT implement extra features.
> Do NOT over-engineer.
> Prefer clarity and correctness over completeness.

---

### Phase 1 - Task 1（你的第一個正式任務）
#### 任務目標（請仔細看）

你要實作 只有一個 endpoint：

``` python
POST /login
```


##### 功能需求（刻意設計過）
- 接收 username / password
- 驗證帳密（不用 DB，用 in-memory repo）
- 成功：
    - 回 200
    - 回 { "access_token": "...", "token_type": "bearer" }
- 失敗：
    - 帳密錯誤 → 401
    - request 不合法 → 422（交給 FastAPI）

- 不要明碼密碼
- 不要把邏輯寫在 router 裡