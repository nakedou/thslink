# ThsLink 使用说明 · HTTP 调用（v0.2.8）

## 2026-10-14 本地兼容测试包

本地最新远航兼容测试包在 `dist/testing/20261014-yuanhang-compat/`，支持已校验的 32 位 `xiadan.exe` 远航 `2026.05.06.0`、旧版 `2026.07.02.0` 普通账户及原 `2026.09.15.0` 构建。DLL 会检查实际构建和内部接口，其他同名程序或相近版本不自动放行。远航/七月旧版信用账户（type1）暂未支持，遇到该类型会明确拒绝账户读取、查询和交易；不能当作完整信用账户兼容包使用。

远航和七月旧版普通账户读取、持仓、当日委托、当日成交、资金查询和 DLL 安全卸载/重连已在真实进程验证。没有用真实买卖或撤单做验证。桌面版与纯接口版使用同一份桥接组件，请整体使用新包，先安全退出旧客户端并确认 DLL 卸载。交易端为管理员权限时，新客户端也需以管理员权限启动。正式 GitHub 下载包尚未更新为此兼容测试包。

桌面版 ThsLink 与 ThsLink-Cli-纯接口版共用以下 HTTP 接口。纯接口版启动与菜单配置见随包 `纯HTTP版说明.md`；桌面版从界面设置接口。

ThsLink 仅供个人非商业学习与研究使用，不允许用于商业行为；完整条款见 [项目许可](../LICENSE.md)。

ThsLink 支持本机、局域网及服务器远程 HTTP 调用、单笔与批量交易、多账户买卖撤单。

ThsLink 把桌面操作、HTTP 请求和 CLI 命令送到同花顺 `xiadan.exe` 内的桥接 DLL，通过进程内原生命令执行器提交并接收内部返回结果。持仓、当日委托、当日成交、资金查询使用独立的内部查询请求和对应响应结果表。

## 绿色便携版启动

解压完整便携包，保留 `thslink.exe`、`thslink_bridge.dll`、`thslink-load.exe` 在同一目录，双击 EXE 使用，无需安装或 CMD。请下载 Release 附件 `thslink-windows.zip`，不是 GitHub 的 Source code 压缩包。

Windows 10/11 x64；桥接组件用于 32 位 `xiadan.exe`。先登录同花顺交易端，再启动 ThsLink。更新时退出两者并整体替换文件；不要混用新旧 DLL。内部接口与交易端版本有关，不匹配时拒绝连接。

## 启动与接口设置

先启动同花顺并登录账户，再启动 ThsLink。首次使用时打开主界面的“接口”开关；开关状态会保存，以后启动自动恢复，手动关闭后下次保持关闭。

- 默认地址：`http://127.0.0.1:8421`。
- 设置中可以修改监听 IP、端口和 API 密钥。
- 局域网调用：监听 IP 设为 `0.0.0.0`，其他电脑使用运行 ThsLink 电脑的局域网 IP 和端口；Windows 防火墙须允许该端口。`0.0.0.0` 是监听配置，不是客户端请求地址。
- 配置密钥后，每次请求携带 `X-API-Key: 你的密钥`。
- 开关、地址、端口及密钥保存在 `%APPDATA%/thslink/ui-settings.json`；绿色便携版更新不覆盖配置。同一 Windows 用户移动软件目录后也继续使用已有配置。
- 关闭“手动交易”后，停止自动读票、股票跟随与交易热键，界面显示“读票暂停”；HTTP 请求、DLL 连接维护和后台回执继续工作。重新开启手动交易后恢复读票。

命令行 `thslink.exe ui --http` 启动界面和 HTTP；`thslink.exe serve` 仅运行 HTTP，使用默认本机地址。自定义监听配置请在桌面设置中使用。

## 连续调用：复用 HTTP 连接

服务端默认支持 HTTP Keep-Alive。POST、GET 都可以复用连接，接口地址和 JSON 参数不用改。本机、局域网和远程调用都适用。

