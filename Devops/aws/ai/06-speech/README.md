# Speech AI (AI Giọng Nói) trên AWS — Toàn Diện

> Speech AI — AI Giọng Nói bao gồm hai hướng chính: **STT** (Speech-to-Text — Chuyển Giọng Nói Thành Văn Bản) và **TTS** (Text-to-Speech — Chuyển Văn Bản Thành Giọng Nói). Trên AWS, hai dịch vụ chủ lực là **Amazon Transcribe** (Nhận Dạng Giọng Nói Tự Động) và **Amazon Polly** (Tổng Hợp Giọng Nói). Cả hai đều là AI Services được quản lý hoàn toàn (fully-managed), gọi qua API, không cần ML expertise (chuyên môn học máy).

---

## 📚 Mục Lục Module

| File | Nội Dung | Trạng Thái |
|---|---|---|
| `README.md` | Tổng quan Speech AI trên AWS, so sánh dịch vụ | ✅ |
| `1-transcribe-fundamentals.md` | Batch, Real-time, Custom Vocabulary, Custom Language Model | ✅ |
| `2-transcribe-call-analytics.md` | Call analytics, Sentiment, Categories, Redaction | ✅ |
| `3-transcribe-medical.md` | Medical transcription, Specialty models | ✅ |
| `4-polly.md` | Neural TTS, SSML, Lexicons, Voice cloning | ✅ |

---

## 🎯 Speech AI Là Gì Và Dùng Để Làm Gì?

Speech AI cho phép ứng dụng tương tác với người dùng qua giọng nói — chuyển đổi hai chiều giữa âm thanh và văn bản:

- **STT — Speech-to-Text** (Nhận Dạng Giọng Nói): Chuyển audio (tệp âm thanh hoặc luồng thời gian thực) thành văn bản
- **TTS — Text-to-Speech** (Tổng Hợp Giọng Nói): Chuyển văn bản thành giọng nói tự nhiên
- **Speaker Diarization** (Phân Biệt Người Nói): Xác định và gán nhãn từng người nói trong một đoạn audio
- **Call Analytics** (Phân Tích Cuộc Gọi): Phân tích nội dung, cảm xúc, vấn đề trong các cuộc gọi tổng đài
- **Medical Transcription** (Phiên Âm Y Tế): Nhận dạng giọng nói chuyên biệt cho ngành y tế với thuật ngữ lâm sàng

### Vị Trí Trong Hệ Sinh Thái AWS AI

```
┌─────────────────────────────────────────────────────────────────┐
│  Tầng 3: AI Services (Dịch Vụ AI Được Quản Lý)  ◄── BẠN Ở ĐÂY │
│  Transcribe │ Transcribe Medical │ Transcribe Call Analytics     │
│  Polly (Neural TTS)                                              │
├─────────────────────────────────────────────────────────────────┤
│  Tầng 2: ML Services                                            │
│   Amazon SageMaker (dùng khi cần custom ASR/TTS model phức tạp)│
├─────────────────────────────────────────────────────────────────┤
│  Tầng 1: ML Framework & Infrastructure                          │
│  Kaldi │ ESPnet │ HuggingFace Wav2Vec │ GPU Instances           │
└─────────────────────────────────────────────────────────────────┘
```

**Quy tắc chọn:** Dùng Transcribe/Polly khi yêu cầu nằm trong phạm vi của dịch vụ có sẵn — nhanh, không cần data huấn luyện, không cần ML expertise. Chỉ dùng SageMaker custom khi yêu cầu rất đặc thù (ngôn ngữ hiếm, accent đặc biệt, model nhúng trên thiết bị edge).

---

## 🗂️ So Sánh Dịch Vụ Speech AI Trên AWS

| Dịch Vụ | Mục Đích | Input | Output | Tính Phí |
|---|---|---|---|---|
| **Amazon Transcribe** | STT tổng quát | Audio file (S3) hoặc audio stream | Văn bản + timestamps + confidence | Per giây audio |
| **Amazon Transcribe Medical** | STT chuyên biệt y tế | Audio ghi chú lâm sàng | Văn bản y tế với entity detection | Per giây audio (cao hơn) |
| **Transcribe Call Analytics** | Phân tích cuộc gọi tổng đài | Audio cuộc gọi 2 kênh (agent + customer) | Transcript + sentiment + categories + issues | Per giây audio (cao hơn) |
| **Amazon Polly** | TTS | Văn bản / SSML | Audio stream hoặc file MP3/OGG/PCM | Per ký tự tổng hợp |

