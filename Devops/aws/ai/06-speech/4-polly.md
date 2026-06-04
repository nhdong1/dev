# Amazon Polly — Tổng Hợp Giọng Nói (Text-to-Speech)

> Amazon Polly — Tổng Hợp Giọng Nói là dịch vụ TTS — Text-to-Speech — Chuyển Văn Bản Thành Giọng Nói được quản lý hoàn toàn của AWS. Polly chuyển đổi văn bản thành audio tự nhiên nghe như người thật, hỗ trợ **30+ ngôn ngữ** và **60+ giọng đọc**, bao gồm cả giọng **Neural** (Thần Kinh) chất lượng cực cao và **Standard** (Tiêu Chuẩn) chi phí thấp hơn.

---

## 🎯 Amazon Polly Làm Được Gì?

| Tính Năng | Mô Tả |
|---|---|
| **Standard TTS** (Tổng Hợp Giọng Nói Tiêu Chuẩn) | Giọng tổng hợp chất lượng tốt, chi phí thấp |
| **Neural TTS** (Tổng Hợp Giọng Nói Thần Kinh) | Giọng cực tự nhiên, gần như người thật — dùng deep learning |
| **Long-form TTS** (TTS Văn Bản Dài) | Tối ưu cho bài viết dài, sách nói, podcast |
| **SSML Support** (Hỗ Trợ Ngôn Ngữ Đánh Dấu Tổng Hợp Giọng Nói) | Kiểm soát phát âm, tốc độ, âm lượng, ngắt nghỉ |
| **Lexicons** (Từ Điển Phát Âm) | Định nghĩa cách phát âm cho từ/cụm từ đặc biệt |
| **Speech Marks** (Nhãn Giọng Nói) | Metadata về timing từng từ để đồng bộ animation/karaoke |
| **Brand Voice** (Giọng Thương Hiệu) | Tạo giọng nói tùy chỉnh cho thương hiệu (dịch vụ theo yêu cầu) |
| **Multiple Output Formats** (Nhiều Định Dạng Đầu Ra) | MP3, OGG Vorbis, PCM, JSON (speech marks) |

---

## 🧠 Standard vs Neural TTS

| Tiêu Chí | Standard TTS | Neural TTS |
|---|---|---|
| **Công nghệ** | Concatenative synthesis (Tổng Hợp Ghép Nối) | Deep neural network (Mạng Thần Kinh Sâu) |
| **Chất lượng giọng** | Tốt, đôi khi nghe "robot" | Rất tự nhiên, gần giọng người |
| **Biểu cảm** | Ít biểu cảm | Phong phú, lên xuống tự nhiên |
| **Giá** | ~$4/1 triệu ký tự | ~$16/1 triệu ký tự (~4x) |
| **Độ trễ** | Thấp | Cao hơn một chút |
| **Số giọng** | 60+ | Đang tăng dần |
| **SSML** | Hỗ trợ đầy đủ | Hỗ trợ (một số tag hạn chế) |
| **Phù hợp** | Thông báo hệ thống, IVR, content ít nhạy cảm | Voice assistant, sách nói, branding, UX quan trọng |

> **Quy tắc chọn:** Dùng Neural cho mọi nơi người dùng nghe thường xuyên (voice assistant, app). Dùng Standard cho hệ thống nội bộ, thông báo một lần, hoặc khi chi phí quan trọng hơn chất lượng.

---

## 🌍 Giọng Đọc Phổ Biến