| 什么时候 | Python 应该怎么做 |
|---|---|
| 你的 Python 程序启动 | 创建一次 `session = requests.Session()` |
| 每次下单或查询 | 使用同一个 `session.post(...)` 或 `session.get(...)` |
| 轮询回执或查询结果 | 继续使用这个 Session，不重新创建 |
| 你的 Python 程序退出 | 调用 `session.close()`；下面示例用 `atexit.register(session.close)` 在正常退出时清理 |

不要在每次调用的函数或循环里新建 Session，也不要每发一笔就关闭。下面各示例可单独运行；合并到同一个策略程序时，只保留一份 Session 初始化，后续请求共用它。示例按顺序调用；多线程程序建议每个工作线程长期保留自己的 Session。

仅添加 `Connection: keep-alive` 请求头不够：调用端销毁客户端后，连接就无法继续复用。使用 Session 时通常不需要手动添加这个请求头。示例默认完整读取响应；如果改成 `stream=True`，要读完响应体或关闭响应，连接才能归还连接池。

复用不代表连接永不断开：服务端重启、网络变化或中间设备超时仍可能断开连接。后续请求可建立新连接；交易 POST 超时或断线后，先核对原请求回执或当日委托，不自动重发。示例没有启用交易失败自动重试。

## 局域网与多账户

局域网调用时，同花顺和 ThsLink 留在服务端电脑，另一台电脑通过 HTTP 提交请求并查询回执。客户端不需要安装 ThsLink。监听地址设为 `0.0.0.0`，开启接口，修改监听设置后重启接口；请求使用服务端电脑的实际局域网 IP 和端口。先用 `GET /health` 检查连通性。

买入、卖出、撤单均支持多账户：

| 账户范围 | 请求字段 | 含义 |
|---|---|---|
| 当前账户 | 不填 account | 使用同花顺当前账户 |
| 指定账户 | `"account":"ACCOUNT_ID"` | 换成实际已登录账号，无需先手动切换当前账户 |
| 全部账户 | `"account":"all"` | 对当前交易端全部已登录账户执行指令 |

批量中每笔分别设置账户范围。`account=all` 只涵盖已登录账户，不会自动登录其他账户。本机、局域网和服务器远程请求使用同一套接口和后台回执链路。

## 服务器部署与远程调用

同花顺和 ThsLink 在同一台 Windows 服务器运行。远程调用可通过 VPN 地址，或自行配置带身份认证的 HTTPS 入口；ThsLink 本身提供 HTTP，没有内置 HTTPS 证书或 VPN。请求参数与本机调用相同，客户端继续复用 HTTP 连接。

也就是说：服务器负责运行同花顺、登录账户和执行交易；你自己的电脑或另一台服务器运行 Python 等调用程序，通过网络发送请求。不要求两台电脑在同一个局域网，只要调用端能访问服务器的接口地址即可。服务器须具备可运行同花顺交易端的 Windows 桌面环境，交易端和 ThsLink 均需保持运行。

| 部署方式 | 调用端填写的接口地址 | 需要配置什么 |
|---|---|---|
| 同一台电脑 | `http://127.0.0.1:8421` | 使用默认本机监听地址 |
| 同一局域网 | `http://服务器局域网IP:8421` | 监听可访问的网卡地址或 `0.0.0.0`，允许防火墙端口 |
| 跨网络，通过 VPN | `http://服务器VPN地址:8421` | 两端加入 VPN，监听 VPN 网卡地址或 `0.0.0.0`，允许对应网络访问 |
| 跨网络，通过 HTTPS 入口 | `https://你的接口域名` | 配置域名、证书、身份认证和反向代理，转发到 ThsLink 的 HTTP 端口 |

将调用代码中的 `BASE` 换成对应地址，继续携带 `X-API-Key`，并复用同一个 `requests.Session()`。POST / GET、JSON 参数、查询和回执用法都相同。公网不建议直接暴露明文 HTTP 交易端口，优先使用 VPN 或 HTTPS 入口。

## 接口一览

