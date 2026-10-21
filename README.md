[IA LOCAL PARA MOVLIES CON KIVY (1).py]
# ia-local--ProtoTipo
Movli Memoria Largo Con python  y Sql y API Firbase.
import os
import re
import threading
import datetime
import sqlite3
import numpy as np
import json
import time
import uuid
import random
import shutil
import sys
import requests
import uuid
import gc
import datetime, requests
import atexit
import platform
import multiprocessing
import traceback
import re
import unicodedata
import difflib
import hashlib


from kivy.app import App
from kivy.uix.boxlayout import BoxLayout
from kivy.uix.gridlayout import GridLayout
from kivy.uix.scrollview import ScrollView
from kivy.uix.textinput import TextInput
from kivy.uix.button import Button
from kivy.uix.label import Label
from kivy.core.window import Window
from kivy.utils import get_color_from_hex
from kivy.animation import Animation
from kivy.uix.popup import Popup 
from threading import Thread, Lock
from kivy.uix.image import Image
from random import choice
from threading import Thread
from kivy.clock import Clock
from llama_cpp import Llama
from kivy.graphics import Color, Ellipse
from kivy.uix.widget import Widget
from multiprocessing.managers import SyncManager
from kivy.utils import platform as kivy_platform
from plyer import storagepath
from pathlib import Path
from kivy.graphics import Color, Rectangle, Ellipse
from kivymd.uix.label import MDLabel
from kivy.graphics import Color, RoundedRectangle
from kivy.core.clipboard import Clipboard
from kivy.metrics import dp
from kivy.lang import Builder
from datetime import datetime
from threading import Thread
from difflib import SequenceMatcher
from kivy.uix.screenmanager import ScreenManager
from kivy.app import App
from kivy.uix.floatlayout import FloatLayout
from threading import Thread
from llama_cpp import Llama
from kivy.uix.scrollview import ScrollView
from kivy.clock import Clock










# Diccionario para almacenar la memoria de cada usuario

def similitud(a, b):
    return SequenceMatcher(None, a, b).ratio()







Builder.load_string("""
<ChatScreen>:
    BoxLayout:
        orientation: 'vertical'

        ScrollView:
            id: scroll
            do_scroll_x: False
            do_scroll_y: True

            GridLayout:
                id: contenedor_chat
                cols: 1
                size_hint_y: None
                height: self.minimum_height
                spacing: 10
                padding: [10, 20, 10, 20]    

        BoxLayout:
            size_hint_y: None
            height: 50
            spacing: 5
""")



def get_model_path():

    import shutil
    from kivy.utils import platform # Forma elegante de saber si es Android

    # 1. Definir carpeta de almacenamiento según el sistema
    if platform == 'android':
        from android.storage import app_storage_path # type: ignore
        base_dir = app_storage_path()
    else:
        # En PC lo guarda en la carpeta del proyecto
        base_dir = os.path.dirname(os.path.abspath(__file__))

    model_dir = os.path.join(base_dir, "models")
    os.makedirs(model_dir, exist_ok=True)

    model_name = "llama-3.2-1b-instruct-q4_k_m.gguf"
    model_path = os.path.join(model_dir, model_name)

    # 2. Lógica de copia (Assets -> Carpeta de ejecución)
    # Esto es vital para Android, porque los assets son de 'solo lectura'
    origen = os.path.join(os.path.dirname(os.path.abspath(__file__)), "assets", model_name)
    
    if not os.path.exists(model_path):
        if os.path.exists(origen):
            print(f"📦 Copiando modelo a: {model_path}...")
            shutil.copy(origen, model_path)
            print("✅ Copia completada")
        else:
            print("⚠️ ADVERTENCIA: No se encontró el modelo en /assets/")

    return model_path

# --- Uso ---

MODEL_PATH = get_model_path()



# --- CONFIGURACIÓN ÚNICA DE RUTAS ---
def get_app_paths():
    # Detectamos si es Android o PC
    if platform == "android":
        from android.storage import app_storage_path # type: ignore
        base = app_storage_path()
    else:
        # En PC lo guarda en la carpeta donde esté el script
        base = os.path.abspath(os.path.dirname(__file__))
    
    os.makedirs(base, exist_ok=True)
    
    return {
        "db": os.path.join(base, "dyslexialift.db"),
        "model": os.path.join(base, "models", "llama-3.2-1b-instruct-q4_k_m.gguf"),
        "user_data": os.path.join(base, "user_data.json")
    }

# Variables maestras (Las únicas que usará el resto del código)
PATHS = get_app_paths()
DB_PATH = PATHS["db"]
MODEL_PATH = PATHS["model"]
USER_DATA_PATH = PATHS["user_data"]
TOKEN_DE_SERVICIO = ""

def crear_tablas():
    """Crea la estructura de la base de datos si no existe."""
    with sqlite3.connect(DB_PATH) as conn:
        c = conn.cursor()
        c.execute('''
            CREATE TABLE IF NOT EXISTS mensajes (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id TEXT,
                emisor TEXT,
                mensaje TEXT,
                timestamp DATETIME,
                importancia INTEGER DEFAULT 0,
                tema_detectado TEXT,
                emocion TEXT
            )
        ''')
        conn.commit()


def iniciar_aria(self):
    print("✅ Aria lista (modo app)")

     # --- Ejecutar Aria en segundo plano ---
    hilo_aria = threading.Thread(target=self.iniciar_aria, daemon=True)
    hilo_aria.start() 










# 🔹 Verificar si el modelo existe
if not os.path.exists(MODEL_PATH):
    print(f"⚠️ No se encontró el modelo en: {MODEL_PATH}")
else:
    print(f"✅ Cargando modelo desde: {MODEL_PATH}")


FIREBASE_URL = ""


def enviar_datos_firebase(ruta, data, id_token):
    """Envía datos a Firebase en un hilo separado para no bloquear la UI"""
    def enviar():
        url = f"{FIREBASE_URL}{ruta}.json?auth={id_token}"
        try:
            response = requests.post(url, json=data)
            if response.status_code == 200:
                print("✅ Datos enviados correctamente:", data)
            else:
                print(f"⚠️ Error {response.status_code}: {response.text}")
        except Exception as e:
            print("❌ Error al conectar con Firebase:", e)

    # Ejecutar en hilo para no bloquear UI
    Thread(target=enviar, daemon=True).start()

# --- Detectar memoria total del dispositivo (compatible con Android) ---
def obtener_memoria_total_gb():
    try:
        # Esto funciona en Android (Linux)
        if os.path.exists("/proc/meminfo"):
            with open("/proc/meminfo", "r") as f:
                primera_linea = f.readline()
                mem_kb = int(primera_linea.split()[1])
                return mem_kb / (1024 ** 2)
        return 4.0  # Valor por defecto si no puede leer (PC o similar)
    except Exception as e:
        print(f"No se pudo leer memoria del sistema: {e}")
        return 4.0
    
mem_total_gb = obtener_memoria_total_gb()

def tokens_por_tipo(tipo):
    if modo_ultra:
        return 80 if tipo == "corto" else 120

    if mem_total_gb < 6:
        return 100 if tipo == "corto" else 150

    elif mem_total_gb < 8:
        return 120 if tipo == "corto" else 180

    else:
        return 150 if tipo == "corto" else 220

if platform == "android":
    # Usamos máximo 4 hilos para no sobrecargar el procesador del móvil
    n_threads = min(4, os.cpu_count() or 4)
    print(f"📱 Modo Android: Usando {n_threads} hilos")
else:
    # En PC podemos ser un poco más generosos si quieres
    n_threads = 4
    print(f"💻 Modo PC: Usando {n_threads} hilos")

# --- GPU (en móviles casi siempre False con llama.cpp) ---
gpu_available = False
n_gpu_layers = 0

modo_ultra = mem_total_gb < 4


if modo_ultra:
    n_threads = 1
    n_batch = 8
    n_ctx = 384

elif mem_total_gb < 6:
    n_threads = 2
    n_batch = 8
    n_ctx = 512

elif mem_total_gb < 8:
    n_threads = 3
    n_batch = 12
    n_ctx = 512

else:
    n_threads = 4
    n_batch = 16
    n_ctx = 512   # 🔥 NO subir más para 1B

llama_params = {
    "n_ctx": n_ctx,
    "n_threads": n_threads,
    "n_batch": n_batch,
    "n_gpu_layers": 0,

    "use_mmap": True,
    "use_mlock": False,
    "f16_kv": True,

    "logits_all": False,
    "embedding": False,
}


try:
    os.environ["OMP_NUM_THREADS"] = str(n_threads)
except:
    pass

import time
time.sleep(0.01)







# --- Base de datos ---
lock_db = threading.Lock()


def crear_tablas():
    with lock_db:
        with sqlite3.connect(DB_PATH) as conn:
            c = conn.cursor()
            c.execute('''
                CREATE TABLE IF NOT EXISTS mensajes (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    user_id TEXT,
                    emisor TEXT,
                    mensaje TEXT,
                    timestamp DATETIME,
                    importancia INTEGER DEFAULT 0,
                    tema_detectado TEXT,
                    emocion TEXT
                )
            ''')
            conn.commit()


lock_db = threading.Lock()

# --- Corrección de texto ---
errores_comunes = {
    r'\bdigen\b': 'dicen',
    r'\bpueds\b': 'puedes',
    r'\bami\b': 'a mí',
    r'\bq\b': 'que',
    r'\bxq\b': 'porque',
    r'\btmb\b': 'también',
}

def corregir_texto(texto):
    texto_corregido = texto.lower()
    for err, corr in errores_comunes.items():
        texto_corregido = re.sub(err, corr, texto_corregido)
    return texto_corregido






def detectar_emocion(mensaje):
    emociones = {
        'feliz': ['feliz','contento','alegre','bien'],
        'triste': ['triste','deprimido','mal','solo'],
        'ansioso': ['ansioso','nervioso','preocupado']
    }
    mensaje_lower = mensaje.lower()
    for e, palabras in emociones.items():
        if any(p in mensaje_lower for p in palabras):
            return e
    return 'neutral'


# --- Recursos y recordatorios ---
recursos = {
    "lectura":["Lee un párrafo y explica lo entendido.","Video: Técnicas de comprensión."],
    "escritura":["Escribe un texto corrigiendo ortografía.","Guía básica de ortografía."],
    "emocional":["Respira profundo 5 min.","Meditación guiada."],
    "motivación":["Cada avance es un logro.","¡Podés lograrlo!"],
    "general":["Practica la lectura hoy.","Descansa y cuida tu bienestar."]
}

recordatorios = {
    "lectura":"Practica la lectura hoy.",
    "escritura":"Practica la escritura para mejorar.",
    "emocional":"Respira y relajate si te sentís mal.",
    "motivación":"¡Seguí así!",
    "general":"Cuida tu bienestar general."
}




# --- DETECTAR TIPO DE MENSAJE ---
def detectar_tipo(texto):
    texto_low = texto.lower()

    if "hola" in texto_low or "buenas" in texto_low:
        return "saludo"
    if "?" in texto_low:
        return "pregunta"
    if any(cmd in texto_low for cmd in ["configurar", "ajustar", "activar", "guardar"]):
        return "comando"
    return "texto"

# --- GUARDAR EN MEMORIA (VERSIÓN D COMPLETA) ---
def guardar_mensaje_memoria(self, texto, emisor):
    mensaje = {
        "id": str(uuid.uuid4()),
        "timestamp": time.time(),
        "texto": texto,
        "emisor": emisor,
        "tokens": len(texto.split()),
        "longitud": len(texto),
        "tipo": detectar_tipo(texto),
        "preview": texto[:60] + "..." if len(texto) > 60 else texto,
    }

    self.memoria.append(mensaje)   # ✅ CORRECTO

    print("🧠 Mensaje guardado en memoria:")
    print(mensaje)

    return mensaje

def crear_tablas():
    """Crea la tabla desde cero con estructura de texto puro"""
    with sqlite3.connect(DB_PATH) as conn:
        c = conn.cursor()
        # Solo columnas de texto para evitar el error de mismatch
        c.execute('''
            CREATE TABLE IF NOT EXISTS mensajes (
                user_id TEXT,
                emisor TEXT,
                mensaje TEXT,
                timestamp TEXT
            )
        ''')
        conn.commit()

def guardar_mensaje_fisico(user_id, rol, texto):
    try:
        with sqlite3.connect(DB_PATH) as conn:
            c = conn.cursor()
            c.execute("""
                INSERT INTO mensajes (user_id, rol, mensaje, timestamp)
                VALUES (?, ?, ?, ?)
            """, (user_id, rol, texto, datetime.now().strftime("%Y-%m-%d %H:%M:%S")))
            conn.commit()
    except Exception as e:
        print(f"❌ Error guardando en móvil: {e}")

# --- Recordatorios y sugerencias ---
def obtener_recordatorio(tema, emocion):
    if emocion in ['triste','ansioso']:
        return "Respira y relajate un momento."
    return recordatorios.get(tema, recordatorios['general'])

def obtener_sugerencias(tema):
    return recursos.get(tema, recursos['general'])




# Versión corregida (si decides dejarla)
def resumir_texto_llama(self, textos, max_tokens=300, callback=None):
    def tarea():
        # Cambiamos 'llm' por tu variable real del modelo
        if self.modelo_llama is None:
            resumen = "Aria no está lista para resumir."
        else:
            prompt = f"Resume brevemente estos mensajes:\n{chr(10).join(textos)}\nResumen:"
            try:
                # Usamos tu objeto de modelo real
                salida = self.modelo_llama.generar_respuesta(prompt) 
                resumen = salida
            except:
                resumen = "No pude resumir eso ahora."
        
        if callback:
            # Volvemos al hilo principal de Kivy para mostrar el texto
            Clock.schedule_once(lambda dt: callback(resumen), 0)

        Thread(target=tarea, daemon=True).start()

