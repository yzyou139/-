# 某讯滑块自动化开发接手说明

这份文档面向“第一次接触这个项目的人”。

目标是让接手者在不了解历史背景的情况下，也能快速回答下面几个问题：

1. 这个项目是做什么的
2. 代码从哪里开始运行
3. 每个核心文件负责什么
4. 登录和滑块分别是怎么自动化的
5. 出问题时应该看哪里、改哪里

---

## 1. 项目目标

这个目录是一套**可独立运行的腾讯滑块浏览器自动化副本**。

它做的事情是：

1. 打开 `https://www.eeo.cn/cn/login`
2. 自动填写账号密码
3. 自动勾选“记住登录状态”
4. 自动勾选“我已阅读并同意 …”
5. 自动点击“登录”
6. 等待腾讯滑块验证码出现
7. 自动抓取验证码图片
8. 自动识别缺口位置
9. 自动拖动滑块
10. 抓取最终 `verify` 请求和响应

这份目录是从原项目中**复制出来的可运行版本**，不依赖原来的 `split_img/` 或 `demo/` 路径。

---

## 2. 一分钟上手

### 运行主流程

```powershell
python D:\工作区文件夹\滑块\腾讯滑块自动化\tdc_solver.py
```

### 运行检测验证

```powershell
python D:\工作区文件夹\滑块\腾讯滑块自动化\capture_verify.py
```

### 当前目录结构

- `tdc_solver.py`
  主入口。负责登录、监听验证码、自动拖动、抓请求结果。

- `detection_core.py`
  缺口检测核心。输入背景图和拼图图，输出缺口位置。

- `capture_verify.py`
  验证脚本。适合在人工成功一次后，对比“检测结果”和“真实提交的 ans”。

- `capture/`
  保存抓到的 `prehandle`、`verify_req.txt`、`verify_resp.json`、`captcha_dom.json`。

- `downloaded_images/`
  保存运行过程中抓到的验证码图片。

- `detection_results/`
  保存可视化结果图。

- `chrome_profile/`
  持久化浏览器数据目录。保存 cookie、tkid、站点状态等。

---

## 3. 整体执行流程

主流程都在 `tdc_solver.py`。

### 第一步：打开登录页

入口函数是：

- `main()`

它会：

1. 初始化目录
2. 启动带持久化 profile 的 Chrome
3. 注册网络响应监听 `on_response`
4. 打开登录页
5. 调用 `auto_login(page)` 执行前置登录动作
6. 进入等待循环，直到抓到滑块资源并自动拖动

### 第二步：自动登录

自动登录逻辑在：

- `auto_login(page)`

这部分负责：

1. 切换到“密码登录”
2. 填账号
3. 填密码
4. 勾选“记住登录状态”
5. 勾选“我已阅读并同意 …”
6. 点击登录按钮

当前这版代码里，测试账号和密码是写在脚本里的：

```python
TEST_USERNAME = "12345678901"
TEST_PASSWORD = "12345678"
```

如果要正式使用，建议后续改成从 `config.json` 或环境变量读取。

### 第三步：监听验证码网络请求

网络监听在：

- `on_response(resp)`

它主要拦截三类接口：

1. `cap_union_prehandle`
   作用：拿验证码配置，包括 `sess`、`pow_cfg`、`fg_elem_list`、`bg_elem_cfg`

2. `cap_union_new_getcapbysig`
   作用：拿背景图和精灵图

3. `cap_union_new_verify`
   作用：拿最终验证请求与响应

抓到的数据会落盘到：

- `capture/prehandle.json`
- `capture/verify_req.txt`
- `capture/verify_resp.json`
- `downloaded_images/live_bg.png`
- `downloaded_images/live_sprite.png`

### 第四步：识别缺口

滑块识别和拖动逻辑在：

- `solve(page)`

`solve(page)` 会：

1. 从 `prehandle` 里提取拼图块坐标配置
2. 调用 `detect_gap_position(...)`
3. 计算目标 `final_x`
4. 在页面里定位滑块把手和背景元素
5. 计算显示层拖动距离
6. 调用 `drive_slider(...)` 执行拖动

缺口检测函数在：

- `detection_core.py -> detect_gap_position(...)`

---

## 4. 缺口检测算法怎么工作的

核心函数：

- `detect_gap_position(bg_path, sprite_path, piece, bg_size2d, y_band=12, top_k=5)`

### 输入

- `bg_path`
  背景图路径

- `sprite_path`
  精灵图路径

- `piece`
  拼图块配置，来自 `prehandle.json` 中的 `fg_elem_list`

- `bg_size2d`
  设计坐标系尺寸，一般是 `[672, 480]`

### 输出

返回 `DetectionResult`，关键字段包括：

- `final_x`
  最终缺口 x 坐标

- `match_y`
  匹配 y 坐标

- `confidence`
  综合置信度

- `hole_score`
  深色洞位评分

- `edge_score`
  边缘评分

### 当前算法思路

这版不是单纯做边缘模板匹配，而是：

1. 从精灵图里裁出真实拼图块
2. 用 alpha 透明通道生成拼图块有效区域
3. 计算背景图在每个候选位置下：
   - 拼图块内部是否明显更暗
   - 拼图块外圈是否明显更亮
4. 把“深色洞位特征”作为主评分
5. 用边缘匹配作为辅助评分
6. 在 `init_y ± y_band` 的限制带内取最佳候选
7. 对 x 做亚像素修正

为什么要这么做：

- 腾讯滑块很多样本不是“纯缺口边缘”，而是一个明显深色占位洞位
- 只靠边缘相关容易被背景纹理误导
- 当前版本已经验证过，“深色洞位优先 + 边缘辅助”更稳