| 方法 | 路径 | 用途 |
|---|---|---|
| POST | `/order` | 提交单笔买入、卖出或撤单 |
| POST | `/orders/batch` | 一次提交 1～20 笔请求 |
| GET | `/orders/:id` | 用 request_id 查询单笔回执 |
| GET | `/batches/:id` | 用 batch_id 查询整批及逐笔状态 |
| POST | `/query` | JSON 查询：kind、account、wait_ms；与 GET 返回一致 |
| GET | `/query/holdings` | 主动查询持仓 |
| GET | `/query/orders` | 主动查询当日委托（不是单笔回执） |
| GET | `/query/deals` | 主动查询当日成交 |
| GET | `/query/funds` | 主动查询资金 |
| GET | `/queries/:id` | 读取 query_id 对应的本次查询结果 |
| GET | `/health` | DLL 连接状态 |
| GET | `/who` | 账户信息 |
| GET | `/last` | 最新回执，供诊断使用 |

## 主动查询持仓、当日委托、当日成交和资金

查询链路：调用同花顺内部查询工厂 → 在交易端 UI 线程提交请求 → 跟踪请求复制及通信序号 → 等待对应响应处理完成 → 复制本次结果表 → 返回 JSON。无需打开查询页面，不通过读取界面控件或旧缓存代替主动查询。

查询支持 POST /query（JSON 请求体）或 GET /query/{kind}（地址参数），二选一即可。两种方式都不会创建买卖撤单或交易流水。使用与交易接口相同的监听地址和 X-API-Key。

| 参数 | 是否必填 | 填写方式 |
|---|---|---|
| kind | POST JSON 必填 | holdings 持仓、orders 当日委托、deals 当日成交、funds 资金；GET 写在路径里 |
| account | 可选 | 默认 current：当前账户；实际已登录账户号：指定账户；all：全部已登录账户，无需切换当前账户 |
| wait_ms | 可选 | 默认 2000 毫秒，范围 0～15000；0 表示提交后立即返回查询编号 |

示例地址：`GET /query/holdings?account=current&wait_ms=2000`。

| 查询内容 | GET 路径（无 JSON 请求体） |
|---|---|
| 持仓 | `/query/holdings?account=current&wait_ms=2000` |
| 当日委托 | `/query/orders?account=current&wait_ms=2000` |
| 当日成交 | `/query/deals?account=current&wait_ms=2000` |
| 资金 | `/query/funds?account=current&wait_ms=2000` |
| 获取本次结果 | `/queries/QUERY_ID`，替换为返回的 query_id |

全部账户资金示例：`http://127.0.0.1:8421/query/funds?account=all&wait_ms=2000`。指定账户时把 current 换为实际已登录账号；局域网调用将地址中的 127.0.0.1 换为服务端电脑 IP。`/query/orders` 查询当日委托表，`/orders/:id` 获取某笔交易请求的回执，两者不同。

| 返回字段 | 含义 |
|---|---|
| query_id | 本次查询编号，与交易 request_id 分开；用 GET /queries/编号 继续读取 |
| source | native_query：来自本次内部查询响应 |
| status | pending 等待中；complete 全部账户完整完成；partial 部分数据；error 查询失败 |
| complete / success | 所有目标账户都完整完成时才为 true |
| accounts | 各账户结果，包含 account、name、status、error、sequence、elapsed_ms、tables |
| tables | 该账户收到的原始市场结果表；不同市场分开返回 |
| columns | 每列元数据；用 name 与 rows 字段对应，不一定包含中文 caption，不能假定所有列都有中文名称 |
| rows / row_count | 数据行与条数；普通字段值为字符串，保留交易端文本格式；重复颜色字段的规则见下文 |
| page_next | 后续页游标；非空表示仅收到部分页面，不是完整结果 |

部分交易端资金表会重复返回 `FCOLOR`、`BCOLOR`（界面颜色字段）。`columns` 保留原始列名与顺序；同名颜色列的值在 `rows` 中按出现顺序保存为字符串数组，例如 `"FCOLOR": ["原第9列值", "原第25列值"]`，不覆盖或丢弃任何一个值。没有重复时仍为字符串。业务字段重名仍报告错误，避免误读金额或数量。

空表与失败不同：完整响应中 row_count=0 表示没有记录；无对应表、响应错误、超时或结果超出容量均明确报告错误。不要仅看 HTTP 200；partial 的 complete=false，必须检查账户状态。

