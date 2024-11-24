
# افزودن قابلیت تبدیل ویس به متن با استفاده از OpenAI Whisper

در این مستند دو روش برای اضافه کردن قابلیت پردازش ویس توضیح داده شده است: 
۱. **استفاده از فانکشن‌ها در Assistant Playground**  
۲. **بدون استفاده از فانکشن‌ها**  

---

## روش اول: استفاده از فانکشن‌ها در Assistant Playground

### مرحله ۱: تعریف فانکشن در Playground
۱. وارد محیط **Assistant Playground** شوید.  
۲. فانکشن زیر را برای پردازش فایل صوتی تعریف کنید:
```json
{
  "name": "process_audio",
  "description": "Convert an audio file to text and get an AI response",
  "parameters": {
    "type": "object",
    "properties": {
      "audio_url": {
        "type": "string",
        "description": "URL of the audio file to process"
      }
    },
    "required": ["audio_url"]
  }
}
```

### مرحله ۲: طراحی سرور برای پردازش ویس
یک سرور Flask طراحی کنید که فایل صوتی را دریافت کرده، متن آن را استخراج و پاسخ هوش مصنوعی را تولید کند:

```python
from flask import Flask, request, jsonify
import whisper
import openai
import os

app = Flask(__name__)
openai.api_key = "YOUR_OPENAI_API_KEY"
whisper_model = whisper.load_model("base")

@app.route('/process_audio', methods=['POST'])
def process_audio():
    data = request.json
    audio_url = data.get('audio_url')
    if not audio_url:
        return jsonify({"error": "No audio URL provided"}), 400

    temp_path = "temp_audio.wav"
    os.system(f"wget {audio_url} -O {temp_path}")

    try:
        transcription = whisper_model.transcribe(temp_path, language="fa")
        user_text = transcription['text']
    except Exception as e:
        os.remove(temp_path)
        return jsonify({"error": f"Whisper error: {e}"}), 500
    finally:
        os.remove(temp_path)

    try:
        response = openai.ChatCompletion.create(
            model="gpt-3.5-turbo",
            messages=[{"role": "user", "content": user_text}]
        )
        ai_text = response['choices'][0]['message']['content']
    except Exception as e:
        return jsonify({"error": f"OpenAI error: {e}"}), 500

    return jsonify({
        "user_text": user_text,
        "ai_response": ai_text
    })

if __name__ == '__main__':
    app.run(port=5000, debug=True)
```

### مرحله ۳: اتصال فانکشن به سرور
۱. فانکشن `process_audio` را با کلاینت خود (مثلاً وب یا اپلیکیشن موبایل) تست کنید.  
۲. فایل صوتی را به آدرس سرور ارسال کنید و پاسخ متنی را دریافت کنید.

---

## روش دوم: بدون استفاده از فانکشن‌ها

### مرحله ۱: طراحی سرور برای پردازش ویس
یک سرور Flask برای دریافت فایل صوتی، تبدیل آن به متن و تولید پاسخ طراحی کنید:

```python
from flask import Flask, request, jsonify
import whisper
import openai
import os

app = Flask(__name__)
openai.api_key = "YOUR_OPENAI_API_KEY"
whisper_model = whisper.load_model("base")

@app.route('/process_audio', methods=['POST'])
def process_audio():
    audio_file = request.files.get('audio')
    if not audio_file:
        return jsonify({"error": "No audio file provided."}), 400

    temp_path = "temp_audio.wav"
    audio_file.save(temp_path)

    try:
        transcription = whisper_model.transcribe(temp_path, language="fa")
        user_text = transcription['text']
    except Exception as e:
        os.remove(temp_path)
        return jsonify({"error": f"Whisper error: {e}"}), 500
    finally:
        os.remove(temp_path)

    try:
        response = openai.ChatCompletion.create(
            model="gpt-3.5-turbo",
            messages=[{"role": "user", "content": user_text}]
        )
        ai_text = response['choices'][0]['message']['content']
    except Exception as e:
        return jsonify({"error": f"OpenAI error: {e}"}), 500

    return jsonify({
        "user_text": user_text,
        "ai_response": ai_text
    })

if __name__ == '__main__':
    app.run(port=5000, debug=True)
```

### مرحله ۲: ارسال ویس به سرور
۱. از ابزارهای HTTP (مثل Postman) برای ارسال درخواست `POST` استفاده کنید.  
۲. فایل صوتی را با پارامتر `audio` ارسال کنید:
```bash
curl -X POST -F "audio=@your_audio_file.wav" http://127.0.0.1:5000/process_audio
```

### مرحله ۳: نمایش پاسخ
پاسخ دریافتی شامل متن تبدیل‌شده و پاسخ هوش مصنوعی است:
```json
{
  "user_text": "متن تبدیل‌شده از ویس",
  "ai_response": "پاسخ تولیدشده توسط هوش مصنوعی"
}
```

---

## مقایسه دو روش
- **استفاده از فانکشن‌ها**: مناسب برای ادغام با محیط‌های توسعه‌محور مثل Playground.  
- **بدون استفاده از فانکشن‌ها**: انعطاف‌پذیری بیشتر در سرورهای مستقل.  