| Ngôn Ngữ | Voice ID | Giới Tính | Loại |
|---|---|---|---|
| Anh (Mỹ) | `Joanna` | Nữ | Neural |
| Anh (Mỹ) | `Matthew` | Nam | Neural |
| Anh (Mỹ) | `Ivy` | Nữ (trẻ em) | Neural |
| Anh (Anh) | `Amy` | Nữ | Neural |
| Anh (Anh) | `Brian` | Nam | Neural |
| Pháp | `Léa` | Nữ | Neural |
| Đức | `Vicki` | Nữ | Neural |
| Nhật | `Takumi` | Nam | Neural |
| Hàn | `Seoyeon` | Nữ | Neural |
| Trung (giản thể) | `Zhiyu` | Nữ | Neural |
| Ả Rập | `Zeina` | Nữ | Standard |
| Tây Ban Nha (Mỹ) | `Lupe` | Nữ | Neural |
| Bồ Đào Nha (Brazil) | `Camila` | Nữ | Neural |

> **Tiếng Việt:** Polly **chưa hỗ trợ tiếng Việt**. Cần tự xây dựng TTS với SageMaker + custom model (VITS, FastSpeech2) nếu cần giọng tiếng Việt chất lượng cao.

---

## 📞 1. Synthesize Speech (Tổng Hợp Giọng Nói Cơ Bản)

```python
import boto3

polly = boto3.client("polly", region_name="us-east-1")

def synthesize_speech(
    text: str,
    voice_id: str = "Joanna",
    engine: str = "neural",           # "neural" hoặc "standard"
    output_format: str = "mp3",       # "mp3", "ogg_vorbis", "pcm"
    language_code: str = "en-US"
) -> bytes:
    """Chuyển văn bản thành audio, trả về bytes."""
    response = polly.synthesize_speech(
        Text=text,
        VoiceId=voice_id,
        Engine=engine,
        OutputFormat=output_format,
        LanguageCode=language_code
    )
    return response["AudioStream"].read()


def save_speech_to_file(text: str, output_path: str, voice_id: str = "Joanna") -> None:
    """Tổng hợp giọng nói và lưu vào file."""
    audio_bytes = synthesize_speech(text, voice_id=voice_id)
    with open(output_path, "wb") as f:
        f.write(audio_bytes)
    print(f"Đã lưu audio: {output_path}")


# Ví dụ sử dụng
save_speech_to_file(
    "Welcome to AWS Polly. This is a neural voice demonstration.",
    "welcome.mp3",
    voice_id="Joanna"
)
```

### Output Format (Định Dạng Đầu Ra)

| Format | Extension | Sample Rate | Phù Hợp |
|---|---|---|---|
| `mp3` | .mp3 | 8000/16000/22050/24000 Hz | Web, app — phổ biến nhất |
| `ogg_vorbis` | .ogg | 8000/16000/22050 Hz | WebRTC, browser streaming |
| `pcm` | .pcm | 8000/16000 Hz | Telephony, real-time processing |
| `json` | .json | N/A | Speech marks — timing metadata |

---

## 📝 2. SSML — Speech Synthesis Markup Language (Ngôn Ngữ Đánh Dấu Tổng Hợp Giọng Nói)

SSML là ngôn ngữ XML dùng để **kiểm soát chi tiết** cách Polly phát âm — ngắt nghỉ, nhấn mạnh, tốc độ, âm lượng, số điện thoại, viết tắt...

### Cú Pháp SSML Cơ Bản

```python
def synthesize_ssml(ssml_text: str, voice_id: str = "Joanna") -> bytes:
    """Tổng hợp giọng nói từ SSML markup."""
    response = polly.synthesize_speech(
        Text=ssml_text,
        TextType="ssml",            # Quan trọng: phải chỉ định TextType="ssml"
        VoiceId=voice_id,
        Engine="neural",
        OutputFormat="mp3"
    )
    return response["AudioStream"].read()


ssml_example = """
<speak>
    Xin chào! Đây là thông báo từ <emphasis level="strong">Amazon Polly</emphasis>.
    
    <break time="1s"/>
    
    Số điện thoại hỗ trợ: <say-as interpret-as="telephone">+1-800-555-0123</say-as>.
    
    <break time="500ms"/>
    
    Phiên bản hiện tại: <say-as interpret-as="characters">AWS</say-as> Polly v2.
    
    <prosody rate="slow" pitch="+5%">
        Cảm ơn bạn đã sử dụng dịch vụ của chúng tôi.
    </prosody>
</speak>
"""

audio = synthesize_ssml(ssml_example)
```