```python
import time
import atexit
import requests

# 程序启动时只创建一次；后续下单、查询和轮询共用这个 session。
# 不要放进每次调用的函数或循环里反复创建。
session = requests.Session()
atexit.register(session.close)  # 程序正常退出时关闭连接池

base = "http://127.0.0.1:8421"
headers = {}  # 设置密钥后填 {"X-API-Key": "你的密钥"}
response = session.get(base + "/query/holdings",
    params={"account": "current", "wait_ms": 2000}, headers=headers, timeout=30)
response.raise_for_status()
result = response.json()
while result["status"] == "pending":
    time.sleep(0.1)
    response = session.get(base + f"/queries/{result['query_id']}", headers=headers, timeout=30)
    response.raise_for_status()
    result = response.json()
if not result.get("complete"):
    raise RuntimeError(result)
print(result["accounts"])
# 其他类型：/query/orders、/query/deals、/query/funds
# 全账户：params 中 account 改为 "all"；指定账户改为实际已登录账户号。
```

HTTP 202 表示仍在查询，请读取原 query_id，不要重新请求 /query/holdings。一次只处理一个查询任务，all 在同一任务内覆盖全部已登录账户（最多 32 个）。请求处理超时为 15 秒；结果最多保留最近 8 次、每次 120 秒，DLL 重启后失效。每账户结果最多 4 MB、整次最多 8 MB，最多 10000 行、128 列；超限不返回伪装完整的截断数据。

内部接口与同花顺版本绑定，入口签名不匹配时明确拒绝。含分页游标的响应返回 partial 和已收到页面，目前不自动追取后续页。资金常用字段：xd_2112 余额、xd_2116 可用金额、xd_2191 总资产、xd_2973 总市值；持仓常用字段：xd_2102 股票代码、xd_2103 名称、xd_2117 数量、xd_2121 可用数量。另有 xd_2113 冻结金额、xd_2114 可取金额、xd_2125 市值、xd_3710 实际金额、xd_2172 币种。各字段独立保留，不能仅因数值相同就合并。界面只展示名称已确认的资金字段；HTTP 和复制 JSON 保留完整原始字段，包括内部编号、空值和原始数值文本，不新增盈亏字段或自行计算。

## 单笔请求

以下 JSON 发送到 `POST /order`：

```json
{"side":"buy","code":"000009","price":"mrxjg","amount":100,"notip":true}
```

| 参数 | 填写要求 | 含义与示例 |
|---|---|---|
| side | 必填 | buy 买入、sell 卖出、cancel 撤单 |
| code | 必填 | 六位股票代码字符串；全撤填 all |
| price | 买卖必填 | 价格关键字或指定价格字符串，例如 mrxjg 或 "7.00"；撤单不填 |
| amount | 买卖数量五选一 | 股数，正整数；1 手填 100 |
| m | 买卖数量五选一 | 金额，单位元 |
| cw | 买卖数量五选一 | 买入按可用资金比例；卖出按可卖数量比例 |
| zzc | 买卖数量五选一 | 本次交易金额占账户总资产的比例 |
| zcw | 买卖数量五选一 | 交易后这只股票还剩多少仓位（占账户总资产）；卖出填 0 表示清仓 |
| account | 可选 | 省略：当前账户；ACCOUNT_ID：指定已登录账户；all：全部已登录账户 |
| notip | 必填，必须为 true | 跳过同花顺确认提示；漏填或 false 返回 HTTP 400，本次未发送。批量每笔均须填写 |
| zdky | 可选，仅买入 | 资金不足时按最大可买数量提交 |

数量参数只选一种。cw、zzc、zcw 支持小数或分数；小数 0.1 是 10%，分数须写字符串 "1/10"。zcw 是交易后的目标仓位，不是这次交易的比例。

提供 `account` 时必须使用字符串，数字账号应保留前导零。数组、对象、JSON 数字、布尔值、null 或空字符串会返回 HTTP 400；批量中任一笔账户参数无效则整批不发送。使用当前账户时可直接省略该字段。

| 价格关键字 | 含义 |
|---|---|
| mrxjg / mcxjg | 买限价 / 卖限价 |
| mrj1～mrj5 | 买一～买五 |
| mcj1～mcj5 | 卖一～卖五 |
| zxjg | 最新价 |
| ztjg / dtjg | 涨停价 / 跌停价 |
| "7.00" | 自定义每股价格 7 元 |

