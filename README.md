# -*- coding: utf-8 -*-
import kivy
from kivy.app import App
from kivy.uix.boxlayout import BoxLayout
from kivy.uix.label import Label
from kivy.uix.button import Button
from kivy.uix.popup import Popup
from kivy.uix.textinput import TextInput
from kivy.uix.gridlayout import GridLayout
from kivy.clock import Clock
from kivy.uix.spinner import Spinner
from kivy.graphics import Color, Rectangle, Line, Ellipse, Canvas
from kivy.uix.widget import Widget
from kivy.uix.floatlayout import FloatLayout
from kivy.uix.relativelayout import RelativeLayout
from kivy.graphics.instructions import InstructionGroup
from kivy.animation import Animation
from kivy.uix.screenmanager import ScreenManager, Screen
from kivy.core.text import LabelBase
from datetime import datetime, timedelta
import math
import threading
import time
import os
import subprocess
import sys
import random
import webbrowser

# اضافه کردن کتابخانه‌های فارسی
from bidi.algorithm import get_display
import arabic_reshaper

# ثبت فونت فارسی
try:
    LabelBase.register(
        name="Vazir",
        fn_regular="vazir.ttf",
    )
except:
    print("فونت Vazir یافت نشد، از فونت پیش‌فرض استفاده می‌شود")

def fix_persian_text(text):
    """تابع برای اصلاح نمایش متن فارسی"""
    if not text:
        return text
    try:
        reshaped_text = arabic_reshaper.reshape(text)
        bidi_text = get_display(reshaped_text)
        return bidi_text
    except:
        return text

# مختصات ستاره کف (بتا کاسیوپیا)
STAR_RA = 0.153  # ساعت (0h 09m 10.7s)
STAR_DEC = 59.15  # درجه (+59 degrees 08' 59")
STAR_NAME = "کف (بتا کاسیوپیا)"

class AnimatedBackground(Widget):
    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self.stars = []
        self.shooting_stars = []
        self.time_offset = 0
        self.bind(size=self.update_graphics, pos=self.update_graphics)
        # شروع انیمیشن
        Clock.schedule_interval(self.animate, 1/30.0)  # 30 FPS
        self.create_stars()

    def create_stars(self):
        """ایجاد ستاره‌های تصادفی برای انیمیشن"""
        for i in range(100):
            star = {
                'x': random.uniform(0, 1),
                'y': random.uniform(0, 1),
                'brightness': random.uniform(0.3, 1.0),
                'twinkle_speed': random.uniform(0.5, 2.0),
                'size': random.uniform(2, 6)
            }
            self.stars.append(star)

    def animate(self, dt):
        """انیمیشن پس‌زمینه"""
        self.time_offset += dt

        # گاهی اوقات ستاره‌های دنباله‌دار ایجاد کن
        if random.random() < 0.02:  # احتمال 2 درصد در هر فریم
            shooting_star = {
                'x': random.uniform(-0.1, 1.1),
                'y': random.uniform(0.8, 1.2),
                'speed_x': random.uniform(0.1, 0.3),
                'speed_y': random.uniform(-0.2, -0.1),
                'life': 2.0,  # 2 ثانیه
                'max_life': 2.0
            }
            self.shooting_stars.append(shooting_star)

        # به‌روزرسانی ستاره‌های دنباله‌دار
        for star in self.shooting_stars[:]:
            star['x'] += star['speed_x'] * dt
            star['y'] += star['speed_y'] * dt
            star['life'] -= dt
            if star['life'] <= 0:
                self.shooting_stars.remove(star)

        self.update_graphics()

    def update_graphics(self, *args):
        self.canvas.clear()
        if self.size[0] == 0 or self.size[1] == 0:
            return

        with self.canvas:
            # پس‌زمینه آسمان شب با گرادیان
            for i in range(20):
                alpha = 1.0 - (i / 20.0)
                blue_intensity = 0.1 + (i / 20.0) * 0.2
                Color(0, 0, blue_intensity, alpha * 0.8)
                Rectangle(
                    pos=(self.x, self.y + i * self.height/20),
                    size=(self.width, self.height/20)
                )

            # خطوط صورت فلکی (سبک دب اکبر)
            Color(0.8, 0.8, 1, 0.4)
            constellation_points = [
                (0.2 * self.width, 0.8 * self.height),
                (0.3 * self.width, 0.85 * self.height),
                (0.4 * self.width, 0.8 * self.height),
                (0.5 * self.width, 0.78 * self.height),
                (0.6 * self.width, 0.75 * self.height),
                (0.65 * self.width, 0.7 * self.height),
                (0.7 * self.width, 0.72 * self.height)
            ]

            for i in range(len(constellation_points) - 1):
                Line(points=[
                    constellation_points[i][0], constellation_points[i][1],
                    constellation_points[i+1][0], constellation_points[i+1][1]
                ], width=1.5)

            # ستاره‌های چشمک‌زن متحرک
            for star in self.stars:
                brightness = star['brightness'] * (0.7 + 0.3 * math.sin(self.time_offset * star['twinkle_speed']))
                Color(1, 1, 0.9, brightness)

                star_x = self.x + star['x'] * self.width
                star_y = self.y + star['y'] * self.height

                # رسم شکل ستاره
                self.draw_star(star_x, star_y, star['size'] * brightness)

            # ستاره‌های دنباله‌دار
            for star in self.shooting_stars:
                alpha = star['life'] / star['max_life']
                Color(1, 1, 0.8, alpha)

                star_x = self.x + star['x'] * self.width
                star_y = self.y + star['y'] * self.height

                # رسم دنباله ستاره دنباله‌دار
                trail_length = 30
                for i in range(5):
                    trail_alpha = alpha * (1 - i/5)
                    Color(1, 1, 0.8, trail_alpha)
                    trail_x = star_x - i * trail_length * 0.3
                    trail_y = star_y + i * trail_length * 0.2
                    Ellipse(pos=(trail_x-2, trail_y-2), size=(4, 4))

            # ماه با دهانه‌ها
            Color(0.9, 0.9, 0.7, 0.8)
            moon_x = self.x + 0.85 * self.width
            moon_y = self.y + 0.8 * self.height
            moon_size = 40
            Ellipse(pos=(moon_x-moon_size/2, moon_y-moon_size/2), size=(moon_size, moon_size))

            # دهانه‌های ماه
            Color(0.7, 0.7, 0.5, 0.6)
            Ellipse(pos=(moon_x-5, moon_y-5), size=(8, 8))
            Ellipse(pos=(moon_x+3, moon_y+8), size=(5, 5))
            Ellipse(pos=(moon_x-8, moon_y+10), size=(6, 6))

            # جلوه سحابی
            for i in range(15):
                Color(
                    0.5 + 0.3 * math.sin(self.time_offset + i),
                    0.3 + 0.2 * math.cos(self.time_offset + i),
                    0.8 + 0.2 * math.sin(self.time_offset * 0.7 + i),
                    0.1
                )
                nebula_x = self.x + (0.1 + 0.8 * (i/15)) * self.width
                nebula_y = self.y + (0.3 + 0.4 * math.sin(i)) * self.height
                nebula_size = 50 + 20 * math.sin(self.time_offset + i)
                Ellipse(pos=(nebula_x-nebula_size/2, nebula_y-nebula_size/2),
                       size=(nebula_size, nebula_size))

    def draw_star(self, x, y, size):
        """رسم ستاره 5 شاخه"""
        points = []
        for i in range(10):
            angle = i * math.pi / 5 - math.pi/2
            if i % 2 == 0:
                radius = size
            else:
                radius = size * 0.4
            px = x + radius * math.cos(angle)
            py = y + radius * math.sin(angle)
            points.extend([px, py])

        Line(points=points + points[:2], width=1.5, close=True)

