---
name: ast-deobfuscate
description: JS AST 反混淆（Babel）。字符串数组还原、控制流还原、常量折叠、死代码删除、与 jshook MCP 配合验证。用户说「反混淆」「还原代码」「_0x 混淆」或需逆向混淆 JS 时使用。
---

# AST 反混淆（Babel + jshook）

## 目录约定

```
project/
├── source/original/          # 原始混淆（只读）
├── scripts/                  # 每步一个脚本
├── intermediate/             # 每步输出，可回滚
└── source/deobfuscated/      # 最终输出
```

每步从上一步输出读入，可从任意 intermediate 重跑。

## 环境

```bash
npm install @babel/parser @babel/traverse @babel/generator @babel/types prettier
```

## 执行策略

- **计划模式**：先完成 Step 0 分析，将特征与拟执行步骤（如 Step 1→2→4→5→6）展示给用户，确认后再执行；无字符串数组则跳过 Step 1，无控制流平坦化则跳过 Step 4。
- jshook：用 search_in_scripts/get_script_source 取源码与特征；用 page_evaluate/debugger_evaluate 做小规模验证；大批量变换用 Node 脚本。

### MCP 使用时机（jshook）

| 阶段 | jshook 用法 |
|------|-------------|
| 分析 | search_in_scripts 搜索混淆特征；get_script_source 获取目标脚本 |
| 方案验证 | page_evaluate / debugger_evaluate_global 执行少量解密调用，确认可行后再写批量脚本 |
| 中间验证 | page_evaluate 抽样对比某步前后的函数输出 |
| 控制流分析 | breakpoint_set + debugger_step_over 动态跟踪 switch-case 执行顺序 |
| 最终验证 | 在浏览器中用解混淆后的代码替换原始代码，验证功能 |

### Subagent 并行（可选）

Step 0 内特征识别与反调试搜索可并行；Step 2 内不同 visitor 可并行；多组验证可并行。

## Step 0：预分析与特征

- 获取源码：本地读文件 / 下载到 source/original / jshook get_all_scripts + get_script_source。
- 格式化后存 intermediate/target_step0.js。
- 特征指纹：高频率前缀（_0x/a0_/_$）+ 大字符串数组 + while+switch → 控制流平坦化；jsjiami.com → sojson；大量 eval/Function → 先沙箱解密。
- 若有 anti-debug（debugger、setInterval(debugger)）：先去掉再继续。
- **可选：AST 体检报告**（文件较大时）：用脚本统计——行数、AST 节点数、函数数；最大数组元素数（>50 多为字符串数组）；CallExpression callee 频次 Top 5；WhileStatement+SwitchStatement 数量；标识符前缀分布；StringLiteral 数量 vs 字符串调用比。用于辅助判断混淆类型与步骤选择。

## Step 1：字符串解密与数组回填

- 条件：存在字符串数组 + 旋转 IIFE + 访问函数。
- 可选：Node vm 沙箱执行数组+旋转+访问函数，遍历 CallExpression 替换为字面量；或静态算旋转后按索引替换。删除已用掉的数组与访问函数。
- 顺带还原 StringLiteral 的 extra（\xNN → 正常字符）、NumericLiteral 的 extra。

## Step 2：表达式简化

- 常量折叠：path.evaluate() 处理二元/一元/逻辑/条件表达式，confident 则 valueToNode 替换。
- 布尔还原：![]→false、!![]→true、!0→true、void 0→undefined。
- 单返回代理函数内联、仅作属性字典的对象内联。

## Step 3：引用与属性规范化

- 合法标识符的 obj['prop'] → obj.prop。
- 逗号表达式拆开：return/if/for 等处的逗号表达式全部分离。

## Step 4：控制流还原

- 条件：while(true){ switch(...) } 存在。
- 用 breakpoint_set + debugger_step_over 动态跟执行顺序，或静态分析得到 case 顺序，按序提取块，replaceWithMultiple 去掉 while/switch/continue。

## Step 5：死代码删除

- 不可达分支：evaluateTruthy 后替换为对应分支。
- 未引用且无副作用的变量/函数删除。
- **Step 2 与 Step 5 必须循环**：先跑 Step 2 的 visitor，再跑 Step 5，有改动则重复，最多约 50 轮，每轮可存 intermediate。

## Step 6：变量重命名与收尾

- 仅对确定模式重命名（如 webpack 的 module/exports/__webpack_require__）。
- 多变量声明拆分、if-return 简化，输出 compact: false。
- 用 page_evaluate 对比关键函数输出与原始。

## Babel API 速查

- 静态求值：path.evaluate() → { confident, value }。
- 替换/删除：path.replaceWith(node)、replaceWithMultiple([nodes])、path.remove()。
- 作用域：path.scope.getBinding(name)、path.scope.rename(old, new)、修改后 path.scope.crawl()。

## 安全与注意

- 混淆代码只在 vm 或 jshook 环境执行，不用裸 eval。
- Step 2/5 循环上限 50，避免无限展开。
- 重命名只用 path.scope.rename()，不直接改 node.name。
- 每步后重新 parse 校验；不确定处宁可不改也不要破坏逻辑。
