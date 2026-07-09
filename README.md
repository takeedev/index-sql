# สรุปเรื่อง Database Index (PostgreSQL เป็นหลัก + เทียบข้ามยี่ห้อ)
 
> เอกสารสรุปความรู้เรื่องการทำ Index — ข้อดี ข้อเสีย การจัดการเมื่อดิสก์โต สิ่งที่ควรรู้ และการเทียบคำสั่งข้าม database
> **หลักการ (หัวข้อ 2–5, 9) ใช้ได้ทุกยี่ห้อ** เพราะเกือบทุก DB ใช้ B-tree เหมือนกัน / **คำสั่ง (หัวข้อ 6–8) เป็นของ PostgreSQL** ดูตารางเทียบยี่ห้ออื่นได้ที่หัวข้อ 11
 
---
 
## 1. Index คืออะไร
 
Index เปรียบเสมือน "สารบัญ" ของตาราง — แทนที่ database จะต้องไล่อ่านข้อมูลทุกแถว (Seq Scan) มันสามารถเปิดสารบัญแล้วกระโดดไปยังแถวที่ต้องการได้ทันที (Index Scan)
 
ตัวอย่างในงานจริง:
 
```sql
create index idx_stmt_act_end
  on cardlite.statement_summary (act_idn_sky, prd_end_dte desc);
```
 
Index นี้ช่วยให้ query ที่ค้นหา "statement รอบปิดล่าสุดของแต่ละ account" ทำงานเร็วขึ้นมาก เพราะข้อมูลถูกเรียงตาม account และวันปิดรอบไว้ล่วงหน้าแล้ว
 
---
 
## 2. ข้อดีของ Index
 
| ข้อดี | คำอธิบาย |
|---|---|
| **Query เร็วขึ้นมาก** | เปลี่ยนจาก Seq Scan (อ่านทุกแถว) เป็น Index Scan (กระโดดหาเฉพาะแถวที่ต้องการ) — จากหลายวินาทีอาจเหลือหลัก ms |
| **ช่วย WHERE / JOIN / ORDER BY** | คอลัมน์ที่ถูกใช้กรอง เชื่อมตาราง หรือเรียงลำดับบ่อย ๆ จะได้ประโยชน์โดยตรง |
| **ลดปัญหา timeout** | Query ที่ช้าจนทำให้ระบบ timeout มักแก้ได้ด้วย index ที่ถูกจุด |
| **ช่วยงาน Top-N-per-group** | เช่น "หารอบล่าสุดของแต่ละ account" — index ที่เรียงลำดับไว้แล้วทำให้หยิบแถวบนสุดได้ทันทีโดยไม่ต้อง sort ใหม่ |
| **บังคับความถูกต้องของข้อมูล** | Unique index ป้องกันข้อมูลซ้ำ (เช่น primary key) |
 
---
 
## 3. ข้อเสียของ Index
 
| ข้อเสีย | คำอธิบาย |
|---|---|
| **กินพื้นที่ดิสก์** | โดยทั่วไปราว 10–30% ของขนาดตาราง ต่อ index หนึ่งตัว |
| **เขียนข้อมูลช้าลง** | ทุก INSERT / UPDATE / DELETE ต้องอัปเดต index ทุกตัวของตารางด้วย — ยิ่ง index เยอะ ยิ่งเขียนช้า |
| **Index บวม (bloat)** | ตารางที่ update/delete บ่อยจะทำให้ index บวม กินที่มากกว่าข้อมูลจริง ต้อง reindex เป็นระยะ |
| **Planner อาจสับสน** | Index เยอะเกินไป อาจทำให้ database เลือกใช้ index ผิดตัว ได้ผลช้ากว่าเดิม |
| **ภาระการดูแล** | ต้องคอยตรวจว่า index ตัวไหนถูกใช้จริง ตัวไหนเป็นขยะ |
 
> **หลักคิด:** ดิสก์ที่เพิ่ม = ราคาที่จ่ายเพื่อแลกกับความเร็ว คำถามไม่ใช่ "ทำยังไงไม่ให้โต" แต่คือ **"คุ้มไหม"** — index ที่แก้ปัญหา timeout ได้ ย่อมคุ้มค่าดิสก์ 1–3 GB เสมอ ส่วน index ที่ไม่มีใครใช้ กี่ MB ก็ไม่คุ้ม
 