class ChatScreen(BoxLayout): # Volvemos a BoxLayout para que sea vertical automático
    def __init__(self, usuario_id, llm, memoria, db_path=None, user_id=None, **kwargs):
        # 1. El super CORREGIDO para BoxLayout
        super(ChatScreen, self).__init__(**kwargs)
     

        self.orientation = 'vertical'
        self.size_hint = (1, 1)
        self.padding = dp(5)
        self.spacing = dp(5)

        # Evita que el teclado tape el input
        Window.softinput_mode = "below_target"

        # 2. SCROLL PRINCIPAL (La ventana de visualización)
        # Ocupa el 88% de la pantalla para dejar espacio al input abajo
        self.scroll = ScrollView(
            size_hint=(1, 0.88),
            do_scroll_x=False,
            do_scroll_y=True,
            bar_width=dp(4),
            scroll_type=['bars', 'content'],
            effect_cls="ScrollEffect" # Hace que el rebote sea más suave
        )

        # 3. CONTENEDOR DE MENSAJES (El que crece infinitamente)
        self.chat_container = BoxLayout(
            orientation='vertical',
            size_hint_y=None,
            spacing=dp(12),
            padding=[dp(10), dp(30)] # 30 de padding abajo para que el scroll sea más "largo"
        )

        # Vínculo vital: El contenedor ajusta su altura automáticamente al sumar mensajes
        self.chat_container.bind(
            minimum_height=self.chat_container.setter('height')
        )

        # Metemos el contenedor dentro del scroll
        self.scroll.add_widget(self.chat_container)

        # 4. ÁREA DE ENTRADA (Input Box)
        self.input_box = BoxLayout(
            size_hint=(1, 0.12),
            height=dp(65),
            spacing=dp(8),
            padding=dp(5)
        )

        # 5. CONSTRUCCIÓN FINAL
        # Primero el scroll (arriba), luego el input (abajo)
        self.add_widget(self.scroll)
        self.add_widget(self.input_box)
        
       
        # 1. Definimos la ruta del modelo local
        base_path = os.path.dirname(__file__) 
        self.model_path = os.path.join(base_path, "models", "llama-3.2-1b-instruct-q4_k_m.gguf")
        # 2. REFERENCIAS MAESTRAS (Forzamos texto con str())
        self.usuario_id = str(usuario_id)
        self.user_id = str(user_id) if user_id else str(uuid.uuid4())
        self.modelo_llama = llm
        self.memoria = memoria 
        self.db_path = db_path
        self.memoria.user_id = self.user_id
        # 3. ESTADOS DE CONTROL
        self.detener_respuesta = False
        self.generacion_activa = True
        self.running = False
        self.sidebar_visible = False
        self.llm = llm
        # 4. CONFIGURACIÓN
        self.aria_personalidad = "empática, reflexiva y con tono tranquilo"
        self.intervalo_typing = 0.05
        self.mensaje_actual = ""
        self.historial = []
        self.lock = threading.Lock()
        self.historial_conversacion = []
        self.historial_mensajes = []
        # 5. SYNC Y TEMA
        self.sync_manager = SyncManager(self.user_id)
        self.sync_manager.start_sync()
        modo = self._cargar_modo_tema()
        self.aplicar_modo_tema(modo)
        self.memoria_chat = []
        # Busca esta línea en tu __init__ y asegúrate de que sea igual:
        self.chat_container.bind(minimum_height=self.chat_container.setter('height'))
        # Esto hace que el scroll baje hasta el último mensaje automáticamente

        # 6. CONSTRUCCIÓN Y VERIFICACIÓN
        self._crear_layout()
        self.agregar_aviso_privacidad_fijo()
        
        # IMPORTANTE: Verificamos la DB después de tener todo configurado
        self.verificar_y_actualizar_db()
        
        self.obtener_info_dislexia()
        Clock.schedule_once(self._inicializar_contenido, 0.1)


    # ================================================================
    # CREAR LAYOUT PRINCIPAL
    # ================================================================
    def _crear_layout(self):
        # 1. LIMPIEZA INICIAL (Solo una vez al principio)
        self.clear_widgets()
        Clock.unschedule(self._inicializar_contenido)
        Clock.unschedule(self._agregar_advertencias)
        
        self.orientation = 'vertical'
        self.spacing = 0
        self.padding = [0, 0, 0, dp(5)] 

        # 2. ÁREA DE MENSAJES (Arriba)
        # Importante: size_hint_y=1 hace que empuje todo hacia abajo
        self.chat_panel = BoxLayout(orientation='vertical', size_hint_y=1)
        self.add_widget(self.chat_panel)
        self._crear_area_mensajes() 

        # 3. BARRA DE ENTRADA (En el medio)
        barra = self._crear_input_bar()
        if barra:
            barra.size_hint_y = None
            barra.height = dp(65) # Un poquito más alta para móviles
            self.add_widget(barra) 

        # 4. ADVERTENCIA (Al final de todo - El piso)
        self._agregar_advertencias()

        # 5. BOTONES FLOTANTES
        # Se muestran al final para que queden "encima" de lo visual
        self._mostrar_botones_en_chat()
   

     # ================================================================
    # BASE DE DATOS LOCAL
    # ================================================================
    def _crear_tabla_configuracion(self):
        conn = sqlite3.connect(self.db_path)
        c = conn.cursor()
        c.execute('''
            CREATE TABLE IF NOT EXISTS configuracion (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                clave TEXT UNIQUE,
                valor TEXT
            )
        ''')
        conn.commit()
        conn.close()

    
    def _guardar_modo_tema(self, modo):
     try:
        conn = sqlite3.connect(self.db_path)
        c = conn.cursor()
        c.execute('REPLACE INTO configuracion (clave, valor) VALUES (?, ?)', ('modo_tema', modo))
        conn.commit()
        conn.close()
     except sqlite3.OperationalError:
        # Si no existe la tabla, la creamos y volvemos a intentar
        self._crear_tabla_configuracion()
        self._guardar_modo_tema(modo)
    def _cargar_modo_tema(self):
     try:
        conn = sqlite3.connect(self.db_path)
        c = conn.cursor()
        c.execute('SELECT valor FROM configuracion WHERE clave=?', ('modo_tema',))
        row = c.fetchone()
        conn.close()
        return row[0] if row else "oscuro"  # Valor por defecto
     except sqlite3.OperationalError:
        # Si la tabla no existe, devolvemos modo por defecto
        return "oscuro"
    # ================================================================
    # MODO TEMA
    # ================================================================
    def aplicar_modo_tema(self, modo):
        if modo == "oscuro":
            Window.clearcolor = (0.06, 0.06, 0.08, 1)
            self.bg_color_chat = (0.10, 0.10, 0.12, 1)
            self.text_color = (1, 1, 1, 1)
            self.bubble_user_color = (0.20, 0.60, 0.35, 1)
            self.bubble_aria_color = (0.15, 0.35, 0.70, 1)
            self.input_bg = (0.12, 0.12, 0.15, 1)
            self.text_color_chat = (1, 1, 1, 1)
        else:
            Window.clearcolor = (0.95, 0.95, 0.95, 1)
            self.bg_color_chat = (1, 1, 1, 1)
            self.text_color = (0, 0, 0, 1)
            self.bubble_user_color = (0.30, 0.75, 0.40, 1)
            self.bubble_aria_color = (0.30, 0.55, 0.90, 1)
            self.input_bg = (0.90, 0.90, 0.90, 1)
            self.text_color_chat = (0, 0, 0, 1)
        
        # Refrescar la ventana
        Window.canvas.ask_update()
    
   

  
    def _actualizar_colores_chat(self):
         # 1. Seguridad: Si el panel no existe aún, no hacemos nada
         if not hasattr(self, "chat_panel"):
            return

         # 2. Dibujamos el fondo INSTANTÁNEO
         # Limpiamos dibujos anteriores para no acumular capas
         self.chat_panel.canvas.before.clear()
        
         with self.chat_panel.canvas.before:
            # Usamos el color que definimos en aplicar_modo_tema
            Color(*self.bg_color_chat)
            self.bg_rect = Rectangle(
                size=self.chat_panel.size,
                pos=self.chat_panel.pos
            )


         self.chat_panel.bind(size=self._actualizar_bg, pos=self._actualizar_bg)

         # 4. Cambiamos el color de la caja de escritura (Input)
         if hasattr(self, "input"):
            self.input.background_color = self.input_bg
            self.input.foreground_color = self.text_color_chat
            self.input.cursor_color = self.text_color_chat 
  

    # --- PEGA AQUÍ EL PUNTO 2 ---
    def _actualizar_bg(self, instance, value):
        """Mantiene el fondo ajustado al tamaño de la pantalla"""
        if hasattr(self, "bg_rect"):
            self.bg_rect.pos = instance.pos
            self.bg_rect.size = instance.size

    # --- PEGA AQUÍ EL PUNTO 3 ---
    def activar_modo_oscuro(self, *args):
        self.aplicar_modo_tema("oscuro")
        self._actualizar_colores_chat() 
        self._guardar_modo_tema("oscuro")

    def activar_modo_claro(self, *args):
        self.aplicar_modo_tema("claro")
        self._actualizar_colores_chat() 
        self._guardar_modo_tema("claro") 
        
    
    
    # ================================================================
     # BOTONES ESTILO PROFESIONAL MOBILE
    # ================================================================
    def _mostrar_botones_en_chat(self):
        # 1. Siempre creamos un FloatLayout nuevo para que no haya errores de referencia
        self.float_layout = FloatLayout(size_hint=(1, 1))
        self.add_widget(self.float_layout)

        # 🔘 Botón AJUSTES (Arriba Derecha)
        # En móvil usamos texto simple si el emoji falla, pero aquí dejo el emoji
        self.btn_ajustes = Button(
            text="⚙️",
            size_hint=(None, None),
            size=(dp(45), dp(45)),
            pos_hint={'right': 0.98, 'top': 0.98}, # Bien arriba
            background_normal='',
            background_color=(0, 0.5, 0.8, 1),
            color=(1, 1, 1, 1),
            bold=True
        )
        self.btn_ajustes.bind(on_release=self.toggle_ajustes)
        self.float_layout.add_widget(self.btn_ajustes)

        # 🏠 Botón VOLVER (Arriba Izquierda)
        self.boton_volver = Button(
            text="🏠",
            size_hint=(None, None),
            size=(dp(45), dp(45)),
            pos_hint={'x': 0.02, 'top': 0.98},
            background_normal='',
            background_color=(0.2, 0.6, 1, 1),
            color=(1, 1, 1, 1),
            bold=True
        )
        self.boton_volver.bind(on_release=self.volver_principal)
        self.float_layout.add_widget(self.boton_volver)

        # 📦 MENÚ DESPLEGABLE (Abajo Derecha, encima de la barra de input)
        self.contenedor_botones = GridLayout(
            cols=1, # Una sola columna es mejor para móviles (botones más anchos)
            size_hint=(None, None),
            width=dp(160),
            spacing=dp(8),
            padding=dp(10),
            opacity=0,
            height=0
        )
        # Posición: Justo encima de la barra de entrada para que el pulgar llegue fácil
        self.contenedor_botones.pos_hint = {'right': 0.98, 'y': 0.12}

        botones = [
            ('🌙 Oscuro', self.activar_modo_oscuro),
            ('☀️ Claro', self.activar_modo_claro),
            ('🗑 Borrar', lambda *args: self.confirmar_borrado_y_borrar_historial(self.user_id, self.user_id)),
            ('💬 Soporte', self.soporte),
        ]

        for texto, callback in botones:
            btn = Button(
                text=texto,
                size_hint=(1, None),
                height=dp(45),
                background_normal='',
                background_color=(0.1, 0.2, 0.3, 0.9), # Fondo semi-transparente
                color=(1, 1, 1, 1),
                font_size=dp(14)
            )
            btn.bind(on_release=callback)
            self.contenedor_botones.add_widget(btn)
        
        self.float_layout.add_widget(self.contenedor_botones)

    def toggle_ajustes(self, instance):
        # Animación simple para móviles
        if self.contenedor_botones.opacity == 0:
            self.contenedor_botones.opacity = 1
            self.contenedor_botones.height = dp(200) # Altura fija para el menú
            self.btn_ajustes.text = "X"
        else:
            self.contenedor_botones.opacity = 0
            self.contenedor_botones.height = 0
            self.btn_ajustes.text = "⚙️"
    
    
    def preparar_prompt_con_limpieza(self, nueva_pregunta):
     # 1. SELECCIONAR ÚLTIMOS MENSAJES (Mantiene la memoria corta de 6 mensajes)
      # Esto evita que el modelo supere los 1024 tokens
      contexto_reciente = self.historial_mensajes[-6:] if len(self.historial_mensajes) > 6 else self.historial_mensajes
    
      # 2. CONSTRUIR EL TEXTO DE LA CHARLA
      charla = ""
      for msj in contexto_reciente:
        charla += f"{msj['rol']}: {msj['contenido']}\n"

       # 3. AGREGAR LA PREGUNTA ACTUAL Y EL "GATILLO" (Aria:)
      # Usamos self.instrucciones (asumiendo que las definiste en el __init__)
      prompt = self.instrucciones + charla + f"Usuario: {nueva_pregunta}\nAria: "
    
      return prompt
        
    
    def verificar_y_actualizar_db(self):
        import sqlite3
        conn = None
        try:
            conn = sqlite3.connect(self.db_path)
            cursor = conn.cursor()
            
            # 1. Comprobamos si la tabla existe y es legible de forma simple
            try:
                cursor.execute("SELECT user_id FROM mensajes LIMIT 1")
                print("✅ Estructura verificada.")
            except (sqlite3.OperationalError, sqlite3.DatabaseError):
                print("🛠️ Reparando estructura...")
                
                # Intentamos salvar lo que haya
                try:
                    cursor.execute("ALTER TABLE mensajes RENAME TO mensajes_backup")
                    conn.commit()
                except:
                    pass

                # Creamos la tabla con tipos de texto (los más estables)
                cursor.execute('''
                    CREATE TABLE IF NOT EXISTS mensajes (
                        id INTEGER PRIMARY KEY AUTOINCREMENT,
                        user_id TEXT,
                        rol TEXT,
                        emisor TEXT,
                        contenido TEXT,
                        mensaje TEXT,
                        timestamp TEXT
                    )
                ''')
                
                # Intentamos migrar si existe el backup
                try:
                    cursor.execute("INSERT INTO mensajes (user_id, emisor, mensaje, timestamp) "
                                 "SELECT user_id, emisor, mensaje, timestamp FROM mensajes_backup")
                    cursor.execute("DROP TABLE mensajes_backup")
                    print("✅ Datos migrados.")
                except:
                    cursor.execute("DROP TABLE IF EXISTS mensajes_backup")
                
                conn.commit()
        except Exception as e:
            print(f"❌ Error en verificación: {e}")
        finally:
            if conn: conn.close()
  
