# prem-ai-assistant
#Python-based AI voice assistant with animation, camera vision, and offline LLM support.
#🚀 Built My Own AI Assistant from Scratch – “PREM AI” 🤖

#I developed a desktop-based AI assistant using Python that can:
#🎤 Listen to voice commands
#🗣️ Speak back naturally
#👀 See through the camera and describe the environment
#🎭 Show real-time animated facial expressions
#🧠 Run fully offline using local AI models (Ollama)

#🛠 Tech Stack:
# Python
# Tkinter (GUI)
# SpeechRecognition
# Pyttsx3 (Text-to-Speech)
# OpenCV (Vision)
# Ollama (LLaMA + Moondream)
import tkinter as tk
from tkinter import Canvas
from PIL import Image, ImageTk
import threading
import random
import pyttsx3
import speech_recognition as sr
import ollama
import time
import math
import cv2
import os
import webbrowser

# --- GLOBAL VARIABLES ---
is_speaking = False
camera_active = False
window = None
canvas = None
cap = None

# Animation States
anim_frame = 0
blink_timer = 0
mouth_open = 0
arm_gesture = 0

# 1. SETUP MOUTH (ROBUST ENGINE - NO CRASHES)
def speak_thread_func(text):
    global is_speaking
    is_speaking = True
    print(f"PREM: {text}")
    
    try:
        # Re-initialize engine every time to avoid "run loop" errors
        engine = pyttsx3.init()
        voices = engine.getProperty('voices')
        
        # --- STRICT MALE VOICE SELECTION ---
        target_voice = None
        
        # Priority 1: Indian Male (Ravi)
        for voice in voices:
            if "Ravi" in voice.name:
                target_voice = voice.id
                break
                
        # Priority 2: US Male (David) - If Ravi is missing
        if target_voice is None:
            for voice in voices:
                if "David" in voice.name:
                    target_voice = voice.id
                    break
        
        # Fallback
        if target_voice is None:
            target_voice = voices[0].id

        engine.setProperty('voice', target_voice)
        engine.setProperty('rate', 155) 
        
        # --- SAFE SPEAKING BLOCK ---
        try:
            engine.say(text)
            engine.runAndWait()
        except RuntimeError:
            # If engine is stuck, force stop and retry
            try:
                engine.stop()
                engine.say(text)
                engine.runAndWait()
            except:
                pass 
            
    except Exception as e:
        print(f"Voice Critical Error: {e}")
        
    time.sleep(0.2)
    is_speaking = False

def speak(text):
    if not is_speaking:
        t = threading.Thread(target=speak_thread_func, args=(text,))
        t.start()

# 2. SETUP EARS
def listen():
    if is_speaking: return ""
    
    r = sr.Recognizer()
    with sr.Microphone() as source:
        print("Listening...")
        r.adjust_for_ambient_noise(source, duration=0.5)
        try:
            audio = r.listen(source, timeout=5, phrase_time_limit=8)
            text = r.recognize_google(audio)
            print(f"You said: {text}")
            return text.lower()
        except:
            return ""

# 3. PC CONTROL BRAIN
def open_system_app(command):
    speak("On it.")
    cmd = command.replace("open", "").strip()
    
    # NOTE: If apps don't open, replace these with full paths
    # e.g. os.startfile("C:\\Program Files\\Google\\Chrome\\Application\\chrome.exe")
    
    if "notepad" in cmd:
        os.system("start notepad")
    elif "calculator" in cmd:
        os.system("start calc")
    elif "paint" in cmd:
        os.system("start mspaint")
    elif "chrome" in cmd:
        os.system("start chrome")
    elif "youtube" in cmd:
        webbrowser.open("https://www.youtube.com")
    elif "google" in cmd:
        webbrowser.open("https://www.google.com")
    else:
        try:
            os.system(f"start {cmd}")
        except:
            speak(f"I cannot open {cmd}.")

# 4. VISION BRAIN (MOONDREAM - LOW RAM)
def see_and_describe():
    global cap
    if not camera_active or cap is None:
        speak("My camera is off.")
        return

    speak("Analyzing...")
    time.sleep(1.0)
    
    ret, frame = cap.read()
    if ret:
        cv2.imwrite("vision_temp.jpg", frame)
        try:
            # USING MOONDREAM (Small & Fast)
            response = ollama.chat(model='moondream', messages=[{
                'role': 'user',
                'content': 'Describe this image clearly in one sentence.',
                'images': ['vision_temp.jpg']
            }])
            speak(response['message']['content'])
        except Exception as e:
            print(f"--- VISION ERROR: {e} ---")
            speak("I failed to process the image. Did you run 'ollama pull moondream'?")

