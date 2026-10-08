---
name: smart-money-concepts
description: วิเคราะห์กราฟและเขียน/แก้ Pine Script ด้วย Smart Money Concepts (SMC) — market structure (BOS / CHoCH), order block, fair value gap, liquidity (EQH / EQL, sweep), premium / discount และการใช้ร่วมกับ MTF Squeeze Momentum ใน repo นี้. ใช้เมื่อผู้ใช้ส่ง screenshot กราฟให้อ่าน, ถามเรื่อง SMC / ICT / order block / FVG / liquidity, หรือขอแก้ไข pine/smc_sqz_v2.pine
---

# Smart Money Concepts (SMC)

โค้ดหลัก: [`pine/smc_sqz_v2.pine`](../../../pine/smc_sqz_v2.pine) (overlay บนกราฟราคา)
ใช้คู่กับ: [`pine/mtf_sqzmom_v3.pine`](../../../pine/mtf_sqzmom_v3.pine) (แผงล่าง) และคู่มือ [`docs/sqzmom_guide.md`](../../../docs/sqzmom_guide.md)

ตอบผู้ใช้เป็นภาษาไทย ใช้ศัพท์เทคนิคภาษาอังกฤษตามที่เทรดเดอร์ใช้กัน (BOS, CHoCH, OB, FVG)
ปิดท้ายทุกการวิเคราะห์ด้วยประโยคว่าไม่ใช่คำแนะนำการลงทุน

---

## 1. แนวคิดหลัก

| แนวคิด | นิยามที่ใช้ใน repo นี้ | ใช้ทำอะไร |
|---|---|---|
| **Swing high / low** | `ta.pivothigh/pivotlow(len, len)` ยืนยันช้า `len` แท่งเสมอ | จุดอ้างอิงของโครงสร้าง |
| **BOS** (Break of Structure) | แท่ง **ปิด** ทะลุ swing ล่าสุด **ไปทางเดียวกับ** trend เดิม | ยืนยันว่า trend ยังไปต่อ |
| **CHoCH** (Change of Character) | แท่งปิดทะลุ swing **สวน** trend เดิม | สัญญาณแรกว่า trend อาจกลับ ยังไม่ใช่การยืนยัน |
| **Order block (OB)** | bullish OB = แท่งที่ low ต่ำสุดใน leg ก่อนเบรกขึ้น / bearish OB = แท่งที่ high สูงสุดก่อนเบรกลง | โซนที่คาดว่าราคาจะกลับมา retest |
| **Mitigation** | OB ถือว่าใช้ไม่ได้เมื่อแท่ง **ปิด** ทะลุอีกฝั่งของโซน | ลบโซนทิ้ง |
| **FVG** (Fair Value Gap) | 3 แท่ง: `low > high[2]` (bullish) หรือ `high < low[2]` (bearish) และกว้าง ≥ `fvgMinAtr × ATR` | ช่องว่างที่ราคามักกลับมาเติม |
| **EQH / EQL** | pivot 2 จุดติดกันห่างกันไม่เกิน `eqThr × ATR` | กองรวม stop loss (liquidity) ที่ราคามักวิ่งไปกวาด |
| **Liquidity sweep** | ไส้เทียนทะลุ EQH/EQL หรือ swing แล้ว **ปิดกลับ** เข้ามา | มักเกิดก่อน CHoCH จริง |
| **Premium / Discount** | ครึ่งบน / ครึ่งล่างของ dealing range (swing high ↔ swing low ล่าสุด) | ซื้อใน discount, ขายใน premium |

### ลำดับเหตุการณ์ที่ดี (A+ setup)

1. TF ใหญ่ (ค่าเริ่มต้น 1D) เป็นขาขึ้น
2. TF เทรดเกิด **liquidity sweep** ใต้ EQL หรือ swing low
3. ตามด้วย **CHoCH ขึ้น** และมี FVG เกิดขึ้นในแรงเบรก
4. ราคาย่อกลับมาแตะ **bullish OB / FVG ที่อยู่ใน discount**
5. โมเมนตัม SQZ เริ่มหันขึ้น เข้าตรงนี้ โดยวาง stop ใต้ OB และเป้าแรกที่ EQH หรือ swing high ถัดไป

ฝั่ง short ให้กลับด้านทั้งหมด

---

## 2. สิ่งที่ `smc_sqz_v2.pine` ทำ (ปรับปรุงจาก SMC ทั่วไป)

- **ไม่ repaint:** ทุกอย่างคำนวณเฉพาะแท่งที่ปิดแล้ว (`barstate.isconfirmed`) โครงสร้าง TF ใหญ่ใช้แท่งที่ปิดแล้ว (`t[1]` + `lookahead_on`)
- **Liquidity sweep (`$`):** ไส้เทียนทะลุ swing high/low หรือ EQH/EQL แล้วปิดกลับเข้ามา ถ้าแท่งปิดทะลุจริง ถือว่า level นั้นหมดความหมาย
- **เกรด A+:** โซน (OB/FVG) ที่เกิดภายใน `sweepWin` แท่งหลัง sweep ฝั่งเดียวกัน ถ้าเปิด *แสดงเฉพาะสัญญาณ A+* จะเหลือเฉพาะ setup ตามลำดับในหัวข้อ 1
- **สัญญาณ confluence `SMC ▲ / ▼`** ขึ้นเมื่อครบทุกข้อที่เปิดใช้:
  - ราคาแตะ OB (หรือ FVG ถ้าเปิด *นับการแตะ FVG*) ฝั่งเดียวกับ trend **เป็นครั้งแรก** โซนที่ถูกแตะแล้วจะจางลง
  - trend ของ TF กราฟตรงกัน
  - อยู่ใน discount (long) หรือ premium (short)
  - momentum ของ LazyBear กำลังเพิ่มขึ้น (long) หรือกำลังลดลง (short)
  - โครงสร้าง TF ใหญ่ตรงกัน ถ้าตั้ง TF ใหญ่เล็กกว่ากราฟ ตัวกรองนี้จะถูกข้ามและตารางขึ้นเตือนสีส้ม