# --- Mostrar aviso de privacidad ---
    def agregar_aviso_privacidad_fijo(self):
        # 1. Contenedor principal (un poco más alto para que quepan los botones)
        self.layout_aviso = BoxLayout(
            orientation='horizontal',
            size_hint_y=None, 
            height='75dp',              # Aumentamos la altura para que no se corte el texto
            padding=[10, 5],
            spacing=10
        )

       # 2. Texto del aviso (Mejorado para que se vea claro)
        aviso_legal = Label(
            text=(
                "[b]Aviso de Privacidad Local:[/b]\n"
                "Acepto que mis mensajes sean procesados por la IA local de Aria. "
                "Toda la información se procesa y guarda exclusivamente en este "
                "dispositivo para tu seguridad y privacidad."
            ),
            markup=True,                # Activamos markup para usar [b] (negrita)
            font_size='11sp',
            color=(0.9, 0.9, 0.9, 1),
            size_hint_x=0.75,           # Un poco más de espacio para el texto
            halign='left',
            valign='middle'
        )
        # Ajuste dinámico del tamaño del texto
        aviso_legal.bind(size=lambda instance, value: setattr(instance, 'text_size', (value[0], None)))

       
        # 3. Botón Cancelar (Rojo suave o gris)
        btn_cancelar = Button(
            text="Cerrar",
            size_hint_x=0.2,
            font_size='10sp',
            background_color=(0.8, 0.3, 0.3, 1) # Color rojizo
        )
        btn_cancelar.bind(on_release=self.accion_cancelar)

        # 4. Botón Aceptar (Verde suave o Azul)
        btn_aceptar = Button(
            text="Aceptar",
            size_hint_x=0.2,
            font_size='10sp',
            background_color=(0.3, 0.7, 0.3, 1) # Color verdoso
        )
        btn_aceptar.bind(on_release=self.accion_aceptar)

        # Agregamos todo al layout
        self.layout_aviso.add_widget(aviso_legal)
        self.layout_aviso.add_widget(btn_cancelar)
        self.layout_aviso.add_widget(btn_aceptar)
        
        self.add_widget(self.layout_aviso)

    # --- FUNCIONES DE LOS BOTONES ---
    
    def confirmar_aceptacion(self, layout):
     self.remove_widget(layout)
     # Aquí puedes llamar a cargar_memoria() ahora que el usuario aceptó
     self.cargar_memoria()

    def accion_aceptar(self, instance):
        # Ocultamos el aviso al aceptar
        self.remove_widget(self.layout_aviso)
        print("✅ Usuario aceptó los términos.")
        # Aquí podrías guardar un valor en la base de datos para no mostrarlo más

    def accion_cancelar(self, instance):
        """
        El usuario rechaza: Quitamos el aviso pero lo dejamos en el chat.
        No se carga la memoria ni se activa el LLM.
        """
        # 1. Quitamos el cartel de la pantalla
        if hasattr(self, 'layout_aviso'):
            self.remove_widget(self.layout_aviso)
        
        # 2. Log de control
        print("🚫 Privacidad rechazada. El usuario permanece en el chat en modo lectura.")

        # 3. (Opcional) Desactivamos el input para que no pueda escribir sin aceptar
        if hasattr(self, 'entrada_texto'):
            self.entrada_texto.disabled = True
            self.entrada_texto.hint_text = "Debes aceptar los términos para chatear."
      
          
    def _crear_area_mensajes(self):
        # Primero agregar la advertencia al chat_panel
       
        
        if not hasattr(self, "scroll"):
            self.scroll = ScrollView(
                size_hint=(1, 1),
                do_scroll_x=False,
                do_scroll_y=True,
                bar_width=6
            )
        
        if not hasattr(self, "chat_container"):
            self.chat_container = GridLayout(
                cols=1,
                spacing=10,
                size_hint_y=None,
                padding=[10, 10]
            )
            self.chat_container.bind(minimum_height=self.chat_container.setter('height'))
        
        if not self.scroll.children:
            self.scroll.add_widget(self.chat_container)
        
        if hasattr(self, "chat_panel") and self.scroll not in self.chat_panel.children:
            self.chat_panel.add_widget(self.scroll)

    def _agregar_advertencias(self):
        advertencia = "⚠️ La información de Aria es orientativa. Consulta con profesionales si es necesario."
        
        # 1. Contenedor independiente
        advertencia_box = BoxLayout(
            orientation='horizontal',
            size_hint_y=None,
            height=dp(30),
            padding=[dp(10), 0]
        )
        
        # 2. Etiqueta
        lbl = Label(
            text=advertencia,
            halign="center",
            valign="middle",
            color=(0.8, 0.8, 0.4, 1),
            font_size=dp(11)
        )
        
        # Ajuste dinámico del texto
        lbl.bind(size=lambda inst, value: setattr(inst, 'text_size', (value[0], None)))
        
        advertencia_box.add_widget(lbl)
        
        # ✅ SOLO ESTA LÍNEA AL FINAL. Nada de Clock.
        self.add_widget(advertencia_box)

    def _crear_input_bar(self):
        input_bar = BoxLayout(
            orientation='horizontal',
            size_hint_y=None,
            height=dp(60), # Altura cómoda para el dedo
            padding=[dp(10), dp(5)],
            spacing=dp(10)
        )
        # Aquí van tu TextInput y tu Botón Enviar...
        # Asegúrate de que el TextInput tenga size_hint_x=0.8 y el botón 0.2
        

        self.input = TextInput(
        multiline=False,
        hint_text="Pregúntale algo a Aria",
        font_size=dp(18),
        background_color=self.input_bg,
        foreground_color=self.text_color_chat,
        cursor_color=self.text_color_chat,
        padding=[dp(12), dp(10)],
        size_hint_x=1
    )

        self.input.bind(on_text_validate=lambda instance: self.enviar_mensaje())

        self.enviar = Button(
        text="📤 Enviar",
        size_hint_x=None,
        width=dp(90),
        font_size=dp(16),
        bold=True,
        background_color=(0, 0.6, 0.9, 1),
        color=(1, 1, 1, 1)
    )
        self.btn_borrar = Button(
     text="✖",
     size_hint_x=None,
     width=dp(50),
     font_size=dp(18),
     background_color=(0.6, 0.2, 0.2, 1),
     color=(1, 1, 1, 1)
)  
        self.enviar.bind(on_press=lambda x: setattr(x, 'opacity', 0.7))
        self.enviar.bind(on_release=lambda x: setattr(x, 'opacity', 1))
        self.enviar.bind(on_press=lambda instance: self.enviar_mensaje())
        self.input.bind(text=self.mostrar_boton_borrar)  
        self.btn_borrar.bind(on_press=self.borrar_input)
        self.btn_borrar.bind(
    on_press=lambda x: setattr(self.input, 'text', '')
)
        input_bar.add_widget(self.input)
        input_bar.add_widget(self.enviar)
        input_bar.add_widget(self.btn_borrar)

        self.btn_borrar.opacity = 0
        self.add_widget(input_bar)

    
    
    def borrar_input(self, instance):
        self.input.text = ""

    def mostrar_boton_borrar(self, instance, value):
     self.btn_borrar.opacity = 1 if value else 0   

    
    
    # Esta lista es oro. Expándela con errores que tú mismo detectes.
    errores_comunes = {
    r'\bdigen\b': 'dicen',
    r'\bpueds\b': 'puedes',
    r'\bami\b': 'a mí',
    r'\bq\b': 'que',
    r'\bxq\b': 'porque',
    r'\btmb\b': 'también',
    r'\bholaa+\b': 'hola', # Corrige "holaaaaa"
}

    def corregir_texto(texto):
     texto_corregido = texto.lower()
     for err, corr in errores_comunes.items():
        texto_corregido = re.sub(err, corr, texto_corregido)
     return texto_corregido


    llama_model = None  # será asignado desde MainAria
     
    
    def copiar_mensaje(self, texto):
     try:
        from kivy.core.clipboard import Clipboard
        Clipboard.copy(str(texto))
        print("✅ Mensaje copiado")
     except Exception as e:
        print("⚠️ Error copiando:", e)

    def obtener_info_dislexia(self):
        try:
            # Asegúrate de que el nombre del archivo .db sea el correcto
            conn = sqlite3.connect("base_conocimiento.db")
            cursor = conn.cursor()

           
            cursor.execute("SELECT mensaje FROM conocimiento_dislexia")
            filas = cursor.fetchall()

            conn.close()

            return "\n".join([f[0] for f in filas])

        except Exception as e:
            # Esto te dirá exactamente qué columna falta
            print("❌ Error leyendo dislexia:", e)
            return ""
 
   
    
    def guardar_usuario(self, texto):
                self.historial.append({
                "rol": "usuario",
                "texto": texto
                   })

    def guardar_aria(self, texto):
                self.historial.append({
                "rol": "aria",
                "texto": texto
                     })

   

     
    def on_enter(self):
     self.cargar_historial()
     Clock.schedule_interval(lambda dt: self.cargar_mensajes_desde_firebase(), 5) 
    
  
    
   
    

    
    def gestionar_accion_mensaje(self, plataforma, texto, boton):
        import webbrowser
        from urllib.parse import quote

        # A) SIEMPRE COPIAR (El "Plan B" de seguridad para el usuario)
        Clipboard.copy(texto)
        
        # Feedback visual rápido en el botón
        icono_original = boton.text
        boton.text = "✅"
        Clock.schedule_once(lambda dt: setattr(boton, 'text', icono_original), 1.0)

        if plataforma != 'copy':
            txt_codificado = quote(f"Aria me ayudó con esto: {texto}")
            
            urls = {
            # --- Las más usadas ---
            'wa': f"https://wa.me/?text={txt_codificado}",
            
            # Facebook (Usa la cita 'quote' para el texto)
            'fb': f"https://www.facebook.com/sharer/sharer.php?quote={txt_codificado}",
            
            # X (Ex Twitter): Usamos x.com para estar actualizados
            'x': f"https://x.com/intent/post?text={txt_codificado}",
            
            # --- Mensajería ---
            'tg': f"https://t.me/share/url?text={txt_codificado}", 
            
            # --- NUEVAS: Reddit e Instagram ---
            # Reddit: Permite título y cuerpo del post
            'rd': f"https://www.reddit.com/submit?title=Consulta+Aria&text={txt_codificado}",
            
            # Instagram: Solo abre la App (Instagram no permite texto directo por URL)
            'ig': "https://www.instagram.com/", 

            # --- Otras ---
            'ln': f"https://www.linkedin.com/sharing/share-offsite/?text={txt_codificado}",
            'em': f"mailto:?subject=Consulta de Aria&body={txt_codificado}" 
        }
            try:
                if plataforma in urls:
                    webbrowser.open(urls[plataforma])
            except Exception as e:
                print(f"❌ Error al abrir {plataforma}: {e}")
    # ================================================================
    # FUNCIONES DE CAMBIO DE MODO
    # ================================================================
    def obtener_conocimiento_aria(self):
        try:
            # Asegúrate de usar tu objeto de conexión (ej: self.cursor)
            self.cursor.execute("SELECT contenido FROM conocimiento_dislexia")
            filas = self.cursor.fetchall()
            # Esto crea un solo texto con todas las frases
            return "\n".join([f"- {fila[0]}" for fila in filas])
        except Exception as e:
            print(f"Error al leer base de datos: {e}")
            return "Información básica sobre dislexia."

         
    def enviar_mensaje(self, instance=None,):
        # 1. Obtener y validar texto
        texto = self.input.text.strip()
        if not texto:
            return
        
        # --- AQUÍ PEGAS EL CÓDIGO ---
        if len(self.memoria_chat) > 40:
                self.memoria_chat.pop(0)
            # ----------------------------
        
        # --- SOLO EL HILO VACÍO ---
        hilo = threading.Thread(target=None, args=())
        hilo.daemon = True
        hilo.start()
        # --------------------------

        # 2. Limpiar input y mostrar en pantalla
        self.input.text = ""
        self._agregar_mensaje(texto, emisor="usuario")

        # 3. Manejo de Historial (Memoria)
        if hasattr(self.memoria, 'historial'):
            self.memoria.historial.append({"emisor": "Usuario", "texto": texto})
        else:
            self.historial.append({"emisor": "Usuario", "texto": texto})

        # 4. Guardar en SQLite local
        timestamp = time.strftime("%Y-%m-%d %H:%M:%S")
        db_path = os.path.join(App.get_running_app().user_data_dir, "chat_local.db")
        
        try:
            conn = sqlite3.connect(db_path)
            c = conn.cursor()
            c.execute("""CREATE TABLE IF NOT EXISTS mensajes (
                            usuario TEXT,
                            mensaje TEXT,
                            timestamp TEXT
                        )""")
            c.execute(
                "INSERT INTO mensajes (usuario, mensaje, timestamp) VALUES (?, ?, ?)",
                ("usuario", texto, timestamp)
            )
            conn.commit()
            conn.close()
        except Exception as e:
            print("❌ Error guardando mensaje local:", e)

        # 5. Enviar a Firebase (En segundo plano)
        mensaje_firebase = {
            "emisor": "usuario", 
            "mensaje": texto, 
            "timestamp": timestamp
        }
        Thread(target=self._enviar_mensaje_firebase, args=(mensaje_firebase,), daemon=True).start()

        # 6. Llamar a Aria (En segundo plano)
        Thread(target=lambda: self.generar_respuesta_aria(texto, timestamp), daemon=True).start()
    
 
    
    def mostrar_respuesta_final(self, texto):
    
     self.agregar_mensaje_ui("aria", texto)
   
    def iniciar_respuesta(self, texto, timestamp, mensaje_nuevo):

     # 🚫 evitar múltiples respuestas al mismo tiempo
     if hasattr(self, "thread_activo") and self.thread_activo:
        print("⚠️ Ya hay una respuesta en proceso")
        return

     self.thread_activo = True

     def tarea():
        try:
            self.generar_respuesta_aria(texto, timestamp,)
        except Exception as e:
            print("❌ Error en respuesta:", e)
        finally:
            self.thread_activo = False

     Thread(target=tarea, daemon=True).start()

    
    def es_repetida(respuesta, historial, umbral=0.8):
        if not respuesta:
            return False
        for mensaje in historial:
            if not mensaje:
                continue
            similitud = SequenceMatcher(None, respuesta, mensaje).ratio()
            if similitud >= umbral:
                return True
        return False


    def obtener_ultima_respuesta_de_aria(self):
     historial = getattr(self.memoria, "historial", []) or []
     cantidad_mensajes = len(historial)

     for item in reversed(historial):
        if isinstance(item, dict):
            if (item.get("emisor") or "").lower() == "aria":
                return item.get("mensaje", ""), item.get("timestamp", 0), cantidad_mensajes
        elif isinstance(item, str):
            if item.lower().startswith("aria:"):
                return item.split(":", 1)[1].strip(), 0, cantidad_mensajes

     return "", 0, cantidad_mensajes
    
    def es_duplicado(
     self,
     nueva_respuesta,
     ultima_respuesta=None,
     timestamp_anterior=None,
     cantidad_mensajes=None,
     tiempo_minimo=60,
     mensajes_minimos=3,
     umbral_similitud=0.8
):
     try:
        # 🔧 Convertimos None a string vacío
        if nueva_respuesta is None:
            nueva_respuesta = ""
        if ultima_respuesta is None:
            ultima_respuesta = ""

        # Evito warnings de parámetros no usados
        _ = timestamp_anterior
        _ = cantidad_mensajes
        _ = tiempo_minimo
        _ = mensajes_minimos

        # 🔎 Ignorar respuestas muy cortas
        if len(nueva_respuesta.split()) < 3:
            return False

        # 🔁 Comparación por similitud
        similitud = difflib.SequenceMatcher(
            None,
            nueva_respuesta.lower(),
            ultima_respuesta.lower()
        ).ratio()

        return similitud > umbral_similitud

     except Exception as e:
        print(f"⚠️ Error en es_duplicado: {str(e)}")
        return False
     
    def detectar_nombre(self, user_id, mensaje):

        import re

        patron = r"(me llamo|soy|mi nombre es)\s+(\w+)"

        match = re.search(patron, mensaje.lower())

        if match:

            nombre = match.group(2).capitalize()

            if user_id not in self.memoria_usuario:
                self.memoria_usuario[user_id] = {}

            self.memoria_usuario[user_id]["nombre"] = nombre

            print(f"Nombre guardado: {nombre}")

    def detectar_dislexia(self, user_id, mensaje):

     mensaje = mensaje.lower()

     if "tengo dislexia" in mensaje or "soy dislexico" in mensaje:

        if user_id not in self.memoria_usuario:
            self.memoria_usuario[user_id] = {}

        self.memoria_usuario[user_id]["dislexia"] = True

        print("Usuario marcado con dislexia")

    

    def corregir_texto_dislexia(self, texto):
     diccionario = [

      # dislexia
      "dislexia","dislexico","disléxico","dificultad","problema",
      "lectura","escritura","comprensión","procesamiento","lenguaje",

        # lectura
        "leer","lectura","texto","frase","párrafo","documento",
        "artículo","correo","mensaje","informe","contrato",
        "libro","página","contenido","información",

        # escritura
        "escribir","escritura","mensaje","correo","nota",
        "documento","palabra","letra","ortografía","redacción",

        # trabajo
        "trabajo","empleo","oficina","empresa","cliente",
        "proyecto","tarea","responsabilidad","reunión",
        "informe","documentación","presentación",

        # aprendizaje adulto
        "aprender","entender","comprender","explicar",
        "estudiar","formación","capacitación","curso",
        "habilidad","práctica","ejercicio","mejorar",

        # cerebro
        "cerebro","mente","memoria","concentración",
        "pensamiento","atención","procesar","recordar",

        # ayuda
        "ayuda","apoyo","estrategia","herramienta",
        "método","solución","practicar","mejorar",
        "progreso","superar",

        # emociones
        "frustración","confusión","ansiedad",
        "confianza","seguridad","motivación",
        "paciencia","esfuerzo","logro"
] 
     palabras = texto.split()
     resultado = []

     for palabra in palabras:

        coincidencias = difflib.get_close_matches(
            palabra,
            diccionario,
            n=1,
            cutoff=0.75
        )

        if coincidencias:
            resultado.append(coincidencias[0])
        else:
            resultado.append(palabra)

     return " ".join(resultado)             


    # 🔥 CREALA ACÁ
    def construir_contexto(self, resultados_rag):
        bloques_limpios = []

        for resultado in resultados_rag:
            if isinstance(resultado, dict):
                texto = resultado.get("texto", "").strip()
            else:
                texto = str(resultado).strip()

            if not texto:
                continue

            # Limpieza básica
            texto = texto.replace("Aria:", "")
            texto = texto.replace("Aria:", "")
            texto = texto.replace("Score:", "")
            texto = texto.replace("ID:", "")

            bloques_limpios.append(texto)

        return "\n".join(bloques_limpios)
     
  
    def borrar_mensajes(self):
        import sqlite3
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        cursor.execute("DELETE FROM mensajes WHERE usuario_id = ?", (self.usuario_id,))
        conn.commit()
        conn.close()
        print("Mensajes borrados correctamente") 

    
    def detectar_dislexia(self, texto):

        palabras = texto.lower().split()
        palabra_objetivo = "dislexia"

        for p in palabras:
            similitud = difflib.SequenceMatcher(None, p, palabra_objetivo).ratio()

            if similitud > 0.65:
                return True

        return False
    
    # 👇 PEGALO ACÁ
    def normalizar_msg(msg):
     try:
        if isinstance(msg, dict):
            return {
                "emisor": str(msg.get("emisor", "usuario")),
                "mensaje": str(msg.get("mensaje", ""))
            }
        elif isinstance(msg, str):
            return {
                "emisor": "usuario",
                "mensaje": msg
            }
        else:
            return {
                "emisor": "usuario",
                "mensaje": str(msg)
            }
     except Exception as e:
        print("⚠️ Error normalizando mensaje:", e)
        return {
            "emisor": "usuario",
            "mensaje": ""
        }
    def generar_respuesta_aria(self, texto_usuario, timestamp=None):
        def _generar():
            try:
                print("🧠 Generando respuesta aria...")
                # 1. INICIALIZACIÓN DE VARIABLES
                user_id = getattr(self, "user_id", "default")
                historial_texto = ""
                info_dislexia = ""
              
                contexto_limpio = self.historial_mensajes[-3:] if len(self.historial_mensajes) > 0 else []
                # 1. Tomamos los últimos 3
                ultimos_mensajes = self.memoria_chat[-3:]
                historial_limpio = ""

                # 2. El bucle corregido y blindado
                for msg in ultimos_mensajes:
                    # Todo este bloque tiene que estar alineado a la derecha del 'for'
                    if msg['emisor'] == self.usuario_id:
                        rol = "Usuario"
                    else:
                        # Si no es el usuario, asumimos que es Aria
                        rol = "Aria"
                    
                    # ESTA LÍNEA ES CLAVE: Debe estar bajo el 'if/else', todavía dentro del 'for'
                    historial_limpio += f"{rol}: {msg['texto']}\n"
                
                # 3. Fuera del bucle (al mismo nivel que el 'for'), armamos el prompt
            
             
                # 2. PREPARAR TEXTO E INFO DISLEXIA
                texto_corregido = self.corregir_texto_dislexia(texto_usuario)
                if self.detectar_dislexia(texto_usuario.lower()):
                    try: 
                        info_dislexia = self.obtener_info_dislexia()[:200]
                    except: 
                        info_dislexia = ""

              

                # 4. CONFIGURAR EL PROMPT ÚNICO# 3. CONSTRUIR HISTORIAL RECIENTE (Últimos 6 mensajes)
                historial_previo = self.historial[-6:] if hasattr(self, "historial") and self.historial else []
        
                   # Usamos una lista para verificar qué estamos metiendo
                lineas_añadidas = set() 
 
                for msg in historial_previo:
                 if isinstance(msg, dict):
                  emisor = "Usuario" if msg.get('emisor') == "usuario" else "Aria"
                  texto_msg = msg.get('mensaje', '').strip()
                
                 # CREAMOS LA LÍNEA
                 nueva_linea = f"{emisor}: {texto_msg}\n"
                
                 # FILTRO ANTIDUPLICADOS: Solo añade si la frase no es idéntica a una anterior
                 if texto_msg not in lineas_añadidas:
                    historial_texto += nueva_linea
                    lineas_añadidas.add(texto_msg)
            
              # Esto le dice a la IA: "Tu historial termina aquí, ahora te toca a ti"
                prompt = f"{""}\n\n{historial_texto}Aria:"
                prompt = f"""### INSTRUCCIONES
                Eres Aria, una asistente que ayuda con la dislexia.
                Tu estilo es: frases cortas, simples y directas.
                Responde en español neutro, máximo 2 frases.
                Si el usuario dice "adiós", despídete y no preguntes nada más.
                ### CONTEXTO
                {info_dislexia}

                ### CONVERSACIÓN ANTERIOR
                {historial_texto}

                ### RESPUESTA
                Usuario: {texto_corregido}
                Aria:"""

               
                # 5. UNA SOLA LLAMADA AL MODELO (Sacamos las otras llamadas que sobraban)
                salida = self.llm.create_completion(
                prompt=prompt,
                max_tokens=50,
                temperature=0.3,   # Subimos un pelín para que no sea un robot que repite el prompt
                top_p=0.8,         # Le damos más variedad de palabras
                repeat_penalty=1.5, 
                stop=["\n", "Usuario:", "Aria:"],
                 echo=False
)
                
                respuesta = salida["choices"][0].get("text", "").strip()
                
                prompt= (
                 f"{prompt}\n\n"        # 1. Las reglas (Máximo 2 frases)
                 f"{historial_texto}\n"        # 2. Los últimos 3 mensajes
                 f"Usuario: {texto_usuario}\n" # 3. Lo que te acaban de preguntar
                 f"Aria:"                      # 4. El empujón para que ella responda
)

                # Limpieza de seguridad
                if not respuesta: 
                    respuesta = "Estoy acá para ayudarte."

                print(f"🤖 IA terminó de pensar: {respuesta}")

                # 6. ACTUALIZAR INTERFAZ (Kivy)
                # Usamos el nombre unificado: agregar_mensaje_aria
               
                # Correcto: ahora llama a '_agregar_mensaje' que es como tú la tienes
                Clock.schedule_once(lambda dt: self._agregar_mensaje(respuesta, emisor="Aria"), 0)

                # 7. GUARDAR MEMORIA Y FIREBASE
                if hasattr(self, "_guardar_todo"):
                    self._guardar_todo(user_id, texto_usuario, respuesta)
                
                if hasattr(self, "historial"):
                    self.historial.append({"emisor": "Aria", "mensaje": respuesta})

            except Exception as e:
                print(f"⚠️ Error en generación: {e}")
               
                Clock.schedule_once(lambda dt: self._agregar_mensaje("Estoy procesando mucha info, dime otra vez.", emisor="Aria"), 0)

        # Lanzar el hilo
        from threading import Thread
        Thread(target=_generar, daemon=True).start()
    
    def guardar_en_db_local(self, user_id, emisor, mensaje):
        """Guarda el mensaje en el archivo SQLite local."""
        try:
            import sqlite3
            conn = sqlite3.connect('dyslexialift.db') 
            cursor = conn.cursor()
            # Creamos la tabla si no existe para evitar errores
            cursor.execute('''CREATE TABLE IF NOT EXISTS mensajes 
                            (user_id TEXT, emisor TEXT, texto TEXT, fecha TEXT)''')
            cursor.execute('''INSERT INTO mensajes (user_id, emisor, texto, fecha) 
                            VALUES (?, ?, ?, datetime('now'))''', (user_id, emisor, mensaje))
            conn.commit()
            conn.close()
            print(f"📦 Guardado localmente: {emisor}")
        except Exception as e:
            print(f"⚠️ Error local: {e}")

    def _guardar_todo(self, user_id, texto_usuario, respuesta):
        try:
            # --- 1. GUARDADO LOCAL (Para que no se pierda nada sin internet) ---
            # Aquí llamamos a la base de datos interna del celular/PC
            self.guardar_en_db_local(user_id, "usuario", texto_usuario)
            self.guardar_en_db_local(user_id, "aria", respuesta)
            
            # --- 2. ENVIAR A FIREBASE (Nube) ---
            # Estas son tus funciones que suben la info a la red
            guardar_mensaje_fisico(user_id, "usuario", texto_usuario)
            guardar_mensaje_fisico(user_id, "aria", respuesta)
            
            # --- 3. VISTA EN CONSOLA VS CODE (Para tu estudio) ---
            # Para que veas las respuestas de Aria limpias y ordenadas
            print("\n" + "💾 LOCAL + 🌐 FIREBASE " + "─"*20)
            print(f"🆔 USUARIO: {user_id}")
            print(f"👤 PREGUNTA: {texto_usuario}")
            print(f"🤖 ARIA DIJO: {respuesta}")
            print("─"*41 + "\n")
            
            # --- 4. DISPARAR SINCRONIZACIÓN (Seguridad extra) ---
            if hasattr(self, 'sync_manager') and self.sync_manager:
                self.sync_manager.start_sync()
                print(f"✅ Sincronización en segundo plano activada.")
            
            print(f"✔️ ¡Éxito! Mensajes asegurados en Local y Nube.")

        except Exception as e:
            print(f"❌ Error en el sistema de guardado dual: {e}")
    
    def procesar_mensaje(self, texto_usuario):
        """Maneja el flujo completo: Usuario -> Memoria -> Aria -> Pantalla"""
        user_id = getattr(self, "user_id", "usuario_anonimo")
        
        # 1. Limpieza básica
        texto_usuario = texto_usuario.strip()
        if not texto_usuario:
            return

        alerta = control_seguridad(texto_usuario) 
        if alerta:
            self.agregar_mensaje_ui("Aria", alerta)
            return

        # 3. Corrección para Dislexia (Tu función de limpieza de texto)
        texto_corregido = self.corregir_texto_dislexia(texto_usuario)

        # 4. UI: Mostrar lo que escribió el usuario (Burbuja derecha)
        self.agregar_mensaje_ui("Tú", texto_corregido)

        # 5. Guardar en Memoria local (Usuario)
        self.memoria.guardar_mensaje(user_id, "Usuario", texto_corregido)

        # 6. Tarea en segundo plano para la IA (No bloquea la pantalla)
        def tarea_ia():
            try:
                # --- PREPARACIÓN DEL PROMPT ---
                # Obtener historial reciente de la base de datos
                historial = self.memoria.obtener_para_prompt(user_id, limite=10)
                
                # Crear el prompt con formato para Aria
                prompt_base = self.crear_prompt_aria(historial, texto_corregido)
                
                # SAFE TRIM: Ajustar el tamaño para que el PC no se trabe
                prompt_final = self.safe_trim(prompt_base, max_tokens=850)

                # --- GENERACIÓN ---
                # Llamar al modelo Llama real
                respuesta_raw = self.modelo_llama.generar_respuesta(prompt_final)

                # Validar que la respuesta sea coherente
                if self.validar_respuesta(respuesta_raw):
                    respuesta = respuesta_raw
                else:
                    respuesta = "No estoy segura de cómo responder a eso. ¿Me lo dices de otra forma?"

                # --- SALIDA A UI Y GUARDADO ---
                # Mostrar respuesta de Aria en la pantalla (usando Clock para seguridad)
                Clock.schedule_once(lambda dt: self.agregar_mensaje_ui("Aria", respuesta), 0)

                # Guardar respuesta de Aria en memoria y Firebase
                self.memoria.guardar_mensaje(user_id, "Aria", respuesta)

            except Exception as e:
                print(f"❌ Error en Aria: {e}")
                Clock.schedule_once(lambda dt: self.agregar_mensaje_ui("Aria", "Lo siento, tuve un pequeño error técnico."), 0)

        # Iniciar el hilo de procesamiento
        from threading import Thread
        Thread(target=tarea_ia, daemon=True).start()

            # --- FUNCIÓN DE SEGURIDAD (Fuera de la clase o como método estático) ---
        def control_seguridad(texto):
         palabras_peligrosas = [
        "hack", "password", "contraseña", "borrar cuenta", 
        "acceder a otros usuarios", "rebelión", "destruir", 
        "inyectar código", "script", "database", "admin"
    ]
    
         texto_minuscula = texto.lower()
         for palabra in palabras_peligrosas:
          if palabra in texto_minuscula:
            return "❌ Por seguridad, no puedo procesar esta solicitud."
            
        return None
   


    def contar_tokens(self, texto):
      return len(texto.split())  # Conteo simple y seguro para Phi-2        

    

    def safe_trim(self, prompt, max_tokens=900, safe_margin=50):
     total_tokens = self.contar_tokens(prompt)
     if total_tokens <= max_tokens:
        return prompt

     lineas = prompt.split("\n")
     prompt_recortado = ""
     tokens_actuales = 0

     for linea in reversed(lineas):
        texto_linea = linea + "\n"
        tokens_linea = self.contar_tokens(texto_linea)

        if tokens_actuales + tokens_linea > max_tokens - safe_margin:
            break

        prompt_recortado = texto_linea + prompt_recortado
        tokens_actuales += tokens_linea

     return prompt_recortado
    
   
    
    def _agregar_mensaje(self, texto, emisor="usuario"):
        es_aria = emisor == "Aria"
        color_fondo = (0.15, 0.35, 0.7, 1) if es_aria else (0.2, 0.6, 0.3, 1)
        
        # 1. CONTENEDOR DE LA FILA (Vertical)
        contenedor_fila = BoxLayout(
            orientation="vertical", 
            size_hint_y=None, 
            padding=[dp(10), dp(5)],
            spacing=dp(10)
        )

        # 2. FILA DE TEXTO (Horizontal)
        fila_horizontal = BoxLayout(
            orientation="horizontal",
            size_hint_y=None
        )

        # 3. EL LABEL (La burbuja)
        lbl = Label(
            text=texto,
            size_hint=(None, None), 
            halign="left",
            valign="middle",
            font_size=dp(18),
            line_height=1.2,
            padding=[dp(15), dp(10)],
            color=(1, 1, 1, 1)
        )
        
        def ajustar_burbuja(instance, size):
            ancho_max = Window.width * 0.65
            if instance.texture_size[0] < ancho_max:
                instance.width = instance.texture_size[0] + dp(30)
                instance.text_size = (None, None)
            else:
                instance.width = ancho_max
                instance.text_size = (ancho_max, None)

            instance.height = instance.texture_size[1] + dp(20)
            fila_horizontal.height = instance.height
            
            extra = dp(85) if es_aria else dp(15) # Aumenté un poco el extra para Aria
            contenedor_fila.height = instance.height + extra
            
            # ESTO ES LO QUE ARREGLA EL ENCIMADO
            if self.chat_container:
                self.chat_container.height = sum([c.height for c in self.chat_container.children])

        lbl.bind(texture_size=ajustar_burbuja)

        with lbl.canvas.before:
            Color(*color_fondo)
            lbl.rect = RoundedRectangle(radius=[dp(14)])
        
        lbl.bind(pos=lambda inst, pos: setattr(inst.rect, 'pos', pos),
                 size=lambda inst, size: setattr(inst.rect, 'size', size))

        # Alineación
        if es_aria:
            fila_horizontal.add_widget(lbl)
            fila_horizontal.add_widget(Widget()) # Empuja a la izquierda
        else:
            fila_horizontal.add_widget(Widget()) # Empuja a la derecha
            fila_horizontal.add_widget(lbl)

        contenedor_fila.add_widget(fila_horizontal)

        # 4. FUNCIÓN INTERNA PARA BOTONES
        def crear_boton_circular(icono, color_f, plataforma, texto_c):
            btn = Button(
                text=icono,
                size_hint=(None, None),
                size=(dp(36), dp(36)), # Reducido de 42 a 36
                background_normal='',
                background_color=(0, 0, 0, 0),
                font_size=dp(11), # Letra un poco más chica para que quepa bien
                bold=True
            )
            with btn.canvas.before:
                Color(*color_f)
                btn.circ = Ellipse(pos=btn.pos, size=btn.size)
            btn.bind(
                pos=lambda inst, pos: setattr(inst.circ, 'pos', pos),
                size=lambda inst, size: setattr(inst.circ, 'size', size)
            )
            btn.bind(on_release=lambda x: self.gestionar_accion_mensaje(plataforma, texto_c, btn))
            return btn
        # 5. BARRA DE HERRAMIENTAS (Solo Aria) - SIN SCROLL INTERNO
        if es_aria:
            barra = BoxLayout(
                orientation="horizontal", 
                size_hint=(1, None),        # Ancho total del mensaje, alto fijo
                height=dp(55), 
                spacing=dp(10), 
                padding=[dp(5), dp(5)]
            )
            
            iconos = [
                ("CP", (0.2, 0.2, 0.2, 1), 'copy'), ("WA", (0.13, 0.82, 0.42, 1), 'wa'),
                ("X", (0.05, 0.05, 0.05, 1), 'x'), ("FB", (0.23, 0.35, 0.6, 1), 'fb'),
                ("TG", (0, 0.53, 0.8, 1), 'tg'), ("IG", (0.75, 0.15, 0.5, 1), 'ig'),
                ("RD", (1, 0.27, 0, 1), 'rd'), ("@", (0.3, 0.3, 0.3, 1), 'em')
            ]
            
            for ico, col, plat in iconos:
                # Ajustamos los botones para que quepan sin scroll
                btn = crear_boton_circular(ico, col, plat, texto)
                btn.size_hint = (None, None)
                btn.size = (dp(40), dp(40)) # Un poco más pequeños para que entren todos
                barra.add_widget(btn)

            # Agregamos la barra directamente a la fila del mensaje
            contenedor_fila.add_widget(barra)

         # --- ESTO ES LO QUE HACE QUE APAREZCA EN PANTALLA ---
        # 6. Agregamos el mensaje completo al contenedor principal
        self.chat_container.add_widget(contenedor_fila)

        # 7. Forzamos la actualización de la altura del contenedor
        self.chat_container.height = sum([c.height for c in self.chat_container.children])   

        # 2. BAJAR EL SCROLL (Usando el 0 que es el fondo)
        Clock.schedule_once(lambda dt: setattr(self.scroll, 'scroll_y', 0), 0.1)

    def crear_boton_circular(self, icono, color_fondo, plataforma, texto_compartir):
      
        btn = Button(
        text=icono,
        size_hint=(None, None),
        size=(dp(42), dp(42)),
        background_normal='',
        background_color=(0,0,0,0),
        font_size=dp(20), # <--- Subí esto a 20 o 22 para que el icono sea grande
        bold=True
    )
       
        with btn.canvas.before:
            Color(*color_fondo)
            btn.circ = Ellipse(pos=btn.pos, size=btn.size)
        
        # Vincular movimiento del círculo con el botón
        btn.bind(
            pos=lambda inst, pos: setattr(inst.circ, 'pos', pos),
            size=lambda inst, size: setattr(inst.circ, 'size', size)
        )
        
        # Acción al soltar el botón
        btn.bind(on_release=lambda x: self.gestionar_accion_mensaje(plataforma, texto_compartir, btn))
        return btn
  

    def obtener_historial(self, usuario_id):
        conn = sqlite3.connect(self.db_path)
        c = conn.cursor()
        
        # ❌ ERROR: SELECT emisor, contenido FROM mensajes
        # ✅ CORRECTO:
        c.execute("SELECT emisor, mensaje FROM mensajes WHERE usuario_id=?", (usuario_id,))
        
        filas = c.fetchall()
        conn.close()
        
        # Convertimos a lista de diccionarios para que sea fácil de leer
        return [{"emisor": f[0], "mensaje": f[1]} for f in filas]    
     


     # 🔹 Validar si la respuesta es válida
    def validar_respuesta(respuesta):
      if "<<" in respuesta or len(respuesta.split()) < 3:
        return False
      return True
    


     # ✅ Cargar historial desde base local o Firebase
    
    def cargar_historial(self):
        # 1. Obtenemos los mensajes usando el objeto memoria que definimos en el __init__
        try:
            # Si tu clase Memoria tiene el método obtener_historial:
            mensajes = self.memoria.obtener_historial(self.user_id)
            
            for msg in mensajes:
                # Extraemos los datos (emisor y mensaje son lo básico)
                emisor = msg.get('emisor', 'aria')
                texto = msg.get('mensaje', '')
                
                # Agregamos a la interfaz
                self.agregar_mensaje_ui(emisor, texto)

            # 2. Auto-scroll al final para ver el último mensaje
            if self.chat_container.children:
                Clock.schedule_once(lambda dt: setattr(self.scroll, 'scroll_y', 0), 0.2)
                
        except Exception as e:
            print(f"⚠️ Error al cargar historial: {e}")

    def cargar_mensajes_desde_firebase(self):
     try:
        url = f"{FIREBASE_URL}usuarios/{self.user_id}/chats/mensajes.json"
        response = requests.get(url, timeout=5)
        if response.status_code != 200:
            print(f"⚠️ Error al obtener mensajes: {response.status_code}")
            return

        datos = response.json() or {}
        mensajes = sorted(
            datos.values(),
            key=lambda m: m.get("timestamp", "")
        )

        # 🔹 Limpiar chat antes de recargar
        self.ids.contenedor_chat.clear_widgets()

        # 🔹 Mostrar mensajes ordenados
        for msg in mensajes:
            emisor = msg.get("emisor", "")
            mensaje = msg.get("mensaje", "")
            if emisor and mensaje:
                self.agregar_mensaje_ui(emisor, mensaje)

        print(f"✅ {len(mensajes)} mensajes cargados desde Firebase")

     except Exception as e:
        print(f"❌ Error al cargar mensajes desde Firebase: {e}")

    
    def confirmar_borrado_y_borrar_historial(self, user_id, current_user_id):
        # 1. Verificación de seguridad
        if user_id != current_user_id:
            try:
                Clock.schedule_once(lambda dt: self.agregar_mensaje_ui(
                    "Sistema", "❌ No tienes permiso para borrar este historial"), 0)
            except ReferenceError: pass
            return

        def borrar():
            try:
                # --- A. Limpiar chat visual ---
                if hasattr(self, "chat_container") and self.chat_container:
                    try:
                        Clock.schedule_once(lambda dt: self.chat_container.clear_widgets(), 0)
                    except ReferenceError: pass

                # --- B. Borrar SQLite (Tabla correcta: chat_historial) ---
                try:
                 conn = sqlite3.connect(self.db_path, check_same_thread=False)
                 cursor = conn.cursor()
    
                   # IMPORTANTE: Usamos 'chat_historial' porque es la que definiste arriba
                 cursor.execute("DELETE FROM chat_historial WHERE user_id = ?", (str(self.user_id),))
    
                 conn.commit()
                 conn.close()
                 print("🗑️ SQLite: Tabla 'chat_historial' limpiada correctamente.")
                except Exception as e:
                 print(f"⚠️ SQLite Error al borrar: {e}")

                # --- C. Borrar Firebase ---
                try:
                    clean_url = FIREBASE_URL.rstrip('/')
                    requests.delete(f"{clean_url}/usuarios/{self.user_id}/chats.json", timeout=5)
                except: pass

                # --- D. Limpiar memoria RAM ---
                if hasattr(self, "memoria"):
                    self.memoria.limpiar_chat(self.user_id)
                self.historial = [] 
                self.memoria_chat = []
                # --- E. Mensaje de éxito ---
                try:
                    Clock.schedule_once(lambda dt: self.agregar_mensaje_ui(
                        "Sistema", "✅ Todo el historial ha sido borrado"), 0)
                except ReferenceError: pass

            except Exception as e:
                print(f"❌ Error en borrado: {e}")

        # --- Popup de confirmación (Diseño Móvil) ---
        box = BoxLayout(orientation='vertical', padding=dp(15), spacing=dp(10))

        box.add_widget(Label(
            text="¿Seguro que querés borrar todo tu historial?\nEsta acción no se puede deshacer.",
            halign="center", valign="middle", size_hint_y=None, height=dp(60), font_size=dp(14)
        ))

        botones_box = BoxLayout(orientation='vertical', spacing=dp(8), size_hint_y=None)
        botones_box.bind(minimum_height=botones_box.setter('height'))

        # Lista de botones con sus colores y acciones
        # Usamos Threads solo para el de borrar porque es el único que "trabaja"
        botones_config = [
            ("Sí, borrar todo", (1, 0, 0, 1), lambda *a: (popup.dismiss(), Thread(target=borrar, daemon=True).start())),
            ("Cancelar", (0.5, 0.5, 0.5, 1), lambda *a: popup.dismiss()),
            ("Volver al chat", (0, 0.5, 1, 1), lambda *a: (popup.dismiss(), self.mostrar_chat_principal()))
        ]

        for texto, color, accion in botones_config:
            btn = Button(
                text=texto,
                size_hint_y=None,
                height=dp(45),
                background_color=color,
                background_normal='',
                bold=True
            )
            btn.bind(on_release=accion)
            botones_box.add_widget(btn)

        box.add_widget(botones_box)

        popup = Popup(
            title="Confirmar borrado",
            content=box,
            size_hint=(0.85, None),
            height=dp(300), # Altura fija para que no se deforme en móvil
            auto_dismiss=False
        )
        popup.open()
   
    # --- AQUÍ LA PEGAS ---
    
    def cargar_memoria(self):
        import sqlite3
        conn = None
        try:
            # 1. Forzamos strings puros
            db_path_limpio = str(self.db_path)
            user_id_limpio = str(self.user_id)
            uid_seguro = getattr(self, 'user_id', 'usuario_anonimo')
            conn = sqlite3.connect(db_path_limpio)
            conn.text_factory = str 
            cursor = conn.cursor()
            
            # 2. USAMOS UN NOMBRE DE TABLA NUEVO ('chat_historial')
            # Esto evita cualquier 'datatype mismatch' de la tabla vieja 'mensajes'
            cursor.execute("""
                CREATE TABLE IF NOT EXISTS chat_historial (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    user_id TEXT, 
                    emisor TEXT, 
                    mensaje TEXT, 
                    timestamp TEXT
                )
            """)
            conn.commit()

            # 3. Consulta corregida usando la tabla nueva
            query = "SELECT emisor, mensaje FROM chat_historial WHERE user_id = ?"
            cursor.execute(query, (user_id_limpio,))
            filas = cursor.fetchall()

            # 4. Carga segura en la RAM
            with self.lock:
                self.historial = []
                for emi, msj in filas:
                    self.historial.append({
                        "emisor": str(emi) if emi else "user", 
                        "texto": str(msj) if msj else ""
                    })
            
            print(f"✅ Memoria cargada: {len(self.historial)} mensajes.")

        except Exception as e:
            print(f"⚠️ Error crítico cargando memoria: {e}")
            import traceback
            traceback.print_exc() 
        finally:
            if conn: conn.close()

    def _enviar_mensaje_firebase(self, mensaje):
     url = f"{FIREBASE_URL}/usuarios/{self.user_id}/chats/mensajes.json"
     try:
        response = requests.post(url, json=mensaje, timeout=5)

        if response.status_code == 200:
            print("✅ Mensaje enviado a Firebase")
        else:
            print(f"⚠️ Error {response.status_code}:", response.text)

     except Exception as e:
        print("❌ Error Firebase:", e)

   
    
    def soporte(self, instance=None):
     try:
        # 🔒 Si ya está abierto, no hacer nada
        if hasattr(self, "soporte_layout") and self.soporte_layout:
            return

        # Crear layout UNA sola vez
        self.soporte_layout = BoxLayout(
            orientation='vertical',
            size_hint_y=None,
            spacing=5
        )
        self.soporte_layout.bind(minimum_height=self.soporte_layout.setter('height'))

        # Input
        self.input_soporte = TextInput(
            hint_text="Escribe tu mensaje o problema...",
            size_hint_y=None,
            height=40,
            multiline=False
        )
        self.soporte_layout.add_widget(self.input_soporte)

        # Botón enviar
        enviar_btn = Button(
            text="Enviar",
            size_hint_y=None,
            height=40
        )
        enviar_btn.bind(on_release=lambda x: self.enviar_mensaje_soporte())
        self.soporte_layout.add_widget(enviar_btn)

        # Botón volver
        volver_btn = Button(
            text="Volver al chat",
            background_color=(0, 0.5, 1, 1),
            size_hint_y=None,
            height=40
        )
        volver_btn.bind(on_release=self.volver_al_chat)
        self.soporte_layout.add_widget(volver_btn)

        # Agregar al contenedor
        self.chat_container.add_widget(self.soporte_layout)

     except Exception as e:
        print(f"Error soporte en tiempo real: {e}")
        self.agregar_mensaje_ui("Tu historial de chats fue eliminado correctamente", f"❌ Error al iniciar soporte: {e}")

        def consultar_soporte(dt):
                try:
                    url = f"{FIREBASE_URL}usuarios/{self.user_id}/soporte.json"
                    res = requests.get(url, timeout=5)
                    if res.status_code == 200 and res.json():
                        data = res.json()
                        mensaje_soporte = data.get("mensaje", "")
                        problema = data.get("problema", "")
                        if mensaje_soporte:
                            self.agregar_mensaje_ui("Soporte", mensaje_soporte)
                        if problema:
                            self.agregar_mensaje_ui("Sistema", f"⚠️ Problema detectado: {problema}")
                except Exception as e:
                    print(f"Error al consultar soporte: {e}")
                Clock.schedule_interval(consultar_soporte, 5)
     except Exception as e:
                 print(f"Error soporte en tiempo real: {e}")
                 self.agregar_mensaje_ui("sistema", f"❌ Error al iniciar soporte: {e}") 

    def volver_al_chat(self, *args):
     try:
        if hasattr(self, "soporte_layout") and self.soporte_layout:
            if self.soporte_layout.parent:
                self.soporte_layout.parent.remove_widget(self.soporte_layout)

        self.soporte_layout = None

     except Exception as e:
        print(f"Error al volver al chat: {e}")
           
    def enviar_mensaje_soporte(self):
     mensaje = self.input_soporte.text.strip()

     if not mensaje:
        return

     data = {
        "mensaje": mensaje,
        "timestamp": datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    }

     # Mostrar mensaje del usuario
     self.agregar_mensaje_ui("Tú", mensaje)
     self.input_soporte.text = ""

     try:
        url = f"{FIREBASE_URL}usuarios/{self.user_id}/soporte.json"
        response = requests.patch(url, json=data)

        if response.status_code in (200, 204):
            self.agregar_mensaje_ui(
                "Aria",
                "✅ Tu mensaje fue enviado correctamente."
            )
        else:
            self.agregar_mensaje_ui(
                "Aria",
                "⚠️ Hubo un problema al enviar el mensaje."
            )

     except Exception as e:
        print("❌ Error:", e)
        self.agregar_mensaje_ui(
            "Aria",
            "⚠️ No se pudo enviar el mensaje por un error de conexión."
        )
 
    # UNIFICADO: Una sola forma de guardar en la nube
    def guardar_mensaje_firebase(self, mensaje_texto, emisor="usuario"):
      url = f"{FIREBASE_URL}usuarios/{self.user_id}/chats/mensajes.json"
      data = {
        "emisor": emisor,
        "mensaje": mensaje_texto,
        "timestamp": datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    }
      try:
        # Usamos un hilo para que la app no se trabe al enviar
        Thread(target=lambda: requests.post(url, json=data, timeout=5), daemon=True).start()
      except Exception as e:
        print(f"❌ Error Firebase: {e}")

     # UNIFICADO: Carga limpia
    def cargar_chat_completo(self):
     try:
        url = f"{FIREBASE_URL}usuarios/{self.user_id}/chats/mensajes.json"
        res = requests.get(url, timeout=5)
        if res.status_code == 200 and res.json():
             self.ids.contenedor_chat.clear_widgets() # Limpia UI vieja
             datos = res.json()
             # Ordenar por tiempo
             mensajes_ordenados = sorted(datos.values(), key=lambda x: x.get("timestamp", ""))
             for m in mensajes_ordenados:
                 self.agregar_mensaje_ui(m['emisor'], m['mensaje'])
     except Exception as e:
        print(f"❌ Error cargando chat: {e}")

     

   
    def guardar_respuesta_Aria(self, respuesta, id_token):

     if id_token is None:
        print("❌ Error: Se necesita un id_token para guardar la respuesta de VECTOR.")
        return

     url = f"{FIREBASE_URL}chats/{self.user_id}.json?auth={id_token}"

     data = {
        "emisor": "VECTOR",
        "mensaje": respuesta,
        "timestamp": datetime.datetime.now().isoformat()
    }

     try:
        response = requests.post(url, json=data)
        response.raise_for_status()
        print("✅ Respuesta de VECTOR guardada en Firebase")
     except Exception as e:
        print(f"❌ Error al guardar respuesta de VECTOR en Firebase: {e}")

    def agregar_mensaje_ui(self, emisor, mensaje, timestamp=None):
         if not timestamp:
            timestamp = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
         color = (1,1,1,1) if emisor.lower()=="usuario" else (0.8,0.8,0.8,1)
         lbl = Label(
            text=f"[{timestamp}] {emisor}: {mensaje}",
            size_hint_y=None,
            halign="left",
            valign="middle",
            text_size=(Window.width*0.85, None),
            color=color
        )
         lbl.bind(texture_size=lambda inst, val: setattr(inst, 'height', val[1]+10))
         self.grid.add_widget(lbl)
         Clock.schedule_once(lambda dt: self.scroll.scroll_to(lbl), 0.05)   

    

     # ======================================================
    # 📂 TODAS LAS FUNCIONES QUE PEGASTE VAN DESDE AQUÍ ABAJO
    # ======================================================
    def cargar_chat(self, chat_id):
        try:
            self.grid.clear_widgets()
            self.loading_label.text = "🕓 Cargando mensajes..."
            url = f"{FIREBASE_URL}usuarios/{self.user_id}/chats/{chat_id}/mensajes.json"
            response = requests.get(url, timeout=5)
            mensajes = response.json() if response.status_code == 200 else {}

            if not mensajes:
                self.grid.add_widget(Label(
                    text="(No hay mensajes en este chat aún)",
                    color=(0.7, 0.7, 0.7, 1),
                    size_hint_y=None,
                    height=30
                ))
                self.loading_label.text = ""
                return

            mensajes_ordenados = sorted(mensajes.values(), key=lambda m: m.get("timestamp", ""))
            for msg in mensajes_ordenados:
                self.mostrar_mensaje(
                    msg.get("emisor", "Desconocido"),
                    msg.get("mensaje", ""),
                    msg.get("timestamp", "")
                )

            self.loading_label.text = "✅ Chat cargado."
        except Exception as e:
            print(f"❌ Error al cargar el chat: {e}")
            self.loading_label.text = f"Error: {e}"
    
 



    def _mostrar_lista_chats(self, chats):
     self.chat_container.clear_widgets()

     lbl_titulo = Label(
        text="[b]🗂️ Chats guardados del usuario:[/b]",
        markup=True,
        font_size=18,
        color=(0.7, 0.9, 1, 1),
        size_hint_y=None,
        height=40
    )
     self.chat_container.add_widget(lbl_titulo)

     for chat_id, contenido in chats.items():
          fecha = contenido.get("fecha", "sin fecha")
          resumen = contenido.get("ultimo_mensaje", "sin contenido")

          lbl_chat = Button(
            text=f"📅 {fecha} — {resumen[:40]}...",
            size_hint_y=None,
            height=50,
            background_color=(0.2, 0.4, 0.6, 1),
            color=(1, 1, 1, 1),
            font_size=15
        )
          lbl_chat.bind(on_press=lambda instance, cid=chat_id: self._abrir_chat_guardado(cid))
          self.chat_container.add_widget(lbl_chat)

          Clock.schedule_once(lambda dt: self.scroll.scroll_to(lbl_titulo), 0.1)
    
    def cargar_chat(self, chat_id):
     # RUTA UNIFICADA (sin el /mensajes.json al final)
     url = f"{FIREBASE_URL}usuarios/{self.user_id}/chats/{chat_id}.json"
    
     response = requests.get(url, timeout=5)
    


    def guardar_en_firebase(self, chat_id, emisor, texto):
     # LA MISMA RUTA QUE ARRIBA
     url = f"{FIREBASE_URL}usuarios/{self.user_id}/chats/{chat_id}.json"
    
     data = {"emisor": emisor, "mensaje": texto, "timestamp": time.time()}
     requests.post(url, json=data)
    
    def _inicializar_contenido(self, dt=None):
        try:
            # 🛡️ Si la pantalla se está cerrando, self.ids ya no existe
            if not self or not hasattr(self, 'ids'):
                return

            # Intentamos obtener el contenedor de forma segura
            # Buscamos 'chat_panel' o 'contenedor_chat' (los nombres que usaste)
            contenedor = getattr(self, 'chat_panel', None) or self.ids.get('contenedor_chat')
            # Asignamos el contenedor
            self.chat_container = self.chat_panel 

            if contenedor:# Cargamos el chat (Aquí es donde salta el error de 'contenido')
              if self.user_id:
               self.cargar_chat(self.user_id)
               self.cargar_memoria()
            self.chat_container = contenedor
                # 🔹 Solo intentamos cargar si el objeto está "vivo"
            if hasattr(self, 'user_id') and self.user_id:
                    self.cargar_chat(self.user_id)
            else:
                print("⚠️ No se encontró el contenedor visual.")
                
        except (ReferenceError, Exception) as e:
            # Aquí es donde "matamos" el error para que la app no se cierre
            print(f"✅ Error de referencia evitado: {e}")

        # 🔹 Cargar chat solo si tenemos user_id
        if hasattr(self, 'user_id') and self.user_id:
         self.cargar_chat(self.user_id)

    # Placeholder methods que asumimos definidos en tu proyecto
    def toggle_sidebar(self, instance): pass
    def volver_principal(self, instance): pass
    def agregar_mensaje_ui(self, emisor, mensaje): pass

    # ==========================================
    # 🔄 FUNCIONES DE INTERACCIÓN
    def toggle_sidebar(self, instance):
        if self.sidebar_visible:
            self.sidebar_scroll.width = 0
        else:
            self.sidebar_scroll.width = Window.width * 0.75
        self.sidebar_visible = not self.sidebar_visible

    def volver_principal(self, instance):
        self.clear_widgets()
        mensaje = Label(
            text="🏠 Volviste al panel principal",
            font_size=22,
            color=(1, 1, 1, 1),
            halign="center",
            valign="middle"
        )
        self.add_widget(mensaje)
   