---
 
## 4. เมื่อไหร่ควรสร้าง Index
 
- คอลัมน์ถูกใช้ใน **WHERE / JOIN / ORDER BY บ่อย ๆ** และตารางมีขนาดใหญ่
- รัน `EXPLAIN ANALYZE` แล้วเห็น **Seq Scan บนตารางใหญ่** ที่กินเวลานาน — นี่คือหลักฐานตรงที่สุด
- Query ช้าจริงและกระทบผู้ใช้ (เช่น รายงาน timeout)
- ตารางโตต่อเนื่อง (เช่น ตาราง statement / transaction ที่เพิ่มทุกเดือน)
**หลักการสำคัญ: สร้างตามหลักฐาน ไม่ใช่ตามความรู้สึก** — รัน `EXPLAIN ANALYZE` ก่อนเสมอ
 
## 5. เมื่อไหร่ *ไม่ควร* สร้าง Index
 
| กรณี | เหตุผล |
|---|---|
| ตารางเล็กมาก (หลักร้อย–พันแถว) | Seq Scan เร็วกว่าอยู่แล้ว database จะไม่ใช้ index ที่สร้าง |
| ตารางที่เขียนถี่มากแต่อ่านน้อย (log, audit) | ทุก index คือภาระตอน insert |
| คอลัมน์ค่าซ้ำเยอะ (low cardinality) เช่น flag '0'/'1' | กรองแล้วยังเหลือครึ่งตาราง index ไม่ช่วย |
| ซ้ำซ้อนกับ index ที่มีอยู่ | เช่น มี `(a, b)` แล้ว ไม่ต้องสร้าง `(a)` เพิ่ม เพราะตัวแรกใช้ค้นด้วย `a` ได้อยู่แล้ว (prefix) |
| สร้างเผื่อ ๆ ทุกคอลัมน์ | "เผื่อได้ใช้" คือกับดัก — เปลืองดิสก์ เขียนช้า planner สับสน |
 
---
 
## 6. จังหวะเวลาในการสร้าง
 
| สถานการณ์ | คำแนะนำ |
|---|---|
| ตารางเล็ก–กลาง | สร้างตอนไหนก็ได้ เสร็จในไม่กี่วินาที |
| ตารางใหญ่บน **Production** | ใช้ `create index concurrently` เพื่อไม่ lock ตาราง |
| ช่วงเวลาที่เหมาะ | นอกเวลาทำการ หรือช่วง traffic ต่ำ |
 
```sql
-- แบบปลอดภัยสำหรับ production (ไม่บล็อก insert/update ระหว่างสร้าง)
create index concurrently idx_stmt_act_end
  on cardlite.statement_summary (act_idn_sky, prd_end_dte desc);
```
 
> ⚠️ ในโปรเจกต์จริง การเพิ่ม index บน UAT/Production ถือเป็น **DB change** ควรผ่าน DBA / change request ตามขั้นตอน ไม่ควรรันเองตรง ๆ บน Production
 
---
 
## 7. วิธีแก้ไขเมื่อดิสก์โตจาก Index
 
### 7.1 เช็คขนาดจริงก่อน
 
```sql
-- ขนาดตาราง + index ทั้งหมด
select
    pg_size_pretty(pg_relation_size('cardlite.statement_summary'))   as table_size,
    pg_size_pretty(pg_indexes_size('cardlite.statement_summary'))    as all_indexes_size,
    pg_size_pretty(pg_total_relation_size('cardlite.statement_summary')) as total;
 
-- ขนาดแยกราย index
select indexrelname,
       pg_size_pretty(pg_relation_size(indexrelid)) as index_size
from pg_stat_user_indexes
where relname = 'statement_summary';
```
 
### 7.2 ลบ index ที่ไม่ถูกใช้ (ได้ผลเร็วสุด)
 
```sql
-- หา index ที่ไม่เคยถูกใช้ (idx_scan = 0)
select indexrelname,
       idx_scan as times_used,
       pg_size_pretty(pg_relation_size(indexrelid)) as size
from pg_stat_user_indexes
where relname = 'statement_summary'
order by idx_scan asc;
```
 
ตัวไหน `times_used = 0` มานาน (และไม่ใช่ unique/PK constraint) → พิจารณา `drop index`
 
