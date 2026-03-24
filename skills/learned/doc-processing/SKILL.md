---
name: doc-processing
description: 文档处理工作流。Excel/PDF/Word/PPT 读写分析、MCP 按需启动命令、思维导图生成、表格框架创建。当用户要求读取/分析/生成 Excel、PDF、Word、PPT 文档时使用。
---

# 文档处理工作流

用户常需要处理 Excel、PDF、Word 文档，通常通过 MCP 工具完成。

## 文档工具（按需后台调用）

PPT/Word/Excel/PDF 不默认启用为 MCP。需要时在项目目录下执行对应命令（后台运行），待服务就绪后通过 MCP 调用。

| 类型 | 用途 | 启动命令 |
|------|------|----------|
| Excel | Excel 读写、表格处理 | `npx -y @negokaz/excel-mcp-server` |
| PDF | PDF 文本提取、阅读 | `npx -y @sylphx/pdf-reader-mcp` |
| Word | Word 文档读写、表格 | `uvx --from office-word-mcp-server word_mcp_server` |
| PPT | PPT 创建、编辑、审阅 | `uvx pptx-mcp serve` |

先确认 `npx`、`uvx` 可用；若用户环境无 uv，Word/PPT 可改用其他已安装方式。

## 常见工作流

### Excel + PDF 交叉处理 (exccc)

```
1. 读取 Excel 数据 (excel_read_sheet)
2. 扫描 PDF 识别匹配数据 (read_pdf)
3. 参照模板 Excel 填写备注
4. 输出处理后的 Excel
```

### Word 文档处理 (kfff)

```
1. 读取 docx 内容 (get_document_text)
2. 分析文档结构 (get_document_outline)
3. 查看 word 内图片
4. 根据内容制定计划
```

### 思维导图 + PDF 生成 (kj)

```
1. 用 MCP 读取分析项目文件
2. 整理优化到新目录
3. 生成思维导图
4. 导出 PDF 文档
```

## 表格框架生成

用户有时根据图片生成表格框架（如从截图创建 Excel/Word 表格）。

## 注意事项

- 文档文件通常在当前项目目录下
- PDF 扫描件需要 OCR 支持
- Excel 处理注意编码（中文列名）
- 多文档交叉引用时，先建立数据映射关系
