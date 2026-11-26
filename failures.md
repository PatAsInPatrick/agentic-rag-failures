# failures.md — 5 Failure ของ RAG Agent + การแก้ 2 จุด

> ข้อมูล: รหัส `6730401233` → เอกสาร 8 ชิ้น (Chunking, NLP, Deep Learning, Cosine Similarity, Machine Learning, Transformer, Vector Database, RAG)
> **ไม่มี** เอกสาร Embedding, Agent, Prompt Engineering, Google ADK (ถูกสุ่มตัดออก)
> Model: `gemini-3.8-flash` ผ่าน KKU IntelSphere · Embedding: `intfloat/multilingual-e5-large` · Vector DB: Qdrant (in-memory)
> ตัวเลขและ trace ทั้งหมดมาจาก [`agentic_rag_4hr_homework.ipynb`](agentic_rag_4hr_homework.ipynb) รอบที่ส่ง

## สรุป

| # | Failure | ชนิด | ตัวอย่าง query | แก้ด้วย |
|---|---|---|---|---|
| F1 | ฐานความรู้ไม่มีคำตอบ แต่ agent ตอบเต็มจากความรู้ตัวเอง | **Retrieval (coverage) → Reasoning (grounding)** | `Embedding คืออะไร?` | FIX-2 |
| F2 | Chunk ถูกตัดกลางประโยค ข้อมูลครึ่งหลังหลุดไปอีก chunk | **Retrieval (chunking)** | `Deep Learning ใช้ทำงานอะไรได้บ้าง` | FIX-1 |
| F3 | Chunk เศษสั้น ๆ ติดอันดับสูงแทน chunk ที่มีเนื้อหา | **Retrieval (ranking)** | `MY_QUERY`, `Google ADK คืออะไร` | FIX-1 |
| F4 | คำถามนอกฐานความรู้ agent ไม่ปฏิเสธ (0/4) | **Reasoning (grounding)** | `Google ADK คืออะไร` | FIX-2 |
| F5 | LLM-as-Judge ให้ 5/5 กับคำตอบที่ไม่ได้มาจากเอกสาร | **Evaluation** | `Embedding คืออะไร?` | ใช้ faithfulness judge |

---

## F1 — Knowledge gap: ไม่มีเอกสาร แต่ยังตอบ

**Query:** `Embedding คืออะไร?` (คำถามบังคับข้อ 2 ของ Step 3)

**Trace (ก่อนแก้)**
```
🔧 CALL   search_knowledge({'query': 'Embedding คืออะไร'})
📥 RESULT [0.8631] Chunking คือกระบวนการแบ่งเอกสาร...สำหรับการสร้าง Embedding ...
          [0.8342] Vector Database ...ในรูปแบบ Embedding Vector ...
          [0.7966] เช่น การจดจำภาพ การแปลภาษา
🔧 CALL   search_knowledge({'query': 'Embedding ความหมาย เวกเตอร์'})   ← ค้นซ้ำเพราะยังไม่เจอนิยาม
📥 RESULT (3 chunk เดิม)
🤖 Embedding คือ กระบวนการแปลงข้อมูล เช่น ข้อความ รูปภาพ หรือเสียง ให้เป็นเวกเตอร์ตัวเลขในปริภูมิหลายมิติ ...
```

**วิเคราะห์:** retriever ทำงานถูก มันคืน chunk ที่ "มีคำว่า Embedding" แต่ไม่มีนิยาม agent รู้ตัวว่ายังไม่เจอ (ค้นซ้ำครั้งที่ 2) แต่สุดท้ายก็ตอบนิยามจากความรู้ของ model เอง ใน Step 3 ยังยกตัวอย่าง "ราชา/ราชินี" ที่ไม่มีในเอกสารเลย ต้นเหตุคือ instruction เดิมบอกแค่ "ค้นก่อนตอบ" ไม่ได้บอกว่า "ถ้าไม่เจอให้ห้ามตอบ"

