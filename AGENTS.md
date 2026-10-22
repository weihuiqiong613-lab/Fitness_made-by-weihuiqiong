# AGENTS.md · 健身打卡助手

> 给接手的 AI 或其他开发者：
> 先读 `README.md` 了解项目，再读 `PRD.md` 确认第一版边界，最后按本文件动手。
> 本文件只写「怎么安全地改、怎么检查」，不重复需求背景。

## 1. 先知道的项目形态

- 项目是**一个文件**：`index.html`，内含 HTML、CSS、原生 JavaScript。
- 没有构建、没有依赖、没有后端、没有网络请求；双击即可运行。
- 数据只存在浏览器 `localStorage`，key 为 `fitness_tracker_v5`。
- 所有界面文案为简体中文；代码保持 ES5 风格（`var`、`function`、字符串拼接），不要引入构建工具或框架。
- 主要验证环境：桌面 Chrome / Edge，手机宽度 390px。

## 2. 代码地图

`index.html` 用注释分区，按标题搜索即可定位（行号会随改动变化）：

- `<style>` 内：
  - `五套皮肤变量`：5 套 `data-skin` 的日/夜 CSS 变量；新增颜色优先走变量。
  - `登录页`、`顶部`、`总览`、`卡片`、`睡眠记录`、`个人档案 / 身体数据台账`、`图表 / 导出`、`小屏适配`。
- 第一个 `<script>`（基础设施）：
  - `全局存储`：`loadStore / persistStore / toast / todayKey`
  - `皮肤 / 夜间`、`登录视图切换`、`验证码（演示）`、`忘记密码`
  - `默认数据`：`defaultData()`
  - `账号密码登录`、`短信登录`、`注册`、`进入主应用`
- 第二个 `<script>`（业务）：
  - `按人群类型分离的数据`：`blankSub / getSub`（**数据兼容的关键**）
  - `运动 MET 库`、`食物热量库`、`基础代谢`、`状态`
  - `渲染：总览 / 饮水 / 运动 / 饮食 / 近7天`
  - `类型切换`、`饮水操作`、`运动表单`、`饮食：专业 / 普通`、`列表事件`、`打卡`、`奖励`
  - `通用输入弹窗`
  - `睡眠记录`：`sleepStats / paintSleepPreview / renderSleep / loadSleepForm / sleepPraise`
  - `个人档案 / BMI / 身体数据台账`：`openProfile / bmiCat / renderLedger / renderWeightTrend`
  - `图表（SVG 手绘）`：`svgBarChart / svgLineChart / buildBarItems / renderCharts`
  - `打卡记录导出（txt）`：`buildExportText / downloadText`
  - `近三个月历史`、`初始化`、`启动`

## 3. 数据模型（改数据前必读）

```js
// localStorage['fitness_tracker_v5']
STORE = {
  users: {
    '<phone>': {
      phone, password,
      data: {
        days: {
          'YYYY-MM-DD': {
            pro:    SUB,
            normal: SUB
          }
        },
        personType: 'pro' | 'normal',
        profile: { weight, height, age, gender },
        ledger: [ { id, date, ts, weight, waist } ]
      }
    }
  },
  current: '<phone>' | null,
  skin: 'green' | 'pink' | 'mono' | 'blue' | 'cyber',
  dark: false
}

SUB = {
  water: 0,
  sports: [ { id, name, minutes, met, gross, net, done } ],
  foods:  [ { id, name, grams | portions, kcal, protein, matched } ],
  checkedIn: false,
  sleep: null | { bed, wake, awake, dur, cycles, quality }
}
```

规则：

- **不要随意改 `STORE_KEY`**。必须改时，要写迁移逻辑并在 README / PRD 说明。
- 新增字段要向后兼容：在 `blankSub()`、`defaultData()` 和 `getSub()` 里补默认值，不能让旧数据缺字段导致报错。
- 不要把两种人群的数据合并；`cur` 始终指向当前 `personType` 的 `SUB`。
- 不要清空用户数据来「修复」问题；先兼容旧数据。
- 所有示例数据必须是合成的，不能提交真实手机号、体重、腰围、病史。