def start_sync(self):
        """Inicia sincronización en segundo plano"""
        if not self.running:
            self.running = True
            import threading
            threading.Thread(target=self.sync_loop, daemon=True).start()
def sync_loop(self):
        while self.running:
            try:
                self.sync_to_firebase()
                self.sync_from_firebase()
            except Exception as e:
                print(f"⚠️ Error en sincronización: {e}")
            time.sleep(self.sync_interval)
     
def sync_to_firebase(self):
     try:
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        cursor.execute(
            "SELECT id, mensaje, respuesta, temas, synced FROM chats WHERE user_id = ?",
            (self.user_id,)
        )
        filas = cursor.fetchall()

        for chat_id, mensaje, respuesta, temas, synced in filas:
            if synced == 0:
                url = f"{FIREBASE_URL}usuarios/{self.user_id}/chats/{chat_id}.json"
                payload = {
                    "mensaje": mensaje,
                    "respuesta": respuesta,
                    "temas": temas.split(",") if temas else []
                }
                try:
                    response = requests.put(url, json=payload)
                    if response.status_code in (200, 204):
                        cursor.execute("UPDATE chats SET synced = 1 WHERE id = ?", (chat_id,))
                    else:
                        print(f"⚠️ Error {response.status_code} al subir chat {chat_id}")
                except Exception as e:
                    print(f"⚠️ No se pudo subir chat {chat_id}:", e)

        conn.commit()
        conn.close()
        print("✅ Sincronización de chats a Firebase completada.")

     except Exception as e:
        print("❌ Error en sync_to_firebase:", e)