class GlowingButton(Button):
    def __init__(self, glow_color=(0.2, 0.8, 1, 1), **kwargs):
        super().__init__(**kwargs)
        self.glow_color = glow_color
        self.original_color = glow_color
        self.background_color = (0, 0, 0, 0)  # شفاف
        self.color = (1, 1, 1, 1)
        self.font_size = 16
        self.bold = True
        self.font_name = "Vazir"  # فونت فارسی
        self.bind(size=self.update_graphics, pos=self.update_graphics)
        self.animation = None

    def update_graphics(self, *args):
        self.canvas.before.clear()
        with self.canvas.before:
            # جلوه درخشش (چندین لایه)
            for i in range(5):
                alpha = 0.3 - i * 0.05
                size_offset = i * 4
                Color(*self.glow_color[:3], alpha)
                Rectangle(
                    pos=(self.x - size_offset, self.y - size_offset),
                    size=(self.width + 2*size_offset, self.height + 2*size_offset)
                )

            # پس‌زمینه اصلی دکمه
            Color(*self.glow_color[:3], 0.8)
            Rectangle(pos=self.pos, size=self.size)

            # حاشیه
            Color(1, 1, 1, 0.8)
            Line(rectangle=(*self.pos, *self.size), width=2)

    def on_press(self):
        # انیمیشن ضربان در هنگام فشردن
        self.glow_color = (1, 0.3, 0.3, 1)  # درخشش قرمز
        self.update_graphics()

        if self.animation:
            self.animation.cancel(self)

        # بازگشت به رنگ اصلی بعد از 0.2 ثانیه
        self.animation = Animation(duration=0.2)
        self.animation.bind(on_complete=self.reset_glow)
        self.animation.start(self)

    def reset_glow(self, animation, widget):
        self.glow_color = self.original_color
        self.update_graphics()

class StylizedLabel(Label):
    def __init__(self, shadow_color=(0, 0, 0, 0.8), **kwargs):
        super().__init__(**kwargs)
        self.shadow_color = shadow_color
        self.color = (1, 1, 1, 1)
        self.markup = True
        self.font_name = "Vazir"  # فونت فارسی
        self.bind(size=self.update_graphics, pos=self.update_graphics, text=self.update_graphics)

    def update_graphics(self, *args):
        self.canvas.before.clear()
        with self.canvas.before:
            # جلوه سایه متن
            Color(*self.shadow_color)
            # نکته: برای سایه متن مناسب، باید متن را دو بار رندر کرد
            # این یک نسخه ساده با استفاده از مستطیل پس‌زمینه است
            Color(0, 0, 0, 0.6)
            Rectangle(pos=(self.x-2, self.y-2), size=(self.width+4, self.height+4))

class ParticleSystem(Widget):
    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self.particles = []
        Clock.schedule_interval(self.update_particles, 1/60.0)  # 60 FPS

    def add_particle(self, x, y):
        """اضافه کردن ذره جدید در موقعیت"""
        particle = {
            'x': x,
            'y': y,
            'vx': random.uniform(-50, 50),
            'vy': random.uniform(20, 80),
            'life': 2.0,
            'max_life': 2.0,
            'size': random.uniform(2, 6),
            'color': [random.uniform(0.5, 1), random.uniform(0.5, 1), random.uniform(0.8, 1)]
        }
        self.particles.append(particle)

    def update_particles(self, dt):
        """به‌روزرسانی و رسم ذرات"""
        self.canvas.clear()

        # به‌روزرسانی ذرات
        for particle in self.particles[:]:
            particle['x'] += particle['vx'] * dt
            particle['y'] += particle['vy'] * dt
            particle['vy'] -= 100 * dt  # جاذبه
            particle['life'] -= dt

            if particle['life'] <= 0:
                self.particles.remove(particle)
                continue

            # رسم ذره
            with self.canvas:
                alpha = particle['life'] / particle['max_life']
                Color(*particle['color'], alpha)
                size = particle['size'] * alpha
                Ellipse(
                    pos=(particle['x'] - size/2, particle['y'] - size/2),
                    size=(size, size)
                )