## 4. 可以改什么 / 不可以做什么

可以：

- 改 UI 文案、布局、颜色、动效和无障碍属性。
- 在现有模块内修 bug、补边界处理、加不破坏数据兼容的小功能。
- 增加帮助说明、演示数据说明、测试清单。
- 在不引入外部依赖的前提下优化 SVG 图表和导出逻辑。

不可以：

- 不引入框架、构建工具、CDN 脚本、远程字体或任何网络请求；项目必须离线可用。
- 不接入真实账号、支付、医疗诊断或第三方埋点/统计。
- 不删除登录和两类人群的数据隔离，除非 PRD 明确改版。
- 不把 `localStorage` 换成需要后端的方案而不更新 README 的启动步骤。
- 不提交 `node_modules`、浏览器用户目录、测试截图以外的临时文件、真实个人信息。
- 不使用「治疗、诊断、保证减脂」等医疗或绝对化表述；BMI / 睡眠 / 消耗都要保留「估算、参考」措辞。

## 5. 怎么检查（每次改完必做）

1. **静态检查**：用 Chrome / Edge 打开 `index.html`，Console 无红色报错；Network 面板没有外部请求。
2. **主流程**：
   - 注册 → 退出 → 密码登录 / 短信演示登录 / 重置密码。
   - 切换「专业 / 普通」人群，确认饮食表单和每日数据不串。
   - 饮水 +250 / +500 / 自定义 / 撤销；运动添加、勾选完成、删除；饮食专业按克、普通按份。
   - 睡眠：23:00–07:00（良好，出现表扬动画）；01:30–07:00（不足，出现「睡眠偏少，高强度运动容易头晕乏力」）；跨午夜、清醒时长超过在床时间（应有错误提示）。
   - 点击手机号 → 计算 BMI → 看评价和建议；录入体重/腰围 → 曲线更新 → 删除一条记录。
   - 历史记录（近 7 天 / 近 3 个月）；运动数据图表（本周/本月切换）；导出 txt 能下载且包含睡眠、台账。
   - 刷新页面后数据仍在；清理 `localStorage` 后能重新开始。
3. **响应式与主题**：900px 桌面、390px 手机宽度无横向滚动；5 套皮肤 + 夜间模式切换正常，夜间模式下「粉色 / 黑白 / 绿色 / 蓝色 / 赛博」文字清晰。
4. **数据清理**：演示后执行 `localStorage.removeItem('fitness_tracker_v5')`，避免把演示数据带进截图或提交。

## 6. 提交前 Definition of Done

- [ ] `README.md`、`PRD.md`、`AGENTS.md` 在仓库根目录，文件名和大小写正确。
- [ ] README 的体验链接或录屏链接已替换为真实地址（二选一）。
- [ ] `index.html` 可直接打开运行，无 Console 报错，无外部网络请求。
- [ ] 桌面和 390px 手机宽度均通过主流程自测。
- [ ] 截图/录屏中的手机号、体重、腰围均为合成数据。
- [ ] 没有提交浏览器配置目录、临时测试文件、真实个人数据。
- [ ] PRD 的「做什么 / 不做什么 / 完成标准」与当前实现一致。
- [ ] README 末尾保留约 200 字协同复盘（或答辩时口头说明）。

## 7. 已知边界

- 账号和密码是前端演示级，不能用于生产。
- 数据只在当前浏览器，换设备/清缓存会丢失；txt 导出不可回导。
- BMI、睡眠周期、MET 消耗均为粗略估算，不是医学结论。
- 图表只支持本周/本月，按当前人群类型统计。
- 没有自动化测试；改动后请按第 5 节手动回归。