### Các Tag SSML Quan Trọng

#### `<break>` — Ngắt Nghỉ

```xml
<break time="500ms"/>        <!-- Dừng 500 milliseconds -->
<break time="2s"/>           <!-- Dừng 2 giây -->
<break strength="medium"/>   <!-- Ngắt theo cường độ: none/x-weak/weak/medium/strong/x-strong -->
```

#### `<emphasis>` — Nhấn Mạnh

```xml
<emphasis level="strong">Quan trọng!</emphasis>    <!-- strong/moderate/reduced -->
<emphasis level="moderate">Lưu ý</emphasis>
```

#### `<prosody>` — Điều Chỉnh Phát Âm

```xml
<prosody rate="slow">Nói chậm rãi.</prosody>       <!-- x-slow/slow/medium/fast/x-fast hoặc % -->
<prosody rate="150%">Nói nhanh 150%.</prosody>
<prosody pitch="+10%">Giọng cao hơn 10%.</prosody>
<prosody pitch="-5%">Giọng thấp hơn 5%.</prosody>
<prosody volume="loud">Nói to hơn.</prosody>        <!-- silent/x-soft/soft/medium/loud/x-loud hoặc dB -->
```

#### `<say-as>` — Đọc Theo Dạng Cụ Thể

```xml
<say-as interpret-as="characters">AWS</say-as>         <!-- Đọc từng chữ: "A W S" -->
<say-as interpret-as="spell-out">SSML</say-as>         <!-- Đánh vần: "S S M L" -->
<say-as interpret-as="telephone">555-0123</say-as>     <!-- Số điện thoại -->
<say-as interpret-as="date" format="mdy">5/15/2026</say-as>  <!-- Ngày tháng -->
<say-as interpret-as="time">14:30</say-as>             <!-- Thời gian -->
<say-as interpret-as="cardinal">123</say-as>           <!-- Số đếm: "một trăm hai mươi ba" -->
<say-as interpret-as="ordinal">1st</say-as>            <!-- Số thứ tự: "first" -->
<say-as interpret-as="fraction">3/4</say-as>           <!-- Phân số: "three quarters" -->
<say-as interpret-as="unit">5kg</say-as>               <!-- Đơn vị đo lường -->
```

#### `<sub>` — Thay Thế Từ Đọc

```xml
<sub alias="Amazon Web Services">AWS</sub>     <!-- Đọc "Amazon Web Services" khi gặp "AWS" -->
<sub alias="mét vuông">m²</sub>               <!-- Đọc "mét vuông" khi gặp "m²" -->
```

#### `<lang>` — Chuyển Đổi Ngôn Ngữ

```xml
<speak>
    This is English text.
    <lang xml:lang="fr-FR">Bonjour, comment allez-vous?</lang>
    Back to English now.
</speak>
```

#### `<phoneme>` — Phát Âm Theo IPA

```xml
<phoneme alphabet="ipa" ph="pɪˈkɑːn">pecan</phoneme>   <!-- Định nghĩa phát âm chính xác -->
<phoneme alphabet="x-sampa" ph="p\ik">pecan</phoneme>   <!-- Dùng x-sampa thay IPA -->
```

#### `<audio>` — Chèn File Âm Thanh

```xml
<speak>
    Vui lòng nghe thông báo sau:
    <audio src="s3://my-bucket/notification.mp3"/>
    Cảm ơn bạn đã lắng nghe.
</speak>
```

### SSML Thực Tế — Thông Báo IVR (Interactive Voice Response — Hệ Thống Trả Lời Tự Động)

