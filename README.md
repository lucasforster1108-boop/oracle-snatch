# oracle-snatch

一个小的 Python 循环脚本：不停用甲骨文云的官方接口申请免费 ARM 实例（VM.Standard.A1.Flex），直到哪次赶上放货、接口点头为止。

## 为什么写这个

Always Free 的 A1.Flex 免费额度（总共 4 核 / 24 GB）是目前最划算的免费服务器，没有之一。问题是热门区域在控制台里基本永远显示「Out of host capacity」（没货）。这个货不是永久没了，而是会不定期放出来一小波，没人通知。对着控制台刷一周刷新纯属浪费生命，不如让脚本替你去问。

2026 年 10 月我用一个粗糙版本在东京区域跑了好几天。日志翻来覆去就是「没货」，中间还因为我把间隔设得太密、吃过几次 429 限流。最后抢到了那台 4 核 24 GB 的机器，我现在自托管的大部分服务都跑在它上面，到今天账单还是 0。这个仓库就是那个循环的整理版，配置都抽了出来。

## 它做什么

- 只用甲骨文官方 API（Python 的 `oci` SDK），不搞控制台自动化、不模拟浏览器。
- 轮询你所在区域的每一个可用域（AD），每轮每个 AD 尝试一次。东京只有一个 AD，多 AD 的区域中签率会好一些。
- 「没货」就当「还没到时候」：歇一会儿，下一个 AD，下一轮。
- 遇到 429 限流就长退避（默认 300 秒）。硬顶着 429 往前冲，只会让限流赖着不走。
- 遇到配额错误（LimitExceeded）直接停，并把原因写清楚：这说明你的免费额度已经用完了，再重试一万次也没用。
- 抢到后等实例变成 RUNNING，拿到公网 IP，写进 `snatched.json`，可选发一条 Telegram 通知，然后退出。
- `--dry-run` 只验证配置：列出可用域、确认会用哪个镜像，不真开机器。第一次先跑这个。

它不做的事：不帮你注册账号、不建 VCN、不生成 API 密钥，更变不出不存在的货。它只是替你不知疲倦地问。

## 安装与使用

需要一个甲骨文云账号，以及 OCI 的配置文件（`~/.oci/config`，里面配好 API 密钥）。用过 OCI CLI 或 SDK 的人都有这个文件。

还需要一个子网 OCID：控制台里 Networking -> Virtual Cloud Networks -> 你的 VCN -> Subnets。想要公网 IP 就选公有子网（你肯定想要）。

```bash
git clone https://github.com/lucasforster1108-boop/oracle-snatch.git
cd oracle-snatch
pip install -r requirements.txt

cp config.example.env config.env
# 改 config.env：必填的只有 SUBNET_ID 和 SSH_PUBLIC_KEY_FILE

python3 snatch.py --dry-run
python3 snatch.py
```

## 长期挂着跑

这东西就是设计成无人值守跑好几天的，所以挂在任何一台你已有的常开机器上就行（别处的免费小鸡、家里的服务器、树莓派都可以）。仓库里附了一个 systemd 服务示例：

```bash
sudo cp oracle-snatch.service /etc/systemd/system/
sudo systemctl enable --now oracle-snatch
tail -f snatch.log
```

## 踩过的坑

- 免费 A1 额度是总量，不是每台一份：一共 4 核 + 24 GB。可以开一台 4/24，也可以拆成两台 2/12。超了就是 LimitExceeded，脚本会主动停，别怪它。
- 账号升级成 PAYG（按量付费）不会吃掉免费额度，额度内的 A1 用量照样不收钱，体感上货还稍微多一点。我的账号就是 PAYG，账单至今 0 元，另外设了个预算警报兜底。
- 启动盘也算在 200 GB 免费块存储总量里。50 GB 对小服务器足够了，别因为「反正免费」默认拉满 200，剩下的额度留给第二台更划算。
- 放货时间没规律，经常在半夜。循环的意义就在这：凌晨 4 点它醒着，你不用。
- 别把重试间隔改成几秒。你不会更快抢到，只会吃 429，然后接下来一阵子什么都抢不到。

## 文件

- `snatch.py`：全部逻辑，一个文件
- `config.example.env`：所有配置项，带注释
- `oracle-snatch.service`：systemd 服务示例

## 授权

MIT
