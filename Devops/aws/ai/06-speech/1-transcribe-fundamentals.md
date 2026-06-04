# Amazon Transcribe — Nhận Dạng Giọng Nói Tự Động (Cơ Bản)

> Amazon Transcribe là dịch vụ ASR — Automatic Speech Recognition — Nhận Dạng Giọng Nói Tự Động được quản lý hoàn toàn (fully-managed) của AWS. Dịch vụ dùng deep learning (học sâu) để chuyển đổi file âm thanh hoặc luồng audio thời gian thực thành văn bản — kèm theo timestamps (nhãn thời gian), confidence scores (điểm tin cậy) và speaker labels (nhãn người nói). Không cần ML expertise, gọi qua API.

---

## 🎯 Amazon Transcribe Làm Được Gì?

| Tính Năng | Mô Tả |
|---|---|
| **Batch Transcription** (Phiên Âm Theo Lô) | Phiên âm file audio từ S3, xử lý bất đồng bộ |
| **Streaming Transcription** (Phiên Âm Luồng Thời Gian Thực) | Phiên âm real-time qua WebSocket hoặc HTTP/2 |
| **Speaker Diarization** (Phân Biệt Người Nói) | Tách và gán nhãn từng người nói |
| **Custom Vocabulary** (Từ Điển Tùy Chỉnh) | Thêm từ chuyên ngành, tên riêng, từ viết tắt |
| **Custom Language Model — CLM** (Mô Hình Ngôn Ngữ Tùy Chỉnh) | Huấn luyện model ngôn ngữ riêng cho domain đặc thù |
| **Vocabulary Filter** (Bộ Lọc Từ Vựng) | Lọc/thay thế từ nhạy cảm, tục tĩu |
| **PII Redaction** (Ẩn Danh Thông Tin Cá Nhân) | Tự động che/xóa PII trong transcript |
| **Language Identification** (Nhận Dạng Ngôn Ngữ Tự Động) | Tự phát hiện ngôn ngữ khi không biết trước |
| **Multi-language Identification** (Nhận Dạng Đa Ngôn Ngữ) | Phát hiện nhiều ngôn ngữ trong cùng một audio |
| **Content Redaction** (Ẩn Danh Nội Dung) | Che số thẻ tín dụng, SSN, số điện thoại trong transcript |

---

## 📦 1. Batch Transcription (Phiên Âm Theo Lô)

Dùng khi audio đã được ghi sẵn (pre-recorded) và lưu trên S3. Job chạy bất đồng bộ — không cần chờ blocking.

### Luồng Xử Lý

```
File audio trên S3
      │
      ▼
StartTranscriptionJob (API call)
      │
      ▼
Transcribe xử lý nền (vài phút đến vài giờ tùy độ dài)
      │
      ▼
TranscriptionJobStatus = COMPLETED
      │
      ▼
Transcript JSON lưu vào S3 (OutputBucketName)
```

### Code Mẫu — Khởi Động Job

```python
import boto3
import time

transcribe = boto3.client("transcribe", region_name="us-east-1")

def start_batch_transcription(
    audio_s3_uri: str,
    job_name: str,
    language_code: str = "en-US",
    output_bucket: str = "my-transcripts-bucket"
) -> str:
    """Khởi động job phiên âm batch từ S3."""
    response = transcribe.start_transcription_job(
        TranscriptionJobName=job_name,
        Media={"MediaFileUri": audio_s3_uri},     # s3://bucket/recording.mp3
        MediaFormat="mp3",                         # mp3, mp4, wav, flac, ogg, amr, webm
        LanguageCode=language_code,
        OutputBucketName=output_bucket,
        Settings={
            "ShowSpeakerLabels": True,             # Bật speaker diarization
            "MaxSpeakerLabels": 5,                 # Tối đa 5 người nói (2-10)
            "ShowAlternatives": True,              # Hiện transcript thay thế
            "MaxAlternatives": 2                   # Tối đa 2 phương án
        }
    )
    return response["TranscriptionJob"]["TranscriptionJobName"]


def wait_for_job(job_name: str, poll_interval: int = 15) -> dict:
    """Polling cho đến khi job hoàn thành hoặc thất bại."""
    while True:
        response = transcribe.get_transcription_job(TranscriptionJobName=job_name)
        job = response["TranscriptionJob"]
        status = job["TranscriptionJobStatus"]

        if status == "COMPLETED":
            return job
        elif status == "FAILED":
            raise RuntimeError(f"Transcription job thất bại: {job.get('FailureReason')}")

        print(f"Trạng thái: {status} — chờ {poll_interval}s...")
        time.sleep(poll_interval)
```