```python
def create_ivr_greeting(customer_name: str, account_balance: float) -> str:
    """Tạo SSML cho lời chào IVR cá nhân hóa."""
    return f"""
<speak>
    <prosody rate="medium" volume="medium">
        Xin chào <emphasis level="moderate">{customer_name}</emphasis>.
        
        <break time="300ms"/>
        
        Số dư tài khoản của bạn là 
        <say-as interpret-as="cardinal">{int(account_balance)}</say-as> 
        đô la Mỹ.
        
        <break time="500ms"/>
        
        Để kiểm tra giao dịch gần đây, nhấn <say-as interpret-as="cardinal">1</say-as>.
        
        <break time="200ms"/>
        
        Để chuyển tiền, nhấn <say-as interpret-as="cardinal">2</say-as>.
        
        <break time="200ms"/>
        
        Để gặp nhân viên hỗ trợ, nhấn <say-as interpret-as="cardinal">0</say-as>.
    </prosody>
</speak>
"""

audio = synthesize_ssml(
    create_ivr_greeting("Nguyễn Văn A", 2500.50),
    voice_id="Joanna"
)
```

---

## 📖 3. Lexicons (Từ Điển Phát Âm Tùy Chỉnh)

Lexicon — Từ Điển Phát Âm cho phép định nghĩa cách phát âm cho từ/cụm từ đặc biệt mà không cần dùng SSML `<phoneme>` tag mỗi lần.

### Định Dạng Lexicon (PLS — Pronunciation Lexicon Specification)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<lexicon version="1.0"
      xmlns="http://www.w3.org/2005/01/pronunciation-lexicon"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:schemaLocation="http://www.w3.org/2005/01/pronunciation-lexicon
        http://www.w3.org/TR/2007/CR-pronunciation-lexicon-20071212/pls.xsd"
      alphabet="ipa"
      xml:lang="en-US">

  <!-- Tên thương hiệu -->
  <lexeme>
    <grapheme>AWS</grapheme>
    <alias>Amazon Web Services</alias>
  </lexeme>

  <!-- Từ kỹ thuật -->
  <lexeme>
    <grapheme>Kubernetes</grapheme>
    <phoneme>ˌkuːbərˈneɪtɪs</phoneme>
  </lexeme>

  <!-- Tên riêng -->
  <lexeme>
    <grapheme>Nguyen</grapheme>
    <alias>noo-yen</alias>
  </lexeme>

  <!-- Từ viết tắt đặc biệt -->
  <lexeme>
    <grapheme>SageMaker</grapheme>
    <alias>Sage Maker</alias>
  </lexeme>

</lexicon>
```

### Upload và Sử Dụng Lexicon

```python
def upload_lexicon(lexicon_name: str, lexicon_content: str) -> None:
    """Upload Pronunciation Lexicon lên Polly."""
    polly.put_lexicon(
        Name=lexicon_name,
        Content=lexicon_content
    )
    print(f"Lexicon '{lexicon_name}' đã được upload.")


def synthesize_with_lexicons(
    text: str,
    lexicon_names: list[str],
    voice_id: str = "Joanna"
) -> bytes:
    """Tổng hợp giọng nói với custom lexicons."""
    response = polly.synthesize_speech(
        Text=text,
        VoiceId=voice_id,
        Engine="neural",
        OutputFormat="mp3",
        LexiconNames=lexicon_names    # Áp dụng tối đa 5 lexicons
    )
    return response["AudioStream"].read()


# Ví dụ
tech_lexicon = """<?xml version="1.0" encoding="UTF-8"?>
<lexicon version="1.0" 
    xmlns="http://www.w3.org/2005/01/pronunciation-lexicon"
    alphabet="ipa" xml:lang="en-US">
  <lexeme><grapheme>AWS</grapheme><alias>Amazon Web Services</alias></lexeme>
  <lexeme><grapheme>ML</grapheme><alias>Machine Learning</alias></lexeme>
</lexicon>"""

upload_lexicon("tech-terms", tech_lexicon)

