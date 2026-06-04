# Amazon Comprehend Medical — Phân Tích Văn Bản Y Tế Chuyên Biệt

> Amazon Comprehend Medical là dịch vụ NLP (Natural Language Processing — Xử Lý Ngôn Ngữ Tự Nhiên) được thiết kế riêng cho lĩnh vực y tế, trích xuất thông tin có cấu trúc từ văn bản y tế phi cấu trúc (unstructured medical text) như bệnh án, ghi chú lâm sàng, toa thuốc, tóm tắt xuất viện. Dịch vụ còn hỗ trợ map (ánh xạ) sang các chuẩn mã hóa y tế quốc tế: **ICD-10-CM** (Mã Bệnh Quốc Tế — International Classification of Diseases) và **RxNorm** (Chuẩn Mã Thuốc), đồng thời phát hiện và ẩn danh hóa **PHI** (Protected Health Information — Thông Tin Y Tế Được Bảo Vệ).

---

## 🎯 Comprehend Medical Làm Được Gì?

| Tính Năng | API | Mô Tả |
|---|---|---|
| **Medical Entity Detection** (Phát Hiện Thực Thể Y Tế) | `detect_entities_v2` | Tìm medication, condition, anatomy, procedure... |
| **ICD-10-CM Inference** (Suy Luận Mã Bệnh) | `infer_icd10_cm` | Map triệu chứng/bệnh sang mã ICD-10-CM |
| **RxNorm Inference** (Suy Luận Mã Thuốc) | `infer_rx_norm` | Map tên thuốc sang mã RxNorm chuẩn |
| **SNOMED CT Inference** | `infer_snomedct` | Map sang ontology y tế SNOMED CT |
| **PHI Detection** (Phát Hiện Thông Tin Cá Nhân Y Tế) | `detect_phi` | Tìm tên, ngày sinh, số điện thoại, địa chỉ... |

> **Lưu ý quan trọng:** Comprehend Medical **chỉ hỗ trợ tiếng Anh** và được thiết kế cho văn bản y tế — không phải công cụ chẩn đoán y tế. Không thay thế phán đoán của chuyên gia y tế.

---

## 🏥 1. Medical Entity Detection (Phát Hiện Thực Thể Y Tế)

### Khởi Tạo Client

```python
import boto3
import json

comprehend_medical = boto3.client("comprehendmedical", region_name="us-east-1")
```

### Phát Hiện Entity Y Tế

```python
def detect_medical_entities(text: str) -> dict:
    """Phát hiện các thực thể y tế trong ghi chú lâm sàng."""
    response = comprehend_medical.detect_entities_v2(Text=text)

    for entity in response["Entities"]:
        print(
            f"[{entity['Category']}] [{entity['Type']}] '{entity['Text']}' "
            f"(tin cậy: {entity['Score']:.2f})"
        )
        # Attributes: thuộc tính liên quan (dosage, frequency, route...)
        for attr in entity.get("Attributes", []):
            print(f"  → {attr['Type']}: '{attr['Text']}' ({attr['Score']:.2f})")
    return response["Entities"]

text = """
Patient is a 65-year-old male with a history of hypertension and type 2 diabetes.
He is currently taking Metformin 500mg twice daily and Lisinopril 10mg once daily.
He presents with chest pain and shortness of breath for the past 3 days.
"""
detect_medical_entities(text)
```

**Output mẫu:**

```
[MEDICAL_CONDITION] [DX_NAME] 'hypertension' (tin cậy: 0.99)
[MEDICAL_CONDITION] [DX_NAME] 'type 2 diabetes' (tin cậy: 0.98)
[MEDICATION] [GENERIC_NAME] 'Metformin' (tin cậy: 0.99)
  → DOSAGE: '500mg' (0.98)
  → FREQUENCY: 'twice daily' (0.97)
  → ROUTE_OR_MODE: 'oral' (0.85)  ← Comprehend suy luận route từ context
[MEDICATION] [GENERIC_NAME] 'Lisinopril' (tin cậy: 0.99)
  → DOSAGE: '10mg' (0.97)
  → FREQUENCY: 'once daily' (0.96)
[MEDICAL_CONDITION] [DX_NAME] 'chest pain' (tin cậy: 0.99)
[MEDICAL_CONDITION] [SYMPTOM] 'shortness of breath' (tin cậy: 0.97)
[TIME_EXPRESSION] [TIME_TO_DX_NAME] '3 days' (tin cậy: 0.95)
```