### Khi Nào Dùng Cái Nào?

```
Bạn cần xử lý gì?
│
├─ Chuyển file âm thanh thành văn bản (batch)?
│   └─► Transcribe — StartTranscriptionJob (batch)
│
├─ Chuyển giọng nói thành văn bản theo thời gian thực (livestream, call)?
│   └─► Transcribe — StartStreamTranscription (real-time streaming)
│
├─ Âm thanh là cuộc gọi tổng đài / call center?
│   └─► Transcribe Call Analytics — phân tích sentiment, issue, category
│
├─ Âm thanh là ghi chú lâm sàng / y tế?
│   └─► Transcribe Medical — mô hình chuyên biệt y tế
│
├─ Cần chuyển văn bản thành giọng nói (đọc thông báo, voice assistant)?
│   └─► Amazon Polly — Neural TTS, hỗ trợ SSML
│
└─ Cần ASR tùy chỉnh cho ngôn ngữ hiếm hoặc domain cực kỳ đặc thù?
    └─► SageMaker với custom ASR model (Kaldi / HuggingFace Wav2Vec2)
```

---

## 🆚 Transcribe vs Polly — Phân Biệt Ngay

| Tiêu Chí | Amazon Transcribe | Amazon Polly |
|---|---|---|
| **Hướng chuyển đổi** | Âm thanh → Văn bản (STT) | Văn bản → Âm thanh (TTS) |
| **Đầu vào** | Audio file (.mp3, .wav, .flac, .ogg...) | Văn bản thuần hoặc SSML |
| **Đầu ra** | JSON transcript với timestamps | Audio stream (MP3, OGG Vorbis, PCM) |
| **Ngôn ngữ hỗ trợ** | 100+ ngôn ngữ | 30+ ngôn ngữ, 60+ giọng đọc |
| **Tùy chỉnh** | Custom Vocabulary, Custom Language Model | Lexicons, SSML, Neural voice |
| **Free Tier** | 60 phút/tháng (12 tháng đầu) | 5 triệu ký tự/tháng (12 tháng đầu) |

---

## 🆚 Transcribe vs Transcribe Medical vs Call Analytics

| Tính Năng | Transcribe | Transcribe Medical | Call Analytics |
|---|---|---|---|
| **Domain** | Tổng quát | Y tế lâm sàng | Tổng đài / Contact center |
| **Từ điển chuyên ngành** | Thông thường | Thuật ngữ y tế, tên thuốc, quy trình | Không |
| **Phân tích cảm xúc** | Không | Không | ✅ (theo đoạn) |
| **Redaction — Ẩn Danh** | ✅ PII Redaction | ✅ PHI Redaction | ✅ PII Redaction |
| **Phân biệt agent/customer** | Không (diarization chung) | Không | ✅ (2 kênh âm thanh) |
| **Issue/Action detection** | Không | Không | ✅ |
| **Giá** | Thấp nhất | Trung bình | Cao nhất |

> **PII** — Personally Identifiable Information — Thông Tin Định Danh Cá Nhân (tên, số điện thoại, số thẻ tín dụng...)
> **PHI** — Protected Health Information — Thông Tin Y Tế Được Bảo Vệ (theo chuẩn HIPAA)

---

## 🚀 Quick Start — Gọi Transcribe & Polly

### Cài Đặt Client (boto3)

```python
import boto3

transcribe = boto3.client("transcribe", region_name="us-east-1")
polly      = boto3.client("polly", region_name="us-east-1")
```

### Phiên Âm Batch (Batch Transcription)

