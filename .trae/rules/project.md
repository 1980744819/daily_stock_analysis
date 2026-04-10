# 项目规则 - daily_stock_analysis

股票智能分析系统，覆盖 A股、港股、美股。主流程：抓取数据 -> 技术分析/新闻检索 -> LLM 分析 -> 生成报告 -> 通知推送。

## 目录边界

| 目录 | 职责 |
|------|------|
| `src/core/services repositories agent` | 后端核心 |
| `data_provider api bot` | 数据源/API/机器人 |
| `apps/dsa-web apps/dsa-desktop` | Web/桌面端 |
| `scripts .github/workflows docs` | 脚本/流水线/文档 |

## 技术栈

Python(uv) | Vue3(pnpm) + Vite + Arco Design | LiteLLM | AkShare/Tushare/Longbridge | k3s/Docker/Jenkins

## 常用命令

```bash
# 后端
python main.py --debug --dry-run --stocks 600519,hk00700,AAPL
./scripts/ci_gate.sh && python -m pytest -m "not network"

# 前端
cd apps/dsa-web && npm ci && npm run lint && npm run build
```

## 硬规则

- 不执行 git commit/tag/push（除非确认）
- 不写死密钥/账号/路径/模型名/端口
- 新增配置项必须更新 `.env.example`
- 涉及用户可见变化必须更新 `docs/CHANGELOG.md`

## 交付说明

改了什么 | 为什么 | 验证情况 | 未验证项 | 风险点 | 回滚方式
