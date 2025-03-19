import pygame
import sys
import random
from pygame.locals import *
from PIL import Image
import io
import ctypes
from win32api import GetSystemMetrics
import win32gui
import win32con
from tkinter import simpledialog
import pygame_gui
import tkinter as tk

# 初始化pygame
pygame.init()

# 获取屏幕宽度和高度
screen_width = GetSystemMetrics(0)
screen_height = GetSystemMetrics(1)

# 创建一个全屏透明窗口
screen = pygame.display.set_mode((screen_width, screen_height), pygame.NOFRAME | pygame.SRCALPHA)

# 加载GIF图片
import os

# 获取当前脚本文件（Z.py）所在的目录
current_dir = os.path.dirname(os.path.abspath(__file__))

# 生成GIF文件的完整路径
gif_file = os.path.join(current_dir, 'ZZZ.gif')
img = Image.open(gif_file)

# 获取GIF的每一帧
frames = []
while True:
    try:
        # 将GIF帧转换为PIL格式
        frame = img.copy().convert("RGBA")
        
        # 转换为pygame.Surface对象
        frame_data = io.BytesIO()
        frame.save(frame_data, format='PNG')
        frame_data.seek(0)
        
        # 使用pygame加载并处理图像
        frame_surface = pygame.image.load(frame_data).convert_alpha()
        frames.append(frame_surface)

        img.seek(img.tell() + 1)
    except EOFError:
        break
# 调整GIF的初始大小
initial_width = 154  # 设置初始宽度
initial_height = 215  # 设置初始高度

# 重新调整所有帧的大小
frames_resized = []
for frame in frames:
    resized_frame = pygame.transform.smoothscale(frame, (initial_width, initial_height))
    frames_resized.append(resized_frame)
frames = frames_resized
# 设置动画播放的频率
frame_rate = 6  # 每秒显示10帧

# 设置桌宠初始位置
x = screen_width - initial_width  # 屏幕宽度 - 桌宠宽度
y = screen_height - initial_height  # 屏幕高度 - 桌宠高度

dragging = False  # 是否正在拖动

# 预设的十条语句
sentences = [
    "你好呀！",
    "今天真是个好日子！",
    "升天！",
    "有空一起喝茶吧！",
    "你今天看起来很精神！",
    "少喝水，多熬夜！",
    "Ciallo~！",
    "变态！",
    "Hentai！",
    "没错，宝宝我是黎曼猜想！"
]

# 获取桌宠非透明区域的点击检测
def is_click_on_pet(mouse_pos, frame, x, y):
    """检查鼠标点击是否在桌宠的非透明区域内"""
    px, py = mouse_pos
    frame_width, frame_height = frame.get_size()

    # 遍历桌宠当前帧的每一个像素，检查透明度
    for row in range(frame_height):
        for col in range(frame_width):
            r, g, b, a = frame.get_at((col, row))  # 获取像素的RGBA值
            if a > 0:  # 如果像素的alpha值大于0，说明它是非透明的
                # 计算该像素在屏幕上的位置
                screen_x = x + col
                screen_y = y + row
                # 检查鼠标位置是否在非透明像素内
                if screen_x == px and screen_y == py:
                    return True
    return False

# 设置窗口透明并穿透
hwnd = pygame.display.get_wm_info()["window"]
ctypes.windll.user32.SetWindowLongW(hwnd, win32con.GWL_EXSTYLE, win32con.WS_EX_LAYERED | win32con.WS_EX_TOOLWINDOW)
ctypes.windll.user32.SetLayeredWindowAttributes(hwnd, 0, 255, win32con.LWA_COLORKEY)

# 设置窗口为最上层
win32gui.SetWindowPos(hwnd, win32con.HWND_TOPMOST, 0, 0, 0, 0, win32con.SWP_NOMOVE | win32con.SWP_NOSIZE)

# 游戏界面管理
manager = pygame_gui.UIManager((screen_width, screen_height))