```python
def start_transcription(audio_s3_uri: str, job_name: str, language_code: str = "vi-VN") -> str:
    """Khởi động job phiên âm batch từ file audio trên S3."""
    response = transcribe.start_transcription_job(
        TranscriptionJobName=job_name,
        Media={"MediaFileUri": audio_s3_uri},        # s3://bucket/audio.mp3
        MediaFormat="mp3",
        LanguageCode=language_code,                  # "vi-VN", "en-US", "ja-JP"...
        OutputBucketName="my-transcripts-bucket"
    )
    return response["TranscriptionJob"]["TranscriptionJobName"]

# Kiểm tra trạng thái job
def get_transcription_result(job_name: str) -> str | None:
    response = transcribe.get_transcription_job(TranscriptionJobName=job_name)
    status = response["TranscriptionJob"]["TranscriptionJobStatus"]
    if status == "COMPLETED":
        return response["TranscriptionJob"]["Transcript"]["TranscriptFileUri"]
    return None
```

### Tổng Hợp Giọng Nói (Text-to-Speech)

```python
def synthesize_speech(text: str, voice_id: str = "Joanna", output_format: str = "mp3") -> bytes:
    """Chuyển văn bản thành giọng nói, trả về bytes audio."""
    response = polly.synthesize_speech(
        Text=text,
        VoiceId=voice_id,           # "Joanna" (Anh-Mỹ), "Amy" (Anh-Anh), "Zhiyu" (Trung)...
        OutputFormat=output_format   # "mp3", "ogg_vorbis", "pcm"
    )
    return response["AudioStream"].read()

audio_bytes = synthesize_speech("Xin chào! Chào mừng bạn đến với AWS Polly.")
with open("output.mp3", "wb") as f:
    f.write(audio_bytes)
```

---

## 🔑 Khái Niệm Chung Cần Nắm

### ASR — Automatic Speech Recognition (Nhận Dạng Giọng Nói Tự Động)

ASR là công nghệ lõi đằng sau Transcribe — model deep learning phân tích sóng âm (audio waveform) và map sang chuỗi ký tự ngôn ngữ. Transcribe dùng kiến trúc sequence-to-sequence với attention mechanism (cơ chế chú ý), được huấn luyện trên hàng triệu giờ audio.

### Confidence Score (Điểm Tin Cậy)

Mỗi từ trong transcript đều kèm `confidence` (0–1). Giá trị thấp ám chỉ âm thanh không rõ, accent khó hiểu hoặc từ nằm ngoài vocabulary (từ vựng). Dùng ngưỡng confidence để:
- Tô đỏ các từ có confidence thấp trong editor (cho phép người dùng sửa)
- Không tự động xử lý các câu có average confidence dưới 0.7

### Speaker Diarization (Phân Tích Người Nói)

Tách audio thành các đoạn gắn nhãn `spk_0`, `spk_1`, `spk_2`... theo từng người nói. Hữu ích khi phiên âm cuộc họp, phỏng vấn hoặc podcast nhiều người.

### Timestamps (Nhãn Thời Gian)

Transcribe trả về timestamp (thời điểm bắt đầu/kết thúc) cho từng từ — cho phép tạo subtitle (phụ đề) .srt/.vtt, seek (tua) audio/video đến từ cụ thể, hoặc alignment với slide trình chiếu.

---

## 📊 Kiến Trúc Điển Hình

### Pipeline Tạo Phụ Đề Tự Động (Auto Subtitle)

```
Video upload lên S3
      │
      ▼ (S3 Event)
Lambda Function
      │
      ├──► Transcribe batch job (language_code="vi-VN")
      │         │ (webhook hoặc polling)
      │         ▼
      │    Transcript JSON (words + timestamps)
      │
      ├──► Convert sang .srt (SubRip Subtitle) / .vtt (WebVTT) format
      │
      ▼
MediaConvert (gắn subtitle vào video)
      │
      ▼
CloudFront (phân phối video + phụ đề)
```

### Pipeline Voice Assistant (Trợ Lý Giọng Nói)

```
Người dùng nói → Microphone
      │
      ▼
Transcribe Streaming (real-time STT)
      │
      ▼
Văn bản → Lambda / LLM (Bedrock) → Câu trả lời văn bản
      │
      ▼
Polly Neural TTS → Audio response
      │
      ▼
Loa phát cho người dùng nghe
```

