# Tutorial 6 worksheet: serving patterns

**Name / ชื่อ:** Thukdanai Thaothawin

**Student ID / รหัสนิสิต:** 6631319121

This is **individual work**: each student completes and submits their own copy, in their own fork.

*งานนี้เป็น**งานเดี่ยว** นิสิตแต่ละคนทำและส่งงานของตัวเอง โดยใช้ forked repository ของตัวเอง*

Copy result rows from `results/runs.csv` (or the load test's output) into the tables. Keep answers short: one to three sentences each. Answer in Thai or English.

ตอนที่ตอบ ให้นิสิตคัดลอกผลจาก `results/runs.csv` (หรือจาก output ของ load test) ลงในตาราง ตอบสั้นๆ ข้อละ 1–3 ประโยค จะตอบเป็นภาษาไทยหรือภาษาอังกฤษก็ได้

## Task 0: Baseline (copy the baseline results from `results/runs.csv` here / copy ผลลัพธ์ของ baseline จาก `results/runs.csv` มาวางที่นี่)

| Run | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|
| t0-large | 500 | 606 | 93.5% | 100% |
| t0-small | 65 | 170 | 72.5% | 0% |

**Q0.** What does the large model buy, and what does it cost?

*large model ให้อะไรเพิ่ม และต้องแลกด้วยอะไร*

> large model ให้ accuracy สูงกว่า (93.5% เทียบกับ 72.5%, +21 จุด) แต่แลกด้วยความช้า (p50 500 ms เทียบกับ 65 ms) และทุก request เป็น call ที่เสียเงิน (large calls 100% เทียบกับ 0%)

## Task 1: Fallback prediction (copy the Task 1 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 1 จาก `results/runs.csv` มาวางที่นี่)

| Run | run time (s) | p50 ms | p95 ms | errors | pending | accuracy |
|---|---|---|---|---|---|---|
| t1-before | ไม่ได้บันทึก | 265 | 475 | 20 | 0 | 92.8% |
| t1-after | ไม่ได้บันทึก | 400 | 1038 | 0 | 28 | 93.0% |

**Q1.1** In `t1-before`, p50 can look *better* than the baseline. Why is p50 misleading here? Which column tells the real story?

*ใน `t1-before` ค่า p50 อาจดู "ดีกว่า" baseline ทำไม p50 จึงทำให้เข้าใจผิดในกรณีนี้ และ column ไหนบอกความจริง*

> ใน `t1-before` p50 = 265 ms ต่ำกว่า baseline (500 ms) เพราะ call ที่ค้างไม่ได้แย่งกำลังของ large model ทำให้ request ปกติที่เหลือเร็วขึ้น ส่วน request ที่ค้างไม่ถูกนับใน latency (นับเฉพาะที่สำเร็จ) column `errors` = 20 (10% ของ request ล้มเหลว) บอกความจริงที่ p50 ซ่อนไว้

**Q1.2** The hang affects only 10% of large-model calls. Why is it worse for the system than 10% of reviews being slow? Think about what a waiting request holds on to in the review service and the gateway.

*การค้างเกิดกับ calls ไปยัง large model แค่ 10% ทำไมจึงส่งผลต่อระบบแย่กว่าการที่รีวิว 10% ช้าลงเฉยๆ ลองคิดว่า request ที่รออยู่ถือทรัพยากรอะไรไว้ใน review service และ gateway*

> request ที่ค้างถือ connection, worker/task และ memory ไว้ใน review service และ gateway นานถึง 20 วินาที (`HANG_SECONDS`) ทั้งที่ request ปกติใช้แค่ ~0.2 วินาที request ที่ค้างจึงสะสมได้ ถ้า connection และ worker/task ใน review service กับ gateway เต็มไปด้วย request ที่ค้าง request ปกติก็ไม่มีที่ว่างให้เข้า ผลกระทบจึงลามไปทั้งระบบ ไม่ใช่แค่ 10% ที่ช้า

**Q1.3** Pending reviews have no label. Suggest what should happen to them, and name the topology from the lecture that does it.

*รีวิวที่ pending ยังไม่มี label ควรจัดการกับรีวิวเหล่านี้อย่างไร และ topology ใดจาก lecture ที่ทำแบบนั้น*

> รีวิวที่ pending ควรถูกเก็บไว้ แล้วให้ batch job หรือ worker มาติด label ทีหลังเมื่อ large model ว่าง/กลับมาปกติ topology ที่ทำแบบนี้คือ Asynchronous / batch (T4) ที่ใช้คิวหรือ message broker (W10)

**Q1.4** We chose a 1-second timeout. What goes wrong if it is 0.1 s? And if it is 10 s?

*เราเลือก timeout 1 วินาที จะเกิดปัญหาอะไรถ้าตั้งเป็น 0.1 วินาที และถ้าตั้งเป็น 10 วินาที*

> ถ้า 0.1 s: large model ใช้เวลาปกติ ~200 ms เกือบทุก call จะ timeout กลายเป็น pending ทั้งที่ระบบปกติดี และยังเสียค่า large call ไปแล้ว ถ้า 10 s: request ที่ค้างยังถือ resource นานเกือบเท่าไม่มี timeout และเกินเป้า QR1/QR2 (ตอบภายใน 1 s) ผู้ใช้ยังรอนานมาก

## Task 2: Cascade (copy the Task 2 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 2 จาก `results/runs.csv` มาวางที่นี่)

| Run | threshold | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|---|
| t2-0.6 | 0.6 | 35 | 551 | 91.5% | 38% |
| t2-0.7 | 0.7 | 396 | 601 | 93.0% | 53% |
| t2-0.8 | 0.8 | 438 | 627 | 93.5% | 64% |
| t2-0.9 | 0.9 | 464 | 607 | 93.5% | 77% |

**Q2.1** Which threshold would you deploy for CampusEats, and why? Name the quality you give up.

*จะเลือก threshold ใดสำหรับ deploy ใน CampusEats เพราะอะไร และต้องยอมเสีย quality ใดไป*

> จะเลือก 0.7: accuracy 93.0% ใกล้เคียง `t0-large` (93.5%) และผ่าน QR3 (>= 90%) แต่เรียก large model แค่ 53% (ประหยัด ~47%) threshold 0.6 ถูกกว่า (38%) แต่ accuracy 91.5% ใกล้เส้น 90% เกินไป ส่วน 0.8/0.9 ได้ accuracy เพิ่มแค่ 0.5 จุด แต่ใช้ large call เพิ่มเป็น 64%/77% สิ่งที่ยอมเสียคือ accuracy เล็กน้อย (0.5 จุด) เพื่อแลกกับค่าใช้จ่ายที่ลดลง

**Q2.2** Look at p50 and p95 for your threshold. Why does the cascade help p50 much more than p95?

*ดูค่า p50 และ p95 ของ threshold ที่เลือก ทำไม cascade จึงช่วย p50 ได้มากกว่า p95 มาก*

> ที่ 0.7 p50 ลดจาก 500 เหลือ 396 ms แต่ p95 แทบไม่เปลี่ยน (606 -> 601 ms) เพราะ p95 คือ 5% ที่ช้าที่สุด ซึ่งคือ request ที่ถูกส่งต่อไป large model เสมอ (และยังต้องผ่าน small model ก่อน) cascade ทำให้เร็วขึ้นได้เฉพาะ request ที่ small model ตอบเอง p50 จะลดมากก็ต่อเมื่อ small model ตอบเกินครึ่ง (เช่น 0.6 ที่ p50 เหลือ 35 ms)

**Q2.3** The cascade trusts the small model's confidence. Give one situation where that trust is misplaced, and its effect on users.

*cascade เชื่อ confidence ของ small model ยกตัวอย่างหนึ่งสถานการณ์ที่ไม่ควรเชื่อ และผลที่เกิดกับผู้ใช้*

> small model อ่านประโยคปฏิเสธหรือ "but" ผิด เช่น "not bad" หรือ "nice place but the food was awful" อาจให้ label ผิดด้วย confidence สูง ซึ่งจะไม่ถูกส่งต่อไป large model ผลคือผู้ใช้/เจ้าของร้านเห็น label ผิดอย่างมั่นใจ เช่น ได้รับ alert ว่าลูกค้าไม่พอใจทั้งที่พอใจ

## Task 3: Cached predictions (copy the Task 3 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 3 จาก `results/runs.csv` มาวางที่นี่)

| Run | p50 ms | p95 ms | accuracy | large calls | cache hits |
|---|---|---|---|---|---|
| t3-cache | 36 | 526 | 93.0% | 38% | 35% |

**Q3.1** Which reviews benefit from the cache, and which never will?

*รีวิวแบบไหนได้ประโยชน์จาก cache และแบบไหนไม่มีวันได้ประโยชน์*

> รีวิวที่ซ้ำกัน เช่น ข้อความสั้นที่พบบ่อยอย่าง "Great food!" (หลัง normalise ตัวพิมพ์เล็ก/ช่องว่าง) ได้ประโยชน์จาก cache (`t3-cache` cache hits 35% และ large calls ลดจาก 53% เหลือ 38% เมื่อเทียบ `t2-0.7`) ส่วนรีวิวยาวที่เขียนเฉพาะตัวและไม่เคยซ้ำจะไม่ได้ประโยชน์เลย

**Q3.2** Next week you deploy a better large model. What must happen to the cache, and how would you design for it?

*สัปดาห์หน้าจะ deploy large model ตัวใหม่ที่ดีกว่าเดิม ต้องทำอะไรกับ cache และจะออกแบบรองรับเรื่องนี้อย่างไร*

> ต้องล้าง (invalidate) cache เพราะคำตอบเก่าเกิดจากโมเดลเก่า ควรออกแบบให้ใส่ version ของโมเดลเป็นส่วนหนึ่งของ cache key (เช่น `model_version + ข้อความที่ normalise แล้ว`) และ/หรือตั้ง TTL เมื่อ deploy โมเดลใหม่ entry เก่าจะ miss เองโดยไม่ต้องลบมือ และ rollback กลับโมเดลเดิมได้

## Debrief

**Q4.** Name one thing you learned from these measurements that the lecture did not cover.

*ยกตัวอย่างหนึ่งสิ่งที่ได้เรียนรู้จากการวัดผลครั้งนี้ ที่ไม่มีพูดถึงใน lecture*

> ได้เห็นว่าเมื่อใส่ timeout แล้ว p95 ไปอยู่ใกล้ค่า timeout เอง: ใน `t1-after` p95 = 1038 ms เทียบกับ timeout 1 s เพราะ request ที่ pending ถูกตัดที่ 1 s พอดี จึงมีเวลา ≥1,000 ms เสมอ และมี 28 จาก 200 รายการ (14%) อยู่ในกลุ่มนั้น ซึ่งมากกว่า 5% ที่ p95 ใช้อยู่แล้ว ดังนั้น p95 ที่สูงนี้สะท้อนสัดส่วน pending ไม่ใช่ว่า call ปกติช้าลง (call ที่ค้างก็ไม่ได้ถือ slot ทั้ง 4 ของ large model ตาม `models.py:65-69`) lecture ไม่ได้บอกว่าต้องดูสัดส่วน pending ควบคู่กับ p95 เมื่อมี timeout นอกจากนี้ pending ที่วัดได้คือ 28 รายการ ส่วนที่ hang rate 10% จาก 200 รายการคาดไว้ ~20