### Đọc Kết Quả Transcript

```python
import json
import urllib.request

def read_transcript(job: dict) -> dict:
    """Đọc nội dung transcript JSON từ URL kết quả."""
    transcript_uri = job["Transcript"]["TranscriptFileUri"]
    with urllib.request.urlopen(transcript_uri) as response:
        return json.loads(response.read())

def parse_transcript(transcript_data: dict) -> str:
    """Trích xuất văn bản đầy đủ từ transcript JSON."""
    return transcript_data["results"]["transcripts"][0]["transcript"]

def parse_words_with_timestamps(transcript_data: dict) -> list:
    """Trích xuất từng từ kèm timestamp và confidence."""
    items = transcript_data["results"]["items"]
    words = []
    for item in items:
        if item["type"] == "pronunciation":    # Bỏ qua punctuation
            words.append({
                "word": item["alternatives"][0]["content"],
                "confidence": float(item["alternatives"][0]["confidence"]),
                "start_time": float(item.get("start_time", 0)),
                "end_time": float(item.get("end_time", 0))
            })
    return words
```

### Phân Biệt Người Nói (Speaker Diarization)

```python
def parse_speaker_segments(transcript_data: dict) -> list:
    """Trích xuất các đoạn văn bản theo từng người nói."""
    segments = transcript_data["results"].get("speaker_labels", {}).get("segments", [])
    result = []
    for segment in segments:
        result.append({
            "speaker": segment["speaker_label"],       # "spk_0", "spk_1"...
            "start_time": float(segment["start_time"]),
            "end_time": float(segment["end_time"])
        })
    return result
```

### Định Dạng File Audio Được Hỗ Trợ

| Định Dạng | Extension | Ghi Chú |
|---|---|---|
| MP3 | .mp3 | Phổ biến nhất, lossy compression |
| MP4 | .mp4 | Video có audio |
| WAV | .wav | Lossless, file lớn |
| FLAC | .flac | Lossless, nén tốt hơn WAV |
| OGG | .ogg | Dùng cho WebRTC, browser |
| AMR | .amr | Phổ biến trên điện thoại di động |
| WebM | .webm | Dùng trên browser |

---

## 🔴 2. Streaming Transcription (Phiên Âm Thời Gian Thực)

Dùng khi cần phiên âm ngay lập tức — livestream, cuộc gọi, voice assistant.

### Giao Thức Kết Nối

- **WebSocket**: Kết nối liên tục hai chiều — phù hợp cho ứng dụng web/mobile
- **HTTP/2 Streaming**: Dùng trong server-side applications với AWS SDK

### Code Mẫu — WebSocket Streaming (boto3)

