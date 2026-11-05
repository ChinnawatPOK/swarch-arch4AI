# Tutorial 6 worksheet: serving patterns

**Name / ชื่อ:** นาย ชินนวัตร แก้วฉิม

**Student ID / รหัสนิสิต:** 6870062321

This is **individual work**: each student completes and submits their own copy, in their own fork.

*งานนี้เป็น**งานเดี่ยว** นิสิตแต่ละคนทำและส่งงานของตัวเอง โดยใช้ forked repository ของตัวเอง*

Copy result rows from `results/runs.csv` (or the load test's output) into the tables. Keep answers short: one to three sentences each. Answer in Thai or English.

ตอนที่ตอบ ให้นิสิตคัดลอกผลจาก `results/runs.csv` (หรือจาก output ของ load test) ลงในตาราง ตอบสั้นๆ ข้อละ 1–3 ประโยค จะตอบเป็นภาษาไทยหรือภาษาอังกฤษก็ได้

## Task 0: Baseline (copy the baseline results from `results/runs.csv` here / copy ผลลัพธ์ของ baseline จาก `results/runs.csv` มาวางที่นี่)

| Run | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|
| t0-large | 491 | 584 | 93.5% | 100% |
| t0-small | 9 | 20 | 72.5% | 0% |

**Q0.** What does the large model buy, and what does it cost?
> ANS: Large model ให้ความแม่นยำเพิ่มขึ้นจาก 72.5% เป็น 93.5% แต่ต้องแลกกับ latency ที่แย่ลง (p50 จาก 9 ms เป็น 491 ms) และค่าใช้จ่ายที่เพิ่มขึ้น เพราะ large calls เป็น 100% คือทุกรีวิวต้องเสียเงินเรียก large model
*large model ให้อะไรเพิ่ม และต้องแลกด้วยอะไร*

>

## Task 1: Fallback prediction (copy the Task 1 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 1 จาก `results/runs.csv` มาวางที่นี่)

| Run | run time (s) | p50 ms | p95 ms | errors | pending | accuracy |
|---|---|---|---|---|---|---|
| t1-before | 26.5 | 248 | 426 | 20 | 0 | 92.8% |
| t1-after | 9.3 | 390 | 1010 | 0 | 27 | 93.1% |

**Q1.1** In `t1-before`, p50 can look *better* than the baseline. Why is p50 misleading here? Which column tells the real story?

*ใน `t1-before` ค่า p50 อาจดู "ดีกว่า" baseline ทำไม p50 จึงทำให้เข้าใจผิดในกรณีนี้ และ column ไหนบอกความจริง*

> ANS: p50 ของ `t1-before` (248 ms) ดูดีกว่า baseline (491 ms) จริง แต่ทำให้เข้าใจผิด เพราะไม่นับ request ที่แย่ที่สุด และ request ที่เหลือได้คิวสั้นลงเพราะระบบกำลังติดขัด column ที่บอกความจริงคือ `errors` ซึ่งมีถึง 20 รายการ

**Q1.2** The hang affects only 10% of large-model calls. Why is it worse for the system than 10% of reviews being slow? Think about what a waiting request holds on to in the review service and the gateway.

*การค้างเกิดกับ calls ไปยัง large model แค่ 10% ทำไมจึงส่งผลต่อระบบแย่กว่าการที่รีวิว 10% ช้าลงเฉยๆ ลองคิดว่า request ที่รออยู่ถือทรัพยากรอะไรไว้ใน review service และ gateway*

> ANS: request ที่ค้างจะถือทรัพยากร (เช่น connection) ของ review service และ gateway ไว้ระหว่างรอ ทำให้แย่ง resource ของคนอื่น และ request อื่นที่ไม่ได้ค้างเข้าใช้งานไม่ได้ จึงไม่ได้ช้าแค่ 10% ของรีวิว ดูได้จาก run time ที่เพิ่มจาก 9.9 s (`t0-large`) เป็น 26.5 s (`t1-before`)

**Q1.3** Pending reviews have no label. Suggest what should happen to them, and name the topology from the lecture that does it.

*รีวิวที่ pending ยังไม่มี label ควรจัดการกับรีวิวเหล่านี้อย่างไร และ topology ใดจาก lecture ที่ทำแบบนั้น*

> ANS: เก็บรีวิวที่ pending ไว้ก่อน แล้วให้ batch job มาประมวลผลรีวิวเหล่านั้นเป็นรอบๆ เพื่อติด label ทีหลัง ซึ่งตรงกับ topology แบบ Asynchronous / batch (T4)

**Q1.4** We chose a 1-second timeout. What goes wrong if it is 0.1 s? And if it is 10 s?

*เราเลือก timeout 1 วินาที จะเกิดปัญหาอะไรถ้าตั้งเป็น 0.1 วินาที และถ้าตั้งเป็น 10 วินาที*

> ANS: ถ้าตั้ง 0.1 s ส่วนมากจะ timeout เพราะ large call ปกติใช้เวลาประมาณ 200 ms รีวิวส่วนใหญ่จึงกลายเป็น pending ถ้าตั้ง 10 s ระบบน่าจะกลับไปเป็นแบบ `t1-before` คือ request ที่ค้างรอนานจน client หมดเวลาและเกิด errors และผิด QR2 ที่ให้รอได้ไม่เกิน 1 วินาที

## Task 2: Cascade (copy the Task 2 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 2 จาก `results/runs.csv` มาวางที่นี่)

| Run | threshold | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|---|
| t2-0.6 | 0.6 | 6 | 545 | 91.5% | 38% |
| t2-0.7 | 0.7 | 359 | 573 | 93.0% | 53% |
| t2-0.8 | 0.8 | 440 | 560 | 93.5% | 64% |
| t2-0.9 | 0.9 | 480 | 588 | 93.5% | 77% |

**Q2.1** Which threshold would you deploy for CampusEats, and why? Name the quality you give up.

*จะเลือก threshold ใดสำหรับ deploy ใน CampusEats เพราะอะไร และต้องยอมเสีย quality ใดไป*

> ANS: เลือก threshold 0.8 เพราะ accuracy เท่ากับ 0.9 (93.5%) และยังเรียก large model น้อยกว่า `t0-large` (64% เทียบกับ 100%) โดยยอมเสีย cost เพิ่มขึ้น 11 จุดเมื่อเทียบกับ 0.7 (64% เทียบกับ 53%)

**Q2.2** Look at p50 and p95 for your threshold. Why does the cascade help p50 much more than p95?

*ดูค่า p50 และ p95 ของ threshold ที่เลือก ทำไม cascade จึงช่วย p50 ได้มากกว่า p95 มาก*

> ANS: ที่ threshold 0.8 p50 ลดจาก 491 เหลือ 440 ms และ p95 ลดจาก 584 เหลือ 560 ms แค่นิดเดียว เพราะรีวิว 64% ยังไป large model ทั้ง median และ tail จึงยังเป็น call ของ large model ผลต่อ p50 จะเห็นชัดเมื่อ large calls ต่ำกว่า 50% เช่น `t2-0.6` ที่ p50 เหลือ 6 ms แต่ p95 ยังเป็น 545 ms เพราะ 5% ที่ช้าที่สุดยังเป็นรีวิวยากที่ต้องรอ large model เสมอ

**Q2.3** The cascade trusts the small model's confidence. Give one situation where that trust is misplaced, and its effect on users.

*cascade เชื่อ confidence ของ small model ยกตัวอย่างหนึ่งสถานการณ์ที่ไม่ควรเชื่อ และผลที่เกิดกับผู้ใช้*

> ANS: small model ไม่เห็นคำปฏิเสธเป็นวลี เช่น "Honestly the salad is not cold." มันตอบ negative ด้วย confidence 0.70 ซึ่งผ่าน threshold 0.7 จึงไม่ถูกส่งต่อให้ large model ผลคือร้านถูกนับรีวิวเชิงลบเกินจริง และเจ้าของร้านอาจได้ alert ผิดโดยระบบไม่รู้ตัวเลยว่าตอบผิด

## Task 3: Cached predictions (copy the Task 3 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 3 จาก `results/runs.csv` มาวางที่นี่)

| Run | p50 ms | p95 ms | accuracy | large calls | cache hits |
|---|---|---|---|---|---|
| t3-cache | 6 | 554 | 93.5% | 40% | 34% |

**Q3.1** Which reviews benefit from the cache, and which never will?

*รีวิวแบบไหนได้ประโยชน์จาก cache และแบบไหนไม่มีวันได้ประโยชน์*

> ANS: รีวิวสั้นที่ซ้ำกันบ่อย เช่น "Delicious!" หรือ "Too salty" ได้ประโยชน์ที่สุด ส่วนรีวิวยาวที่เขียนไม่เหมือนใครจะไม่มีวัน hit เพราะ key ต้องตรงกันทั้งข้อความ ใน `t3-cache` hit 34% ทำให้ large calls ลดจาก 64% เหลือ 40% และ p50 ลดจาก 440 เหลือ 6 ms โดย accuracy เท่าเดิม (93.5%)

**Q3.2** Next week you deploy a better large model. What must happen to the cache, and how would you design for it?

*สัปดาห์หน้าจะ deploy large model ตัวใหม่ที่ดีกว่าเดิม ต้องทำอะไรกับ cache และจะออกแบบรองรับเรื่องนี้อย่างไร*

> ANS: ต้องเลิกใช้คำตอบเก่าใน cache เพราะยังเป็นผลของโมเดลเดิม ถ้าไม่ทำรีวิวที่ซ้ำจะไม่ได้ประโยชน์จากโมเดลใหม่ ควรใส่ model version ไว้ใน cache key (เช่น `v2:too salty`) พอ deploy แล้ว entry เก่าจะไม่ถูกอ่านอีกและ LRU จะค่อยๆ ลบออกเอง แต่ช่วงแรกหลัง deploy cache จะว่าง จึงต้องเผื่อ capacity ของ large model ไว้ด้วย

## Debrief

**Q4.** Name one thing you learned from these measurements that the lecture did not cover.

*ยกตัวอย่างหนึ่งสิ่งที่ได้เรียนรู้จากการวัดผลครั้งนี้ ที่ไม่มีพูดถึงใน lecture*

> ANS: pattern ที่ใช้ช่วย p50 และ cost ได้มาก แต่แทบไม่ช่วย tail latency เช่น `t3-cache` มี p50 แค่ 6 ms และ large calls 40% แต่ p95 ยัง 554 ms ใกล้กับ `t0-large` ที่ 584 ms เพราะรีวิวที่ช้าที่สุดยังต้องรอ large model อยู่ ถ้าดูแค่ค่าเฉลี่ยหรือ p50 จะเข้าใจผิดว่าระบบเร็วขึ้นสำหรับผู้ใช้ทุกคน
