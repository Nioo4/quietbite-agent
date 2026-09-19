# QuietBite Shortcut 配置

本文件只描述真实 iPhone 上的配置步骤。Windows 端先启动 `server.py`，并让
iPhone 与 Windows 处于可访问的同一网络；不要在 Windows 上伪造或导入
`.shortcut` 文件。

## 准备

1. 在 Windows 的 PowerShell 中设置 `BIGMODEL_API_KEY`、`AMAP_WEB_KEY`、
   `PHONE_AGENT_TOKEN`、`AGENT_BIND_HOST`、`AGENT_PORT` 和
   `JOB_DEADLINE_SECONDS`。密钥只放在进程环境变量中。
2. 启动服务：

   ```powershell
   $env:BIGMODEL_API_KEY = "真实密钥"
   $env:AMAP_WEB_KEY = "真实高德 Web 服务 Key"
   $env:PHONE_AGENT_TOKEN = "与快捷指令相同的现有 Token"
   $env:AGENT_BIND_HOST = "0.0.0.0"
   $env:AGENT_PORT = "8765"
   $env:JOB_DEADLINE_SECONDS = "60"
   python server.py
   ```

3. 用浏览器或本地工具确认 `GET http://<Windows-IP>:8765/health` 返回
   `status=ok`。正式演示前把 Windows 防火墙规则限定到可信的本地网络。

## 从零创建快捷指令

为避免多个网络动作分别手工输入 Token 而产生差异，在动作列表最前面添加
一个 **Text**，内容为单行 `Bearer <PHONE_AGENT_TOKEN>`（`Bearer` 大小写固定，
后面只有一个半角空格，不包含尖括号或引号），并保存为变量 `auth_header`。
下面所有请求的 `Authorization` 请求头值均插入这个 `auth_header` 魔法变量。
Token 部分必须与 Windows 进程中的 `PHONE_AGENT_TOKEN` 逐字符一致。

1. 打开 iPhone“快捷指令”，新建快捷指令并命名为 `QuietBite`，将它添加到
   主屏幕。
2. 添加 **Ask for Input**，类型选“文本”，提示语例如“今晚想吃什么？”，
   保存为变量 `instruction`。
3. 添加 **Get Current Location**，保存为变量 `current_location`。
4. 添加 **Text** 动作，把 `current_location` 变量放入文本中。该动作应产生
   多行地址文本，例如：

   ```text
   中国
   广东省
   深圳市 南山区
   示例路4387号
   ```

   上例道路和门牌是合成测试值；不要在仓库、截图或文档中写入真实门牌号。
   服务端会在内存中把它转换为 `广东省 深圳市 南山区 示例路附近`，真实
   门牌不会进入云端请求或日志。

   可选增强：紧接 `Get Current Location` 添加两次 **Get Details of Locations**，
   分别读取 `Latitude` 和 `Longitude`，保存为 `latitude`、`longitude`。二者必须
   同时存在，并在 POST JSON 中选择“数字”类型。服务端会先保留 3 位小数再调用
   高德 3 公里周边搜索，不写日志或 Notes。若不希望向高德提供约百米级位置，
   可完全省略这两个动作；现有脱敏地址文本搜索仍能运行。
   即使提供坐标，需求中明确写出“海岸城附近”等地点时仍优先搜索该地点。

5. 添加 **Current Date**，格式化为带时区的 ISO 8601 文本（例如
   `2026-09-18T17:30:00+08:00`），保存为 `client_now`。
6. 优先添加 **Generate UUID**，保存为 `request_id`。如果当前 iOS 只有
   **生成哈希值**，则使用兼容流程：添加 **随机数**（范围
   `100000000`–`999999999`），再添加 **Text**，依次插入 `client_now`、随机数
   和 `instruction`，最后用 **生成哈希值** 的 `SHA-256` 对该 Text 计算哈希，
   将哈希结果保存为 `request_id`。每次点击都应生成新的 `request_id`；重试
   同一请求时必须复用同一结果，以利用服务端幂等。