```python
import asyncio
import sounddevice as sd
import boto3
from amazon_transcribe.client import TranscribeStreamingClient
from amazon_transcribe.handlers import TranscriptResultStreamHandler
from amazon_transcribe.model import TranscriptEvent

class MyEventHandler(TranscriptResultStreamHandler):
    """Xử lý sự kiện khi nhận được kết quả transcript."""

    async def handle_transcript_event(self, transcript_event: TranscriptEvent):
        results = transcript_event.transcript.results
        for result in results:
            for alt in result.alternatives:
                if not result.is_partial:        # Chỉ lấy kết quả cuối cùng (không phải intermediate)
                    print(f"[FINAL] {alt.transcript}")
                else:
                    print(f"[PARTIAL] {alt.transcript}", end="\r")


async def stream_microphone():
    """Phiên âm thời gian thực từ microphone."""
    client = TranscribeStreamingClient(region="us-east-1")

    stream = await client.start_stream_transcription(
        language_code="en-US",
        media_sample_rate_hz=16000,
        media_encoding="pcm"
    )

    async def send_audio():
        loop = asyncio.get_event_loop()
        input_queue = asyncio.Queue()

        def callback(indata, frame_count, time_info, status):
            loop.call_soon_threadsafe(input_queue.put_nowait, bytes(indata))

        with sd.RawInputStream(
            samplerate=16000,
            channels=1,
            dtype="int16",
            callback=callback
        ):
            while True:
                chunk = await input_queue.get()
                await stream.input_stream.send_audio_event(audio_chunk=chunk)

    handler = MyEventHandler(stream.output_stream)
    await asyncio.gather(send_audio(), handler.handle_events())
```

### Latency (Độ Trễ) Streaming

- **Partial results** (Kết Quả Tạm Thời): Trả về sau ~300ms — hiển thị real-time nhưng có thể thay đổi
- **Final results** (Kết Quả Cuối Cùng): Sau khi Transcribe xác định kết thúc utterance (phát ngôn) — chính xác hơn
- Cấu hình `EnablePartialResultsStabilization=True` để ổn định kết quả tạm thời (giảm flicker)

---

## 📖 3. Custom Vocabulary (Từ Điển Tùy Chỉnh)

Custom Vocabulary — Từ Điển Tùy Chỉnh cho phép thêm từ không có trong dictionary mặc định: tên thương hiệu, thuật ngữ kỹ thuật, tên người, từ viết tắt nội bộ.

### Hai Loại Custom Vocabulary

| Loại | Mô Tả | Khi Nào Dùng |
|---|---|---|
| **Vocabulary Table** (Bảng Từ Vựng) | File .txt hoặc .csv với Phrase, IPA, SoundsLike, DisplayAs | Cần kiểm soát chi tiết cách phát âm |
| **Vocabulary List** (Danh Sách Từ Vựng) | Danh sách từ/cụm từ đơn giản | Nhanh hơn, ít cấu hình hơn |

### Tạo Custom Vocabulary (Vocabulary Table)

Tạo file `vocabulary.txt` dạng CSV có header:

```
Phrase	IPA	SoundsLike	DisplayAs
AWS	eɪ dʌbljuː ɛs		AWS
boto3	boʊtoʊ θriː		boto3
SageMaker	seɪdʒmeɪkər		SageMaker
Nguyen		noo-yen	Nguyễn
TTNT			TTNT
```

- **Phrase**: Từ/cụm từ cần nhận dạng (viết thường)
- **IPA**: Ký hiệu phiên âm quốc tế (International Phonetic Alphabet — Bảng Ký Hiệu Ngữ Âm Quốc Tế)
- **SoundsLike**: Phát âm tương tự theo từ thông thường (dùng khi không biết IPA)
- **DisplayAs**: Cách hiển thị trong transcript (hữu ích cho viết hoa, ký tự đặc biệt)

### Code Mẫu — Tạo và Dùng Custom Vocabulary

