---
name: doc-processing
description: 文档处理工作流。Excel/PDF/Word 读取分析、MCP 文档工具集成、思维导图生成、表格框架创建。从 exccc/kfff/kj 等项目提取。
---

# 文档处理工作流 (Auto-Learned)

用户常需要处理 Excel、PDF、Word 文档，通常通过 MCP 工具完成。

## MCP 文档工具集

| 工具 | 用途 | 安装位置 |
|------|------|----------|
| docx MCP | Word 文档读写、表格操作 | 项目级 mcp.json |
| pdf-reader MCP | PDF 文本提取、OCR | 项目级 mcp.json |
| excel MCP | Excel 读写、格式化 | 项目级 mcp.json |
| document-loader | AWS 文档加载器 | 项目级 mcp.json |

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

## MCP 配置模式

用户习惯在 `mcp.json` 中按需添加文档处理 MCP：
```json
{
  "mcpServers": {
    "docx": {
      "command": "...",
      "args": ["..."]
    },
    "pdf-reader": {
      "command": "...",
      "args": ["..."]
    }
  }
}
```

## 注意事项

- 文档文件通常在当前项目目录下
- PDF 扫描件需要 OCR 支持
- Excel 处理注意编码（中文列名）
- 多文档交叉引用时，先建立数据映射关系