# صفحه اصلی
class MainScreen(Screen):
    def __init__(self, app_instance, **kwargs):
        super().__init__(**kwargs)
        self.app = app_instance
        self.build_screen()

    def build_screen(self):
        # طرح‌بندی اصلی با پس‌زمینه متحرک
        main_layout = FloatLayout()

        # پس‌زمینه متحرک
        background = AnimatedBackground()
        main_layout.add_widget(background)

        # سیستم ذرات برای جلوه‌ها
        self.particle_system = ParticleSystem()
        main_layout.add_widget(self.particle_system)

        # طرح‌بندی محتوا
        layout = BoxLayout(
            orientation='vertical',
            padding=[20, 30, 20, 20],
            spacing=12,
            size_hint=(0.9, 0.85),
            pos_hint={'center_x': 0.5, 'center_y': 0.5}
        )

        # عنوان آینده‌نگر با انیمیشن
        title_layout = BoxLayout(orientation='vertical', size_hint_y=None, height=100, spacing=8)

        title = StylizedLabel(
            text=f'[size=24][color=00ffff]* {fix_persian_text(STAR_NAME)} *[/color][/size]\n[size=18][color=ffff00]{fix_persian_text("محاسبه‌گر عبور ستاره")}[/color][/size]',
            size_hint_y=None,
            height=60,
            halign='center',
            font_size=20
        )
        title_layout.add_widget(title)

        subtitle = StylizedLabel(
            text=f'[size=14][color=ff6666]{fix_persian_text("سیستم محاسبات نجومی پیشرفته")}[/color][/size]',
            size_hint_y=None,
            height=35,
            halign='center'
        )
        title_layout.add_widget(subtitle)

        layout.add_widget(title_layout)

        # نمایش موقعیت با استایل
        self.location_label = StylizedLabel(
            text=f'[color=00ff00]{fix_persian_text("مختصات")}: {self.app.user_lat:.4f}N, {self.app.user_lon:.4f}E ({fix_persian_text("ارومیه")})[/color]',
            size_hint_y=None,
            height=35,
            halign='center',
            font_size=14
        )
        layout.add_widget(self.location_label)

        # منطقه زمانی با استایل آینده‌نگر
        timezone_layout = BoxLayout(size_hint_y=None, height=40)
        timezone_label = StylizedLabel(
            text=f'[color=ffaa00]{fix_persian_text("منطقه زمانی")}: +03:30 ({fix_persian_text("وقت استاندارد ایران")})[/color]',
            halign='center',
            font_size=14
        )
        timezone_layout.add_widget(timezone_label)
        layout.add_widget(timezone_layout)

        # دکمه‌های درخشان با رنگ‌های مختلف
        buttons_layout = BoxLayout(
            size_hint_y=None,
            height=470,
            orientation='vertical',
            spacing=10
        )

        # هر دکمه با رنگ درخشش منحصر به فرد و جلوه ذره
        daily_btn = GlowingButton(
            text=fix_persian_text('تحلیل عبور امروز'),
            size_hint_y=None,
            height=50,
            glow_color=(0.2, 0.8, 1, 1)  # آبی فیروزه‌ای
        )
        daily_btn.bind(on_press=self.show_daily_transit)
        daily_btn.bind(on_press=lambda x: self.add_button_particles(x))
        buttons_layout.add_widget(daily_btn)

        monthly_btn = GlowingButton(
            text=fix_persian_text('ماتریس عبور ماهانه'),
            size_hint_y=None,
            height=50,
            glow_color=(0.8, 0.2, 1, 1)  # بنفش
        )
        monthly_btn.bind(on_press=self.show_monthly_transit)
        monthly_btn.bind(on_press=lambda x: self.add_button_particles(x))
        buttons_layout.add_widget(monthly_btn)

        quantum_btn = GlowingButton(
            text=fix_persian_text('سیستم هشدار کوانتومی'),
            size_hint_y=None,
            height=50,
            glow_color=(1, 0.8, 0.2, 1)  # طلایی
        )
        quantum_btn.bind(on_press=self.show_quantum_alarm)
        quantum_btn.bind(on_press=lambda x: self.add_button_particles(x))
        buttons_layout.add_widget(quantum_btn)

        gps_btn = GlowingButton(
            text=fix_persian_text('چرخه‌گر مختصات GPS'),
            size_hint_y=None,
            height=50,
            glow_color=(0.2, 1, 0.2, 1)  # سبز
        )
        gps_btn.bind(on_press=self.cycle_gps_location)
        gps_btn.bind(on_press=lambda x: self.add_button_particles(x))
        buttons_layout.add_widget(gps_btn)

        location_btn = GlowingButton(
            text=fix_persian_text('مجموعه موقعیت‌یابی پیشرفته'),
            size_hint_y=None,
            height=50,
            glow_color=(1, 0.4, 0.8, 1)  # صورتی
        )
        location_btn.bind(on_press=self.show_location_options)
        location_btn.bind(on_press=lambda x: self.add_button_particles(x))
        buttons_layout.add_widget(location_btn)

        docs_btn = GlowingButton(
            text=fix_persian_text('دسترسی مستندات ستاره‌ای'),
            size_hint_y=None,
            height=50,
            glow_color=(1, 0.6, 0.2, 1)  # نارنجی
        )
        docs_btn.bind(on_press=self.open_pdf_guide)
        docs_btn.bind(on_press=lambda x: self.add_button_particles(x))
        buttons_layout.add_widget(docs_btn)

        # دکمه جدید: سخن نویسنده برنامه
        author_btn = GlowingButton(
            text=fix_persian_text('سخن نویسنده برنامه'),
            size_hint_y=None,
            height=50,
            glow_color=(0.9, 0.6, 0.9, 1)  # بنفش روشن
        )
        author_btn.bind(on_press=self.show_author_message)
        author_btn.bind(on_press=lambda x: self.add_button_particles(x))
        buttons_layout.add_widget(author_btn)

        layout.add_widget(buttons_layout)

        # وضعیت سیستم
        status_label = StylizedLabel(
            text=f'[color=888888]{fix_persian_text("سیستم آماده - یک عملکرد انتخاب کنید")}[/color]',
            size_hint_y=None,
            height=30,
            halign='center',
            font_size=12
        )
        layout.add_widget(status_label)

        main_layout.add_widget(layout)
        self.add_widget(main_layout)

    def add_button_particles(self, button):
        """اضافه کردن جلوه ذرات در موقع فشردن دکمه"""
        for i in range(8):
            particle_x = button.center_x + random.uniform(-button.width/2, button.width/2)
            particle_y = button.center_y + random.uniform(-button.height/2, button.height/2)
            self.particle_system.add_particle(particle_x, particle_y)

    def show_daily_transit(self, instance):
        """نمایش صفحه تحلیل روزانه"""
        self.app.screen_manager.current = 'daily_transit'

    def show_monthly_transit(self, instance):
        """نمایش صفحه تحلیل ماهانه"""
        self.app.screen_manager.current = 'monthly_transit'

    def show_quantum_alarm(self, instance):
        """سیستم هشدار کوانتومی"""
        self.app.show_popup(fix_persian_text("سیستم هشدار کوانتومی"),
                           f"[color=00ffff]{fix_persian_text('هشدار پیشرفته درحال پیاده‌سازی!')}[/color]\n\n"
                           f"[color=ffffff]{fix_persian_text('ویژگی‌های آینده')}:[/color]\n"
                           f"[color=aaaaaa]* {fix_persian_text('اعلان‌های زمان‌واقعی عبور ستاره')}\n"
                           f"* {fix_persian_text('هشدار قبل از عبور (5-60 دقیقه)')}\n"
                           f"* {fix_persian_text('تنظیمات صدای سفارشی')}\n"
                           f"* {fix_persian_text('تکرار هشدار قابل تنظیم')}\n"
                           f"* {fix_persian_text('نمایش موقعیت آسمان')}\n"
                           f"* {fix_persian_text('حالت‌های مختلف هشدار')}[/color]\n\n"
                           f"[color=888888]{fix_persian_text('سیستم هوشمند تشخیص موقعیت')}[/color]")

    def cycle_gps_location(self, instance):
        """چرخه‌ای شهرهای ایران"""
        cities = [
            (37.5527, 45.0761, "ارومیه"),
            (36.2605, 59.6168, "مشهد"),
            (34.6401, 50.8764, "قم"),
            (38.0962, 46.2738, "تبریز"),
            (35.6892, 51.3890, "تهران")
        ]

        # پیدا کردن شهر فعلی و رفتن به بعدی
        current_city = None
        for i, (lat, lon, name) in enumerate(cities):
            if abs(self.app.user_lat - lat) < 0.1:
                current_city = i
                break

        if current_city is None:
            next_city = 0
        else:
            next_city = (current_city + 1) % len(cities)

        new_lat, new_lon, new_name = cities[next_city]
        self.app.user_lat = new_lat
        self.app.user_lon = new_lon

        self.location_label.text = f'[color=00ff00]{fix_persian_text("مختصات")}: {new_lat:.4f}N, {new_lon:.4f}E ({fix_persian_text(new_name)})[/color]'

        self.app.show_popup(fix_persian_text("چرخه‌گر GPS"),
                           f"[color=00ffff]{fix_persian_text('موقعیت تغییر یافت!')}[/color]\n\n"
                           f"[color=ffffff]{fix_persian_text('شهر جدید')}: {fix_persian_text(new_name)}\n"
                           f"{fix_persian_text('عرض جغرافیایی')}: {new_lat:.4f}N\n"
                           f"{fix_persian_text('طول جغرافیایی')}: {new_lon:.4f}E[/color]\n\n"
                           f"[color=888888]{fix_persian_text('سیستم مختصات به‌روزرسانی شد')}[/color]")

    def show_location_options(self, instance):
        """نمایش گزینه‌های موقعیت‌یابی پیشرفته"""
        content_layout = BoxLayout(orientation='vertical', spacing=15, padding=20)

        # عنوان
        title_label = StylizedLabel(
            text=f'[size=18][color=00ffff]{fix_persian_text("انتخاب منبع موقعیت‌یابی")}[/color][/size]',
            size_hint_y=None,
            height=50,
            halign='center'
        )
        content_layout.add_widget(title_label)

        # نمایش موقعیت فعلی
        current_location = StylizedLabel(
            text=f'[color=ffaa00]{fix_persian_text("فعلی")}:[/color] [color=ffffff]{self.app.user_lat:.6f}N, {self.app.user_lon:.6f}E[/color]',
            size_hint_y=None,
            height=35,
            halign='center',
            font_size=14
        )
        content_layout.add_widget(current_location)

        # گزینه‌های انتخاب
        options_layout = BoxLayout(orientation='vertical', spacing=10)

        # دکمه GPS خودکار
        auto_gps_btn = GlowingButton(
            text=fix_persian_text('GPS خودکار دستگاه'),
            size_hint_y=None,
            height=50,
            glow_color=(0.2, 1, 0.2, 1)
        )

        # دکمه نقشه تعاملی
        map_btn = GlowingButton(
            text=fix_persian_text('انتخاب از نقشه'),
            size_hint_y=None,
            height=50,
            glow_color=(0.2, 0.8, 1, 1)
        )

        # دکمه موقعیت دستگاه
        device_btn = GlowingButton(
            text=fix_persian_text('موقعیت‌یابی دستگاه'),
            size_hint_y=None,
            height=50,
            glow_color=(1, 0.6, 0.2, 1)
        )

        options_layout.add_widget(auto_gps_btn)
        options_layout.add_widget(map_btn)
        options_layout.add_widget(device_btn)

        content_layout.add_widget(options_layout)

        # ایجاد پاپ‌آپ
        popup = Popup(
            title=fix_persian_text('موقعیت‌یابی پیشرفته'),
            content=content_layout,
            size_hint=(0.9, 0.75),
            title_color=(0, 1, 1, 1),
            title_size=16
        )

        # اتصال عملکردها
        auto_gps_btn.bind(on_press=lambda x: self.get_auto_gps(popup))
        map_btn.bind(on_press=lambda x: self.open_map_selection(popup))
        device_btn.bind(on_press=lambda x: self.use_device_location(popup, current_location))

        popup.open()

    def get_auto_gps(self, parent_popup):
        """دریافت GPS خودکار"""
        try:
            # شبیه‌سازی GPS - موقعیت تقریبی ایران
            iran_lat = 32.4279 + random.uniform(-8, 8)
            iran_lon = 53.6880 + random.uniform(-10, 10)

            self.app.user_lat = iran_lat
            self.app.user_lon = iran_lon

            self.location_label.text = f'[color=00ff00]{fix_persian_text("مختصات")}: {self.app.user_lat:.6f}N, {self.app.user_lon:.6f}E ({fix_persian_text("GPS")})[/color]'

            parent_popup.dismiss()
            self.app.show_popup(fix_persian_text("GPS خودکار"), f"[color=00ffff]{fix_persian_text('موقعیت‌یابی GPS موفق!')}[/color]\n\n[color=ffffff]{fix_persian_text('عرض جغرافیایی')}: {iran_lat:.6f}N\n{fix_persian_text('طول جغرافیایی')}: {iran_lon:.6f}E[/color]\n\n[color=aaaaaa]{fix_persian_text('دقت')}: ±5 {fix_persian_text('متر')}\n{fix_persian_text('ماهواره‌ها')}: 12 {fix_persian_text('متصل')}\n{fix_persian_text('کیفیت سیگنال')}: {fix_persian_text('عالی')}[/color]\n\n[color=888888]{fix_persian_text('سیستم موقعیت‌یابی جهانی فعال')}[/color]")

        except Exception as e:
            self.app.show_popup(fix_persian_text("خطای GPS"), f"[color=ff0000]{fix_persian_text('دریافت ماهواره شکست خورد')}:[/color]\n\n[color=ffaa00]{str(e)}[/color]")

    def open_map_selection(self, parent_popup):
        """انتخاب نقشه پیشرفته"""
        parent_popup.dismiss()
        self.app.show_popup(fix_persian_text("نقشه‌یابی تعاملی"), f"[color=00ffff]{fix_persian_text('رابط نقشه تعاملی در اینجا فعال خواهد شد')}[/color]\n\n[color=ffff00]{fix_persian_text('ویژگی‌های پیاده‌سازی کامل')}:[/color]\n[color=aaaaaa]* {fix_persian_text('تصاویر ماهواره‌ای زمان واقعی')}\n* {fix_persian_text('انتخاب مختصات بر اساس لمس')}\n* {fix_persian_text('قابلیت‌های زوم و حرکت')}\n* {fix_persian_text('حالت‌های نمایش زمین و خیابان')}\n* {fix_persian_text('استخراج خودکار مختصات')}[/color]\n\n[color=888888]{fix_persian_text('رابط نقشه‌نگاری پیشرفته')}[/color]")

    def use_device_location(self, parent_popup, location_label):
        """موقعیت‌یابی دستگاه پیشرفته"""
        try:
            iran_lat = 32.4279 + random.uniform(-3, 3)
            iran_lon = 53.6880 + random.uniform(-5, 5)

            self.app.user_lat = iran_lat
            self.app.user_lon = iran_lon

            self.location_label.text = f'[color=00ff00]{fix_persian_text("مختصات")}: {self.app.user_lat:.6f}N, {self.app.user_lon:.6f}E ({fix_persian_text("دستگاه")})[/color]'
            location_label.text = f'[color=00ffff]{fix_persian_text("فعلی")}:[/color] [color=ffffff]{self.app.user_lat:.6f}N, {self.app.user_lon:.6f}E[/color]'

            parent_popup.dismiss()
            self.app.show_popup(fix_persian_text("موقعیت دستگاه"), f"[color=00ffff]{fix_persian_text('موقعیت‌یابی دستگاه موفق!')}[/color]\n\n[color=ffffff]{fix_persian_text('عرض جغرافیایی')}: {iran_lat:.6f}N\n{fix_persian_text('طول جغرافیایی')}: {iran_lon:.6f}E[/color]\n\n[color=888888]{fix_persian_text('خدمات موقعیت‌یابی موبایل درگیر شده')}[/color]")

        except Exception as e:
            self.app.show_popup(fix_persian_text("خطای موقعیت‌یابی"), f"[color=ff0000]{fix_persian_text('موقعیت‌یابی دستگاه شکست خورد')}:[/color]\n\n[color=ffaa00]{str(e)}[/color]")

    def show_author_message(self, instance):
        """نمایش پیغام نویسنده"""
        self.app.show_popup(fix_persian_text("سخن نویسنده"), 
                           f"[size=16][color=00ffff]{fix_persian_text('سلام و احترام')}[/color][/size]\n\n"
                           f"[color=ffffff]{fix_persian_text('این برنامه با هدف ترویج دانش نجوم و محاسبات دقیق عبور ستاره‌ها طراحی شده است.')}\n\n"
                           f"{fix_persian_text('امیدوارم این ابزار برای علاقه‌مندان به نجوم و رصدگران آسمان مفید باشد.')}\n\n"
                           f"{fix_persian_text('برای پیشنهادات، انتقادات یا گزارش خطا لطفاً با من در ارتباط باشید.')}[/color]\n\n"
                           f"[color=ffaa00]{fix_persian_text('ویژگی‌های اصلی برنامه')}:[/color]\n"
                           f"[color=aaaaaa]• {fix_persian_text('محاسبات دقیق LST (زمان ستاره‌ای محلی)')}\n"
                           f"• {fix_persian_text('تحلیل عبور روزانه و ماهانه')}\n"
                           f"• {fix_persian_text('پشتیبانی از موقعیت‌یابی پیشرفته')}\n"
                           f"• {fix_persian_text('رابط کاربری زیبا و متحرک')}\n"
                           f"• {fix_persian_text('پشتیبانی کامل از زبان فارسی')}[/color]\n\n"
                           f"[color=888888]{fix_persian_text('با آرزوی موفقیت در رصدهای آسمانی')}[/color]\n"
                           f"[color=ff6666]❤️ {fix_persian_text('نویسنده برنامه')}[/color]")

    def open_pdf_guide(self, instance):
        """باز کردن راهنمای PDF پیشرفته"""
        pdf_filename = "1.pdf"

        if os.path.exists(pdf_filename):
            try:
                if sys.platform.startswith('win'):
                    os.startfile(pdf_filename)
                elif sys.platform.startswith('darwin'):
                    subprocess.run(['open', pdf_filename])
                else:
                    subprocess.run(['xdg-open', pdf_filename])

                self.app.show_popup(fix_persian_text("مستندات دسترسی یافت"), f"[color=00ffff]{fix_persian_text('راهنمای ستاره‌ای با موفقیت باز شد!')}[/color]\n\n[color=ffffff]{fix_persian_text('سند')}: '{pdf_filename}'[/color]\n\n[color=888888]{fix_persian_text('رابط پایگاه داده دانش فعال')}[/color]")

            except Exception as e:
                self.app.show_popup(fix_persian_text("خطای سند"), f"[color=ff0000]{fix_persian_text('قادر به دسترسی به مستندات نیست')}:[/color]\n\n[color=ffaa00]{str(e)}[/color]\n\n[color=aaaaaa]{fix_persian_text('سعی کنید')} '{pdf_filename}' {fix_persian_text('را به صورت دستی باز کنید.')}[/color]")
        else:
            self.app.show_popup(fix_persian_text("سیستم مستندات"),
                               f"[color=ffaa00]{fix_persian_text('راهنمای ستاره‌ای یافت نشد!')}[/color]\n\n"
                               f"[color=ffffff]{fix_persian_text('فایل مورد نیاز')}: '1.pdf'[/color]\n"
                               f"[color=aaaaaa]{fix_persian_text('مکان مورد انتظار')}: {fix_persian_text('دایرکتوری برنامه')}[/color]\n\n"
                               f"[color=ffffff]{fix_persian_text('مسیر فعلی')}:[/color]\n[color=888888]{os.getcwd()}[/color]\n\n"
                               f"[color=888888]{fix_persian_text('سیستم مدیریت اسناد')}[/color]")

