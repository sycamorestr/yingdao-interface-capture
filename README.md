# Yingdao Interface Capture

一个面向影刀（Yingdao）网页自动化的 Codex Skill。它用于检查已经登录的业务后台页面，优先捕获并复用页面真实使用的接口、请求运行时、webpack 模块、SDK 或已加载状态，最终生成可直接放入影刀“网页执行 JavaScript”动作中的脚本。

## 核心原则

本 Skill 优先使用稳定的数据路径，而不是模拟鼠标点击：

```text
页面已加载状态
→ 页面业务函数、SDK、请求封装、webpack 模块或 mtop
→ 捕获 fetch / XMLHttpRequest
→ 页面上下文中的同源 fetch
→ UI 操作（仅在用户明确要求时）
```

这样生成的影刀脚本通常更稳定，也更适合分页查询、批量处理、异步导出和结构化结果回传。

## 适用场景

- 抓取已登录 ERP、广告投放、电商或财务后台的数据。
- 定位页面实际调用的 API、请求参数和响应字段。
- 处理 Cookie、CSRF Token、动态签名、webpack 或 mtop 运行时。
- 生成或修复影刀网页 JavaScript。
- 处理多层 iframe、动态表单和异步导出任务。
- 将长耗时任务拆成“启动任务 + 轮询状态”脚本。
- 在明确授权后执行批量创建、修改、作废等业务操作，并进行最终核验。

不适合直接用作通用桌面 RPA、独立 HTTP 客户端或浏览器扩展生成器。

## 仓库结构

```text
yingdao-interface-capture/
├── README.md
├── SKILL.md
└── agents/
    └── openai.yaml
```

- `SKILL.md`：完整工作流、约束、接口捕获方法和经验规则。
- `agents/openai.yaml`：Skill 的显示名称、简介和默认提示词。

## 安装

将本仓库克隆到 Codex 的个人 Skills 目录：

### Windows PowerShell

```powershell
git clone https://github.com/sycamorestr/yingdao-interface-capture.git "$env:USERPROFILE\.codex\skills\yingdao-interface-capture"
```

### macOS / Linux

```bash
git clone https://github.com/sycamorestr/yingdao-interface-capture.git "$HOME/.codex/skills/yingdao-interface-capture"
```

重启或刷新 Codex 后，即可通过 `$yingdao-interface-capture` 调用。

## 使用方式

调用时应先打开并登录目标业务后台，然后把准确的目标页面切到前台。

示例提示词：

```text
使用 $yingdao-interface-capture 检查我当前打开的后台页面，找出商品列表接口，并生成影刀可运行的网页 JavaScript。输入是商品 ID 列表，输出保留输入顺序。
```

```text
使用 $yingdao-interface-capture 分析当前报表页面的导出流程，生成启动导出任务、轮询状态和提取下载地址的影刀脚本。
```

如果确实需要页面操作，应明确说明：

```text
使用 $yingdao-interface-capture 操作当前页面的日期控件，选择昨天，点击查询，并验证页面显示的日期范围。
```

## 标准工作流

1. 绑定用户已经打开的目标标签页。
2. 核对当前 URL、页面标题和 iframe 结构。
3. 按优先级寻找已加载状态、业务函数、SDK、webpack 模块或网络接口。
4. 对未知接口安装临时 `fetch` / XHR 监听，再触发最小的只读查询。
5. 用小范围真实数据验证参数、日期、分页、单位和响应路径。
6. 输出影刀可直接运行的函数与一行式输入、结果解析表达式。
7. 停止监听并解绑浏览器会话，不关闭用户标签页。

## 影刀脚本约定

最终脚本通常采用以下入口：

```js
async function (element, input) {
  var result = {
    ok: true,
    blocked: false,
    message: "",
    rawInput: input,
    href: location.href,
    data: []
  };

  return JSON.stringify(result);
}
```

影刀输入被视为字符串边界。结构化参数建议使用一行 Python 表达式序列化：

```python
__import__('json').dumps({"date": str(run_date), "shop": str(shop_name)}, ensure_ascii=False)
```

结果也使用一行表达式提取：

```python
__import__('json').loads(js_result).get("data", [])
```

## 异步导出和长任务

对于耗时较长的接口调用或报表导出，推荐拆分为：

```text
启动脚本 → 返回 jobId → 影刀定时轮询状态脚本 → 返回最终数据或 downloadUrl
```

页面内存中的任务会在刷新、跳转或关闭页面后丢失，因此执行过程中应保持目标页面不变。

对于文件导出，网页 JavaScript 主要负责取得 `taskId` 和 `downloadUrl`；如需保存到任意本地目录，应在影刀中使用独立的下载动作或 PowerShell `Invoke-WebRequest`。

## 安全和操作边界

- 默认只做只读检查；写操作必须由用户明确要求。
- 不凭接口名称猜测请求方法或参数，必须根据实时页面证据确认。
- 不创建空白页探测已登录后台，不随意跳转或关闭用户标签页。
- 遇到 CAPTCHA、二次验证、强风控或登录失效时返回明确阻塞信息。
- 批量写操作应提供预检、正式执行和最终验证阶段。
- 不把 Cookie、Token、账号数据、真实业务响应或本地绝对路径写入仓库。

## 进一步阅读

完整规则、捕获模板、影刀兼容性说明以及抖店优惠券、聚水潭财务导出等经验见 [`SKILL.md`](./SKILL.md)。
