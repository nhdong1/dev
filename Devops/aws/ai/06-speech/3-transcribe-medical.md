# Amazon Transcribe Medical — Phiên Âm Y Tế Chuyên Biệt

> Amazon Transcribe Medical — Phiên Âm Y Tế là dịch vụ ASR chuyên biệt được huấn luyện riêng với từ điển y tế đồ sộ: tên thuốc, thuật ngữ lâm sàng, quy trình phẫu thuật, tên bệnh, chỉ số xét nghiệm... Dịch vụ đáp ứng tiêu chuẩn **HIPAA** — Health Insurance Portability and Accountability Act — Luật Trách Nhiệm Giải Trình và Khả Năng Chuyển Đổi Bảo Hiểm Y Tế (Mỹ) — cho phép xử lý **PHI** (Protected Health Information — Thông Tin Y Tế Được Bảo Vệ).

---

## 🏥 Tại Sao Cần Transcribe Medical Riêng?

Transcribe thông thường **không đủ cho y tế** vì:

| Vấn Đề | Ví Dụ | Hệ Quả |
|---|---|---|
| Từ điển y tế chuyên sâu | "metformin", "myocardial infarction", "thoracoscopy" | Nhận dạng sai → sai thuốc, sai chẩn đoán |
| Từ viết tắt y tế | "CABG", "MI", "PCI", "COPD" | Phiên âm thành chữ thường, mất nghĩa |
| Tên thuốc phức tạp | "atorvastatin", "hydrochlorothiazide" | Nhận dạng sai tên thuốc → nguy hiểm |
| Giọng đọc chuyên nghiệp | Bác sĩ đọc nhanh, ít ngừng nghỉ | WER (Word Error Rate — Tỉ Lệ Lỗi Từ) cao hơn |
| Yêu cầu HIPAA compliance | Dữ liệu bệnh nhân phải được bảo vệ | Transcribe thường không cam kết HIPAA |

Transcribe Medical giải quyết tất cả các vấn đề trên.

---

## 🎯 Use Cases (Trường Hợp Sử Dụng) Chính

| Use Case | Mô Tả |
|---|---|
| **Clinical Documentation** (Ghi Chú Lâm Sàng) | Bác sĩ đọc to ghi chú khám bệnh, tự động chuyển thành văn bản |
| **Dictation** (Đọc Chính Tả) | Tự động hóa ghi chú bác sĩ thay vì gõ phím |
| **EHR Integration** (Tích Hợp Hồ Sơ Bệnh Án Điện Tử) | Nạp trực tiếp vào EHR — Electronic Health Record — Hồ Sơ Sức Khỏe Điện Tử |
| **Medical Education** (Giáo Dục Y Khoa) | Phiên âm bài giảng y khoa, hội thảo chuyên khoa |
| **Telemedicine** (Y Tế Từ Xa) | Phiên âm cuộc tư vấn từ xa real-time |
| **Radiology Reports** (Báo Cáo Hình Ảnh Y Khoa) | Bác sĩ chẩn đoán hình ảnh đọc kết quả X-quang, CT, MRI |

---

## 🔬 Specialty Models (Mô Hình Chuyên Khoa)

Transcribe Medical cung cấp model chuyên biệt cho các chuyên khoa khác nhau:

| Specialty (Chuyên Khoa) | Mô Tả | Từ Điển Đặc Trưng |
|---|---|---|
| `PRIMARYCARE` | Y tế tổng quát, khám tổng quát | Từ điển y tế tổng quát |
| `CARDIOLOGY` | Tim mạch | ECG, arrhythmia, stents, CABG, ejection fraction |
| `NEUROLOGY` | Thần kinh học | MRI brain, stroke, seizure, cognitive assessment |
| `ONCOLOGY` | Ung thư học | Chemotherapy, biopsy, tumor markers, staging |
| `RADIOLOGY` | Chẩn đoán hình ảnh | X-ray findings, CT, MRI, PET scan, DICOM |
| `UROLOGY` | Tiết niệu học | PSA, cystoscopy, nephrolithiasis |

> **Lưu ý:** Chọn đúng specialty giúp tăng độ chính xác đáng kể. Nếu không chắc, dùng `PRIMARYCARE` làm fallback.

---

## 📦 1. Batch Medical Transcription (Phiên Âm Y Tế Theo Lô)