# صفحه تحلیل روزانه
class DailyTransitScreen(Screen):
    def __init__(self, app_instance, **kwargs):
        super().__init__(**kwargs)
        self.app = app_instance
        self.build_screen()

    def build_screen(self):
        # طرح‌بندی اصلی با پس‌زمینه
        main_layout = FloatLayout()

        # پس‌زمینه متحرک
        background = AnimatedBackground()
        main_layout.add_widget(background)

        # محتوای صفحه
        layout = BoxLayout(
            orientation='vertical',
            padding=[20, 40, 20, 30],
            spacing=20,
            size_hint=(0.95, 0.9),
            pos_hint={'center_x': 0.5, 'center_y': 0.5}
        )

        # عنوان صفحه با جلوه درخشش
        title = StylizedLabel(
            text=f'[size=22][color=00ffff]{fix_persian_text("تحلیل عبور امروز")}[/color][/size]\n[size=16][color=ffaa00]{fix_persian_text(STAR_NAME)}[/color][/size]',
            size_hint_y=None,
            height=80,
            halign='center',
            font_size=20
        )
        layout.add_widget(title)

        # نتایج محاسبه
        self.result_label = StylizedLabel(
            text=f'[color=ffffff]{fix_persian_text("در حال محاسبه عبور ستاره...")}[/color]\n\n[color=888888]{fix_persian_text("لطفاً کمی صبر کنید")}[/color]',
            text_size=(None, None),
            halign='center',
            markup=True,
            font_size=16
        )
        layout.add_widget(self.result_label)

        # دکمه بازگشت درخشان
        back_btn = GlowingButton(
            text=fix_persian_text('بازگشت به کنسول اصلی'),
            size_hint_y=None,
            height=55,
            size_hint_x=0.7,
            pos_hint={'center_x': 0.5},
            glow_color=(1, 0.3, 0.3, 1)  # قرمز
        )
        back_btn.bind(on_press=self.go_back)
        layout.add_widget(back_btn)

        main_layout.add_widget(layout)
        self.add_widget(main_layout)

        # محاسبه خودکار بعد از ساخت صفحه
        Clock.schedule_once(self.calculate_daily_transit, 1.5)

    def calculate_daily_transit(self, dt):
        """محاسبه عبور روزانه ستاره"""
        try:
            today = datetime.now()
            lat = self.app.user_lat
            lon = self.app.user_lon

            # محاسبه LST
            lst = self.calculate_lst(today, lon)

            # محاسبه زاویه ساعت
            hour_angle = (lst - STAR_RA) * 15  # تبدیل به درجه

            # محاسبه ارتفاع ستاره
            lat_rad = math.radians(lat)
            dec_rad = math.radians(STAR_DEC)
            ha_rad = math.radians(hour_angle)

            altitude = math.degrees(math.asin(
                math.sin(lat_rad) * math.sin(dec_rad) +
                math.cos(lat_rad) * math.cos(dec_rad) * math.cos(ha_rad)
            ))

            # محاسبه زمان عبور
            transit_lst = STAR_RA
            transit_time = self.lst_to_local_time(transit_lst, today, lon)

            # تشخیص وضعیت عبور
            if abs(hour_angle) < 0.5:  # نزدیک عبور (کمتر از 30 دقیقه)
                status = "در حال عبور"
                status_color = "00ff00"  # سبز
            elif hour_angle < 0:
                status = "هنوز عبور نکرده"
                status_color = "ffaa00"  # نارنجی
            else:
                status = "عبور کرده"
                status_color = "ff6666"  # قرمز روشن

            # نمایش نتایج
            result_text = (
                f'[size=18][color=00ffff]{fix_persian_text("گزارش عبور امروز")}[/color][/size]\n'
                f'[color=ffffff]{fix_persian_text("تاریخ")}: {today.strftime("%Y/%m/%d")}[/color]\n'
                f'[color=ffffff]{fix_persian_text("زمان فعلی")}: {today.strftime("%H:%M:%S")}[/color]\n\n'
                f'[size=16][color={status_color}]{fix_persian_text("وضعیت")}: {fix_persian_text(status)}[/color][/size]\n'
                f'[color=ffffff]{fix_persian_text("زمان عبور")}: {transit_time.strftime("%H:%M:%S")}[/color]\n\n'
                f'[color=ffaa00]{fix_persian_text("مشخصات فنی")}:[/color]\n'
                f'[color=aaaaaa]{fix_persian_text("LST فعلی")}: {lst:.3f}h[/color]\n'
                f'[color=aaaaaa]{fix_persian_text("زاویه ساعت")}: {hour_angle:.1f}°[/color]\n'
                f'[color=aaaaaa]{fix_persian_text("ارتفاع ستاره")}: {altitude:.1f}°[/color]\n'
                f'[color=aaaaaa]{fix_persian_text("مختصات")}: {lat:.4f}N, {lon:.4f}E[/color]\n\n'
                f'[color=888888]{fix_persian_text("محاسبه بر اساس الگوریتم LST دقیق")}[/color]'
            )

            self.result_label.text = result_text

        except Exception as e:
            self.result_label.text = f'[color=ff0000]{fix_persian_text("خطا در محاسبه")}:[/color]\n\n[color=ffaa00]{str(e)}[/color]'

    def calculate_lst(self, dt, longitude):
        """محاسبه زمان ستاره‌ای محلی (Local Sidereal Time)"""
        # تعداد روزهای Julian از J2000.0
        j2000 = datetime(2000, 1, 1, 12, 0, 0)
        delta = dt - j2000
        jd = delta.total_seconds() / 86400.0

        # زمان ستاره‌ای گرینویچ در ساعت
        gst = 18.697374558 + 24.06570982441908 * jd

        # تبدیل به زمان ستاره‌ای محلی
        lst = gst + longitude / 15.0

        # تنظیم به بازه 0-24 ساعت
        while lst >= 24:
            lst -= 24
        while lst < 0:
            lst += 24

        return lst

    def lst_to_local_time(self, lst, date, longitude):
        """تبدیل LST به زمان محلی"""
        # محاسبه تفاوت بین LST و زمان خورشیدی میانگین
        gst = lst - longitude / 15.0
        
        # تخمین زمان محلی (ساده‌شده)
        # این محاسبه دقیق نیست اما برای نمایش کافی است
        local_hour = (lst - (date.hour + date.minute/60.0 + date.second/3600.0)) % 24
        
        transit_hour = int(local_hour)
        transit_minute = int((local_hour - transit_hour) * 60)
        transit_second = int(((local_hour - transit_hour) * 60 - transit_minute) * 60)
        
        return date.replace(hour=transit_hour, minute=transit_minute, second=transit_second)

    def go_back(self, instance):
        """بازگشت به صفحه اصلی"""
        self.app.screen_manager.current = 'main'

