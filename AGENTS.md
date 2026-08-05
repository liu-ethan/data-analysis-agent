# AGENTS.md
 
## Python / 配置（强制）

- Python：仅用 conda `/home/user/miniconda3/envs/python3.12`（或其 `bin/python` / `bin/pip`）；禁止系统 Python 与仓库 `.venv`
- 配置：仅用根目录 `config.yaml`（模板 `config_template.yaml`）；禁止 `.env`；勿提交密钥；测试可用 `APP_CONFIG`

## 代码规范

- 后端：Python 3.12 + FastAPI；前端：React + Vite + TypeScript + Tailwind
- 匹配现有风格；少改无关文件；不做未要求的抽象或过度设计
- Agent 各节点独立文件；SQL 权限校验与沙箱执行必须独立实现
- 不引入未约定依赖

## 前端

- 做前端 UI / 视觉设计时，必须先读并遵循 `frontend-design` skill
- `/` 为营销感登录注册页，`/app` 为工作台；布局与交互以 `docs/04-接口与前端.md` 为准
- 视觉细节模糊时先问用户
