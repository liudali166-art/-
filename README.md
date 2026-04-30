# -
改卷
import tkinter as tk
from PIL import Image, ImageGrab
import requests
import base64
import io
import json

# ================= 配置区域 =================
# 请在此处填写您的小米MIMO API配置
API_KEY = "YOUR_MIMO_API_KEY"
API_URL = "https://api.mimo.xiaomi.com/v1/chat/completions" # 示例URL，请以文档为准
MODEL_NAME = "mimo-v1" # 示例模型名称
# ============================================

class StarReadApp:
    def __init__(self):
        self.root = tk.Tk()
        self.root.title("星阅 - AI实时改卷助手")
        self.root.geometry("300x150")
        
        self.label = tk.Label(self.root, text="点击下方按钮开始选区批改", pady=20)
        self.label.pack()
        
        self.btn = tk.Button(self.root, text="截图并批改", command=self.start_capture, bg="#107C10", fg="white")
        self.btn.pack()

    def start_capture(self):
        """进入截图模式"""
        self.root.withdraw()  # 隐藏主窗口
        self.capture_win = tk.Toplevel(self.root)
        self.capture_win.attributes('-alpha', 0.3)  # 设置透明度
        self.capture_win.attributes('-fullscreen', True)
        self.capture_win.config(cursor="cross")
        
        self.canvas = tk.Canvas(self.capture_win, cursor="cross", bg="grey")
        self.canvas.pack(fill="both", expand=True)
        
        self.canvas.bind("<ButtonPress-1>", self.on_button_press)
        self.canvas.bind("<B1-Motion>", self.on_move_press)
        self.canvas.bind("<ButtonRelease-1>", self.on_button_release)
        
        self.rect = None
        self.start_x = None
        self.start_y = None

    def on_button_press(self, event):
        self.start_x = event.x
        self.start_y = event.y
        self.rect = self.canvas.create_rectangle(self.start_x, self.start_y, 1, 1, outline='red', width=2)

    def on_move_press(self, event):
        cur_x, cur_y = (event.x, event.y)
        self.canvas.coords(self.rect, self.start_x, self.start_y, cur_x, cur_y)

    def on_button_release(self, event):
        end_x, end_y = (event.x, event.y)
        self.capture_win.destroy()
        self.root.deiconify()
        
        # 截取选定区域
        bbox = (min(self.start_x, end_x), min(self.start_y, end_y), max(self.start_x, end_x), max(self.start_y, end_y))
        screenshot = ImageGrab.grab(bbox)
        self.analyze_paper(screenshot)

    def encode_image(self, image):
        """将图片转换为Base64"""
        buffer = io.BytesIO()
        image.save(buffer, format="JPEG")
        return base64.b64encode(buffer.getvalue()).decode('utf-8')

    def analyze_paper(self, image):
        """调用AI API进行分析"""
        print("正在上传图片并分析中...")
        base64_image = self.encode_image(image)
        
        headers = {
            "Content-Type": "application/json",
            "Authorization": f"Bearer {API_KEY}"
        }
        
        payload = {
            "model": MODEL_NAME,
            "messages": [
                {
                    "role": "system",
                    "content": "你是一位专业的特级教师。请识别图片中的题目和学生的答案，根据标准逻辑进行批改。给出得分、错误原因以及改进建议。请使用Markdown格式输出。"
                },
                {
                    "role": "user",
                    "content": [
                        {"type": "text", "text": "请批改这张试卷选区内容："},
                        {"type": "image_url", "image_url": {"url": f"data:image/jpeg;base64,{base64_image}"}}
                    ]
                }
            ],
            "max_tokens": 1000
        }

        try:
            response = requests.post(API_URL, headers=headers, json=payload)
            result = response.json()
            content = result['choices'][0]['message']['content']
            self.show_result(content)
        except Exception as e:
            print(f"请求失败: {e}")
            self.show_result(f"分析失败，请检查网络或API配置。\n错误信息: {str(e)}")

    def show_result(self, text):
        """展示批改结果"""
        res_win = tk.Toplevel(self.root)
        res_win.title("星阅 - 批改结果")
        res_win.geometry("500x600")
        
        text_area = tk.Text(res_win, wrap="word", padx=10, pady=10)
        text_area.insert("1.0", text)
        text_area.config(state="disabled")
        text_area.pack(fill="both", expand=True)

if __name__ == "__main__":
    app = StarReadApp()
    app.root.mainloop()