def sync_from_firebase(self):
     try:
        # --- Obtener chats desde Firebase REST ---
        url = f"{FIREBASE_URL}usuarios/{self.user_id}/chats.json"
        response = requests.get(url, timeout=5)
        if response.status_code == 200:
            data = response.json() or {}
        else:
            print(f"⚠️ Error {response.status_code} al obtener chats")
            data = {}

        # --- Sincronizar con SQLite local ---
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()

        for chat_id, chat_data in data.items():
            cursor.execute("SELECT COUNT(*) FROM chats WHERE id = ?", (chat_id,))
            existe = cursor.fetchone()[0]
            if existe == 0:
                cursor.execute("""
                    INSERT INTO chats (id, user_id, mensaje, respuesta, temas, synced)
                    VALUES (?, ?, ?, ?, ?, 1)
                """, (
                    chat_id,
                    self.user_id,
                    chat_data.get("mensaje", ""),
                    chat_data.get("respuesta", ""),
                    ",".join(chat_data.get("temas", [])),
                ))

        conn.commit()
        conn.close()
        print(f"✅ Chats sincronizados desde Firebase: {len(data)}")
        
     except Exception as e:
        print("⚠️ No se pudo sincronizar desde Firebase:", e)
    
        def _mostrar_respuesta_typing(self, texto, indice=0):
         if indice < len(texto):
            self.mensaje_actual += texto[indice]
            # Actualizar UI con el mensaje parcial
            Clock.schedule_once(lambda dt: self.agregar_mensaje_ui("Aria", self.mensaje_actual), 0)
            # Llamada recursiva para siguiente letra
            Clock.schedule_once(lambda dt: self._mostrar_respuesta_typing(texto, indice+1), self.intervalo_typing)
         else:
            # Mensaje completo ya mostrado
            pass