```python
import boto3
import time

transcribe = boto3.client("transcribe", region_name="us-east-1")

def start_medical_transcription(
    audio_s3_uri: str,
    job_name: str,
    output_bucket: str,
    specialty: str = "PRIMARYCARE",
    type_: str = "DICTATION",
    language_code: str = "en-US"
) -> str:
    """Khởi động job phiên âm y tế batch.
    
    specialty: PRIMARYCARE | CARDIOLOGY | NEUROLOGY | ONCOLOGY | RADIOLOGY | UROLOGY
    type_: DICTATION (bác sĩ đọc chính tả) | CONVERSATION (cuộc trò chuyện bác sĩ-bệnh nhân)
    """
    response = transcribe.start_medical_transcription_job(
        MedicalTranscriptionJobName=job_name,
        Media={"MediaFileUri": audio_s3_uri},
        MediaFormat="mp3",
        LanguageCode=language_code,            # Hiện chỉ hỗ trợ "en-US"
        OutputBucketName=output_bucket,
        Specialty=specialty,
        Type=type_,
        Settings={
            "ShowSpeakerLabels": True,
            "MaxSpeakerLabels": 2              # Thường 2: bác sĩ và bệnh nhân
        }
    )
    return response["MedicalTranscriptionJob"]["MedicalTranscriptionJobName"]


def wait_for_medical_job(job_name: str) -> dict:
    """Chờ job hoàn thành với polling."""
    while True:
        response = transcribe.get_medical_transcription_job(MedicalTranscriptionJobName=job_name)
        job = response["MedicalTranscriptionJob"]
        status = job["MedicalTranscriptionJobStatus"]

        if status == "COMPLETED":
            print(f"Hoàn thành! Transcript tại: {job['Transcript']['TranscriptFileUri']}")
            return job
        elif status == "FAILED":
            raise RuntimeError(f"Job thất bại: {job.get('FailureReason')}")

        print(f"Trạng thái: {status}...")
        time.sleep(15)
```

### Phân Biệt Type: DICTATION vs CONVERSATION

| Type | Mô Tả | Khi Nào Dùng |
|---|---|---|
| `DICTATION` | Một người đọc liên tục (như đọc chính tả) | Bác sĩ đọc báo cáo, ghi chú, kết quả xét nghiệm |
| `CONVERSATION` | Nhiều người nói xen kẽ | Cuộc khám bệnh với bệnh nhân, telemedicine |

---

## 🔴 2. Real-time Medical Streaming (Phiên Âm Y Tế Thời Gian Thực)

Dùng trong hệ thống EHR real-time — bác sĩ nói, văn bản hiện ngay trên màn hình:

```python
import asyncio
from amazon_transcribe.client import TranscribeStreamingClient
from amazon_transcribe.handlers import TranscriptResultStreamHandler
from amazon_transcribe.model import TranscriptEvent

class MedicalTranscriptHandler(TranscriptResultStreamHandler):
    """Xử lý kết quả phiên âm y tế real-time."""

    def __init__(self, output_stream, ehr_client=None):
        super().__init__(output_stream)
        self.ehr_client = ehr_client        # Client tích hợp EHR
        self.final_transcript = []

    async def handle_transcript_event(self, transcript_event: TranscriptEvent):
        results = transcript_event.transcript.results

        for result in results:
            for alt in result.alternatives:
                if not result.is_partial:
                    text = alt.transcript
                    self.final_transcript.append(text)
                    print(f"[FINAL] {text}")

                    # Tự động lưu vào EHR (tích hợp)
                    if self.ehr_client:
                        await self.ehr_client.append_note(text)
                else:
                    print(f"[LIVE] {alt.transcript}", end="\r")


async def stream_medical_dictation(audio_generator, specialty: str = "PRIMARYCARE"):
    """Phiên âm y tế real-time từ microphone/audio stream."""
    client = TranscribeStreamingClient(region="us-east-1")

    stream = await client.start_medical_stream_transcription(
        language_code="en-US",
        media_sample_rate_hz=16000,
        media_encoding="pcm",
        specialty=specialty,
        type="DICTATION",
        enable_channel_identification=False,
        number_of_channels=1
    )

    handler = MedicalTranscriptHandler(stream.output_stream)
    await asyncio.gather(
        audio_generator(stream.input_stream),
        handler.handle_events()
    )

    return "\n".join(handler.final_transcript)
```

---

## 🔒 3. PHI Detection và Redaction (Phát Hiện và Ẩn Danh Thông Tin Y Tế)

**PHI** — Protected Health Information — Thông Tin Y Tế Được Bảo Vệ bao gồm bất kỳ thông tin nào có thể nhận dạng bệnh nhân cùng với thông tin sức khỏe: tên, ngày sinh, địa chỉ, số điện thoại, số hồ sơ bệnh nhân, số bảo hiểm y tế...

Transcribe Medical tự động phát hiện PHI trong transcript.