> ⚠️ สถิติ `idx_scan` นับตั้งแต่ reset ล่าสุด — ต้องแน่ใจว่าระบบรันมานานพอ ครอบคลุมงาน batch สิ้นเดือนด้วย ก่อนตัดสินว่า "ไม่ถูกใช้"
 
### 7.3 ลบ index ซ้ำซ้อน
 
มี `(act_idn_sky)` เดี่ยว และ `(act_idn_sky, prd_end_dte)` อยู่คู่กัน → ตัวแรกซ้ำซ้อน drop ได้เลย
 
### 7.4 REINDEX เมื่อ index บวม (bloat)
 
```sql
-- Postgres 12+ แบบไม่ lock เหมาะกับ production
reindex index concurrently cardlite.idx_stmt_act_end;
```
 
### 7.5 Partial Index — สร้างเฉพาะส่วนที่ใช้
 
ถ้ารายงานดึงแค่ข้อมูลช่วงหลัง ๆ ไม่ต้อง index ทั้งประวัติศาสตร์:
 
```sql
create index idx_stmt_act_end_recent
  on cardlite.statement_summary (act_idn_sky, prd_end_dte desc)
  where prd_end_dte >= date '2025-01-01';
```
 
Index เล็กลงมากและเร็วขึ้น — แต่ query ต้องมีเงื่อนไขสอดคล้องกับ `where` ของ index จึงจะถูกใช้
 
### 7.6 เก็บกวาดระดับตาราง
 
```sql
vacuum analyze cardlite.statement_summary;
```
 
ปกติ autovacuum ทำให้อยู่แล้ว แต่ควรเช็คว่าทำงานสม่ำเสมอ
 
---
 
## 8. Checklist ก่อนสร้าง Index
 
```sql
-- 1. ดูว่า query ช้าตรงไหน
explain analyze <query>;
 
-- 2. ดูว่าตารางมี index อะไรอยู่แล้ว (กันสร้างซ้ำ)
select indexname, indexdef
from pg_indexes
where tablename = 'statement_summary';
 
-- 3. ดูขนาดตาราง (เล็กมากก็ไม่ต้องสร้าง)
select count(*) from cardlite.statement_summary;
```
 
**หลังสร้าง:** รัน `explain analyze` ซ้ำ เทียบก่อน–หลัง
- เวลาเร็วขึ้น + เห็น Index Scan → สำเร็จ
- เวลาไม่ต่าง → index ไม่ถูกใช้ พิจารณา drop ทิ้ง
---
 
## 9. สิ่งที่ควรรู้เพิ่มเติม
 
### 9.1 ลำดับคอลัมน์ใน composite index สำคัญ
 
Index `(act_idn_sky, prd_end_dte)` ใช้ค้นด้วย:
- ✅ `act_idn_sky` อย่างเดียว (prefix)
- ✅ `act_idn_sky + prd_end_dte` คู่กัน
- ❌ `prd_end_dte` อย่างเดียว — ข้าม prefix ไม่ได้
จึงควรวางคอลัมน์ที่ถูกใช้กรอง/join บ่อยที่สุดไว้หน้าสุด
 
### 9.2 การห่อคอลัมน์ด้วยฟังก์ชันทำให้ index ไม่ถูกใช้
 
```sql
-- ❌ index บน prd_end_dte จะไม่ถูกใช้
where date_trunc('month', prd_end_dte) = ...
 
-- ✅ ใช้ index ได้ปกติ
where prd_end_dte <= cast(:asOfDate as date)
```
 
ถ้าจำเป็นต้องใช้ฟังก์ชัน ให้สร้าง **functional index** เช่น
`create index on t (date_trunc('month', prd_end_dte));`
 
> นี่เป็นอีกเหตุผลที่ solution แบบเทียบวันที่ตรง ๆ (`prd_end_dte <= asOfDate`) ดีกว่าแบบ `date_trunc` — เพราะใช้ index ได้เต็มที่
 
### 9.3 ORDER BY + LIMIT ทำงานร่วมกับ index ได้ดีมาก
 
```sql
order by prd_end_dte desc
limit 1
```
 
ถ้ามี index เรียงลำดับไว้ตรงกัน database จะ "หยิบแถวบนสุด" ผ่าน index ทันที ไม่ต้อง sort — เหมาะกับงานหารอบล่าสุด (Top-N-per-group) โดยเฉพาะเมื่อใช้คู่กับ `LEFT JOIN LATERAL`
 