### Các Category Entity Y Tế

| Category (Danh Mục) | Type (Loại) | Ví Dụ |
|---|---|---|
| `MEDICATION` (Thuốc) | `GENERIC_NAME`, `BRAND_NAME` | "Metformin", "Glucophage" |
| `MEDICATION` (Thuộc Tính Thuốc) | `DOSAGE`, `FREQUENCY`, `ROUTE_OR_MODE`, `DURATION` | "500mg", "twice daily", "oral", "2 weeks" |
| `MEDICAL_CONDITION` (Tình Trạng Y Tế) | `DX_NAME` (chẩn đoán), `SYMPTOM`, `SIGN` | "hypertension", "chest pain", "tachycardia" |
| `ANATOMY` (Giải Phẫu) | `SYSTEM_ORGAN_SITE`, `DIRECTION` | "left ventricle", "bilateral" |
| `TEST_TREATMENT_PROCEDURE` (Xét Nghiệm/Điều Trị) | `TEST_NAME`, `TREATMENT_NAME`, `PROCEDURE_NAME` | "CBC", "MRI", "appendectomy" |
| `PROTECTED_HEALTH_INFORMATION` (PHI) | `NAME`, `DATE`, `AGE`, `ADDRESS`, `ID`, `PHONE` | "John Doe", "01/01/1960", "555-1234" |
| `TIME_EXPRESSION` (Biểu Thức Thời Gian) | `TIME_TO_DX_NAME`, `TIME_OF_EVENT` | "3 days", "last week" |

### Relationship Extraction (Trích Xuất Mối Quan Hệ)

Comprehend Medical tự động liên kết **Attributes** (thuộc tính) với entity cha tương ứng:

```
Metformin [MEDICATION]
    ├── 500mg [DOSAGE]       → liên kết với Metformin
    ├── twice daily [FREQUENCY] → liên kết với Metformin
    └── oral [ROUTE_OR_MODE]  → liên kết với Metformin

Lisinopril [MEDICATION]
    ├── 10mg [DOSAGE]        → liên kết với Lisinopril
    └── once daily [FREQUENCY] → liên kết với Lisinopril
```

---

## 🔢 2. ICD-10-CM Inference (Suy Luận Mã Bệnh ICD-10-CM)

**ICD-10-CM** — International Classification of Diseases, 10th Revision, Clinical Modification — Phân Loại Bệnh Tật Quốc Tế Sửa Đổi Lâm Sàng là hệ thống mã hóa bệnh chuẩn của WHO được dùng rộng rãi trong hệ thống y tế Mỹ và toàn cầu.

```python
def infer_icd10_cm(text: str) -> None:
    """Map triệu chứng và chẩn đoán sang mã ICD-10-CM."""
    response = comprehend_medical.infer_icd10_cm(Text=text)

    for entity in response["Entities"]:
        print(f"\nVăn bản: '{entity['Text']}'")
        print(f"Category: {entity['Category']}, Type: {entity['Type']}")

        # ICD-10-CM Concepts: danh sách mã khả năng theo thứ tự tin cậy
        for concept in entity.get("ICD10CMConcepts", [])[:3]:  # Top 3
            print(f"  Mã ICD-10-CM: {concept['Code']} — {concept['Description']} "
                  f"(tin cậy: {concept['Score']:.3f})")

text = "Patient has essential hypertension and type 2 diabetes mellitus without complications."
infer_icd10_cm(text)
```

**Output mẫu:**

```
Văn bản: 'essential hypertension'
Category: MEDICAL_CONDITION, Type: DX_NAME
  Mã ICD-10-CM: I10 — Essential (primary) hypertension (tin cậy: 0.998)
  Mã ICD-10-CM: I15.9 — Secondary hypertension, unspecified (tin cậy: 0.021)

Văn bản: 'type 2 diabetes mellitus without complications'
Category: MEDICAL_CONDITION, Type: DX_NAME
  Mã ICD-10-CM: E11.9 — Type 2 diabetes mellitus without complications (tin cậy: 0.997)
  Mã ICD-10-CM: E11.65 — Type 2 diabetes mellitus with hyperglycemia (tin cậy: 0.015)
```

### Ứng Dụng ICD-10-CM

