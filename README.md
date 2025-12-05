# UCA-Hackathon
First task/
# main.py
import os
import json
from datetime import datetime

from kivy.app import App
from kivy.uix.boxlayout import BoxLayout
from kivy.uix.screenmanager import ScreenManager, Screen, SlideTransition
from kivy.properties import (
    ListProperty,
    NumericProperty,
    StringProperty,
    ObjectProperty,
)
from kivy.clock import Clock
from kivy.lang import Builder
from kivy.uix.screenmanager import Screen
from kivy.uix.videoplayer import VideoPlayer
from data import universities, get_programs_for_university, find_university

# ========= КОНСТАНТЫ =========
PROFILE_FILE = "user_profile.json"


# ========= ЛОГИКА ИИ-ЧАТА =========

def simple_ai_answer(message: str) -> str:
    msg = message.lower()

    if "грант" in msg or "стипенд" in msg:
        return ("По грантам: многие вузы РК дают госгранты и внутренние стипендии. "
                "Откройте карточку вуза → раздел «Прием и поступление» → «Стипендии».")
    if "it" in msg or "айти" in msg or "программи" in msg or "computer science" in msg:
        return ("Для IT посмотрите программы «Computer Science» и «Информационные системы». "
                "В меню выберите «Университеты» и используйте поиск.")
    if "алматы" in msg:
        return ("Чтобы увидеть вузы Алматы, перейдите в раздел «Университеты» и отфильтруйте по городу Алматы.")
    if "астан" in msg or "нур-султан" in msg:
        return ("В Астане один из ключевых вузов — Nazarbayev University. "
                "Сравните его с другими через раздел «Сравнение» в меню.")
    if "как выбрать" in msg or "выбрать университет" in msg:
        return ("Сначала определите направление (IT, экономика, медицина), затем "
                "используйте фильтры в разделе «Университеты» и функцию сравнения.")
    return ("Я — ассистент UCA. Спросите меня о вузах РК, программах, поступлении, грантах "
            "или сравните несколько университетов.")


def gemini_answer_or_fallback(message: str) -> str:
    """
    Заготовка под API Gemini.
    Чтобы использовать:
      1) pip install google-generativeai
      2) вставить свой API-ключ вместо 'YOUR_GEMINI_API_KEY'
    Если что-то пойдет не так, вернется simple_ai_answer().
    """
    try:
        import google.generativeai as genai

        genai.configure(api_key="YOUR_GEMINI_API_KEY")  # TODO: заменить ключ
        model = genai.GenerativeModel("gemini-1.5-flash")
        resp = model.generate_content(message)
        text = getattr(resp, "text", None)
        if text:
            return text.strip()
        return simple_ai_answer(message)
    except Exception:
        # На хакатоне, если нет интернета/библиотеки, приложение не сломается.
        return simple_ai_answer(message)


# ========= ЭКРАНЫ =========

class OnboardingScreen(Screen):
    """Первичный опросник: имя, направление, город, бюджет, язык, гранты."""

    def complete_profile(self):
        app = App.get_running_app()

        name = self.ids.name_input.text.strip() or "Абитуриент"

        # Направление
        if self.ids.field_it.state == "down":
            field = "IT"
        elif self.ids.field_business.state == "down":
            field = "Бизнес/экономика"
        elif self.ids.field_med.state == "down":
            field = "Медицина"
        elif self.ids.field_engineering.state == "down":
            field = "Инженерия"
        else:
            field = "Не выбрано"

        # Город
        if self.ids.city_almaty.state == "down":
            city = "Алматы"
        elif self.ids.city_astana.state == "down":
            city = "Астана"
        elif self.ids.city_shymkent.state == "down":
            city = "Шымкент"
        else:
            city = "Любой город"