```python
def create_custom_vocabulary(vocabulary_name: str, vocabulary_s3_uri: str, language_code: str = "en-US") -> None:
    """Tạo custom vocabulary từ file trên S3."""
    transcribe.create_vocabulary(
        VocabularyName=vocabulary_name,
        LanguageCode=language_code,
        VocabularyFileUri=vocabulary_s3_uri     # s3://bucket/vocabulary.txt
    )
    print(f"Custom vocabulary '{vocabulary_name}' đang được tạo...")


def start_transcription_with_vocabulary(
    audio_s3_uri: str,
    job_name: str,
    vocabulary_name: str,
    language_code: str = "en-US"
) -> str:
    """Phiên âm với custom vocabulary."""
    response = transcribe.start_transcription_job(
        TranscriptionJobName=job_name,
        Media={"MediaFileUri": audio_s3_uri},
        MediaFormat="mp3",
        LanguageCode=language_code,
        Settings={
            "VocabularyName": vocabulary_name     # Áp dụng custom vocabulary
        }
    )
    return response["TranscriptionJob"]["TranscriptionJobName"]
```

---

## 🧠 4. Custom Language Model — CLM (Mô Hình Ngôn Ngữ Tùy Chỉnh)

Custom Language Model — CLM — Mô Hình Ngôn Ngữ Tùy Chỉnh là bước tiếp theo sau Custom Vocabulary. Thay vì chỉ thêm từ vào dictionary, CLM huấn luyện lại một phần model ngôn ngữ trên dữ liệu văn bản của domain bạn — cải thiện đáng kể accuracy (độ chính xác) cho nội dung đặc thù.

### CLM vs Custom Vocabulary — Khi Nào Dùng Cái Nào?

| Tiêu Chí | Custom Vocabulary | Custom Language Model (CLM) |
|---|---|---|
| **Mục đích** | Thêm từ mới vào dictionary | Cải thiện context understanding (hiểu ngữ cảnh) |
| **Dữ liệu cần** | Danh sách từ (tối thiểu vài từ) | Văn bản domain (tối thiểu vài MB) |
| **Thời gian tạo** | Vài phút | Vài giờ (training) |
| **Chi phí** | Miễn phí (chỉ trả phí transcription) | Phí training theo giờ |
| **Hiệu quả** | Tốt cho từ/tên riêng lẻ | Tốt cho domain có ngữ pháp/context đặc thù |
| **Kết hợp** | Dùng riêng lẻ | Có thể kết hợp với Custom Vocabulary |

### Dữ Liệu Huấn Luyện CLM

- **Định dạng**: File .txt (plain text, UTF-8), mỗi câu/đoạn văn bản một dòng
- **Khuyến nghị**: Ít nhất vài MB văn bản; càng nhiều dữ liệu đại diện cho domain càng tốt
- **Nguồn dữ liệu tốt**: Tài liệu nội bộ, script, transcript đã được review, báo cáo ngành
- **Không dùng**: Văn bản không liên quan, văn bản chất lượng kém

```python
def create_custom_language_model(
    model_name: str,
    training_data_s3_uri: str,
    base_model_name: str = "NarrowBand",
    language_code: str = "en-US",
    data_access_role_arn: str = "arn:aws:iam::123456789012:role/TranscribeAccess"
) -> None:
    """Tạo Custom Language Model từ dữ liệu văn bản trên S3."""
    transcribe.create_language_model(
        LanguageCode=language_code,
        BaseModelName=base_model_name,          # "NarrowBand" (8kHz — điện thoại) hoặc "WideBand" (16kHz)
        ModelName=model_name,
        InputDataConfig={
            "S3Uri": training_data_s3_uri,       # s3://bucket/training-texts/
            "DataAccessRoleArn": data_access_role_arn
        }
    )


def transcribe_with_clm(
    audio_s3_uri: str,
    job_name: str,
    clm_name: str,
    vocabulary_name: str | None = None
) -> str:
    """Phiên âm với Custom Language Model (kết hợp với Custom Vocabulary tùy chọn)."""
    settings = {"LanguageModelName": clm_name}
    if vocabulary_name:
        settings["VocabularyName"] = vocabulary_name

    response = transcribe.start_transcription_job(
        TranscriptionJobName=job_name,
        Media={"MediaFileUri": audio_s3_uri},
        MediaFormat="mp3",
        LanguageCode="en-US",
        ModelSettings=settings
    )
    return response["TranscriptionJob"]["TranscriptionJobName"]
```