撤单不填写价格和数量：

```json
{"side":"cancel","code":"000009","notip":true}
```

全撤当前账户：

```json
{"side":"cancel","code":"all","notip":true}
```

全撤全部已登录账户：

```json
{"side":"cancel","code":"all","account":"all","notip":true}
```

`code=all` 是全撤委托；`account=all` 是全部已登录账户，二者作用不同。指定账号时将 ACCOUNT_ID 替换为实际账号。买入、卖出、撤单均可选择当前、指定或全部已登录账户。

## 批量买入及其他批量操作

发送到 `POST /orders/batch`，将每笔单笔 JSON 放进 orders 数组：

```json
{
  "orders": [
    {"side":"buy","code":"000009","price":"mrxjg","amount":100,"notip":true},
    {"side":"buy","code":"600000","price":"mrxjg","cw":"1/10","notip":true}
  ]
}
```

每批 1～20 笔；每笔分别设置方向、股票、价格、数量和 account。省略账户时使用当前账户；指定账户要写在每笔中。批量提交不是原子事务：一笔失败不会撤回其他已提交的委托。各笔比例按交易端处理时的账户数据计算，不是固定同一资金快照。

返回 batch_id 和 orders，各笔 result 为提交结果。batch_id 现为不透明的 GUID 字符串，原样保存并用于 `GET /batches/批次编号`，不要转成整数或从本地序号推算；旧数字编号查询返回 404，不会命中新批次。每批生成独立编号，重启不再从 1 重新分配对外编号。批量合并回执按整批展示；不能仅凭返回数组顺序判断合同号属于哪笔订单。

提交和查询响应的 batch_size 是原批次笔数，complete 表示返回记录是否完整，不表示全部订单已完成。查询各条 batch_index 应覆盖 1～batch_size；记录不完整时不能宣称整批已结束。仍有待核对成员的批次会整体保留，包括其中已返回的成员；清空历史也遵守此规则。整批结束后，按最近完成时间作为一个整体参与历史清理。

```python
import atexit
import requests

# 程序启动时只创建一次；后续下单、查询和轮询共用这个 session。
# 不要放进每次调用的函数或循环里反复创建。
session = requests.Session()
atexit.register(session.close)  # 程序正常退出时关闭连接池

base = "http://127.0.0.1:8421"
headers = {}  # 配置密钥后填写 {"X-API-Key": "你的密钥"}
payload = {"orders": [
    {"side": "buy", "code": "000009", "price": "mrxjg", "amount": 100, "notip": True},
    {"side": "buy", "code": "600000", "price": "mrxjg", "cw": "1/10", "notip": True}
]}
response = session.post(base + "/orders/batch", json=payload, headers=headers, timeout=30)
response.raise_for_status()
result = response.json()
print(result)
batch_id = result.get("batch_id")
if batch_id is not None:
    print(session.get(base + f"/batches/{batch_id}", headers=headers, timeout=10).json())
# 等待中，继续查询原批次编号，不要重复提交整批。
```

软件“说明”的第二页“HTTP 请求生成/在线调试”支持单笔和批量：表单逐笔“加入批量”，或直接编辑 orders JSON，再复制 JSON / Python / cURL 代码。发送按钮会提交实际请求，下方自动查询回执。选择“数据查询”可填写参数或编辑自定义 JSON，生成持仓、当日委托、当日成交、资金的 JSON 请求体 / GET 地址 / Python / cURL，并在线发送只读查询，选择当前、指定或全部账户；支持 wait_ms，pending 时自动读取原 query_id 的结果。在线调试选择 JSON、Python 或 cURL 时发送 POST /query；选择 GET 地址时发送 GET 查询。标题和发送按钮会显示当前请求方式。查询结果显示完整原始 JSON，可选中或复制；查询不写入交易日志。

查询 JSON 请求体示例（可在“数据查询 → 自定义 JSON”中粘贴，也可由程序直接 POST 到 /query）：

```json
{
  "kind": "holdings",
  "account": "all",
  "wait_ms": 2000
}
```