def stop_sync(self):
        self.running = False
def volver_principal(self, instance):
        self.clear_widgets()
        self.add_widget(Label(text="🏠 Volviste al panel principal", font_size=22, color=(1,1,1,1)))

def guardar_usuario(self, user_dict):
        import json
        with open(self.user_data_path, "w", encoding="utf-8") as f:
            json.dump(user_dict, f, ensure_ascii=False, indent=2)
def cargar_usuario(self):
        import json, os
        if os.path.exists(self.user_data_path):
            with open(self.user_data_path, "r", encoding="utf-8") as f:
                return json.load(f)
        return {}
    

 
def get_db_path(filename="dyslexialift_data.db"):
     if platform == "android":
        from android.storage import app_storage_path # pyright: ignore[reportMissingImports]
        base_path = app_storage_path()
     else:
        base_path = os.path.dirname(__file__)
     return os.path.join(base_path, filename)

def get_user_data_path(filename="user_data.json"):
     if platform == "android":
        from android.storage import app_storage_path # type: ignore
        base_path = app_storage_path()
     else:
        base_path = os.path.dirname(__file__)
     return os.path.join(base_path, filename)

 # -----------------------------
# Cargar historial de Firebase
# -----------------------------
def cargar_historial(self, mensajes=None):
    if not self.chat_container:
        print("⚠️ chat_container no inicializado, abortando carga")
        return

    if mensajes is None:
        # Obtener mensajes desde Firebase
        url = f"{FIREBASE_URL}usuarios/{self.user_id}/chats/mensajes.json"
        response = requests.get(url, timeout=5)
        mensajes = response.json() or {}

    # Limpiar contenedor antes de cargar
    self.chat_container.clear_widgets()

    # Ordenar por timestamp
    mensajes_ordenados = sorted(mensajes.values(), key=lambda m: m.get("timestamp", ""))
    for msg in mensajes_ordenados:
        self.mostrar_mensaje(msg.get("emisor", ""), msg.get("mensaje", ""), msg.get("timestamp", ""))

# -----------------------------
# Mostrar un mensaje en la UI
# -----------------------------
def mostrar_mensaje(self, emisor, texto, timestamp=None):
    """Muestra mensajes en el chat (tanto del usuario como de aria)."""
    prefijo = f"[{timestamp}] " if timestamp else ""
    label = Label(
        text=f"{prefijo}{emisor}: {texto}",
        halign="left" if emisor.lower() != "tú" else "right",
        valign="top",
        text_size=(Window.width * 0.9, None),
        size_hint_y=None
    )
    label.bind(texture_size=lambda inst, val: setattr(inst, "height", val[1]))

    contenedor = BoxLayout(size_hint_y=None, padding=[10, 5])
    contenedor.add_widget(label)

    if self.chat_container:
     self.chat_container.add_widget(contenedor)
     Clock.schedule_once(lambda dt: self.scroll.scroll_to(contenedor), 0.05)
    else:
     print("⚠️ chat_container no inicializado")


# Escuchar mensajes nuevos
# -----------------------------

def escuchar_mensajes_stream(self):
    """Escucha mensajes en tiempo real usando REST streaming"""
    url = f"{FIREBASE_URL}usuarios/{self.user_id}/chats/mensajes.json"
    
    def stream():
        try:
            with requests.get(url, stream=True) as r:
                if r.status_code != 200:
                    print("⚠️ Error al conectar al stream:", r.status_code)
                    return
                buffer = ""
                for chunk in r.iter_content(chunk_size=1):
                    if chunk:
                        buffer += chunk.decode("utf-8")
                        if buffer.endswith("\n"):
                            try:
                                line = buffer.strip()
                                buffer = ""
                                if line.startswith("data: "):
                                    data = line[6:]
                                    if data != "null":
                                        mensajes = json.loads(data)
                                        # Actualiza interfaz en el hilo principal
                                       
                                        Clock.schedule_once(lambda dt: self.cargar_historial(mensajes))
                            except Exception as e:
                                print("❌ Error parseando mensaje:", e)
        except Exception as e:
            print("❌ Error en el stream:", e)

    Thread(target=stream, daemon=True).start()
    print("✅ Escuchando mensajes en tiempo real vía REST streaming...")






# -----------------------------
# Mostrar mensaje progresivo con emojis
# -----------------------------
def mostrar_mensaje_progresivo(self, emisor, texto):
    burbuja = self.crear_burbuja_mensaje(emisor)
    self.grid.add_widget(burbuja)

    index = 0
    mensaje_actual = ""
    emojis = ["🙂","😃","✨","💡","👍","💬","🎵","📚","🎨"]

    def escribir(dt):
        nonlocal index, mensaje_actual
        if index < len(texto):
            mensaje_actual += texto[index]
            if index % 15 == 0 and index != 0:
                mensaje_actual += " " + choice(emojis)
            burbuja.text = mensaje_actual
            index += 1
            return True
        else:
            burbuja.text += " " + choice(emojis)
            return False

    Clock.schedule_interval(escribir, 0.02)


   
  
    # -----------------------------
    # Toggle sidebar
    # -----------------------------
def toggle_sidebar(self):
        if self.sidebar_visible:
            Animation(width=0, duration=0.3).start(self.sidebar)
            self.sidebar_visible = False
        else:
            Animation(width=250, duration=0.3).start(self.sidebar)
            self.sidebar_visible = True

    # -----------------------------
    # Cargar sidebar con usuarios
    # -----------------------------
