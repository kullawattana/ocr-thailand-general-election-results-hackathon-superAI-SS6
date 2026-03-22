# OCR Thailand General Election Results

ระบบดึงข้อมูลผลการเลือกตั้งไทยด้วย AI Vision — Hackathon superAI SS6

---

## ภาพรวมโปรเจกต์

โปรเจกต์นี้เป็นระบบอัตโนมัติสำหรับ **อ่านและแปลงข้อมูลผลการเลือกตั้งทั่วไปของไทย** จากเอกสารสแกน (รูปภาพ PNG) ให้กลายเป็นข้อมูล CSV ที่มีโครงสร้าง โดยใช้ Vision AI Model (Typhoon-OCR) ในการอ่านตัวเลขภาษาไทยและข้อความในตาราง

**เทคโนโลยีหลัก:**
- Python + Jupyter Notebook
- [Typhoon-OCR](https://api.opentyphoon.ai) — Vision Language Model สำหรับ OCR
- Pandas, PIL (Pillow), OpenAI SDK

---

## โครงสร้างไฟล์และโฟลเดอร์

```
.
├── Thailand_General_Election_Results.ipynb   # Notebook หลัก
├── submission.csv                            # ผลลัพธ์ที่ส่ง (10,053 แถว)
└── data/
    ├── images/                               # รูปภาพเอกสารเลือกตั้ง (847 ไฟล์)
    │   ├── constituency_10_1.png
    │   ├── constituency_10_1_page2.png
    │   ├── party_list_10_2.png
    │   └── ...
    └── sample_labels/                        # Ground truth JSON (5 ตัวอย่าง)
        ├── constituency_10_1.json
        ├── constituency_12_6.json
        ├── party_list_10_2.json
        ├── party_list_16_4.json
        └── party_list_33_3.json
```

### รูปแบบชื่อไฟล์รูปภาพ

| รูปแบบ | ความหมาย |
|--------|----------|
| `constituency_XX_YY.png` | บัตรเขต จังหวัด XX เขตเลือกตั้ง YY (หน้า 1) |
| `constituency_XX_YY_pageN.png` | บัตรเขต หน้าที่ N (เอกสารหลายหน้า) |
| `party_list_XX_YY.png` | บัตรบัญชีรายชื่อ จังหวัด XX กลุ่ม YY |

---

## โครงสร้างข้อมูล

### 1. รูปภาพเอกสาร (`data/images/`)

- **ขนาด:** 2480 × 3507 px (A4 สแกน 300 DPI)
- **รูปแบบ:** PNG, RGB
- **เนื้อหา:** ตารางผลการนับคะแนน แบบฟอร์มทางการของ กกต. มีทั้งตัวเลขไทย (๐–๙) และอารบิก
- **จำนวน:** 847 ไฟล์ ครอบคลุมหลายจังหวัดทั่วประเทศ

### 2. Ground Truth Labels (`data/sample_labels/`)

#### บัตรเขต (`constituency_*.json`)
```json
{
  "province_name": "กรุงเทพมหานคร",
  "province_code": "10",
  "constituency_number": 1,
  "type": "constituency",
  "summary_by_section": {
    "1.1": 130445,
    "1.2": 82421
  },
  "results": [
    {
      "number": 9,
      "name": "นายพีรวุฒิ พิมพ์สมฤดี",
      "party": "ประชาธิปัตย์",
      "votes": 14813
    }
  ]
}
```

| Field | Type | ความหมาย |
|-------|------|----------|
| `province_name` | string | ชื่อจังหวัด |
| `province_code` | string | รหัสจังหวัด (2 หลัก) |
| `constituency_number` | int | หมายเลขเขตเลือกตั้ง |
| `type` | string | `"constituency"` หรือ `"party_list"` |
| `summary_by_section` | object | คะแนนรวมแต่ละหน่วยเลือกตั้ง |
| `results[].number` | int | หมายเลขผู้สมัคร |
| `results[].name` | string | ชื่อผู้สมัคร |
| `results[].party` | string | ชื่อพรรค |
| `results[].votes` | int | จำนวนคะแนน |

#### บัตรบัญชีรายชื่อ (`party_list_*.json`)
```json
{
  "type": "party_list",
  "province_code": "10",
  "constituency_number": 2,
  "results": [
    {
      "number": 1,
      "party": "ไทยทรัพย์ทวี",
      "votes": 39
    }
  ],
  "matched_parties": [...],
  "unmatched_parties": [...]
}
```

| Field | Type | ความหมาย |
|-------|------|----------|
| `results[].number` | int | ลำดับพรรค |
| `results[].party` | string | ชื่อพรรคการเมือง |
| `results[].votes` | int | จำนวนคะแนน |
| `matched_parties` | array | พรรคที่จับคู่ได้สำเร็จ |
| `unmatched_parties` | array | พรรคที่จับคู่ไม่ได้ |

### 3. Submission CSV (`submission.csv`)

```csv
id,votes
constituency_10_1_1,14813
constituency_10_1_2,14368
...
party_list_34_11_57,0
```

| Column | รูปแบบ | ความหมาย |
|--------|--------|----------|
| `id` | `{type}_{province}_{constituency}_{number}` | รหัสเฉพาะของแต่ละแถว |
| `votes` | int | จำนวนคะแนนที่อ่านได้ (0 = ไม่พบหรืออ่านไม่ได้) |

- **จำนวนแถว:** 10,053 แถว (รวม header)
- **Non-zero:** 7,403 แถว (73.6%)

---

## การทำงานของระบบ

### Pipeline ภาพรวม

```
Template CSV
     │
     ▼
จัดกลุ่มตาม doc_id
     │
     ▼ (สำหรับแต่ละเอกสาร)
โหลดรูปภาพ (1–3 หน้า)
     │
     ▼
Resize + แปลงเป็น Base64
     │
     ▼
ส่งไปยัง Typhoon-OCR API
     │
     ▼
รับ HTML table กลับมา
     │
     ▼
Parse HTML → {ชื่อพรรค: คะแนน}
     │
     ▼ (ถ้ามีหลายหน้า)
Merge หน้า (เลือก max ต่อพรรค)
     │
     ▼
Clean + Validate คะแนน
     │
     ▼
Map กลับไปยัง submission ID
     │
     ▼
บันทึก checkpoint
     │
     ▼
รวมผลทั้งหมด → submission.csv
```

### ฟังก์ชันหลัก

| ฟังก์ชัน | หน้าที่ |
|----------|---------|
| `load_template()` | โหลด CSV template และจัดกลุ่มตาม doc_id |
| `get_doc_images(doc_id)` | ค้นหาไฟล์รูปภาพทุกหน้าของเอกสาร |
| `to_base64(path)` | แปลงรูปภาพเป็น Base64 (resize ถ้าเกิน 1600px) |
| `call_one_image(img, parties, client)` | เรียก OCR API พร้อม retry logic (3 ครั้ง) |
| `parse_votes_from_ocr(raw, party_list)` | Parse HTML/JSON response → dict คะแนน |
| `merge_pages(pages)` | รวมผลหลายหน้า โดยเลือกค่าสูงสุดต่อพรรค |
| `extract_votes(parties, images, client)` | Orchestrate การ OCR ทั้งเอกสาร |
| `clean_votes(raw)` | ล้างข้อมูล → ตัวเลขล้วน |
| `load_checkpoint()` / `save_checkpoint()` | บันทึก/โหลดความคืบหน้า |

### การรองรับตัวเลขไทย

ระบบแปลงเลขไทย → อารบิกโดยอัตโนมัติ:

```
๐→0  ๑→1  ๒→2  ๓→3  ๔→4
๕→5  ๖→6  ๗→7  ๘→8  ๙→9
```

### Retry & Rate Limit

| สถานการณ์ | การจัดการ |
|-----------|----------|
| HTTP 429 (Rate limit) | รอ 65 วินาที แล้ว retry |
| HTTP 400 (Bad request) | คืน empty dict ทันที |
| Error อื่น ๆ | Exponential backoff, retry สูงสุด 3 ครั้ง |
| ระหว่างเอกสาร | หน่วงเวลา 3.5 วินาที (≤20 req/min) |

### Multi-page Merge Logic

เอกสารที่มีหลายหน้า จะ OCR แยกแต่ละหน้า แล้วรวมโดยเลือก **ค่าสูงสุด** ต่อพรรค เนื่องจากหน้าสุดท้ายมักเป็นผลรวมสะสม (ตัวเลขสูงกว่า)

---

## วิธีการรัน

### 1. ติดตั้ง Dependencies

```bash
pip install pandas pillow openai
```

### 2. ตั้งค่า API Key

```python
TYPHOON_API_KEY = "your-typhoon-api-key"
```

### 3. เตรียมข้อมูล

วางไฟล์รูปภาพในโฟลเดอร์ `data/images/` และเตรียม template CSV

### 4. รัน Notebook

เปิด `Thailand_General_Election_Results.ipynb` และรันทุก cell

### 5. ปรับ Configuration (ตัวแปรสำคัญ)

```python
RESET = False   # True = เริ่มใหม่, False = ต่อจาก checkpoint
LIMIT = None    # จำกัดจำนวนเอกสาร (None = ทั้งหมด, 1 = ทดสอบ)
```

---

## ผลลัพธ์

| Metric | ค่า |
|--------|-----|
| เอกสารทั้งหมด | 847 รูปภาพ |
| แถวผลลัพธ์ | 10,053 แถว |
| อ่านคะแนนสำเร็จ | 7,403 แถว (73.6%) |
| คะแนน = 0 | 2,650 แถว (26.4%) |

---

## หมายเหตุ

- โปรเจกต์นี้พัฒนาสำหรับ **superAI Hackathon SS6**
- ข้อมูลเลือกตั้งเป็นของจริงจาก กกต. (คณะกรรมการการเลือกตั้ง)
- การใช้งาน OCR API มีค่าใช้จ่ายตาม usage ของ Typhoon API