- **Medical coding automation** (Tự Động Hóa Mã Hóa Y Tế): Hỗ trợ bộ phận billing (thanh toán bảo hiểm) gán mã bệnh từ ghi chú bác sĩ
- **Clinical data analytics** (Phân Tích Dữ Liệu Lâm Sàng): Chuẩn hóa chẩn đoán để phân tích xu hướng bệnh
- **EHR integration** (Tích Hợp Hồ Sơ Bệnh Án Điện Tử — Electronic Health Record): Tự động điền mã ICD vào hệ thống EHR

---

## 💊 3. RxNorm Inference (Suy Luận Mã Thuốc RxNorm)

**RxNorm** là ontology (bản thể luận) chuẩn của Thư Viện Y Tế Quốc Gia Mỹ (NLM — National Library of Medicine), định danh duy nhất cho mỗi loại thuốc bất kể tên thương mại hay generic.

```python
def infer_rx_norm(text: str) -> None:
    """Map tên thuốc sang mã RxNorm chuẩn."""
    response = comprehend_medical.infer_rx_norm(Text=text)

    for entity in response["Entities"]:
        print(f"\nThuốc: '{entity['Text']}'")

        # Attributes: dosage, frequency, route
        for attr in entity.get("Attributes", []):
            print(f"  {attr['Type']}: '{attr['Text']}'")

        # RxNorm Concepts: mã định danh chuẩn
        for concept in entity.get("RxNormConcepts", [])[:2]:
            print(f"  RxNorm Code: {concept['Code']} — {concept['Description']} "
                  f"(tin cậy: {concept['Score']:.3f})")

text = "Start Atorvastatin 40mg at bedtime. Continue Aspirin 81mg daily."
infer_rx_norm(text)
```

**Output mẫu:**

```
Thuốc: 'Atorvastatin'
  DOSAGE: '40mg'
  FREQUENCY: 'at bedtime'
  RxNorm Code: 83367 — Atorvastatin (tin cậy: 0.999)
  RxNorm Code: 617310 — Atorvastatin 40 MG Oral Tablet (tin cậy: 0.956)

Thuốc: 'Aspirin'
  DOSAGE: '81mg'
  FREQUENCY: 'daily'
  RxNorm Code: 1191 — Aspirin (tin cậy: 0.999)
  RxNorm Code: 308416 — Aspirin 81 MG Oral Tablet (tin cậy: 0.921)
```

### Ứng Dụng RxNorm

- **Medication reconciliation** (Đối Chiếu Thuốc): So sánh danh sách thuốc từ nhiều nguồn khác nhau bằng mã chuẩn
- **Drug interaction checking** (Kiểm Tra Tương Tác Thuốc): Tra cứu tương tác dựa trên RxNorm code
- **Pharmacy system integration** (Tích Hợp Hệ Thống Dược): Chuẩn hóa đơn thuốc vào hệ thống pharmacy

---

## 🛡️ 4. PHI Detection (Phát Hiện Thông Tin Y Tế Được Bảo Vệ)

**PHI** — Protected Health Information — Thông Tin Y Tế Được Bảo Vệ là thông tin cá nhân gắn với sức khỏe của bệnh nhân, được bảo vệ bởi **HIPAA** (Health Insurance Portability and Accountability Act — Luật Bảo Mật Thông Tin Y Tế Mỹ).

### Phát Hiện PHI

```python
def detect_phi(text: str) -> list:
    """Phát hiện các thông tin cá nhân y tế (PHI) trong văn bản."""
    response = comprehend_medical.detect_phi(Text=text)

    phi_entities = []
    for entity in response["Entities"]:
        print(
            f"[PHI — {entity['Type']}] '{entity['Text']}' "
            f"(vị trí: {entity['BeginOffset']}-{entity['EndOffset']}, "
            f"tin cậy: {entity['Score']:.2f})"
        )
        phi_entities.append(entity)
    return phi_entities

text = """
Patient: John Smith, DOB: 01/15/1960
Phone: (555) 123-4567, Address: 123 Main St, Seattle, WA 98101
MRN: 78901234
"""
detect_phi(text)
```

**Output mẫu:**