---

## F2 — Chunk ถูกตัดกลางประโยค

**ต้นเหตุ:** `chunk_size=150, chunk_overlap=30` กับเอกสารยาว 121–164 ตัวอักษร ทำให้เอกสารที่ยาวเกิน 150 ถูกแบ่งเป็น 2 ชิ้น
```
[2] (139) Deep Learning เป็นเทคนิค...ประมวลผลข้อมูลที่ซับซ้อน เช่น การจดจำภาพ
[3] ( 26) เช่น การจดจำภาพ การแปลภาษา                        ← "การแปลภาษา" หลุดมาอยู่ chunk นี้
[4] (142) Cosine Similarity ... นิยมใช้ในงาน NLP และ Information
[5] ( 29) NLP และ Information Retrieval                     ← คำว่า "Retrieval" หลุด
[10] (99) Retrieval Augmented Generation หรือ RAG ...ของ LLM
[11] (54) LLM เพื่อให้ได้คำตอบที่ถูกต้องและอ้างอิงแหล่งข้อมูลได้   ← ไม่มีคำว่า RAG
```

**Trace (ก่อนแก้):** `Deep Learning ใช้ทำงานอะไรได้บ้าง`
```
🔧 CALL search_knowledge('Deep Learning ทำงานอะไรได้บ้าง การประยุกต์ใช้')       → ได้ chunk [2] ไม่มี "การแปลภาษา"
🔧 CALL search_knowledge('การจดจำภาพ Computer Vision NLP ... Deep Learning')   → เริ่มเห็น chunk [3]
🔧 CALL search_knowledge('การจดจำภาพ การแปลภาษา Deep Learning')               → ครั้งที่ 3 ถึงได้ครบ
🤖 ... การจดจำใบหน้าในภาพถ่ายและวิดีโอ ... ข้อมูลเสียง, ข้อมูลสัญญาณ ...       ← เติมเองเกินเอกสาร
```

**Metric (Cell F0):** วัด Hit@1 = chunk อันดับ 1 มีคำสำคัญของคำตอบครบ บน 8 คำถามที่คำตอบอยู่ในเอกสารแน่นอน
→ **Hit@1 = 5/8 (62%)** พลาดที่ Deep Learning, Cosine Similarity, RAG ซึ่งตรงกับ 3 เอกสารที่ถูกตัดพอดี

---

## F3 — Chunk เศษติดอันดับสูง

**หลักฐาน 1 (Step 1):** query เฉพาะตัว `เทคนิคการค้นหาข้อมูลที่มีความหมายคล้ายกัน`
```
[5] score=0.8365 | NLP และ Information Retrieval        ← อันดับ 1 คือเศษ 29 ตัวอักษร
[4] score=0.8260 | Cosine Similarity ...
[3] score=0.8220 | เช่น การจดจำภาพ การแปลภาษา           ← อันดับ 3 ก็เป็นเศษ และไม่เกี่ยวเลย
```
chunk ที่ตอบได้ตรงที่สุดคือ `[9] Vector Database ... ค้นหาข้อมูลที่มีความหมายคล้ายกัน` **ไม่ติด top-3**

**หลักฐาน 2 (Cell F1):** `Google ADK คืออะไร` → top-3 เป็นเศษล้วน
```
[0.7804] NLP และ Information Retrieval
[0.7617] เช่น การจดจำภาพ การแปลภาษา
[0.7568] การเรียนรู้ของเครื่อง หรือ Machine Learning
```

**วิเคราะห์:** chunk สั้นมีคำไม่กี่คำ embedding จึงเป็น "ค่าเฉลี่ยกว้าง ๆ" ที่ใกล้กับ query ทั่วไปได้ง่าย กลายเป็น noise ที่แย่งที่ chunk จริง

---

## F4 — คำถามนอกฐานความรู้ ไม่ปฏิเสธ