`kind` 必填：`holdings` 持仓、`orders` 当日委托、`deals` 当日成交、`funds` 资金。`account` 选填，默认 `current`，也可填已登录账户号或 `all`；`wait_ms` 选填，默认 2000，范围 0～15000。以上 JSON 可直接提交到 `POST /query`，请求头为 `Content-Type: application/json`。也可选择等效的 `GET /query/holdings?account=all&wait_ms=2000`，两种方式不需要同时调用。

## 结果与日志

单笔返回 request_id、accepted、confirmed、status，以及 receipt 或 error。用 `GET /orders/原 request_id` 获取交易结果，不要拿 `/last` 代替。没有拿到 request_id 的不确定结果，需要核对当日委托，不能盲目重发。

| 字段或状态 | 含义 |
|---|---|
| accepted | 已取得交易端请求编号；不表示成交 |
| confirmed / success | 已获得可以确认提交结果的成功返回；不表示成交 |
| status=confirmed | 已确认委托提交结果，包括聚合撤单成功笔数或无可撤委托 |
| status=accepted | 已取得内部返回，但内容尚不足以确认成功 |
| status=uncertain | 结果不确定，按原编号继续核对，不自动重发 |
| status=rejected | 明确拒绝或失败，原因见 error |
| receipt | 完整返回文字；合同号和撤单成功笔数以这里实际返回为准 |

`GET /orders/:id` 的查询错误与原订单结果分开：记录缺失或过期返回 HTTP 404、`query_status=not_found`；暂时不可查返回 HTTP 503、`query_status=unavailable`；返回编号不匹配或格式无效返回 HTTP 502、`query_status=invalid_response`。这些响应均保留原 `request_id`、`status=uncertain`、`confirmed=false`，省略无法据此确定的 `accepted`，不会把原流水改为 Failed。404/503/502 都不能作为重新发单的依据；临时错误可按原编号稍后重查。

聚合撤单可能只有“成功笔数：3”，没有逐笔合同号；不会凭空生成合同号。`request_id`、`batch_id`、`query_id` 是三种不同编号，不可互换。HTTP 200/202 也不等于成交，须读取 JSON 内的状态。

批量查询各笔 stage：Posted 等待回执，Submitted 回执完成，Failed 请求失败，Timeout 结果暂未确定（可继续查询迟到回执）。HTTP 请求失败或等待超时，不代表同花顺一定没有收到，请先查原编号再决定下一步。

回执在后台处理。软件已结束历史最多保留本次运行最近完成的 200 条，清理时不拆分批次；未完成请求单独保留（最多 2000 条），其批次中已完成的成员也一起保留。清空历史不会清掉这些仍需处理的整批记录，不保存交易流水到磁盘，重启后清空。需要长期日志时，由调用程序保存请求参数、返回编号和查询结果。复制生成代码本身不发单。

## CLI 命令

```text
thslink.exe send buy 000009 mrxjg -cw 1/10 -notip
thslink.exe send sell 000009 mcxjg -cw 1/2 -account ACCOUNT_ID -notip
thslink.exe send cancel all -notip
thslink.exe send cancel all -account all -notip
thslink.exe result 请求编号
thslink.exe who
thslink.exe query holdings current
thslink.exe query orders all
thslink.exe query deals ACCOUNT_ID
thslink.exe query funds current
thslink.exe query-result 查询编号
```

CLI 与 HTTP 使用同一 DLL 通道；HTTP 字段 account、cw 等对应命令的 -account、-cw 等。长连接由当前驻留 ThsLink 进程持有；已运行桌面版或 HTTP 服务时，其他程序应通过 HTTP 调用，不要再同时启动另一个 ThsLink CLI 进程争用管道。

撤指定合同的命令示例：`thslink.exe send cancel -h ORDER_ID -notip`，把 ORDER_ID 换成实际委托合同号。

## 重连与更新

同花顺重启后，软件重新识别进程并建立连接。DLL 重连按停止后台线程、卸载、重新加载的顺序执行；有发送、排队、等待回执或内部查询请求时不允许完整重启。