- **SL / TP ของสัญญาณล่าสุด:** SL = ขอบนอกของโซนที่แตะ ± `slBuf × ATR14`, TP1 = `rrTp` R, TP2 = liquidity ฝั่งตรงข้ามที่ยังไม่ถูกกวาด
- **ตารางสรุป:** trend ของ TF กราฟ / TF ใหญ่, ราคาอยู่กี่ % ของ range (premium/discount), โมเมนตัม, เหตุการณ์ล่าสุด
- **เบาบนมือถือ:** จำกัดจำนวน OB และ FVG ต่อฝั่ง ใช้ `extend.right` จึงไม่ต้องอัปเดตกล่องทุกแท่ง ส่วน premium/discount และตารางวาดแค่บนแท่งสุดท้าย
- **Alerts:** CHoCH, BOS, sweep, zone tap, SMC ▲/▼, SMC ▲/▼ A+ ทุกตัวควรตั้งเป็น **Once Per Bar Close**
- ชื่อป้ายไม่ชนกับ MTF SQZ (BUY / SELL / SHORT / COVER) จึงเปิดสองตัวพร้อมกันได้

---

## 3. วิธีอ่าน screenshot กราฟที่ผู้ใช้ส่งมา

ทำตามลำดับนี้ แล้วตอบเป็นหัวข้อสั้น ๆ:

1. **บริบท:** สินทรัพย์, TF, ราคาปัจจุบัน และอินดิเคเตอร์ที่เปิดอยู่ (บางตัวอาจถูกซ่อน เช่นไอคอนตาขีดฆ่า)
2. **โครงสร้าง:** swing high/low ที่เห็นชัด, BOS หรือ CHoCH ล่าสุดเป็นทางไหน และตอนนี้เป็น trend อะไร
3. **โซน:** OB, FVG, EQH/EQL ที่ยังไม่ถูกใช้ ใกล้ราคาที่สุดฝั่งละโซน
4. **ตำแหน่ง:** ราคาอยู่ใน premium หรือ discount ของ range ล่าสุด
5. **โมเมนตัม:** ถ้ามีแผง MTF SQZ ให้อ่านแถวที่มี ★ และแถว bias (ดู `docs/sqzmom_guide.md` หัวข้อ 3)
6. **สรุปเป็นสถานการณ์ (scenario)** ไม่ใช่คำสั่งซื้อขาย: "ถ้าราคาลงมาแตะ X แล้วเกิด CHoCH ขึ้นใน TF เล็ก → long, stop ใต้ Y, เป้า Z"

ห้ามเดาตัวเลขที่อ่านจากรูปไม่ออก ถ้าอ่านไม่ชัดให้บอกตรง ๆ และขอรูปที่ซูมเข้าไป

---

## 4. กฎการเขียน / แก้ Pine ด้าน SMC

- ใช้ `//@version=6` และหัวไฟล์ MPL-2.0 แบบไฟล์อื่นใน `pine/`
- **ห้ามคัดลอกโค้ดจาก LuxAlgo หรือสคริปต์ที่ใช้สัญญาอนุญาต CC BY-NC-SA** เขียนเองจากแนวคิดเท่านั้น
- ต้องเรียก `ta.*` ทุกแท่งที่ global scope ห้ามเรียกใน `if` หรือ loop
- ถ้าวน array แล้วลบสมาชิก ให้วนจากท้ายไปหน้า และเช็ก `size() > 0` ก่อนเสมอ (`for i = -1 to 0` ใน Pine ยังวนอยู่)
- `request.security` ไป TF ที่ใหญ่กว่า ต้องใช้ค่า `[1]` + `lookahead_on` เท่านั้น
- ระวังขีดจำกัด `max_boxes_count`, `max_lines_count`, `max_labels_count` (สูงสุด 500) ให้ลบของเก่าเองเมื่อเกินจำนวนที่ผู้ใช้ตั้ง
- ข้อความ input และ tooltip เป็นภาษาไทย ส่วนข้อความ alert เป็นภาษาอังกฤษ
- รันใน TradingView ไม่ได้จาก session นี้ ให้บอกผู้ใช้ทุกครั้งว่ายังไม่ได้คอมไพล์จริง และให้ส่ง error กลับมาถ้ามี

---

## 5. ค่าเริ่มต้นตามสไตล์

| สไตล์ | กราฟ | Swing length | TF ใหญ่ | ใช้คู่กับ MTF SQZ trigger |
|---|---|---|---|---|
| Day trade | 15m / 1H | 5–8 | 4H | Row 1 (1H) |
| **Swing สั้น** ⭐ | **4H** | **10** | **1D** | Row 2 (4H) |
| Position | 1D | 10–20 | 1W | Row 3 (1D) |

ทองคำ (XAUUSD) วิ่งแรงช่วงข่าว US ให้เพิ่ม `fvgMinAtr` เป็น 0.4–0.5 เพื่อลด FVG เล็ก ๆ จากข่าว

---

## 6. สิ่งที่ยังไม่ได้ทำ

- [ ] คอมไพล์และรันจริงใน TradingView
- [ ] internal structure (swing เล็กภายใน swing ใหญ่)
- [ ] แปลงเป็น `strategy()` เพื่อ backtest (ใช้ SL / TP ที่คำนวณไว้แล้ว)
- [ ] trailing stop หลังถึง TP1
