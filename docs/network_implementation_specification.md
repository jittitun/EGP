# Implementation Specification
## Corrected Financial Model for network.html

**Version:** 1.0  
**Status:** Approved implementation specification before code modification  
**Target:** `network.html` single-file client-side application  
**Test dataset:** 16 CSV files, Chiang Mai, Chiang Rai, Lamphun and Mae Hong Son, fiscal years 2566–2569

## 1. Objective

ปรับโครงสร้างข้อมูลและสูตรคำนวณของ `network.html` เพื่อแยกค่าระดับโครงการออกจากค่าระดับสัญญา ป้องกันการนำมูลค่ารวมของโครงการไปนับซ้ำให้ผู้รับจ้างแต่ละรายในโครงการที่มีหลายสัญญา พร้อมแสดงวงเงินงบประมาณ ราคากลาง มูลค่าจัดหารวม และมูลค่าตามสัญญาอย่างครบถ้วนและตรงระดับข้อมูล

ทุกผลลัพธ์ยังคงเป็น **สัญญาณคัดกรอง** ไม่ใช่ข้อสรุปว่ามีการทุจริต

## 2. Non-negotiable constraints

1. คงเป็น single-file HTML ไม่มี build step
2. ประมวลผล client-side 100% ไม่มี backend/API ส่งข้อมูลออก
3. รักษา PapaParse worker, chunked processing, IndexedDB fallback และ GitHub discovery fallback
4. ไม่ใช้ `project_id` เป็น dedup key เพียงลำพัง
5. แดงใช้เฉพาะสัญญาณความเสี่ยง; data-quality ใช้ amber/blue
6. การแก้ logic ต้องสร้าง corrected baseline ใหม่ และเก็บ legacy baseline ไว้เปรียบเทียบ
7. ห้าม fallback `sum_price_agree` ให้ทุกแถวในโครงการหลายสัญญา

## 3. Financial data dictionary

| Source field | UI label | Data level | Aggregation | Primary use |
|---|---|---|---|---|
| `project_money` | วงเงินงบประมาณ | Project | หนึ่งครั้งต่อ `project_id` | เกณฑ์วงเงิน วิธีเฉพาะเจาะจง |
| `price_build` | ราคากลาง | Project | หนึ่งครั้งต่อ `project_id` | ฐานคำนวณส่วนลด |
| `sum_price_agree` | มูลค่าจัดหารวม | Project | หนึ่งครั้งต่อ `project_id` | ผลการจัดหารวมของโครงการ |
| `contract_price_agree` | มูลค่าตามสัญญา | Contract/Vendor | รวมตาม contract key | Network, vendor, agency, HHI, trend |

## 4. Canonical entities

### 4.1 Raw row
ผลหลัง full-row dedup ด้วย `STD_COLS` เดิม ห้ามเปลี่ยน

### 4.2 Project entity
Key: `project_id`

Fields:
- `projectId`
- `projectName`
- `province`
- `year`
- `dept`
- `agency`
- `method`
- `workCategory`
- `projectBudget`
- `referencePrice`
- `procurementValue`
- `contractCount`
- `vendorCount`
- `contractValueSum`
- `reconciliationDifference`
- `reconciliationStatus`
- variant counts สำหรับตรวจค่าระดับโครงการไม่สอดคล้องกัน

Canonical money rule:
- ใช้ mode ของค่าที่มากกว่า 0 ภายใน project
- ถ้ามี tie ใช้ค่ามากกว่าและติด data-quality flag
- ห้ามรวมค่าระดับโครงการตามจำนวน raw rows

### 4.3 Contract entity
Primary key:
`project_id + contract_no + winner_tin + winner_name`

กรณีข้อมูลไม่ครบ:
- หากไม่มี TIN ใช้ชื่อผู้รับจ้าง
- หากไม่มี contract no แต่มีผู้รับจ้าง ให้ contract key ยังแยกตามผู้รับจ้าง
- หากไม่มีทั้ง contract no และ vendor identity ให้ใช้ row-specific fallback key และติด data-quality flag