### 9.4 ดูว่า index ถูกใช้จริงไหมด้วย EXPLAIN
 
| สิ่งที่เห็นใน plan | ความหมาย |
|---|---|
| `Index Scan` / `Index Only Scan` | ใช้ index อยู่ ✅ |
| `Bitmap Index Scan` | ใช้ index แบบรวมกลุ่มแถวก่อนอ่าน ✅ |
| `Seq Scan` บนตารางใหญ่ | ไม่ใช้ index — อาจขาด index หรือเงื่อนไขห่อฟังก์ชันไว้ ⚠️ |
 
### 9.5 ตรวจสุขภาพ index เป็นระยะ
 
ทุก 3–6 เดือน (หรือหลัง release ใหญ่) ควรรัน query ในข้อ 7.2 เพื่อหา index ที่ไม่ถูกใช้ และเช็ค bloat — ทำให้ดิสก์โต "แบบมีเหตุผล"
 
---
 
## 10. สรุปตารางปัญหา–ทางแก้
 
| ปัญหา | ทางแก้ |
|---|---|
| Query ช้า เห็น Seq Scan บนตารางใหญ่ | สร้าง index บนคอลัมน์ที่ใช้ WHERE/JOIN/ORDER BY |
| Index ไม่เคยถูกใช้ | drop ทิ้ง (เช็คจาก `pg_stat_user_indexes.idx_scan`) |
| Index ซ้ำซ้อน | drop ตัวที่ถูก composite index ครอบคลุมอยู่แล้ว |
| Index บวมจาก update/delete เยอะ | `reindex concurrently` |
| Index ใหญ่เกินจำเป็น | เปลี่ยนเป็น partial index |
| Insert ช้าลงหลังมี index เยอะ | ทบทวนและลด index เหลือเท่าที่ใช้จริง |
| กังวลดิสก์ล่วงหน้า | เช็คขนาดจริงก่อน (`pg_indexes_size`) มักเล็กกว่าที่คิด |
 
---
 
## 11. เทียบคำสั่ง Index ข้ามยี่ห้อ (Postgres / SQL Server / MySQL / Oracle)
 
หลักการทั้งหมดในเอกสารนี้ใช้ได้ทุกยี่ห้อ (เพราะพื้นฐานคือ B-tree เหมือนกัน) แต่ **คำสั่ง** ต่างกันตามตารางนี้:
 
| เรื่อง | PostgreSQL | SQL Server | MySQL (InnoDB) | Oracle |
|---|---|---|---|---|
| สร้าง index พื้นฐาน | `create index i on t (a, b)` | เหมือนกัน | เหมือนกัน | เหมือนกัน |
| สร้างแบบไม่ lock ตาราง | `create index concurrently` | `create index ... with (online = on)` (Enterprise เท่านั้น) | `alter table ... add index ..., algorithm=inplace, lock=none` | `create index ... online` |
| Partial / filtered index | ✅ `create index ... where prd_end_dte >= '2025-01-01'` | ✅ filtered index — `create index ... where col = 1` | ❌ ไม่มี | ❌ ไม่มีตรง ๆ (เลี่ยงด้วย function-based index ให้แถวที่ไม่ต้องการเป็น NULL) |
| Functional / expression index | ✅ `create index on t (date_trunc('month', d))` | ใช้ computed column + index แทน | ✅ (8.0+) `create index on t ((year(d)))` | ✅ function-based index |
| ดู execution plan | `explain analyze <query>` | เปิด "Include Actual Execution Plan" หรือ `set statistics profile on` | `explain analyze <query>` (8.0.18+) | `explain plan for <query>` แล้ว `select * from table(dbms_xplan.display)` |
| หา index ที่ไม่ถูกใช้ | `pg_stat_user_indexes` (ดู `idx_scan`) | `sys.dm_db_index_usage_stats` | `sys.schema_unused_indexes` | `alter index ... monitoring usage` แล้วดู `v$object_usage` |
| ดูขนาด index | `pg_relation_size(indexrelid)` | `sys.dm_db_partition_stats` / sp_spaceused | `information_schema.tables` (index_length) | `dba_segments` |
| แก้ index บวม (rebuild) | `reindex concurrently` | `alter index ... rebuild` (with online=on ได้) | `optimize table t` | `alter index ... rebuild online` |
| ลบ index | `drop index i` | `drop index i on t` (ต้องระบุตาราง) | `drop index i on t` | `drop index i` |
 
