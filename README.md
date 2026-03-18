# My Skills

这是一个 Claude Code Skills 集合仓库，包含用于扩展 Claude Code 功能的自定义技能。

## 📚 什么是 Skills？

Skills 是 Claude Code 的扩展机制，允许你定义可重用的工作流程和任务模板。每个 skill 都是一个包含详细步骤说明的 `SKILL.md` 文件，Claude Code 可以根据这些说明自动执行复杂的多步骤任务。

## 🎯 可用 Skills

### json2md-feishu-publisher

将 JSON 格式文档转换为 Markdown 并发布到飞书文档。

**功能：**
- 自动解析 JSON 结构
- 生成格式化的 Markdown（支持表格、图片）
- 创建飞书云文档并分享给指定用户

**适用场景：**
- 将数据在飞书上展示
- 将 API 响应或数据导出发布到飞书
- 团队数据共享和协作

**使用方式：**
```
将这个 JSON 文件发布到飞书
```

详细文档：[json2md-feishu-publisher/SKILL.md](./json2md-feishu-publisher/SKILL.md)

## 🚀 如何使用

### 1. 在 Claude Code 中使用 Skills

将此仓库克隆到本地，然后在项目目录中与 Claude Code 交互时，直接描述你的需求：

```bash
# 示例
"将 data.json 发布到飞书，分享给 user@example.com"
```

Claude Code 会自动识别并执行相应的 skill。

### 2. 配置环境变量

某些 skills 需要配置环境变量。在你的 shell 配置文件（如 `~/.zshrc` 或 `~/.bashrc`）中添加：

```bash
# 飞书分享默认邮箱
export FEISHU_SHARE_EMAIL="your-email@example.com"
```

### 3. 安装依赖

根据具体 skill 的要求安装依赖。例如：

```bash
# Python 依赖
pip install -r <skill-directory>/requirements.txt

# MCP 服务
# 确保已配置相应的 MCP 服务（如 feishu-mcp）
```

## 📁 项目结构

```
my_skills/
├── README.md                          # 本文件
├── .gitignore                         # Git 忽略规则
├── json2md-feishu-publisher/          # JSON 转 Markdown 并发布飞书
│   ├── SKILL.md                       # Skill 定义
│   ├── README.md                      # 详细文档
│   ├── scripts/                       # 脚本文件
│   ├── examples/                      # 示例文件
│   └── tests/                         # 测试用例
└── [其他 skills...]
```

## 🛠️ 创建新 Skill

1. 在项目根目录创建新的 skill 目录
2. 创建 `SKILL.md` 文件，包含以下结构：

```markdown
---
name: your-skill-name
description: 简短描述你的 skill 功能
version: 1.0.0
metadata:
  author: your-name
---

# Your Skill Name

详细说明...

## 步骤
Step1: ...
Step2: ...
```

3. 添加示例、测试和文档
4. 更新本 README 文件

## 📝 开发指南

### Skill 最佳实践

- **清晰的步骤**：将复杂任务分解为明确的步骤
- **错误处理**：考虑异常情况和错误处理
- **可配置性**：使用环境变量或参数提供灵活性
- **文档完善**：提供示例和详细说明
- **可测试性**：包含测试用例和验证方法

### 代码规范

- Python 代码遵循 PEP 8 规范
- Bash 脚本使用 shellcheck 验证
- 所有脚本包含错误处理和输出说明

## 🤝 贡献

欢迎贡献新的 skills 或改进现有 skills！

1. Fork 本仓库
2. 创建你的特性分支
3. 提交你的改动
4. 推送到分支
5. 创建 Pull Request

## 📄 许可证

本项目采用 MIT 许可证 - 详见 LICENSE 文件

## 🔗 相关资源

- [Claude Code 文档](https://docs.anthropic.com/claude-code)
- [Skills 开发指南](https://docs.anthropic.com/claude-code/skills)
- [MCP 协议](https://modelcontextprotocol.io/)

## 📮 联系方式

- 作者：xuzhaopu
- 问题反馈：通过 GitHub Issues 提交

---

**注意**：使用这些 skills 前，请确保已安装 Claude Code 并配置好相应的 MCP 服务。
