# AI_USAGE.md — การใช้ AI ในการบ้านนี้

**เครื่องมือ:** Claude Code (Anthropic)

> หัว notebook เขียนว่า "ห้ามใช้ AI ช่วยเขียนโค้ด" แต่ข้อกำหนดการบ้าน Day 3 ของคอร์ส อนุญาตให้ใช้ AI ได้ถ้า**ประกาศใน AI_USAGE.md และวิจารณ์จุดที่ AI พลาด** ไฟล์นี้จึงบันทึกการใช้ทั้งหมด

## 1. ใช้ AI ตรงไหน

| ส่วน | AI ทำอะไร | ทำเอง / ตัดสินใจเอง |
|---|---|---|
| Step 1 (Cell 8) | ร่างโค้ด chunk → embed → Qdrant ตาม hint ใน notebook | รันใน Colab, ตรวจว่าได้ 12 chunks |
| Step 2 (Cell 10) | ร่าง tool `calc_electricity_bill`, instruction และฟังก์ชัน `chat_with_agent` ที่พิมพ์ trace | เลือก tool ค่าไฟฟ้า, ตรวจตัวเลข 1,053.30 บาท |
| Step 3 (Cell 12) | เติม `search_knowledge`, ห่อ judge ให้ดึง JSON ออกมาได้ | รัน, อ่าน trace, สังเกตว่าข้อ Embedding ไม่มีเอกสารรองรับ |
| ระดับกลาง (F0–SUMMARY) | ออกแบบชุดทดสอบ 8+4 คำถาม, metric Hit@1 / refusal / faithfulness, โค้ด FIX-1 และ FIX-2 | รันทุก cell, ตัดสินใจแสดงผล over-refusal แทนการซ่อน |
| Debug | วิเคราะห์ error 401 / 429 / 503 / 404 / timeout และเสนอทางแก้ | จัดการ API key เอง, ตัดสินใจย้ายไป KKU IntelSphere |
| เอกสาร | ร่าง README, failures.md, AI_USAGE.md จากผลจริงใน notebook | ตรวจตัวเลขกับ output ใน notebook |

## 2. Prompt หลักที่ใช้

- "ทำการบ้าน Day 3 ระดับกลาง (เพดาน 8/10)" พร้อมแนบภาพโจทย์และไฟล์ notebook
- "บอกว่าต้องแก้ไฟล์ตรงไหน แล้วบอกว่ารัน cell ไหน" (ให้ AI บอก แล้วแก้ใน Colab เอง)
- ส่ง error เต็ม ๆ ให้ AI ทุกครั้งที่ cell ล้ม เช่น `401 UNAUTHENTICATED`, `429 RESOURCE_EXHAUSTED`, `503 UNAVAILABLE`
- "ระหว่าง api key ของ google studio กับ key ของ kku intelsphere อันไหนดีกว่า"
- "ปรับให้ตารางตรงหน่อย" (ตาราง SUMMARY เหลื่อมเพราะภาษาไทย)

## 3. AI แนะนำผิดตรงไหน (สำคัญ)

| # | AI แนะนำว่า | ความจริง | ผลกระทบ / แก้อย่างไร |
|---|---|---|---|
| 1 | "API key ของ AI Studio ต้องขึ้นต้นด้วย `AIza` ถ้าเป็น `AQ.` แปลว่าผิด" | AI Studio ออก key รูปแบบใหม่ `AQ.` แล้ว key ถูกต้อง ปัญหาจริงคือไลบรารีเก่า | เสียเวลาไปลอง Vertex AI และเกือบสร้าง key ใหม่โดยไม่จำเป็น แก้ด้วย `pip install -U` |
| 2 | "key `AQ.` เป็นของ Vertex AI express mode ให้ตั้ง `GOOGLE_GENAI_USE_VERTEXAI=TRUE`" | ใช้ไม่ได้ ได้ 401 เหมือนเดิม | ต้องย้อนการแก้ และ restart session |
| 3 | "ถ้า 3.8-flash ยุ่ง ให้ใช้ `gemini-2.5-flash`" | `404 ... no longer available to new users` | เสีย 1 รอบ จึงเปลี่ยนเป็นเขียน cell ทดสอบทุกรุ่นก่อนเลือก |
| 4 | ตัวลองใหม่รอ 20 → 40 → 60 วินาที และไม่มี timeout | request ที่ค้างไม่ error จึงไม่เข้าตัวลองใหม่ เลยค้าง 20 นาที และทุกครั้งที่ลองใหม่ต้องเริ่มคำถามใหม่ทั้งหมด | เพิ่ม `asyncio.wait_for(timeout=90)` และลดเวลารอ |
| 5 | `print(litellm.__version__)` | `AttributeError` เพราะ litellm ไม่มี `__version__` | เปลี่ยนเป็น `importlib.metadata.version()` |
| 6 | FIX-1 รอบแรกเสนอ `chunk_size=300` | ทำให้ splitter รวม 2 เอกสารเป็น chunk เดียว (เหลือ 5 chunks) | AI ตรวจพบเองก่อนส่งโค้ดด้วยการแบ่ง chunk ทดสอบในเครื่อง แล้วเปลี่ยนเป็น 200 |
| 7 | ใช้ `ljust`/f-string จัดตาราง SUMMARY | สระ/วรรณยุกต์ไทยทำให้คอลัมน์เหลื่อม | เปลี่ยนเป็น pandas DataFrame |
| 8 | คาดว่า agent v2 "ตอบคำถามในฐานความรู้ได้ 100% เหมือนเดิม" | ได้ 88% เพราะปฏิเสธคำถาม RAG 1 ข้อ (over-refusal) | ไม่ได้ซ่อนผล แต่รายงานเป็น trade-off ใน failures.md |
| 9 | ตอนแรกคาดว่า F3 จะเห็นชัดที่คำถาม `Machine Learning คืออะไร` | ผลจริง chunk ที่ถูกขึ้นอันดับ 1 อยู่แล้ว เศษ chunk อยู่อันดับ 2 | ใช้หลักฐาน F3 จาก Step 1 (`MY_QUERY`) และ `Google ADK` แทน ซึ่งเห็นชัดกว่า |

## 4. สิ่งที่ตรวจเองก่อนส่ง

- ตัวเลขทุกตัวใน README และ failures.md ตรงกับ output ใน notebook (Hit@1 5/8 → 8/8, refusal 0/4 → 4/4, answered 8/8 → 7/8, faithfulness 2/5 → 5/5)
- Verification Code `87aa5a0c619b` ตรงกับรหัส `6730401233`
- ไม่มี API key ใน notebook, screenshot และ commit (key อยู่ใน Colab Secrets เท่านั้น ดู `.env.example`)
- ไม่ได้ใส่เบอร์โทรและ LINE ID ใน notebook ที่ขึ้น repo แบบ Public