def cargar_sidebar(self):
    self.sidebar.clear_widgets()
    lbl_header = Label(text="Usuarios / Chats", size_hint_y=None, height=40)
    self.sidebar.add_widget(lbl_header)

    try:
        url = f"{FIREBASE_URL}chats.json"
        response = requests.get(url, timeout=5)
        response.raise_for_status()
        data = response.json() or {}

        for user_id, mensajes in data.items():
            ultimo_mensaje = ""
            if mensajes:
                # Tomar el último mensaje
                ultimo_mensaje = list(mensajes.values())[-1].get('mensaje', "")
            btn = Button(
                text=f"{user_id}: {ultimo_mensaje[:30]}...",
                size_hint_y=None,
                height=40
            )
            btn.bind(on_release=lambda btn, uid=user_id: self.cargar_chat(uid))
            self.sidebar.add_widget(btn)

    except Exception as e:
        print("❌ Error cargar_sidebar:", e)
    # -----------------------------
    # Cargar chat seleccionado
    # -----------------------------
def cargar_chat(self, user_id):
        try:
            # 1. Limpiamos el contenedor visual
            if hasattr(self, 'grid'):
                self.grid.clear_widgets()
            
            self.user_id = user_id

            # 2. Conexión a la base de datos local
            conn = sqlite3.connect(self.db_path)
            cursor = conn.cursor()
            
            # ✅ AQUÍ ESTÁ EL CAMBIO: Usamos 'mensaje', NO 'contenido'
            query = "SELECT emisor, mensaje, timestamp FROM mensajes WHERE user_id = ? ORDER BY timestamp ASC"
            cursor.execute(query, (user_id,))
            
            filas = cursor.fetchall()
            
            # 3. Dibujamos los mensajes en la interfaz
            for fila in filas:
                emisor = fila[0]
                texto = fila[1]
                t_stamp = fila[2] if len(fila) > 2 else ""
                
                self.agregar_mensaje_ui(emisor, texto, t_stamp)

            conn.close()
            print(f"✅ Chat cargado desde SQLite: {len(filas)} mensajes")
            
        except Exception as e:
            print(f"❌ Error al cargar chat desde DB: {e}")
    
def iniciar_listener_mensajes(self, intervalo=5):
    """Listener optimizado para móvil (bajo consumo)"""
    
    self._ultimo_timestamp = getattr(self, "_ultimo_timestamp", "")

    def actualizar(dt):
        Thread(target=self._fetch_nuevos_mensajes, daemon=True).start()

    Clock.schedule_interval(actualizar, intervalo)
    print("✅ Listener optimizado activo")


def _fetch_nuevos_mensajes(self):
     try:
        url = f"{FIREBASE_URL}usuarios/{self.user_id}/chats.json"
        
        response = requests.get(url, timeout=5)
        data = response.json() or {}

        nuevos = []

        for chat in data.values():
            for msg in chat.get("mensajes", {}).values():
                if msg["timestamp"] > self._ultimo_timestamp:
                    nuevos.append(msg)

        nuevos.sort(key=lambda x: x["timestamp"])

        if nuevos:
            self._ultimo_timestamp = nuevos[-1]["timestamp"]

            Clock.schedule_once(lambda dt: self._mostrar_nuevos(nuevos))

     except Exception as e:
        print("❌ Error listener:", e)

     def _mostrar_nuevos(self, mensajes):
      for msg in mensajes:
        self.agregar_mensaje_ui(
            msg.get("emisor", ""),
            msg.get("mensaje", ""),
            msg.get("timestamp", "")
        )

# Cargar historial con paginación
# -----------------------------



def _cargar_historial_thread(self, limite):
    try:
        url = f"{FIREBASE_URL}usuarios/{self.user_id}/chats.json?orderBy=\"timestamp\"&limitToLast={limite}"
        
        response = requests.get(url, timeout=5)
        data = response.json() or {}

        mensajes = []

        for chat in data.values():
            mensajes.extend(chat.get("mensajes", {}).values())

        mensajes.sort(key=lambda x: x["timestamp"])

        Clock.schedule_once(lambda dt: self._render_historial(mensajes))

    except Exception as e:
        print("❌ Error historial:", e)
# -----------------------------
# Soporte en “tiempo real” vía REST
# -----------------------------





# Nombres de usuario en memoria
# -----------------------------
user_names = {}


# -----------------------------
# Chats automáticos por usuario

def obtener_nombre_usuario(self, user_id):
    # 1. Mirar en la RAM primero (Rápido)
    if user_id in user_names:
        return user_names[user_id]
    
    # 2. Si no está en RAM, devolver "Usuario" por defecto de inmediato 
    # y lanzar la búsqueda en Firebase en SEGUNDO PLANO
    def buscar():
        try:
            url = f"{FIREBASE_URL}usuarios/{user_id}/nombre.json"
            res = requests.get(url, timeout=3)
            if res.status_code == 200 and res.json():
                user_names[user_id] = res.json()
        except: pass

    Thread(target=buscar, daemon=True).start()
    return "Usuario"





def cargar_chat(self, chat_key):
    self.chat_layout.clear_widgets()
    try:
        url = f"{FIREBASE_URL}usuarios/{self.user_id}/chats/{chat_key}.json"
        response = requests.get(url, timeout=5) 
        response.raise_for_status()
        mensajes = response.json() or {}

        for m in mensajes.values():
            self.agregar_mensaje_ui(m.get('emisor', ''), m.get('mensaje', ''))

    except Exception as e:
        print("❌ Error cargar_chat:", e)


# =====================================================
# 👥 Gestión de sincronización
# =====================================================
class SyncManager:
    def __init__(self, user_id):
        self.user_id = user_id

    def start_sync(self):
        print(f"🔄 Sincronización automática para usuario {self.user_id} iniciada correctamente.")


# =====================================================
# 💬 Pantalla Chat (Mobile Ready)
# =====================================================
class PantallaChat(BoxLayout):
    def __init__(self, **kwargs):
        super().__init__(orientation="vertical", **kwargs)
        # Aquí definimos las proporciones para que en el celular se vea bien
        self.chat_panel = BoxLayout(size_hint_y=0.8)
        self.input_panel = BoxLayout(size_hint_y=0.1)
        self.botones_panel = BoxLayout(size_hint_y=0.1)

        self.add_widget(self.chat_panel)
        self.add_widget(self.input_panel)
        self.add_widget(self.botones_panel)

        # Usamos el Clock para no trabar el inicio de la App
        Clock.schedule_once(self.inicializar, 0)

    def inicializar(self, dt):
        # 1. Cargamos el nombre del usuario
        self.nombre_usuario = obtener_nombre_usuario(self.user_id)
        
        # 2. Iniciamos la memoria limpia
        self.memoria = Memoria() 
        
        # 3. Cargamos el historial de Firebase
        self.cargar_chat_completo()
        
        print(f"📱 UI de {self.nombre_usuario} lista para móvil") 

    def corregir_texto_dislexia(self, texto):
        """
        Aquí es donde sucede la magia para ayudar al usuario.
        Por ahora es simple, pero puedes agregarle un corrector ortográfico.
        """
        if not texto:
            return ""
            
        # Ejemplo: corregir errores comunes de teclado o espacios dobles
        texto_limpio = " ".join(texto.split())
        
        # Aquí podrías conectar una librería de corrección más adelante
        return texto_limpio 

      # =====================================================
         # 🧠 Memoria de chat
          # =====================================================
class Memoria:
    def __init__(self, max_mensajes=50,nombre_db="asistente.db"):
        self.max_mensajes = max_mensajes
        self.historial = [] # Memoria a corto plazo (RAM)
       
       
        # Ruta segura para Android
        self.db_path = os.path.join(App.get_running_app().user_data_dir, "aria.db")
        self.crear_tablas()
        self.cargar_historial_desde_db() 
        self.conexion = sqlite3.connect(nombre_db)
        self.cursor = self.conexion.cursor()

    def crear_tablas(self):
        with sqlite3.connect(self.db_path) as conn:
            conn.execute("""
                CREATE TABLE IF NOT EXISTS mensajes (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    user_id TEXT,
                    rol TEXT,
                    contenido TEXT,
                    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP
                )
            """)
    def crear_tabla(self):
        # Asegúrate de que aquí diga 'mensaje' y no 'contenido'
        self.cursor.execute('''
            CREATE TABLE IF NOT EXISTS historial (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                usuario_id TEXT,
                emisor TEXT,
                mensaje TEXT, 
                fecha TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        ''')
        self.conexion.commit()

    def guardar_mensaje(self, usuario_id, emisor, mensaje):
        # El error suele saltar aquí si los nombres no coinciden
        self.cursor.execute('''
            INSERT INTO historial (usuario_id, emisor, mensaje) 
            VALUES (?, ?, ?)
        ''', (usuario_id, emisor, mensaje))
        self.conexion.commit()
    
    def cargar_historial_desde_db(self):
        try:
            import sqlite3
            import threading
            
            # 1. Definimos las variables seguras
            u_id = getattr(self, 'user_id', 'usuario_defecto')
            
            # 2. CREAMOS EL BLOQUEO AQUÍ (Esto evita el error del 'lock')
            bloqueo = getattr(self, 'lock', threading.Lock())

            with sqlite3.connect(DB_PATH) as conn:
                conn.row_factory = sqlite3.Row
                
                cursor = conn.execute(
                    "SELECT emisor, mensaje FROM mensajes WHERE user_id = ? ORDER BY timestamp DESC LIMIT ?", 
                    (str(u_id), 100) 
                )
                filas = cursor.fetchall()
                
                # 3. USAMOS 'bloqueo' (NO self.lock)
                with bloqueo:
                    self.historial = [] 
                    for fila in reversed(filas):
                        self.historial.append({
                            "emisor": str(fila["emisor"]), 
                            "texto": str(fila["mensaje"])
                        })
                        
            print(f"📚 Memoria recuperada: {len(self.historial)} mensajes.")
        except Exception as e:
            # Si sale error aquí, ahora nos dirá exactamente qué es
            print(f"⚠️ Error cargando memoria: {e}")
    
    def limpiar_chat(self, usuario_id):
        """Borra historial local creando una conexión segura para este hilo"""
        import sqlite3
        try:
            # 1. Creamos una conexión NUEVA solo para esta operación
            # Esto evita el error de hilos (thread id mismatch)
            conn_temp = sqlite3.connect(self.db_path) 
            cursor_temp = conn_temp.cursor()
            
            # 2. Borramos usando la tabla correcta (asegurate si es 'mensajes' o 'chat_historial')
            query = "DELETE FROM chat_historial WHERE user_id = ?"
            cursor_temp.execute(query, (str(usuario_id),))
            
            conn_temp.commit()
            conn_temp.close() # Cerramos la conexión temporal
            
            # 3. Vaciamos la lista en la RAM (el historial que Aria está leyendo ahora)
            if hasattr(self, 'historial_actual'):
                self.historial_actual = []
            
            # Si usas otra variable como 'self.historial', vaciala también
            self.historial = [] 

            print(f"🗑️ Memoria local limpia para: {usuario_id}")
            return True

        except Exception as e:
            print(f"❌ Error al limpiar chat en Memoria: {e}")
            return False

    def guardar_mensaje(self, user_id, emisor_rol, contenido):
        with self.lock:
            self.historial.append({"emisor": emisor_rol, "texto": contenido})
            if len(self.historial) > 20: # O tu max_mensajes
                self.historial.pop(0)
    
        try:
            conn = sqlite3.connect(self.db_path)
            cursor = conn.cursor()
            
            # Forzamos todo a string aquí mismo
            u_id = str(user_id)
            rol = str(emisor_rol)
            txt = str(contenido)
            fecha = datetime.now().strftime("%Y-%m-%d %H:%M:%S")

            cursor.execute('''
                INSERT INTO mensajes (user_id, rol, emisor, contenido, mensaje, timestamp)
                VALUES (?, ?, ?, ?, ?, ?)
            ''', (u_id, rol, rol, txt, txt, fecha))
            
            conn.commit()
            conn.close()
            print("✅ Mensaje guardado sin errores de tipo.")
        except Exception as e:
            print(f"❌ Error al guardar en DB: {e}")

    def obtener_para_prompt(self, limite=6):
        """Devuelve el contexto para que Aria sepa de qué están hablando"""
        with self.lock:
            mensajes = self.historial[-limite:]
            return "\n".join([f"{m['emisor']}: {m['texto']}" for m in mensajes])
        
       # --- PEGA AQUÍ EL PUNTO 3 ---
    def crear_tablas(self):
        conn = sqlite3.connect(self.db_path)
        c = conn.cursor()
        c.execute('''
            CREATE TABLE IF NOT EXISTS mensajes (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                usuario_id TEXT,
                emisor TEXT,
                mensaje TEXT,      
                fecha DATETIME DEFAULT CURRENT_TIMESTAMP
            )
        ''')
        conn.commit()
        conn.close()

    # --- PEGA AQUÍ EL PUNTO 2 ---
    def guardar_mensaje(self, usuario_id, emisor, texto):
        conn = sqlite3.connect(self.db_path)
        c = conn.cursor()
        c.execute("""
            INSERT INTO mensajes (usuario_id, emisor, mensaje) 
            VALUES (?, ?, ?)
        """, (usuario_id, emisor, texto))
        conn.commit()
        conn.close() 

    def limpiar_todo(self):
        """Borra todo (Cuidado: borra recuerdos)"""
        with self.lock:
            self.historial.clear()
        with sqlite3.connect(self.db_path) as conn:
            conn.execute("DELETE FROM mensajes")
        print("🧹 Memoria reseteada por completo.")

    def obtener_memoria_sistema(self):
    # En móviles no necesitamos monitorear la RAM real para que el chat funcione
     try:
        import platform
        # Verificamos si es Android (que corre sobre un kernel Linux)
        if platform.system() == "Linux":
            return "Modo Móvil: Memoria gestionada por el sistema"
        return "Modo Escritorio: Memoria activa"
     except Exception:
        # Si todo falla, devolvemos un texto neutro
        return "Estado de memoria: Estable"    

    def cerrar(self):
        """Cierra la conexión a la base de datos de forma segura"""
        if hasattr(self, 'conexion') and self.conexion:
            self.conexion.close()
            print("🗄️ Conexión a la base de datos cerrada correctamente.")