### Pipeline Phân Tích Tổng Đài

```
Cuộc gọi tổng đài (2 kênh: agent + customer)
      │
      ▼
Transcribe Call Analytics
      ├──► Transcript đầy đủ với phân biệt agent/customer
      ├──► Sentiment theo từng đoạn (positive/negative/neutral/mixed)
      ├──► Issue Categories (loại vấn đề) — tự định nghĩa
      ├──► Action items (việc cần làm sau cuộc gọi)
      └──► PII Redaction (che thông tin nhạy cảm)
            │
            ▼
      Redshift / QuickSight (phân tích xu hướng)
      SNS Alert (nếu phát hiện sentiment âm hoặc compliance issue)
```

---

## 💰 Lưu Ý Chi Phí

| Dịch Vụ | Mô Hình Giá | Mẹo Tiết Kiệm |
|---|---|---|
| Transcribe | Per giây audio (~$0.024/phút) | Dùng batch thay streaming khi không cần real-time |
| Transcribe Medical | Per giây audio (~$0.075/phút) | Chỉ dùng khi audio thực sự là nội dung y tế |
| Transcribe Call Analytics | Per giây audio (~$0.03/phút) | Chỉ phân tích cuộc gọi có giá trị, bỏ qua cuộc gọi quá ngắn |
| Polly Standard | Per 1 triệu ký tự (~$4) | Cache audio đã tổng hợp tránh tổng hợp lại |
| Polly Neural | Per 1 triệu ký tự (~$16) | Dùng Standard cho nội dung ít quan trọng |

> **Free Tier:** Transcribe miễn phí 60 phút/tháng (12 tháng đầu); Polly miễn phí 5 triệu ký tự/tháng Standard + 1 triệu ký tự/tháng Neural (12 tháng đầu).

> **Mẹo quan trọng:** Caching (Lưu Đệm) — nếu cùng một đoạn văn bản được đọc nhiều lần (VD: thông báo chào mừng), lưu file audio đã tổng hợp lên S3 và phát lại, tránh gọi Polly mỗi lần.

---

## 🔗 Điều Hướng Module

| Chủ Đề | File |
|---|---|
| Transcribe fundamentals — Batch, Real-time, Custom Vocabulary | [1-transcribe-fundamentals.md](1-transcribe-fundamentals.md) |
| Transcribe Call Analytics — Phân tích tổng đài, sentiment, categories | [2-transcribe-call-analytics.md](2-transcribe-call-analytics.md) |
| Transcribe Medical — Phiên âm y tế, HIPAA, PHI | [3-transcribe-medical.md](3-transcribe-medical.md) |
| Amazon Polly — Neural TTS, SSML, Lexicons | [4-polly.md](4-polly.md) |

---

## 🎯 Câu Hỏi Phỏng Vấn Nhanh

1. **Transcribe vs Polly — khác nhau cơ bản thế nào?** → Transcribe là STT (âm thanh → văn bản); Polly là TTS (văn bản → âm thanh). Hai dịch vụ bổ sung nhau trong pipeline voice assistant.

2. **Khi nào dùng Transcribe Medical thay vì Transcribe thường?** → Khi audio là ghi chú lâm sàng, toa thuốc, báo cáo y tế — Transcribe Medical có model riêng được huấn luyện với thuật ngữ y tế, tên thuốc, quy trình phẫu thuật, cho độ chính xác cao hơn nhiều với nội dung y tế.

3. **Speaker Diarization là gì?** → Tính năng tách audio thành các đoạn theo từng người nói, gắn nhãn `spk_0`, `spk_1`... Dùng khi phiên âm cuộc họp, phỏng vấn hoặc podcast nhiều người tham gia.

4. **Neural TTS của Polly tốt hơn Standard TTS thế nào?** → Neural TTS (Tổng Hợp Giọng Nói Thần Kinh) dùng kiến trúc deep learning nên giọng đọc tự nhiên hơn, ít "robot" hơn — nhưng giá cao hơn ~4 lần so với Standard. Dùng Neural cho trải nghiệm người dùng quan trọng.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