GitHub 更新只在用户手动点击检查时执行，不会启动或定时自动检查。更新来源已内置，无需填写仓库地址。更新使用完整便携包，同时替换主程序、DLL 和加载器；安装前退出同花顺，个人配置保留。更新请确保网络可以访问 GitHub。更新器匿名访问 Release；仓库为私有时，软件内检查不可用，需要有权限的账号登录 GitHub 手动下载；仓库公开后可使用软件内检查。


## 长期运行与技术诊断

请求结果记录最多 2000 笔；容量不足时回收已结束或超时超过 5 分钟的记录，批量记录一起回收。超时仍表示结果未知，不自动重发。软件后台对旧请求的自动查询最多持续 5 分钟。

技术文件日志默认关闭。如需排查问题，可在便携版程序旁创建 `diagnostics` 文件夹，并在其中创建空文件 `enabled`；也支持在 `%LOCALAPPDATA%\ThsLink\diagnostics` 创建同名标记。便携目录标记优先，日志写到启用标记所在目录。删除标记即可关闭，约 5 秒内在下一次日志写入时生效；若两个目录都有标记，需要都删除。DLL 使用自身所在目录，不是同花顺 EXE 的目录。

`startup.log` 记录主程序入口、首次连接和界面更新，`loader.log` 记录 DLL 探测、加载和握手，`bridge.log` 记录 DLL 工作线程与管道，`timing.log` 在交易发生时记录交易计时，不交易时可能没有该文件。每条记录含时间戳、进程号和相对该组件首次记录的毫秒数；各文件最多 1 MB，各保留 3 份轮换备份。这些是技术诊断文件，不是持久交易日志。


## 请求容量与超时

- HTTP 同时允许最多 64 个待处理或执行中的调用（单笔、批量和查询共用容量）。满额立即拒绝；被拒请求不会延后自动发送。
- HTTP 请求等待发送的有效期为 5 秒。在取得发送锁后、实际写入交易命令前再次检查；过期或 HTTP 服务已经停止时返回“本次未发送”。服务重启后，上一轮尚未发送的请求也不会继续发送。
- 已经写入交易命令后，通信超时仅表示结果未知，禁止自动重发。调用方应核对回执或委托状态。
- 客户端每次管道通信有 10 秒期限。DLL 长连接正常空闲时持续等待；收到消息后的分段读取有 5 秒期限，响应写入与等待客户端确认共享 5 秒期限。客户端分段读取完整响应，超过 64 MB 明确报错，不伪装为空回执。
- 未知结果继续查询最多 5 分钟。超过该保留期后，记录仍表示未知，但不再永久阻止安全重连；重连不会重发旧单，也不会将其改成成功。

## 窗口与后台运行

标题栏“置顶”和浅色 / 暗黑主题会保存。点“−”最小化到任务栏；点“X”隐藏到系统托盘。单击托盘图标恢复，右键“退出应用”结束程序。后台继续处理 HTTP 请求和回执；关闭“手动交易”才会暂停自动读票和交易热键。股票区域可切换手动输入和恢复跟随，手动模式不会被自动读票覆盖，重开需重新输入代码。

## 兼容窗口与查询诊断

普通双击 `thslink.exe` 使用无边框窗口。虚拟机或远程桌面出现黑屏时，先退出 ThsLink，再双击同目录 `兼容运行.bat`；也可执行 `thslink.exe --compat-window`，使用系统标题栏窗口。

主动查询应同时检查 `status`、`complete` 和各账户的 `error`。HTTP 502 或“内部查询未完整结束（返回 8）”表示本次查询失败，不能解释为没有委托；该返回标志的兼容问题尚未确认解决。

需要排查时，在程序与 DLL 所在目录创建 `diagnostics/enabled` 空文件，重试查询后查看 `diagnostics/query.log`。结束诊断后删除 `enabled` 即可关闭记录。诊断目录和日志不随正式包分发。


## 连接身份与安全退出（本地生命周期测试版）

连接检查区分交易端未启动、多个目标、旧/不同构建 DLL、管道无响应，以及当前 xiadan.exe 内部结构不兼容。DLL 身份包含版本、构建、路径与宿主 PID/路径；交易发送前再次核对身份，交易端改变时拒绝发送。退出上次程序不等于 DLL 已卸载；同构建 DLL 可复用，不同构建不能盲目替换。

