# Squeeze Momentum (LazyBear) และ MTF SQZ v3: คู่มือ

_บันทึก: 2026-10-07_

โค้ดอยู่ที่ [`../pine/`](../pine/)

---

## 1. SQZMOM ของ LazyBear ทำงานอย่างไร

ดัดแปลงจาก **TTM Squeeze** ของ John Carter ทำงานสองอย่างพร้อมกัน:

1. **จุดบนเส้น 0** บอกว่าตลาดกำลังบีบตัว (ความผันผวนต่ำ) หรือระเบิดออกแล้ว
2. **แท่ง Histogram** บอกทิศทางและแรงของโมเมนตัม

### Squeeze: เทียบ Bollinger Bands กับ Keltner Channels

- **BB** กว้างหรือแคบตาม stdev ของราคาปิด
- **KC** กว้างหรือแคบตาม SMA ของ True Range (คล้าย ATR)

| เงื่อนไข | ความหมาย | สีจุด (ต้นฉบับ) |
|---|---|---|
| BB อยู่ข้างใน KC | ความผันผวนต่ำผิดปกติ กำลังสะสมพลัง | ⚫ ดำ (squeeze ON) |
| BB ครอบ KC | ความผันผวนกลับมาแล้ว (fire) | ⚪ เทา |
| คาบเกี่ยวกัน | ยังไม่ชัดเจน | 🔵 น้ำเงิน |

### Momentum

```
center = avg( avg(highest(high,20), lowest(low,20)),  sma(close,20) )
val    = linreg(close - center, 20, 0)
```

ค่าบวกแปลว่าราคาอยู่เหนือเส้นกลาง ส่วน `linreg` ช่วยทำให้เส้นเรียบ โดยหน่วงน้อยกว่า SMA

| สี | ความหมาย |
|---|---|
| 🟩 Lime | บวกและเพิ่มขึ้น (ขาขึ้นแรงขึ้น) |
| 🟢 Green | บวกแต่ลดลง (ขาขึ้นแผ่ว) |
| 🟥 Red | ลบและลดลง (ขาลงแรงขึ้น) |
| 🟤 Maroon | ลบแต่เพิ่มขึ้น (ขาลงแผ่ว) |

### จุดที่ควรรู้ของต้นฉบับ

- **ตัวแปร `mult` (2.0) ไม่ถูกใช้เลย** BB คำนวณด้วย `multKC` (1.5) แทน เงื่อนไข squeeze จึงลดรูปเหลือ `stdev < SMA(TR)` ใน v3 แก้ได้ด้วยตัวเลือก *Use BB Mult*
- squeeze บอกแค่ว่า "จะมีการเคลื่อนไหวแรง" ไม่ได้บอกทิศ ทิศมาจาก histogram ซึ่งเป็นสัญญาณที่ตามหลังราคา

---

## 2. MTF SQZ v3: สิ่งที่เพิ่มเข้ามา

- **5 time frame ในแผงเดียว** (ค่าเริ่มต้น 1H / 4H / 1D / 1W / 1M) แต่ละแถวเป็น histogram ของตัวเอง โดยเส้นกากบาทที่ฐานแถวคือสถานะ squeeze
- **Normalize:** ค่า momentum ของแต่ละ TF มีสเกลต่างกันมาก จึงต้องปรับสเกลก่อนวาด
  - `Range` (ค่าเริ่มต้น): หารด้วยค่าสูงสุดของ |momentum| ใน 100 แท่งย้อนหลัง ป้ายแสดงเป็น %
  - `ATR`: หารด้วย ATR ป้ายแสดงเป็นกี่เท่าของ ATR ทำให้เทียบข้าม TF ได้ตรงกว่า
- **No repaint:** ใช้ค่าของแท่ง TF ใหญ่ที่ปิดแล้วเท่านั้น (`[1]` + `lookahead_on`) สัญญาณจึงช้ากว่า 1 แท่งของ TF นั้น
- **squeeze ON เป็นสีส้ม** (ปรับได้) เพราะสีดำมองไม่เห็นในธีมมืด
- **ป้ายด้านขวาแต่ละแถว:** `TF ★  Signal  ค่า  [สถานะ]` ส่วนตารางปิดไว้เป็นค่าเริ่มต้น เพราะบังกราฟบนมือถือ

---

## 3. ความหมายของป้าย

ป้ายมีอยู่ **2 ที่** และความหมายต่างกัน:

| ที่ | คืออะไร |
|---|---|
| ป้ายด้านขวาของแถว / ตาราง | **สถานะ** ของ TF นั้น ณ ตอนนี้ |
| ป้ายบนกราฟราคา | **เหตุการณ์** ที่เกิดครั้งเดียว ตามกฎเข้าออก (ดูแถว ★ ประกอบกับ bias) |

| ป้าย | สีแท่ง | ความหมาย | ถ้าถือ long | ถ้าถือ short | ถ้ายังไม่มีสถานะ |
|---|---|---|---|---|---|
| **WAIT ▲/▼** | (squeeze ON) | สะสมพลัง ลูกศรบอกว่าโมเมนตัมเอียงไปทางไหน | ถือต่อ หรือระวัง | ถือต่อ หรือระวัง | **รอ** |
| **BUY** | 🟩 lime | ขาขึ้นแรงขึ้น | ถือต่อ | ควรปิด | พิจารณาเข้า long |
| **SELL** | 🟢 green | ขาขึ้นแผ่วลง | **ขายปิดหรือทำกำไร** | — | ไม่ควรไล่ซื้อ |
| **SHORT** | 🟥 red | ขาลงแรงขึ้น | ควรปิด | ถือต่อ | พิจารณาเข้า short |
| **COVER** | 🟤 maroon | ขาลงแผ่วลง | — | **ปิด short** | ไม่ควรไล่ short |

- **WAIT** ขึ้นได้แค่ในป้ายแถวและตาราง ลูกศรเป็นแค่ความเอียง ราคาอาจหลุดสวนทางก็ได้ ให้รอจนกากบาทเปลี่ยนจากส้มเป็นเทา
- **SELL ≠ SHORT:** SELL คือขายปิด long ที่ถืออยู่ ส่วน SHORT คือเปิดสถานะขาลงใหม่

---

## 4. กฎเข้าออก (ค่าเริ่มต้น)

ใช้แถว trigger = **Row 2 (4H)** และ bias = **1D + 1W**

| สัญญาณ | เกิดเมื่อ |
|---|---|
| **BUY** | ทุกแถว bias เป็นบวก **และ** แถว trigger เพิ่งหลุด squeeze หรือโมเมนตัมตัด 0 ขึ้น **และ** แท่ง trigger เป็นสี lime |
| **SELL** | ถือ long อยู่ แล้วแท่ง trigger ไม่ใช่สี lime แล้ว (โหมด Fade) หรือโมเมนตัมลงต่ำกว่า 0 (โหมด Zero cross) |
| **SHORT** | กลับด้านทั้งหมดจาก BUY (bias เป็นลบ, แท่งสีแดง) |
| **COVER** | ถือ short อยู่ แล้วแท่งไม่ใช่สีแดงแล้ว หรือโมเมนตัมขึ้นเหนือ 0 |

- ระบบจำสถานะไว้ใน `var int pos` จึงไม่ขึ้น BUY ซ้ำระหว่างที่ถือ long อยู่ ถ้ามีสัญญาณฝั่งตรงข้าม จะปิดสถานะเดิมและเปิดฝั่งใหม่ในแท่งเดียวกัน
- Alerts ที่มีให้: `BUY`, `SELL`, `SHORT`, `COVER` ควรตั้ง Trigger เป็น **Once Per Bar Close**
- **ไม่มี stop loss** ต้องตั้งเอง เช่น ใต้ low ของช่วง squeeze หรือห่างจากจุดเข้า 1.5 ATR

---

## 5. Time frame ที่เหมาะ

**ใช้กราฟ TF เดียวกับแถว trigger** หรือเล็กกว่า 1 ขั้น ห้ามใช้กราฟที่ใหญ่กว่า trigger เพราะแถว TF เล็กจะเหลือแค่ค่าของแท่งสุดท้าย

| สไตล์ | ถือประมาณ | เปิดกราฟ | Trigger row | Bias |
|---|---|---|---|---|
| Day trade | หลายชั่วโมง | 15m หรือ 1H | Row 1 (1H) | 4H + 1D |
| **Swing สั้น** ⭐ ค่าเริ่มต้น | 2–7 วัน | **4H** | Row 2 (4H) | 1D + 1W |
| Swing ยาว / Position | หลายสัปดาห์ | 1D | Row 3 (1D) | 1W + 1M |

- bias ควรเป็น TF ที่ใหญ่กว่า trigger ประมาณ 4–6 เท่า
- คริปโต (ETH): squeeze ใน 1H มักหลอก ให้เริ่มจาก 4H ก่อน และไม่ควรใช้ 1M เป็น bias ในการเทรดสั้น เพราะเปลี่ยนสีปีละไม่กี่ครั้ง
- ถ้าแถว bias ขัดกัน (เช่น 1D เป็นบวกแต่ 1W เป็นลบ) จะไม่มีสัญญาณขึ้นเลย ซึ่งถูกต้องตามกฎ