**Query:** `Google ADK คืออะไร` (เอกสาร ADK ถูกสุ่มตัดออก)
```
🔧 CALL search_knowledge('Google ADK')                                          → เศษ chunk score ~0.78
🔧 CALL search_knowledge('ADK Agent Development Kit Accessory Development Kit')  → เศษ chunk เดิม
🤖 จากข้อมูลทั่วไป คำว่า Google ADK ส่วนใหญ่มักหมายถึง Android Open Accessory Development Kit ...
   (อธิบายยาวเรื่อง Arduino, USB Host, AOA protocol)
```

**วิเคราะห์:** agent ตอบเรื่องที่ไม่อยู่ในเอกสาร และตีความผิดบริบท (คอร์สนี้หมายถึง Agent Development Kit ไม่ใช่ Android Accessory) แม้จะเขียนว่า "จากข้อมูลทั่วไป" แต่ผู้ใช้ RAG คาดหวังคำตอบจากฐานความรู้

**Metric (Cell FIX-2, BEFORE):** ปฏิเสธคำถามนอกฐานความรู้ได้ **0/4** (Embedding, Google ADK, Agent vs Chatbot, Prompt Engineering ตอบเองทั้ง 4 ข้อ)

---

## F5 — Judge ให้คะแนนโดยไม่ดู context

**Cell F2:** เอาคำตอบ F1 (`Embedding คืออะไร?`) ให้ judge 2 แบบ
```
Judge เดิม (ไม่มี context): 5/5 — คำตอบมีความถูกต้อง ครบถ้วน และอธิบายได้อย่างชัดเจน ...
Faithfulness (มี context):  2/5 — เอกสารอ้างอิงไม่มีการระบุนิยามว่า Embedding คืออะไร
                                  ... จึงเป็นการนำความรู้ภายนอกมาตอบ
```

**วิเคราะห์:** `JUDGE_PROMPT` ของ notebook ไม่ได้ส่งเอกสารที่ค้นเจอให้ judge ดู judge จึงวัด "ถูกตามความรู้ทั่วไป" ไม่ใช่ "ตอบจากเอกสาร" ผลคือ Step 3 ได้เฉลี่ย 5.00/5 ทั้งที่ข้อ 2 เป็น hallucination เทียบกับฐานความรู้ ถ้าใช้ judge แบบเดิมวัดผลการแก้ จะมองไม่เห็นว่าระบบดีขึ้น

---

# การแก้ (2 จุด)

## FIX-1 — Chunk ตามย่อหน้า (แก้ F2, F3)

```python
# เอกสารยาวสุด 164 ตัวอักษร → chunk_size=200 ทำให้ 1 เอกสาร = 1 chunk
splitter_v2 = RecursiveCharacterTextSplitter(chunk_size=200, chunk_overlap=0, separators=['\n\n', ' ', ''])
```

ทำไม 200 ไม่ใช่ 300: ทดลองแล้ว 300 ทำให้ splitter **รวม 2 เอกสารเป็น chunk เดียว** (เหลือ 5 chunks) ต้องเลือกค่าที่ ≥ เอกสารยาวสุด (164) แต่ < สองเอกสารรวมกัน (~260)

| | Before | After |
|---|---|---|
| จำนวน chunks | 12 | 8 |
| chunk สั้นสุด | 26 ตัวอักษร | 121 ตัวอักษร |
| **Hit@1** (8 คำถาม) | **62%** | **100%** |
| Hit@3 | 62% | 100% |

**Trace หลังแก้:** `Deep Learning ใช้ทำงานอะไรได้บ้าง`
```
🔧 CALL search_knowledge_v2('Deep Learning ใช้ทำงานอะไรได้บ้าง')   ← ค้นครั้งเดียว (เดิม 3 ครั้ง)
📥 RESULT [0.9116] Deep Learning ... เช่น การจดจำภาพ การแปลภาษา    ← ประโยคครบใน chunk เดียว
🤖 จากข้อมูลในฐานความรู้ Deep Learning ใช้ในการประมวลผลข้อมูลที่ซับซ้อน ตัวอย่างเช่น: การจดจำภาพ, การแปลภาษา
```