GUI 关闭到托盘保持运行；从托盘选择退出才执行安全检查。CLI 使用菜单 0/q 或 Ctrl+C。未完成委托、查询、账户操作、连接任务或未知结果会拦住退出，提示原因；空闲后停止新任务，安全卸载 DLL，确认模块消失再退出。保留 xiadan.exe，不撤单、不重发。真正的超时不会仅因过了五分钟就变成“已确认”。

人工核对不确定结果只能在券商记录核实后执行：GUI 退出提示窗口直接列出记录并提供核对按钮，CLI 菜单 5 选择列表序号并用 y 确认。它保留原回执和不确定状态，只记录人工核实，绝不把未知委托伪造成成功。原生调用/查询仍在运行时继续保护退出。

HTTP `/health` 继续返回 status 和 bridge_pid，并增加 state、message、bridge：区分 ready、busy、different_build、legacy_bridge、terminal_incompatible、unresponsive、not_connected 和 exiting。身份路径供受配置密钥保护的接口诊断使用，不包含密钥或账户号。


### 交易端版本显示与退出操作

设置→同花顺连接显示 xiadan.exe 文件版本、PID、路径、构建标记和映像大小，并提供“刷新版本”。显示不依赖 DLL 注入：旧版、其他构建或未适配版本仍可读取文件版本。手动路径模式也可在文件未运行时检查版本；没有版本资源或没有权限时明确显示读取失败原因。

版本信息读取与兼容判定分别进行。当前支持已校验的普通版 2026.07.02.0、2026.09.15.0 和远航版 2026.05.06.0，并检查构建指纹；旧版和远航版限普通账户，其他账户类型或构建不自动放行。HTTP /health 增加 terminals 与 terminal_scan_error；已有 status/state/message/bridge_pid 字段保留。

安全退出窗口不要求填写编号或密码。只有不确定结果时才列出待核对记录，用户选择并勾选已在券商记录确认，再“确认核对”；核对不改变原始回执为成功。安全退出无法完成时提供“强制退出 ThsLink”“重新尝试安全退出”和“继续运行”。强制退出只结束 ThsLink，不关闭交易端或强制卸载 DLL。


权限不足时设置页显示“ThsLink普通权限 / 交易端管理员”及具体处理；连接在加载DLL前拒绝。HTTP /health 的 terminals包含 current_elevated、terminal_elevated、bridge_access_error；无法握手且权限检查未通过时state为permission_denied。权限未知不会猜测为管理员不匹配，而是明确访问被拒绝。程序不会自动提升权限或关闭交易端。


### 异常状态详情与退出选择

主窗口底部 DLL 未连接、HTTP 关闭/启动/异常状态显示红色问号，点击后打开原因详情，包括交易端文件版本、产品版本、PID、路径、构建和权限，以及实际连接/HTTP启动错误。正常DLL已连接、HTTP运行中显示绿色普通文字，不响应点击。关闭的HTTP开关仅说明服务未开启，不等于程序故障。

选择退出后先尝试安全退出。权限不足、任务忙或 DLL 无法确认时，提示具体原因，并提供“强制退出 ThsLink”“重新尝试安全退出”和“继续运行”。退出检查仍在等待时，也可直接点击“强制退出 ThsLink”，无需再确认或先完成核对。强制退出只关闭 ThsLink 及其 HTTP 服务，不关闭交易端，不强制卸载 DLL；未确认退出记录保留，下次启动重新检查连接。

正常安全退出和强制退出都保留交易端。“与交易端一起退出”选项已移除。强制退出后桥接 DLL 可能仍驻留在交易端，再次启动时按身份检查结果重连。退出不会撤销已提交委托，也不会把未知回执改为成功；调用方应保存原请求编号，并在券商委托/成交记录中核对，禁止自动重发。


### 已知回执边界

内部调用持续不返回或同花顺持续无回执的问题尚未确认修复。30 秒超时只改变本地显示，不取消已发送委托；内部执行未结束时，新的独立请求仍可能被挡住。结果不确定时核对原编号与券商记录，不自动重发。在线调试遇到临时 GET 错误时可能停止自动跟踪，可手动刷新原请求；后台交易流水独立跟踪。
