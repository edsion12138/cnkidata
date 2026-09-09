# 逐页全选 + 导出 代码模板

以下代码通过 MCP Playwright 的 `browser_evaluate` / `browser_run_code_unsafe` 执行。核心原则：DOM 操作全用 `page.evaluate`（绕过可见性检查），导出菜单点击用 Playwright locator + `force:true`。

## 检索 + 精准锁定 + 排序 + 50条/页

```javascript
// 参数：field = 'LY'(文献来源) | 'SU'(主题) | 'TI'(篇名) | 'AU'(作者) | 'FU'(基金) 等
// 参数：query = 检索词；needCheckbox = 是否需要精准复选框
async function search(field, query, needCheckbox) {
  window.alert = function(){};
  document.querySelector('.sort-default')?.click();          // 展开字段下拉
  await new Promise(r => setTimeout(r, 500));
  (document.querySelector('li[data-val="' + field + '"] a')
    || document.querySelector('a[title="文献来源"]'))?.click();  // 选字段
  await new Promise(r => setTimeout(r, 300));
  document.querySelector('.sort-default')?.click();          // 折叠下拉（关键）
  await new Promise(r => setTimeout(r, 300));

  const inp = document.querySelector('input.search-input');
  const ns = Object.getOwnPropertyDescriptor(HTMLInputElement.prototype, 'value').set;
  ns.call(inp, query);
  inp.dispatchEvent(new Event('input', {bubbles: true}));
  await new Promise(r => setTimeout(r, 2500));                // 等自动补全

  if (needCheckbox) {
    // 勾选精确刊名左侧复选框
    for (const cb of document.querySelectorAll('input[type="checkbox"]')) {
      const t = cb.parentElement?.innerText?.trim() || '';
      if (t === query) { cb.checked = true; cb.click();
        cb.dispatchEvent(new Event('change', {bubbles: true})); break; }
    }
  }
  document.querySelector('.sort-default')?.click();          // 再折叠
  await new Promise(r => setTimeout(r, 300));
  document.querySelector('input.search-btn')?.click();

  // 等结果 + 50条/页 + 发表时间降序
  for (let i=0;i<60;i++){ if(document.body.innerText.includes('条结果')) break;
    await new Promise(r=>setTimeout(r,500)); }
  await new Promise(r => setTimeout(r, 2000));
  document.querySelector('#perPageDiv a')?.click();
  await new Promise(r => setTimeout(r, 500));
  for (const a of document.querySelectorAll('#perPageDiv a'))
    if (a.textContent.trim()==='50') { a.click(); break; }
  await new Promise(r => setTimeout(r, 2000));
  for (const el of document.querySelectorAll('a,span,li'))
    if (el.innerText?.trim()==='发表时间') { el.click(); break; }
  await new Promise(r => setTimeout(r, 800));
  for (const el of document.querySelectorAll('a,span,li'))
    if (el.innerText?.trim()==='发表时间') { el.click(); break; }
  await new Promise(r => setTimeout(r, 3000));
}
```

## 逐页全选（一批约10页，含停止判断）

```javascript
// 参数：startYear = 目标起始年(如2020)。含 startYear 的页面都选，整页 < startYear 才停
async function selectBatch(startYear) {
  window.alert = function(){};
  for (let i=0;i<5;i++){ document.querySelector('.checkcount a')?.click();
    await new Promise(r=>setTimeout(r,200)); }   // 清除旧选
  let total = 0;
  for (let i=0;i<10;i++) {
    if (i>0) {
      let cl=false;
      for (const a of document.querySelectorAll('a'))
        if (a.innerText?.trim()==='下一页') { a.click(); cl=true; break; }
      if (!cl) break;
      await new Promise(r => setTimeout(r, 1200));
    }
    // 看整页所有年份
    const allYrs = [...new Set((document.body.innerText.match(/(\d{4})-\d{2}-\d{2}/g)||[])
      .map(d=>d.substring(0,4)))];
    if (!allYrs.some(y => parseInt(y) >= startYear)) break;   // 整页都太老，停
    const cb = document.querySelector('#selectCheckAll1');
    if (cb) {
      cb.checked = false;
      await new Promise(r => setTimeout(r, 100));
      cb.dispatchEvent(new MouseEvent('click', {bubbles:true, cancelable:true}));
      await new Promise(r => setTimeout(r, 300));
      if (typeof cb.onclick === 'function') cb.onclick();   // 触发计数
      await new Promise(r => setTimeout(r, 500));
    }
    total = parseInt((document.body.innerText.match(/已选\s*(\d+)/)||[])[1])||0;
    if (total >= 450) break;   // 接近500上限，本批结束
  }
  return total;
}
```