---

## 5. 拖动逻辑怎么工作的

拖动逻辑有两层：

### 1. 计算拖动距离

在 `solve(page)` 中：

```python
distance_px = (fx - ix) * scale * DRAG_RATIO
```

含义：

- `fx`
  检测出的目标 x

- `ix`
  拼图块初始 x

- `scale`
  页面显示宽度 / 设计宽度

- `DRAG_RATIO`
  拖动补偿比例，默认 `1.0`

### 2. 生成人类化轨迹

函数：

- `human_track(distance, y_base)`

特点：

- 先加速后减速
- 带轻微抖动
- 有少量过冲和回拉
- 释放前停顿一下

真正执行鼠标拖动的是：

- `drive_slider(page, hx, hy, distance_px)`

---

## 6. 当前最重要的可调参数

### `TEST_USERNAME`

位置：

- `tdc_solver.py`

作用：

- 自动登录使用的账号

### `TEST_PASSWORD`

位置：

- `tdc_solver.py`

作用：

- 自动登录使用的密码

### `DRAG_RATIO`

位置：

- `tdc_solver.py`

作用：

- 解决“识别位置正确但实际拖动略短/略长”的问题

默认：

```python
DRAG_RATIO = 1.0
```

如果验证码总是差一点点，可以从这里微调，比如：

- `0.98`
- `1.02`

### `WAIT_SECONDS`

位置：

- `tdc_solver.py`

作用：

- 等待验证码出现的最长秒数

---

## 7. 登录页元素现在是怎么定位的

登录页前置自动化当前依赖这些已确认结构：

### 账号框

- `#accountInput`
- placeholder: `请输入手机号/邮箱`

### 密码框

- placeholder: `请输入6-20位密码`
- 兜底：`input[type='password']`

### 勾选框

当前页面用的是：

```html
<label class="el-checkbox">
  <span class="el-checkbox__input">
    <span class="el-checkbox__inner"></span>
    <input type="checkbox" class="el-checkbox__original">
  </span>
</label>
```

当前处理方式是：

- 登录区域第 1 个 `label.el-checkbox` 视为“记住登录状态”
- 第 2 个 `label.el-checkbox` 视为“同意协议”
- 点击的是真实可点节点 `span.el-checkbox__inner`

### 登录按钮

当前页面登录按钮结构是：

```html
<div class="submit-btn">登录</div>
```

所以当前代码优先点：

- `div.submit-btn`

---

## 8. 常见问题怎么排查

### 问题 1：账密填进去了，但勾选框没点上

先看终端里有没有这些日志：

- `[login] 已勾选: 记住登录状态`
- `[login] 已勾选: 同意协议`
- `[login] 已点击勾选框: ...`

如果没有，优先检查：

1. `label.el-checkbox` 的顺序有没有变化
2. 页面是不是改成了新的 checkbox 结构
3. 登录框是不是在 iframe 或弹层里

### 问题 2：登录按钮没点上

先看登录按钮 DOM 是否还是：

```html
<div class="submit-btn">登录</div>
```

如果改了，就调整 `auto_login(page)` 里登录按钮定位逻辑。

### 问题 3：滑块没出现

排查顺序：

1. 登录前置动作是否真的成功触发
2. 页面是否跳转或刷新
3. `on_response` 是否抓到了 `cap_union_prehandle`
4. 是否抓到了背景图和精灵图

### 问题 4：检测位置明显错误

优先看这些文件：

- `downloaded_images/live_bg.png`
- `downloaded_images/live_sprite.png`
- `detection_results/`
- `capture/prehandle.json`

如果要验证检测器是否偏移，可运行：

```powershell
python D:\工作区文件夹\滑块\腾讯滑块自动化\capture_verify.py
```

### 问题 5：识别对了，但 verify 失败

通常看这几个方向：

1. `DRAG_RATIO` 需要微调
2. 拖动轨迹太生硬
3. 滑块把手中心点拿得不准
4. 页面缩放比例有变化

---

## 9. 修改代码时，优先改哪里

### 只改登录页自动化

改：

- `tdc_solver.py -> auto_login(page)`

### 只改滑块检测

改：

- `detection_core.py -> detect_gap_position(...)`

### 只改拖动行为

改：

- `tdc_solver.py -> human_track(...)`
- `tdc_solver.py -> drive_slider(...)`
- `tdc_solver.py -> DRAG_RATIO`

### 只改抓包 / 日志

改：

- `tdc_solver.py -> on_response(resp)`
- `tdc_solver.py -> report()`

---

## 10. 建议的下一步优化

当前版本已经可以用，但从工程角度还可以继续优化：

### 1. 不要把账号密码写死在代码里

推荐改成：

- `config.json`
  或
- 环境变量

### 2. 把登录页选择器配置化

比如单独做一个：

- `selectors.json`

把账号框、密码框、checkbox、登录按钮的候选选择器集中管理。

### 3. 给每次运行生成时间戳目录

现在抓图和结果图会覆盖，后续可以改成：

- `capture/20260826_091500/...`
- `downloaded_images/20260826_091500/...`

### 4. 把运行参数集中起来

比如统一到：

- `settings.py`

包括：

- 登录 URL
- 等待秒数
- 拖动比例
- 测试账号密码来源

---

## 11. 结论

如果你是第一次接手这个目录，记住一句话就够了：

- **主入口看 `tdc_solver.py`**
- **识别算法看 `detection_core.py`**
- **验证检测误差看 `capture_verify.py`**

如果登录失败，就看 `auto_login(page)`。

如果滑块识别失败，就看 `detect_gap_position(...)`。

如果滑块识别对了但验证失败，就看 `DRAG_RATIO` 和拖动轨迹。