7. 添加 **Get Contents of URL**，配置为 `POST` 到
   `http://<Windows-IP>:8765/v1/jobs`。请求头增加：

   ```text
   Authorization: Bearer <PHONE_AGENT_TOKEN>
   Content-Type: application/json
   ```

   请求体选择 JSON，字段右侧必须插入对应的快捷指令魔法变量（不是把
   `request_id`、`instruction` 等变量名当成输入字面量）：

   ```json
   {
    "request_id": "插入 UUID 或 SHA-256 哈希结果的魔法变量",
     "instruction": "插入 Ask for Input 的魔法变量",
     "location_text": "插入上一步 Text 动作的魔法变量",
     "client_now": "插入 Current Date 格式化结果的魔法变量",
     "timezone": "Asia/Shanghai",
     "latitude": "可选：数字类型的纬度魔法变量",
     "longitude": "可选：数字类型的经度魔法变量"
   }
   ```

   未配置坐标时把 `latitude`、`longitude` 两行一起删除，不能只保留其中一个。

   保存响应为 `submit_response`，读取其中的 `status` 和 `job_id`。
8. 在轮询前用 **Number** 创建数值 `0`，并保存为变量 `delivered`，作为本次快捷指令的防重
   开关。`submit_response.status` 为 `accepted`、`received`、`parsing`、
   `searching` 或 `validating` 且存在 `job_id` 时都进入轮询，以兼容同一
   `request_id` 的网络重试。添加 **Repeat**，次数设为 35；在循环内添加 **Wait 2
   Seconds**，再用 **Get Contents of URL** 以 `GET` 请求
   `/v1/jobs/<job_id>`，同样带 Bearer Token，并读取 `status`。比较
   `status` 前，先把取得的词典值单独插入一个 **Text** 动作并保存为
   `poll_status_text`；部分 iOS 版本会把“获取词典值”的输出推断为任意类型，
   导致 **If** 只能选择“有任意值”。后续状态比较统一使用这个文本变量。
   每轮先判断 `delivered`，已经为 `1` 就跳过后续动作。若提交响应已经是 `ready`，
   直接进入下一步；若已经是 `completed`，立即静默停止，不能再次建 Note。
9. 当轮询状态为 `ready` 且 `delivered=0` 时，先添加一个 **Text** 动作，
   依次插入 `note.title` 魔法变量、换行、`note.body` 魔法变量。把这个
   Text 的结果交给 **Create Note**；常见动作界面是“用 [Text] 创建备忘录
   到 [Folder]”，标题取 Text 的首行，因此不要把 title/body 当成两个手写
   输入框。Folder 选择专用文件夹 `QuietBite`，若界面显示 **Show Compose
   Sheet** 则关闭它。不要用固定布尔值代替创建动作的返回值。
10. 对 **Create Note** 动作的真实输出使用 **Has Any Value**（或等价条件）
    计算布尔值 `note_created`：返回的新备忘录对象有任意值时为 `true`，
    否则为 `false`。
11. 添加 **Get Contents of URL**，以 `POST` 请求
    `/v1/jobs/<job_id>/complete`，发送
    `{"note_created": note_created, "completed_at": <当前 ISO 时间>}`。
    回调请求发送完成后才把 `delivered=1`，随后立即执行 **Stop This
    Shortcut**，保证 Repeat 不会再次 Create Note。只有服务端收到
    `note_created=true` 后才会输出 `[DONE]`。
12. `rejected` 或 `failed` 状态立即执行 **Stop This Shortcut**；所有分支
    最后静默结束快捷指令，不添加 **Show Result**、**Show Alert**、
    **Show Notification**、**Open App**、**Quick Look**，也不要打开 Notes
    编辑页面。`ready` 以外的状态应跳过 Create Note；`rejected` 或 `failed`
    直接结束并保留服务端状态供排查。35 次轮询后仍无终态也静默停止，且
    不发送完成回调；这种情况不是成功，Windows 不得输出 `[DONE]`。

### 完整 Repeat 动作树

以下变量必须在进入循环前已经存在：`job_id`、`delivered=0` 和
`PHONE_AGENT_TOKEN`。缩进表示动作必须放在对应的 **If** 或 **Repeat**
框内：