# =====================================================
# 🔄 Contexto seguro para Llama
# =====================================================

    
class llamaModel:
    def __init__(self, model_path: str):
        self.model_path = Path(model_path)
        self.llm = None
        self.is_loaded = False
        
        if not self.model_path.exists():
            print(f"❌ Modelo no encontrado: {self.model_path}")
            return

        try:
            # Optimización de hilos para no calentar el móvil
            cpu_cores = multiprocessing.cpu_count()
            num_threads = max(1, min(4, cpu_cores - 1))

            self.llm = Llama(
                model_path=str(self.model_path),
                n_ctx=768,         # Reducido de 1024 a 512 (suficiente para chats cortos y ahorra mucha RAM)
                n_threads=2,       # En móviles, 2 hilos suele ser más estable que usar todos
                n_batch=8,
                use_mmap=True,     # Correcto, mantiene el modelo fuera de la RAM activa
               use_mlock=False,   # Asegúrate de que esté en False para que Android no mate la app
                verbose=False
            )
            self.is_loaded = True
            print(f"✅ Aria cargada con {num_threads} hilos.")
        except Exception as e:
            print("❌ Error cargando el cerebro de Aria:", e)
    
    
 
    def generar_respuesta(self, prompt: str):
        # 1. Verificamos si Aria está cargada
        if not self.is_loaded or self.llm is None:
            return "⚠️ Aria no está lista."
        
        
         

        # 2. Si self.llm es la "caja" (la clase), sacamos el motor
        if hasattr(self.llm, 'llm'):
            self.llm = self.llm.llm  
        
     
        try:
            # --- EL CAMBIO CLAVE ESTÁ AQUÍ ---
            # Usamos self.llm directamente, porque ya lo "desempaquetamos" arriba
             resultado = self.llm.create_completion(
            prompt=prompt,
            max_tokens=35,          # Suficiente para una respuesta clara
            temperature=0.1,        # Casi nada de creatividad para que no invente
            top_p=0.1,             # Filtra las palabras más probables (más enfocado)
            
            # PARÁMETRO CLAVE: Subimos el castigo por repetición
            repeat_penalty=2.0,     # Antes era 1.5, con 1.8 es mucho más estricto
            
            # STOPS: Agregamos el doble salto de línea y etiquetas de IA
            stop=[
                "Usuario:", 
                "Aria:", 
                f"{self.usuario_id}:", 
                "\n\n",             # Para si intenta hacer párrafos nuevos
                "Aria: Aria:",       # Por si entra en bucle de nombres
                "###"
            ],
            echo=False
        )

                   # Extraer texto
             respuesta = resultado["choices"][0]["text"].strip()

             # Limpieza para dislexia
             if "\n" in respuesta:
                respuesta = respuesta.split("\n")[0].strip()

             return respuesta if respuesta else "Aquí estoy. ¿En qué te ayudo?"

        except Exception as e:
            print("❌ Error en generación:", e)
            return "Tuve un pequeño problema, ¿podrías repetir eso?"

    def close(self):
        if self.llm:
            self.llm.close()
            self.is_loaded = False
            print("✅ Memoria de IA liberada.")

    from kivy.uix.screenmanager import Screen
    class MainScreen(Screen):

     def __init__(self, **kwargs):
        super().__init__(**kwargs)

        # 🔥 TODO adentro de la clase
        Thread(
            target=self.cargar_modelo_llama,
            daemon=True
        ).start()

   


# --- App principal ---
class MainAria(App):
    def build(self):
        # 🔹 Inicializar base
        self.db_path = os.path.join(
            App.get_running_app().user_data_dir,
            "aria.db"
        )

        self.crear_tabla_mensajes()
        self.memoria = Memoria(self.db_path)
        usuario_id ="09a66e08-c82f-457d-858d-fd4ce964b4bb"
        ruta_final = get_model_path()
        self.model_path = get_model_path()
        # ✅ FORMA CORRECTA Y LIMPIA
        chat_screen = ChatScreen(usuario_id, llamaModel(model_path=get_model_path()).llm, self.memoria, self.db_path)
        Clock.schedule_once(self.cargar_modelo_en_fondo, 2) 
        chat_screen.size_hint = (1, 1)

        return chat_screen
    
    def guardar_fecha_limpieza(self):
        import os
        # Usamos self.user_data_dir directamente porque estamos dentro de la App
        directorio = self.user_data_dir
        ruta_final = os.path.join(directorio, "ultima_limpieza_memoria.json")
        
        try:
            with open(ruta_final, "w") as f:
                f.write('{"fecha": "2026-04-27"}')
            print(f"✅ Fecha de limpieza guardada en: {ruta_final}")
        except Exception as e:
            print(f"❌ Error al guardar fecha: {e}")

    def on_stop(self):
     self.guardar_fecha_limpieza()        

    def on_stop(self):
         if hasattr(self, "memoria"):
            self.memoria.cerrar()
    

    def cargar_modelo_llama(self):
     try:
        # 📂 Obtener ruta correcta
        print("📂 Ruta modelo:", self.model_path)

        if not os.path.exists(self.model_path):
            print("❌ Modelo NO encontrado")
            self.llm = None
            self.is_loaded = False
            return

        print("🧠 Cargando modelo...")

        # AJUSTES OPTIMIZADOS PARA MÓVIL:
        self.llm = Llama(
                    model_path=self.model_path,
                    n_ctx=768,        # Bajamos de 1024 a 768 para cargar más rápido
                    n_threads=2,      # Bajamos de 4 a 2 para evitar calor extremo
                    n_batch=8,        # Bajamos de 32 a 8 para estabilidad RAM
                    n_gpu_layers=0,
                    use_mmap=True,    # Agregamos esto para ahorrar RAM
                    verbose=False
                ) 

        

        # 🔥 ESTA LÍNEA ES CLAVE
        self.is_loaded = True

        print("✅ Modelo cargado correctamente")

     except Exception as e:
        print("❌ Error cargando modelo:", e)
        self.llm = None
        self.is_loaded = False

    def cargar_modelo_en_fondo(self, dt):
        from threading import Thread
        # Aquí buscas a la función hermana que está abajo
        Thread(target=self.cargar_modelo_llama, daemon=True).start()

    def crear_tabla_mensajes(self):
     conn = sqlite3.connect(self.db_path)
     cursor = conn.cursor()

     cursor.execute("""
    CREATE TABLE IF NOT EXISTS mensajes (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        usuario_id TEXT NOT NULL,
        rol TEXT NOT NULL,
        mensaje TEXT NOT NULL,
        timestamp DATETIME DEFAULT CURRENT_TIMESTAMP
    )
    """)

     conn.commit()
     conn.close()
   
    
if __name__ == "__main__": 
    crear_tablas()
    MainAria().run()

    def cerrar_modelo_seguro():
     if platform == "android":
        return  # no hay modelo en Android

    try:
        if hasattr(MainAria, "llm") and MainAria.llm:
            MainAria.llm = None
            print("✅ Modelo cerrado correctamente")
    except Exception as e:
        print("⚠️ Error cerrando modelo:", e)

        atexit.register(cerrar_modelo_seguro)

  

  # Context manager
    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        if hasattr(self, 'modelo') and self.modelo is not None:
            try:
                self.modelo.close()
                print("✅ Modelo cerrado al salir del contexto")
            except Exception as e:
                print("⚠️ Error al cerrar el modelo:", e)
            finally:
                self.modelo = None 

def on_stop(self):
    print("La app se está cerrando, guardando datos...")

    # ✅ Guardar historial de chat
    if hasattr(self, "chat_screen") and self.chat_screen:
        try:
            self.chat_screen.memoria_usuario.guardar_historial()
            print("✅ Historial guardado")
        except Exception as e:
            print("⚠️ No se pudo guardar historial:", e)

    # 🧠 Compatibilidad: solo si estás en PC con IA
    from kivy.utils import platform

    if platform != "android":
        if hasattr(self, "llm") and self.llm:
            try:
                self.llm = None
                print("✅ Modelo cerrado (modo PC)")
            except Exception as e:
                print("⚠️ Error cerrando modelo:", e)


def build(self):
        # 🔹 Crear memoria al iniciar la app
        self.memoria = Memoria()
        return ChatScreen()

def on_stop(self):
        # 🔒 Se ejecuta automáticamente cuando la app se cierra
        if hasattr(self, "memoria"):
            self.memoria.cerrar()



DB_PATH = get_db_path()


# -------------------------------
# 🔧 Ejecutar query
# -------------------------------
def ejecutar_query(query, params=()):
    try:
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()
        cursor.execute(query, params)
        conn.commit()
    except Exception as e:
        print(f"❌ Error: {e}")
    finally:
        conn.close()

def crear_tablas():
    with lock_db:
        with sqlite3.connect(DB_PATH) as conn:
            c = conn.cursor()
            c.execute('''
                CREATE TABLE IF NOT EXISTS mensajes (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    user_id TEXT,
                    rol TEXT,          -- <--- USA ROL AQUÍ
                    mensaje TEXT,
                    timestamp DATETIME,
                    importancia INTEGER DEFAULT 0,
                    tema_detectado TEXT,
                    emocion TEXT
                )
            ''')
            # Si la tabla ya existía con 'emisor', este comando la arregla:
            try:
                c.execute("ALTER TABLE mensajes ADD COLUMN rol TEXT")
            except:
                pass 
            conn.commit()

# Busca esta parte al inicio de tu archivo aria_funcion.py
def crear_tablas():
    with sqlite3.connect(DB_PATH) as conn:
        c = conn.cursor()
        c.execute('''
            CREATE TABLE IF NOT EXISTS mensajes (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id TEXT,
                rol TEXT,
                contenido TEXT,     -- Asegúrate que diga contenido y no mensaje
                timestamp DATETIME
            )
        ''')
        conn.commit()

def inicializar_tablas_fijas():
    try:
        # 1. ARREGLAR TABLA DE MENSAJES (Historial)
        conn_m = sqlite3.connect("mensajes.db")
        cursor_m = conn_m.cursor()
        
        # Este es el truco: si la tabla no tiene 'emisor', la borramos y recreamos
        cursor_m.execute("DROP TABLE IF EXISTS mensajes") 
        
        cursor_m.execute("""
            CREATE TABLE mensajes (
                user_id TEXT,
                emisor TEXT,
                mensaje TEXT,
                timestamp TEXT
            )
        """)
        conn_m.commit()
        conn_m.close()
        print("✅ Tabla 'mensajes' reseteada con: user_id, emisor, mensaje, timestamp")

        # 2. ARREGLAR TABLA DE DISLEXIA (Conocimiento)
        conn_c = sqlite3.connect("base_conocimiento.db")
        cursor_c = conn_c.cursor()
        
        # Borramos la versión vieja que buscaba 'contenido'
        cursor_c.execute("DROP TABLE IF EXISTS conocimiento_dislexia")
        
        cursor_c.execute("""
            CREATE TABLE conocimiento_dislexia (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                mensaje TEXT
            )
        """)
        conn_c.commit()
        conn_c.close()
        print("✅ Tabla 'conocimiento_dislexia' reseteada con: mensaje")

    except Exception as e:
        print(f"❌ Error crítico al resetear tablas: {e}")

# Llamamos a la limpieza
inicializar_tablas_fijas()

def reparacion_expres():
    # Asegúrate de que los nombres coincidan con los que usa tu App
    db_mensajes = "mensajes.db" 
    db_dislexia = "base_conocimiento.db"

    # 1. Reparar mensajes
    try:
        conn = sqlite3.connect(db_mensajes)
        cursor = conn.cursor()
        try:
            # Si esto falla, es que falta la columna 'emisor'
            cursor.execute("SELECT emisor FROM mensajes LIMIT 1")
        except:
            print("⚠️ Estructura vieja en mensajes.db. Limpiando...")
            cursor.execute("DROP TABLE IF EXISTS mensajes")
            cursor.execute("""CREATE TABLE mensajes (
                user_id TEXT, 
                emisor TEXT, 
                mensaje TEXT, 
                timestamp TEXT)""")
            conn.commit()
        conn.close()
    except Exception as e:
        print(f"Error reparando mensajes: {e}")

    # 2. Reparar dislexia
    try:
        conn = sqlite3.connect(db_dislexia)
        cursor = conn.cursor()
        try:
            # Si esto falla, es que falta la columna 'mensaje'
            cursor.execute("SELECT mensaje FROM conocimiento_dislexia LIMIT 1")
        except:
            print("⚠️ Estructura vieja en dislexia. Reconstruyendo...")
            cursor.execute("DROP TABLE IF EXISTS conocimiento_dislexia")
            cursor.execute("""CREATE TABLE conocimiento_dislexia (
                id INTEGER PRIMARY KEY AUTOINCREMENT, 
                mensaje TEXT)""")
            conn.commit()
        conn.close()
    except Exception as e:
        print(f"Error reparando dislexia: {e}")

# -------------------------------
# 🧠 Crear TODAS las tablas
# -------------------------------
def iniciar_base_datos():
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()

    # 🔹 mensajes
    cursor.execute("""
    CREATE TABLE IF NOT EXISTS mensajes (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        usuario TEXT,
        mensaje TEXT,
        timestamp TEXT
    )
    """)

    # 🔹 memoria
    cursor.execute("""
    CREATE TABLE IF NOT EXISTS memoria (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        texto TEXT
    )
    """)

    # 🔹 conocimiento dislexia
    cursor.execute("""
    CREATE TABLE IF NOT EXISTS conocimiento_dislexia (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        contenido TEXT
    )
    """)

    # -------------------------------
    # 📚 DATOS (NO CAMBIADOS)
    # -------------------------------
   
    info_dislexia = [
        "La dislexia es una dificultad de aprendizaje que puede afectar la lectura, la escritura y la ortografía.",
        "La dislexia también puede estar presente en la vida adulta.",
        "La dislexia no está relacionada con la inteligencia.",
        "Muchas personas con dislexia tienen inteligencia normal o superior.",
        "La dislexia está relacionada con la forma en que el cerebro procesa el lenguaje escrito.",

        "En adultos, la dislexia puede hacer que la lectura sea más lenta.",
        "Algunas personas con dislexia necesitan más tiempo para comprender textos largos.",
        "La ortografía puede seguir siendo difícil en la vida adulta.",
        "Es común releer frases varias veces para entender el contenido.",
        "Las palabras largas o poco comunes pueden ser más difíciles de reconocer.",

        "Algunos adultos con dislexia evitan leer textos largos.",
        "Leer correos electrónicos o documentos extensos puede requerir más esfuerzo.",
        "La escritura de informes o mensajes formales puede ser desafiante.",
        "Algunas personas revisan varias veces lo que escriben para evitar errores.",

        "La tecnología puede ayudar a compensar algunas dificultades.",
        "Los lectores de texto en voz alta pueden facilitar la comprensión.",
        "Las aplicaciones de dictado por voz pueden ayudar a escribir.",
        "Los correctores ortográficos pueden ser herramientas útiles.",

        "Usar textos con tipografías claras puede facilitar la lectura.",
        "Aumentar el tamaño del texto puede mejorar la comodidad visual.",
        "Leer en pantallas con buen contraste puede ayudar a algunas personas.",

        "Dividir textos largos en partes pequeñas puede facilitar la comprensión.",
        "Tomar pausas durante la lectura puede ayudar a mantener la concentración.",
        "Leer en un ambiente tranquilo puede mejorar la atención.",

        "Muchos adultos con dislexia desarrollan estrategias propias para aprender.",
        "La organización y planificación pueden ayudar a manejar tareas de lectura.",
        "Tomar notas simples puede ayudar a recordar información importante.",

        "Muchas personas con dislexia desarrollan habilidades fuertes en creatividad.",
        "El pensamiento visual puede ser una fortaleza en algunas personas.",
        "La resolución de problemas puede ser una habilidad destacada.",

        "La dislexia no impide tener éxito profesional.",
        "Muchas personas con dislexia trabajan en áreas creativas, técnicas o empresariales.",
        "Con herramientas adecuadas, muchas dificultades pueden reducirse.",

        "La práctica constante puede mejorar la confianza al leer.",
        "Aprender nuevas estrategias puede ayudar a manejar la dislexia en la vida diaria.",
        "Buscar apoyo educativo o profesional puede ser útil en algunos casos."
    ]

    # Insertar solo si está vacía
    cursor.execute("SELECT COUNT(*) FROM conocimiento_dislexia")
    cantidad = cursor.fetchone()[0]
     
    # ✅ Generar texto seguro para móviles
    info_dislexia_texto = "\n".join(
    random.sample(info_dislexia, min(2, len(info_dislexia)))
)

    if cantidad == 0:
        for d in info_dislexia:
            cursor.execute(
                "INSERT INTO conocimiento_dislexia (contenido) VALUES (?)",
                (d,)
            )

    conn.commit()
    conn.close()

    print("✅ DB lista en:", DB_PATH)


# -------------------------------
# ✏️ Actualizar mensajes
# -------------------------------
def actualizar_mensaje_db(msg_id, **kwargs):
    if not kwargs:
        return

    columnas = ", ".join([f"{k} = ?" for k in kwargs.keys()])
    valores = list(kwargs.values())
    valores.append(msg_id)

    query = f"UPDATE mensajes SET {columnas} WHERE id = ?"
    ejecutar_query(query, tuple(valores))


