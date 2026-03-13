# GoGoCode 整体技术架构与核心流程详解

> GoGoCode 是一款基于 AST（抽象语法树）的代码转换工具，为 JavaScript/TypeScript/HTML/Vue 提供类 jQuery 的操作 API，让开发者无需深入编译原理即可实现强大的代码分析与自动化重构。

---

## 目录

1. [项目整体架构概览](#1-项目整体架构概览)
2. [核心处理管道（完整流程）](#2-核心处理管道完整流程)
3. [解析阶段（Parsing Phase）](#3-解析阶段parsing-phase)
4. [选择器转换阶段（Selector Compilation）](#4-选择器转换阶段selector-compilation)
5. [模式匹配引擎（Pattern Matching Engine）](#5-模式匹配引擎pattern-matching-engine)
6. [AST 操作与转换阶段（Transformation Phase）](#6-ast-操作与转换阶段transformation-phase)
7. [代码生成阶段（Code Generation Phase）](#7-代码生成阶段code-generation-phase)
8. [多语言抽象层（Multi-Language Abstraction）](#8-多语言抽象层multi-language-abstraction)
9. [插件系统架构（Plugin System）](#9-插件系统架构plugin-system)
10. [CLI 工具实现原理（CLI Implementation）](#10-cli-工具实现原理cli-implementation)
11. [核心数据结构全景](#11-核心数据结构全景)
12. [关键 API 深度解析](#12-关键-api-深度解析)
13. [通配符匹配机制详解](#13-通配符匹配机制详解)
14. [Vue 单文件组件处理流程](#14-vue-单文件组件处理流程)
15. [HTML 文件处理流程](#15-html-文件处理流程)
16. [错误处理与容错机制](#16-错误处理与容错机制)
17. [性能优化设计](#17-性能优化设计)
18. [扩展性设计原则](#18-扩展性设计原则)

---

## 1. 项目整体架构概览

### 1.1 项目定位

GoGoCode 是一个代码变换工具链，其核心能力是：**将源代码解析为 AST → 以声明式模式定位目标节点 → 变换节点 → 重新生成代码**。这个流程与信息检索系统的「检索-增强-生成」模式高度相似：

- **检索（Retrieval）**：从 AST 中精确定位匹配的代码结构
- **增强（Augmentation）**：将匹配结果与转换规则结合，生成变换指令
- **生成（Generation）**：将变换后的 AST 序列化为新的代码文本

### 1.2 包结构总览

```
gogocode/
├── packages/
│   ├── gogocode-core/          # 核心库：AST 解析、匹配、变换引擎
│   ├── gogocode-cli/           # 命令行工具：批量文件变换
│   ├── gogocode-plugin-vue/    # Vue 2→Vue 3 迁移插件
│   ├── gogocode-plugin-element/# ElementUI→ElementPlus 迁移插件
│   ├── gogocode-plugin-sample/ # 插件开发示例模板
│   ├── gogocode-starter/       # 项目初始化脚手架
│   ├── gogocode-vue-playground/# Vue 变换在线演示
│   ├── gogocode-element-playground/ # Element 变换在线演示
│   └── esbuild-import-plugin/  # esbuild 按需导入插件
└── package.json                # Lerna monorepo 配置
```

### 1.3 依赖关系图

```
用户代码
    │
    ▼
gogocode-cli（命令行入口）
    │  加载
    ▼
gogocode-plugin-vue / gogocode-plugin-element（转换规则集合）
    │  调用 api.gogocode
    ▼
gogocode-core（核心引擎）
    │  依赖
    ├─► @babel/parser          （JS/TS 解析）
    ├─► recast-yx              （源码位置保留 + 代码生成）
    ├─► hyntax-yx              （HTML 解析）
    └─► vue3-browser-compiler-yx（Vue SFC 解析）
```

---

## 2. 核心处理管道（完整流程）

整个变换流程可以用以下管道来描述：

```
┌─────────────────────────────────────────────────────────────────┐
│                    GoGoCode 核心处理管道                          │
└─────────────────────────────────────────────────────────────────┘

  输入代码字符串/文件
        │
        ▼
  ┌─────────────┐
  │  语言检测    │  ← parseOptions.language / 文件扩展名
  └──────┬──────┘
         │
    ┌────┴─────┬─────────┐
    ▼          ▼         ▼
  JS/TS      HTML       Vue SFC
    │          │          │
    ▼          ▼          ▼
  ┌─────────────────────────────────┐
  │          解析阶段 (Parse)        │
  │  代码字符串 → AST 对象           │
  │  保留源码位置/格式/注释信息       │
  └────────────────┬────────────────┘
                   │
                   ▼
  ┌─────────────────────────────────┐
  │        NodePath 封装层           │
  │  为 AST 节点附加导航上下文        │
  │  (parent/parentPath/__childCache)│
  └────────────────┬────────────────┘
                   │
                   ▼
  ┌─────────────────────────────────┐
  │         AST 类实例化             │
  │  提供 jQuery 风格 API            │
  │  支持链式调用 (find/replace/attr)│
  └────────────────┬────────────────┘
                   │
         ┌─────────┴──────────┐
         ▼                    ▼
  ┌─────────────┐      ┌─────────────┐
  │  find() 查询 │      │ 直接操作    │
  │             │      │ attr/append │
  └──────┬──────┘      └─────┬───────┘
         │                   │
         ▼                   │
  ┌─────────────────────┐    │
  │  选择器编译阶段       │    │
  │  字符串 → 模式 AST   │    │
  │  $_ $ 替换为 expando │    │
  └──────────┬──────────┘    │
             │               │
             ▼               │
  ┌─────────────────────┐    │
  │  模式匹配引擎         │    │
  │  visit() 遍历 AST    │    │
  │  checkIsMatch() 递归 │    │
  │  捕获通配符数据       │    │
  └──────────┬──────────┘    │
             │               │
             ▼               │
  ┌─────────────────────────────────┐
  │         变换阶段 (Transform)     │
  │  replace/replaceBy/remove/insert│
  │  替换器处理通配符数据             │
  │  AST 节点原地更新                │
  └────────────────┬────────────────┘
                   │
                   ▼
  ┌─────────────────────────────────┐
  │        代码生成阶段 (Generate)   │
  │  recast.print() / prettyPrint() │
  │  保留未修改部分的原始格式         │
  └─────────────────────────────────┘
                   │
                   ▼
          输出代码字符串/文件
```

---

## 3. 解析阶段（Parsing Phase）

### 3.1 入口函数 `$.js`

`gogocode-core` 的入口文件 `src/$.js` 是整个系统的门户。

```javascript
// 核心逻辑（简化版）
const langCoreMap = {
  vue: vueCore,
  html: htmlCore,
  js: jsCore,
};

function main(code, options = {}) {
  const { parseOptions = {} } = options;
  
  // 1. 语言检测
  let lang = parseOptions.language || 'js';
  if (parseOptions.html) lang = 'html';
  
  // 2. 委派给对应语言核心
  const core = langCoreMap[lang] || jsCore;
  
  // 3. 构建 AST
  const astInfo = core.buildAstByAstStr(code, null, parseOptions);
  
  // 4. 包装为 AST 实例
  return new AST(astInfo.nodePath, astInfo.nodeType, options);
}
```

**关键设计决策**：语言路由在入口层完成，下游模块只需处理自己语言的 AST，做到职责单一。

### 3.2 JavaScript/TypeScript 解析

**核心文件**：`packages/gogocode-core/src/js-core/parse.js`

**解析器选型**：
- 使用 `@babel/parser`（v7.7.7+）作为 AST 解析器
- 使用 `recast-yx`（Recast 的定制 fork）作为解析框架

**为什么选 Recast 而不是直接用 Babel？**

Recast 解决了一个关键问题：**源码位置保留**。当对 AST 进行修改后再序列化时，Recast 只会重新打印被修改过的节点，**保留其余节点的原始格式（缩进、换行、注释）**。这对于大规模代码重构极为重要——避免了重新格式化整个文件导致不必要的 diff。

```javascript
// parse.js 核心实现
const recast = require('recast-yx');
const babelParse = require('@babel/parser');

module.exports = function parse(code, options) {
  return recast.parse(code, {
    parser: {
      parse(src) {
        return babelParse.parse(src, {
          sourceType: 'module',
          strictMode: false,
          allowImportExportEverywhere: true,
          allowReturnOutsideFunction: true,
          startLine: 1,
          tokens: true,
          plugins: [
            'asyncGenerators',
            'bigInt',
            'classPrivateMethods',
            'classPrivateProperties',
            'classProperties',
            'decorators-legacy',
            'doExpressions',
            'dynamicImport',
            'exportDefaultFrom',
            'exportNamespaceFrom',
            'functionBind',
            'functionSent',
            'importMeta',
            'logicalAssignment',
            'nullishCoalescingOperator',
            'numericSeparator',
            'objectRestSpread',
            'optionalCatchBinding',
            'optionalChaining',
            ['pipelineOperator', { proposal: 'minimal' }],
            'throwExpressions',
            'topLevelAwait',
            'jsx',
            'typescript',
          ],
        });
      },
    },
  });
};
```

**解析后的 AST 结构（以 `const a = 1` 为例）**：

```
Program
└── body: []
    └── VariableDeclaration
        ├── kind: "const"
        └── declarations: []
            └── VariableDeclarator
                ├── id: Identifier { name: "a" }
                └── init: NumericLiteral { value: 1 }
```

**每个节点携带的元信息**：
```javascript
{
  type: "VariableDeclaration",
  start: 0,           // 字符偏移量（起始）
  end: 12,            // 字符偏移量（结束）
  loc: {              // 行列位置
    start: { line: 1, column: 0 },
    end: { line: 1, column: 12 }
  },
  // ... 具体节点属性
}
```

### 3.3 `buildAstByAstStr` 函数的深度实现

`js-core/core.js` 中的 `buildAstByAstStr` 是解析的核心，比简单的 `parse()` 更智能：

```javascript
function buildAstByAstStr(str, astPatialMap, options) {
  // 步骤 1：如果有占位符替换映射，先替换字符串中的占位符
  if (astPatialMap) {
    str = replaceStrByAst(str, astPatialMap);
  }
  
  // 步骤 2：尝试作为完整代码解析
  let ast;
  try {
    ast = parse(str, options);
  } catch (e) {
    // 步骤 3：解析失败时尝试特殊语法构建器
    ast = buildMap.tryBuild(str);
    if (!ast) throw e;
  }
  
  // 步骤 4：提取 program body（处理表达式语句包裹）
  const body = ast.program.body;
  
  // 步骤 5：确定节点类型
  const nodeType = detectNodeType(body, str);
  
  // 步骤 6：包装为 NodePath
  const nodePath = buildNodePath(ast, nodeType);
  
  return { nodePath, nodeType };
}
```

**`buildMap` 特殊构建器**：

某些 AST 片段无法作为独立语句解析（如对象字面量、解构参数、装饰器），`buildMap` 为这些情况提供专门的构建策略：

| 构建器名称 | 处理的代码片段 | 实现策略 |
|---|---|---|
| `ObjectExpression` | `{a: 1, b: 2}` | 包裹为 `(${str})` 后解析 |
| `DestructuringParam` | `{a, b} = obj` | 包裹为函数参数后解析 |
| `ObjectProperty` | `key: value` | 包裹为对象后提取属性 |
| `ObjectMethod` | `method() {}` | 包裹为对象方法后提取 |
| `Decorators` | `@decorator` | 包裹为类声明后提取 |
| `ClassProperty` | `prop = value` | 包裹为类体后提取 |

### 3.4 NodePath 封装层

**核心文件**：`packages/gogocode-core/src/NodePath.js`

NodePath 是对 AST 节点的封装，提供导航上下文：

```javascript
class NodePath {
  constructor(node, parentNode, parentPath, name) {
    this.node = node;           // 实际 AST 节点
    this.value = node;          // 节点值的引用（兼容 Recast API）
    this.parent = parentNode;   // 父节点对象
    this.parentPath = parentPath; // 父 NodePath（用于向上遍历）
    this.__childCache = {};     // 子节点缓存（延迟初始化兄弟节点用）
    this.name = name;           // 在父节点中的属性名（如 'body', 'declarations'）
  }
  
  // 导航到子节点
  get(key) {
    if (!this.__childCache[key]) {
      this.__childCache[key] = new NodePath(
        this.node[key],
        this.node,
        this,
        key
      );
    }
    return this.__childCache[key];
  }
  
  // 替换当前节点
  replace(newNode) {
    if (Array.isArray(this.parent[this.name])) {
      // 在父数组中找到当前节点并替换
      const index = this.parent[this.name].indexOf(this.node);
      this.parent[this.name].splice(index, 1, newNode);
    } else {
      // 直接属性替换
      this.parent[this.name] = newNode;
    }
    this.node = newNode;
    this.value = newNode;
  }
}
```

**为什么需要 NodePath 而不直接用 AST 节点？**

AST 节点本身只包含语义信息，不知道自己在树中的位置。NodePath 的作用是：
1. 记录父节点引用，使得「向上遍历」和「替换当前节点」成为可能
2. 提供 `get()` 方法进行懒加载的子节点导航
3. 提供 `replace()` 方法进行原地节点替换（修改父节点的引用）

---

## 4. 选择器转换阶段（Selector Compilation）

### 4.1 概述

当用户调用 `$(code).find('const a = $_$')` 时，字符串 `'const a = $_$'` 需要被编译成一个**模式对象**，供匹配引擎使用。这个过程由 `js-core/get-selector.js` 负责。

### 4.2 `getSelector` 函数实现原理

```
输入: "const $_$ = $_$"
    │
    ▼
步骤 1: 识别 $_$ 通配符，生成唯一 expando 键
    │   $_$ → "g1a2o3g4o5" (expando)
    ▼
步骤 2: 替换后解析字符串
    │   "const g1a2o3g4o5 = g1a2o3g4o5_1" → @babel/parser → AST
    ▼
步骤 3: 提取 AST 中的节点类型
    │   VariableDeclaration (kind=const, declarations=[...])
    ▼
步骤 4: 过滤无关属性 (filterProps)
    │   去掉 loc, range, start, end, raw 等位置/格式属性
    ▼
步骤 5: 将 expando 标记保留在结构中作为通配符占位
    │
    ▼
输出: { nodeType: 'VariableDeclaration', structure: { kind: 'const', ... } }
```

### 4.3 属性过滤机制（`filter-prop.js`）

属性过滤是保证匹配准确性的关键。以下属性在匹配时会被忽略：

```javascript
const FILTERED_PROPS = [
  // 源码位置信息（每个节点的位置必然不同，不参与匹配）
  'range', 'loc', 'start', 'end',
  
  // 语法糖标记（对语义无影响）
  'computed',     // a.b vs a['b']
  'raw',          // 字面量的原始文本
  'shorthand',    // { a } vs { a: a }
  'static',       // 类成员是否 static
  'trailing',     // 尾随逗号
  'leading',      // 前导符号
  'method',       // { foo() {} } 的 method 标记
  
  // 内部标记
  '__clone',
  '__proto__',
  'extra',        // Babel 的额外信息
];
```

**特殊节点的处理**：

以下节点类型在匹配时直接视为通配符匹配通过（因为它们本身没有结构化内容可匹配）：

```javascript
const ALWAYS_MATCH_TYPES = [
  'Super',           // super 关键字
  'Import',          // import() 动态导入
  'ThisExpression',  // this 关键字
];
```

### 4.4 expando 机制详解

expando 是一个在每次调用时随机生成的唯一字符串，用于标识通配符占位符。

```javascript
// 生成方式（伪代码）
const expando = 'g' + Math.random().toString(36).slice(2) + 'o';
// 例如: "g7f3k9o" 或 "g2m8p4o"
```

**为什么要用随机字符串而不是简单的 `__WILDCARD__`？**

因为用户的代码中可能包含名为 `__WILDCARD__` 的变量。随机 expando 确保不与用户代码冲突，同时各次调用之间相互隔离。

**expando 的多个变体**：
- `expando` → 匹配第 0 个 `$_$`
- `expando_1` → 匹配第 1 个 `$_$`
- `expando_2` → 匹配第 2 个 `$_$`
- `expando_$$$` → 匹配 `$$$` 多值通配符

---

## 5. 模式匹配引擎（Pattern Matching Engine）

### 5.1 整体架构

模式匹配引擎位于 `js-core/find/general.js`，是 GoGoCode 最核心的算法模块。

```
find(nodeType, structure, strictSequence, deep, expando)
    │
    ▼
recast.visit(ast, {
  // 为每种目标节点类型注册访问器
  visitVariableDeclaration: function(path) { ... },
  visitFunctionDeclaration: function(path) { ... },
  // ...
})
    │
    ▼ 对每个访问到的节点
checkIsMatch(fullNode, patternStructure, extraData, strictSequence)
    │
    ├── 通过 → 记录到 nodePathList + matchWildCardList
    └── 失败 → 继续遍历
```

### 5.2 `checkIsMatch` 递归匹配算法

这是整个系统最精妙的部分。以下是其核心逻辑：

```javascript
function checkIsMatch(full, partial, extraData, strictSequence) {
  // 情况 1: partial 是 expando 标记（通配符）
  if (isExpando(partial, expando)) {
    // 捕获整个 full 节点到 extraData
    const wildcardIndex = getWildcardIndex(partial);
    extraData[wildcardIndex] = extraData[wildcardIndex] || [];
    extraData[wildcardIndex].push({
      node: full,
      value: full.name || full.value,
      raw: generate(full),
    });
    return true; // 通配符始终匹配
  }
  
  // 情况 2: partial 是普通对象（AST 节点）
  if (isObject(partial) && isObject(full)) {
    // 遍历 partial 的每个属性，检查 full 是否满足
    return Object.keys(partial).every(prop => {
      if (partial[prop] === null || partial[prop] === undefined) {
        return true; // null 属性跳过匹配
      }
      
      if (Array.isArray(partial[prop])) {
        return checkArrayMatch(full[prop], partial[prop], extraData, strictSequence);
      }
      
      if (isObject(partial[prop])) {
        return checkIsMatch(full[prop], partial[prop], extraData, strictSequence);
      }
      
      // 基本类型比较
      return full[prop] === partial[prop];
    });
  }
  
  // 情况 3: 直接相等
  return full === partial;
}
```

### 5.3 数组匹配的 `strictSequence` 策略

数组匹配是模式匹配中最复杂的部分，`strictSequence` 标志控制匹配行为：

**严格序列匹配（strictSequence = true）**：用于函数参数、数组字面量等顺序敏感的场景。

```
Pattern 数组: [A, expando, B]
Full 数组:    [A, X, B, C]

严格匹配逻辑:
  位置 0: A 匹配 A ✓
  位置 1: expando 检测到 $$$（多值通配符）→ 贪心匹配剩余元素
  位置 2: 继续向后匹配...
```

**非严格序列匹配（strictSequence = false）**：用于对象属性等顺序无关的场景。

```
Pattern: { b: expando, a: 1 }
Full:    { a: 1, b: 2, c: 3 }

非严格匹配逻辑:
  - a: 1 在 full 中找到 ✓
  - b: expando 在 full 中找到 b: 2，捕获 2 ✓
  - c: 3 在 pattern 中不存在，忽略 ✓
```

### 5.4 `$$$` 多值通配符的匹配

```javascript
function find$$$(patternArray, fullArray, extraData, strictSequence) {
  // $$$在 patternArray 中的位置
  const expandoIndex = patternArray.findIndex(item => isMultiExpando(item));
  
  // $$$之前的确定元素
  const before = patternArray.slice(0, expandoIndex);
  // $$$之后的确定元素
  const after = patternArray.slice(expandoIndex + 1);
  
  // 检查 before 部分是否匹配 fullArray 开头
  if (!matchSequence(before, fullArray.slice(0, before.length))) return false;
  
  // 检查 after 部分是否匹配 fullArray 结尾
  if (!matchSequence(after, fullArray.slice(fullArray.length - after.length))) return false;
  
  // $$$ 捕获中间部分
  const captured = fullArray.slice(before.length, fullArray.length - after.length);
  extraData['$$$'] = captured.map(node => ({
    node,
    value: node.name || node.value,
    raw: generate(node),
  }));
  
  return true;
}
```

### 5.5 匹配结果数据结构

每次成功匹配后，引擎返回：

```javascript
{
  nodePathList: [NodePath, NodePath, ...],  // 所有匹配节点的 NodePath
  matchWildCardList: [                      // 每个匹配节点对应的通配符捕获
    {
      '0': [{ node, value, raw }],          // 第 0 个 $_$ 捕获的数据
      '1': [{ node, value, raw }],          // 第 1 个 $_$ 捕获的数据
      '$$$': [{ node, value, raw }, ...]    // $$$ 捕获的多个节点
    },
    // ...
  ]
}
```

---

## 6. AST 操作与转换阶段（Transformation Phase）

### 6.1 AST 类的 jQuery 风格 API

**核心文件**：`packages/gogocode-core/src/Ast.js`

AST 类是对 NodePath 列表的封装，提供类似 jQuery 的链式 API：

```javascript
class AST {
  constructor(nodePath, nodeType, options) {
    // 支持数组索引访问（类 jQuery 设计）
    this[0] = { nodePath, match: null };
    this.length = 1;
    
    // 上下文信息
    this.rootNode = findRoot(nodePath);  // 根节点引用
    this.expando = generateExpando();    // 本次操作的通配符标识
    this.parseOptions = options.parseOptions || {};
  }
}
```

### 6.2 `find()` 方法的完整实现

```javascript
AST.prototype.find = function(selector, options = {}) {
  // 情况 1: selector 是字符串
  if (typeof selector === 'string') {
    const results = [];
    
    // 对每个当前节点执行查找
    for (let i = 0; i < this.length; i++) {
      const { nodePath } = this[i];
      
      // 调用 core.getAstsBySelector 进行 AST 遍历和匹配
      const { nodePathList, matchWildCardList } = core.getAstsBySelector(
        nodePath,
        selector,
        { ...options, expando: this.expando }
      );
      
      // 收集结果
      nodePathList.forEach((path, idx) => {
        results.push({
          nodePath: path,
          match: matchWildCardList[idx]
        });
      });
    }
    
    // 创建新 AST 实例存放结果
    return cloneAST(this, results);
  }
  
  // 情况 2: selector 是 AST 节点对象
  if (isAstNode(selector)) {
    return this.findByNodeObject(selector, options);
  }
  
  return cloneAST(this, []);  // 无匹配
};
```

### 6.3 `replace()` 方法的核心实现

`replace()` 是 GoGoCode 最强大的变换方法，支持字符串模板替换和函数替换两种模式。

```javascript
AST.prototype.replace = function(selector, replacer, options) {
  // 对每个当前匹配节点执行替换
  for (let i = 0; i < this.length; i++) {
    const { nodePath } = this[i];
    
    core.replaceSelBySel(
      nodePath,
      selector,
      replacer,
      options,
      this.parseOptions,
      this.expando
    );
  }
  
  return this; // 链式调用
};
```

**`replaceSelBySel` 内部流程**：

```javascript
function replaceSelBySel(ast, selector, replacer, strictSequence, parseOptions, expando) {
  // 步骤 1: 找到所有匹配
  const { nodePathList, matchWildCardList } = getAstsBySelector(ast, selector, {
    strictSequence, expando
  });
  
  // 步骤 2: 从后往前替换（避免索引偏移问题）
  [...nodePathList].reverse().forEach((path, revIdx) => {
    const idx = nodePathList.length - 1 - revIdx;
    const matchData = matchWildCardList[idx];
    
    // 步骤 3: 处理替换器
    let newCode;
    if (typeof replacer === 'function') {
      // 函数替换器：将捕获数据传入，获取新代码
      newCode = replacer(matchData, path);
    } else if (typeof replacer === 'string') {
      // 字符串模板替换：将 $_$ 替换为捕获值
      newCode = processTemplate(replacer, matchData, expando);
    } else {
      newCode = replacer; // AST 节点直接使用
    }
    
    // 步骤 4: 解析新代码为 AST
    const newAst = buildAstByAstStr(newCode, null, parseOptions);
    
    // 步骤 5: 执行 AST 节点替换
    replaceAstByAst(path, newAst);
  });
}
```

### 6.4 字符串模板替换的通配符处理

当 replacer 是字符串时，其中的 `$_$` 需要被替换为匹配到的实际值：

```javascript
function processTemplate(template, matchData, expando) {
  let result = template;
  
  // 替换 $_$ → 捕获的第 0 个值
  // 替换 $_$1 → 捕获的第 1 个值
  // 替换 $$$ → 捕获的多个值（逗号分隔）
  
  Object.keys(matchData).forEach(wildcardIndex => {
    const captured = matchData[wildcardIndex];
    
    if (wildcardIndex === '$$$') {
      // 多值通配符：将数组元素按原始分隔符连接
      const values = captured.map(item => item.raw).join(', ');
      result = result.replace(/\$\$\$/g, values);
    } else {
      // 单值通配符
      const value = captured[0]?.raw || captured[0]?.value || '';
      const pattern = new RegExp(
        escapeRegExp(expando) + (wildcardIndex === '0' ? '' : `_${wildcardIndex}`)
      );
      result = result.replace(pattern, value);
    }
  });
  
  return result;
}
```

### 6.5 `replaceAstByAst` 节点替换机制

```javascript
function replaceAstByAst(oldPath, newAst) {
  if (Array.isArray(newAst)) {
    // 场景 1: 一对多替换（一个节点替换为多个节点）
    const parent = oldPath.parent;
    const parentKey = oldPath.name;
    
    if (Array.isArray(parent[parentKey])) {
      const index = parent[parentKey].indexOf(oldPath.node);
      parent[parentKey].splice(index, 1, ...newAst);
    }
  } else if (newAst.type === 'BlockStatement') {
    // 场景 2: BlockStatement 展开（{ a; b; } → a; b; 两个语句）
    const body = newAst.body;
    const parent = oldPath.parent;
    const index = parent.body.indexOf(oldPath.node);
    parent.body.splice(index, 1, ...body);
  } else {
    // 场景 3: 一对一替换（常规情况）
    oldPath.replace(newAst);
  }
}
```

### 6.6 其他变换方法

**`attr()` - 属性读写**

```javascript
// 读取属性（支持点分路径）
const name = $(code).find('function $_$(){}').attr('id.name');
// 等价于: matchedNode.id.name

// 设置属性
$(code).find('function a(){}').attr('id.name', 'b');
// 将函数名 a 改为 b
```

**内部实现使用点路径解析**：

```javascript
function getAttrValue(node, attr) {
  return attr.split('.').reduce((obj, key) => {
    return obj ? obj[key] : undefined;
  }, node);
}

function setAttrValue(node, attrMap) {
  Object.entries(attrMap).forEach(([path, value]) => {
    const keys = path.split('.');
    const lastKey = keys.pop();
    const target = keys.reduce((obj, key) => obj[key], node);
    target[lastKey] = value;
  });
}
```

**`append()` / `prepend()` - 插入子节点**

```javascript
// 在函数体末尾插入语句
$(code)
  .find('function a(){}')
  .append('body', 'console.log("end")');

// 内部实现：
function append(ast, attrName, newNode) {
  const targetArray = ast.nodePath.node[attrName];
  const newNodeAst = buildAstByAstStr(newNode).program.body[0];
  targetArray.push(newNodeAst);
}
```

**`before()` / `after()` - 插入兄弟节点**

```javascript
// 内部实现：
function insertBefore(path, nodeList) {
  const parent = path.parent;
  const bodyArray = findBodyArray(parent);
  const index = bodyArray.indexOf(path.node);
  bodyArray.splice(index, 0, ...nodeList);
}
```

**`remove()` - 删除节点**

```javascript
// 有选择器：删除匹配的子节点
$(code).find('class A {}').remove('extends $_$');

// 无选择器：删除当前节点本身
$(code).find('debugger').remove();

// 内部：从父数组中 splice 掉节点
function removePathSafe(path) {
  const parent = path.parent;
  const parentKey = path.name;
  if (Array.isArray(parent[parentKey])) {
    const index = parent[parentKey].indexOf(path.node);
    if (index !== -1) parent[parentKey].splice(index, 1);
  } else {
    parent[parentKey] = null;
  }
}
```

---

## 7. 代码生成阶段（Code Generation Phase）

### 7.1 `generate()` 方法

**核心文件**：`js-core/generate.js`

```javascript
module.exports = function generate(ast, isPretty) {
  if (isPretty) {
    // 格式化打印（整理缩进/换行）
    return recast.prettyPrint(ast, { tabWidth: 2 }).code;
  } else {
    // 保留原格式打印（只重新生成被修改的部分）
    return recast.print(ast).code;
  }
};
```

### 7.2 Recast 的保留格式原理

这是 GoGoCode 相比直接使用 Babel generate 的核心优势。

**Recast 的原理**：

1. **解析时**：在每个 AST 节点上保存 `original` 引用，记录节点在原始源码中的位置
2. **修改时**：被修改的节点失去 `original` 引用，标记为「需要重新生成」
3. **生成时**：遍历 AST 时，如果节点有 `original`，直接输出原始源码切片；否则调用 Babel/Recast 的代码生成器重新生成

```
原始代码: "const a = 1;\nconst b  =  2;" (b有多余空格)
       ↓ parse
AST: VariableDeclaration[0] { original: "const a = 1;" }
     VariableDeclaration[1] { original: "const b  =  2;" }
       ↓ 修改 a 的初始值为 99
AST: VariableDeclaration[0] { original: null }     ← 标记为需重新生成
     VariableDeclaration[1] { original: "const b  =  2;" } ← 保留
       ↓ generate
输出: "const a = 99;\nconst b  =  2;"  ← b 的格式完全保留
```

### 7.3 Vue/HTML 的代码生成

**Vue 生成**：分别对 template/script/style 各部分生成，然后重新拼合为 `.vue` 文件格式。

**HTML 生成**：`html-core/serialize-node.js` 将 HTML AST 节点序列化回 HTML 字符串，保留属性顺序和空白字符。

---

## 8. 多语言抽象层（Multi-Language Abstraction）

### 8.1 语言路由机制

`langCoreMap` 实现了统一 API 到各语言实现的路由：

```javascript
const langCoreMap = {
  js:   require('./js-core/core'),
  html: require('./html-core/core'),
  vue:  require('./vue-core/core'),
};
```

所有语言核心模块暴露相同的接口契约：

```typescript
interface LanguageCore {
  buildAstByAstStr(str: string, map: object, options: object): NodeInfo;
  getAstsBySelector(ast: NodePath, selector: string, opts: object): MatchResult;
  replaceSelBySel(ast: NodePath, sel: string, rep: any, opts: object): void;
  replaceAstByAst(oldAst: NodePath, newAst: NodePath): void;
  generateCode(ast: NodePath, isPretty: boolean): string;
  getParentListByAst(path: NodePath): NodePath[];
  getPrevAst(path: NodePath): NodePath | null;
  getNextAst(path: NodePath): NodePath | null;
}
```

### 8.2 各语言解析器对比

| 属性 | JS/TS | HTML | Vue SFC |
|---|---|---|---|
| **解析器** | `@babel/parser` | `hyntax-yx` | `vue3-browser-compiler-yx` |
| **封装层** | `recast-yx` | 无 | 无 |
| **节点遍历** | Recast `visit()` | 自定义 DFS | 路由至 HTML/JS core |
| **代码生成** | `recast.print()` | `serialize-node.js` | 分段生成后拼合 |
| **通配符** | `$_$` / `$$$` | `$_$` / `$$$` | 按块分流 |
| **位置保留** | Recast 保留 | 字符偏移 | 按块保留 |

---

## 9. 插件系统架构（Plugin System）

### 9.1 插件的标准接口

GoGoCode 插件是遵循以下接口的 Node.js 模块：

```javascript
module.exports = {
  // 可选：在转换开始前对整个项目进行全局扫描
  preTransform(api, options) {
    // api.gogocode: $ 函数
    // options: 用户传入的配置
    // 通常用于收集全局信息（如所有 Vue.component 注册）
  },
  
  // 必选：对单个文件执行转换
  transform(fileInfo, api, options) {
    // fileInfo.source: 源码字符串
    // fileInfo.path: 文件路径
    // api.gogocode: $ 函数
    // options: 用户传入的配置（preTransform 阶段收集的数据也在此）
    
    const $ = api.gogocode;
    const ast = $(fileInfo.source, {
      parseOptions: { language: 'js' }
    });
    
    // 执行转换...
    ast.find('Vue.set($_$, $_$, $_$)')
       .replace('Vue.set($_$, $_$, $_$)', '($_$[$_$] = $_$)');
    
    return ast.generate();
  },
  
  // 可选：在所有文件处理完成后进行清理
  postTransform(options) { }
};
```

### 9.2 CLI 执行插件的管道

**核心文件**：`packages/gogocode-cli/src/commands/transform.js`

```javascript
async function runTransform(src, transform, output, options) {
  // 步骤 1: 加载插件
  const plugins = await requireTransforms(transform);
  
  // 步骤 2: 执行 preTransform（全局扫描）
  for (const plugin of plugins) {
    if (plugin.preTransform) {
      await plugin.preTransform({ gogocode: $ }, options);
    }
  }
  
  // 步骤 3: 获取所有需要处理的文件列表
  const files = getFilesRecursively(src, {
    ignore: BINARY_FILE_EXTENSIONS  // 忽略图片、字体等二进制文件
  });
  
  // 步骤 4: 显示进度条
  const progress = new ProgressBar(files.length);
  
  // 步骤 5: 逐文件处理
  for (const file of files) {
    const source = fs.readFileSync(file, 'utf-8');
    
    // 插件链：每个插件的输出作为下一个插件的输入
    let result = source;
    for (const plugin of plugins) {
      if (plugin.transform) {
        result = await plugin.transform(
          { source: result, path: file },
          { gogocode: $ },
          options
        );
      }
    }
    
    // 步骤 6: 写入结果（非 dry run 模式）
    if (!options.dry) {
      const outputPath = getOutputPath(file, src, output);
      fs.writeFileSync(outputPath, result, 'utf-8');
    }
    
    progress.tick();
  }
  
  // 步骤 7: 执行 postTransform
  for (const plugin of plugins) {
    if (plugin.postTransform) {
      await plugin.postTransform(options);
    }
  }
}
```

### 9.3 gogocode-plugin-vue 插件深度解析

**文件结构**：

```
gogocode-plugin-vue/src/
├── index.js                    # 插件入口，串联所有转换规则
├── collection.js               # preTransform: 收集全局 Vue 信息
├── vue-helpers.js              # 通用辅助函数
└── vue-transform-rules/        # 具体转换规则（每条规则一个文件）
    ├── app-mount.js            # Vue() → createApp().mount()
    ├── filter-transform.js     # Vue filters → 方法/计算属性
    ├── global-api-transform.js # Vue.xxx → 具名导出
    ├── vue-config-transform.js # Vue.config.xxx → app.config.xxx
    ├── vue-router-transform.js # Vue Router 4 迁移
    ├── vuex-transform.js       # Vuex 4 迁移
    └── ... (30+ 个规则文件)
```

**`collection.js` preTransform 阶段**：

```javascript
// preTransform: 扫描整个项目，收集：
// 1. 所有通过 Vue.component('name', def) 注册的全局组件名
// 2. 所有通过 Vue.config.keyCodes 配置的自定义键码
// 3. 所有通过 Vue.filter('name', fn) 注册的全局过滤器

module.exports.preTransform = function({ gogocode: $ }, options) {
  const globalComponents = new Set();
  const globalKeyCodes = {};
  
  // 扫描所有 .js 文件中的 Vue.component 调用
  options.filePaths.forEach(filePath => {
    if (!/\.js$/.test(filePath)) return;
    
    const source = fs.readFileSync(filePath, 'utf-8');
    const ast = $(source);
    
    ast.find('Vue.component($_$, $_$)').each(node => {
      const name = node.match[0]?.[0]?.value;
      if (name) globalComponents.add(name);
    });
  });
  
  options._globalComponents = globalComponents;
  options._globalKeyCodes = globalKeyCodes;
};
```

**一条转换规则的实现示例（`app-mount.js`）**：

将 Vue 2 的初始化方式迁移到 Vue 3：

```javascript
// Vue 2: new Vue({ el: '#app', render: h => h(App) })
// Vue 3: createApp(App).mount('#app')

module.exports = function transformAppMount(ast, api, options) {
  const $ = api.gogocode;
  
  // 匹配: new Vue({ el: '#app', render: h => h(App) })
  ast.find('new Vue({$$$})').each(vueInstance => {
    const options = vueInstance.match['$$$'];
    
    const elProp = options.find(prop => prop.raw?.includes('el:'));
    const renderProp = options.find(prop => prop.raw?.includes('render:'));
    
    if (elProp && renderProp) {
      // 提取 el 值
      const elValue = extractElValue(elProp);
      // 提取 App 组件
      const appComponent = extractAppComponent(renderProp);
      
      // 替换为 Vue 3 语法
      vueInstance.replaceBy(
        `createApp(${appComponent}).mount(${elValue})`
      );
    }
  });
  
  return ast;
};
```

---

## 10. CLI 工具实现原理（CLI Implementation）

### 10.1 命令行解析

**核心文件**：`packages/gogocode-cli/index.js`

使用 `commander` 解析命令行参数：

```javascript
const program = require('commander');

program
  .command('transform')
  .alias('t')
  .description('Run code transformation')
  .option('-s, --src <path>',       '源文件/目录路径')
  .option('-t, --transform <path>', '转换插件路径（支持逗号分隔多个）')
  .option('-o, --out <path>',       '输出目录（省略则原地修改）')
  .option('-d, --dry',              'Dry run：只输出变换结果，不写文件')
  .option('-p, --params <json>',    '传给插件的自定义参数（JSON 字符串）')
  .option('-i, --info',             '显示详细日志')
  .action(async (opts) => {
    await transformCommand(opts);
  });

program
  .command('init')
  .description('初始化插件开发项目')
  .action(async () => {
    await initCommand();
  });

program.parse(process.argv);
```

### 10.2 文件遍历策略

```javascript
function getFilesRecursively(srcPath, options) {
  const stats = fs.statSync(srcPath);
  
  if (stats.isFile()) {
    return [srcPath];
  }
  
  if (stats.isDirectory()) {
    const entries = fs.readdirSync(srcPath);
    const files = [];
    
    for (const entry of entries) {
      const fullPath = path.join(srcPath, entry);
      const entryStats = fs.statSync(fullPath);
      
      if (entryStats.isDirectory()) {
        // 递归遍历子目录（排除 node_modules、.git 等）
        if (!IGNORED_DIRS.includes(entry)) {
          files.push(...getFilesRecursively(fullPath, options));
        }
      } else if (entryStats.isFile()) {
        // 过滤二进制文件
        if (!isBinaryFile(fullPath)) {
          files.push(fullPath);
        }
      }
    }
    
    return files;
  }
  
  return [];
}

// 二进制文件扩展名黑名单
const BINARY_EXTENSIONS = [
  '.gif', '.jpg', '.jpeg', '.png', '.svg', '.ico',
  '.woff', '.woff2', '.ttf', '.eot', '.otf',
  '.mp4', '.mp3', '.avi', '.mov',
  '.zip', '.tar', '.gz', '.rar',
  '.pdf', '.doc', '.docx', '.xls', '.xlsx',
];
```

### 10.3 插件加载机制

```javascript
async function requireTransforms(transformPaths) {
  const paths = transformPaths.split(',').map(p => p.trim());
  
  return paths.map(transformPath => {
    // 支持 npm 包名
    if (!transformPath.startsWith('.') && !path.isAbsolute(transformPath)) {
      return require(transformPath);
    }
    
    // 支持本地文件路径
    const absolutePath = path.resolve(process.cwd(), transformPath);
    return require(absolutePath);
  });
}
```

### 10.4 输出路径计算

```javascript
function getOutputPath(filePath, srcRoot, outputRoot) {
  if (!outputRoot) {
    // 没有指定输出目录：原地修改
    return filePath;
  }
  
  // 计算相对路径并映射到输出目录
  const relativePath = path.relative(srcRoot, filePath);
  const outputPath = path.join(outputRoot, relativePath);
  
  // 确保输出目录存在
  fs.mkdirSync(path.dirname(outputPath), { recursive: true });
  
  return outputPath;
}
```

---

## 11. 核心数据结构全景

### 11.1 AST 实例结构

```
AST {
  [0]: { nodePath: NodePath, match: MatchData | null }
  [1]: { nodePath: NodePath, match: MatchData | null }
  ... (length 个)
  
  length: number            // 匹配节点总数
  rootNode: NodePath        // 根 Program 节点
  expando: string           // 本次操作的通配符唯一标识
  parseOptions: {           // 解析配置
    language: 'js' | 'html' | 'vue'
  }
}
```

### 11.2 NodePath 结构

```
NodePath {
  node: ASTNode             // 实际 AST 节点
  value: ASTNode            // 同 node（Recast 兼容）
  parent: ASTNode           // 父 AST 节点
  parentPath: NodePath      // 父 NodePath（用于向上导航）
  name: string              // 在父节点中的属性名
  __childCache: {           // 子 NodePath 缓存
    [key]: NodePath
  }
}
```

### 11.3 选择器结构

```
Selector {
  nodeType: string          // 目标 AST 节点类型
  structure: {              // 模式结构（已过滤无关属性）
    [prop]: value | expando | Selector  // 递归嵌套
  }
}
```

### 11.4 MatchData 结构

```
MatchData {
  '0': [MatchItem]          // 第 0 个 $_$ 捕获
  '1': [MatchItem]          // 第 1 个 $_$ 捕获
  '2': [MatchItem]          // 第 2 个 $_$ 捕获
  '$$$': [MatchItem, ...]   // $$$ 多值捕获
}

MatchItem {
  node: ASTNode             // 捕获的 AST 节点
  value: string | number    // 节点的基本值（标识符名/字面量值）
  raw: string               // 节点生成的代码字符串
}
```

### 11.5 语言核心映射

```
langCoreMap {
  'js':   JSCore  { buildAstByAstStr, getAstsBySelector, replaceSelBySel, ... }
  'html': HTMLCore { buildAstByAstStr, getAstsBySelector, replaceSelBySel, ... }
  'vue':  VueCore  { buildAstByAstStr, getAstsBySelector, replaceSelBySel, ... }
}
```

---

## 12. 关键 API 深度解析

### 12.1 `.find(selector, options)` 完整参数

```javascript
// options 说明
{
  ignoreSequence: false,   // true: 数组匹配时忽略顺序（用于对象属性匹配）
  strictSequence: true,    // false: 允许数组中部分匹配（用于超集匹配）
  deep: true,              // false: 只在当前层级查找，不递归子节点
}
```

### 12.2 `.replace(selector, replacer, options)` 完整用法

```javascript
// 1. 字符串替换（基础用法）
ast.replace('var $_$ = $_$', 'let $_$ = $_$');

// 2. 函数替换（高级用法，可获取捕获数据）
ast.replace('const $_$ = require($_$)', (match, path) => {
  const varName = match['0'][0].value;    // 变量名
  const modulePath = match['1'][0].value; // 模块路径
  return `import ${varName} from '${modulePath}'`;
});

// 3. 带选项
ast.replace('{ $_$: $_$ }', '{ $_$: $_$ }', {
  ignoreSequence: true  // 对象属性顺序不敏感
});
```

### 12.3 `.attr(key, value?)` 点路径示例

```javascript
// 读取深层属性
const funcName = ast.find('function $_$(){}').attr('id.name');
// → 'functionName'

// 读取数组元素
const firstParam = ast.find('function a($_$){}').attr('params.0.name');
// → 'paramName'

// 设置属性
ast.find('class $_$ {}').attr({ 'id.name': 'NewClassName' });

// 设置多个属性
ast.find('import $_$ from $_$').attr({
  'specifiers.0.local.name': 'newName',
  'source.value': './new-path'
});
```

### 12.4 `.each(callback)` 迭代用法

```javascript
const results = [];
ast.find('const $_$ = $_$').each((node, index) => {
  // node: 当前匹配的 AST 实例
  // index: 当前索引
  const varName = node.match['0'][0].value;
  const value = node.match['1'][0].raw;
  results.push({ varName, value });
});
```

### 12.5 `.parent()` 向上导航

```javascript
// 获取直接父节点
const parent = ast.find('const a = 1').parent();

// 获取 N 层父节点
const grandParent = ast.find('const a = 1').parent(2);

// 获取满足条件的父节点
const funcParent = ast.find('console.log()').parent(node => 
  node.nodePath.node.type === 'FunctionDeclaration'
);
```

---

## 13. 通配符匹配机制详解

### 13.1 `$_$` 单值通配符

`$_$` 能匹配任意单个 AST 节点：

| 模式 | 可匹配 | 不可匹配 |
|---|---|---|
| `const a = $_$` | `const a = 1` | `const a = 1, b = 2` |
| `foo($_$)` | `foo(bar)`, `foo(1)` | `foo(a, b)` |
| `$_$.value` | `obj.value`, `arr.value` | `(1+2).value`（部分情况） |
| `[$_$]` | `[1]`, `['str']` | `[1, 2]` |

**命名通配符**：`$_$0`, `$_$1` 允许在替换模板中引用特定捕获：

```javascript
// 交换两个变量的值
ast.replace('[$_$0, $_$1] = [$_$1, $_$0]', '[$_$1, $_$0] = [$_$0, $_$1]');
// 错误示例：上面语法不完全准确，实际需根据具体 API 使用
```

### 13.2 `$$$` 多值通配符

`$$$` 能匹配任意数量（包括零个）的 AST 节点：

| 模式 | 可匹配 |
|---|---|
| `foo($$$)` | `foo()`, `foo(a)`, `foo(a, b, c, d)` |
| `[1, $$$]` | `[1]`, `[1, 2]`, `[1, 2, 3, 'x']` |
| `{ a: 1, $$$ }` | `{ a: 1 }`, `{ a: 1, b: 2, c: 3 }` |
| `function name($$$) {}` | 任意参数列表的函数 |

**在替换中使用 `$$$`**：

```javascript
// 在函数调用前添加一个额外参数
ast.replace('foo($$$)', 'foo(extraArg, $$$)');

// foo(a, b) → foo(extraArg, a, b)
// foo()     → foo(extraArg)
```

### 13.3 通配符组合使用

```javascript
// 匹配 Vue 2 的 $set 调用并转换
ast.replace(
  'this.$set($_$, $_$, $_$)',    // 匹配 this.$set(obj, key, val)
  '($_$[$_$] = $_$)'             // 替换为 (obj[key] = val)
);

// 匹配 require 并转换为 import
ast.replace(
  'const $_$ = require($_$)',
  (match) => {
    const varName = match['0'][0].value;
    const modulePath = match['1'][0].value;
    return `import ${varName} from ${modulePath}`;
  }
);
```

---

## 14. Vue 单文件组件处理流程

### 14.1 Vue SFC 的解析策略

**核心文件**：`vue-core/core.js`

Vue 的 `.vue` 文件包含三个块：`<template>`, `<script>`, `<style>`。

```javascript
function buildAstByAstStr(str, map, options) {
  // 使用 vue3-browser-compiler-yx 解析 .vue 文件
  const descriptor = vueCompiler.parse(str).descriptor;
  
  // 分别保存各块
  return {
    template: descriptor.template,    // 模板块
    script: descriptor.script,        // 普通 <script>
    scriptSetup: descriptor.scriptSetup, // <script setup>
    styles: descriptor.styles,        // 样式块
    
    // 缓存各块的 AST（懒加载）
    _templateAst: null,
    _scriptAst: null,
    _scriptSetupAst: null,
  };
}
```

### 14.2 选择器路由至正确的块

```javascript
function getAstsBySelector(ast, selector, options) {
  // 判断选择器目标块类型
  if (isTemplateSelector(selector)) {
    // HTML 选择器（如 '<div class="$_$">'）→ 路由至 HTML core
    const templateAst = getOrParseTemplate(ast);
    return htmlCore.getAstsBySelector(templateAst, selector, options);
  } else {
    // JS/TS 选择器 → 路由至 JS core
    const scriptAst = getOrParseScript(ast);
    const results = jsCore.getAstsBySelector(scriptAst, selector, options);
    
    // 如果有 <script setup>，也搜索它
    if (ast.scriptSetup) {
      const setupAst = getOrParseScriptSetup(ast);
      const setupResults = jsCore.getAstsBySelector(setupAst, selector, options);
      mergeResults(results, setupResults);
    }
    
    return results;
  }
}
```

### 14.3 Vue 文件的代码生成

```javascript
function generateCode(ast, isPretty) {
  const parts = [];
  
  // 生成 template 部分
  if (ast.template) {
    const templateCode = htmlCore.generateCode(ast._templateAst || ast.template);
    parts.push(`<template>\n${templateCode}\n</template>`);
  }
  
  // 生成 script 部分
  if (ast.script) {
    const scriptCode = jsCore.generateCode(ast._scriptAst || ast.script);
    parts.push(`<script>\n${scriptCode}\n</script>`);
  }
  
  // 生成 script setup 部分
  if (ast.scriptSetup) {
    const setupCode = jsCore.generateCode(ast._scriptSetupAst || ast.scriptSetup);
    parts.push(`<script setup>\n${setupCode}\n</script>`);
  }
  
  // 保留 style 部分（通常不变换）
  ast.styles.forEach(style => {
    const lang = style.lang ? ` lang="${style.lang}"` : '';
    const scoped = style.scoped ? ' scoped' : '';
    parts.push(`<style${lang}${scoped}>\n${style.content}\n</style>`);
  });
  
  return parts.join('\n\n');
}
```

---

## 15. HTML 文件处理流程

### 15.1 HTML 解析器

**解析器**：`hyntax-yx`（hyntax 的定制 fork）

hyntax 是一个专注于 HTML 解析的工具，特点：
- 保留全部源码信息（属性、注释、空白字符）
- 生成带有位置信息的 Token 流
- 构建完整的 HTML AST 树

**解析流程**：
1. **词法分析（Tokenization）**：将 HTML 字符串切分为 Token（开标签、文本、属性等）
2. **句法分析（Tree Construction）**：将 Token 流组装为树形 AST
3. **NodePath 封装**：为每个节点附加 `parentRef` 用于向上导航

### 15.2 HTML 选择器的特殊处理

HTML 选择器（如 `<div class="$_$">` 或 `<MyComponent :prop="$_$">`) 需要特殊处理：

```javascript
// HTML 选择器会被解析为 HTML AST 片段
// <div class="foo"> 匹配 <div class="foo"> ... </div>
// <div> 匹配任意 <div> 元素

function getHtmlSelector(selectorStr, expando) {
  // 将 $_$ 替换为 expando 标记
  const withExpando = selectorStr.replace(/\$_\$/g, expando);
  
  // 解析为 HTML AST
  const ast = htmlParse(withExpando);
  
  // 提取元素节点类型和属性结构
  const element = findFirstElement(ast);
  return {
    nodeType: 'element',
    tagName: element.tagName,
    attributes: extractAttributes(element, expando),
  };
}
```

---

## 16. 错误处理与容错机制

### 16.1 解析错误处理

```javascript
function buildAstByAstStr(str, map, options) {
  try {
    // 尝试标准解析
    return parseAsStatement(str);
  } catch (e1) {
    try {
      // 尝试作为表达式解析
      return parseAsExpression(str);
    } catch (e2) {
      try {
        // 尝试特殊构建器
        return buildMap.tryBuild(str);
      } catch (e3) {
        // 所有方式都失败，抛出描述性错误
        throw new Error(
          `GoGoCode: Failed to parse "${str}"\n` +
          `Original error: ${e1.message}`
        );
      }
    }
  }
}
```

### 16.2 匹配失败处理

匹配失败不是错误，而是返回空结果集：

```javascript
// 当没有匹配时，返回 length=0 的 AST 实例
// 链式调用可以安全继续，不会抛错
$(code).find('nonexistentPattern').replace('...', '...').generate();
// ↑ 即使没找到，generate() 仍然正常返回原始代码
```

### 16.3 替换器错误处理

```javascript
function replaceSelBySel(ast, selector, replacer, ...) {
  // 如果替换器是函数，捕获其中的错误
  if (typeof replacer === 'function') {
    try {
      const newCode = replacer(matchData, path);
      // ...
    } catch (e) {
      console.warn(`GoGoCode: replacer function threw an error: ${e.message}`);
      return; // 跳过这个替换，继续处理其他匹配
    }
  }
}
```

---

## 17. 性能优化设计

### 17.1 延迟初始化（Lazy Initialization）

父节点链和兄弟节点列表只在第一次访问时计算：

```javascript
function initParent(ast) {
  // 只在第一次调用 .parent() 时遍历建立父节点引用
  if (ast._parentInitialized) return;
  
  traverse(ast.rootNode, (node, parent) => {
    node.__parent = parent;
  });
  
  ast._parentInitialized = true;
}
```

### 17.2 `__childCache` 子节点缓存

NodePath 的 `get()` 方法使用缓存避免重复创建 NodePath 对象：

```javascript
NodePath.prototype.get = function(key) {
  if (!this.__childCache[key]) {
    this.__childCache[key] = new NodePath(
      this.node[key], this.node, this, key
    );
  }
  return this.__childCache[key];
};
```

### 17.3 从后向前替换策略

当同一父数组中有多个节点需要替换时，**从后往前替换**避免索引错位：

```javascript
// 错误：从前往后替换会导致索引偏移
nodePathList.forEach(path => replaceAstByAst(path, newAst)); // ❌

// 正确：从后往前替换保持索引稳定
[...nodePathList].reverse().forEach(path => replaceAstByAst(path, newAst)); // ✅
```

### 17.4 Recast 的增量代码生成

Recast 通过 `original` 标记实现增量代码生成，只重新生成被修改的节点，大幅减少代码生成的计算量和不必要的格式变化。

---

## 18. 扩展性设计原则

### 18.1 开闭原则（OCP）

通过插件系统实现扩展：添加新的转换规则**不需要修改 gogocode-core**，只需创建新的插件。

### 18.2 语言无关的统一 API

无论处理 JS、HTML 还是 Vue，用户使用完全相同的 `$()` API，语言差异被封装在内部的 language core 中。

### 18.3 声明式变换

用户通过声明「什么变成什么」来描述变换，而不是手动操作 AST 节点，使变换规则更具可读性和可维护性：

```javascript
// 声明式（GoGoCode 风格）✅
ast.replace('Vue.set($_$, $_$, $_$)', '($_$[$_$] = $_$)');

// 命令式（直接操作 AST）❌
ast.find('CallExpression').filter(node => {
  return node.callee.type === 'MemberExpression' &&
    node.callee.object.name === 'Vue' &&
    node.callee.property.name === 'set';
}).forEach(node => {
  // 手动构建新 AST...
});
```

### 18.4 可组合的变换管道

插件是可组合的——多个插件可以串联形成变换管道，每个插件专注于一类变换，共同完成复杂迁移：

```bash
gogocode -s ./src \
  -t gogocode-plugin-vue,gogocode-plugin-element \
  -o ./src-vue3
```

---

## 总结

GoGoCode 的整个处理管道可以精炼为以下七个阶段：

```
① 语言检测  →  ② AST 解析  →  ③ 选择器编译  →  ④ 模式匹配
                                                      ↓
⑦ 代码输出  ←  ⑥ 代码生成  ←  ⑤ 节点变换（replace/append/remove）
```

**每阶段的核心技术**：

| 阶段 | 核心技术 | 关键文件 |
|---|---|---|
| 语言检测 | parseOptions.language 路由 + langCoreMap | `$.js` |
| AST 解析 | @babel/parser + recast-yx 封装 | `js-core/parse.js` |
| 选择器编译 | 字符串→AST + expando 通配符替换 + 属性过滤 | `js-core/get-selector.js` |
| 模式匹配 | Recast visitor + checkIsMatch 递归算法 | `js-core/find/general.js` |
| 节点变换 | AST 原地修改 + 从后向前替换策略 | `js-core/core.js` |
| 代码生成 | Recast 增量打印（保留原始格式） | `js-core/generate.js` |
| 代码输出 | 文件 I/O + 进度条 + 插件管道 | `gogocode-cli/transform.js` |

GoGoCode 通过这套精心设计的流水线，将复杂的 AST 操作封装成简洁的声明式 API，使大规模代码迁移（如 Vue 2→3 升级）从数周的手工工作缩短为分钟级的自动化操作。