## 导出（主页面菜单 → 导出页 → 自定义18列 → xls）

用 `browser_run_code_unsafe` 执行（需要真实鼠标点击）：

```javascript
async (page) => {
  // 主页面：导出菜单
  await page.locator('text=导出与分析').first().click({force:true});
  await page.waitForTimeout(1200);
  for (let i=0;i<await page.locator('a:has-text("导出文献")').count();i++) {
    if ((await page.locator('a:has-text("导出文献")').nth(i).innerText()).trim()==='导出文献') {
      const b = await page.locator('a:has-text("导出文献")').nth(i).boundingBox().catch(()=>null);
      if (b) { await page.mouse.move(b.x+b.width/2, b.y+b.height/2); break; }
    }
  }
  await page.waitForTimeout(800);
  // 精确点"自定义"（不是"查新（自定义引文格式）"）
  for (let i=0;i<await page.locator('a:text-is("自定义")').count();i++) {
    const b = await page.locator('a:text-is("自定义")').nth(i).boundingBox().catch(()=>null);
    if (b && b.width>0 && b.width<500) { await page.locator('a:text-is("自定义")').nth(i).click({force:true}); break; }
  }
  await page.waitForTimeout(4000);

  // 导出页
  const ep = page.context().pages().find(p => p.url().includes('export.html'));
  if (!ep) return 'NO EP - 可能验证码';
  await ep.bringToFront(); await ep.waitForTimeout(4000);
  await ep.locator('ul.formatlist li').first().waitFor({state:'visible', timeout:10000});
  // 导出页再选一次"自定义"
  for (const li of await ep.locator('ul.formatlist li').all()) {
    if ((await li.innerText()).trim()==='自定义') {
      await li.locator('a').click({force:true}); await ep.waitForTimeout(1500); break;
    }
  }
  // 全选字段 + xls（用 evaluate，因为"全选"可能不可见）
  await ep.evaluate(async () => {
    for (const a of document.querySelectorAll('a'))
      if (a.innerText?.trim()==='全选') { a.click(); await new Promise(r=>setTimeout(r,200)); a.click(); break; }
    await new Promise(r=>setTimeout(r,300));
    for (const a of document.querySelectorAll('a'))
      if (a.innerText?.trim()==='xls') { a.click(); break; }
    await new Promise(r=>setTimeout(r,3000));
  });
  await page.waitForTimeout(15000);   // 关键：等验证码处理
  await ep.close();
  return 'ok';
}
```

## 批次间导航（首页 + 精准翻页）

```javascript
// 每批导出后，回到首页再精准翻到下一批起始页
async function gotoPage(targetPage) {
  window.alert = function(){};
  for (const a of document.querySelectorAll('a'))
    if (a.innerText?.trim()==='首页') { a.click(); break; }
  await new Promise(r => setTimeout(r, 2000));
  for (let i=1;i<targetPage;i++) {
    let cl=false;
    for (const a of document.querySelectorAll('a'))
      if (a.innerText?.trim()==='下一页') { a.click(); cl=true; break; }
    if (!cl) break;
    await new Promise(r => setTimeout(r, 600));   // 每次翻页间隔，避免跳页
  }
}
```

## 合并去重（Python）

```python
import pandas as pd, glob

target = '目标刊名'   # 精确刊名
dfs = []
for f in glob.glob('下载目录/CNKI-*.xls'):
    df = pd.read_html(f, encoding='utf-8')[0]
    df.columns = df.iloc[0]
    df = df.iloc[1:]
    if len(df.columns) == 18:
        sub = df[df.iloc[:, 4] == target]   # 严格精确过滤，防混刊
        if len(sub) > 0:
            dfs.append(sub)

merged = pd.concat(dfs, ignore_index=True)
merged = merged.drop_duplicates(subset=merged.columns[1])   # 按Title去重
merged.iloc[:, 10] = pd.to_numeric(merged.iloc[:, 10], errors='coerce')
clean = merged[merged.iloc[:, 10] >= 2020].sort_values(merged.columns[7], ascending=False)
# 核验年份覆盖
yrs = clean.iloc[:, 10].dropna().astype(int)
print({int(k):int(v) for k,v in yrs.value_counts().sort_index().items()})
clean.to_excel('输出/{}.xlsx'.format(target), index=False)
```
