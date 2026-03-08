---
name: tcaptcha-solver
description: 腾讯防水墙(TCaptcha)滑块验证码逆向与登录集成。cap_union_prehandle、cap_union_new_verify、TDC.js 行为数据、YOLO 缺口检测、ticket/randstr 获取。适用于星巴克中国等使用腾讯防水墙的站点。
---

# 腾讯防水墙 (TCaptcha) 滑块

## 流程概览

```
prehandle (GET JSONP) → sess, pow_cfg, tdc_path, bg_elem_cfg, fg_elem_list
下载背景图 → YOLO 检测缺口 x1
slide_distance = gap_x1 - fg_elem_list[id=1].init_pos[0]
solve_pow(prefix, md5) → pow_answer, pow_calc_time
generate_slide_data(slide_x) → track, events
get_tdc_data(tdc_path, mouse_events) → collect, eks, tlg
encode_drag_ans(x=gap_x1, y=拼图 init_pos[1]) → ans
verify (POST application/x-www-form-urlencoded) → ticket, randstr
```

## 关键实现

### ans 参数

- 格式：`[{"elem_id":0,"type":"DynAnswerType_POS","data":"X,Y"}]`
- **X**：缺口左缘 x（YOLO 的 x1），**Y**：拼图块的 init_pos[1]（不是缺口 y），因拼图仅水平移动。
- 坐标均为**整图空间**（如 672px 宽），非缩放后的显示坐标。

### verify 请求

- 必须 POST，Content-Type: application/x-www-form-urlencoded（非 JSON）。
- 字段：collect（TDC.getData(true)）、tlg（len(collect)）、eks（TDC.getInfo().info）、sess、ans、pow_answer、pow_calc_time。
- 请求头需 Origin/Referer: https://turing.captcha.gtimg.com/

### 服务端 errorCode

- 返回为**字符串**（如 "0"），比较前需 int() 转换。

### TDC.js

- 需类浏览器环境。Node 下用 vm + 虚拟 document/window/navigator/screen，加载 TDC.js 后 setData(mouseEvents)，取 getData(true) 与 getInfo().info。服务端可能返回 gzip，需解压。

### PoW

- 求 counter 使 md5(prefix + str(counter)) == target；pow_answer 提交 prefix + str(counter) 完整串。

### 时序

- verify 前加 2–4 秒随机延迟；prehandle 到 verify 总时长过短易被拒。

## YOLO 缺口

- 用训练好的 best.pt 对背景图 predict，取 conf 最高框的 x1 作为缺口左缘。

## 登录集成（如星巴克中国）

- 拿到 ticket、randstr 后，按站点登录接口传 captcha: { ticket, randstr }；星巴克中国 App ID 2045345034，登录为 POST auth.starbucks.com.cn/web/login，Basic 鉴权 + JSON body。

## 常见坑

- verify 用 form 编码不是 JSON；ans 的 y 用拼图 init_pos[1]；errorCode 是字符串；TDC 需解 gzip；坐标按整图 672 宽；开发时可 httpx verify=False。