> 💡 ข้อสังเกต: คำสั่ง **สร้าง** เหมือนกันเกือบหมด ที่ต่างจริง ๆ คือคำสั่ง **บริหารจัดการ** (monitor, rebuild, online option)
 
---
 
## 12. Clustered Index — แนวคิดสำคัญที่ต่างกันมากระหว่างยี่ห้อ
 
นี่คือความต่างเชิงสถาปัตยกรรมที่ควรรู้ที่สุดถ้าทำงานข้าม DB:
 
| | ความหมาย |
|---|---|
| **Clustered Index** | ตัวข้อมูลในตาราง "ถูกจัดเรียงจริง ๆ บนดิสก์" ตามลำดับของ index — มีได้ **ตัวเดียว** ต่อตาราง |
| **Non-clustered (Secondary) Index** | สารบัญแยกต่างหาก ชี้กลับไปหาแถวจริง — มีได้หลายตัว |
 
**พฤติกรรมแต่ละยี่ห้อ:**
 
- **SQL Server:** Primary key เป็น clustered index โดย default — ตารางทั้งตารางเรียงตาม PK การเลือกว่าคอลัมน์ไหนเป็น clustered มีผลต่อ performance มาก
- **MySQL (InnoDB):** เหมือน SQL Server — PK คือ clustered index เสมอ และ secondary index ทุกตัว "พ่วง PK" ไว้ข้างใน (PK ยาว → index ทุกตัวใหญ่ตาม)
- **PostgreSQL:** **ไม่มี clustered index ถาวร** — ทุก index เป็น secondary หมด ตารางเก็บแบบ heap (ไม่เรียง) มีคำสั่ง `cluster t using i` ที่จัดเรียงครั้งเดียว แต่ข้อมูลใหม่ที่ insert เข้ามาจะไม่เรียงตาม ต้องรันซ้ำเอง
- **Oracle:** ตารางปกติเป็น heap เหมือน Postgres แต่มีตัวเลือก Index-Organized Table (IOT) ถ้าอยากให้เรียงตาม PK
**ผลในทางปฏิบัติ:** ถ้าย้ายจาก Postgres → SQL Server/MySQL ต้องคิดเรื่อง "เลือกคอลัมน์อะไรเป็น clustered/PK" เพิ่ม เพราะกระทบทั้ง performance การอ่านช่วง (range scan) และขนาดของ index อื่น ๆ ทั้งหมด
 
---
 
## 13. ชนิดของ Index ที่ควรรู้จัก (นอกจาก B-tree)
 
B-tree คือ default และตอบโจทย์ 95% ของงาน แต่บางงานมี index เฉพาะทางที่ดีกว่า:
 
| ชนิด | เหมาะกับ | มีใน |
|---|---|---|
| **B-tree** (default) | เปรียบเทียบ `= < > between`, ORDER BY — งานทั่วไปทั้งหมด | ทุกยี่ห้อ |
| **Hash** | เทียบ `=` อย่างเดียว (เร็วกว่า B-tree เล็กน้อยเฉพาะกรณีนี้) | Postgres, MySQL (memory engine) |
| **GIN** | ค้นใน array, JSONB, full-text search | Postgres |
| **GiST / SP-GiST** | ข้อมูลเชิงพื้นที่ (geometry), ช่วงเวลา (range) | Postgres |
| **BRIN** | ตารางใหญ่มาก ๆ ที่ข้อมูลเรียงตามเวลาอยู่แล้ว (เช่น log, transaction รายวัน) — index เล็กกว่า B-tree หลายร้อยเท่า | Postgres |
| **Full-text** | ค้นหาข้อความ | SQL Server, MySQL, Oracle (Postgres ใช้ GIN) |
| **Columnstore** | งาน analytics/รายงานที่ scan คอลัมน์เดียวจำนวนแถวมหาศาล | SQL Server, Oracle |
 