audio = synthesize_with_lexicons(
    "AWS ML services include SageMaker and Bedrock.",
    ["tech-terms"]
)
```

---

## ⏱️ 4. Speech Marks (Nhãn Đồng Bộ Giọng Nói)

Speech Marks — Nhãn Giọng Nói trả về **metadata JSON** chứa thông tin timing (thời điểm) của từng từ, câu, viseme (hình dạng miệng). Dùng để đồng bộ animation, highlight text (karaoke), điều khiển avatar.

### Các Loại Speech Mark

| Loại | Mô Tả | Ứng Dụng |
|---|---|---|
| `word` | Thời điểm bắt đầu từng từ (ms) | Karaoke highlight, subtitle sync |
| `sentence` | Thời điểm bắt đầu từng câu (ms) | Phân đoạn nội dung |
| `viseme` | Hình dạng miệng — 22 viseme khác nhau | Lip-sync cho avatar ảo |
| `ssml` | Sự kiện SSML (ví dụ: `<mark>` tag) | Trigger action tại điểm cụ thể |

```python
import json

def get_speech_marks(
    text: str,
    voice_id: str = "Joanna",
    mark_types: list[str] = ["word", "sentence", "viseme"]
) -> list[dict]:
    """Lấy speech marks để đồng bộ animation/highlight."""
    response = polly.synthesize_speech(
        Text=text,
        VoiceId=voice_id,
        Engine="neural",
        OutputFormat="json",        # Phải là "json" cho speech marks
        SpeechMarkTypes=mark_types
    )

    marks = []
    for line in response["AudioStream"].read().decode("utf-8").strip().split("\n"):
        marks.append(json.loads(line))
    return marks


marks = get_speech_marks("Hello, welcome to Amazon Polly!")
for mark in marks:
    print(mark)
# {"time":0,"type":"sentence","start":0,"end":31,"value":"Hello, welcome to Amazon Polly!"}
# {"time":6,"type":"word","start":0,"end":5,"value":"Hello"}
# {"time":6,"type":"viseme","value":"e"}
# {"time":246,"type":"word","start":7,"end":14,"value":"welcome"}
# ...
```

### Ứng Dụng Speech Marks — Karaoke Style Highlight

```python
def create_karaoke_sync(text: str) -> tuple[bytes, list[dict]]:
    """Tạo audio + timing data cho karaoke highlight."""
    # Lấy audio song song với speech marks
    audio_future = synthesize_speech(text, output_format="mp3")

    marks = get_speech_marks(text, mark_types=["word"])

    word_timings = [
        {
            "word": mark["value"],
            "start_ms": mark["time"],
            "end_ms": marks[i + 1]["time"] if i + 1 < len(marks) else None
        }
        for i, mark in enumerate(marks)
        if mark["type"] == "word"
    ]

    return audio_future, word_timings
```

---

## 📦 5. Long-form Synthesis và Async (Tổng Hợp Văn Bản Dài)

Transcribe `synthesize_speech` có giới hạn **3.000 ký tự** (Standard) hoặc **3.000 ký tự** (Neural). Với văn bản dài hơn, dùng **Start Speech Synthesis Task** (Tác Vụ Tổng Hợp Bất Đồng Bộ):

```python
def start_long_synthesis_task(
    text: str,
    output_bucket: str,
    output_key: str,
    voice_id: str = "Joanna",
    engine: str = "neural",
    output_format: str = "mp3"
) -> str:
    """Tổng hợp văn bản dài bất đồng bộ — lưu kết quả vào S3."""
    response = polly.start_speech_synthesis_task(
        Text=text,
        VoiceId=voice_id,
        Engine=engine,
        OutputFormat=output_format,
        OutputS3BucketName=output_bucket,
        OutputS3KeyPrefix=output_key      # Prefix cho file kết quả trên S3
    )
    return response["SynthesisTask"]["TaskId"]