# 创建滑条组件用于调节GIF大小
slider = pygame_gui.elements.UIHorizontalSlider(relative_rect=pygame.Rect((screen_width // 2 - 150, screen_height // 2 - 25), (300, 50)),
                                                start_value=100,
                                                value_range=(50, 500),
                                                manager=manager)

# 创建按钮用于显示语句
sentence_display_surface = None
sentence_text = ''

# 显示语句的气泡
def show_sentence_bubble(sentence, x, y):
    """显示气泡窗口"""
    global sentence_display_surface, sentence_start_time
    # 使用字体支持中文（例如 SimHei 字体）
    font = pygame.font.Font("C:/Windows/Fonts/simhei.ttf", 10)  # 使用系统中自带的 SimHei 字体

    text_surface = font.render(sentence, True, (255, 255, 255))

    # 创建半透明的背景
    sentence_display_surface = pygame.Surface((text_surface.get_width() + 20, text_surface.get_height() + 20), pygame.SRCALPHA)
    pygame.draw.rect(sentence_display_surface, (255, 182, 193, 150), (0, 0, sentence_display_surface.get_width(), sentence_display_surface.get_height()))  # 半透明粉色
    sentence_display_surface.blit(text_surface, (10, 10))

    # 记录开始显示的时间
    sentence_start_time = pygame.time.get_ticks()

    return sentence_display_surface

# 变量来跟踪气泡窗口显示的时间
sentence_start_time = None
sentence_display_duration = 2000  # 气泡窗口显示的时间，单位毫秒，3秒

# 显示菜单
def show_menu(x, y):
    """在鼠标右键点击时显示菜单"""
    menu_font = pygame.font.Font("C:/Windows/Fonts/simhei.ttf", 16)  # 使用系统中自带的 SimHei 字体
    menu_bg_color = (200, 200, 200)
    menu_item_color = (0, 0, 0)
    menu_highlight_color = (100, 100, 100)
    
    # 菜单选项
    menu_items = ["调节GIF大小", "中断程序"]
    menu_height = len(menu_items) * 40

    # 绘制菜单背景
    menu_rect = pygame.Rect(x, y, 100, menu_height)
    pygame.draw.rect(screen, menu_bg_color, menu_rect)
    
    # 绘制菜单项
    for i, item in enumerate(menu_items):
        item_rect = pygame.Rect(x, y + i * 40, 100, 40)
        pygame.draw.rect(screen, menu_item_color, item_rect, 2)  # 绘制菜单项边框
        label = menu_font.render(item, True, menu_item_color)
        screen.blit(label, (x + 10, y + i * 40 + 10))
    
    pygame.display.update()
    
    return menu_rect, menu_items

# 处理菜单项的选择
def handle_menu_selection(menu_rect, menu_items, mouse_pos):
    """根据鼠标位置选择菜单项"""
    if menu_rect.collidepoint(mouse_pos):
        menu_index = (mouse_pos[1] - menu_rect.top) // 40
        if 0 <= menu_index < len(menu_items):
            if menu_items[menu_index] == "调节GIF大小":
                # 弹出窗口调节GIF大小
                root = tk.Tk()
                root.withdraw()  # 隐藏根窗口
                width = simpledialog.askinteger("GIF宽度", "输入新的GIF宽度：", minvalue=1, maxvalue=screen_width)
                height = simpledialog.askinteger("GIF高度", "输入新的GIF高度：", minvalue=1, maxvalue=screen_height)
                if width and height:
                    resize_gif(width, height)
            elif menu_items[menu_index] == "中断程序":
                return True
    return False

# 调整GIF大小
def resize_gif(width, height):
    """重新调整GIF的尺寸"""
    global frames
    resized_frames = []
    for frame in frames:
        resized_frame = pygame.transform.scale(frame, (width, height))
        resized_frames.append(resized_frame)
    frames = resized_frames

# 游戏循环，显示GIF动画
running = True
clock = pygame.time.Clock()
frame_index = 0
dragging = False
showing_menu = False
selected_item = None

while running:
    for event in pygame.event.get():
        if event.type == QUIT:
            running = False
        elif event.type == MOUSEBUTTONDOWN:
            mouse_pos = pygame.mouse.get_pos()

            if event.button == 3:  # 右键点击
                # 显示右键菜单
                menu_rect, menu_items = show_menu(mouse_pos[0], mouse_pos[1])
                showing_menu = True
            elif showing_menu:
                # 处理菜单项的选择
                if handle_menu_selection(menu_rect, menu_items, mouse_pos):
                    running = False  # 中断程序

            elif not showing_menu:
                # 检查是否点击桌宠
                if is_click_on_pet(mouse_pos, frames[frame_index], x, y):
                    dragging = True  # 开始拖动
                    mouse_x, mouse_y = mouse_pos
                    offset_x = x - mouse_x
                    offset_y = y - mouse_y
                    # 随机选取一句语句
                    sentence_text = random.choice(sentences)
                    sentence_display_surface = show_sentence_bubble(sentence_text, mouse_pos[0], mouse_pos[1])

        elif event.type == MOUSEBUTTONUP:
            dragging = False  # 结束拖动

        elif event.type == MOUSEMOTION:
            if dragging:
                mouse_x, mouse_y = pygame.mouse.get_pos()
                x = mouse_x + offset_x
                y = mouse_y + offset_y

                # 限制桌宠位置在屏幕范围内
                x = max(0, min(x, screen_width - frames[frame_index].get_width()))
                y = max(0, min(y, screen_height - frames[frame_index].get_height()))

        # 如果点击其他地方关闭菜单
        if showing_menu and event.type == MOUSEBUTTONDOWN:
            if not menu_rect.collidepoint(mouse_pos):
                showing_menu = False

        # 更新UI管理器
        manager.process_events(event)

    if not showing_menu:
        # 填充透明背景
        screen.fill((0, 0, 0, 0))  # 保持透明背景

        # 显示当前帧
        screen.blit(frames[frame_index], (x, y))  # 桌宠的显示位置随着(x, y)移动

        # 如果有语句，显示气泡
        if sentence_display_surface:
            # 获取当前时间
            current_time = pygame.time.get_ticks()

            # 如果气泡已经显示超过设定时间，隐藏它
            if current_time - sentence_start_time > sentence_display_duration:
                sentence_display_surface = None  # 删除气泡窗

            # 显示气泡
            if sentence_display_surface:  # 确保不会尝试绘制 None
                screen.blit(sentence_display_surface, (mouse_pos[0] + 20, mouse_pos[1] - 30))

        # 更新窗口
        pygame.display.update()

    # 控制帧率
    frame_index = (frame_index + 1) % len(frames)  # 动画帧循环
    clock.tick(frame_rate)
    
pygame.quit()
sys.exit()