> 💡 สำหรับตารางแนว `transaction_summary` ที่โตตามเวลาและ query ด้วยช่วงวันที่ — **BRIN บน `ptg_dte`** เป็นตัวเลือกที่น่าสนใจใน Postgres: เล็กมาก สร้างเร็ว เหมาะกับข้อมูลที่ insert เรียงตามวัน
 
---
 
## 14. Pattern "หาแถวล่าสุดต่อกลุ่ม" (Top-N-per-group) กับ Index
 
Pattern ที่ใช้แก้บั๊กรายงาน CardLite — สรุปไว้เพื่ออ้างอิง เพราะเป็นโจทย์ที่เจอบ่อยมากในงานรายงาน:
 
**โจทย์:** หา statement รอบปิดล่าสุด (`prd_end_dte` มากสุด) ของแต่ละ account ที่ไม่เกิน asOfDate
 
| วิธี | เขียนยังไง | ใช้ได้ที่ไหน | ความเร็ว (เมื่อมี index) |
|---|---|---|---|
| `distinct on` | `select distinct on (act_idn_sky) * ... order by act_idn_sky, prd_end_dte desc` | Postgres เท่านั้น | เร็ว |
| `row_number()` | `row_number() over (partition by act_idn_sky order by prd_end_dte desc)` แล้วกรอง `rn = 1` | **ทุกยี่ห้อ** (ANSI) | เร็ว–กลาง (sort ทั้ง partition) |
| `left join lateral` + `limit 1` | subquery เห็นค่าแถวซ้าย หยิบแถวบนสุดผ่าน index | Postgres (SQL Server ใช้ `outer apply`) | **เร็วสุด** |
 
**Index ที่รองรับ pattern นี้:** `(act_idn_sky, prd_end_dte desc)` — จัดกลุ่ม + เรียงในกลุ่มไว้ล่วงหน้า ทำให้ทุกวิธีข้างบน "เด้งหยิบแถวบนสุด" ได้โดยไม่ต้อง sort ก้อนใหญ่
 
**เทียบ Postgres ↔ SQL Server:**
 
```sql
-- PostgreSQL
left join lateral (
    select * from cardlite.statement_summary ss
    where ss.act_idn_sky = a.act_idn_sky
      and ss.prd_end_dte <= cast(:asOfDate as date)
    order by ss.prd_end_dte desc
    limit 1
) ss on true
 
-- SQL Server (ความหมายเดียวกันเป๊ะ)
outer apply (
    select top 1 * from cardlite.statement_summary ss
    where ss.act_idn_sky = a.act_idn_sky
      and ss.prd_end_dte <= cast(@asOfDate as date)
    order by ss.prd_end_dte desc
) ss
```
 
> จำคู่นี้ไว้: `lateral ... limit 1` (Postgres) = `outer apply ... top 1` (SQL Server)
 
---
 
## 15. บทเรียนจากเคสจริง (CardLite Statement Report)
 
บันทึกไว้เป็นกรณีศึกษา:
 
1. **อาการ:** รายงานแสดง account ไม่ครบ — account ที่รอบบิลเริ่มปลายเดือนหายไป
2. **Root cause:** query เลือกรอบ statement ด้วย `asOfDate − interval '1 month'` แล้วเทียบ "วันที่" กับช่วงรอบบิล → ผลผูกกับวันโหลด ไม่ใช่รอบบิลจริง แถมการลบเดือนยังเพี้ยนในเดือน 30/31 วัน
3. **Solution:** เปลี่ยนเป็นเลือก "รอบที่ปิดล่าสุด ณ asOfDate" ด้วย `distinct on (act_idn_sky) ... order by prd_end_dte desc` — เทียบวันที่ตรง ๆ ไม่ลบเดือน ตัดปัญหา 30/31 วันไปในตัว
4. **บทเรียนเรื่อง index:** เงื่อนไข `prd_end_dte <= asOfDate` (ไม่ห่อฟังก์ชัน) ใช้ index ได้เต็มที่ ต่างจากแนวทาง `date_trunc('month', ...)` ที่เคยพิจารณา ซึ่งจะทำให้ index ใช้ไม่ได้
5. **จุดตัดสินใจที่ต้องยืนยันกับ business:** `<` vs `<=` — ถ้า asOfDate ตรงวันปิดรอบพอดี จะนับรอบนั้นหรือไม่ (test case บ่งชี้ว่าต้องเป็น `<=`)