```python
def start_medical_transcription_with_phi_redaction(
    audio_s3_uri: str,
    job_name: str,
    output_bucket: str
) -> str:
    """Phiên âm y tế với tự động ẩn danh PHI — tuân thủ HIPAA."""
    response = transcribe.start_medical_transcription_job(
        MedicalTranscriptionJobName=job_name,
        Media={"MediaFileUri": audio_s3_uri},
        MediaFormat="mp3",
        LanguageCode="en-US",
        OutputBucketName=output_bucket,
        Specialty="PRIMARYCARE",
        Type="CONVERSATION",
        ContentIdentificationType="PHI",    # Đánh dấu PHI trong transcript
        # Hoặc dùng ContentRedactionType="PHI" để ẩn hoàn toàn
    )
    return response["MedicalTranscriptionJob"]["MedicalTranscriptionJobName"]
```

### PHI Identification vs PHI Redaction

| Tùy Chọn | Mô Tả | Khi Nào Dùng |
|---|---|---|
| `ContentIdentificationType="PHI"` | Đánh dấu (tag) các đoạn PHI trong transcript nhưng không xóa | Khi cần biết đâu là PHI để review thủ công |
| `ContentRedactionType="PHI"` | Thay thế PHI bằng `[PHI]` — xóa khỏi transcript | Khi cần transcript an toàn cho downstream processing |

### Ví Dụ Kết Quả PHI Redaction

**Transcript gốc:**
```
Patient John Smith, date of birth March 15, 1978, SSN 123-45-6789, 
presented with chest pain. Contact number 555-0123.
```

**Transcript sau PHI Redaction:**
```
Patient [PHI], date of birth [PHI], SSN [PHI], 
presented with chest pain. Contact number [PHI].
```

---

## 📋 4. Medical Custom Vocabulary (Từ Điển Y Tế Tùy Chỉnh)

Tương tự Custom Vocabulary của Transcribe thường nhưng dành riêng cho Transcribe Medical — thêm tên thuốc nội bộ, thuật ngữ nghiên cứu, tên quy trình đặc thù của bệnh viện:

```python
def create_medical_vocabulary(
    vocabulary_name: str,
    vocabulary_s3_uri: str,
    language_code: str = "en-US"
) -> None:
    """Tạo Medical Custom Vocabulary."""
    transcribe.create_medical_vocabulary(
        VocabularyName=vocabulary_name,
        LanguageCode=language_code,
        VocabularyFileUri=vocabulary_s3_uri    # s3://bucket/medical-vocab.txt
    )
```

### File Từ Điển Y Tế Mẫu

```
Phrase	IPA	SoundsLike	DisplayAs
Xarelto		ZAR-el-toh	Xarelto
bivalirudin		by-val-ih-ROO-din	bivalirudin
TAVR		TAY-ver	TAVR
Watchman		watch-man	Watchman
HbA1c		H-B-A-1-C	HbA1c
eGFR		E-G-F-R	eGFR
```

---

## 🏗️ Kiến Trúc Ứng Dụng Y Tế Thực Tế

### Hệ Thống Ghi Chú Lâm Sàng Tự Động (Clinical Documentation System)

```
Bác sĩ nói vào microphone (trong phòng khám)
      │
      ▼
Transcribe Medical Streaming (real-time, specialty=PRIMARYCARE)
      │
      ▼
Transcript real-time hiển thị trên màn hình bác sĩ
      │ (bác sĩ review và confirm)
      ▼
Comprehend Medical (trích xuất MEDICATION, CONDITION, PROCEDURE, TEST_TREATMENT_PROCEDURE)
      │
      ├──► Map MEDICATION → RxNorm (mã thuốc chuẩn quốc tế)
      ├──► Map CONDITION → ICD-10-CM (mã bệnh quốc tế)
      └──► Structured data → EHR system (FHIR — Fast Healthcare Interoperability Resources)
```

### Pipeline Xử Lý Hồ Sơ Audio Lưu Trữ

```
Kho lưu trữ recording (S3 — mã hóa AES-256)
      │
      ▼
Transcribe Medical Batch (Specialty theo loại audio)
      │
      ▼
PHI Identification → Human review PHI trước khi dùng
      │
      ▼
PHI Redaction → Transcript ẩn danh
      │
      ▼
Comprehend Medical → Entity extraction (trích xuất thực thể)
      │
      ▼
Data warehouse (Redshift — đã ẩn danh) → Research analytics
```

---

## 🔐 Bảo Mật Và HIPAA Compliance

### Yêu Cầu HIPAA Khi Dùng Transcribe Medical

| Yêu Cầu | Cách Thực Hiện |
|---|---|
| **Encryption in transit** (Mã Hóa Khi Truyền) | TLS 1.2+ tự động — không cần cấu hình |
| **Encryption at rest** (Mã Hóa Khi Lưu) | S3 SSE-KMS (Server-Side Encryption với KMS) cho audio và transcript |
| **Access control** (Kiểm Soát Truy Cập) | IAM policy chặt chẽ — least privilege (quyền tối thiểu cần thiết) |
| **Audit logging** (Ghi Log Kiểm Toán) | AWS CloudTrail bật để ghi log mọi API call |
| **Data residency** (Lưu Trữ Dữ Liệu Tại Chỗ) | Chọn AWS Region phù hợp (VD: dữ liệu Mỹ — us-east-1) |
| **BAA** (Business Associate Agreement — Thỏa Thuận Đối Tác Kinh Doanh) | Ký BAA với AWS trước khi xử lý PHI |