```
[PHI — NAME] 'John Smith' (vị trí: 9-19, tin cậy: 0.99)
[PHI — DATE] '01/15/1960' (vị trí: 26-36, tin cậy: 0.99)
[PHI — PHONE_OR_FAX] '(555) 123-4567' (vị trí: 44-58, tin cậy: 0.98)
[PHI — ADDRESS] '123 Main St, Seattle, WA 98101' (vị trí: 69-99, tin cậy: 0.97)
[PHI — ID] '78901234' (vị trí: 105-113, tin cậy: 0.96)
```

### Các Loại PHI Được Phát Hiện

| Loại PHI | Ví Dụ |
|---|---|
| `NAME` | "John Smith", "Dr. Jane Doe" |
| `DATE` | "01/15/1960", "March 2024", "yesterday" |
| `AGE` | "65-year-old", "age 42" |
| `PHONE_OR_FAX` | "(555) 123-4567", "+1-800-555-0100" |
| `EMAIL` | "patient@example.com" |
| `ADDRESS` | "123 Main St, Seattle, WA" |
| `ID` | "MRN: 78901234", "SSN: 123-45-6789" |
| `URL` | "http://patient-portal.hospital.com" |
| `PROFESSION` | "nurse", "teacher" (nếu định danh bệnh nhân) |

### De-identification (Ẩn Danh Hóa Văn Bản Y Tế)

Dùng kết quả PHI detection để **che/thay thế** thông tin cá nhân trước khi chia sẻ dữ liệu:

```python
def deidentify_text(text: str) -> str:
    """Ẩn danh hóa văn bản y tế bằng cách thay thế PHI bằng nhãn loại."""
    response = comprehend_medical.detect_phi(Text=text)
    entities = response["Entities"]

    # Sắp xếp từ cuối lên đầu để offset không bị sai khi thay thế
    entities_sorted = sorted(entities, key=lambda e: e["BeginOffset"], reverse=True)

    text_list = list(text)
    for entity in entities_sorted:
        begin = entity["BeginOffset"]
        end = entity["EndOffset"]
        replacement = f"[{entity['Type']}]"
        text_list[begin:end] = list(replacement)

    return "".join(text_list)

original = "Patient John Smith, DOB 01/15/1960, has hypertension."
deidentified = deidentify_text(original)
print(deidentified)
# → "Patient [NAME], DOB [DATE], has hypertension."
```

---

## 🔬 5. SNOMED CT Inference (Suy Luận Ontology Y Tế SNOMED CT)

**SNOMED CT** — Systematized Nomenclature of Medicine Clinical Terms — Danh Pháp Y Khoa Hệ Thống là ontology y tế toàn diện nhất thế giới, dùng làm chuẩn trong EHR và hệ thống y tế toàn cầu.

```python
def infer_snomedct(text: str) -> None:
    """Map entity y tế sang SNOMED CT concepts."""
    response = comprehend_medical.infer_snomedct(Text=text)

    for entity in response["Entities"]:
        print(f"\nEntity: '{entity['Text']}' [{entity['Category']}]")
        for concept in entity.get("SNOMEDCTConcepts", [])[:2]:
            print(f"  SNOMED CT: {concept['Code']} — {concept['Description']} "
                  f"({concept['Score']:.3f})")

infer_snomedct("Patient has acute myocardial infarction.")
# → Entity: 'acute myocardial infarction' [MEDICAL_CONDITION]
# →   SNOMED CT: 57054005 — Acute myocardial infarction (0.998)
# →   SNOMED CT: 304914007 — Acute myocardial infarction of anterior wall (0.012)
```

---

## 📊 So Sánh Ba Chuẩn Mã Hóa

| Chuẩn | Phạm Vi | Dùng Cho |
|---|---|---|
| **ICD-10-CM** | Bệnh, triệu chứng, chấn thương | Thanh toán bảo hiểm, thống kê y tế |
| **RxNorm** | Thuốc, liều lượng, dạng bào chế | Hệ thống dược, kê toa điện tử |
| **SNOMED CT** | Toàn bộ y học (bệnh, thủ thuật, giải phẫu) | EHR, clinical decision support |

---

## 🏗️ Kiến Trúc Ứng Dụng Y Tế Thực Tế

### Pipeline Xử Lý Bệnh Án Điện Tử (EHR Processing)