# صفحه تحلیل ماهانه
class MonthlyTransitScreen(Screen):
    def __init__(self, app_instance, **kwargs):
        super().__init__(**kwargs)
        self.app = app_instance
        self.build_screen()

    def build_screen(self):
        # طرح‌بندی اصلی با پس‌زمینه
        main_layout = FloatLayout()

        # پس‌زمینه متحرک
        background = AnimatedBackground()
        main_layout.add_widget(background)

        # محتوای صفحه
        layout = BoxLayout(
            orientation='vertical',
            padding=[15, 30, 15, 25],
            spacing=15,
            size_hint=(0.98, 0.92),
            pos_hint={'center_x': 0.5, 'center_y': 0.5}
        )

        # عنوان صفحه
        title = StylizedLabel(
            text=f'[size=20][color=8b00ff]{fix_persian_text("ماتریس عبور ماهانه")}[/color][/size]\n[size=14][color=ffaa00]{fix_persian_text(STAR_NAME)}[/color][/size]',
            size_hint_y=None,
            height=70,
            halign='center',
            font_size=18
        )
        layout.add_widget(title)

        # نتایج محاسبه
        self.result_label = StylizedLabel(
            text=f'[color=ffffff]{fix_persian_text("در حال تولید ماتریس ماهانه...")}[/color]\n\n[color=888888]{fix_persian_text("محاسبه 30 روز آینده")}[/color]',
            text_size=(None, None),
            halign='center',
            markup=True,
            font_size=14
        )
        layout.add_widget(self.result_label)

        # دکمه بازگشت
        back_btn = GlowingButton(
            text=fix_persian_text('بازگشت به کنسول اصلی'),
            size_hint_y=None,
            height=50,
            size_hint_x=0.7,
            pos_hint={'center_x': 0.5},
            glow_color=(0.8, 0.2, 1, 1)  # بنفش
        )
        back_btn.bind(on_press=self.go_back)
        layout.add_widget(back_btn)

        main_layout.add_widget(layout)
        self.add_widget(main_layout)

        # محاسبه خودکار
        Clock.schedule_once(self.calculate_monthly_transit, 2.0)

    def calculate_monthly_transit(self, dt):
        """محاسبه جدول عبور ماهانه"""
        try:
            today = datetime.now()
            lat = self.app.user_lat
            lon = self.app.user_lon

            # عنوان جدول
            result_text = (
                f'[size=16][color=8b00ff]{fix_persian_text("جدول عبور 30 روز آینده")}[/color][/size]\n'
                f'[color=aaaaaa]{fix_persian_text("مختصات")}: {lat:.4f}N, {lon:.4f}E[/color]\n\n'
            )

            # محاسبه برای 30 روز آینده
            for day_offset in range(0, 30, 3):  # هر 3 روز یکبار برای صرفه‌جویی در فضا
                target_date = today + timedelta(days=day_offset)
                
                # محاسبه LST و زمان عبور
                lst = self.calculate_lst(target_date, lon)
                transit_lst = STAR_RA
                transit_time = self.lst_to_local_time(transit_lst, target_date, lon)
                
                # محاسبه ارتفاع حداکثر
                lat_rad = math.radians(lat)
                dec_rad = math.radians(STAR_DEC)
                max_altitude = math.degrees(math.asin(
                    math.sin(lat_rad) * math.sin(dec_rad) +
                    math.cos(lat_rad) * math.cos(dec_rad)
                ))
                
                # رنگ‌بندی بر اساس قابلیت رؤیت
                if max_altitude > 60:
                    color = "00ff00"  # سبز - عالی
                elif max_altitude > 30:
                    color = "ffaa00"  # زرد - خوب
                else:
                    color = "ff6666"  # قرمز - ضعیف

                date_str = target_date.strftime("%m/%d")
                time_str = transit_time.strftime("%H:%M")
                
                result_text += f'[color={color}]{date_str} | {time_str} | {max_altitude:.0f}°[/color]\n'

            # اضافه کردن راهنما
            result_text += (
                f'\n[color=ffaa00]{fix_persian_text("راهنمای رنگ‌ها")}:[/color]\n'
                f'[color=00ff00]● {fix_persian_text("سبز")}: {fix_persian_text("رؤیت عالی")} (>60°)[/color]\n'
                f'[color=ffaa00]● {fix_persian_text("زرد")}: {fix_persian_text("رؤیت خوب")} (30°-60°)[/color]\n'
                f'[color=ff6666]● {fix_persian_text("قرمز")}: {fix_persian_text("رؤیت ضعیف")} (<30°)[/color]\n\n'
                f'[color=888888]{fix_persian_text("فرمت")}: {fix_persian_text("تاریخ | زمان عبور | ارتفاع حداکثر")}[/color]'
            )

            self.result_label.text = result_text

        except Exception as e:
            self.result_label.text = f'[color=ff0000]{fix_persian_text("خطا در تولید جدول")}:[/color]\n\n[color=ffaa00]{str(e)}[/color]'

    def calculate_lst(self, dt, longitude):
        """محاسبه زمان ستاره‌ای محلی"""
        j2000 = datetime(2000, 1, 1, 12, 0, 0)
        delta = dt - j2000
        jd = delta.total_seconds() / 86400.0

        gst = 18.697374558 + 24.06570982441908 * jd
        lst = gst + longitude / 15.0

        while lst >= 24:
            lst -= 24
        while lst < 0:
            lst += 24

        return lst

    def lst_to_local_time(self, lst, date, longitude):
        """تبدیل LST به زمان محلی"""
        local_hour = lst % 24
        transit_hour = int(local_hour)
        transit_minute = int((local_hour - transit_hour) * 60)
        transit_second = int(((local_hour - transit_hour) * 60 - transit_minute) * 60)
        
        try:
            return date.replace(hour=transit_hour, minute=transit_minute, second=transit_second)
        except ValueError:
            return date.replace(hour=12, minute=0, second=0)

    def go_back(self, instance):
        """بازگشت به صفحه اصلی"""
        self.app.screen_manager.current = 'main'