Fields:
- project context fields
- `contractNo`
- `contractDate`
- `vendor`
- `winnerTin`
- `contractValueRaw`
- `contractValueEffective`
- `contractValueSource`

Contract value rule:
1. ใช้ `contract_price_agree` เมื่อมากกว่า 0
2. หากโครงการมี contract entity เพียงหนึ่งรายการและ `contract_price_agree` หาย ให้ fallback เป็น `sum_price_agree`
3. หากโครงการมีหลาย contract entity และค่ามูลค่าสัญญาหาย ให้เป็น 0/null สำหรับ value aggregation และติด `missing_multi_contract`
4. ห้ามกระจายหรือคัดลอก `sum_price_agree` ไปให้แต่ละสัญญา

## 5. Reconciliation

ต่อหนึ่ง project:
`contractValueSum = Σ contractValueEffective`

Tolerance:
`max(1 บาท, procurementValue × 0.1%)`

Statuses:
- `matched`
- `mismatch`
- `incomplete_contract_values`
- `missing_procurement_value`
- `missing_both`

Data-quality statuses ไม่ใช่ risk flags

## 6. Metric-to-level mapping

### Project-level metrics
- จำนวนโครงการ
- วงเงินงบประมาณรวม
- ราคากลางรวม
- มูลค่าจัดหารวม
- สัดส่วนวิธีจัดหา
- เกณฑ์เฉพาะเจาะจง >500,000
- ช่วง 450,000–500,000
- ช่วง 300,000–500,000
- ส่วนลดจากราคากลาง
- จัดหาสูงกว่าราคากลาง
- มูลค่าจัดหากลม
- e-bidding ส่วนลดประมาณ 0%

Primary threshold field for specific procurement:
`project_money`

Discount:
`(price_build - sum_price_agree) / price_build`

### Contract-level metrics
- จำนวนสัญญา/contract records
- มูลค่าที่ผู้รับจ้างได้รับ
- มูลค่า edge หน่วยงาน–ผู้รับจ้าง
- Vendor ranking
- Agency spending by vendor
- HHI
- Top 10/20/50/100 concentration
- Capture
- Method-lock
- Cross-type
- High-spread
- Single-source dependence
- Vendor pool
- Split procurement
- Persistent winner
- New entrant spike
- Growing no competition
- Cross-province

## 7. Risk flag rules after correction

สูตรเกณฑ์ตัวเลขคงเดิม แต่ฐานมูลค่าเปลี่ยนเป็น `contractValueEffective` สำหรับธงระดับ vendor/agency

- Capture: vendor เชื่อมหน่วยงานเดียว และ contract value รวม > `capMin`
- Method-lock: contract records ของ vendor เป็นเฉพาะเจาะจงทั้งหมด และ contract value รวม > `lockMin`
- Cross-type: จำนวน work category และ contract value
- High-spread: จำนวนหน่วยงาน
- SSD: top vendor share จาก contract value
- Vendor pool: unique vendors / contract records

## 8. Split procurement

Unit: contract entity ไม่ใช่ raw row

Group key:
`agency + vendor + contract_date + specific method`

Count:
จำนวน contract entities ไม่ซ้ำ

Value:
ผลรวม `contractValueEffective`

Drill-down ต้องแสดง:
- รหัสโครงการ
- เลขที่สัญญา
- ผู้รับจ้าง
- วงเงินงบประมาณ
- ราคากลาง
- มูลค่าจัดหารวม
- มูลค่าตามสัญญา
- แหล่งที่มาของมูลค่าสัญญา

## 9. UI changes

### 9.1 Table conventions
Project table:
- วงเงินงบประมาณ
- ราคากลาง
- มูลค่าจัดหารวม
- จำนวนสัญญา
- ผลรวมมูลค่าตามสัญญา
- reconciliation status

Contract table:
- วงเงินงบประมาณ (ค่าระดับโครงการ)
- ราคากลาง (ค่าระดับโครงการ)
- มูลค่าจัดหารวม (ค่าระดับโครงการ)
- มูลค่าตามสัญญา (ค่าที่รวมได้)

Tooltip:
“ค่าระดับโครงการแสดงซ้ำเพื่อประกอบการอ่าน ห้ามนำมารวมตามจำนวนแถว”

