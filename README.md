import os
import threading
from flask import Flask, request, jsonify
from gtts import gTTS
from moviepy.editor import *
from PIL import Image, ImageDraw, ImageFont

app = Flask(__name__)

def generate_voiceover(text, output_path):
    tts = gTTS(text=text, lang="ar")
    tts.save(output_path)

def create_text_image(text, image_path, output_path):
    img = Image.open(image_path).convert("RGBA")
    draw = ImageDraw.Draw(img)
    font = ImageFont.truetype("arial.ttf", 60)
    text_position = (50, img.height // 2)
    draw.text(text_position, text, font=font, fill=(255, 255, 255))
    img.save(output_path)

def create_video(text, bg_video, music, output_video):
    voice_path = f"static/{output_video}_voice.mp3"
    text_image_path = f"static/{output_video}_text.png"
    output_video_path = f"static/{output_video}.mp4"
    
    generate_voiceover(text, voice_path)
    create_text_image(text, "background.jpg", text_image_path)
    
    video_clip = VideoFileClip(bg_video).subclip(0, 10)
    audio_clip = AudioFileClip(voice_path)
    music_clip = AudioFileClip(music).set_duration(video_clip.duration)
    final_audio = CompositeAudioClip([audio_clip, music_clip.volumex(0.3)])
    
    text_img_clip = ImageClip(text_image_path).set_duration(video_clip.duration).set_position("center")
    final_video = CompositeVideoClip([video_clip, text_img_clip]).set_audio(final_audio)
    
    final_video.write_videofile(output_video_path, fps=24, codec="libx264", threads=4)
    return output_video_path

def generate_video_thread(text, video_id):
    create_video(text, "background.mp4", "background.mp3", video_id)

@app.route("/generate_video", methods=["POST"])
def generate_video():
    data = request.json
    text = data.get("text")
    video_id = f"video_{len(os.listdir('static'))}"
    
    thread = threading.Thread(target=generate_video_thread, args=(text, video_id))
    thread.start()
    
    return jsonify({"message": "جاري إنشاء الفيديو...", "video_id": video_id})

@app.route("/get_video/<video_id>", methods=["GET"])
def get_video(video_id):
    video_path = f"static/{video_id}.mp4"
    if os.path.exists(video_path):
        return jsonify({"status": "تم الإنشاء", "video_url": video_path})
    return jsonify({"status": "قيد المعالجة"})

if __name__ == "__main__":
    if not os.path.exists("static"):
        os.makedirs("static")
    app.run(host="0.0.0.0", port=5000)