```
Bác sĩ nhập ghi chú lâm sàng (clinical notes)
      │
      ▼
Amazon Transcribe Medical (nếu giọng nói) → Văn bản
      │
      ▼
Comprehend Medical
      ├──► detect_entities_v2 → Medications, Conditions, Procedures
      ├──► infer_icd10_cm → Mã bệnh cho billing
      ├──► infer_rx_norm → Chuẩn hóa thuốc
      └──► detect_phi → Xác định PHI cần bảo vệ
            │
            ▼
      FHIR Resources (HL7 FHIR — Health Level 7 Fast Healthcare Interoperability Resources)
            │
            ▼
      AWS HealthLake hoặc DynamoDB (lưu trữ có cấu trúc)
```

### Pipeline De-identification Cho Nghiên Cứu

```
Hồ sơ bệnh nhân gốc (có PHI)
      │
      ▼
Comprehend Medical detect_phi
      │
      ▼
De-identification pipeline (che PHI bằng labels)
      │
      ▼
Dữ liệu ẩn danh → S3 (data lake cho nghiên cứu)
      │
      ▼
SageMaker / ML training (an toàn — không có PHI)
```

---

## 🔒 Tuân Thủ Bảo Mật Và Quy Định

### HIPAA Compliance (Tuân Thủ HIPAA)

Comprehend Medical là **HIPAA eligible** (đủ điều kiện HIPAA) — có thể xử lý PHI khi:

1. **BAA** (Business Associate Agreement — Thỏa Thuận Đối Tác Kinh Doanh) được ký với AWS
2. Dữ liệu truyền qua **HTTPS/TLS** (Transport Layer Security — Bảo Mật Tầng Truyền Tải)
3. Dữ liệu tại rest được mã hóa bằng **AWS KMS** (Key Management Service — Dịch Vụ Quản Lý Khóa)
4. Audit log qua **AWS CloudTrail** (Dấu Vết Đám Mây)

### Lưu Ý Quan Trọng

- Comprehend Medical **không lưu trữ** văn bản đầu vào sau khi xử lý xong
- Kết quả là NLP inference — không phải chẩn đoán y tế chính thức
- Luôn có **bác sĩ/chuyên gia y tế review** kết quả trước khi dùng trong quyết định lâm sàng

---

## 💰 Chi Phí Comprehend Medical

| API | Giá | So Với Comprehend Thường |
|---|---|---|
| `detect_entities_v2` | Per 100 ký tự | Đắt hơn ~2-3x |
| `infer_icd10_cm` | Per 100 ký tự | Đắt hơn ~2-3x |
| `infer_rx_norm` | Per 100 ký tự | Đắt hơn ~2-3x |
| `detect_phi` | Per 100 ký tự | Đắt hơn ~2-3x |

> **Mẹo:** Chỉ gọi Comprehend Medical với văn bản thực sự là y tế. Với text thông thường, dùng Comprehend thường để tiết kiệm chi phí.

---

## ❓ Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Comprehend Medical khác Comprehend thường thế nào?**

> Comprehend Medical được huấn luyện chuyên biệt cho văn bản y tế — hiểu thuật ngữ lâm sàng, mối quan hệ thuốc-liều-tần suất, và map sang chuẩn y tế quốc tế (ICD-10-CM, RxNorm, SNOMED CT). Comprehend thường không có khả năng này và sẽ bỏ sót nhiều entity y tế chuyên biệt.

**Q: PHI là gì và tại sao cần detect PHI?**

> PHI (Protected Health Information) là thông tin cá nhân gắn với sức khỏe bệnh nhân, được bảo vệ bởi HIPAA. Phát hiện PHI là bước đầu tiên để de-identification (ẩn danh hóa) — cho phép chia sẻ dữ liệu y tế cho nghiên cứu mà không vi phạm quyền riêng tư bệnh nhân.

**Q: ICD-10-CM khác RxNorm thế nào?**

> ICD-10-CM mã hóa **bệnh tật và triệu chứng** (dùng cho billing và thống kê bệnh). RxNorm mã hóa **thuốc và liều lượng** (dùng cho hệ thống dược và e-prescribing). Hai chuẩn bổ trợ cho nhau: bác sĩ chẩn đoán bệnh (ICD) → kê thuốc (RxNorm).

**Q: Có thể dùng Comprehend Medical để chẩn đoán bệnh không?**

> Không. Comprehend Medical là công cụ NLP để trích xuất và cấu trúc hóa thông tin từ văn bản — không phải hệ thống chẩn đoán y tế. Mọi kết quả cần được bác sĩ có chuyên môn xem xét trước khi dùng trong quyết định lâm sàng.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