### Cấu Hình S3 An Toàn Cho Dữ Liệu Y Tế

```python
import boto3

s3 = boto3.client("s3")

def create_hipaa_compliant_bucket(bucket_name: str, kms_key_id: str) -> None:
    """Tạo S3 bucket tuân thủ HIPAA cho audio y tế."""
    s3.create_bucket(Bucket=bucket_name)

    # Bật mã hóa mặc định với KMS key
    s3.put_bucket_encryption(
        Bucket=bucket_name,
        ServerSideEncryptionConfiguration={
            "Rules": [{
                "ApplyServerSideEncryptionByDefault": {
                    "SSEAlgorithm": "aws:kms",
                    "KMSMasterKeyID": kms_key_id
                },
                "BucketKeyEnabled": True   # Giảm chi phí KMS API calls
            }]
        }
    )

    # Chặn public access hoàn toàn
    s3.put_public_access_block(
        Bucket=bucket_name,
        PublicAccessBlockConfiguration={
            "BlockPublicAcls": True,
            "IgnorePublicAcls": True,
            "BlockPublicPolicy": True,
            "RestrictPublicBuckets": True
        }
    )

    # Bật versioning (cần cho audit trail)
    s3.put_bucket_versioning(
        Bucket=bucket_name,
        VersioningConfiguration={"Status": "Enabled"}
    )
```

---

## ⚠️ Giới Hạn Quan Trọng

| Giới Hạn | Giá Trị | Ghi Chú |
|---|---|---|
| Ngôn ngữ hỗ trợ | Chỉ tiếng Anh (`en-US`) | Giới hạn lớn nhất — không hỗ trợ tiếng Việt |
| Thời lượng audio tối đa | 4 giờ / job | |
| Kích thước file tối đa | 2 GB | |
| Specialty models | 6 chuyên khoa | Có thể mở rộng trong tương lai |

> **Lưu ý đặc biệt cho ứng dụng tại Việt Nam:** Transcribe Medical hiện **chỉ hỗ trợ tiếng Anh**. Nếu cần phiên âm y tế tiếng Việt, phải dùng Transcribe thông thường với Custom Vocabulary và Custom Language Model — không được HIPAA-certified cho tiếng Việt.

---

## 💰 Chi Phí

| Chế Độ | Giá | So Với Transcribe Thường |
|---|---|---|
| Batch Medical | ~$0.075/phút audio | ~3x đắt hơn |
| Streaming Medical | ~$0.078/phút audio | ~3x đắt hơn |

> **Free Tier:** Transcribe Medical có 60 phút miễn phí/tháng trong 12 tháng đầu (chung với Transcribe thường).

---

## ❓ Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Khi nào dùng Transcribe Medical thay vì Transcribe thường?**

> Khi audio chứa nội dung y tế lâm sàng — ghi chú bác sĩ, toa thuốc, báo cáo chẩn đoán, cuộc khám bệnh. Transcribe Medical có model chuyên biệt với từ điển y tế đồ sộ, cho WER (Word Error Rate) thấp hơn nhiều với thuật ngữ y tế. Ngoài ra, Transcribe Medical được HIPAA-eligible — cần thiết cho xử lý PHI.

**Q: HIPAA compliance trong AWS có nghĩa là gì?**

> HIPAA là luật bảo vệ dữ liệu y tế tại Mỹ. HIPAA-eligible service nghĩa là AWS cam kết ký BAA (Business Associate Agreement) và dịch vụ đáp ứng các control bảo mật HIPAA. Cần: ký BAA với AWS, mã hóa dữ liệu, kiểm soát truy cập, audit logging, và quy trình xử lý PHI theo chuẩn.

**Q: PHI Identification khác PHI Redaction thế nào?**

> Identification đánh dấu (tag) PHI trong transcript nhưng không xóa — dùng khi cần human review để xác nhận. Redaction thay PHI bằng `[PHI]` — dùng khi cần transcript an toàn để downstream processing hoặc lưu trữ không giới hạn.

**Q: Transcribe Medical có hỗ trợ tiếng Việt không?**

> Không — hiện chỉ hỗ trợ `en-US`. Đây là giới hạn quan trọng. Cho ứng dụng y tế tiếng Việt, cần tự xây dựng pipeline tùy chỉnh với Transcribe thông thường + Custom Vocabulary + Custom Language Model.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