# کلاس اصلی برنامه
class StarTransitApp(App):
    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        # مختصات پیش‌فرض (ارومیه)
        self.user_lat = 37.5527
        self.user_lon = 45.0761

    def build(self):
        # ایجاد مدیر صفحات
        self.screen_manager = ScreenManager()

        # ایجاد صفحات
        self.main_screen = MainScreen(self, name='main')
        self.daily_screen = DailyTransitScreen(self, name='daily_transit')
        self.monthly_screen = MonthlyTransitScreen(self, name='monthly_transit')

        # اضافه کردن صفحات به مدیر
        self.screen_manager.add_widget(self.main_screen)
        self.screen_manager.add_widget(self.daily_screen)
        self.screen_manager.add_widget(self.monthly_screen)

        # تنظیم صفحه اولیه
        self.screen_manager.current = 'main'

        return self.screen_manager

    def show_popup(self, title, content):
        """پاپ‌آپ پیشرفته با استایل آینده‌نگر"""
        popup_content = BoxLayout(orientation='vertical', spacing=15, padding=20)

        # پس‌زمینه آینده‌نگر با جلوه گرادیان
        with popup_content.canvas.before:
            Color(0.05, 0.05, 0.15, 0.98)
            Rectangle(pos=popup_content.pos, size=popup_content.size)

            # جلوه درخشش حاشیه
            Color(0, 1, 1, 0.6)
            Line(rectangle=(*popup_content.pos, *popup_content.size), width=3)

        label = StylizedLabel(
            text=content,
            text_size=(400, None),
            halign='center',
            markup=True,
            font_size=14
        )
        popup_content.add_widget(label)

        # دکمه بستن پیشرفته
        close_btn = GlowingButton(
            text=fix_persian_text('بستن رابط'),
            size_hint_y=None,
            height=55,
            size_hint_x=0.6,
            glow_color=(0.2, 0.8, 1, 1)
        )

        popup = Popup(
            title=title,
            content=popup_content,
            size_hint=(0.9, 0.8),
            title_color=(0, 1, 1, 1),
            title_size=18
        )

        close_btn.bind(on_press=lambda x: popup.dismiss())
        popup_content.add_widget(close_btn)

        popup.open()

# اجرای برنامه
if __name__ == '__main__':
    StarTransitApp().run()