```text
Repeat 35 Times
  Wait 2 Seconds

  Text: http://<Windows-IP>:8765/v1/jobs/[job_id]
  Get Contents of URL
    Method: GET
    Authorization: Bearer <PHONE_AGENT_TOKEN>
  Set Variable: poll_response

  Get Dictionary Value: status from poll_response
  Text: [上一步词典值魔法变量]
  Set Variable: poll_status_text

  If poll_status_text is ready
    If delivered is 0
      Get Dictionary Value: note from poll_response
      Set Variable: note_dict
      Get Dictionary Value: title from note_dict
      Set Variable: note_title
      Get Dictionary Value: body from note_dict
      Set Variable: note_body

      Text:
        [note_title]
        [note_body]
      Create Note in QuietBite (Show Compose Sheet: Off)
      Set Variable: created_note

      Current Date
      Format Date: ISO 8601, include time
      Set Variable: completed_at

      If created_note has any value
        Text: http://<Windows-IP>:8765/v1/jobs/[job_id]/complete
        Get Contents of URL
          Method: POST
          Authorization: Bearer <PHONE_AGENT_TOKEN>
          Content-Type: application/json
          JSON note_created: Boolean true
          JSON completed_at: [completed_at]
        Number: 1
        Set Variable: delivered
        Stop This Shortcut
      Otherwise
        Text: http://<Windows-IP>:8765/v1/jobs/[job_id]/complete
        Get Contents of URL
          Method: POST
          Authorization: Bearer <PHONE_AGENT_TOKEN>
          Content-Type: application/json
          JSON note_created: Boolean false
          JSON completed_at: [completed_at]
        Stop This Shortcut
      End If
    End If
  End If

  If poll_status_text is rejected
    Stop This Shortcut
  End If
  If poll_status_text is failed
    Stop This Shortcut
  End If
  If poll_status_text is completed
    Stop This Shortcut
  End If
End Repeat

Stop This Shortcut
```

`received`、`parsing`、`searching` 和 `validating` 不需要单独分支；遇到这些
状态时本轮自然结束，下一轮等待 2 秒后继续查询。三个状态判断中的自动
**Otherwise** 可以留空。最后一个 **Stop This Shortcut** 处理 35 次轮询后
仍未到终态的超时情形，不能在这里发送完成回调。

## 首次运行授权

正式部署前运行一次快捷指令并选择 **Always Allow**：当前位置、本地网络、
备忘录和网络请求。确认日志中没有精确地址或坐标；不要把真实 Token、BigModel
Key 或高德 Key 写进截图、演示视频、备忘录或仓库。

## 导出最终 Shortcut

建议保留两个副本：

- `QuietBite-Demo`：本机演示版，保留真实 Windows IP 和 Token，不共享、不上传。
- `QuietBite-Release`：交付版，把 Token 改为 `Bearer CHANGE_ME`，把 Windows IP
  改为 `WINDOWS_IP`，供评审导入后自行配置。

导出前删除 **Quick Look**、**Show Result**、**Show Alert**、**Show Notification**
以及首次 POST 后为了调试临时添加的 **Stop This Shortcut**。不要删除 Repeat 内
`ready`、`rejected`、`failed`、`completed` 分支中的 Stop，也不要删除 Repeat 后的
超时 Stop；这些属于正式控制流。

在 iPhone 上导出：

1. 打开“快捷指令”App，进入 `QuietBite-Release` 编辑页面。
2. 轻点顶部名称旁的向下箭头，选择 **导出文件（Export File）**。
3. 如果使用共享按钮路径，则依次选择 **选项 → 文件 → 任何人 → 完成**。
4. 选择 **存储到“文件”**，保存为 `QuietBite-Release.shortcut`。
5. 从“文件”App 再导入一次副本，确认能够添加，并按本指南重新填写 IP 和 Token。

Apple 官方共享说明：<https://support.apple.com/en-lb/guide/shortcuts/apdf01f8c054/ios>

## 排查

- `401`：检查 Bearer Token 是否与 `PHONE_AGENT_TOKEN` 完全一致。
- `LOCATION_UNUSABLE`：确认 Get Current Location 的 Text 结果包含城市。
- `LIVE_QUEUE_UNAVAILABLE`：需求要求“必须不用排队”，V0.1 不会编造结果。
- `NO_VERIFIABLE_CANDIDATES`：没有具体门店、目标区域、带来源地址或合法 URL 时接受 0 家；
  营业时间和人均缺失本身不会再触发该错误，备忘录会省略对应字段。
- 快捷指令没有弹窗不代表成功；以 `GET /v1/jobs/<job_id>` 和成功回调为准。