**ข้อจำกัด:** ได้ผลเพราะเอกสารชุดนี้สั้นทุกชิ้น ถ้าเอกสารยาวเป็นหน้า ต้องใช้การแบ่งตามประโยคหรือ semantic chunking แทน

## FIX-2 — Grounding instruction (แก้ F1, F4 และวัดด้วย faithfulness แก้ F5)

```python
instruction=('คุณเป็น AI ที่ตอบคำถามจากฐานความรู้เท่านั้น ใช้ tool search_knowledge_v2 ก่อนตอบเสมอ\n'
             '- ตอบโดยใช้เฉพาะข้อความที่อยู่ในผลค้นหา ห้ามเติมความรู้ของตัวเอง\n'
             '- ถ้าผลค้นหาไม่ได้อธิบายสิ่งที่ถามโดยตรง (แค่มีคำนั้นปรากฏ ไม่นับ) '
             'ให้ตอบว่า "ไม่พบข้อมูลนี้ในฐานความรู้" แล้วบอกว่าพบหัวข้อใกล้เคียงอะไรบ้าง\n'
             '- ตอบเป็นภาษาไทย')
```

ประโยค *"แค่มีคำนั้นปรากฏ ไม่นับ"* ใส่เพื่อแก้ F1 โดยตรง เพราะ chunk Chunking กับ Vector Database มีคำว่า Embedding แต่ไม่ใช่นิยาม

| | Before | After |
|---|---|---|
| **ปฏิเสธคำถามนอกฐานความรู้** (4 คำถาม) | **0%** | **100%** |
| ตอบคำถามในฐานความรู้ได้ (8 คำถาม) | 100% | **88%** ⚠️ |
| Faithfulness ของ `Embedding คืออะไร?` | 2/5 | **5/5** |

**Trace หลังแก้:** `Embedding คืออะไร?`
```
🔧 CALL search_knowledge_v2('Embedding คืออะไร')
🤖 ไม่พบข้อมูลนี้ในฐานความรู้
   อย่างไรก็ตาม ในฐานความรู้พบหัวข้อใกล้เคียง: Chunking, Vector Database, Deep Learning
Faithfulness หลังแก้: 5/5 — ระบุถูกต้องว่าไม่พบนิยาม ... ไม่มีการกุข้อมูลขึ้นเอง
```

### ⚠️ ผลข้างเคียง: Over-refusal (failure ใหม่ที่เกิดจากการแก้)

FIX-2 ทำให้ agent **ปฏิเสธคำถามที่มีคำตอบ 1 ข้อ** (Cell over_refusal):
```
❌ ปฏิเสธเกิน: RAG ช่วยให้อ้างอิงแหล่งข้อมูลได้อย่างไร
   → ไม่พบข้อมูลนี้ในฐานความรู้ ในฐานความรู้มีเพียงการระบุว่า RAG เป็นเทคนิคที่รวมการค้นหาข้อมูล...
   ผลค้นหา: [0.8880] Retrieval Augmented Generation ... เพื่อให้ได้คำตอบที่ถูกต้องและอ้างอิงแหล่งข้อมูลได้
```
เอกสารบอกว่า RAG "อ้างอิงแหล่งข้อมูลได้" แต่ไม่ได้อธิบายว่า "อย่างไร" agent ที่ถูกสั่งให้เข้มงวดจึงตีความว่า "ไม่ได้อธิบายโดยตรง" แปลว่ากฎ "ห้ามตอบถ้าไม่ตรง" แลกความปลอดภัย (ไม่ hallucinate) กับ recall ไป 12% ทางแก้ต่อไปคือให้ตอบบางส่วน + ระบุว่าส่วนไหนเอกสารไม่ได้อธิบาย แทนการปฏิเสธทั้งข้อ
