# Codex 操作工作流

## 1. 启动与诊断

没有专用浏览器 MCP 时使用 `agent-browser`：

```powershell
agent-browser skills get core
agent-browser --session cnkidata --headed --download-path "绝对输出目录" open "https://kns.cnki.net/kns8s/AdvSearch"
agent-browser --session cnkidata snapshot -i -u
```

启动失败时先运行 `agent-browser doctor --offline --quick`。若本机已有可调试的 Chrome，可尝试 `--auto-connect`；不要执行带 `--fix` 的修复，除非用户另行批准。

页面发生跳转、搜索、翻页、弹窗或切换标签后，旧的 `@eN` 引用立即失效，必须重新 `snapshot -i`。

## 2. 检索

优先通过快照找到字段选择器、搜索框和搜索按钮。依次完成：

1. 选择用户指定字段；期刊任务选择“文献来源”。
2. 输入精确刊名，等待自动补全。
3. 勾选与目标刊名完全一致的候选项。
4. 限定目标年份并执行检索。
5. 将每页条数设为 50，按发表时间降序。

交互元素没有稳定引用时，使用语义定位或 CSS 选择器。复杂 DOM 操作用 `agent-browser eval -b <base64>`，避免 PowerShell 转义破坏 JavaScript。

## 3. 单页选择与年份判断

页面全选的最小脚本如下。执行后必须读取页面“已选”计数确认生效：

```javascript
async () => {
  window.alert = () => {};
  const sleep = ms => new Promise(r => setTimeout(r, ms));
  const cb = document.querySelector('#selectCheckAll1');
  if (!cb) return { error: 'select_all_not_found' };
  cb.checked = false;
  await sleep(100);
  cb.dispatchEvent(new MouseEvent('click', { bubbles: true, cancelable: true }));
  await sleep(300);
  if (typeof cb.onclick === 'function') cb.onclick();
  await sleep(500);
  const dates = document.body.innerText.match(/\d{4}-\d{2}-\d{2}/g) || [];
  return { dates, text: document.body.innerText.match(/已选\s*\d+/)?.[0] || '' };
}
```

只选择日期仍落在目标年份范围内的结果。若检索页混有其他年份，优先使用 CNKI 年份筛选，不要依赖事后截断来掩盖漏选或多选。

## 4. 导出

1. 点击“导出与分析”与“导出文献”。
2. 精确点击“自定义”，不要点击名称中包含额外文字的选项。
3. 切换到新打开的导出标签页并重新快照。
4. 在 `ul.formatlist li` 中再次选择文本恰为“自定义”的项。
5. 全选字段并核对导出列数。
6. 使用 `agent-browser download <selector> <绝对文件路径>` 点击 xls；若下载由新标签或脚本触发，则设置 `--download-path` 后点击 xls。
7. 等待 15 秒并检查是否出现验证码。出现时保持有头浏览器打开，请用户处理后继续。

每批文件使用不会覆盖的名称，例如 `CNKI-教育研究-2019-batch01.xls`。

## 5. 合并与核验

CNKI 的 `.xls` 可能实际是 HTML 表格。先尝试 `pandas.read_excel`，失败时使用 `pandas.read_html`。合并逻辑：

```python
from pathlib import Path
import pandas as pd

target = "目标刊名"
year_from = year_to = 2019
frames = []

for path in Path("下载目录").glob("CNKI-*.xls*"):
    try:
        frame = pd.read_excel(path)
    except Exception:
        frame = pd.read_html(path, encoding="utf-8")[0]
        if "Title-题名" not in frame.columns:
            frame.columns = frame.iloc[0]
            frame = frame.iloc[1:]
    if len(frame.columns) == 18:
        frames.append(frame[frame["Source-文献来源"] == target])

merged = pd.concat(frames, ignore_index=True)
before = len(merged)
merged["Year-年"] = pd.to_numeric(merged["Year-年"], errors="coerce")
merged = merged[merged["Year-年"].between(year_from, year_to)]
merged = merged.drop_duplicates(subset="Title-题名")
merged = merged.sort_values(["Year-年", "Period-期", "PageCount-页码"], kind="stable")
print({
    "导出前记录": before,
    "最终记录": len(merged),
    "重复记录": before - len(merged),
    "字段数": len(merged.columns),
    "来源": merged["Source-文献来源"].dropna().unique().tolist(),
    "各期篇数": merged.groupby("Period-期").size().to_dict(),
})
merged.to_excel("目标刊名_年份.xlsx", index=False)
```

空文件、没有任何 18 列批次、来源不唯一、缺期或字段数不等于 18 时停止交付并回到对应步骤补采。