def get_synthesis_task_status(task_id: str) -> dict:
    """Kiểm tra trạng thái task tổng hợp."""
    response = polly.get_speech_synthesis_task(TaskId=task_id)
    task = response["SynthesisTask"]
    return {
        "status": task["TaskStatus"],                        # scheduled/inProgress/completed/failed
        "output_uri": task.get("OutputUri"),                 # URL S3 khi completed
        "characters_synthesized": task.get("RequestCharacters")
    }
```

### Giới Hạn Kích Thước Input

| Chế Độ | Giới Hạn |
|---|---|
| `synthesize_speech` (sync) | 3.000 ký tự (bao gồm SSML tags) |
| `start_speech_synthesis_task` (async) | 100.000 ký tự |

---

## 🔄 6. Danh Sách Giọng Và Kiểm Tra Hỗ Trợ

```python
def list_available_voices(language_code: str = None, engine_type: str = None) -> list[dict]:
    """Liệt kê các giọng đọc có sẵn với bộ lọc tùy chọn."""
    params = {}
    if language_code:
        params["LanguageCode"] = language_code
    if engine_type:
        params["Engine"] = engine_type         # "standard", "neural", "long-form"

    response = polly.describe_voices(**params)
    voices = []
    for voice in response["Voices"]:
        voices.append({
            "id": voice["Id"],
            "name": voice["Name"],
            "gender": voice["Gender"],
            "language": voice["LanguageName"],
            "language_code": voice["LanguageCode"],
            "engines": voice.get("SupportedEngines", [])
        })
    return voices


# Tìm tất cả giọng Neural tiếng Anh (Mỹ)
en_neural_voices = list_available_voices(language_code="en-US", engine_type="neural")
for v in en_neural_voices:
    print(f"{v['id']} — {v['name']} ({v['gender']})")
# Joanna — Joanna (Female)
# Matthew — Matthew (Male)
# Ivy — Ivy (Female)
# ...
```

---

## 🏗️ Kiến Trúc Ứng Dụng Thực Tế

### Voice Assistant (Trợ Lý Giọng Nói) Với Polly + Bedrock

```
Người dùng nói
      │
      ▼
Transcribe Streaming (STT)
      │
      ▼
Văn bản → Lambda
      │
      ├──► Amazon Lex (phân tích intent nếu cần)
      │
      └──► Bedrock (Claude) → Câu trả lời
               │
               ▼
           Polly Neural TTS (synthesize_speech)
               │
               ▼
           Audio stream → Loa thiết bị
```

### Hệ Thống Sách Nói Tự Động (Audiobook)

```
File văn bản (PDF/DOCX) → S3
      │
      ▼
Textract (nếu cần OCR — trích xuất từ PDF)
      │
      ▼
Tiền xử lý: chia chương, thêm SSML markup (ngắt nghỉ giữa câu)
      │
      ▼
Polly start_speech_synthesis_task (async — tổng hợp từng chương)
      │
      ▼
Audio MP3 files → S3
      │
      ▼
CloudFront phân phối + Player web/mobile
```

### Hệ Thống Thông Báo Đa Ngôn Ngữ

```python
ANNOUNCEMENT_VOICES = {
    "en": ("Joanna", "neural"),
    "fr": ("Léa", "neural"),
    "de": ("Vicki", "neural"),
    "ja": ("Takumi", "neural"),
    "zh": ("Zhiyu", "neural"),
    "ko": ("Seoyeon", "neural"),
}

def announce_in_all_languages(message_template: dict[str, str]) -> dict[str, bytes]:
    """Tổng hợp thông báo bằng nhiều ngôn ngữ song song."""
    results = {}
    for lang_code, text in message_template.items():
        if lang_code in ANNOUNCEMENT_VOICES:
            voice_id, engine = ANNOUNCEMENT_VOICES[lang_code]
            results[lang_code] = synthesize_speech(text, voice_id=voice_id, engine=engine)
    return results