### Base Model (Mô Hình Nền)

| Base Model | Sample Rate (Tần Số Lấy Mẫu) | Phù Hợp |
|---|---|---|
| `NarrowBand` | 8kHz | Điện thoại, hệ thống IVR — chất lượng âm thanh thấp |
| `WideBand` | 16kHz | Microphone, meeting room — chất lượng âm thanh tốt |

---

## 🔒 5. PII Redaction (Ẩn Danh Thông Tin Cá Nhân)

PII — Personally Identifiable Information — Thông Tin Định Danh Cá Nhân: tên, địa chỉ, email, số điện thoại, số thẻ tín dụng, SSN (Social Security Number — Số An Sinh Xã Hội)...

Transcribe có thể **tự động phát hiện và che** các PII trong transcript — thay bằng `[PII]` hoặc xóa hoàn toàn.

```python
def start_transcription_with_pii_redaction(
    audio_s3_uri: str,
    job_name: str,
    language_code: str = "en-US"
) -> str:
    """Phiên âm với tự động ẩn danh thông tin PII."""
    response = transcribe.start_transcription_job(
        TranscriptionJobName=job_name,
        Media={"MediaFileUri": audio_s3_uri},
        MediaFormat="mp3",
        LanguageCode=language_code,
        ContentRedaction={
            "RedactionType": "PII",
            "RedactionOutput": "redacted",        # "redacted" hoặc "redacted_and_unredacted"
            "PiiEntityTypes": [                   # Danh sách loại PII cần ẩn (bỏ trống = tất cả)
                "NAME",
                "PHONE",
                "EMAIL",
                "CREDIT_DEBIT_NUMBER",
                "SSN",
                "ADDRESS"
            ]
        }
    )
    return response["TranscriptionJob"]["TranscriptionJobName"]
```

### Các Loại PII Được Hỗ Trợ

| Loại PII | Mô Tả |
|---|---|
| `NAME` | Tên người |
| `PHONE` | Số điện thoại |
| `EMAIL` | Địa chỉ email |
| `ADDRESS` | Địa chỉ thực |
| `SSN` | Social Security Number |
| `CREDIT_DEBIT_NUMBER` | Số thẻ tín dụng/ghi nợ |
| `CREDIT_DEBIT_CVV` | Mã CVV thẻ |
| `CREDIT_DEBIT_EXPIRY` | Ngày hết hạn thẻ |
| `PIN` | Mã PIN |
| `BANK_ACCOUNT_NUMBER` | Số tài khoản ngân hàng |
| `DATE_OF_BIRTH` | Ngày sinh |
| `PASSPORT_NUMBER` | Số hộ chiếu |
| `DRIVER_ID` | Số bằng lái xe |

---

## 🌍 6. Language Identification (Tự Động Nhận Dạng Ngôn Ngữ)

Khi không biết trước ngôn ngữ của audio, bật `IdentifyLanguage=True` — Transcribe tự phát hiện trong số các ngôn ngữ ứng viên được chỉ định.

```python
def start_transcription_auto_language(
    audio_s3_uri: str,
    job_name: str,
    candidate_languages: list[str] | None = None
) -> str:
    """Phiên âm với tự động nhận dạng ngôn ngữ."""
    params = {
        "TranscriptionJobName": job_name,
        "Media": {"MediaFileUri": audio_s3_uri},
        "MediaFormat": "mp3",
        "IdentifyLanguage": True,
        "OutputBucketName": "my-transcripts-bucket"
    }
    if candidate_languages:
        # Giới hạn phạm vi tìm kiếm (tốt hơn về chi phí và tốc độ)
        params["LanguageOptions"] = candidate_languages   # ["en-US", "vi-VN", "ja-JP"]

    return transcribe.start_transcription_job(**params)["TranscriptionJob"]["TranscriptionJobName"]
```