### 9.2 New data-quality panel
ชื่อ: `🧾 ความสอดคล้องของข้อมูลมูลค่า`

KPIs:
- matched projects
- mismatch projects
- incomplete contract values
- multiple project-level values
- duplicate contract number
- same contract multiple vendors

ใช้ amber/blue ไม่ใช้ red

### 9.3 Terminology
ห้ามใช้คำว่า “ราคากลาง” กับ `project_money`
ห้ามใช้ “มูลค่า” โดยไม่ระบุชนิด
ใช้:
- วงเงินงบประมาณ
- ราคากลาง
- มูลค่าจัดหารวม
- มูลค่าตามสัญญา

## 10. Code-change map

1. `normalizeRow()`  
   เพิ่ม money fields ทั้ง 4; เปลี่ยนชื่อกำกวม `v`, `budget`

2. หลัง dedup  
   สร้าง `PROJECT_MAP`, `CONTRACT_ROWS`

3. `computeMetrics()`  
   รับ contract rows; ใช้ contract value

4. Specific view  
   รับ project rows; นับ project ID ครั้งเดียว

5. Pricing view  
   รับ project rows; `price_build` เทียบ `sum_price_agree`

6. Split view  
   รับ contract rows; count contract entities

7. Trend  
   ใช้ contract rows; project filter context ต้องสอดคล้อง

8. Detail/export renderers  
   เพิ่ม financial columns และ level labels

9. Session persistence  
   เก็บ normalized project/contract model หรือ rebuild อย่าง deterministic โดยไม่เกิน quota

10. Regression diagnostics  
    รองรับ legacy baseline และ corrected baseline คนละ version

## 11. Acceptance criteria

Functional:
- โหลด 16 ไฟล์พร้อมกันได้
- UI ไม่ค้าง
- ไม่มี console error
- project count = 137,099
- contract records = 146,829
- contracts with vendor = 146,766
- corrected network value = 170,454,025,199.45 บาท
- procurement value total = 169,094,601,123.27 บาท
- matched projects = 137,061
- mismatch projects = 20
- incomplete projects = 18

Corrected risk baseline:
- Capture 1,098
- Method-lock 505
- Cross-type 512
- High-spread 76
- SSD 68
- Vendor pool 27
- Top 10/20/50/100 = 12.8514% / 18.3278% / 30.0222% / 39.5426%
- Split groups 918
- Split contracts 7,109
- Split contract value 5,675,981,446.11 บาท

Project-level baseline:
- Specific projects 111,458
- Specific project share 81.2975%
- Specific procurement value 49,649,656,680.38 บาท
- Budget >500,000 = 4,469
- Budget 450,000–500,000 = 32,816
- Budget 300,000–500,000 = 70,758
- Pricing comparable projects = 136,782
- Zero discount = 71,345
- Procurement over reference price = 1,190
- Over reference price ≥1.5x = 246
- e-bidding zero discount = 1,457

## 12. Required test cases

1. Single project, single contract, all values present
2. Single project, multiple contracts, values sum to procurement total
3. Multiple contracts, one contract value missing
4. Single contract with missing contract value; fallback allowed
5. Same contract number, multiple vendors
6. Project-level values conflict across rows
7. Missing project ID
8. Missing winner TIN
9. Multiple provinces and years
10. Session restore
11. Parameter change and recalculation
12. CSV export preserves four financial fields
13. Existing e-GP links still work
14. Screening-language disclaimer remains visible

## 13. Codex implementation sequence

1. Create branch `fix/contract-level-financial-model`
2. Add baseline JSON and specification
3. Introduce pure helper functions and tests
4. Build project/contract entities
5. Replace network calculations
6. Replace project-level views
7. Add data-quality view
8. Update UI/export
9. Run corrected regression
10. Compare screenshots and responsive behavior
11. Commit in logical stages
12. Open draft PR with baseline results

## 14. Out of scope for this iteration

- Server-side processing
- AI-generated fraud conclusions
- Vendor identity normalization by TIN replacing legacy name key
- Community detection
- New weighted risk score
- Legal conclusion on specific-procurement authority
