---

title: "自动部署多个Cloudflare Pages项目"

author: "xxsky"

type: "posts"

date: 2025-12-17T23:35:07+08:00

subtitle: ""

image: ""

tags:
  - 技术
  - 学习

---

主仓库自动部署到多个pages项目

<!--more-->

**核心组件**  
* GitHub Actions: 用于执行自动化流程的 CI/CD 工具。
* Cloudflare API 令牌 (API Token): 用于授权 GitHub Actions 操作您的 Cloudflare 账户资源，比全局API密钥更安全、更推荐。
* GitHub Secrets: 用于安全地存储您的 API 令牌和账户ID，避免敏感信息暴露在代码中。

## 步骤一：创建 Cloudflare API 令牌

*为了实现最佳的便利性和安全性，我们创建一个可以管理您名下所有账户的用户API令牌。*

1. 登录 Cloudflare。  
2. 点击右上角您的头像，选择 “My Profile” (我的个人资料)。
3. 在左侧菜单选择 “API Tokens” (API令牌)，然后点击 “创建令牌 (Create Token)”。
4. 创建自定义令牌：点击 “创建自定义令牌 (Create Custom Token)” 下面的 “开始 (Get started)” 按钮。
5. 配置权限 (Permissions)：这是最关键的一步，我们只授予部署所需的最小权限。
   在权限表格中，添加以下唯一一行权限：
   | 权限组 | 资源 | 权限级别 |
   | :------------------- | :------------------- | :--------------- |
   | 账户 (Account) | Cloudflare Pages | 编辑 (Edit) |
6. 配置账户资源 (Account Resources)：为了让这个令牌能“通用”，请确保设置为：
   包括 (Include) → 该用户所属的所有账户 (All accounts this user is a member of)
7. 检查限制（重要）：
   IP地址过滤 (IP Address Filtering)：请确保此项为空，不要填写任何IP地址，否则GitHub Actions服务器将无法使用。
   TTL / 有效期 (End Date)：建议将结束日期留空，使其永不过期，以防自动化流程突然中断。
8. 点击 “创建令牌 (Create Token)”，然后立刻复制新生成的令牌值，此值只会出现一次。

## 步骤二：配置 GitHub Secrets

*进入您需要部署的 GitHub 仓库，点击 Settings > Secrets and variables > Actions，添加以下所有必需的密钥。*

| Secret名称  | 对应的值                 | 描述                     |
| --------- | -------------------- | ---------------------- |
| cf_hxjx_a | 账户A的API令牌            | 可以使用您上一步创建的那个“通用”令牌    |
| id_hxjx_a | 账户A的账户ID(Account ID) | 可以在Cloudflare账户主页右下角找到 |
| cf_hxjx_b | 账户B的API令牌            | 可以使用您上一步创建的那个“通用”令牌    |
| id_hxjx_b | 账户B的账户ID(Account ID) | 可以在Cloudflare账户主页右下角找到 |
| cf_hxjx_c | 账户C的API令牌            | 可以使用您上一步创建的那个“通用”令牌    |
| id_hxjx_c | 账户C的账户ID(Account ID) | 可以在Cloudflare账户主页右下角找到 |

## 步骤三：编写GitHub Actions Workflow文件

*在您的代码仓库根目录下，创建 .github/workflows/ 文件夹（如果不存在的话），然后在其中创建一个名为 deploy.yml 的文件。将以下最终的、经过验证的正确代码完整复制进去。*

```javascript
name: Deploy Cloudflare Pages

on:
  push:
    branches:
      - main

jobs:
  # 任务一：部署到账户 A
  deploy_account_A:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Deploy to Account A 
        if: github.ref == 'refs/heads/main'
        run: npx wrangler pages deploy . --project-name="2520sss2"
        env:
          CLOUDFLARE_API_TOKEN: ${{ secrets.tk_hxjxaa }}
          CLOUDFLARE_ACCOUNT_ID: ${{ secrets.id_hxjxaa }}

  # 任务二：部署到账户 B
  deploy_account_B:
    needs: deploy_account_A
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Deploy to Account B 
        if: github.ref == 'refs/heads/main'
        run: npx wrangler pages deploy . --project-name="2521sss2"
        env:
          CLOUDFLARE_API_TOKEN: ${{ secrets.tk_hxjxaaa }}
          CLOUDFLARE_ACCOUNT_ID: ${{ secrets.id_hxjxaaa }}

  # 任务三：部署到账户 C (这是新增的部分)
  deploy_account_C:
    needs: deploy_account_B # 确保在任务B成功后，此任务才会开始
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Deploy to Account C 
        if: github.ref == 'refs/heads/main'
        run: npx wrangler pages deploy . --project-name="2522sss2" # <-- 替换成您第三个项目的名称
        env:
          CLOUDFLARE_API_TOKEN: ${{ secrets.tk_hxjxab }}      # <-- 使用您为账户C创建的新密钥
          CLOUDFLARE_ACCOUNT_ID: ${{ secrets.id_hxjxab }}       # <-- 使用您为账户C创建的新密钥

   # 任务三：部署到账户 D (这是新增的部分)
  deploy_account_D:
    needs: deploy_account_C # 确保在任务C成功后，此任务才会开始
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Deploy to Account D 
        if: github.ref == 'refs/heads/main'
        run: npx wrangler pages deploy . --project-name="2527sss2" # <-- 替换成您第三个项目的名称
        env:
          CLOUDFLARE_API_TOKEN: ${{ secrets.tk_hxjxabc }}      # <-- 使用您为账户C创建的新密钥
          CLOUDFLARE_ACCOUNT_ID: ${{ secrets.id_hxjxabc }}       # <-- 使用您为账户C创建的新密钥
          
  deploy_account_E:
    needs: deploy_account_D
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Deploy to Account E 
        if: github.ref == 'refs/heads/main'
        run: npx wrangler pages deploy . --project-name="2526sss2" 
        env:
          CLOUDFLARE_API_TOKEN: ${{ secrets.tk_hxjxabd }}      
          CLOUDFLARE_ACCOUNT_ID: ${{ secrets.id_hxjxabd }}          
```