### Multi-language Identification (Đa Ngôn Ngữ Trong Một Audio)

```python
# Phát hiện nhiều ngôn ngữ trong cùng một file audio (VD: cuộc họp quốc tế)
response = transcribe.start_transcription_job(
    TranscriptionJobName="multilang-job",
    Media={"MediaFileUri": "s3://bucket/mixed-language-audio.mp3"},
    MediaFormat="mp3",
    IdentifyMultipleLanguages=True,
    LanguageOptions=["en-US", "vi-VN", "ja-JP", "zh-CN"]
)
```

---

## 📊 Quota (Hạn Mức) Và Giới Hạn

| Giới Hạn | Giá Trị |
|---|---|
| Thời lượng audio tối đa (batch) | 4 giờ / job |
| Kích thước file audio tối đa | 2 GB |
| Ngôn ngữ hỗ trợ | 100+ |
| Concurrent jobs (job đồng thời) | 250 (có thể tăng theo request) |
| Custom Vocabulary tối đa | 100 vocabularies / tài khoản |
| Custom Language Model tối đa | 10 models / tài khoản |
| Streaming — thời lượng tối đa | 4 giờ liên tục |

---

## 🏗️ Kiến Trúc Ứng Dụng Thực Tế

### Pipeline Tạo Phụ Đề Tự Động

```
Video upload → S3
      │ (S3 Event Notification)
      ▼
Lambda: StartTranscriptionJob
      │
      ▼
EventBridge: Transcribe job COMPLETED event
      │
      ▼
Lambda: Đọc transcript JSON → Convert sang .srt / .vtt
      │
      ▼
S3 (lưu subtitle file)
      │
      ▼
CloudFront + MediaConvert (gắn subtitle vào video)
```

### Pipeline Meeting Notes (Ghi Chú Cuộc Họp Tự Động)

```
Audio recording cuộc họp (từ Chime / Zoom)
      │
      ▼
S3 → Transcribe batch (speaker diarization bật)
      │
      ▼
Transcript JSON với speaker labels (spk_0, spk_1...)
      │
      ▼
Bedrock (Claude): "Tóm tắt cuộc họp, liệt kê action items"
      │
      ▼
Email/Slack thông báo kết quả
```

---

## ❓ Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Custom Vocabulary khác Custom Language Model thế nào?**

> Custom Vocabulary thêm từ mới vào dictionary để Transcribe nhận dạng đúng chính tả — nhanh, không cần training data. Custom Language Model huấn luyện lại model ngôn ngữ trên văn bản của domain bạn để model hiểu context tốt hơn — hiệu quả hơn với domain đặc thù nhưng cần dữ liệu và thời gian training.

**Q: Khi nào dùng Streaming thay vì Batch?**

> Streaming khi cần phiên âm ngay lập tức trong khi audio đang diễn ra — voice assistant, live captioning (phụ đề trực tiếp), real-time customer service. Batch khi audio đã được ghi xong và lưu trên S3 — podcast, video, meeting recording — nhanh hơn và rẻ hơn.

**Q: Speaker Diarization hoạt động thế nào?**

> Transcribe phân tích sự thay đổi trong đặc trưng giọng nói (acoustic features) để phát hiện khi nào người nói thay đổi. Kết quả gán nhãn `spk_0`, `spk_1`... — không biết tên người nói, chỉ phân biệt "người nói này" và "người nói khác". Hỗ trợ 2-10 người nói.

**Q: PII Redaction trong Transcribe có đảm bảo che hết mọi PII không?**

> Không đảm bảo 100%. PII Redaction dùng ML để phát hiện — có thể bỏ sót hoặc nhận dạng nhầm. Với yêu cầu compliance nghiêm ngặt (GDPR, HIPAA), nên kết hợp với human review (xem xét của con người) cho dữ liệu nhạy cảm cao.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