# 5. CAMERA CONTROL
def toggle_camera(state):
    global camera_active, cap
    if state == "open":
        if not camera_active:
            cap = cv2.VideoCapture(0)
            camera_active = True
            speak("Camera ON.")
    else:
        if camera_active:
            camera_active = False
            if cap: cap.release()
            canvas.delete("cam_feed")
            speak("Camera OFF.")

# 6. ANIMATION (SPONGEBOB)
def draw_prem():
    global anim_frame, blink_timer, mouth_open, arm_gesture
    canvas.delete("drawing")
    cx, cy = 300, 300 
    
    anim_frame += 1
    bob_y = math.sin(anim_frame / 15.0) * 4
    blink_timer += 1
    is_blinking = False
    if blink_timer > random.randint(100, 300):
        is_blinking = True
        if blink_timer > 310: blink_timer = 0
            
    if is_speaking:
        mouth_open = random.randint(5, 25)
        arm_gesture = math.sin(anim_frame / 5.0) * 15
    else:
        mouth_open = 2
        arm_gesture = 0

    base_y = cy + bob_y - 50

    # Draw Legs
    canvas.create_rectangle(cx-40, base_y+130, cx-30, base_y+170, fill="#FFFF00", outline="black", width=2, tags="drawing")
    canvas.create_rectangle(cx-40, base_y+130, cx-30, base_y+150, fill="white", outline="black", width=2, tags="drawing")
    canvas.create_oval(cx-50, base_y+170, cx-20, base_y+190, fill="black", tags="drawing")
    canvas.create_rectangle(cx+30, base_y+130, cx+40, base_y+170, fill="#FFFF00", outline="black", width=2, tags="drawing")
    canvas.create_rectangle(cx+30, base_y+130, cx+40, base_y+150, fill="white", outline="black", width=2, tags="drawing")
    canvas.create_oval(cx+20, base_y+170, cx+50, base_y+190, fill="black", tags="drawing")

    # Draw Body
    canvas.create_rectangle(cx-90, base_y-100, cx+90, base_y+80, fill="#FFFF00", outline="black", width=3, tags="drawing")
    canvas.create_oval(cx-70, base_y-80, cx-50, base_y-60, fill="#CCCC00", outline="#CCCC00", tags="drawing")
    canvas.create_oval(cx+60, base_y-70, cx+80, base_y-50, fill="#CCCC00", outline="#CCCC00", tags="drawing")

    # Draw Clothes
    canvas.create_rectangle(cx-90, base_y+80, cx+90, base_y+105, fill="white", outline="black", width=3, tags="drawing")
    canvas.create_rectangle(cx-90, base_y+105, cx+90, base_y+130, fill="#8B4513", outline="black", width=3, tags="drawing")
    canvas.create_line(cx-80, base_y+115, cx+80, base_y+115, fill="black", dash=(4,4), width=2, tags="drawing")
    canvas.create_polygon(cx, base_y+80, cx-10, base_y+100, cx, base_y+115, cx+10, base_y+100, fill="red", outline="black", tags="drawing")

    # Draw Face
    eye_y = base_y - 20
    canvas.create_oval(cx-65, eye_y-35, cx-5, eye_y+35, fill="white", outline="black", width=2, tags="drawing")
    canvas.create_oval(cx-50, eye_y-15, cx-20, eye_y+15, fill="#00BFFF", outline="black", tags="drawing")
    canvas.create_oval(cx-42, eye_y-7, cx-28, eye_y+7, fill="black", tags="drawing")
    canvas.create_oval(cx+5, eye_y-35, cx+65, eye_y+35, fill="white", outline="black", width=2, tags="drawing")
    canvas.create_oval(cx+20, eye_y-15, cx+50, eye_y+15, fill="#00BFFF", outline="black", tags="drawing")
    canvas.create_oval(cx+28, eye_y-7, cx+42, eye_y+7, fill="black", tags="drawing")
    
    if is_blinking:
        canvas.create_arc(cx-65, eye_y-35, cx-5, eye_y+35, start=0, extent=180, fill="#FFFF00", outline="black", tags="drawing")
        canvas.create_arc(cx+5, eye_y-35, cx+65, eye_y+35, start=0, extent=180, fill="#FFFF00", outline="black", tags="drawing")
    
    canvas.create_line(cx-35, eye_y-35, cx-35, eye_y-45, width=2, fill="black", tags="drawing")
    canvas.create_line(cx+35, eye_y-35, cx+35, eye_y-45, width=2, fill="black", tags="drawing")
    canvas.create_arc(cx-10, eye_y, cx+15, eye_y+25, start=0, extent=180, style=tk.ARC, outline="black", width=2, tags="drawing")
    canvas.create_line(cx-10, eye_y+12, cx+15, eye_y+12, fill="#FFFF00", width=5, tags="drawing")

    # Mouth
    mouth_y = base_y + 30
    if is_speaking:
        canvas.create_arc(cx-50, mouth_y, cx+50, mouth_y+mouth_open+10, start=180, extent=180, fill="#660000", outline="black", width=2, tags="drawing")
        canvas.create_oval(cx-15, mouth_y+mouth_open, cx+15, mouth_y+mouth_open+10, fill="pink", tags="drawing")
    else:
        canvas.create_arc(cx-50, mouth_y, cx+50, mouth_y+40, start=180, extent=180, style=tk.ARC, outline="black", width=2, tags="drawing")
        canvas.create_rectangle(cx-15, mouth_y+20, cx-5, mouth_y+35, fill="white", outline="black", tags="drawing")
        canvas.create_rectangle(cx+5, mouth_y+20, cx+15, mouth_y+35, fill="white", outline="black", tags="drawing")

    # Draw Arms
    hand_y = base_y+80 - arm_gesture
    hand_x = cx+120 + (arm_gesture/2)
    canvas.create_line(cx-90, base_y+40, cx-120, base_y+80, fill="#FFFF00", width=10, capstyle=tk.ROUND, tags="drawing")
    canvas.create_oval(cx-130, base_y+70, cx-110, base_y+90, fill="#FFFF00", outline="black", tags="drawing")
    canvas.create_line(cx+90, base_y+40, cx+120, hand_y, fill="#FFFF00", width=10, capstyle=tk.ROUND, tags="drawing")
    canvas.create_oval(hand_x-10, hand_y-10, hand_x+10, hand_y+10, fill="#FFFF00", outline="black", tags="drawing")

    # Camera Feed
    if camera_active and cap:
        ret, frame = cap.read()
        if ret:
            frame = cv2.resize(frame, (320, 240))
            frame = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
            img = ImageTk.PhotoImage(image=Image.fromarray(frame))
            canvas.image_ref = img 
            canvas.create_image(700, 300, image=img, tags="cam_feed")
            canvas.create_rectangle(540, 180, 860, 420, outline="#FFFF00", width=4, tags="cam_feed_border")
            canvas.create_text(700, 165, text="PREM VISION", fill="white", font=("Arial", 12, "bold"), tags="cam_feed_border")

    window.after(40, draw_prem)