```

---

## 💰 Chi Phí Và Tối Ưu

| Engine | Giá | Ghi Chú |
|---|---|---|
| Standard | ~$4 / 1 triệu ký tự | Đầu tiên 5 triệu ký tự/tháng miễn phí (12 tháng) |
| Neural | ~$16 / 1 triệu ký tự | Đầu tiên 1 triệu ký tự/tháng miễn phí (12 tháng) |
| Long-form | ~$100 / 1 triệu ký tự | Chất lượng cao nhất cho nội dung dài |

### Chiến Lược Tiết Kiệm Chi Phí

```python
import hashlib
import boto3

s3 = boto3.client("s3")

def cached_synthesize(
    text: str,
    voice_id: str,
    cache_bucket: str
) -> bytes:
    """Tổng hợp giọng nói với caching — tránh tổng hợp lại cùng nội dung."""
    # Tạo cache key từ hash của text + voice
    cache_key = hashlib.md5(f"{text}:{voice_id}".encode()).hexdigest()
    s3_key = f"polly-cache/{cache_key}.mp3"

    try:
        # Kiểm tra cache
        obj = s3.get_object(Bucket=cache_bucket, Key=s3_key)
        return obj["Body"].read()
    except s3.exceptions.NoSuchKey:
        # Cache miss — tổng hợp mới
        audio = synthesize_speech(text, voice_id=voice_id)
        s3.put_object(Bucket=cache_bucket, Key=s3_key, Body=audio)
        return audio
```

> **Mẹo:** Caching là kỹ thuật quan trọng nhất để tiết kiệm chi phí Polly. Với các câu thông báo lặp lại (lời chào, menu IVR), cache một lần và dùng nhiều lần.

---

## ❓ Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Standard TTS vs Neural TTS — khi nào chọn cái nào?**

> Neural khi chất lượng âm thanh quan trọng với người dùng cuối: voice assistant, sách nói, ứng dụng B2C. Standard khi chi phí là ưu tiên và người dùng nghe ít: hệ thống nội bộ, thông báo một lần, IVR cũ. Neural đắt hơn ~4 lần nhưng cho trải nghiệm tự nhiên hơn nhiều.

**Q: SSML là gì và khi nào cần dùng?**

> SSML (Speech Synthesis Markup Language) là XML dùng để điều khiển chi tiết cách Polly phát âm. Cần dùng khi: muốn ngắt nghỉ đúng chỗ, đọc số điện thoại/ngày tháng/chữ viết tắt đúng cách, nhấn mạnh từ quan trọng, điều chỉnh tốc độ/âm lượng, hoặc chèn file audio. Không cần SSML cho văn bản tự nhiên đơn giản.

**Q: Lexicon khác SSML `<phoneme>` thế nào?**

> `<phoneme>` áp dụng cho một lần trong đoạn text cụ thể — phải thêm vào mỗi lần văn bản xuất hiện từ đó. Lexicon là từ điển toàn cục — định nghĩa một lần, áp dụng tự động cho mọi lần từ đó xuất hiện trong text. Lexicon phù hợp cho từ đặc thù của domain (tên thương hiệu, thuật ngữ kỹ thuật) xuất hiện thường xuyên.

**Q: Speech Marks dùng để làm gì?**

> Speech Marks trả về timing metadata — thời điểm bắt đầu từng từ, câu, hoặc viseme (hình dạng miệng). Ứng dụng: karaoke (highlight từng từ theo nhịp), lip-sync cho avatar ảo, subtitle tự động đồng bộ, trigger animation tại điểm cụ thể trong audio.

**Q: Polly có hỗ trợ tiếng Việt không?**

> Chưa — Polly hiện không hỗ trợ tiếng Việt. Với ứng dụng TTS tiếng Việt, cần tự xây dựng với SageMaker và các mô hình open-source như VITS, FastSpeech2, hoặc dùng dịch vụ TTS của bên thứ ba.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