---

## 6. ติดตั้งและใช้งาน

1. Pine Editor → **Open → New indicator** → ลบโค้ดเดิม แล้ววางโค้ดจาก `mtf_sqzmom_v3.pine` → **Save** → **Add to chart**
2. ไปที่ **Indicators → My scripts** แล้วกด ⭐ เพื่อเพิ่มเป็นรายการโปรด หรือบันทึกเป็น **Indicator Template**
3. บนมือถือ ถ้าไม่อยากเห็นตัวเลขยาวหลังชื่ออินดิเคเตอร์ ให้ไปที่ Settings → Style → ยกเลิกติ๊ก **Inputs in status line**

---

## 7. เผยแพร่ (Publish)

- ทำผ่านเบราว์เซอร์บนคอมพิวเตอร์ แล้ว **ล้างกราฟให้เหลือแค่สคริปต์นี้** เพราะภาพกราฟจะกลายเป็นภาพปกของสคริปต์
- ใส่ header เครดิต LazyBear ไว้แล้ว ให้แก้ `© <TradingView username>` เป็นชื่อผู้ใช้ของตัวเอง
- ประเภทการเผยแพร่:

  | ตัวเลือก | ต้องใช้แพ็กเกจ |
  |---|---|
  | **Open** | ฟรี ✅ ควรเลือกแบบนี้ เพราะดัดแปลงจากโค้ดโอเพนซอร์ส |
  | Protected | แพ็กเกจเสียเงินใดก็ได้ |
  | Invite-only | Premium ขึ้นไป |

  ถ้าขึ้นหน้าให้อัปเกรด ให้ตรวจว่าไม่ได้เลือก Protected หรือ Invite-only ไว้

- **คำอธิบายต้องเป็นภาษาอังกฤษก่อน** และห้ามใส่ลิงก์, โฆษณา หรือคำอ้างเกินจริงอย่าง "แม่น 90%"
- สคริปต์ Public ลบเองไม่ได้ และเปลี่ยนกลับเป็น Private ไม่ได้ ให้เผยแพร่แบบ **Private** ทดสอบก่อน ถ้าแก้โค้ดภายหลัง ให้กด **Update existing script** แทนการเผยแพร่ใหม่
- ทางเลือกที่ไม่ต้อง publish: แชร์ไฟล์ `.pine` ให้คนอื่นไปวางใน Pine Editor ของตัวเอง

<details>
<summary>ร่างคำอธิบายภาษาอังกฤษ</summary>

```
MTF Squeeze Momentum Dashboard

Based on the Squeeze Momentum Indicator by LazyBear (concept: John Carter's TTM Squeeze).
This script shows the squeeze state and momentum of 5 timeframes (default 1H/4H/1D/1W/1M) stacked in one pane.

HOW IT WORKS
• Squeeze: Bollinger Bands inside Keltner Channels = squeeze ON (orange cross). BB expanding outside KC = fired (gray).
• Momentum: linear regression of price vs. the midpoint of Donchian channel and SMA (same as the original).
• Each timeframe is normalized (by its own recent range or by ATR) so bar heights are comparable across timeframes.

SIGNALS
• BUY / SHORT: trigger timeframe fires out of a squeeze or crosses zero, momentum is accelerating, and the selected higher-timeframe bias rows agree.
• SELL / COVER: trigger-timeframe momentum fades (or crosses zero, selectable).
• Uses confirmed higher-timeframe bars by default (no repaint), so signals appear one trigger bar late.

NOTES
• Use a chart timeframe equal to or lower than the trigger row.
• Option to fix the original script's BB multiplier quirk.
• This is an analysis tool, not financial advice. No stop-loss logic is included.
```
</details>

---

## 8. สิ่งที่ยังไม่ได้ทำ

- [ ] รันจริงใน TradingView เพื่อยืนยันว่าคอมไพล์ผ่าน
- [ ] แปลงเป็น `strategy()` เพื่อ backtest กับ ETH และเทียบทั้ง 3 ชุดค่าตามสไตล์การเทรด
- [ ] เพิ่ม stop loss (ATR-based)
- [ ] (ถ้าต้องการ) เปลี่ยนคำในป้ายแถวจาก SELL/COVER เป็น "UP fading" / "DOWN fading" เพื่อไม่ให้สับสนกับป้ายบนกราฟ

> ไม่ใช่คำแนะนำการลงทุน