# 7. MAIN BRAIN
def brain_main():
    time.sleep(1)
    speak("Prem Systems Online.")
    
    persona = "You are Prem, a smart Indian AI assistant. Answer briefly."
    history = [{'role': 'system', 'content': persona}]

    while True:
        while is_speaking: time.sleep(0.1)
        user_voice = listen()
        if user_voice:
            # 1. COMMAND: OPEN APP
            if "open" in user_voice:
                if "cam" in user_voice or "camera" in user_voice:
                    toggle_camera("open")
                else:
                    open_system_app(user_voice)
                continue
                
            # 2. COMMAND: CLOSE APP
            elif "close" in user_voice and "cam" in user_voice:
                toggle_camera("close")
                continue
                
            # 3. COMMAND: VISION
            elif "what" in user_voice and ("see" in user_voice or "looking" in user_voice):
                see_and_describe()
                continue
                
            # 4. COMMAND: EXIT
            elif "quit" in user_voice or "exit" in user_voice:
                speak("Shutting down.")
                if cap: cap.release()
                window.quit()
                break
            
            # 5. GENERAL CHAT
            history.append({'role': 'user', 'content': user_voice})
            try:
                # Standard chat uses llama3.2 (fast)
                response = ollama.chat(model='llama3.2', messages=history)
                reply = response['message']['content']
                speak(reply)
                history.append({'role': 'assistant', 'content': reply})
            except Exception as e:
                print(e)

if __name__ == "__main__":
    window = tk.Tk()
    window.title("Prem - AI Assistant")
    window.configure(bg="#4da6ff") 
    window.geometry("900x600")
    
    canvas = tk.Canvas(window, width=900, height=600, bg="#4da6ff", highlightthickness=0)
    canvas.pack()

    t = threading.Thread(target=brain_main)
    t.daemon = True
    t.start()

    draw_prem()
    window.mainloop()
