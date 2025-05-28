---
title: Git Commit规范指南
date: 2025-05-28 11:28:43
tags: Git Doc
---

## **1. Commit Message 的基本结构**

一个规范的 Commit Message 通常包含 **3 个部分**：

1. **Header（标题行）**：必填，格式为 `<type>(<scope>): <subject>`
2. **Body（正文）**：可选，详细描述修改内容
3. **Footer（脚注）**：可选，用于关联 Issue 或标注破坏性变更

```
<type>(<scope>): <subject>
<空行>
<body>
<空行>
<footer>
```

---

## **2. Commit 类型（Type）**

| 类型       | 说明                                             |
| ---------- | ------------------------------------------------ |
| `feat`     | 新增功能（feature）                              |
| `fix`      | 修复 bug                                         |
| `docs`     | 文档更新（README、注释等）                       |
| `style`    | 代码格式化（不影响逻辑，如空格、换行、分号等）   |
| `refactor` | 重构代码（非功能新增，也非 bug 修复）            |
| `perf`     | 性能优化                                         |
| `test`     | 测试用例（新增或修改测试）                       |
| `chore`    | 构建或辅助工具变动（如依赖更新、CI/CD 配置修改） |
| `ci`       | CI/CD 相关修改（如 GitHub Actions、Jenkins）     |
| `revert`   | 回滚之前的 commit                                |

---

## **3. Scope（可选）**

用于说明影响范围，通常是模块、组件或文件名，例如：

- `feat(user): 新增用户注册功能`
- `fix(auth): 修复登录失败问题`

---

## **4. Subject（标题）**

- **简短清晰**（不超过 50 字符）
- **使用祈使语气**（如 "Fix" 而不是 "Fixed" 或 "Fixes"）
- **首字母小写**，不加句号
- **避免模糊描述**（如 "Update code" 或 "Fix bug"）

❌ **错误的例子**：

- `fixed a bug`（应使用祈使语气）
- `update`（太模糊）
- `fix: fixed the login issue.`（重复 "fix"，且加了句号）

✅ **正确的例子**：

- `feat: add user registration`
- `fix: handle null pointer in login API`
- `docs: update README installation guide`

---

## **5. Body（正文，可选）**

如果修改较复杂，可以在 Body 中详细说明：

- **为什么修改**（动机）
- **如何修改的**（关键代码逻辑）
- **与之前行为的对比**（如果有破坏性变更）

✅ **例子**：

```
fix: prevent race condition in user creation

Previously, concurrent user creation could lead to duplicate emails due to a missing database lock. This commit adds a unique constraint and uses `SELECT FOR UPDATE` to ensure atomicity.
```

---

## **6. Footer（脚注，可选）**

用于：

- **关联 Issue**（如 `Closes #123`, `Fixes #45`）
- **标注 BREAKING CHANGE**（如果有不兼容的修改）

✅ **例子**：

```
feat: migrate to Vue 3

BREAKING CHANGE: Drop support for IE11 due to Vue 3 compatibility.
```

```
fix: resolve memory leak in event handler

Closes #123
See also #456
```

---

## **7. 完整示例**

### **示例 1：新功能（feat）**

```
feat(user): add email verification

- Add `sendVerificationEmail` method
- Store verification token in database
- Update user schema to track verification status

Closes #42
```

### **示例 2：Bug 修复（fix）**

```
fix(auth): handle expired JWT tokens

Previously, expired tokens would cause a 500 error. Now, the API returns a 401 with a clear error message.

Fixes #78
```

### **示例 3：重构（refactor）**

```
refactor(api): simplify error handling

- Replace custom error classes with `http-errors` library
- Standardize error responses across endpoints
```

### **示例 4：文档更新（docs）**

```
docs: update API usage examples

Add examples for pagination and error handling in the OpenAPI spec.
```

### **示例 5：破坏性变更（BREAKING CHANGE）**

```
feat(config): replace JSON config with YAML

BREAKING CHANGE: The old `config.json` is no longer supported. Migrate to `config.yml` using the provided script.
```

---

## **8. 如何写好 Commit Message？**

1. **先写标题，再补充细节**（可以用 `git commit -v` 查看代码变动）
2. **避免一个 Commit 做多件事**（如同时改功能和修 bug）
3. **团队统一规范**（可以用 `commitlint` 或 `husky` 自动化检查）
4. **使用 `git rebase -i` 整理提交历史**（让 Commit 更清晰）

---

## **9. 常见错误**

❌ **"Update code"**（太模糊）  
✅ **"fix: resolve null reference in payment service"**  

❌ **"Fixed bug"**（没说明具体问题）  
✅ **"fix(auth): handle invalid OTP tokens"**  

❌ **"Add new feature"**（没说明是什么功能）  
✅ **"feat(profile): add dark mode toggle"**  

---

## **总结**

| 要点                  | 示例                         |
| --------------------- | ---------------------------- |
| **类型 + 描述**       | `feat: add search API`       |
| **带 scope**          | `fix(login): handle timeout` |
| **Body 详细说明**     | 解释修改原因和影响           |
| **Footer 关联 Issue** | `Closes #123`                |

**好的 Commit Message 能让团队协作更高效！** 
