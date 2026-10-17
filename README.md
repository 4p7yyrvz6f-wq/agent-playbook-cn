# agent-playbook-cn

面向中文用户的 AI Agent Skill 合集。每个 skill 均来自真实生产环境的实战踩坑，非理论整理——所有"坑位"都实际踩过并验证过解法。

> 适用于 Claude Code / WorkBuddy / OpenClaw 等支持 SKILL.md 规范的 Agent 框架。将对应目录复制到你的 skills 目录（如 `~/.claude/skills/` 或 `~/.workbuddy/skills/`）即可使用。

## Skills

### 📄 [camscanner-full-backup](./camscanner-full-backup/) — 扫描全能王云端文档全量备份

把 CamScanner 云端文档全量导出到本地磁盘并去重。核心价值是四个官方文档不会告诉你的 API 陷阱：

- `search_cloud_doc` 单次上限 50 条且**无 offset 参数**
- 只传 `end_time` 时按 **modify_time** 过滤 → 漏掉"创建早、近期改动过"的文档（最隐蔽的坑）
- 按年份分片 + `cs_doc_id` 去重的正确枚举姿势
- 下载链接过期规避、PDF 三重完整性校验（结构 / 字节数对账 / pypdf strict 逐页解压）
- MD5 + aHash 感知哈希 + 像素级抽样的三层去重（CamScanner PDF 无 OCR 文本层，不能靠文本比质量）
- Windows 瞬时文件占用重试、重编号撞名的两阶段重命名

**依赖**：CamScanner MCP（或等价 API 通道）、Python 3（pypdf、Pillow 可选）

### 🏢 [enterprise-risk-list-verify](./enterprise-risk-list-verify/) — 企业风险名单核实与研判

拿到"预警名单 / 团伙画像 / 空壳名单"类 CSV 时，先验证数据本身、再研判风险。行业里这类表最常见的失败模式不是造假，而是**关键词模糊匹配 + 笛卡尔积展开**造成的名称-代码错乱。

- GB 32100-2015 统一社会信用代码第 18 位校验码算法（Python 开箱即用）
- 名称-代码错乱映射识别（P0 缺陷定位：法人与注册资本双空行 ≈ 100% 噪声）
- 壳群 9 特征定性表（同日批量设立 / 0 实缴 0 社保 / 循环持股…）
- 实控人穿透路径、时间窗口分析（换壳/资产平移信号）
- 布尔值风险字段掩盖升级信号等易踩坑提醒

**依赖**：无硬依赖（校验位算法纯 Python 标准库）；主体核实环节可接任意工商数据 API 或 WebSearch

## 安全与合规

- 仓库内**不含**任何 API Key、Token、密码、个人电脑路径、真实企业名单或个人身份信息。
- 所有示例均为**方法论与通用占位符**（如 `XX银行XX有限公司`），不对应任何真实主体。
- `camscanner-full-backup` 仅适用于备份**使用者本人账号**的数据；请遵守对应服务的条款。
- `enterprise-risk-list-verify` 输出的是**分析结论而非事实认定**，仅供参考，须以官方登记与生效法律文书为准；请仅用于合法的尽职调查与风控场景。
- 两个 skill 均不联网自发请求，不收集、不上传任何用户数据；所有处理在本地完成。

## 目录结构

```
agent-playbook-cn/
├── README.md
├── LICENSE
├── camscanner-full-backup/
│   └── SKILL.md
└── enterprise-risk-list-verify/
    └── SKILL.md
```

## 贡献

欢迎 issue / PR 补充新的实战 skill。要求：来自真实踩坑、含可复现代码片段、不含任何个人路径与凭证。

## License

[MIT](./LICENSE)
