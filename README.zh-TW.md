[中文](README.md) | **繁體中文** | [English](README.en.md) | [日本語](README.ja.md) | [한국어](README.ko.md)

# oracle-snatch

![GitHub stars](https://img.shields.io/github/stars/lucasforster1108-boop/oracle-snatch) ![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)

一個小的 Python 迴圈指令碼：不停用甲骨文雲的官方介面申請免費 ARM 例項（VM.Standard.A1.Flex），直到哪次趕上放貨、介面點頭為止。

## 為什麼寫這個

Always Free 的 A1.Flex 免費額度（總共 4 核 / 24 GB）是目前最划算的免費伺服器，沒有之一。問題是熱門區域在控制檯裡基本永遠顯示「Out of host capacity」（沒貨）。這個貨不是永久沒了，而是會不定期放出來一小波，沒人通知。對著控制檯刷一週重新整理純屬浪費生命，不如讓指令碼替你去問。

2026 年 10 月我用一個粗糙版本在東京區域跑了好幾天。日誌翻來覆去就是「沒貨」，中間還因為我把間隔設得太密、吃過幾次 429 限流。最後搶到了那臺 4 核 24 GB 的機器，我現在自託管的大部分服務都跑在它上面，到今天賬單還是 0。這個倉庫就是那個迴圈的整理版，配置都抽了出來。

後來我把這段經歷發到 X 上，帖子有 31,885 次瀏覽、470 人收藏——看來被「沒貨」折磨的人遠不止我一個。

## 它做什麼

- 只用甲骨文官方 API（Python 的 `oci` SDK），不搞控制檯自動化、不模擬瀏覽器。
- 輪詢你所在區域的每一個可用域（AD），每輪每個 AD 嘗試一次。東京只有一個 AD，多 AD 的區域中籤率會好一些。
- 「沒貨」就當「還沒到時候」：歇一會兒，下一個 AD，下一輪。
- 遇到 429 限流就長退避（預設 300 秒）。硬頂著 429 往前衝，只會讓限流賴著不走。
- 遇到配額錯誤（LimitExceeded）直接停，並把原因寫清楚：這說明你的免費額度已經用完了，再重試一萬次也沒用。
- 搶到後等例項變成 RUNNING，拿到公網 IP，寫進 `snatched.json`，發一條通知，然後按 `TARGET_COUNT` 決定是退出還是接著搶下一臺（預設 1 臺；設成 2 再配上 2 核 / 12 GB，就是一次搶兩臺小的）。
- 通知不止 Telegram：設了 `WEBHOOK_URL` 還會往通用 Webhook 發一條 JSON，Bark、Server醬、企業微信機器人都能接。子網支援 IPv6 的話，打開 `ASSIGN_IPV6_IP` 還能給例項分配 IPv6 地址。
- `--dry-run` 只驗證配置：列出可用域、確認會用哪個映象，不真開機器。第一次先跑這個。

它不做的事：不幫你註冊賬號、不建 VCN、不生成 API 金鑰，更變不出不存在的貨。它只是替你不知疲倦地問。

## 安裝與使用

需要一個甲骨文雲賬號，以及 OCI 的配置檔案（`~/.oci/config`，裡面配好 API 金鑰）。用過 OCI CLI 或 SDK 的人都有這個檔案。

還需要一個子網 OCID：控制檯裡 Networking -> Virtual Cloud Networks -> 你的 VCN -> Subnets。想要公網 IP 就選公有子網（你肯定想要）。

```bash
git clone https://github.com/lucasforster1108-boop/oracle-snatch.git
cd oracle-snatch
pip install -r requirements.txt

cp config.example.env config.env
# 改 config.env：必填的只有 SUBNET_ID 和 SSH_PUBLIC_KEY_FILE

python3 snatch.py --dry-run
python3 snatch.py
```

## 長期掛著跑

這東西就是設計成無人值守跑好幾天的，所以掛在任何一臺你已有的常開機器上就行（別處的免費小雞、家裡的伺服器、樹莓派都可以）。倉庫裡附了一個 systemd 服務示例：

```bash
sudo cp oracle-snatch.service /etc/systemd/system/
sudo systemctl enable --now oracle-snatch
tail -f snatch.log
```

## 踩過的坑

- 免費 A1 額度是總量，不是每臺一份：一共 4 核 + 24 GB。可以開一臺 4/24，也可以拆成兩臺 2/12。超了就是 LimitExceeded，指令碼會主動停，別怪它。
- 賬號升級成 PAYG（按量付費）不會吃掉免費額度，額度內的 A1 用量照樣不收錢，體感上貨還稍微多一點。我的賬號就是 PAYG，賬單至今 0 元，另外設了個預算警報兜底。
- 啟動盤也算在 200 GB 免費塊儲存總量裡。50 GB 對小伺服器足夠了，別因為「反正免費」預設拉滿 200，剩下的額度留給第二臺更划算。
- 放貨時間沒規律，經常在半夜。迴圈的意義就在這：凌晨 4 點它醒著，你不用。
- 別把重試間隔改成幾秒。你不會更快搶到，只會吃 429，然後接下來一陣子什麼都搶不到。

## 檔案

- `snatch.py`：全部邏輯，一個檔案
- `config.example.env`：所有配置項，帶註釋
- `oracle-snatch.service`：systemd 服務示例

## 授權

MIT
