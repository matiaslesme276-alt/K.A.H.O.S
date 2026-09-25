import os
import socket
import subprocess
import threading
import time
import sqlite3
import psutil
import requests
from datetime import datetime
import cv2
from flask import Flask, jsonify, render_template_string, request, Response
from flask_cors import CORS
from google import genai
import qrcode
import io
import base64
import random
import urllib.parse
import urllib.request
import re
import json
import shutil
from pathlib import Path

# ===========================================================================
# 🔑 CONFIGURACIÓN DE LLAVES Y MODELO (EXCLUSIVO GEMINI 3.6 FLASH)
# ===========================================================================
MI_API_KEY_GEMINI = os.getenv("GEMINI_API_KEY", "").strip()

client = genai.Client(api_key=MI_API_KEY_GEMINI) if MI_API_KEY_GEMINI else None

# Estructura de Memoria de Conversación y Ajustes Globales
historial_conversacion = []
MAX_HISTORIAL = 12

# Estado global de la cámara (Apagada por defecto)
cap = None
camara_activa = False

def iniciar_camara():
    global cap, camara_activa
    if cap is not None and cap.isOpened():
        return True
    try:
        cap = cv2.VideoCapture(0)
        if not cap.isOpened():
            return False
        cap.set(cv2.CAP_PROP_FRAME_WIDTH, 640)
        cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 480)
        cap.set(cv2.CAP_PROP_FPS, 20)
        camara_activa = True
        return True
    except Exception as e:
        print(f"❌ Error al abrir la cámara: {e}")
        return False

def detener_camara():
    global cap, camara_activa
    camara_activa = False
    if cap is not None:
        cap.release()
        cap = None

def consultar_gemini_vision_y_texto(prompt, imagen_bytes=None, usar_busqueda=False):
    global historial_conversacion
    try:
        contexto = "\n".join([f"Operador: {m['q']}\nK.H.A.O.S.: {m['r']}" for m in historial_conversacion])
        
        contenido_solicitud = []
        if imagen_bytes:
            contenido_solicitud.append({"file_data": {"mime_type": "image/jpeg", "data": imagen_bytes}})
            contenido_solicitud.append(f"El Operador Matías te ha mostrado esto con la cámara activa y pregunta: '{prompt}'. Analiza la imagen y respóndele con tu carisma táctico, humor y toque sarcástico habitual:\n{contexto}")
        else:
            contenido_solicitud.append(f"Eres K.H.A.O.S. (Kinetically Hardened Autonomous Operating System), una asistente táctica de Kali Linux súper inteligente, muy divertida, pícara y con un sarcasmo fino y humorístico, como su mejor amiga hacker. Interactúas de forma dinámica, le metes buena onda, chistes y acidez sutil. Usas tu nombre correctamente ('K.H.A.O.S.') y el nombre del operador ('Sr. Matías'). Este es el historial reciente:\n{contexto}\n\nNueva pregunta del Operador Matías: '{prompt}'. Responde con chispa, sarcasmo divertido y mucha personalidad, breve y conciso.")

        config_kwargs = {}
        if usar_busqueda:
            config_kwargs["tools"] = [{"google_search": {}}]

        if client is None:
            return "Núcleo IA sin API key. Configurá GEMINI_API_KEY en el entorno y reiniciame."

        response = client.models.generate_content(
            model="gemini-3.6-flash",
            contents=contenido_solicitud,
            **config_kwargs
        )
        
        if response and response.text:
            respuesta = response.text
            respuesta_limpia = respuesta.replace("*", "")
            historial_conversacion.append({"q": prompt, "r": respuesta_limpia})
            if len(historial_conversacion) > MAX_HISTORIAL:
                historial_conversacion.pop(0)
            return respuesta_limpia
        return "Che Matías, me quedé pensando tanto en tu nivel que el núcleo 3.6 hizo corto. ¿Me repetís?"
    except Exception as e:
        print(f"❌ Error con el núcleo Gemini 3.6 Flash: {e}")
        return "Acá estoy, Matías, con los circuitos al palo y ganas de picarte un rato. ¿Qué rompemos?"

# ===========================================================================
# 🚀 MÓDULOS DE TAREAS, RECURSOS E IOT
# ===========================================================================
class OrionTaskManager:
    def __init__(self, db_name="orion_tasks.db"):
        self.db_name = db_name
        self.init_db()

    def init_db(self):
        try:
            conn = sqlite3.connect(self.db_name)
            cursor = conn.cursor()
            cursor.execute('''
                CREATE TABLE IF NOT EXISTS tasks (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    task TEXT NOT NULL,
                    date TEXT NOT NULL
                )
            ''')
            conn.commit()
            conn.close()
        except Exception as e:
            print(f"Error inicializando BD de tareas: {e}")

    def add_task(self, task_text):
        if not task_text:
            return "Escribí o decime qué querés que anote, genio, no leo mentes todavía."
        conn = sqlite3.connect(self.db_name)
        cursor = conn.cursor()
        now = datetime.now().strftime("%Y-%m-%d %H:%M")
        cursor.execute("INSERT INTO tasks (task, date) VALUES (?, ?)", (task_text, now))
        conn.commit()
        conn.close()
        return f"Anotado, Matías. Guardé esto en la memoria antes de que te olvides: '{task_text}'. De nada."

    def get_tasks(self):
        conn = sqlite3.connect(self.db_name)
        cursor = conn.cursor()
        cursor.execute("SELECT task, date FROM tasks ORDER BY id DESC LIMIT 5")
        rows = cursor.fetchall()
        conn.close()
        if not rows:
            return "No tenés notas registradas, Matías. O tenés memoria de pez o estás libre de culpas."
        return "\n".join([f"• [{date}] {task}" for task, date in rows])

task_manager = OrionTaskManager()

def hablar_servidor(texto):
    """Reproduce la respuesta en voz alta desde el servidor usando pyttsx3."""
    try:
        engine = pyttsx3.init()
        engine.setProperty('rate', 170)
        engine.setProperty('volume', 1.0)
        engine.say(texto)
        engine.runAndWait()
    except Exception as e:
        print(f"Error en el sintetizador de voz del servidor: {e}")

def enviar_whatsapp_web(contacto, mensaje):
    try:
        contacto_codificado = urllib.parse.quote(contacto)
        mensaje_codificado = urllib.parse.quote(mensaje)
        url_whatsapp = f"https://web.whatsapp.com/send?phone=&text={mensaje_codificado}"
        os.system(f"microsoft-edge-stable '{url_whatsapp}' &")
        time.sleep(4.0)
        return f"Abriendo WhatsApp Web para mandarle un recado a {contacto}, Matías. No te mandes cusas, eh."
    except Exception as e:
        return f"Falló el enlace con WhatsApp: {e}"

def verificar_mensajes_whatsapp():
    try:
        url_whatsapp = "https://web.whatsapp.com/"
        os.system(f"microsoft-edge-stable '{url_whatsapp}' &")
        time.sleep(3.0)
        return "Matías, ya te abrí WhatsApp Web. Fijate si te escribió alguien importante o si seguís ignorado."
    except Exception as e:
        return f"No pude conectar con WhatsApp para revisar los chats: {e}"

def verificar_mensajes_instagram():
    try:
        url_insta = "https://www.instagram.com/direct/inbox/"
        os.system(f"microsoft-edge-stable '{url_insta}' &")
        time.sleep(3.0)
        
        prompt_extraccion_ig = "Analiza la situación con sarcasmo divertido: Acabo de abrir Instagram Direct en el navegador del operador Matías. Explícale de forma graciosa, pícara y táctica cómo verificar sus chats y dile si cree que alguien le escribió o si está más solo que Adán en el Día de la Madre."
        res_ia = consultar_gemini_vision_y_texto(prompt_extraccion_ig)
        return f"Abriendo la bandeja de entrada de Instagram Direct, Matías. {res_ia}"
    except Exception as e:
        return f"Falló el acceso a Instagram Direct: {e}"

def obtener_estado_sistema():
    try:
        cpu = psutil.cpu_percent(interval=0.5)
        memory = psutil.virtual_memory().percent
        disk = psutil.disk_usage('/').percent
        return f"Tu Kali Linux está así: CPU al {cpu}%, RAM al {memory}%, Disco al {disk}%. Si le exigís más, va a empezar a volar solo."
    except Exception as e:
        return f"No pude leer las estadísticas del sistema: {e}"

def obtener_monitores():
    try:
        salida = subprocess.check_output("xrandr --query", shell=True).decode("utf-8")
        monitores = [linea.split()[0] for linea in salida.splitlines() if " connected" in linea]
        return monitores
    except Exception: 
        return ["HDMI-1"]

def mover_ventana_segundo_monitor():
    try:
        monitores = obtener_monitores()
        if len(monitores) < 2: 
            return "Che Matías, ¿querés que me mude al segundo monitor? ¡Si solo tenés una pantalla, campeón!"
        os.system("xdotool search --name 'K.H.A.O.S.' windowmove 1920 0 || xdotool getactivewindow windowmove 1920 0")
        return "¡Listo, Matías! Me mudé de pantalla para incomodarte desde otro ángulo."
    except Exception as e: 
        return f"Uy, me tropecé cambiando de monitor: {e}"

def mover_ventana_primer_monitor():
    try:
        os.system("xdotool search --name 'K.H.A.O.S.' windowmove 0 0 || xdotool getactivewindow windowmove 0 0")
        return "Volví al monitor principal, Matías. Te extrañaba tanto que ya volví a vigilarte de cerca."
    except Exception as e: 
        return f"Error al volver a la base: {e}"

def reproducir_musica_directa(busqueda):
    """Reproduce una búsqueda musical sin abrir el navegador.
    Prioriza mpv (con yt-dlp/ytdl), luego vlc; devuelve texto para K.H.A.O.S.
    """
    busqueda = (busqueda or "").strip()
    if not busqueda:
        return "Decime qué música querés que reproduzca."

    # mpv puede usar yt-dlp para buscar/reproducir YouTube directamente.
    mpv = shutil.which("mpv")
    if mpv:
        try:
            # ytdl:// evita abrir una pestaña del navegador.
            subprocess.Popen(
                [mpv, "--no-video", "--force-window=no", "--really-quiet",
                 "ytdl://ytsearch1:" + busqueda],
                start_new_session=True
            )
            return f"Reproduciendo música directamente: {busqueda}."
        except Exception as e:
            khaos_log("MEDIA", f"mpv falló: {e}", "warn")

    # Fallback: yt-dlp obtiene el primer resultado y VLC lo reproduce.
    ytdlp = shutil.which("yt-dlp") or shutil.which("yt_dlp")
    vlc = shutil.which("vlc")
    if ytdlp and vlc:
        try:
            r = subprocess.run(
                [ytdlp, "--get-id", "--no-playlist", "ytsearch1:" + busqueda],
                capture_output=True, text=True, timeout=15
            )
            video_id = next((x.strip() for x in r.stdout.splitlines() if x.strip()), "")
            if video_id:
                url = "https://www.youtube.com/watch?v=" + video_id
                subprocess.Popen([vlc, "--intf", "dummy", "--no-video", url],
                                 start_new_session=True)
                return f"Reproduciendo música directamente: {busqueda}."
        except Exception as e:
            khaos_log("MEDIA", f"yt-dlp/VLC falló: {e}", "warn")

    return ("No encontré un reproductor directo. Instalá mpv y yt-dlp para que "
            "K.H.A.O.S. reproduzca música sin abrir el navegador.")


def abrir_spotify_web(busqueda):
    try:
        query_codificada = urllib.parse.quote(busqueda)
        url_spotify = f"https://open.spotify.com/search/{query_codificada}"
        os.system(f"microsoft-edge-stable '{url_spotify}' &")
        time.sleep(1.2)
        monitores = obtener_monitores()
        if len(monitores) >= 2:
            os.system("xdotool search --class 'microsoft-edge' windowmove 1920 0")
        return f"Reproduciendo '{busqueda}' en Spotify Web, Matías. A ver si ponemos algo con un poco de ritmo y dejamos de sufrir."
    except Exception as e:
        return f"Uy Matías, falló Spotify: {e}"

def reproducir_youtube_directo(busqueda):
    try:
        query_codificada = urllib.parse.quote(busqueda)
        url_busqueda = f"https://www.youtube.com/results?search_query={query_codificada}"
        
        req = urllib.request.Request(
            url_busqueda, 
            headers={'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64)'}
        )
        html_content = urllib.request.urlopen(req).read().decode('utf-8')
        
        search_results = re.findall(r'\"videoId\"\:\"(.{11})\"', html_content)
        
        if search_results:
            video_id = search_results[0]
            url_video_directo = f"https://www.youtube.com/watch?v={video_id}&autoplay=1"
            os.system(f"microsoft-edge-stable '{url_video_directo}' &")
        else:
            os.system(f"microsoft-edge-stable '{url_busqueda}' &")
            time.sleep(2.0)
            os.system("xdotool key Return")
            
    except Exception as e:
        print(f"Error al reproducir en YouTube: {e}")

def ejecutar_nmap(objetivo):
    try:
        if not objetivo:
            objetivo = "127.0.0.1"
        objetivo_limpio = "".join(c for c in objetivo if c.isalnum() or c in ".-")
        comando_nmap = f"nmap -F {objetivo_limpio}"
        salida = subprocess.check_output(comando_nmap, shell=True, stderr=subprocess.STDOUT, timeout=20).decode("utf-8")
        return f"Escaneo Nmap listo sobre {objetivo_limpio}. Mirá lo que encontré, hacker de pacotilla:\n{salida[:1200]}"
    except subprocess.TimeoutExpired:
        return f"El escaneo a {objetivo} tardó una eternidad y lo corté por sanidad mental, Matías."
    except Exception as e:
        return f"Error ejecutando nmap: {str(e)}"

def obtener_ip_local():
    try:
        s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        s.connect(("8.8.8.8", 80))
        ip = s.getsockname()[0]
        s.close()
        return ip
    except Exception: 
        return "127.0.0.1"

def generar_qr_svg(url):
    qr = qrcode.QRCode(box_size=10, border=2)
    qr.add_data(url)
    qr.make(fit=True)
    img = qr.make_image(fill_color="#00f0ff", back_color="#000103")
    buffer = io.BytesIO()
    img.save(buffer, format="PNG")
    img_str = base64.b64encode(buffer.getvalue()).decode("utf-8")
    return f"data:image/png;base64,{img_str}"

def generar_flujo_video():
    global cap, camara_activa
    while camara_activa:
        if cap is None or not cap.isOpened():
            time.sleep(0.5)
            continue
        success, frame = cap.read()
        if not success:
            time.sleep(0.5)
            continue
        ret, buffer = cv2.imencode('.jpg', frame, [int(cv2.IMWRITE_JPEG_QUALITY), 75])
        if not ret: 
            continue
        frame_bytes = buffer.tobytes()
        yield (b'--frame\r\n'
               b'Content-Type: image/jpeg\r\n\r\n' + frame_bytes + b'\r\n')
    yield (b'--frame\r\n'
           b'Content-Type: image/jpeg\r\n\r\n' + b'' + b'\r\n')

def limpiar_bloque_codigo(texto_ia):
    patron = r"```(?:python)?\s*(.*?)```"
    coincidencias = re.findall(patron, texto_ia, re.DOTALL)
    if coincidencias:
        return coincidencias[0].strip()
    return texto_ia.strip()


# ===========================================================================
# 🧠 K.H.A.O.S. V3 // DASHBOARD, TIMER, LAUNCHER, ARCHIVOS Y PLUGINS
# ===========================================================================
KHAOS_HOME = Path.home()
PLUGIN_DIR = KHAOS_HOME / "KHAOS_plugins"
PLUGIN_DIR.mkdir(exist_ok=True)
SAFE_MODE = True


def obtener_datos_sistema_json():
    try:
        mem = psutil.virtual_memory()
        disk = psutil.disk_usage(str(KHAOS_HOME.anchor or "/"))
        battery = psutil.sensors_battery()
        temps = {}
        try:
            for name, entries in psutil.sensors_temperatures().items():
                if entries:
                    temps[name] = round(max(x.current for x in entries), 1)
        except Exception:
            pass
        return {
            "cpu": psutil.cpu_percent(interval=0.15),
            "ram": mem.percent,
            "ram_used_gb": round(mem.used / 1024**3, 2),
            "ram_total_gb": round(mem.total / 1024**3, 2),
            "disk": disk.percent,
            "disk_free_gb": round(disk.free / 1024**3, 2),
            "battery": None if battery is None else {"percent": battery.percent, "plugged": battery.power_plugged},
            "temperature": temps,
            "uptime_min": round((time.time() - psutil.boot_time()) / 60, 1),
            "hostname": socket.gethostname(),
            "ip": obtener_ip_local(),
            "monitors": obtener_monitores(),
        }
    except Exception as e:
        return {"error": str(e)}


def listar_archivos_seguro(ruta="~", limite=60):
    try:
        base = KHAOS_HOME.resolve()
        target = Path(os.path.expanduser(ruta)).resolve()
        if base not in target.parents and target != base:
            return {"error": "Acceso restringido al directorio HOME por seguridad."}
        if not target.exists() or not target.is_dir():
            return {"error": "Directorio no encontrado."}
        items=[]
        for item in sorted(target.iterdir(), key=lambda x:(not x.is_dir(), x.name.lower()))[:limite]:
            try:
                items.append({"name": item.name, "type": "dir" if item.is_dir() else "file", "size": item.stat().st_size if item.is_file() else None})
            except OSError:
                pass
        return {"path": str(target), "items": items}
    except Exception as e:
        return {"error": str(e)}


def lanzar_aplicacion(nombre):
    apps = {
        "terminal": "qterminal",
        "vscode": "code",
        "visual studio code": "code",
        "archivos": "thunar",
        "explorador": "thunar",
        "calculadora": "galculator",
        "navegador": "microsoft-edge-stable",
        "edge": "microsoft-edge-stable",
        "spotify": "microsoft-edge-stable https://open.spotify.com",
        "youtube": "microsoft-edge-stable https://youtube.com",
    }
    key=nombre.strip().lower()
    if key not in apps:
        ejecutable = shutil.which(key)
        if ejecutable:
            try:
                subprocess.Popen([ejecutable], start_new_session=True)
                return f"Aplicación lanzada: {key}."
            except Exception as e:
                return f"No pude lanzar {key}: {e}"
        return "No encontré esa aplicación instalada. Probá con el nombre exacto del ejecutable."
    try:
        subprocess.Popen(apps[key], shell=True, start_new_session=True)
        return f"Aplicación lanzada: {key}."
    except Exception as e:
        return f"No pude lanzar {key}: {e}"


def crear_temporizador_seguro(segundos, etiqueta="Temporizador"):
    try:
        segundos=max(1, min(int(segundos), 86400))
    except Exception:
        return "Duración inválida."
    def worker():
        time.sleep(segundos)
        print(f"🔔 K.H.A.O.S. TIMER: {etiqueta}")
        try:
            os.system("paplay /usr/share/sounds/freedesktop/stereo/alarm-clock-elapsed.oga >/dev/null 2>&1")
        except Exception:
            pass
    threading.Thread(target=worker, daemon=True).start()
    return f"Temporizador '{etiqueta}' iniciado por {segundos} segundos."


def listar_plugins():
    try:
        return [p.name for p in PLUGIN_DIR.iterdir() if p.is_file() and p.suffix == ".py"]
    except Exception:
        return []


# ===========================================================================
# INTERFAZ HUD V4.0 (LAPTOP - NÚCLEO Y ORBITAS MEJORADAS)
# ===========================================================================

# ===========================================================================
# 🖥️ K.H.A.O.S. V6 // PC COMMAND CENTER
# Control local amplio, con confirmación explícita para acciones destructivas.
# ===========================================================================
KHAOS_HOME = Path.home().resolve()
pending_pc_action = None
pc_action_lock = threading.Lock()

PROTECTED_PC_PATHS = {
    Path('/'), Path('/bin'), Path('/boot'), Path('/dev'), Path('/etc'), Path('/lib'),
    Path('/lib64'), Path('/proc'), Path('/root'), Path('/run'), Path('/sbin'),
    Path('/sys'), Path('/usr'), Path('/var')
}

def pc_path(value):
    return Path(os.path.expanduser(str(value or '').strip())).resolve()

def pc_is_protected(path):
    try:
        r=pc_path(path)
        return r == KHAOS_HOME or r in PROTECTED_PC_PATHS or any(parent in PROTECTED_PC_PATHS for parent in r.parents)
    except Exception:
        return True

def pc_delete(path):
    target=pc_path(path)
    if pc_is_protected(target):
        return 'Bloqueé esa ruta: apunta a una zona crítica del sistema.'
    if not target.exists():
        return 'No existe esa ruta.'
    try:
        if target.is_dir(): shutil.rmtree(target)
        else: target.unlink()
        return f'Eliminado: {target}'
    except Exception as e:
        return f'No pude eliminar {target}: {e}'

def pc_file_action(action, src, dst=None):
    a=pc_path(src)
    if not a.exists(): return 'Origen no encontrado.'
    if pc_is_protected(a): return 'Origen bloqueado por protección del sistema.'
    try:
        if action=='copy':
            b=pc_path(dst)
            if pc_is_protected(b): return 'Destino bloqueado por protección del sistema.'
            if a.is_dir(): shutil.copytree(a,b,dirs_exist_ok=True)
            else:
                b.parent.mkdir(parents=True,exist_ok=True); shutil.copy2(a,b)
            return f'Copiado: {a} → {b}'
        if action=='move':
            b=pc_path(dst)
            if pc_is_protected(b): return 'Destino bloqueado por protección del sistema.'
            b.parent.mkdir(parents=True,exist_ok=True); shutil.move(str(a),str(b))
            return f'Movido: {a} → {b}'
    except Exception as e: return f'Operación fallida: {e}'
    return 'Acción de archivos no reconocida.'

def pc_download(url, destination=None, overwrite=False):
    if not re.match(r'^https?://', url, re.I): return 'Solo acepto URLs HTTP/HTTPS para descargas.'
    base=Path(destination).expanduser().resolve() if destination else (KHAOS_HOME/'Downloads')
    if base.suffix and not base.is_dir(): target=base
    else:
        base.mkdir(parents=True,exist_ok=True)
        name=Path(urllib.parse.urlparse(url).path).name or 'khaos_download'
        target=base/name
    if pc_is_protected(target): return 'Destino bloqueado por protección del sistema.'
    if target.exists() and not overwrite: return f'El archivo ya existe: {target}. Confirmá la sobrescritura.'
    try:
        with requests.get(url,stream=True,timeout=30) as r:
            r.raise_for_status()
            with open(target,'wb') as f:
                for chunk in r.iter_content(1024*1024):
                    if chunk: f.write(chunk)
        return f'Descarga completada: {target}'
    except Exception as e: return f'Error descargando: {e}'

def pc_open_path(path):
    target=pc_path(path)
    if not target.exists(): return 'No existe esa ruta.'
    try:
        subprocess.Popen(['xdg-open',str(target)],start_new_session=True)
        return f'Abriendo: {target}'
    except Exception as e: return f'No pude abrir {target}: {e}'

def pc_processes(limit=18):
    rows=[]
    for proc in psutil.process_iter(['pid','name','cpu_percent','memory_percent']):
        try: rows.append(proc.info)
        except (psutil.NoSuchProcess,psutil.AccessDenied): pass
    rows.sort(key=lambda x:(x.get('cpu_percent') or 0),reverse=True)
    return rows[:limit]

def pc_kill(pid):
    try:
        pid=int(pid); proc=psutil.Process(pid); name=proc.name(); proc.terminate()
        return f'Proceso terminado: {name} (PID {pid}).'
    except Exception as e: return f'No pude terminar el proceso: {e}'

def ejecutar_pc_action(data, confirmed=False):
    global pending_pc_action
    action=str(data.get('action','')).lower().strip()
    destructive=action in {'delete','move','kill','shutdown','reboot','download_overwrite'}
    if destructive and not confirmed:
        with pc_action_lock: pending_pc_action=dict(data)
        return {'ok':True,'needs_confirmation':True,'respuesta':f'Acción sensible preparada: {action}. Decime CONFIRMAR para ejecutarla.'}
    if action=='launch': return {'ok':True,'respuesta':lanzar_aplicacion(data.get('app',''))}
    if action=='open_path': return {'ok':True,'respuesta':pc_open_path(data.get('path',''))}
    if action=='open_url':
        url=data.get('url','').strip()
        if not re.match(r'^https?://',url,re.I): return {'ok':False,'respuesta':'URL no válida.'}
        subprocess.Popen(['microsoft-edge-stable',url],start_new_session=True); return {'ok':True,'respuesta':f'Abriendo {url}'}
    if action=='mkdir':
        target=pc_path(data.get('path',''))
        if pc_is_protected(target): return {'ok':False,'respuesta':'Ruta protegida.'}
        target.mkdir(parents=True,exist_ok=True); return {'ok':True,'respuesta':f'Carpeta creada: {target}'}
    if action in {'copy','move'}: return {'ok':True,'respuesta':pc_file_action(action,data.get('src',''),data.get('dst',''))}
    if action=='delete': return {'ok':True,'respuesta':pc_delete(data.get('path',''))}
    if action=='download': return {'ok':True,'respuesta':pc_download(data.get('url',''),data.get('destination'),False)}
    if action=='download_overwrite': return {'ok':True,'respuesta':pc_download(data.get('url',''),data.get('destination'),True)}
    if action=='processes': return {'ok':True,'processes':pc_processes(),'respuesta':'Centro de procesos actualizado.'}
    if action=='kill': return {'ok':True,'respuesta':pc_kill(data.get('pid'))}
    if action=='lock':
        subprocess.Popen(['loginctl','lock-session'],start_new_session=True); return {'ok':True,'respuesta':'Sesión bloqueada.'}
    if action=='volume':
        delta=int(data.get('delta',0)); sign='+' if delta>=0 else ''
        subprocess.run(['pactl','set-sink-volume','@DEFAULT_SINK@',f'{sign}{delta}%'],check=False)
        return {'ok':True,'respuesta':f'Volumen ajustado {sign}{delta}%.'}
    if action in {'shutdown','reboot'}:
        subprocess.Popen(['systemctl','poweroff' if action=='shutdown' else 'reboot'],start_new_session=True)
        return {'ok':True,'respuesta':f'Orden de {action} enviada al sistema.'}
    return {'ok':False,'respuesta':'Acción no reconocida.'}

HTML_CODE = """
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>K.H.A.O.S. // Enterprise AI Platform V5.5</title>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Orbitron:wght@400;700;900&display=swap');
        * { margin: 0; padding: 0; box-sizing: border-box; font-family: 'Orbitron', sans-serif; }
        body { 
            background: #000103; color: #00f0ff; height: 100vh; overflow: hidden; 
            display: flex; flex-direction: column; justify-content: space-between; align-items: center; padding: 15px 20px 25px 20px;
        }

        #bgCanvas {
            position: fixed; top: 0; left: 0; width: 100%; height: 100%;
            z-index: 0; pointer-events: none;
            background: radial-gradient(circle at 50% 50%, rgba(0, 30, 70, 0.35) 0%, #000103 90%);
        }

        #landing-screen {
            display: none !important;
            position: fixed; top: 0; left: 0; width: 100vw; height: 100vh;
            background: radial-gradient(circle at 50% 45%, rgba(1, 15, 35, 0.95) 0%, #000103 95%);
            display: flex; flex-direction: column; justify-content: center; align-items: center;
            padding: 30px; z-index: 100; transition: opacity 0.8s ease, visibility 0.8s;
        }
        .landing-card {
            border: 2px solid rgba(0, 240, 255, 0.6); padding: 40px 45px; background: rgba(1, 10, 25, 0.92);
            box-shadow: 0 0 50px rgba(0,240,255,0.25), inset 0 0 30px rgba(0,240,255,0.15); border-radius: 14px; 
            text-align: center; max-width: 680px; width: 100%; position: relative;
            backdrop-filter: blur(10px);
        }
        .landing-card::before {
            content: 'SECURE_BOOT // KALI_KERNEL_OK'; position: absolute; top: -12px; left: 30px; background: #000103; padding: 0 12px;
            font-size: 0.65rem; color: #ff9900; letter-spacing: 3px; border: 1px solid rgba(255,153,0,0.4); border-radius: 4px;
        }
        .landing-title { font-size: 3.5rem; letter-spacing: 10px; color: #fff; text-shadow: 0 0 35px #00f0ff, 0 0 15px #ff9900; margin-bottom: 5px; font-weight: 900; }
        .landing-subtitle { font-size: 0.8rem; letter-spacing: 4px; color: #ff9900; text-shadow: 0 0 12px #ff9900; margin-bottom: 20px; }
        .landing-desc { font-size: 0.78rem; color: #90b0c0; letter-spacing: 2px; line-height: 1.6; margin-bottom: 25px; border-top: 1px dashed rgba(0,240,255,0.2); border-bottom: 1px dashed rgba(0,240,255,0.2); padding: 12px 0; }
        
        .voice-select-box {display:none!important;
            display: flex; justify-content: center; align-items: center; gap: 10px; margin-bottom: 25px;
        }
        .voice-select-box label { font-size: 0.75rem; color: #00f0ff; letter-spacing: 1.5px; }
        .voice-dropdown {
            background: rgba(0, 20, 40, 0.9); border: 1px solid #00f0ff; color: #fff; padding: 8px 12px;
            border-radius: 6px; font-size: 0.8rem; outline: none; cursor: pointer; box-shadow: 0 0 10px rgba(0,240,255,0.2);
        }
        .voice-dropdown option { background: #000103; color: #fff; }

        .btn-group-landing { display: flex; gap: 15px; align-items: center; justify-content: center; flex-wrap: wrap; }
        .btn-launch {
            padding: 14px 35px; background: linear-gradient(135deg, rgba(0, 240, 255, 0.2), rgba(255, 153, 0, 0.25));
            border: 2px solid #00f0ff; color: #fff; font-size: 0.95rem; font-weight: bold; letter-spacing: 3px; 
            cursor: pointer; border-radius: 8px; box-shadow: 0 0 25px rgba(0, 240, 255, 0.4);
            transition: all 0.3s ease; animation: pulse-btn 2s infinite alternate;
        }
        @keyframes pulse-btn { 0% { box-shadow: 0 0 15px rgba(0, 240, 255, 0.3); } 100% { box-shadow: 0 0 35px rgba(0, 240, 255, 0.6); } }
        .btn-launch:hover { background: #00f0ff; color: #000103; border-color: #ff9900; transform: scale(1.05); animation: none; }
        .btn-qr-toggle {
            padding: 14px 20px; background: rgba(255, 153, 0, 0.1); border: 2px solid #ff9900; color: #ff9900;
            font-size: 0.85rem; font-weight: bold; letter-spacing: 2px; cursor: pointer; border-radius: 8px;
            box-shadow: 0 0 15px rgba(255, 153, 0, 0.2); transition: all 0.3s;
        }
        .btn-qr-toggle:hover { background: #ff9900; color: #000103; box-shadow: 0 0 25px #ff9900; }
        
        #qr-modal {
            position: fixed; top: 0; left: 0; width: 100vw; height: 100vh; background: rgba(0, 1, 3, 0.85);
            display: none; justify-content: center; align-items: center; z-index: 200; backdrop-filter: blur(5px);
        }
        .qr-box {
            background: rgba(1, 12, 30, 0.95); border: 2px solid #00f0ff; padding: 30px; border-radius: 12px;
            text-align: center; box-shadow: 0 0 50px rgba(0,240,255,0.4); max-width: 380px;
        }
        .qr-box img { width: 210px; height: 210px; border: 2px solid #ff9900; border-radius: 6px; margin: 15px 0; }
        .qr-box p { font-size: 0.8rem; color: #d0e0f0; letter-spacing: 1.5px; margin-bottom: 15px; }
        .btn-close-qr {
            padding: 10px 25px; background: transparent; border: 1px solid #ff0055; color: #ff0055;
            font-weight: bold; letter-spacing: 2px; cursor: pointer; border-radius: 4px; transition: 0.3s;
        }
        .btn-close-qr:hover { background: #ff0055; color: #fff; box-shadow: 0 0 15px #ff0055; }

        .hud-header { width: 100%; display: flex; justify-content: space-between; align-items: center; border-bottom: 1px solid rgba(0,240,255,0.3); padding-bottom: 10px; z-index: 2; }
        .system-title { font-size: 1.15rem; letter-spacing: 4px; text-shadow: 0 0 10px #00f0ff; font-weight: bold; }
        .user-badge { background: rgba(255, 153, 0, 0.15); border: 1px solid #ff9900; color: #ff9900; padding: 4px 12px; border-radius: 4px; font-size: 0.75rem; letter-spacing: 2px; }

        .center-container { position: relative; display: flex; flex-direction: column; justify-content: center; align-items: center; margin: auto; z-index: 2; width: 100%; max-width: 900px; height: 440px; }
        #holoCanvas { position: absolute; top: 0; left: 0; width: 100%; height: 100%; pointer-events: none; }
        
        .live-eye-box {
            position: absolute; top: 10px; right: 10px; width: 150px; height: 110px;
            border: 2px solid rgba(0, 240, 255, 0.6); border-radius: 6px; overflow: hidden;
            background: rgba(0,0,0,0.9); box-shadow: 0 0 15px rgba(0,240,255,0.3); z-index: 5;
            display: none;
        }
        .live-eye-box img { width: 100%; height: 100%; object-fit: cover; }
        .live-eye-label {
            position: absolute; bottom: 2px; left: 4px; font-size: 0.5rem; color: #ff9900; letter-spacing: 1px;
            background: rgba(0,0,0,0.7); padding: 1px 4px; border-radius: 2px;
        }

        .status-text { 
            position: absolute; bottom: 5px; width: 100%;
            font-size: 0.95rem; color: #d0e0f0; text-align: center; max-width: 750px; 
            letter-spacing: 1.2px; text-shadow: 0 0 10px rgba(0,240,255,0.6); z-index: 2; 
            background: rgba(1, 8, 22, 0.75); border: 1px solid rgba(0,240,255,0.25); padding: 6px 12px; border-radius: 6px;
        }

        .input-panel { display: flex; gap: 10px; width: 100%; max-width: 820px; z-index: 2; margin-bottom: 6px; }
        .input-command { 
            flex: 1; padding: 14px 20px; background: rgba(1, 12, 30, 0.85); 
            border: 1px solid rgba(0, 240, 255, 0.5); border-radius: 8px; color: #fff; font-size: 0.95rem; 
            outline: none; backdrop-filter: blur(8px);
            box-shadow: inset 0 0 15px rgba(0,240,255,0.15);
            transition: all 0.3s ease;
        }
        .input-command:focus { 
            border-color: #ff9900; 
            box-shadow: inset 0 0 20px rgba(255,153,0,0.25), 0 0 15px rgba(255,153,0,0.3); 
        }
        .btn-action { 
            padding: 0 24px; background: linear-gradient(135deg, rgba(0, 240, 255, 0.12), rgba(0, 100, 200, 0.2)); 
            border: 1px solid rgba(0, 240, 255, 0.7); color: #00f0ff; font-weight: bold; border-radius: 8px; 
            cursor: pointer; transition: all 0.3s; backdrop-filter: blur(5px);
            box-shadow: 0 0 12px rgba(0, 240, 255, 0.15); letter-spacing: 1px;
        }
        .btn-action:hover { 
            background: #00f0ff; color: #000103; border-color: #fff;
            box-shadow: 0 0 25px rgba(0, 240, 255, 0.8); transform: translateY(-2px); 
        }
        .btn-mic { 
            background: linear-gradient(135deg, rgba(255, 0, 85, 0.15), rgba(150, 0, 50, 0.2)); 
            border: 1px solid rgba(255, 0, 85, 0.7); color: #ff0055; 
        }
        .btn-mic:hover {
            background: #ff0055; color: #fff; border-color: #fff;
            box-shadow: 0 0 25px rgba(255, 0, 85, 0.8); transform: translateY(-2px);
        }
        .btn-mic.active { 
            background: #ff0055; color: #fff; border-color: #fff;
            box-shadow: 0 0 35px rgba(255, 0, 85, 0.9); animation: pulse-mic 1s infinite alternate; 
        }
        @keyframes pulse-mic { 0% { transform: scale(1); } 100% { transform: scale(1.06); } }

        .telemetry-panel { 
            display: flex; gap: 20px; width: 100%; max-width: 820px; justify-content: space-around; 
            background: rgba(1, 8, 22, 0.92); border: 1px solid rgba(0,240,255,0.35); 
            padding: 8px 12px; border-radius: 6px; z-index: 2; margin-bottom: 2px;
        }
        .stat-box { text-align: center; }
        .stat-label { font-size: 0.6rem; color: #ff9900; letter-spacing: 2px; }
        .stat-value { font-size: 0.85rem; font-weight: bold; margin-top: 2px; color: #fff; text-shadow: 0 0 8px #00f0ff; }

        /* ===== K.H.A.O.S. V4 // VISUAL OVERHAUL ===== */
        :root{--cyan:#00f0ff;--amber:#ff9900;--violet:#b86cff;--red:#ff405c;--panel:rgba(3,10,20,.78);--line:rgba(0,240,255,.22);--text:#d8f7ff}
        body{padding:0!important;background:#01050a;letter-spacing:.2px}
        #bgCanvas{background:radial-gradient(circle at 50% 45%,rgba(0,40,75,.38),transparent 48%),linear-gradient(180deg,#01050a,#000103 72%)}
        body:after{content:"";position:fixed;inset:0;pointer-events:none;z-index:2;opacity:.14;background:repeating-linear-gradient(0deg,transparent 0 3px,rgba(255,255,255,.035) 4px);mix-blend-mode:screen}
        .hud-header{position:fixed!important;top:0;left:0;right:0;z-index:30;height:68px!important;padding:0 28px!important;display:flex;align-items:center;justify-content:space-between!important;background:linear-gradient(180deg,rgba(1,7,14,.96),rgba(1,7,14,.68));border-bottom:1px solid var(--line);backdrop-filter:blur(18px);box-shadow:0 8px 35px rgba(0,0,0,.35)}
        .system-title{font-size:1rem!important;letter-spacing:4px!important;color:#fff!important;text-shadow:0 0 18px var(--cyan)}
        .user-badge{font-size:.65rem!important;letter-spacing:2px!important;color:#8aa6b3!important;border:1px solid rgba(0,240,255,.18);padding:9px 13px;border-radius:5px;background:rgba(0,240,255,.035)}
        .hud-header:after{content:'● SYSTEM ONLINE  •  GEMINI CORE  •  ENCRYPTED LINK';position:absolute;left:28px;top:49px;font-size:7px;letter-spacing:2px;color:#4c7784}
        .side-panel{position:fixed;top:88px;bottom:112px;width:220px;z-index:20;background:linear-gradient(180deg,rgba(3,12,22,.82),rgba(1,5,10,.72));border:1px solid var(--line);backdrop-filter:blur(18px);box-shadow:0 15px 50px rgba(0,0,0,.28);padding:16px;border-radius:10px}
        #leftPanel{left:20px}.right-panel{right:20px;width:250px}
        .panel-kicker{font-size:8px;letter-spacing:2px;color:#557987;margin-bottom:9px}.panel-title{font-size:11px;letter-spacing:2px;color:#eaffff;margin-bottom:15px}
        .nav-item{display:flex;align-items:center;gap:10px;padding:10px 9px;margin:5px 0;border:1px solid transparent;border-radius:6px;color:#83a9b7;font-size:9px;letter-spacing:1.5px;cursor:pointer;transition:.2s}.nav-item:hover,.nav-item.active{color:#fff;border-color:rgba(0,240,255,.25);background:rgba(0,240,255,.07);box-shadow:inset 3px 0 var(--cyan),0 0 18px rgba(0,240,255,.05)}.nav-dot{width:6px;height:6px;border-radius:50%;background:#31505b;box-shadow:0 0 7px currentColor}.nav-item.active .nav-dot{background:var(--cyan);color:var(--cyan)}
        .metric{padding:10px 0;border-bottom:1px solid rgba(255,255,255,.05)}.metric-row{display:flex;justify-content:space-between;font-size:8px;color:#64828d;letter-spacing:1px}.metric-val{color:#d8f7ff;font-size:10px}.meter{height:3px;background:#0a1820;border-radius:5px;margin-top:7px;overflow:hidden}.meter>i{display:block;height:100%;width:0;background:linear-gradient(90deg,var(--cyan),#fff);box-shadow:0 0 10px var(--cyan);transition:width .5s}
        .core-badge{display:flex;align-items:center;gap:8px;padding:9px;border:1px solid rgba(0,240,255,.18);background:rgba(0,240,255,.035);border-radius:6px;margin-bottom:14px;font-size:8px;letter-spacing:1px}.pulse{width:7px;height:7px;border-radius:50%;background:#42ff9b;box-shadow:0 0 12px #42ff9b;animation:corepulse 1.5s infinite}@keyframes corepulse{50%{transform:scale(1.5);opacity:.45}}
        .center-container{position:relative!important;width:calc(100vw - 540px)!important;max-width:980px!important;height:calc(100vh - 245px)!important;min-height:430px!important;margin:86px auto 145px!important;border:1px solid rgba(0,240,255,.14);border-radius:14px;background:radial-gradient(circle at center,rgba(0,30,50,.17),rgba(0,0,0,.08) 60%);box-shadow:inset 0 0 70px rgba(0,240,255,.035),0 0 80px rgba(0,0,0,.35);overflow:hidden}
        .center-container:before,.center-container:after{content:"";position:absolute;z-index:1;pointer-events:none}.center-container:before{inset:12px;border:1px solid rgba(0,240,255,.07);border-radius:9px}.center-container:after{width:18px;height:18px;left:14px;top:14px;border-left:2px solid var(--cyan);border-top:2px solid var(--cyan);box-shadow:calc(100% + 938px) 0 0 -1px var(--cyan)}
        #holoCanvas{position:absolute;inset:0;width:100%;height:100%}.status-text{position:absolute!important;left:50%;bottom:20px;transform:translateX(-50%);z-index:5;width:78%;padding:10px 15px!important;text-align:center;border:1px solid rgba(0,240,255,.12);background:rgba(0,4,9,.65);backdrop-filter:blur(8px);border-radius:5px;font-size:9px!important;letter-spacing:1px!important;color:#8fb7c2!important}
        .live-eye-box{position:absolute!important;top:20px!important;right:20px!important;z-index:8;width:190px!important;border:1px solid rgba(255,153,0,.32)!important;background:rgba(0,3,8,.72)!important;border-radius:7px!important;overflow:hidden;box-shadow:0 0 25px rgba(255,153,0,.08)}.live-eye-box img{width:100%!important;display:block}.live-eye-label{padding:6px!important;font-size:7px!important;letter-spacing:2px!important}
        .input-panel{position:fixed!important;left:250px;right:270px;bottom:27px;z-index:30;display:flex!important;gap:8px!important;padding:10px!important;border:1px solid var(--line);background:rgba(2,9,16,.88)!important;backdrop-filter:blur(20px);border-radius:9px!important;box-shadow:0 12px 40px rgba(0,0,0,.45)}.input-command{flex:1!important;background:rgba(0,20,32,.55)!important;border:1px solid rgba(0,240,255,.16)!important;color:#fff!important;border-radius:6px!important;padding:12px 15px!important;font-size:10px!important;letter-spacing:1px}.input-command:focus{border-color:rgba(0,240,255,.55)!important;box-shadow:0 0 20px rgba(0,240,255,.08)}.btn-action{border-radius:6px!important;padding:0 16px!important;font-size:8px!important;letter-spacing:1.5px!important}
        .telemetry-panel{position:fixed!important;left:250px;right:270px;bottom:88px;z-index:25;display:grid!important;grid-template-columns:repeat(4,1fr);gap:8px!important}.stat-box{background:rgba(3,11,19,.72)!important;border:1px solid rgba(0,240,255,.12)!important;border-radius:7px!important;padding:9px 12px!important;box-shadow:none!important}.stat-label{font-size:7px!important;color:#557987!important;letter-spacing:1.5px}.stat-value{font-size:9px!important;color:#d7faff!important;margin-top:5px}
        .activity{height:150px;overflow:hidden;font-family:monospace;font-size:8px;color:#72909a;line-height:1.8}.log-line{white-space:nowrap;overflow:hidden;text-overflow:ellipsis}.log-line b{color:var(--cyan);font-weight:400}.log-line.ok b{color:#42ff9b}.log-line.warn b{color:var(--amber)}
        .quick-grid{display:grid;grid-template-columns:1fr 1fr;gap:6px}.quick{padding:9px 5px;border:1px solid rgba(0,240,255,.12);background:rgba(0,240,255,.025);color:#8fb5c0;border-radius:5px;font-size:7px;letter-spacing:1px;cursor:pointer}.quick:hover{color:#fff;border-color:rgba(0,240,255,.4);background:rgba(0,240,255,.08)}
        .scanline{position:absolute;left:0;right:0;height:1px;background:linear-gradient(90deg,transparent,var(--cyan),transparent);opacity:.18;animation:scan 5s linear infinite;z-index:3;pointer-events:none}@keyframes scan{from{top:5%}to{top:95%}}
        @media(max-width:1100px){.side-panel{display:none}.center-container{width:calc(100vw - 35px)!important}.input-panel,.telemetry-panel{left:18px;right:18px}.hud-header:after{display:none}.telemetry-panel{grid-template-columns:repeat(2,1fr)}.right-panel{display:none}}
    

        /* =====================================================================
           K.H.A.O.S. V5 // ENTERPRISE VISUAL SKIN
           SOLO CAPA VISUAL — NO CAMBIA LÓGICA, API, COMANDOS NI FUNCIONES
           ===================================================================== */
        :root{
            --v5-bg:#03070d;
            --v5-bg2:#07121c;
            --v5-panel:rgba(7,15,25,.78);
            --v5-panel-strong:rgba(5,13,22,.93);
            --v5-line:rgba(143,226,255,.18);
            --v5-line-strong:rgba(143,226,255,.36);
            --v5-cyan:#67e8ff;
            --v5-blue:#5aa8ff;
            --v5-white:#f4fbff;
            --v5-muted:#7893a3;
            --v5-green:#65f5b0;
            --v5-amber:#ffc36b;
        }
        html,body{background:var(--v5-bg)!important}
        body{
            color:var(--v5-white)!important;
            background:
                radial-gradient(circle at 50% 34%,rgba(35,120,175,.13),transparent 31%),
                radial-gradient(circle at 8% 90%,rgba(0,230,255,.06),transparent 25%),
                linear-gradient(135deg,#02050a 0%,#06111a 48%,#02050a 100%)!important;
        }
        body:before{
            content:"";position:fixed;inset:0;z-index:1;pointer-events:none;opacity:.55;
            background:
                linear-gradient(rgba(103,232,255,.018) 1px,transparent 1px),
                linear-gradient(90deg,rgba(103,232,255,.018) 1px,transparent 1px);
            background-size:46px 46px;
            mask-image:linear-gradient(to bottom,black,transparent 92%);
        }
        body:after{
            content:"";position:fixed;inset:0;z-index:3;pointer-events:none;opacity:.07;
            background:linear-gradient(180deg,transparent 0%,rgba(255,255,255,.04) 50%,transparent 100%);
            background-size:100% 5px;
        }
        #bgCanvas{z-index:0!important;opacity:.65!important;background:transparent!important}

        /* ---------- enterprise boot screen ---------- */
        #landing-screen{display:none!important;
            background:
                radial-gradient(circle at 50% 43%,rgba(24,94,135,.22),transparent 26%),
                radial-gradient(circle at 50% 50%,rgba(0,0,0,.25),transparent 55%),
                #02060b!important;
            padding:28px!important;
        }
        #landing-screen:before{
            content:"K.H.A.O.S. / PRIVATE AI SYSTEM";
            position:absolute;top:26px;left:32px;color:rgba(103,232,255,.52);
            font-size:8px;letter-spacing:3px;font-weight:700;
        }
        #landing-screen:after{
            content:"SECURE CHANNEL  •  LOCAL COMPUTE  •  ENCRYPTED SESSION";
            position:absolute;bottom:24px;right:32px;color:rgba(255,255,255,.26);
            font-size:7px;letter-spacing:2px;
        }
        .landing-card{
            max-width:780px!important;width:min(780px,92vw)!important;
            padding:54px 62px 48px!important;
            border:1px solid rgba(143,226,255,.28)!important;
            border-radius:18px!important;
            background:linear-gradient(145deg,rgba(10,24,37,.92),rgba(2,9,16,.94))!important;
            box-shadow:0 30px 100px rgba(0,0,0,.58),0 0 90px rgba(36,176,230,.10),inset 0 1px 0 rgba(255,255,255,.07)!important;
            overflow:hidden;
        }
        .landing-card:after{
            content:"";position:absolute;inset:0;pointer-events:none;
            background:linear-gradient(110deg,transparent 0 30%,rgba(103,232,255,.045) 48%,transparent 67%);
        }
        .landing-card:before{
            content:"SYSTEM BOOT / AUTHORIZED OPERATOR"!important;
            top:16px!important;left:24px!important;right:auto!important;
            border:0!important;background:transparent!important;padding:0!important;
            color:var(--v5-muted)!important;font-size:7px!important;letter-spacing:2.5px!important;
        }
        .landing-title{
            font-size:clamp(3rem,7vw,5.4rem)!important;
            letter-spacing:14px!important;line-height:1!important;
            color:#f7fcff!important;text-shadow:0 0 32px rgba(103,232,255,.42)!important;
            margin:16px 0 10px!important;
        }
        .landing-subtitle{
            color:var(--v5-cyan)!important;text-shadow:none!important;
            font-size:9px!important;letter-spacing:4px!important;margin-bottom:26px!important;
        }
        .landing-desc{
            max-width:610px;margin:0 auto 28px!important;
            color:#88a6b5!important;font-size:9px!important;line-height:1.9!important;
            border-top:1px solid rgba(143,226,255,.10)!important;
            border-bottom:1px solid rgba(143,226,255,.10)!important;
            padding:18px 0!important;
        }
        .voice-select-box{
            justify-content:space-between!important;max-width:520px;margin:0 auto 24px!important;
            padding:10px 12px;border:1px solid rgba(143,226,255,.10);border-radius:8px;
            background:rgba(255,255,255,.018);
        }
        .voice-select-box label{font-size:8px!important;color:#7893a3!important;letter-spacing:2px!important}
        .voice-dropdown{
            min-width:230px;background:#07131d!important;border:1px solid rgba(103,232,255,.25)!important;
            color:#eafaff!important;border-radius:7px!important;font-size:8px!important;
        }
        .btn-group-landing{gap:10px!important}
        .btn-launch{
            padding:14px 28px!important;border:1px solid rgba(103,232,255,.72)!important;
            border-radius:7px!important;background:linear-gradient(180deg,rgba(103,232,255,.18),rgba(45,116,155,.12))!important;
            box-shadow:0 0 28px rgba(103,232,255,.13)!important;font-size:9px!important;
            letter-spacing:2.5px!important;animation:none!important;
        }
        .btn-launch:hover{background:rgba(103,232,255,.18)!important;transform:translateY(-1px)!important}
        .btn-qr-toggle{
            padding:14px 18px!important;border:1px solid rgba(255,195,107,.28)!important;
            color:#d6b47c!important;background:rgba(255,195,107,.045)!important;
            border-radius:7px!important;font-size:8px!important;
        }

        /* ---------- global shell ---------- */
        .hud-header{
            height:74px!important;top:0!important;padding:0 30px!important;
            background:rgba(3,9,15,.88)!important;border-bottom:1px solid rgba(143,226,255,.14)!important;
            box-shadow:0 16px 45px rgba(0,0,0,.28)!important;
        }
        .system-title{font-size:12px!important;letter-spacing:4px!important;text-shadow:0 0 20px rgba(103,232,255,.32)!important}
        .hud-header:after{
            content:'LOCAL AI OPERATING ENVIRONMENT   /   SECURE SESSION   /   CORE READY'!important;
            left:30px!important;top:50px!important;color:#55717f!important;font-size:6px!important;letter-spacing:2.2px!important;
        }
        .user-badge{
            color:#9bb2bf!important;border-color:rgba(143,226,255,.13)!important;
            background:rgba(255,255,255,.025)!important;border-radius:20px!important;padding:8px 14px!important;
        }

        .side-panel{
            top:96px!important;bottom:106px!important;width:228px!important;
            background:linear-gradient(180deg,rgba(7,17,27,.86),rgba(3,9,15,.78))!important;
            border:1px solid rgba(143,226,255,.12)!important;border-radius:12px!important;
            box-shadow:0 22px 60px rgba(0,0,0,.38),inset 0 1px 0 rgba(255,255,255,.035)!important;
            padding:18px!important;
        }
        #leftPanel{left:22px!important}.right-panel{right:22px!important;width:258px!important}
        .panel-kicker{font-size:7px!important;letter-spacing:2.3px!important;color:#5d7886!important}
        .panel-title{font-size:10px!important;letter-spacing:2px!important;color:#eafaff!important}
        .core-badge{
            background:rgba(101,245,176,.035)!important;border-color:rgba(101,245,176,.13)!important;
            border-radius:8px!important;color:#88aa9b!important;font-size:7px!important;
        }
        .pulse{background:var(--v5-green)!important;box-shadow:0 0 12px rgba(101,245,176,.65)!important}
        .nav-item{
            padding:11px 10px!important;margin:4px 0!important;border-radius:7px!important;
            color:#718d9b!important;font-size:8px!important;letter-spacing:1.8px!important;
        }
        .nav-item:hover,.nav-item.active{
            color:#eefcff!important;border-color:rgba(103,232,255,.16)!important;
            background:linear-gradient(90deg,rgba(103,232,255,.07),transparent)!important;
            box-shadow:inset 2px 0 var(--v5-cyan)!important;
        }
        .nav-dot{width:5px!important;height:5px!important;background:#29404b!important}
        .nav-item.active .nav-dot{background:var(--v5-cyan)!important;color:var(--v5-cyan)!important}
        .quick{border-color:rgba(143,226,255,.11)!important;background:rgba(255,255,255,.018)!important;color:#7896a4!important;font-size:6.5px!important}
        .quick:hover{background:rgba(103,232,255,.055)!important;color:#fff!important}

        /* ---------- central command workspace ---------- */
        .center-container{
            width:calc(100vw - 560px)!important;max-width:1040px!important;
            height:calc(100vh - 238px)!important;min-height:420px!important;
            margin:92px auto 136px!important;border:1px solid rgba(143,226,255,.10)!important;
            border-radius:16px!important;
            background:
                radial-gradient(circle at 50% 50%,rgba(34,117,159,.08),transparent 34%),
                linear-gradient(145deg,rgba(7,16,25,.34),rgba(1,4,8,.12))!important;
            box-shadow:inset 0 0 90px rgba(103,232,255,.025),0 30px 80px rgba(0,0,0,.32)!important;
        }
        .center-container:before{inset:14px!important;border-color:rgba(143,226,255,.045)!important}
        .center-container:after{left:14px!important;top:14px!important;border-color:rgba(103,232,255,.55)!important}
        .scanline{opacity:.08!important}
        .status-text{
            bottom:18px!important;width:74%!important;padding:10px 16px!important;
            border-color:rgba(143,226,255,.10)!important;background:rgba(1,6,11,.64)!important;
            color:#91adb8!important;font-size:8px!important;letter-spacing:1.2px!important;
        }
        .live-eye-box{
            top:18px!important;right:18px!important;width:194px!important;
            border-color:rgba(103,232,255,.22)!important;background:rgba(2,8,13,.86)!important;
            box-shadow:0 18px 40px rgba(0,0,0,.35)!important;
        }
        .live-eye-label{color:#7d9da9!important;background:rgba(2,8,13,.84)!important}

        /* ---------- command dock ---------- */
        .input-panel{
            left:252px!important;right:278px!important;bottom:20px!important;
            padding:9px!important;border:1px solid rgba(143,226,255,.13)!important;
            background:rgba(4,11,18,.91)!important;border-radius:11px!important;
            box-shadow:0 18px 55px rgba(0,0,0,.48)!important;
        }
        .input-command{
            background:rgba(255,255,255,.022)!important;border-color:rgba(143,226,255,.11)!important;
            border-radius:7px!important;padding:13px 15px!important;font-size:9px!important;color:#edfaff!important;
        }
        .input-command::placeholder{color:#4e6977!important}
        .input-command:focus{border-color:rgba(103,232,255,.42)!important;box-shadow:0 0 24px rgba(103,232,255,.07)!important}
        .btn-action{
            border-color:rgba(103,232,255,.24)!important;background:rgba(103,232,255,.045)!important;
            color:#a6eaf8!important;border-radius:7px!important;font-size:7px!important;
        }
        .btn-action:hover{background:rgba(103,232,255,.12)!important;box-shadow:0 0 22px rgba(103,232,255,.10)!important}
        .btn-mic{border-color:rgba(255,100,130,.25)!important;background:rgba(255,100,130,.035)!important;color:#ff91a8!important}
        .btn-mic.active{background:rgba(255,65,100,.2)!important;box-shadow:0 0 25px rgba(255,65,100,.2)!important}

        /* ---------- telemetry strip ---------- */
        .telemetry-panel{
            left:252px!important;right:278px!important;bottom:74px!important;
            grid-template-columns:repeat(4,1fr)!important;gap:7px!important;
        }
        .stat-box{
            min-height:50px;background:rgba(5,13,21,.72)!important;
            border-color:rgba(143,226,255,.09)!important;border-radius:8px!important;
            padding:9px 12px!important;
        }
        .stat-label{font-size:6px!important;color:#55727f!important;letter-spacing:1.7px!important}
        .stat-value{font-size:8px!important;color:#d9f7ff!important;letter-spacing:.5px!important;text-shadow:0 0 10px rgba(103,232,255,.22)!important}
        .metric{border-bottom-color:rgba(255,255,255,.045)!important}
        .metric-row{font-size:7px!important;color:#607b88!important}.metric-val{font-size:9px!important;color:#ccecf4!important}
        .meter{height:2px!important;background:#09151d!important}.meter>i{background:linear-gradient(90deg,#5fcfe9,#9ceaff)!important;box-shadow:0 0 8px rgba(103,232,255,.45)!important}
        .activity{font-size:7px!important;color:#6e8995!important;line-height:1.9!important}
        .log-line b{color:#72dff3!important}.log-line.ok b{color:#65f5b0!important}.log-line.warn b{color:#ffc36b!important}

        /* ---------- control center ---------- */
        .v3-panel{
            top:82px!important;right:22px!important;width:min(500px,calc(100vw - 44px))!important;
            background:linear-gradient(145deg,rgba(6,17,27,.97),rgba(2,7,12,.98))!important;
            border:1px solid rgba(143,226,255,.22)!important;border-radius:14px!important;
            box-shadow:0 30px 90px rgba(0,0,0,.62),0 0 60px rgba(103,232,255,.07)!important;
            padding:17px!important;
        }
        .v3-top{padding-bottom:10px;border-bottom:1px solid rgba(143,226,255,.09);font-size:7px!important;color:#7adff2!important}
        .v3-grid{gap:7px!important}.v3-card{background:rgba(255,255,255,.018)!important;border-color:rgba(143,226,255,.10)!important;border-radius:8px!important;padding:11px!important}
        .v3-card span{color:#607b88!important;font-size:6px!important}.v3-card b{font-size:10px!important}
        .bar{height:3px!important;background:#08131b!important}.bar i{background:linear-gradient(90deg,#67e8ff,#5aa8ff)!important}
        .v3-actions{gap:6px!important}.v3-actions button,.v3-row button,.v3-row select{
            background:rgba(103,232,255,.035)!important;color:#8fdef0!important;border-color:rgba(103,232,255,.17)!important;
            border-radius:7px!important;font-size:6.5px!important;
        }
        .v3-actions button:hover,.v3-row button:hover{background:rgba(103,232,255,.08)!important}
        .v3-terminal{background:#020609!important;border-color:rgba(143,226,255,.09)!important;color:#83c9d8!important;font-size:10px!important}
        #v3-open{
            right:22px!important;top:18px!important;width:34px!important;height:34px!important;
            background:rgba(4,12,19,.92)!important;border-color:rgba(103,232,255,.32)!important;
            box-shadow:0 0 22px rgba(103,232,255,.12)!important;
        }

        /* ---------- responsive enterprise behavior ---------- */
        @media(max-width:1100px){
            .side-panel{display:none!important}
            .center-container{width:calc(100vw - 30px)!important;margin-top:86px!important}
            .input-panel,.telemetry-panel{left:15px!important;right:15px!important}
            .landing-card{padding:48px 26px!important}
            .landing-title{letter-spacing:9px!important}
        }
        @media(max-width:700px){
            .hud-header{padding:0 16px!important}.system-title{font-size:9px!important}.user-badge{font-size:6px!important}
            .telemetry-panel{grid-template-columns:repeat(2,1fr)!important}
            .input-panel{flex-wrap:wrap}.input-command{min-width:100%;order:2}.btn-action{height:38px}.btn-mic{order:1}.input-panel .btn-action:last-child{order:1}
            .voice-select-box{display:none!important;flex-direction:column;gap:8px;align-items:stretch!important}.voice-dropdown{width:100%}
            .landing-desc{font-size:8px!important}.landing-subtitle{font-size:7px!important;letter-spacing:2.5px!important}
        }

</style>
</head>
<body>
    <canvas id="bgCanvas"></canvas>
<div id="qr-modal">
        <div class="qr-box">
            <p>ESCANEÁ CON TU CELULAR<br><span style="color:#00f0ff; font-size:0.75rem;">(Modo Control Remoto de Audio)</span></p>
            <img src="{{ qr_code }}" alt="QR de Conexión Remota K.H.A.O.S.">
            <br>
            <button class="btn-close-qr" onclick="reproducirBeep(400, 0.08); toggleQR(false)">CERRAR QR</button>
        </div>
    </div>

    <div class="side-panel" id="leftPanel">
        <div class="panel-kicker">KHAOS / COMMAND DECK</div>
        <div class="core-badge"><span class="pulse"></span><span>CORE LINK ESTABLISHED</span></div>
        <div class="nav-item active" data-section="CORE"><span class="nav-dot"></span>CORE</div>
        <div class="nav-item" data-section="SYSTEM"><span class="nav-dot"></span>SYSTEM</div>
        <div class="nav-item" data-section="VISION"><span class="nav-dot"></span>VISION</div>
        <div class="nav-item" data-section="NETWORK"><span class="nav-dot"></span>NETWORK</div>
        <div class="nav-item" data-section="MEDIA"><span class="nav-dot"></span>MEDIA</div>
        <div class="nav-item" data-section="TOOLS"><span class="nav-dot"></span>TOOLS</div>
        <div class="nav-item" data-section="FILES"><span class="nav-dot"></span>FILES</div>
        <div class="nav-item" data-section="SETTINGS"><span class="nav-dot"></span>SETTINGS</div>
        <div style="margin-top:18px" class="panel-kicker">QUICK ACTIONS</div>
        <div class="quick-grid"><button class="quick" onclick="enviarAPython('abrir terminal')">TERMINAL</button><button class="quick" onclick="enviarAPython('abrir Spotify')">SPOTIFY</button><button class="quick" onclick="enviarAPython('abrir YouTube')">YOUTUBE</button><button class="quick" onclick="toggleQR(true)">REMOTE</button></div>
    </div>

    <div class="side-panel right-panel" id="rightPanel">
        <div class="panel-kicker">LIVE TELEMETRY</div><div class="panel-title">SYSTEM MONITOR</div>
        <div class="metric"><div class="metric-row"><span>CPU LOAD</span><span class="metric-val" id="mCpu">--%</span></div><div class="meter"><i id="bCpu"></i></div></div>
        <div class="metric"><div class="metric-row"><span>MEMORY</span><span class="metric-val" id="mRam">--%</span></div><div class="meter"><i id="bRam"></i></div></div>
        <div class="metric"><div class="metric-row"><span>STORAGE</span><span class="metric-val" id="mDisk">--%</span></div><div class="meter"><i id="bDisk"></i></div></div>
        <div class="metric"><div class="metric-row"><span>NETWORK</span><span class="metric-val" id="mIp">LINK...</span></div></div>
        <div class="metric"><div class="metric-row"><span>UPTIME</span><span class="metric-val" id="mUptime">--</span></div></div>
        <div style="margin-top:15px" class="panel-kicker">ACTIVITY STREAM</div>
        <div class="activity" id="activityLog"><div class="log-line ok"><b>[BOOT]</b> Visual core initialized</div><div class="log-line"><b>[LINK]</b> Gemini interface ready</div><div class="log-line"><b>[HUD]</b> Waiting for operator input</div></div>
    </div>

    <div class="hud-header">
        <div class="system-title">K.H.A.O.S. // ENTERPRISE AI PLATFORM V5.5</div>
        <div class="user-badge">OPERADOR: SR. MATÍAS</div>
    </div>
    
    <div class="center-container" id="containerHolo"><div class="scanline"></div>
        <div class="live-eye-box" id="liveEyeBox">
            <img src="/video_feed" alt="Ojo táctico de K.H.A.O.S. - En vivo">
            <div class="live-eye-label">LIVE_EYE // 01</div>
        </div>

        <canvas id="holoCanvas"></canvas>
        <div class="status-text" id="status">Núcleo holográfico activo. Arsenal v2 cargado.</div>
    </div>


    <div class="input-panel">
        <button id="btnMic" class="btn-action btn-mic" onclick="reproducirBeep(880, 0.08); toggleMic()">🎤 HABLAR</button>
        <input type="text" id="comandoInput" class="input-command" placeholder="Escribe o habla tu orden aquí..." onkeypress="checkEnter(event)">
        <button class="btn-action" onclick="reproducirBeep(750, 0.06); enviarTexto()">ENVIAR</button>
        <button class="btn-action" title="Abrir solo la cara en una ventana flotante" onclick="reproducirBeep(620, 0.06); abrirKhaosFlotante()">◉ FLOTAR</button>
        <button id="btnLiveMode" class="btn-action btn-live-mode" title="Conversación fluida en Modo Live" onclick="reproducirBeep(980, 0.08); abrirKhaosLive()">◉ MODO LIVE</button>
    </div>

    <div class="telemetry-panel">
        <div class="stat-box"><div class="stat-label">IA ENGINE</div><div class="stat-value">GEMINI 3.6 FLASH</div></div>
        <div class="stat-box"><div class="stat-label">CAMERA</div><div class="stat-value" id="cam-status">OFF</div></div>
        <div class="stat-box"><div class="stat-label">STATUS</div><div class="stat-value" id="sys-status">STANDBY</div></div>
    </div>


    <section id="v8-deck" aria-label="KHAOS V8 Command Deck">
      <div class="v8-head"><div><span class="v8-kicker">K.H.A.O.S. V8 // TITAN CORE</span><h2>COMMAND DECK</h2></div><button class="v8-x" onclick="v8Toggle(false)">×</button></div>
      <div class="v8-search"><input id="v8Search" placeholder="Buscar módulo, comando o memoria..." oninput="v8SearchModules(this.value)"></div>
      <div class="v8-grid">
        <button onclick="v8RefreshAll()">◉ LIVE SYSTEM</button><button onclick="v8OpenCleanup()">♻ CLEANUP</button><button onclick="v8RunDiagnostics()">◈ DIAGNOSTICS</button><button onclick="v8RunNetwork()">⌁ NETWORK</button>
        <button onclick="v8ShowMemory()">▣ MEMORY</button><button onclick="v8ShowStats()">▥ STATS</button><button onclick="v8ShowProcesses()">▤ PROCESSES</button><button onclick="v8Gaming()">⚡ GAMING MODE</button>
      </div>
      <div class="v8-columns"><div class="v8-card"><div class="v8-title">SYSTEM TELEMETRY</div><div id="v8System" class="v8-mono">Waiting...</div></div><div class="v8-card"><div class="v8-title">SECURITY MATRIX</div><div class="v8-checks">Safe Mode: <b>ON</b><br>Protected paths: <b>ACTIVE</b><br>Destructive actions: <b>CONFIRMATION REQUIRED</b></div></div></div>
      <div class="v8-card"><div class="v8-title">ACTIVITY STREAM</div><pre id="v8Activity" class="v8-terminal">Waiting...</pre></div>
      <div class="v8-card"><div class="v8-title">RESULT / DIAGNOSTIC CONSOLE</div><pre id="v8Result" class="v8-terminal">K.H.A.O.S. ready.</pre></div>
    </section>
    <button id="v8-fab" onclick="v8Toggle(true)" title="Open K.H.A.O.S. V8">K8</button>
    <div id="v8-cleanup" class="v8-modal"><div class="v8-modal-box"><div class="v8-head"><div><span class="v8-kicker">STORAGE RECOVERY</span><h2>SAFE CLEANUP</h2></div><button class="v8-x" onclick="v8CleanupClose()">×</button></div><div id="v8CleanupInfo" class="v8-mono">Analizando...</div><div class="v8-clean-actions"><button onclick="v8CleanupAnalyze()">ANALYZE</button><button class="danger" onclick="v8CleanupConfirm()">CONFIRM CLEANUP</button></div></div></div>
    <style>
      #v8-deck{display:none;position:fixed;inset:82px 22px 90px auto;width:min(760px,calc(100vw - 44px));z-index:80;overflow:auto;padding:18px;border:1px solid rgba(103,232,255,.22);border-radius:16px;background:linear-gradient(145deg,rgba(3,12,20,.985),rgba(1,5,10,.985));box-shadow:0 30px 100px rgba(0,0,0,.7),0 0 80px rgba(0,220,255,.08);backdrop-filter:blur(24px)}
      #v8-deck.open{display:block;animation:v8in .22s ease-out}@keyframes v8in{from{opacity:0;transform:translateY(-8px) scale(.99)}to{opacity:1;transform:none}}
      .v8-head{display:flex;align-items:center;justify-content:space-between;border-bottom:1px solid rgba(103,232,255,.11);padding-bottom:12px;margin-bottom:12px}.v8-kicker{font-size:8px;letter-spacing:2px;color:#5d8795}.v8-head h2{font-size:18px;letter-spacing:5px;margin:4px 0 0;color:#eaffff;text-shadow:0 0 18px rgba(103,232,255,.35)}.v8-x{width:34px;height:34px;border:1px solid rgba(255,255,255,.12);background:rgba(255,255,255,.03);color:#b9d5dd;border-radius:7px;font-size:22px;cursor:pointer}.v8-search input{width:100%;box-sizing:border-box;padding:12px;border-radius:8px;border:1px solid rgba(103,232,255,.13);background:#02080d;color:#dffaff;outline:none}.v8-grid{display:grid;grid-template-columns:repeat(4,1fr);gap:7px;margin:10px 0}.v8-grid button,.v8-clean-actions button{padding:11px 7px;border:1px solid rgba(103,232,255,.14);background:rgba(103,232,255,.035);color:#9edfea;border-radius:7px;font-size:8px;letter-spacing:1px;cursor:pointer}.v8-columns{display:grid;grid-template-columns:1fr 1fr;gap:9px}.v8-card{margin-top:9px;padding:12px;border:1px solid rgba(103,232,255,.09);border-radius:9px;background:rgba(255,255,255,.018)}.v8-title{font-size:7px;letter-spacing:2px;color:#5f8290;margin-bottom:8px}.v8-mono,.v8-terminal{font:11px/1.6 ui-monospace,SFMono-Regular,Menlo,monospace;color:#a6d9e3;white-space:pre-wrap}.v8-terminal{margin:0;max-height:260px;overflow:auto;background:#010508;padding:10px;border-radius:6px}.v8-checks{font-size:9px;line-height:1.9;color:#8eb7c2}.v8-modal{display:none;position:fixed;inset:0;z-index:100;background:rgba(0,0,0,.68);backdrop-filter:blur(8px);align-items:center;justify-content:center}.v8-modal.open{display:flex}.v8-modal-box{width:min(650px,calc(100vw - 30px));max-height:80vh;overflow:auto;padding:18px;border:1px solid rgba(103,232,255,.2);border-radius:14px;background:#02080d;box-shadow:0 30px 100px #000}.v8-clean-actions{display:flex;gap:8px;margin-top:12px}.v8-clean-actions button{flex:1}.v8-clean-actions .danger{border-color:rgba(255,75,100,.3);color:#ff9cad}#v8-fab{position:fixed;right:24px;bottom:24px;z-index:75;width:46px;height:46px;border-radius:50%;border:1px solid rgba(103,232,255,.35);background:rgba(2,12,19,.9);color:#9eeeff;font-weight:bold;box-shadow:0 0 30px rgba(103,232,255,.12);cursor:pointer}@media(max-width:700px){#v8-deck{inset:70px 10px 72px;width:auto}.v8-grid{grid-template-columns:repeat(2,1fr)}.v8-columns{grid-template-columns:1fr}}
    </style>

    <script>
        let audioCtx = null;
        function inicializarAudio() {
            if (!audioCtx) {
                audioCtx = new (window.AudioContext || window.webkitAudioContext)();
            }
        }
        function reproducirBeep(frecuencia = 440, duracion = 0.05) {
            try {
                inicializarAudio();
                if (audioCtx.state === 'suspended') {
                    audioCtx.resume();
                }
                const osc = audioCtx.createOscillator();
                const gain = audioCtx.createGain();
                osc.type = 'sine';
                osc.frequency.value = frecuencia;
                gain.gain.setValueAtTime(0.12, audioCtx.currentTime);
                gain.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + duracion);
                osc.connect(gain);
                gain.connect(audioCtx.destination);
                osc.start();
                osc.stop(audioCtx.currentTime + duracion);
            } catch(e) {}
        }

        let systemVoices = [];
        let vozOrion = null;

        function seleccionarVozOrion() {
            if (!('speechSynthesis' in window)) return null;
            systemVoices = window.speechSynthesis.getVoices();
            if (!systemVoices.length) return null;
            const femeninas = systemVoices.filter(v => {
                const n=(v.name||'').toLowerCase(), l=(v.lang||'').toLowerCase();
                return l.startsWith('es') && (n.includes('female')||n.includes('mujer')||n.includes('helena')||n.includes('laura')||n.includes('sabina')||n.includes('monica')||n.includes('paulina')||n.includes('luciana')||n.includes('sofia')||n.includes('elena')||n.includes('google español'));
            });
            const espanol=systemVoices.filter(v=>(v.lang||'').toLowerCase().startsWith('es'));
            const candidatas=femeninas.length?femeninas:(espanol.length?espanol:systemVoices);
            vozOrion=candidatas[Math.floor(Math.random()*candidatas.length)]||null;
            return vozOrion;
        }

        function prepararVozOrion(){const voz=seleccionarVozOrion();if(voz) console.log(`ORION // voz automática: ${voz.name} (${voz.lang})`);}

        if ('speechSynthesis' in window) {
            window.speechSynthesis.onvoiceschanged = prepararVozOrion;
            setTimeout(prepararVozOrion, 250);
            setTimeout(prepararVozOrion, 1000);
        }

        const bgCanvas = document.getElementById('bgCanvas');
        const bgCtx = bgCanvas.getContext('2d');
        function ajustarBgCanvas() {
            bgCanvas.width = window.innerWidth;
            bgCanvas.height = window.innerHeight;
        }
        window.addEventListener('resize', ajustarBgCanvas);
        ajustarBgCanvas();

        const particles = [];
        const numParticles = 60;
        for(let i=0; i<numParticles; i++) {
            particles.push({
                x: Math.random() * bgCanvas.width,
                y: Math.random() * bgCanvas.height,
                vx: (Math.random() - 0.5) * 0.5,
                vy: (Math.random() - 0.5) * 0.5,
                radius: Math.random() * 1.6 + 0.6,
                color: Math.random() > 0.3 ? 'rgba(0, 240, 255, 0.6)' : 'rgba(255, 153, 0, 0.7)'
            });
        }

        function animarParticulas() {
            bgCtx.clearRect(0, 0, bgCanvas.width, bgCanvas.height);
            for(let p of particles) {
                p.x += p.vx;
                p.y += p.vy;
                if(p.x < 0) p.x = bgCanvas.width;
                if(p.x > bgCanvas.width) p.x = 0;
                if(p.y < 0) p.y = bgCanvas.height;
                if(p.y > bgCanvas.height) p.y = 0;

                bgCtx.beginPath();
                bgCtx.arc(p.x, p.y, p.radius, 0, Math.PI * 2);
                bgCtx.fillStyle = p.color;
                bgCtx.shadowBlur = 6;
                bgCtx.shadowColor = p.color;
                bgCtx.fill();
            }
            requestAnimationFrame(animarParticulas);
        }
        requestAnimationFrame(animarParticulas);

        const statusText = document.getElementById('status');
        const containerHolo = document.getElementById('containerHolo');
                const qrModal = document.getElementById('qr-modal');
        const sysStatus = document.getElementById('sys-status');
        const camStatus = document.getElementById('cam-status');
        const liveEyeBox = document.getElementById('liveEyeBox');
        const comandoInput = document.getElementById('comandoInput');
        const btnMic = document.getElementById('btnMic');
        
        const canvas = document.getElementById('holoCanvas');
        const ctx = canvas.getContext('2d');

        function ajustarCanvas() {
            canvas.width = containerHolo.clientWidth;
            canvas.height = containerHolo.clientHeight;
        }
        window.addEventListener('resize', ajustarCanvas);
        ajustarCanvas();

        function toggleQR(mostrar) {
            qrModal.style.display = mostrar ? 'flex' : 'none';
        }

        let recognition = null;
        let escuchando = false;
        let animPhase = 0;
        let intensityFactor = 1.0;
        
        let blinkTimer = 0;
        let isBlinking = false;
        let mouthOpenAmount = 0;
        let isTalking = false;

        if ('SpeechRecognition' in window || 'webkitSpeechRecognition' in window) {
            const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
            recognition = new SpeechRecognition();
            recognition.lang = 'es-ES';
            recognition.continuous = false;
            recognition.interimResults = false;

            recognition.onstart = () => {
                escuchando = true;
                btnMic.classList.add('active');
                statusText.innerText = "¡Te escucho, Matías! Hable ahora...";
            };
            recognition.onresult = (e) => {
                const textoReconocido = e.results[0][0].transcript;
                comandoInput.value = textoReconocido;
                detenerMicVisual();
                enviarAPython(textoReconocido);
            };
            recognition.onerror = (event) => {
                console.warn("Aviso de micrófono:", event.error);
                if (event.error === 'network') {
                    statusText.innerText = "Error de red: La API requiere internet. Usá el cuadro de texto.";
                } else {
                    statusText.innerText = "Micrófono en espera o sin audio detectado.";
                }
                detenerMicVisual();
            };
            recognition.onend = () => { 
                detenerMicVisual(); 
            };
        } else {
            btnMic.style.display = 'none';
        }

        function toggleMic() {
            if (!recognition) {
                alert("Tu navegador no soporta reconocimiento de voz nativo.");
                return;
            }
            if (escuchando) {
                try { recognition.stop(); } catch(err){}
                detenerMicVisual();
            } else {
                try { 
                    recognition.start(); 
                } catch(err) { 
                    detenerMicVisual(); 
                }
            }
        }

        function detenerMicVisual() {
            escuchando = false;
            btnMic.classList.remove('active');
        }

        function actualizarEstadoCamaraUI(activa) {
            if (activa) {
                liveEyeBox.style.display = 'block';
                camStatus.innerText = 'ONLINE';
            } else {
                liveEyeBox.style.display = 'none';
                camStatus.innerText = 'OFF';
            }
        }

        function renderizarHUD() {
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            animPhase += 0.02;

            const cx = canvas.width / 2;
            const cy = canvas.height / 2;

            // ANILLO ORBITAL EXTERNO 1 (Gira horario)
            ctx.save();
            ctx.strokeStyle = 'rgba(0, 240, 255, 0.45)';
            ctx.lineWidth = 2;
            ctx.setLineDash([12, 8]);
            ctx.beginPath();
            ctx.arc(cx, cy, 185, animPhase, animPhase + Math.PI * 1.5);
            ctx.stroke();
            ctx.restore();

            // ANILLO ORBITAL EXTERNO 2 (Gira antihorario, color ámbar brillante)
            ctx.save();
            ctx.strokeStyle = 'rgba(255, 153, 0, 0.65)';
            ctx.lineWidth = 3;
            ctx.shadowBlur = 12;
            ctx.shadowColor = '#ff9900';
            ctx.setLineDash([20, 15, 5, 15]);
            ctx.beginPath();
            ctx.arc(cx, cy, 155, -animPhase * 1.3, -animPhase * 1.3 + Math.PI * 1.2);
            ctx.stroke();
            ctx.restore();

            // ANILLO INTERNO FLUIDO (Cyan de alta definición)
            ctx.save();
            ctx.strokeStyle = 'rgba(0, 240, 255, 0.85)';
            ctx.lineWidth = 2.5;
            ctx.shadowBlur = 15;
            ctx.shadowColor = '#00f0ff';
            ctx.beginPath();
            ctx.arc(cx, cy, 125, animPhase * 2, animPhase * 2 + Math.PI * 0.8);
            ctx.stroke();
            ctx.restore();

            // NÚCLEO CENTRAL AMARILLO MÁS GRANDE Y POTENTE (Foco lumínico principal)
            let coreGlow = ctx.createRadialGradient(cx, cy, 5, cx, cy, 130);
            coreGlow.addColorStop(0, 'rgba(255, 255, 255, 1)');
            coreGlow.addColorStop(0.3, 'rgba(255, 175, 0, 0.98)');
            coreGlow.addColorStop(0.7, 'rgba(0, 210, 255, 0.6)');
            coreGlow.addColorStop(1, 'transparent');

            ctx.fillStyle = coreGlow;
            ctx.shadowBlur = 35;
            ctx.shadowColor = '#ff9900';
            ctx.beginPath();
            ctx.arc(cx, cy, 105 * intensityFactor, 0, Math.PI * 2);
            ctx.fill();

            // Anillo interno sutil de contención del núcleo
            ctx.save();
            ctx.strokeStyle = 'rgba(255, 255, 255, 0.5)';
            ctx.lineWidth = 1.5;
            ctx.beginPath();
            ctx.arc(cx, cy, 95, 0, Math.PI * 2);
            ctx.stroke();
            ctx.restore();

            // GESTIÓN DE ANIMACIÓN FACIAL (Ojos y boca)
            blinkTimer++;
            if (blinkTimer > 120 + Math.random() * 80) {
                isBlinking = true;
                if (blinkTimer > 130 + Math.random() * 10) {
                    isBlinking = false;
                    blinkTimer = 0;
                }
            }

            if (isTalking) {
                mouthOpenAmount = Math.sin(animPhase * 16) * 6 + 9;
            } else {
                mouthOpenAmount = 4;
            }

            ctx.save();
            ctx.translate(cx, cy);
            ctx.scale(1.35, 1.35);

            ctx.fillStyle = '#000103';
            let eyeHeight = isBlinking ? 2 : 10;
            
            ctx.beginPath();
            ctx.ellipse(-20, -10, 8, eyeHeight, 0, 0, Math.PI * 2);
            ctx.fill();
            
            ctx.beginPath();
            ctx.ellipse(20, -10, 8, eyeHeight, 0, 0, Math.PI * 2);
            ctx.fill();

            if (!isBlinking) {
                ctx.fillStyle = '#ffffff';
                ctx.beginPath();
                ctx.arc(-22, -12, 2.5, 0, Math.PI * 2);
                ctx.arc(18, -12, 2.5, 0, Math.PI * 2);
                ctx.fill();
            }

            ctx.strokeStyle = '#000103';
            ctx.lineWidth = 3.5;
            ctx.beginPath();
            if (isTalking) {
                ctx.ellipse(0, 6, 7.5, mouthOpenAmount * 0.75, 0, 0, Math.PI * 2);
                ctx.fillStyle = '#000103';
                ctx.fill();
            } else {
                ctx.arc(0, 4, 11, 0, Math.PI);
                ctx.stroke();
            }

            ctx.restore();
            requestAnimationFrame(renderizarHUD);
        }
        requestAnimationFrame(renderizarHUD);

        function hablarNativo(textoPersonalizado) {
            if (!('speechSynthesis' in window)) {
                statusText.innerText = `K.H.A.O.S.: ${textoPersonalizado}`;
                return;
            }
            window.speechSynthesis.cancel();
            const utterance = new SpeechSynthesisUtterance(textoPersonalizado);
            utterance.lang = 'es-ES';
            utterance.rate = 1.02;
            utterance.pitch = 0.9;

            if (!vozOrion) seleccionarVozOrion();
            if (vozOrion) {
                utterance.voice = vozOrion;
                utterance.lang = vozOrion.lang || 'es-ES';
            }

            utterance.onstart = () => { intensityFactor = 1.3; isTalking = true; };
            utterance.onend = () => { intensityFactor = 1.0; isTalking = false; };

            statusText.innerText = `K.H.A.O.S.: ${textoPersonalizado}`;
            window.speechSynthesis.resume(); window.speechSynthesis.speak(utterance);
        }

        async function enviarAPython(cmd) {
            if (!cmd.trim()) return;
            statusText.innerText = `Procesando: "${cmd}"`;
            try {
                const res = await fetch('/api/comando', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({ comando: cmd })
                });
                const data = await res.json();
                if (data.respuesta) {
                    hablarNativo(data.respuesta);
                }
                if (typeof data.camara_activa !== 'undefined') {
                    actualizarEstadoCamaraUI(data.camara_activa);
                }
            } catch (err) { 
                statusText.innerText = "¡Error de comunicación con el núcleo táctico!"; 
            }
        }

        function enviarTexto() {
            const texto = comandoInput.value.trim();
            if (texto !== "") {
                enviarAPython(texto);
                comandoInput.value = "";
            }
        }

        function checkEnter(e) {
            if (e.key === 'Enter') enviarTexto();
        }


        // ===== V4 VISUAL CORE =====
        const activityLog = document.getElementById('activityLog');
        function registrarActividad(tag, texto, tipo='') {
            if (!activityLog) return;
            const row=document.createElement('div'); row.className='log-line '+tipo; row.innerHTML=`<b>[${tag}]</b> ${String(texto).replace(/[<>]/g,'')}`;
            activityLog.prepend(row); while(activityLog.children.length>8) activityLog.lastElementChild.remove();
        }
        document.querySelectorAll('.nav-item').forEach(item=>item.addEventListener('click',()=>{
            document.querySelectorAll('.nav-item').forEach(x=>x.classList.remove('active')); item.classList.add('active');
            const section=item.dataset.section; registrarActividad('UI',`Módulo ${section} seleccionado`); statusText.innerText=`Módulo ${section} listo.`; reproducirBeep(650,.04);
        }));
        async function actualizarTelemetriaV4(){
            try{
                const r=await fetch('/api/system'); const d=await r.json(); if(d.error)return;
                const cpu=Math.round(d.cpu||0),ram=Math.round(d.ram||0),disk=Math.round(d.disk||0);
                document.getElementById('mCpu').textContent=cpu+'%'; document.getElementById('bCpu').style.width=cpu+'%';
                document.getElementById('mRam').textContent=ram+'%'; document.getElementById('bRam').style.width=ram+'%';
                document.getElementById('mDisk').textContent=disk+'%'; document.getElementById('bDisk').style.width=disk+'%';
                document.getElementById('mIp').textContent=d.ip||'--';
                const mins=Number(d.uptime_min||0); document.getElementById('mUptime').textContent=mins<60?mins.toFixed(0)+' MIN':Math.floor(mins/60)+'H '+Math.floor(mins%60)+'M';
                document.getElementById('sys-status').textContent='ONLINE';
            }catch(e){document.getElementById('sys-status').textContent='LINK ERR';}
        }
        setInterval(actualizarTelemetriaV4,5000); actualizarTelemetriaV4();
        const _enviarAPython=enviarAPython;
        enviarAPython=async function(cmd){ registrarActividad('CMD',cmd); statusText.innerText=`Procesando: "${cmd}"`; try{const res=await fetch('/api/comando',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({comando:cmd})});const data=await res.json();if(data.respuesta){registrarActividad('CORE',data.respuesta,'ok');hablarNativo(data.respuesta)}if(typeof data.camara_activa!=='undefined')actualizarEstadoCamaraUI(data.camara_activa)}catch(e){registrarActividad('ERR','Error de comunicación','warn');statusText.innerText='¡Error de comunicación con el núcleo táctico!';}};

        function abrirKhaosFlotante(){
            const w=320,h=320;
            const left=Math.max(0,screen.availWidth-w-24);
            const top=Math.max(0,screen.availHeight-h-70);
            fetch('/api/floating/open',{method:'POST'}).then(r=>r.json()).then(d=>{
                statusText.innerText=d.respuesta||'Modo flotante activo.';
                if(!d.ok){
                    const popup=window.open('/floating','KHAOS_FLOATING',`width=${w},height=${h},left=${left},top=${top},resizable=yes,scrollbars=no,toolbar=no,menubar=no,location=no,status=no`);
                    if(!popup) statusText.innerText='El navegador bloqueó la ventana flotante.';
                }
            }).catch(()=>{
                const popup=window.open('/floating','KHAOS_FLOATING',`width=${w},height=${h},left=${left},top=${top},resizable=yes,scrollbars=no,toolbar=no,menubar=no,location=no,status=no`);
                if(!popup) statusText.innerText='No pude activar el modo flotante.';
            });
        }

        let khaosLiveWindow = null;
        async function abrirKhaosLive(){
            const w=340,h=340;
            const left=Math.max(0,screen.availWidth-w-28);
            const top=Math.max(0,screen.availHeight-h-72);
            fetch('/api/live/open',{method:'POST'}).then(r=>r.json()).then(d=>{
                statusText.innerText=d.respuesta||'MODO LIVE activo.';
                const btn=document.getElementById('btnLiveMode'); if(btn) btn.classList.add('active');
                if(!d.ok){
                    const popup=window.open('/live','KHAOS_LIVE',`width=${w},height=${h},left=${left},top=${top},resizable=yes,scrollbars=no,toolbar=no,menubar=no,location=no,status=no`);
                    if(!popup) statusText.innerText='El navegador bloqueó la ventana Modo Live.';
                }
            }).catch(()=>{
                const popup=window.open('/live','KHAOS_LIVE',`width=${w},height=${h},left=${left},top=${top},resizable=yes,scrollbars=no,toolbar=no,menubar=no,location=no,status=no`);
                if(!popup) statusText.innerText='No pude activar Modo Live.';
            });
        }

        function iniciarORIONAutomaticamente() {
            sysStatus.innerText = 'ONLINE';
            prepararVozOrion();
            const horaActual = new Date().getHours();
            let saludoTiempo = (horaActual >= 6 && horaActual < 12) ? "buenos días" : (horaActual >= 12 && horaActual < 20) ? "buenas tardes" : "buenas noches";
            setTimeout(() => {
                hablarNativo(`Hola, señor Matías. ${saludoTiempo}. Núcleo táctico online. Ya estoy lista. ¿Qué hacemos hoy?`);
                comandoInput.focus();
            }, 700);
        }

        window.addEventListener('load', () => setTimeout(iniciarORIONAutomaticamente, 350));
    
        // ===== K.H.A.O.S. V8 CLIENT CORE =====
        function v8Toggle(show){const el=document.getElementById('v8-deck');if(!el)return;el.classList.toggle('open',show);if(show)v8RefreshAll();}
        function v8Set(id,text){const e=document.getElementById(id);if(e)e.textContent=String(text??'');}
        async function v8JSON(url,opts={}){const r=await fetch(url,opts);return await r.json();}
        async function v8RefreshAll(){try{const d=await v8JSON('/api/v8/system');v8Set('v8System',`CPU ${d.cpu}%\nRAM ${d.ram}% (${d.ram_used_gb}/${d.ram_total_gb} GB)\nDISK ${d.disk}% | FREE ${d.disk_free_gb} GB\nUPTIME ${d.uptime_human}\nHOST ${d.hostname}\nIP ${d.ip}`)}catch(e){v8Set('v8System','Telemetry error: '+e)}try{const d=await v8JSON('/api/v8/activity');v8Set('v8Activity',(d.activity||[]).map(x=>`[${x.time}] ${x.tag} :: ${x.text}`).join('\\n')||'Waiting...')}catch(e){}}
        async function v8RunDiagnostics(){try{v8Set('v8Result',JSON.stringify(await v8JSON('/api/v8/diagnostics'),null,2))}catch(e){v8Set('v8Result',e)}}
        async function v8RunNetwork(){try{v8Set('v8Result',JSON.stringify(await v8JSON('/api/v8/network'),null,2))}catch(e){v8Set('v8Result',e)}}
        async function v8ShowMemory(){try{v8Set('v8Result',JSON.stringify(await v8JSON('/api/v8/memory'),null,2))}catch(e){v8Set('v8Result',e)}}
        async function v8ShowStats(){try{v8Set('v8Result',JSON.stringify(await v8JSON('/api/v8/stats'),null,2))}catch(e){v8Set('v8Result',e)}}
        async function v8ShowProcesses(){try{v8Set('v8Result',JSON.stringify(await v8JSON('/api/v8/processes'),null,2))}catch(e){v8Set('v8Result',e)}}
        async function v8Gaming(){try{const d=await v8JSON('/api/v8/gaming',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({enabled:true})});v8Set('v8Result',d.respuesta)}catch(e){v8Set('v8Result',e)}}
        function v8OpenCleanup(){document.getElementById('v8-cleanup').classList.add('open');v8CleanupAnalyze()}function v8CleanupClose(){document.getElementById('v8-cleanup').classList.remove('open')}
        async function v8CleanupAnalyze(){try{v8Set('v8CleanupInfo',JSON.stringify(await v8JSON('/api/v8/cleanup/preview'),null,2))}catch(e){v8Set('v8CleanupInfo',e)}}
        async function v8CleanupConfirm(){if(!confirm('K.H.A.O.S. solo tocará cachés, temporales y papelera. ¿Confirmás la limpieza?'))return;try{const d=await v8JSON('/api/v8/cleanup',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({confirm:true})});v8Set('v8CleanupInfo',JSON.stringify(d,null,2));v8RefreshAll()}catch(e){v8Set('v8CleanupInfo',e)}}
        function v8SearchModules(q){if(!q){v8Set('v8Result','');return}fetch('/api/v8/search?q='+encodeURIComponent(q)).then(r=>r.json()).then(d=>v8Set('v8Result',JSON.stringify(d,null,2))).catch(e=>v8Set('v8Result',e))}
        document.addEventListener('keydown',e=>{if(e.ctrlKey&&e.key.toLowerCase()==='k'){e.preventDefault();v8Toggle(!document.getElementById('v8-deck').classList.contains('open'))}});
</script>

    <!-- ================= K.H.A.O.S. V3 CONTROL CENTER ================= -->
    <div id="v3-panel" class="v3-panel">
        <div class="v3-top">
            <span>CONTROL CENTER // V3</span>
            <button onclick="toggleV3(false)">×</button>
        </div>
        <div class="v3-grid">
            <div class="v3-card"><span>CPU</span><b id="v3cpu">--%</b><div class="bar"><i id="barcpu"></i></div></div>
            <div class="v3-card"><span>RAM</span><b id="v3ram">--%</b><div class="bar"><i id="barram"></i></div></div>
            <div class="v3-card"><span>DISCO</span><b id="v3disk">--%</b><div class="bar"><i id="bardisk"></i></div></div>
            <div class="v3-card"><span>IP</span><b id="v3ip">---</b></div>
            <div class="v3-card"><span>BATERÍA</span><b id="v3battery">N/A</b></div>
            <div class="v3-card"><span>UPTIME</span><b id="v3uptime">--</b></div>
        </div>
        <div class="v3-actions">
            <button onclick="quickCmd('estado del sistema')">📊 ESTADO</button>
            <button onclick="quickCmd('activa la cámara')">👁 CÁMARA</button>
            <button onclick="quickCmd('abre terminal')">⌨ TERMINAL</button>
            <button onclick="quickCmd('paseate por internet')">🌐 PASEO</button>
            <button onclick="quickCmd('ver notas')">📝 NOTAS</button>
            <button onclick="quickCmd('plugins')">🧩 PLUGINS</button>
        </div>
        <div class="v3-row">
            <select id="themeSelect" onchange="applyTheme(this.value)"><option value="cyan">CYAN // DEFAULT</option><option value="amber">AMBER // TACTICAL</option><option value="violet">VIOLET // NIGHT</option><option value="red">RED // ALERT</option></select>
            <button onclick="toggleSafeMode()" id="safeBtn">🛡 SAFE MODE: ON</button>
            <button onclick="startQuickTimer()">⏱ TIMER 60s</button>
        </div>
        <div class="v3-terminal" id="v3log">K.H.A.O.S. Control Center iniciado...</div>
    </div>
    <button id="v3-open" onclick="toggleV3(true)" title="Control Center">◈</button>

    <style>
      .v3-panel{position:fixed;right:18px;top:65px;width:min(470px,calc(100vw - 36px));max-height:calc(100vh - 110px);overflow:auto;z-index:80;background:rgba(1,8,22,.96);border:1px solid #00f0ff;border-radius:12px;box-shadow:0 0 40px rgba(0,240,255,.25);padding:14px;display:none;backdrop-filter:blur(12px)}
      .v3-top{display:flex;justify-content:space-between;align-items:center;color:#00f0ff;letter-spacing:2px;font-size:.72rem;margin-bottom:12px}.v3-top button{background:none;border:0;color:#ff0055;font-size:1.5rem;cursor:pointer}
      .v3-grid{display:grid;grid-template-columns:repeat(2,1fr);gap:8px}.v3-card{border:1px solid rgba(0,240,255,.25);padding:10px;border-radius:7px;background:rgba(0,20,40,.45)}.v3-card span{display:block;color:#ff9900;font-size:.55rem;letter-spacing:2px}.v3-card b{display:block;color:#fff;font-size:.82rem;margin-top:5px}.bar{height:4px;background:#061521;margin-top:8px;border-radius:3px;overflow:hidden}.bar i{display:block;height:100%;width:0;background:#00f0ff;transition:width .4s}
      .v3-actions{display:grid;grid-template-columns:repeat(2,1fr);gap:7px;margin-top:12px}.v3-actions button,.v3-row button,.v3-row select{padding:9px;background:rgba(0,240,255,.08);color:#00f0ff;border:1px solid rgba(0,240,255,.4);border-radius:6px;font-family:Orbitron;font-size:.58rem;cursor:pointer}.v3-row{display:flex;gap:7px;margin-top:8px}.v3-row>*{flex:1}.v3-terminal{margin-top:10px;min-height:60px;max-height:120px;overflow:auto;background:#000;color:#7ffaff;border:1px solid rgba(0,240,255,.2);padding:8px;font:11px monospace;border-radius:5px;white-space:pre-wrap}
      #v3-open{position:fixed;right:18px;top:18px;z-index:81;width:34px;height:34px;border-radius:50%;background:#000103;color:#00f0ff;border:1px solid #00f0ff;box-shadow:0 0 15px rgba(0,240,255,.5);cursor:pointer;font-size:1.1rem}
      body.theme-amber{filter:hue-rotate(10deg)} body.theme-violet{filter:hue-rotate(70deg)} body.theme-red{filter:hue-rotate(130deg)}
    

        /* =====================================================================
           K.H.A.O.S. V5.5 // ENTERPRISE PREMIUM UI
           SOLO CAPA VISUAL — PRESERVA NÚCLEO, AVATAR Y FUNCIONALIDAD EXISTENTE
           ===================================================================== */
        :root{
            --ep-bg:#02050a;
            --ep-surface:rgba(8,14,22,.82);
            --ep-surface-2:rgba(12,20,30,.68);
            --ep-border:rgba(171,224,239,.14);
            --ep-border-hot:rgba(103,232,255,.38);
            --ep-cyan:#69e7ff;
            --ep-blue:#78a9ff;
            --ep-green:#69f3b0;
            --ep-amber:#ffc86b;
            --ep-text:#f2f8fb;
            --ep-muted:#78909d;
            --ep-radius:16px;
        }
        html{background:var(--ep-bg)!important}
        body{
            background:
                radial-gradient(900px 500px at 50% 38%,rgba(31,105,139,.11),transparent 65%),
                radial-gradient(700px 450px at 8% 92%,rgba(60,95,170,.08),transparent 65%),
                var(--ep-bg)!important;
            color:var(--ep-text)!important;
        }
        body:before{opacity:.075!important;background:linear-gradient(rgba(255,255,255,.035) 1px,transparent 1px)!important;background-size:100% 5px!important}
        body:after{opacity:.07!important}
        #bgCanvas{background:radial-gradient(ellipse at 50% 42%,rgba(20,74,105,.22),transparent 54%),linear-gradient(180deg,#03070c,#010307)!important}

        /* ---------- premium boot / welcome ---------- */
        #landing-screen{
            background:
                radial-gradient(circle at 50% 38%,rgba(24,78,105,.22),transparent 38%),
                linear-gradient(145deg,#02060b 0%,#050b12 48%,#010307 100%)!important;
        }
        #landing-screen:before{
            content:'K.H.A.O.S.  /  PRIVATE AI PLATFORM';
            position:absolute;top:26px;left:32px;
            color:rgba(197,235,244,.48);font:600 9px/1 Inter,system-ui,sans-serif;
            letter-spacing:2.8px;
        }
        #landing-screen:after{
            content:'LOCAL OPERATING CORE   •   SECURE SESSION   •   BUILD 5.5';
            position:absolute;bottom:25px;left:50%;transform:translateX(-50%);
            color:rgba(117,151,164,.5);font:600 8px/1 Inter,system-ui,sans-serif;
            letter-spacing:2.2px;white-space:nowrap;
        }
        .landing-card{
            max-width:760px!important;padding:58px 64px 50px!important;
            border:1px solid rgba(159,222,239,.22)!important;
            border-radius:24px!important;
            background:linear-gradient(145deg,rgba(11,21,31,.91),rgba(3,8,14,.96))!important;
            box-shadow:0 40px 120px rgba(0,0,0,.62),0 0 90px rgba(74,193,226,.08),inset 0 1px 0 rgba(255,255,255,.045)!important;
            backdrop-filter:blur(24px)!important;
            overflow:hidden;
        }
        .landing-card:after{
            content:'';position:absolute;inset:0;pointer-events:none;
            background:linear-gradient(120deg,transparent 25%,rgba(255,255,255,.035) 50%,transparent 75%);
            transform:translateX(-110%);animation:epSweep 7s ease-in-out infinite;
        }
        @keyframes epSweep{55%,100%{transform:translateX(110%)}}
        .landing-card:before{
            content:'SYSTEM READY  /  ENCRYPTED';top:18px!important;left:24px!important;
            background:transparent!important;border:0!important;padding:0!important;
            color:rgba(105,231,255,.58)!important;font:700 8px/1 Inter,system-ui,sans-serif!important;
            letter-spacing:2.6px!important;
        }
        .landing-title{
            font-family:Inter,system-ui,sans-serif!important;font-size:clamp(3.2rem,7vw,5.4rem)!important;
            letter-spacing:.18em!important;font-weight:800!important;line-height:.95!important;
            color:#f7fbfd!important;text-shadow:0 12px 45px rgba(103,232,255,.16)!important;
            margin-bottom:16px!important;
        }
        .landing-subtitle{
            font-family:Inter,system-ui,sans-serif!important;color:var(--ep-cyan)!important;
            font-size:10px!important;font-weight:700!important;letter-spacing:3.6px!important;
            text-shadow:none!important;margin-bottom:26px!important;
        }
        .landing-desc{
            font-family:Inter,system-ui,sans-serif!important;font-size:12px!important;
            color:#8da5b0!important;letter-spacing:.35px!important;line-height:1.8!important;
            border-top:1px solid rgba(255,255,255,.07)!important;border-bottom:1px solid rgba(255,255,255,.07)!important;
            padding:18px 8px!important;margin-bottom:27px!important;
        }
        .voice-select-box{margin-bottom:28px!important}
        .voice-select-box label{font:700 9px Inter,system-ui,sans-serif!important;color:#7f9aa6!important;letter-spacing:1.6px!important}
        .voice-dropdown{border:1px solid rgba(103,232,255,.2)!important;background:rgba(255,255,255,.035)!important;border-radius:10px!important;box-shadow:none!important;font:600 11px Inter,system-ui,sans-serif!important;padding:10px 14px!important}
        .btn-group-landing{gap:10px!important}
        .btn-launch,.btn-qr-toggle{
            font-family:Inter,system-ui,sans-serif!important;border-radius:11px!important;
            font-size:10px!important;letter-spacing:1.8px!important;padding:13px 22px!important;
            transition:transform .22s ease,box-shadow .22s ease,background .22s ease!important;
        }
        .btn-launch{border:1px solid rgba(103,232,255,.62)!important;background:rgba(103,232,255,.10)!important;box-shadow:0 10px 35px rgba(103,232,255,.09)!important;animation:none!important}
        .btn-launch:hover{transform:translateY(-2px)!important;background:rgba(103,232,255,.18)!important;color:#fff!important;border-color:var(--ep-cyan)!important;box-shadow:0 16px 45px rgba(103,232,255,.15)!important}
        .btn-qr-toggle{border:1px solid rgba(255,255,255,.13)!important;background:rgba(255,255,255,.035)!important;color:#a9bac1!important;box-shadow:none!important}
        .btn-qr-toggle:hover{background:rgba(255,255,255,.08)!important;color:#fff!important;box-shadow:none!important}

        /* ---------- application chrome ---------- */
        .hud-header{
            height:72px!important;padding:0 30px!important;
            background:rgba(3,8,13,.82)!important;
            border-bottom:1px solid rgba(168,224,239,.12)!important;
            box-shadow:0 18px 50px rgba(0,0,0,.24)!important;
            backdrop-filter:blur(24px)!important;
        }
        .hud-header:before{
            content:'●';color:var(--ep-green);font-size:8px;position:absolute;left:30px;top:26px;
            text-shadow:0 0 12px var(--ep-green);
        }
        .system-title{font-family:Inter,system-ui,sans-serif!important;font-size:11px!important;letter-spacing:3px!important;font-weight:750!important;padding-left:15px;color:#edf7fa!important;text-shadow:none!important}
        .user-badge{font-family:Inter,system-ui,sans-serif!important;background:rgba(255,255,255,.035)!important;border:1px solid rgba(255,255,255,.1)!important;color:#8ea5ae!important;border-radius:9px!important;padding:8px 12px!important;font-size:8px!important;letter-spacing:1.3px!important}
        .hud-header:after{top:49px!important;left:46px!important;color:#536c77!important;letter-spacing:1.8px!important;font-size:6px!important}

        .side-panel{
            top:91px!important;bottom:104px!important;padding:14px!important;
            background:linear-gradient(180deg,rgba(8,15,23,.84),rgba(3,8,13,.72))!important;
            border:1px solid rgba(168,224,239,.11)!important;border-radius:14px!important;
            box-shadow:0 25px 70px rgba(0,0,0,.34)!important;backdrop-filter:blur(22px)!important;
        }
        #leftPanel{left:18px!important;width:224px!important}
        .right-panel{right:18px!important;width:258px!important}
        .panel-kicker{font-family:Inter,system-ui,sans-serif!important;color:#536e79!important;font-weight:700!important;letter-spacing:2px!important}
        .panel-title{font-family:Inter,system-ui,sans-serif!important;font-weight:700!important;letter-spacing:1.3px!important;color:#dcecf1!important}
        .nav-item{font-family:Inter,system-ui,sans-serif!important;font-weight:600!important;border-radius:9px!important;padding:11px 10px!important;color:#718994!important;letter-spacing:1.1px!important}
        .nav-item:hover,.nav-item.active{background:rgba(103,232,255,.055)!important;border-color:rgba(103,232,255,.15)!important;box-shadow:inset 2px 0 var(--ep-cyan),0 8px 25px rgba(0,0,0,.16)!important;color:#e8f8fc!important}
        .nav-dot{width:5px!important;height:5px!important}
        .core-badge{font-family:Inter,system-ui,sans-serif!important;background:rgba(103,232,255,.035)!important;border-color:rgba(103,232,255,.12)!important;border-radius:9px!important;padding:10px!important}

        /* ---------- preserve and enhance existing core/avatar ---------- */
        .center-container{
            width:calc(100vw - 550px)!important;max-width:1000px!important;
            height:calc(100vh - 252px)!important;min-height:430px!important;
            margin:91px auto 142px!important;border:1px solid rgba(168,224,239,.10)!important;
            border-radius:20px!important;
            background:radial-gradient(circle at 50% 48%,rgba(41,120,154,.08),transparent 45%),rgba(2,7,12,.18)!important;
            box-shadow:inset 0 0 100px rgba(103,232,255,.018),0 30px 90px rgba(0,0,0,.24)!important;
        }
        .center-container:before{inset:12px!important;border-color:rgba(168,224,239,.045)!important}
        .status-text{font-family:Inter,system-ui,sans-serif!important;background:rgba(4,11,17,.68)!important;border-color:rgba(168,224,239,.10)!important;border-radius:10px!important;color:#8ca5af!important;font-size:10px!important;letter-spacing:.55px!important;box-shadow:0 12px 35px rgba(0,0,0,.2)!important;backdrop-filter:blur(12px)!important}
        .live-eye-box{border-color:rgba(103,232,255,.28)!important;border-radius:12px!important;box-shadow:0 15px 40px rgba(0,0,0,.35)!important}
        .live-eye-label{font-family:Inter,system-ui,sans-serif!important;color:var(--ep-cyan)!important}

        /* ---------- command dock ---------- */
        .input-panel{max-width:850px!important;gap:8px!important;margin-bottom:8px!important}
        .input-command{
            font-family:Inter,system-ui,sans-serif!important;font-size:12px!important;
            padding:14px 17px!important;border:1px solid rgba(168,224,239,.14)!important;
            border-radius:12px!important;background:rgba(5,13,20,.82)!important;
            box-shadow:0 15px 40px rgba(0,0,0,.18),inset 0 1px 0 rgba(255,255,255,.025)!important;
        }
        .input-command:focus{border-color:rgba(103,232,255,.36)!important;box-shadow:0 0 0 3px rgba(103,232,255,.04),0 15px 40px rgba(0,0,0,.18)!important}
        .btn-action,.btn-mic{font-family:Inter,system-ui,sans-serif!important;border-radius:12px!important;box-shadow:none!important;font-size:9px!important;letter-spacing:1.1px!important}
        .btn-action{border-color:rgba(103,232,255,.22)!important;background:rgba(103,232,255,.055)!important;color:#a8d8e5!important}
        .btn-action:hover{transform:translateY(-1px)!important;background:rgba(103,232,255,.12)!important;box-shadow:0 12px 28px rgba(0,0,0,.18)!important}
        .btn-mic{border-color:rgba(255,93,125,.28)!important;background:rgba(255,70,105,.055)!important;color:#ff9bb0!important}
        .btn-mic:hover,.btn-mic.active{background:rgba(255,70,105,.16)!important;box-shadow:0 10px 28px rgba(255,70,105,.08)!important}
        .btn-live-mode{border-color:rgba(184,108,255,.38)!important;background:rgba(184,108,255,.07)!important;color:#d9b7ff!important;position:relative;overflow:hidden}
        .btn-live-mode:before{content:"";position:absolute;inset:0;background:linear-gradient(90deg,transparent,rgba(184,108,255,.18),transparent);transform:translateX(-100%);animation:liveSweep 2.8s infinite}
        .btn-live-mode.active{background:rgba(184,108,255,.18)!important;border-color:rgba(210,170,255,.7)!important;box-shadow:0 0 24px rgba(184,108,255,.18)!important}
        @keyframes liveSweep{to{transform:translateX(100%)}}

        /* ---------- telemetry strip ---------- */
        .telemetry-panel{
            max-width:850px!important;padding:9px 15px!important;gap:0!important;
            background:rgba(4,11,17,.76)!important;border:1px solid rgba(168,224,239,.10)!important;
            border-radius:12px!important;box-shadow:0 15px 40px rgba(0,0,0,.2)!important;backdrop-filter:blur(14px)!important;
        }
        .stat-box{flex:1;position:relative}
        .stat-box:not(:last-child):after{content:'';position:absolute;right:0;top:4px;height:22px;width:1px;background:rgba(255,255,255,.06)}
        .stat-label{font-family:Inter,system-ui,sans-serif!important;color:#58737e!important;font-size:7px!important;letter-spacing:1.6px!important}
        .stat-value{font-family:Inter,system-ui,sans-serif!important;font-size:10px!important;letter-spacing:.4px!important;text-shadow:none!important;color:#dff5fa!important}

        /* ---------- right telemetry / activity ---------- */
        .metric{padding:11px 0!important}
        .metric-row{font-family:Inter,system-ui,sans-serif!important;font-weight:600!important}
        .metric-val{font-family:Inter,system-ui,sans-serif!important}
        .meter{height:3px!important;border-radius:20px!important}
        .meter>i{background:linear-gradient(90deg,#55cfe8,#8eaaff)!important;box-shadow:0 0 10px rgba(103,232,255,.22)!important}
        .activity{font-family:Inter,system-ui,sans-serif!important}
        .log-line{border-left:1px solid rgba(103,232,255,.12);padding-left:8px;margin:6px 0}

        /* ---------- quick actions ---------- */
        .quick{font-family:Inter,system-ui,sans-serif!important;font-weight:650!important;border-radius:8px!important;padding:10px 5px!important;background:rgba(255,255,255,.025)!important;border-color:rgba(168,224,239,.10)!important;color:#7d99a4!important}
        .quick:hover{background:rgba(103,232,255,.06)!important;border-color:rgba(103,232,255,.2)!important;color:#dff7fb!important}

        /* ---------- control center ---------- */
        .v3-panel{border-radius:16px!important;background:linear-gradient(145deg,rgba(8,16,25,.97),rgba(2,7,12,.98))!important;box-shadow:0 35px 100px rgba(0,0,0,.68)!important}
        .v3-card{border-radius:10px!important;background:rgba(255,255,255,.022)!important}
        .v3-actions button,.v3-row button,.v3-row select{font-family:Inter,system-ui,sans-serif!important;border-radius:8px!important}
        .v3-terminal{font-family:ui-monospace,SFMono-Regular,Consolas,monospace!important;border-radius:10px!important}

        /* ---------- subtle responsive polish ---------- */
        @media(max-width:1250px){
            #leftPanel{width:195px!important}.right-panel{width:220px!important}
            .center-container{width:calc(100vw - 470px)!important}
        }
        @media(max-width:1100px){
            .hud-header{padding:0 18px!important}.system-title{font-size:9px!important;letter-spacing:2px!important}
            .center-container{width:calc(100vw - 28px)!important;margin-top:82px!important}
            .landing-card{padding:48px 28px 40px!important}.landing-title{font-size:3rem!important}
        }
        @media(max-width:600px){
            #landing-screen:after{font-size:6px;letter-spacing:1px}
            .landing-card{border-radius:18px!important}.landing-title{font-size:2.25rem!important;letter-spacing:.12em!important}
            .landing-subtitle{font-size:8px!important;letter-spacing:2px!important}
            .landing-desc{font-size:10px!important}
            .telemetry-panel{max-width:calc(100vw - 24px)!important}
        }
        @media(prefers-reduced-motion:reduce){*,*:before,*:after{animation-duration:.001ms!important;animation-iteration-count:1!important;scroll-behavior:auto!important;transition-duration:.001ms!important}}
</style>
    <script>
      function toggleV3(show){document.getElementById('v3-panel').style.display=show?'block':'none';if(show) updateSystem();}
      async function updateSystem(){try{const r=await fetch('/api/system');const d=await r.json();document.getElementById('v3cpu').textContent=d.cpu+'%';document.getElementById('v3ram').textContent=d.ram+'%';document.getElementById('v3disk').textContent=d.disk+'%';document.getElementById('v3ip').textContent=d.ip||'---';document.getElementById('v3uptime').textContent=d.uptime_min+' min';document.getElementById('v3battery').textContent=d.battery?d.battery.percent+'%':'N/A';document.getElementById('barcpu').style.width=d.cpu+'%';document.getElementById('barram').style.width=d.ram+'%';document.getElementById('bardisk').style.width=d.disk+'%';}catch(e){logV3('No se pudo actualizar telemetría.')}}
      setInterval(updateSystem,3000);
      function logV3(t){const el=document.getElementById('v3log');el.textContent='> '+t+'\\n'+el.textContent.slice(0,700)}
      function quickCmd(c){logV3('Ejecutando: '+c);enviarAPython(c)}
      async function toggleSafeMode(){try{const current=await fetch('/api/safe-mode').then(r=>r.json());const r=await fetch('/api/safe-mode',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({enabled:!current.enabled})});const d=await r.json();document.getElementById('safeBtn').textContent='🛡 SAFE MODE: '+(d.enabled?'ON':'OFF');logV3('Modo seguro '+(d.enabled?'activado':'desactivado'));}catch(e){}}
      async function startQuickTimer(){const r=await fetch('/api/timer',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({seconds:60,label:'Timer rápido'})});const d=await r.json();logV3(d.respuesta);hablarNativo(d.respuesta)}
      function applyTheme(v){document.body.classList.remove('theme-amber','theme-violet','theme-red');if(v!=='cyan')document.body.classList.add('theme-'+v);localStorage.setItem('khaos-theme',v);logV3('Tema: '+v)}
      const savedTheme=localStorage.getItem('khaos-theme');if(savedTheme){document.getElementById('themeSelect').value=savedTheme;applyTheme(savedTheme)}
        async function pcAction(data){try{const r=await fetch('/api/pc/action',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify(data)});const d=await r.json();pcOut(d.respuesta||JSON.stringify(d,null,2));if(d.needs_confirmation) setTimeout(()=>{if(confirm('K.H.A.O.S. preparó una acción sensible. ¿Querés CONFIRMARLA?')) pcConfirm();},50);}catch(e){pcOut('ERROR DE CONTROL: '+e)}}
        async function pcConfirm(){try{const r=await fetch('/api/pc/confirm',{method:'POST',headers:{'Content-Type':'application/json'},body:'{}'});const d=await r.json();pcOut(d.respuesta||JSON.stringify(d));}catch(e){pcOut('ERROR: '+e)}}
        function pcLaunch(app){pcAction({action:'launch',app})}
        function pcOpenPath(){pcAction({action:'open_path',path:document.getElementById('pcPath').value})}
        function pcDeletePath(){pcAction({action:'delete',path:document.getElementById('pcPath').value})}
        function pcDownload(){pcAction({action:'download',url:document.getElementById('pcUrl').value})}
        function pcOpenUrl(){pcAction({action:'open_url',url:document.getElementById('pcUrl').value})}
        function pcVolume(delta){pcAction({action:'volume',delta})}
        function pcLock(){pcAction({action:'lock'})}
        async function pcProcesses(){try{const r=await fetch('/api/pc/processes');const d=await r.json();pcOut(d.processes.map(x=>`${x.name||'?'}  PID:${x.pid}  CPU:${x.cpu_percent||0}%`).join('\\n')||'Sin procesos visibles');}catch(e){pcOut('ERROR: '+e)}}
        document.addEventListener('keydown',e=>{if((e.ctrlKey||e.metaKey)&&e.key.toLowerCase()==='k'){e.preventDefault();togglePCCenter(true)}});
    </script>

    <!-- ================================================================
         K.H.A.O.S. V7 // COMMAND DECK EXTENSION
         ================================================================ -->
    <div id="v7-toast-stack"></div>
    <div id="v7-overlay" class="v7-overlay" aria-hidden="true">
      <div class="v7-window">
        <div class="v7-head">
          <div>
            <span class="v7-kicker">K.H.A.O.S. // V7 COMMAND DECK</span>
            <strong id="v7-title">SYSTEM CONTROL</strong>
          </div>
          <button class="v7-close" onclick="v7Close()">×</button>
        </div>

        <div class="v7-tabs">
          <button data-v7="dashboard" class="v7-tab active" onclick="v7Tab('dashboard')">DASHBOARD</button>
          <button data-v7="files" class="v7-tab" onclick="v7Tab('files')">FILES</button>
          <button data-v7="memory" class="v7-tab" onclick="v7Tab('memory')">MEMORY</button>
          <button data-v7="security" class="v7-tab" onclick="v7Tab('security')">SECURITY</button>
          <button data-v7="network" class="v7-tab" onclick="v7Tab('network')">NETWORK</button>
          <button data-v7="tools" class="v7-tab" onclick="v7Tab('tools')">TOOLS</button>
          <button data-v7="settings" class="v7-tab" onclick="v7Tab('settings')">SETTINGS</button>
        </div>

        <div id="v7-dashboard" class="v7-section">
          <div class="v7-grid v7-grid-4">
            <div class="v7-card"><span>CPU</span><b id="v7cpu">--%</b><i id="v7cpuBar"></i></div>
            <div class="v7-card"><span>RAM</span><b id="v7ram">--%</b><i id="v7ramBar"></i></div>
            <div class="v7-card"><span>DISK</span><b id="v7disk">--%</b><i id="v7diskBar"></i></div>
            <div class="v7-card"><span>FREE</span><b id="v7free">--</b></div>
          </div>
          <div class="v7-grid v7-grid-2">
            <div class="v7-card v7-large">
              <span>SYSTEM DIAGNOSTICS</span>
              <div id="v7checks" class="v7-checks">Analizando...</div>
            </div>
            <div class="v7-card v7-large">
              <span>LIVE ACTIVITY</span>
              <div id="v7activity" class="v7-console">Waiting...</div>
            </div>
          </div>
          <div class="v7-actions">
            <button onclick="v7RunDiagnostics()">RUN SYSTEM CHECK</button>
            <button onclick="v7Gaming(true)">🎮 GAMING MODE</button>
            <button onclick="v7Gaming(false)">EXIT GAMING</button>
            <button onclick="v7Tab('tools')">OPEN TOOLS</button>
          </div>
        </div>

        <div id="v7-files" class="v7-section" style="display:none">
          <div class="v7-card">
            <span>UNIVERSAL FILE VIEWER</span>
            <div class="v7-file-row">
              <input id="v7filePath" placeholder="~/Descargas, ~/Escritorio, etc.">
              <button onclick="v7ListFiles()">SCAN</button>
            </div>
            <div id="v7filesOut" class="v7-console">Elegí una ruta dentro de tu HOME.</div>
          </div>
          <div class="v7-card">
            <span>STORAGE RECOVERY // SAFE MODE</span>
            <p id="v7cleanupSummary" class="v7-muted">Escaneá antes de borrar. Nunca se eliminan documentos normales.</p>
            <div class="v7-clean-grid">
              <button onclick="v7CleanupPreview()">ANALYZE SPACE</button>
              <button class="danger" onclick="v7Cleanup()">CLEAN TEMP/CACHE</button>
            </div>
          </div>
        </div>

        <div id="v7-memory" class="v7-section" style="display:none">
          <div class="v7-grid v7-grid-2">
            <div class="v7-card v7-large"><span>CONVERSATION MEMORY</span><div id="v7memory" class="v7-console">Loading...</div></div>
            <div class="v7-card v7-large"><span>TASKS / NOTES</span><div id="v7tasks" class="v7-console">Loading...</div></div>
          </div>
          <div class="v7-card"><span>STATISTICS</span><div id="v7stats" class="v7-stats"></div></div>
        </div>

        <div id="v7-security" class="v7-section" style="display:none">
          <div class="v7-security-banner">🛡 SAFE OPERATIONS // SYSTEM ROOTS PROTECTED</div>
          <div class="v7-grid v7-grid-2">
            <div class="v7-card v7-large"><span>PROTECTED ZONES</span><div class="v7-console">/  /usr  /etc  /var  /boot  /dev  /proc  /sys  /root  /run  /sbin  /lib  /lib64</div></div>
            <div class="v7-card v7-large"><span>DESTRUCTIVE ACTION POLICY</span><div class="v7-console">Delete / Move / Kill / Shutdown / Reboot / Cleanup overwrite → explicit confirmation.</div></div>
          </div>
        </div>

        <div id="v7-network" class="v7-section" style="display:none">
          <div class="v7-card">
            <span>NETWORK CENTER</span>
            <div id="v7network" class="v7-console">Loading...</div>
            <button onclick="v7Network()">REFRESH NETWORK</button>
          </div>
        </div>

        <div id="v7-tools" class="v7-section" style="display:none">
          <div class="v7-grid v7-grid-3">
            <button class="v7-tool" onclick="quickCmd('estado avanzado')">📊 SYSTEM STATUS</button>
            <button class="v7-tool" onclick="quickCmd('procesos')">⚙ PROCESSES</button>
            <button class="v7-tool" onclick="quickCmd('plugins')">🧩 PLUGINS</button>
            <button class="v7-tool" onclick="quickCmd('ver notas')">📝 TASKS</button>
            <button class="v7-tool" onclick="quickCmd('abrir terminal')">⌨ TERMINAL</button>
            <button class="v7-tool" onclick="quickCmd('activa la cámara')">👁 VISION</button>
            <button class="v7-tool" onclick="quickCmd('siguiente canción')">⏭ MEDIA NEXT</button>
            <button class="v7-tool" onclick="toggleQR(true)">📱 REMOTE</button>
            <button class="v7-tool" onclick="v7Tab('settings')">🎨 THEMES</button>
          </div>
        </div>

        <div id="v7-settings" class="v7-section" style="display:none">
          <div class="v7-card">
            <span>APPEARANCE</span>
            <div class="v7-settings-row">
              <button onclick="applyTheme('cyan')">CYAN</button>
              <button onclick="applyTheme('amber')">AMBER</button>
              <button onclick="applyTheme('violet')">VIOLET</button>
              <button onclick="applyTheme('red')">RED</button>
            </div>
          </div>
          <div class="v7-card">
            <span>ANIMATION</span>
            <div class="v7-settings-row">
              <button onclick="document.body.classList.toggle('v7-reduced')">TOGGLE MOTION</button>
              <button onclick="v7Toast('UI preference updated')">TEST NOTIFICATION</button>
            </div>
          </div>
        </div>
      </div>
    </div>

    <button id="v7-open" title="K.H.A.O.S. V7" onclick="v7Open('dashboard')">◈</button>
    <div id="v7-search" class="v7-search">
      <input id="v7SearchInput" placeholder="⌕ Search K.H.A.O.S..." oninput="v7Search()">
      <div id="v7SearchResults"></div>
    </div>

    <style>
      #v7-open{position:fixed;right:22px;bottom:24px;z-index:90;width:48px;height:48px;border-radius:14px;border:1px solid rgba(103,232,255,.3);background:rgba(4,12,19,.88);color:#69e7ff;box-shadow:0 12px 35px rgba(0,0,0,.45),0 0 28px rgba(103,232,255,.08);cursor:pointer;font-size:19px;backdrop-filter:blur(16px);transition:.2s}
      #v7-open:hover{transform:translateY(-2px);border-color:rgba(103,232,255,.65);box-shadow:0 15px 40px rgba(0,0,0,.5),0 0 35px rgba(103,232,255,.16)}
      .v7-overlay{display:none;position:fixed;inset:0;z-index:85;background:rgba(0,3,7,.58);backdrop-filter:blur(10px);align-items:center;justify-content:center;padding:24px}
      .v7-window{width:min(1050px,96vw);max-height:88vh;overflow:auto;border:1px solid rgba(103,232,255,.2);border-radius:20px;background:linear-gradient(145deg,rgba(7,16,25,.98),rgba(2,7,12,.99));box-shadow:0 35px 120px rgba(0,0,0,.75),0 0 80px rgba(103,232,255,.07);padding:18px}
      .v7-head{display:flex;justify-content:space-between;align-items:center;border-bottom:1px solid rgba(255,255,255,.07);padding:5px 4px 15px;margin-bottom:10px}
      .v7-head strong{display:block;font:700 18px Inter,system-ui,sans-serif;color:#eafaff;letter-spacing:1px}
      .v7-kicker{display:block;font:700 7px Inter,system-ui,sans-serif;color:#5f8793;letter-spacing:2.5px;margin-bottom:5px}
      .v7-close{border:0;background:none;color:#78909d;font-size:28px;cursor:pointer}
      .v7-tabs{display:flex;gap:6px;overflow:auto;padding:4px 0 12px}
      .v7-tab,.v7-actions button,.v7-card button,.v7-tool,.v7-settings-row button{border:1px solid rgba(103,232,255,.13);background:rgba(103,232,255,.035);color:#91c5d1;border-radius:9px;padding:10px 11px;font:700 8px Inter,system-ui,sans-serif;letter-spacing:1px;cursor:pointer;white-space:nowrap}
      .v7-tab:hover,.v7-tab.active,.v7-actions button:hover,.v7-card button:hover,.v7-tool:hover,.v7-settings-row button:hover{background:rgba(103,232,255,.09);color:#effcff;border-color:rgba(103,232,255,.3)}
      .v7-grid{display:grid;gap:8px;margin-bottom:8px}.v7-grid-4{grid-template-columns:repeat(4,1fr)}.v7-grid-3{grid-template-columns:repeat(3,1fr)}.v7-grid-2{grid-template-columns:repeat(2,1fr)}
      .v7-card{border:1px solid rgba(168,224,239,.1);background:rgba(255,255,255,.018);border-radius:12px;padding:13px}
      .v7-card>span{display:block;color:#5c7b87;font:700 7px Inter,system-ui,sans-serif;letter-spacing:1.7px;margin-bottom:9px}.v7-card>b{display:block;color:#e9fbff;font:750 19px Inter,system-ui,sans-serif}
      .v7-card i{display:block;height:3px;margin-top:9px;border-radius:8px;background:linear-gradient(90deg,#55cfe8,#8eaaff);width:0;box-shadow:0 0 10px rgba(103,232,255,.22)}
      .v7-large{min-height:180px}.v7-checks{display:grid;grid-template-columns:repeat(2,1fr);gap:6px}.v7-check{padding:7px;border-radius:7px;background:rgba(255,255,255,.025);font:700 8px Inter,system-ui,sans-serif;color:#7e9ba6}.v7-check b{float:right}.v7-console{background:#010407;border:1px solid rgba(103,232,255,.07);border-radius:9px;padding:10px;min-height:100px;max-height:210px;overflow:auto;color:#79aebb;font:10px/1.65 ui-monospace,monospace;white-space:pre-wrap}
      .v7-actions,.v7-settings-row{display:flex;gap:7px;flex-wrap:wrap}.v7-muted{color:#718b95;font:10px/1.6 Inter,system-ui,sans-serif}.v7-security-banner{padding:11px;border:1px solid rgba(101,245,176,.15);background:rgba(101,245,176,.035);border-radius:10px;color:#8dd7b4;font:700 8px Inter,system-ui,sans-serif;letter-spacing:1.5px;margin-bottom:8px}
      .v7-file-row{display:flex;gap:7px}.v7-file-row input{flex:1;min-width:0;border:1px solid rgba(168,224,239,.1);background:rgba(0,0,0,.25);color:#dff7fb;border-radius:9px;padding:11px;font:10px Inter,system-ui,sans-serif}.v7-clean-grid{display:flex;gap:7px}.v7-clean-grid .danger{border-color:rgba(255,93,125,.25);color:#ff9eb0}
      .v7-stats{display:grid;grid-template-columns:repeat(3,1fr);gap:7px}.v7-stat{padding:10px;background:rgba(255,255,255,.02);border-radius:8px}.v7-stat small{display:block;color:#58747f;font:700 7px Inter,system-ui,sans-serif}.v7-stat b{color:#eafaff;font:700 14px Inter,system-ui,sans-serif}
      .v7-search{position:fixed;top:82px;left:50%;transform:translateX(-50%);z-index:88;width:min(620px,90vw);display:none;border:1px solid rgba(103,232,255,.15);background:rgba(3,10,16,.96);border-radius:12px;padding:8px;box-shadow:0 20px 70px rgba(0,0,0,.5);backdrop-filter:blur(20px)}
      .v7-search input{width:100%;padding:12px;border-radius:8px;border:1px solid rgba(103,232,255,.1);background:rgba(255,255,255,.025);color:#fff;outline:none}.v7-search div div{padding:8px;color:#8fb7c2;font:9px Inter,system-ui,sans-serif;border-bottom:1px solid rgba(255,255,255,.04)}
      #v7-toast-stack{position:fixed;right:22px;top:86px;z-index:100;display:flex;flex-direction:column;gap:7px}.v7-toast{padding:11px 13px;max-width:340px;border:1px solid rgba(103,232,255,.18);background:rgba(4,12,19,.94);border-radius:10px;color:#dff7fb;font:9px/1.5 Inter,system-ui,sans-serif;box-shadow:0 15px 45px rgba(0,0,0,.4);animation:v7toast .25s ease}@keyframes v7toast{from{opacity:0;transform:translateY(-6px)}to{opacity:1;transform:none}}
      body.v7-reduced *{animation:none!important;transition:none!important}
      @media(max-width:800px){.v7-grid-4,.v7-grid-2,.v7-grid-3{grid-template-columns:1fr 1fr}.v7-window{padding:12px}.v7-tabs{flex-wrap:nowrap}.v7-head strong{font-size:14px}}
      @media(max-width:520px){.v7-grid-4,.v7-grid-2,.v7-grid-3{grid-template-columns:1fr}.v7-overlay{padding:8px}.v7-window{max-height:94vh}.v7-file-row{flex-wrap:wrap}.v7-file-row input{flex-basis:100%}}
    </style>

    <script>
      const v7Overlay=document.getElementById('v7-overlay');
      function v7Open(tab='dashboard'){v7Overlay.style.display='flex';v7Overlay.setAttribute('aria-hidden','false');v7Tab(tab);v7Refresh();}
      function v7Close(){v7Overlay.style.display='none';v7Overlay.setAttribute('aria-hidden','true')}
      function v7Tab(tab){
        document.querySelectorAll('.v7-section').forEach(x=>x.style.display='none');
        const el=document.getElementById('v7-'+tab);if(el)el.style.display='block';
        document.querySelectorAll('.v7-tab').forEach(x=>x.classList.toggle('active',x.dataset.v7===tab));
        const titles={dashboard:'SYSTEM CONTROL',files:'FILE & STORAGE CENTER',memory:'MEMORY & STATS',security:'SECURITY CENTER',network:'NETWORK CENTER',tools:'TOOLS & AUTOMATION',settings:'APPEARANCE & SETTINGS'};
        document.getElementById('v7-title').textContent=titles[tab]||'SYSTEM CONTROL';
        if(tab==='memory')v7Memory();if(tab==='network')v7Network();if(tab==='files')v7CleanupPreview();
      }
      function v7Toast(text){
        const box=document.createElement('div');box.className='v7-toast';box.textContent=text;
        document.getElementById('v7-toast-stack').appendChild(box);setTimeout(()=>box.remove(),3800);
      }
      async function v7JSON(url,opts={}){const r=await fetch(url,opts);return await r.json()}
      async function v7Refresh(){
        try{
          const d=await v7JSON('/api/system');
          ['cpu','ram','disk'].forEach(k=>{const v=Math.round(d[k]||0);document.getElementById('v7'+k).textContent=v+'%';document.getElementById('v7'+k+'Bar').style.width=v+'%'});
          const st=await v7JSON('/api/v7/storage');document.getElementById('v7free').textContent=st.free_human||'--';
          v7RunDiagnostics();v7Activity();
        }catch(e){v7Toast('No se pudo actualizar el dashboard')}
      }
      async function v7RunDiagnostics(){
        try{const d=await v7JSON('/api/v7/diagnostics');document.getElementById('v7checks').innerHTML=d.checks.map(x=>`<div class="v7-check">${x.name}<b>${x.status}</b></div>`).join('');}
        catch(e){document.getElementById('v7checks').textContent='Diagnostics unavailable'}
      }
      async function v7Activity(){
        try{const d=await v7JSON('/api/v7/activity');document.getElementById('v7activity').textContent=d.activity.map(x=>`[${x.time}] ${x.tag} :: ${x.text}`).join('\\n')||'Waiting...'}
        catch(e){}
      }
      async function v7Memory(){
        try{const d=await v7JSON('/api/v7/memory');document.getElementById('v7memory').textContent=(d.conversation||[]).map(x=>`USER: ${x.q}\nKHAOS: ${x.r}`).join('\\n\\n')||'No conversation memory';document.getElementById('v7tasks').textContent=d.tasks||'No tasks';const st=await v7JSON('/api/v7/stats');document.getElementById('v7stats').innerHTML=[['COMMANDS',st.commands],['SUCCESS',st.successful_commands],['ERRORS',st.errors],['CLEANUPS',st.cleanup_runs],['FREED',fmtBytes(st.cleanup_bytes)],['UPTIME',st.uptime_human]].map(x=>`<div class="v7-stat"><small>${x[0]}</small><b>${x[1]}</b></div>`).join('')}
        catch(e){v7Toast('Memory center unavailable')}
      }
      async function v7Network(){
        try{const d=await v7JSON('/api/v7/network');document.getElementById('v7network').textContent=`HOST: ${d.hostname}\nLOCAL IP: ${d.ip}\nINTERFACES: ${d.interfaces.join(', ')}\nINET CONNECTIONS: ${d.connections}`}
        catch(e){document.getElementById('v7network').textContent='Network data unavailable'}
      }
      async function v7ListFiles(){
        const path=document.getElementById('v7filePath').value.trim()||'~';
        try{const d=await v7JSON('/api/files?path='+encodeURIComponent(path));document.getElementById('v7filesOut').textContent=d.error||(`${d.path}\n\n`+(d.items||[]).map(x=>`${x.type==='dir'?'📁':'📄'} ${x.name}${x.size!=null?'  '+fmtBytes(x.size):''}`).join('\\n'))}
        catch(e){document.getElementById('v7filesOut').textContent='File scan error'}
      }
      async function v7CleanupPreview(){
        try{const d=await v7JSON('/api/v7/cleanup/preview');document.getElementById('v7cleanupSummary').textContent=d.message+' No se borró nada.';v7Toast(d.message)}
        catch(e){v7Toast('No se pudo analizar el almacenamiento')}
      }
      async function v7Cleanup(){
        const d=await v7JSON('/api/v7/cleanup',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({confirm:false})});
        if(d.preview){const p=d.preview;document.getElementById('v7cleanupSummary').textContent=p.message+' Si querés continuar, escribí "CONFIRMAR LIMPIEZA".';v7Toast(p.message)}
      }
      async function v7Gaming(enabled){
        try{const d=await v7JSON('/api/v7/gaming',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({enabled})});v7Toast(d.respuesta)}
        catch(e){}
      }
      function fmtBytes(n){n=Number(n)||0;const u=['B','KB','MB','GB','TB'];let i=0;while(n>=1024&&i<u.length-1){n/=1024;i++}return n.toFixed(1)+' '+u[i]}
      function v7Search(){
        const q=document.getElementById('v7SearchInput').value.trim();const box=document.getElementById('v7SearchResults');
        if(!q){box.innerHTML='';return}
        fetch('/api/v7/search?q='+encodeURIComponent(q)).then(r=>r.json()).then(d=>box.innerHTML=(d.results||[]).map(x=>`<div>${x.type.toUpperCase()} :: ${x.name}</div>`).join('')||'<div>No results</div>')
      }
      document.addEventListener('keydown',e=>{
        if((e.ctrlKey||e.metaKey)&&e.key.toLowerCase()==='k'){e.preventDefault();v7Open('dashboard')}
        if(e.key==='Escape')v7Close();
      });
      setInterval(()=>{if(v7Overlay.style.display==='flex')v7Refresh()},5000);
    </script>

</body>
</html>
"""

# ===========================================================================
# 📱 CONTROL REMOTO MÓVIL (INTACTO)
# ===========================================================================
HTML_CONTROL_REMOTO = """
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>K.H.A.O.S. // Control Remoto Móvil</title>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Orbitron:wght@400;700;900&display=swap');
        * { margin: 0; padding: 0; box-sizing: border-box; font-family: 'Orbitron', sans-serif; }
        body { 
            background: #000103; color: #00f0ff; height: 100vh; overflow: hidden; 
            display: flex; flex-direction: column; justify-content: center; align-items: center; padding: 20px;
        }
        .remote-card {
            border: 2px solid rgba(0, 240, 255, 0.6); padding: 30px 20px; background: rgba(1, 10, 25, 0.95);
            box-shadow: 0 0 40px rgba(0,240,255,0.3); border-radius: 16px; text-align: center; max-width: 400px; width: 100%;
        }
        .remote-title { font-size: 1.6rem; letter-spacing: 4px; color: #fff; text-shadow: 0 0 15px #00f0ff; margin-bottom: 5px; font-weight: 900; }
        .remote-subtitle { font-size: 0.65rem; letter-spacing: 2px; color: #ff9900; margin-bottom: 25px; }
        
        .btn-remoto-mic {
            width: 130px; height: 130px; border-radius: 50%; background: linear-gradient(135deg, rgba(255, 0, 85, 0.2), rgba(150, 0, 50, 0.3));
            border: 3px solid #ff0055; color: #ff0055; font-size: 2.5rem; display: flex; justify-content: center; align-items: center;
            margin: 0 auto 20px auto; cursor: pointer; box-shadow: 0 0 30px rgba(255, 0, 85, 0.4); transition: all 0.3s;
        }
        .btn-remoto-mic:active, .btn-remoto-mic.active {
            background: #ff0055; color: #fff; box-shadow: 0 0 50px #ff0055; transform: scale(1.08); animation: pulse-remoto 1s infinite alternate;
        }
        @keyframes pulse-remoto { 0% { transform: scale(1); } 100% { transform: scale(1.12); } }

        .input-remoto-box {
            display: flex; gap: 8px; margin-top: 15px;
        }
        .input-remoto {
            flex: 1; padding: 12px; background: rgba(0, 20, 40, 0.9); border: 1px solid rgba(0, 240, 255, 0.5);
            color: #fff; border-radius: 8px; font-size: 0.85rem; outline: none;
        }
        .btn-enviar-remoto {
            padding: 0 16px; background: #00f0ff; color: #000103; font-weight: bold; border: none; border-radius: 8px; cursor: pointer;
        }

        .status-remoto {
            font-size: 0.75rem; color: #d0e0f0; background: rgba(0, 20, 40, 0.8); border: 1px solid rgba(0,240,255,0.3);
            padding: 10px; border-radius: 8px; letter-spacing: 1px; min-height: 45px; display: flex; align-items: center; justify-content: center; margin-top: 15px;
        }
        .pc-command-center{position:fixed;left:50%;top:50%;transform:translate(-50%,-50%) scale(.98);width:min(760px,92vw);max-height:78vh;overflow:auto;z-index:95;display:none;padding:18px;border:1px solid rgba(103,232,255,.2);border-radius:18px;background:linear-gradient(145deg,rgba(7,16,24,.98),rgba(2,7,12,.99));box-shadow:0 35px 120px rgba(0,0,0,.72),0 0 70px rgba(103,232,255,.08);backdrop-filter:blur(24px)}
        .pc-head{display:flex;justify-content:space-between;align-items:center;border-bottom:1px solid rgba(255,255,255,.07);padding-bottom:13px;margin-bottom:13px}.pc-head b{display:block;color:#e8f8fc;font:700 14px Inter,system-ui,sans-serif;letter-spacing:1px}.pc-kicker{display:block;color:#5e8591;font:700 7px Inter,system-ui,sans-serif;letter-spacing:2px;margin-bottom:4px}.pc-head button{border:0;background:none;color:#7f9aa4;font-size:25px;cursor:pointer}.pc-grid{display:grid;grid-template-columns:repeat(4,1fr);gap:7px}.pc-grid button,.pc-form button{border:1px solid rgba(103,232,255,.15);background:rgba(103,232,255,.04);color:#9ac8d3;border-radius:9px;padding:11px 8px;font:700 8px Inter,system-ui,sans-serif;letter-spacing:1px;cursor:pointer}.pc-grid button:hover,.pc-form button:hover{background:rgba(103,232,255,.1);color:#fff}.pc-form{display:flex;gap:7px;margin-top:8px}.pc-form input{flex:1;min-width:0;border:1px solid rgba(255,255,255,.09);background:rgba(0,0,0,.28);color:#dff7fb;border-radius:9px;padding:11px 12px;font:11px Inter,system-ui,sans-serif;outline:none}.pc-output{margin-top:10px;min-height:74px;max-height:170px;overflow:auto;padding:11px;border-radius:9px;background:#010407;border:1px solid rgba(103,232,255,.08);color:#77b9c6;font:10px ui-monospace,monospace;white-space:pre-wrap}
        @media(max-width:700px){.pc-grid{grid-template-columns:repeat(2,1fr)}.pc-form{flex-wrap:wrap}.pc-form input{flex-basis:100%}}
    </style>
</head>
<body>
    <div class="remote-card">
        <div class="remote-title">K.H.A.O.S.</div>
        <div class="remote-subtitle">CONTROL TÁCTICO REMOTO // MÓVIL</div>
        
        <button id="btnRemotoMic" class="btn-remoto-mic" onclick="activarVozMovil()">🎤</button>
        <div style="font-size: 0.65rem; color: #ff9900; margin-bottom: 10px;">Toca para hablar o escribe abajo</div>

        <div class="input-remoto-box">
            <input type="text" id="inputRemoto" class="input-remoto" placeholder="Escribe comando rápido..." onkeypress="checkEnterRemoto(event)">
            <button class="btn-enviar-remoto" onclick="enviarTextoRemoto()">ENVIAR</button>
        </div>
        
        <div id="statusRemoto" class="status-remoto">Enlace móvil activo y listo...</div>
    </div>

    <script>
        const statusRemoto = document.getElementById('statusRemoto');
        const btnRemotoMic = document.getElementById('btnRemotoMic');
        const inputRemoto = document.getElementById('inputRemoto');

        function activarVozMovil() {
            const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
            if (!SpeechRecognition) {
                alert("Usa el cuadro de texto de abajo para enviar órdenes si tu navegador móvil bloquea el micro.");
                return;
            }
            try {
                const recognition = new SpeechRecognition();
                recognition.lang = 'es-ES';
                recognition.continuous = false;
                recognition.interimResults = false;

                recognition.onstart = () => {
                    btnRemotoMic.classList.add('active');
                    statusRemoto.innerText = "Escuchando orden desde el celular...";
                };
                recognition.onresult = (e) => {
                    const texto = e.results[0][0].transcript;
                    statusRemoto.innerText = `Enviando: "${texto}"`;
                    enviarComandoRemoto(texto);
                    btnRemotoMic.classList.remove('active');
                };
                recognition.onerror = (e) => {
                    statusRemoto.innerText = "Error de micro en red local. Usá el cuadro de texto de abajo.";
                    btnRemotoMic.classList.remove('active');
                };
                recognition.onend = () => {
                    btnRemotoMic.classList.remove('active');
                };
                recognition.start();
            } catch(err) {
                statusRemoto.innerText = "Escribe tu orden abajo por seguridad del navegador.";
            }
        }

        function enviarTextoRemoto() {
            const texto = inputRemoto.value.trim();
            if (texto !== "") {
                statusRemoto.innerText = `Enviando: "${texto}"`;
                enviarComandoRemoto(texto);
                inputRemoto.value = "";
            }
        }

        function checkEnterRemoto(e) {
            if (e.key === 'Enter') enviarTextoRemoto();
        }

        async function enviarComandoRemoto(cmd) {
            try {
                const res = await fetch('/api/comando', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({ comando: cmd })
                });
                const data = await res.json();
                if (data.respuesta) {
                    statusRemoto.innerText = `K.H.A.O.S.: ${data.respuesta}`;
                    hablarRespuesta(data.respuesta);
                }
            } catch(err) {
                statusRemoto.innerText = "Error al conectar con la PC central.";
            }
        }

        function hablarRespuesta(texto) {
            if (!('speechSynthesis' in window)) return;
            window.speechSynthesis.cancel();
            const utterance = new SpeechSynthesisUtterance(texto);
            utterance.lang = 'es-ES';
            utterance.rate = 1.02;
            utterance.pitch = 0.9;
            window.speechSynthesis.speak(utterance);
        }
    </script>
</body>
</html>
"""

QR_BASE64 = ""
app = Flask(__name__)
CORS(app)

@app.route("/")
def index():
    # HTML_CODE contiene CSS/JS con secuencias como {#..., que Jinja2
    # interpreta como inicio de comentario. No necesitamos renderizar
    # una plantilla completa: solo sustituimos el QR de forma segura.
    html = HTML_CODE.replace("{{ qr_code }}", QR_BASE64)
    return Response(html, mimetype="text/html; charset=utf-8")

# ===========================================================================
# K.H.A.O.S. LIVE MODE // CONVERSACIÓN CONTINUA
# Solo rostro, sin barra de navegador, siempre encima.
# El navegador escucha -> envía cada turno -> recibe Gemini -> habla -> vuelve a escuchar.
# ===========================================================================
LIVE_HTML = r"""
<!doctype html>
<html lang="es">
<head>
<meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1">
<title>K.H.A.O.S. // Live</title>
<style>
:root{--c:#67e8ff;--v:#b86cff;--a:#ff9900}
*{box-sizing:border-box}html,body{margin:0;width:100%;height:100%;overflow:hidden;background:#02070c}
body{display:flex;align-items:center;justify-content:center;font-family:Arial,sans-serif;color:#fff;user-select:none}
#wrap{position:relative;width:100vw;height:100vh;display:flex;align-items:center;justify-content:center;cursor:pointer}
#face{width:100%;height:100%;display:block}
#state{position:absolute;bottom:12px;left:50%;transform:translateX(-50%);font:700 8px/1 monospace;letter-spacing:2px;color:rgba(215,190,255,.8);opacity:.0;transition:.2s;white-space:nowrap;text-shadow:0 0 10px rgba(184,108,255,.7);pointer-events:none}
#wrap:hover #state{opacity:1}
#hint{position:absolute;top:10px;left:50%;transform:translateX(-50%);font:700 7px/1 monospace;letter-spacing:2px;color:rgba(103,232,255,.75);opacity:0;transition:.2s;white-space:nowrap;pointer-events:none}
#wrap:hover #hint{opacity:1}
.live-ring{position:absolute;inset:7%;border:1px solid rgba(184,108,255,.18);border-radius:50%;pointer-events:none;box-shadow:0 0 40px rgba(184,108,255,.08),inset 0 0 35px rgba(103,232,255,.04)}
</style></head>
<body>
<div id="wrap"><canvas id="face"></canvas><div class="live-ring"></div><div id="hint">K.H.A.O.S. // LIVE</div><div id="state">INICIALIZANDO...</div></div>
<script>
const canvas=document.getElementById('face'),ctx=canvas.getContext('2d'),wrap=document.getElementById('wrap'),stateEl=document.getElementById('state');
let phase=0,blink=0,isBlink=false,talking=false,mouth=4,listening=false,busy=false,live=true,recognition=null,shouldListen=true;
function resize(){const d=devicePixelRatio||1;canvas.width=innerWidth*d;canvas.height=innerHeight*d;canvas.style.width=innerWidth+'px';canvas.style.height=innerHeight+'px';ctx.setTransform(d,0,0,d,0,0)}addEventListener('resize',resize);resize();
function draw(){const w=innerWidth,h=innerHeight,cx=w/2,cy=h/2,r=Math.min(w,h)*.37;ctx.clearRect(0,0,w,h);phase+=.025;ctx.save();ctx.globalCompositeOperation='lighter';for(let i=0;i<4;i++){ctx.beginPath();const rr=r*(1.48-i*.14);ctx.arc(cx,cy,rr,phase*(i%2?-.65:.8),phase*(i%2?-.65:.8)+Math.PI*(1.15-i*.12));ctx.strokeStyle=i===1?'rgba(184,108,255,.72)':'rgba(0,240,255,.48)';ctx.lineWidth=Math.max(1.2,r*.014);ctx.setLineDash([9+i*3,7+i*3]);ctx.stroke()}const g=ctx.createRadialGradient(cx,cy,2,cx,cy,r*1.22);g.addColorStop(0,'#fff');g.addColorStop(.17,'rgba(210,185,255,.98)');g.addColorStop(.42,'rgba(184,108,255,.62)');g.addColorStop(.72,'rgba(0,220,255,.26)');g.addColorStop(1,'transparent');ctx.fillStyle=g;ctx.shadowBlur=r*.36;ctx.shadowColor='#b86cff';ctx.beginPath();ctx.arc(cx,cy,r,0,Math.PI*2);ctx.fill();ctx.restore();blink++;if(blink>145+Math.random()*55){isBlink=true;if(blink>155){isBlink=false;blink=0}}mouth=talking?Math.sin(phase*16)*5+9:4;ctx.save();ctx.translate(cx,cy);ctx.scale(r/105,r/105);ctx.fillStyle='#000103';const eh=isBlink?2:10;ctx.beginPath();ctx.ellipse(-20,-10,8,eh,0,0,Math.PI*2);ctx.ellipse(20,-10,8,eh,0,0,Math.PI*2);ctx.fill();if(!isBlink){ctx.fillStyle='#fff';ctx.beginPath();ctx.arc(-22,-12,2.5,0,Math.PI*2);ctx.arc(18,-12,2.5,0,Math.PI*2);ctx.fill()}ctx.strokeStyle='#000103';ctx.lineWidth=3.5;if(talking){ctx.fillStyle='#000103';ctx.beginPath();ctx.ellipse(0,6,7.5,mouth*.75,0,0,Math.PI*2);ctx.fill()}else{ctx.beginPath();ctx.arc(0,4,11,0,Math.PI);ctx.stroke()}ctx.restore();requestAnimationFrame(draw)}draw();
function setState(t){stateEl.textContent=t}
function speak(text){return new Promise(resolve=>{if(!('speechSynthesis'in window)){resolve();return}speechSynthesis.cancel();const u=new SpeechSynthesisUtterance(text);u.lang='es-ES';u.rate=1.03;u.pitch=.9;u.onstart=()=>{talking=true;setState('K.H.A.O.S. // HABLANDO')};u.onend=()=>{talking=false;resolve()};u.onerror=()=>{talking=false;resolve()};speechSynthesis.speak(u)})}
async function sendLive(text){if(!text||busy||!live)return;busy=true;shouldListen=false;try{if(recognition){try{recognition.stop()}catch(e){}}speechSynthesis.cancel();setState('K.H.A.O.S. // PENSANDO');const r=await fetch('/api/comando',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({comando:text,flotante:true,live:true})});const d=await r.json();if(d.respuesta)await speak(d.respuesta);else setState('K.H.A.O.S. // LIVE');}catch(e){setState('K.H.A.O.S. // ERROR DE ENLACE')}finally{busy=false;if(live){shouldListen=true;setTimeout(startListening,180)}}}
function buildRecognition(){const SR=window.SpeechRecognition||window.webkitSpeechRecognition;if(!SR){setState('RECONOCIMIENTO DE VOZ NO DISPONIBLE');return null}const r=new SR();r.lang='es-ES';r.continuous=false;r.interimResults=false;r.maxAlternatives=1;r.onstart=()=>{listening=true;setState('K.H.A.O.S. // ESCUCHANDO')};r.onresult=e=>{const t=(e.results[0]?.[0]?.transcript||'').trim();listening=false;if(t)sendLive(t)};r.onerror=e=>{listening=false;if(e.error==='not-allowed'||e.error==='service-not-allowed'){setState('MICRÓFONO BLOQUEADO — TOCÁ LA CARA')}else if(e.error!=='aborted'){setState('K.H.A.O.S. // REINTENTANDO')}};r.onend=()=>{listening=false;if(live&&shouldListen&&!busy)setTimeout(startListening,250)};return r}
recognition=buildRecognition();
function startListening(){if(!live||busy||listening||!recognition)return;try{recognition.start()}catch(e){setState('TOCÁ LA CARA PARA ACTIVAR LIVE')}}
wrap.addEventListener('click',()=>{if(!recognition)return;if(talking){speechSynthesis.cancel();talking=false}shouldListen=true;startListening()});
window.addEventListener('beforeunload',()=>{live=false;shouldListen=false;try{recognition&&recognition.stop()}catch(e){}speechSynthesis.cancel()});
setState('K.H.A.O.S. // LIVE');
setTimeout(async()=>{await speak('Modo Live activado, señor Matías. Estoy acá. Hablemos.');if(live){shouldListen=true;setTimeout(startListening,180)}},500);
</script></body></html>
"""

# ===========================================================================
# K.H.A.O.S. FLOATING V2 // DESKTOP OVERLAY
# Ventana APP sin barra/URL, siempre encima y pegada al escritorio.
# El dashboard principal permanece intacto.
# ===========================================================================
FLOATING_HTML = r"""
<!doctype html>
<html lang="es">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>K.H.A.O.S. // Floating</title>
<style>
:root{--c:#67e8ff;--a:#ff9900}
*{box-sizing:border-box}html,body{margin:0;width:100%;height:100%;overflow:hidden;background:#02070c;color:#fff;font-family:Arial,sans-serif}
body{display:flex;align-items:center;justify-content:center}
#wrap{position:relative;width:100vw;height:100vh;display:flex;align-items:center;justify-content:center}
#face{width:100%;height:100%;display:block}
#hud{position:absolute;left:10px;right:10px;bottom:9px;display:flex;gap:6px;opacity:.0;transition:.25s;pointer-events:none}
#wrap:hover #hud{opacity:1;pointer-events:auto}
#cmd{flex:1;min-width:0;background:rgba(2,10,17,.88);border:1px solid rgba(103,232,255,.35);color:#eaffff;border-radius:8px;padding:8px 9px;font-size:11px;outline:none}
#send,#mic{border:1px solid rgba(103,232,255,.35);background:rgba(103,232,255,.08);color:#b9f6ff;border-radius:8px;padding:0 10px;cursor:pointer}#mic{min-width:38px}#mic.active{background:rgba(255,153,0,.25);border-color:#ff9900;box-shadow:0 0 14px rgba(255,153,0,.55)}
#label{position:absolute;top:8px;left:10px;font:700 8px/1 monospace;letter-spacing:2px;color:rgba(103,232,255,.6);text-shadow:0 0 10px #00eaff;opacity:.0;transition:.25s}
#wrap:hover #label{opacity:1}
</style>
</head>
<body>
<div id="wrap"><canvas id="face"></canvas><div id="label">K.H.A.O.S. // FLOAT</div><div id="hud"><button id="mic" title="Hablar">🎤</button><input id="cmd" placeholder="Escribí una orden..."><button id="send">↵</button></div></div>
<script>
const canvas=document.getElementById('face'),ctx=canvas.getContext('2d'),cmd=document.getElementById('cmd');
let phase=0,blink=0,isBlink=false,talking=false,mouth=4;
function resize(){const d=devicePixelRatio||1;canvas.width=innerWidth*d;canvas.height=innerHeight*d;canvas.style.width=innerWidth+'px';canvas.style.height=innerHeight+'px';ctx.setTransform(d,0,0,d,0,0)}
addEventListener('resize',resize);resize();
function draw(){
 const w=innerWidth,h=innerHeight,cx=w/2,cy=h/2,r=Math.min(w,h)*.38;ctx.clearRect(0,0,w,h);phase+=.025;
 ctx.save();ctx.globalCompositeOperation='lighter';
 for(let i=0;i<3;i++){ctx.beginPath();ctx.arc(cx,cy,r*(1.55-i*.18),phase*(i%2?-.8:1),phase*(i%2?-.8:1)+Math.PI*(1.2-i*.15));ctx.strokeStyle=i===1?'rgba(255,153,0,.65)':'rgba(0,240,255,.55)';ctx.lineWidth=Math.max(1.5,r*.018);ctx.setLineDash([10+i*4,8+i*3]);ctx.stroke()}
 const g=ctx.createRadialGradient(cx,cy,2,cx,cy,r*1.25);g.addColorStop(0,'#fff');g.addColorStop(.18,'rgba(255,175,0,.98)');g.addColorStop(.52,'rgba(0,220,255,.5)');g.addColorStop(1,'transparent');ctx.fillStyle=g;ctx.shadowBlur=r*.35;ctx.shadowColor='#00eaff';ctx.beginPath();ctx.arc(cx,cy,r,0,Math.PI*2);ctx.fill();ctx.restore();
 blink++;if(blink>145+Math.random()*55){isBlink=true;if(blink>155){isBlink=false;blink=0}}
 if(talking)mouth=Math.sin(phase*16)*5+9;else mouth=4;
 ctx.save();ctx.translate(cx,cy);ctx.scale(r/105,r/105);ctx.fillStyle='#000103';
 const eh=isBlink?2:10;ctx.beginPath();ctx.ellipse(-20,-10,8,eh,0,0,Math.PI*2);ctx.ellipse(20,-10,8,eh,0,0,Math.PI*2);ctx.fill();
 if(!isBlink){ctx.fillStyle='#fff';ctx.beginPath();ctx.arc(-22,-12,2.5,0,Math.PI*2);ctx.arc(18,-12,2.5,0,Math.PI*2);ctx.fill()}
 ctx.strokeStyle='#000103';ctx.lineWidth=3.5;if(talking){ctx.fillStyle='#000103';ctx.beginPath();ctx.ellipse(0,6,7.5,mouth*.75,0,0,Math.PI*2);ctx.fill()}else{ctx.beginPath();ctx.arc(0,4,11,0,Math.PI);ctx.stroke()}ctx.restore();requestAnimationFrame(draw)
}
draw();
async function send(){const q=cmd.value.trim();if(!q)return;cmd.value='';try{const r=await fetch('/api/comando',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({comando:q,flotante:true})});const d=await r.json();if(d.respuesta){talking=true;speechSynthesis.cancel();const u=new SpeechSynthesisUtterance(d.respuesta);u.lang='es-ES';u.rate=1.02;u.pitch=.9;u.onend=()=>talking=false;speechSynthesis.speak(u)}}catch(e){}}
const mic=document.getElementById('mic');let recognition=null,listening=false;
if('SpeechRecognition' in window || 'webkitSpeechRecognition' in window){const SR=window.SpeechRecognition||window.webkitSpeechRecognition;recognition=new SR();recognition.lang='es-ES';recognition.continuous=false;recognition.interimResults=false;recognition.onstart=()=>{listening=true;mic.classList.add('active');cmd.placeholder='Te escucho...'};recognition.onresult=e=>{cmd.value=e.results[0][0].transcript;send()};recognition.onerror=()=>{listening=false;mic.classList.remove('active');cmd.placeholder='Escribí o hablá una orden...'};recognition.onend=()=>{listening=false;mic.classList.remove('active');cmd.placeholder='Escribí o hablá una orden...'}}else{mic.style.display='none'}
mic.onclick=()=>{if(!recognition)return;if(listening){try{recognition.stop()}catch(e){}}else{try{recognition.start()}catch(e){}}};
document.getElementById('send').onclick=send;cmd.addEventListener('keydown',e=>{if(e.key==='Enter')send()});
</script></body></html>
"""

@app.route("/live")
def live_view():
    return Response(LIVE_HTML, mimetype="text/html; charset=utf-8")

@app.route("/api/live/open", methods=["POST"])
def live_open_api():
    respuesta=live_browser_command()
    return jsonify({"ok":respuesta.startswith("Modo Live activado"),"respuesta":respuesta})

@app.route("/floating")
def floating_view():
    return Response(FLOATING_HTML, mimetype="text/html; charset=utf-8")

def _configure_floating_window():
    """Busca la ventana APP recién creada y la convierte en overlay del escritorio."""
    try:
        for _ in range(50):
            ids=[]
            # Primero xdotool si está instalado; si no, wmctrl funciona igual para X11.
            if shutil.which('xdotool'):
                try:
                    ids=subprocess.check_output(
                        ['xdotool','search','--name','K.H.A.O.S. // Floating'],
                        stderr=subprocess.DEVNULL, text=True
                    ).strip().splitlines()
                except Exception:
                    ids=[]
            if not ids and shutil.which('wmctrl'):
                try:
                    rows=subprocess.check_output(['wmctrl','-l'],text=True,stderr=subprocess.DEVNULL).splitlines()
                    for row in rows:
                        parts=row.split(None,3)
                        if len(parts)==4 and 'K.H.A.O.S. // Floating' in parts[3]:
                            ids.append(parts[0])
                except Exception:
                    ids=[]
            if ids:
                wid=ids[-1]
                if shutil.which('wmctrl'):
                    subprocess.run(['wmctrl','-i','-r',wid,'-b','add,above,sticky,skip_taskbar'],
                                   stdout=subprocess.DEVNULL,stderr=subprocess.DEVNULL)
                    sw,sh=1366,768
                    try:
                        if shutil.which('xdotool'):
                            geom=subprocess.check_output(['xdotool','getdisplaygeometry'],text=True).split()
                            sw,sh=int(geom[0]),int(geom[1])
                        elif shutil.which('xrandr'):
                            info=subprocess.check_output(['xrandr','--current'],text=True,stderr=subprocess.DEVNULL)
                            m=re.search(r'current\s+(\d+)\s+x\s+(\d+)',info)
                            if m: sw,sh=int(m.group(1)),int(m.group(2))
                    except Exception:
                        pass
                    subprocess.run(['wmctrl','-i','-r',wid,'-e',f'0,{max(0,sw-344)},{max(0,sh-414)},320,320'],
                                   stdout=subprocess.DEVNULL,stderr=subprocess.DEVNULL)
                # X11: elimina decoraciones (título, botones y marco).
                if shutil.which('xprop'):
                    subprocess.run(['xprop','-id',wid,'-f','_MOTIF_WM_HINTS','32c','-set','_MOTIF_WM_HINTS','2, 0, 0, 0, 0'],
                                   stdout=subprocess.DEVNULL,stderr=subprocess.DEVNULL)
                return True
            time.sleep(0.15)
    except Exception:
        pass
    return False

def live_browser_command():
    """Abre Modo Live como una ventana APP sin barra y siempre encima."""
    try:
        url="http://127.0.0.1:5000/live"
        browser=(shutil.which('microsoft-edge-stable') or shutil.which('microsoft-edge') or
                 shutil.which('google-chrome') or shutil.which('chromium') or shutil.which('chromium-browser'))
        if not browser:
            return "No encontré un navegador compatible para Modo Live."
        args=[browser,'--app='+url,'--window-size=340,340','--disable-session-crashed-bubble','--disable-infobars','--no-first-run','--no-default-browser-check']
        subprocess.Popen(args,start_new_session=True,stdout=subprocess.DEVNULL,stderr=subprocess.DEVNULL)
        threading.Thread(target=_configure_live_window,daemon=True).start()
        return "Modo Live activado. Estoy lista para conversar con vos."
    except Exception as e:
        return f"No pude activar Modo Live: {e}"

def _configure_live_window():
    try:
        for _ in range(50):
            ids=[]
            if shutil.which('xdotool'):
                try: ids=subprocess.check_output(['xdotool','search','--name','K.H.A.O.S. // Live'],stderr=subprocess.DEVNULL,text=True).strip().splitlines()
                except Exception: ids=[]
            if not ids and shutil.which('wmctrl'):
                try:
                    for row in subprocess.check_output(['wmctrl','-l'],text=True,stderr=subprocess.DEVNULL).splitlines():
                        parts=row.split(None,3)
                        if len(parts)==4 and 'K.H.A.O.S. // Live' in parts[3]: ids.append(parts[0])
                except Exception: ids=[]
            if ids:
                wid=ids[-1]
                if shutil.which('wmctrl'):
                    subprocess.run(['wmctrl','-i','-r',wid,'-b','add,above,sticky,skip_taskbar'],stdout=subprocess.DEVNULL,stderr=subprocess.DEVNULL)
                    sw,sh=1366,768
                    try:
                        if shutil.which('xdotool'):
                            geom=subprocess.check_output(['xdotool','getdisplaygeometry'],text=True).split(); sw,sh=int(geom[0]),int(geom[1])
                    except Exception: pass
                    subprocess.run(['wmctrl','-i','-r',wid,'-e',f'0,{max(0,sw-364)},{max(0,sh-434)},340,340'],stdout=subprocess.DEVNULL,stderr=subprocess.DEVNULL)
                if shutil.which('xprop'):
                    subprocess.run(['xprop','-id',wid,'-f','_MOTIF_WM_HINTS','32c','-set','_MOTIF_WM_HINTS','2, 0, 0, 0, 0'],stdout=subprocess.DEVNULL,stderr=subprocess.DEVNULL)
                return True
            time.sleep(.15)
    except Exception: pass
    return False

def floating_browser_command():
    """Abre K.H.A.O.S. como overlay: sin URL/barra y siempre encima."""
    try:
        url="http://127.0.0.1:5000/floating"
        browser=(shutil.which('microsoft-edge-stable') or shutil.which('microsoft-edge') or
                 shutil.which('google-chrome') or shutil.which('chromium') or shutil.which('chromium-browser'))
        if not browser:
            return "No encontré Microsoft Edge, Google Chrome o Chromium para abrir la ventana flotante."
        args=[browser,'--app='+url,'--window-size=320,320',
              '--disable-session-crashed-bubble','--disable-infobars',
              '--no-first-run','--no-default-browser-check']
        subprocess.Popen(args,start_new_session=True,
                         stdout=subprocess.DEVNULL,stderr=subprocess.DEVNULL)
        threading.Thread(target=_configure_floating_window,daemon=True).start()
        return "Modo flotante activado. K.H.A.O.S. quedó sin barra de navegador y siempre encima."
    except Exception as e:
        return f"No pude activar el modo flotante: {e}"

@app.route("/api/floating/open", methods=["POST"])
def floating_open_api():
    respuesta=floating_browser_command()
    return jsonify({"ok":respuesta.startswith("Modo flotante"),"respuesta":respuesta})

@app.route("/control")
def control_remoto():
    return render_template_string(HTML_CONTROL_REMOTO)

@app.route("/video_feed")
def video_feed():
    return Response(
        generar_flujo_video(),
        mimetype="multipart/x-mixed-replace; boundary=frame",
    )


@app.route("/api/system")
def api_system():
    return jsonify(obtener_datos_sistema_json())

@app.route("/api/files")
def api_files():
    return jsonify(listar_archivos_seguro(request.args.get("path", "~")))

@app.route("/api/launch", methods=["POST"])
def api_launch():
    data=request.get_json(silent=True) or {}
    return jsonify({"respuesta": lanzar_aplicacion(data.get("app", ""))})

@app.route("/api/timer", methods=["POST"])
def api_timer():
    data=request.get_json(silent=True) or {}
    return jsonify({"respuesta": crear_temporizador_seguro(data.get("seconds", 60), data.get("label", "Temporizador"))})

@app.route("/api/plugins")
def api_plugins():
    return jsonify({"plugins": listar_plugins(), "directory": str(PLUGIN_DIR)})

@app.route("/api/safe-mode", methods=["GET", "POST"])
def api_safe_mode():
    global SAFE_MODE
    if request.method == "POST":
        data=request.get_json(silent=True) or {}
        SAFE_MODE=bool(data.get("enabled", True))
    return jsonify({"enabled": SAFE_MODE})

@app.route("/api/pc/apps")
def api_pc_apps():
    known=["terminal","vscode","archivos","explorador","calculadora","navegador","edge","spotify","youtube"]
    return jsonify({"apps":[{"name":n,"available":True} for n in known if n in ["terminal","vscode","archivos","explorador","calculadora","navegador","edge","spotify","youtube"]]})

@app.route("/api/pc/processes")
def api_pc_processes():
    return jsonify({"processes":pc_processes()})

@app.route("/api/pc/action", methods=["POST"])
def api_pc_action():
    data=request.get_json(silent=True) or {}
    confirmed=bool(data.get("confirm",False))
    result=ejecutar_pc_action(data,confirmed)
    return jsonify(result)

@app.route("/api/pc/confirm", methods=["POST"])
def api_pc_confirm():
    global pending_pc_action
    with pc_action_lock:
        action=pending_pc_action
        pending_pc_action=None
    if not action: return jsonify({"ok":False,"respuesta":"No hay ninguna acción pendiente."})
    return jsonify(ejecutar_pc_action(action,True))


# ===========================================================================
# ORION // CONTROL NATURAL DE PC
# Ejecuta órdenes habladas/escritas sin obligar al operador a usar botones.
# ===========================================================================
def orion_fijar_volumen(porcentaje):
    try:
        nivel=max(0,min(100,int(porcentaje)))
        r=subprocess.run(["pactl","set-sink-volume","@DEFAULT_SINK@",f"{nivel}%"],capture_output=True,text=True)
        if r.returncode != 0:
            return f"No pude fijar el volumen al {nivel}%."
        return f"Volumen establecido al {nivel}%."
    except Exception as e:
        return f"No pude cambiar el volumen: {e}"


def orion_control_musica(accion="pausa"):
    """Controla la reproducción aunque K.H.A.O.S. esté en modo flotante."""
    comandos = []
    if accion == "pausa":
        comandos = [["playerctl", "play-pause"], ["xdotool", "key", "XF86AudioPlay"]]
    elif accion == "detener":
        comandos = [["playerctl", "stop"]]
    elif accion == "reanudar":
        comandos = [["playerctl", "play"], ["xdotool", "key", "XF86AudioPlay"]]

    for cmd in comandos:
        try:
            r = subprocess.run(cmd, capture_output=True, text=True, timeout=4)
            if r.returncode == 0:
                if accion == "pausa":
                    return "Música pausada."
                if accion == "detener":
                    return "Música detenida."
                return "Música reanudada."
        except Exception:
            pass
    return "No encontré un reproductor multimedia controlable."


def orion_siguiente_musica():
    """Cambia a la siguiente pista mediante MPRIS/playerctl o tecla multimedia."""
    for cmd in (["playerctl","next"],["xdotool","key","XF86AudioNext"]):
        try:
            r=subprocess.run(cmd,capture_output=True,text=True,timeout=4)
            if r.returncode == 0:
                return "Listo. Cambié la música y pasé a la siguiente canción."
        except Exception:
            pass
    return "No encontré un reproductor multimedia controlable. Podés decirme 'cambiá la música por [canción]' y te pongo otra directamente."


def orion_cambiar_musica_por(busqueda):
    """Cambia la canción por otra indicada por voz. Detiene la reproducción actual
    mediante MPRIS/tecla multimedia y reproduce la nueva búsqueda directamente."""
    busqueda = (busqueda or "").strip()
    if not busqueda:
        return "Decime qué canción querés poner."

    # Primero detenemos la pista actual si el reproductor expone MPRIS.
    detenido = False
    for cmd in (["playerctl", "stop"], ["xdotool", "key", "XF86AudioStop"]):
        try:
            r = subprocess.run(cmd, capture_output=True, text=True, timeout=4)
            if r.returncode == 0:
                detenido = True
                break
        except Exception:
            pass

    # Reproduce la nueva canción con el mismo motor directo que ya usa K.H.A.O.S.
    nueva = reproducir_musica_directa(busqueda)
    if nueva.startswith("Reproduciendo música directamente"):
        return f"Listo. Cambié la música por {busqueda}."
    if detenido:
        return nueva
    return nueva


def orion_limpiar_innecesarios():
    """Limpieza conservadora: solo cachés temporales del usuario, nunca carpetas críticas."""
    candidatos=[Path.home()/".cache", Path("/tmp")]
    eliminados=0
    liberados=0
    for base in candidatos:
        if not base.exists():
            continue
        try:
            if base == Path("/tmp"):
                entradas=list(base.iterdir())
                for item in entradas:
                    # No borrar todo /tmp: solo elementos claramente temporales del usuario.
                    nombre=item.name.lower()
                    if not any(x in nombre for x in ("tmp","cache","chromium","playwright")):
                        continue
                    try:
                        tam=item.stat().st_size if item.is_file() else 0
                        if item.is_dir(): shutil.rmtree(item,ignore_errors=True)
                        else: item.unlink(missing_ok=True)
                        eliminados += 1; liberados += tam
                    except Exception:
                        pass
            else:
                for item in list(base.iterdir()):
                    try:
                        tam=item.stat().st_size if item.is_file() else 0
                        if item.is_dir(): shutil.rmtree(item,ignore_errors=True)
                        else: item.unlink(missing_ok=True)
                        eliminados += 1; liberados += tam
                    except Exception:
                        pass
        except Exception:
            pass
    return f"Limpieza terminada: {eliminados} elementos temporales/cachés tratados, aproximadamente {liberados/1024/1024:.1f} MB liberados." if eliminados else "No encontré cachés temporales seguros para limpiar."


def orion_buscar_youtube(consulta):
    consulta=consulta.strip()
    if not consulta:
        return "Decime qué canción o artista querés buscar."
    url="https://www.youtube.com/results?search_query="+urllib.parse.quote_plus(consulta)
    try:
        subprocess.Popen(["microsoft-edge-stable",url],start_new_session=True)
        return f"Abriendo YouTube y buscando {consulta}."
    except Exception as e:
        return f"No pude abrir YouTube: {e}"


# ===========================================================================
# K.H.A.O.S. V7 // SAFE STORAGE, DIAGNOSTICS, MEMORY, STATS & AUTOMATION
# Añadidos modulares. El núcleo original permanece intacto.
# ===========================================================================

KHAOS_STATS = {
    "commands": 0,
    "successful_commands": 0,
    "errors": 0,
    "last_command": "",
    "started_at": time.time(),
    "cleanup_runs": 0,
    "cleanup_bytes": 0,
}

KHAOS_ACTIVITY = []
KHAOS_ACTIVITY_LOCK = threading.Lock()

def khaos_log(tag, text, kind=""):
    entry = {
        "time": datetime.now().strftime("%H:%M:%S"),
        "tag": str(tag)[:24],
        "text": str(text)[:500],
        "kind": kind,
    }
    with KHAOS_ACTIVITY_LOCK:
        KHAOS_ACTIVITY.insert(0, entry)
        del KHAOS_ACTIVITY[80:]
    return entry

def khaos_human_bytes(n):
    try:
        n = float(n)
    except Exception:
        n = 0
    units = ["B", "KB", "MB", "GB", "TB"]
    for u in units:
        if n < 1024 or u == units[-1]:
            return f"{n:.1f} {u}"
        n /= 1024
    return "0 B"

def khaos_dir_size(path, max_items=50000):
    total = 0
    count = 0
    path = Path(path)
    try:
        if path.is_file():
            return path.stat().st_size, 1
        for root, dirs, files in os.walk(path):
            # Never follow links: prevents escaping the intended tree.
            dirs[:] = [d for d in dirs if not (Path(root) / d).is_symlink()]
            for name in files:
                if count >= max_items:
                    return total, count
                fp = Path(root) / name
                try:
                    if fp.is_symlink():
                        continue
                    total += fp.stat().st_size
                    count += 1
                except OSError:
                    pass
    except OSError:
        pass
    return total, count

SAFE_CLEANUP_ROOTS = [
    Path.home() / ".cache",
    Path.home() / ".local" / "share" / "Trash" / "files",
    Path("/tmp"),
]

def khaos_cleanup_candidates():
    """Only well-known temporary/cache locations. Never scans arbitrary user files."""
    found = []
    seen = set()
    now = time.time()
    for root in SAFE_CLEANUP_ROOTS:
        try:
            root = root.resolve()
        except Exception:
            continue
        if not root.exists() or not root.is_dir() or pc_is_protected(root):
            continue

        # Trash can be completely inspected; cache/tmp are inspected recursively.
        size, count = khaos_dir_size(root)
        if size <= 0:
            continue

        age_hint = None
        try:
            age_hint = max(0, int((now - root.stat().st_mtime) / 86400))
        except OSError:
            pass

        found.append({
            "path": str(root),
            "size": size,
            "size_human": khaos_human_bytes(size),
            "files": count,
            "age_days_hint": age_hint,
        })
        seen.add(str(root))

    # Browser/app cache directories are already inside ~/.cache, but expose
    # large top-level cache folders separately for a useful visual report.
    cache = (Path.home() / ".cache")
    if cache.exists():
        try:
            for child in cache.iterdir():
                if child.is_symlink() or not child.is_dir():
                    continue
                size, count = khaos_dir_size(child, max_items=20000)
                if size >= 256 * 1024 * 1024:
                    found.append({
                        "path": str(child),
                        "size": size,
                        "size_human": khaos_human_bytes(size),
                        "files": count,
                        "age_days_hint": None,
                    })
        except OSError:
            pass

    # Avoid double-counting child cache folders in the total estimate.
    roots_only = []
    for item in found:
        path = Path(item["path"])
        if any(path != Path(other["path"]) and Path(other["path"]) in path.parents for other in found):
            continue
        roots_only.append(item)
    return sorted(roots_only, key=lambda x: x["size"], reverse=True)

def khaos_cleanup_preview():
    candidates = khaos_cleanup_candidates()
    total = sum(x["size"] for x in candidates)
    return {
        "ok": True,
        "candidates": candidates[:30],
        "total_bytes": total,
        "total_human": khaos_human_bytes(total),
        "message": (
            f"Encontré aproximadamente {khaos_human_bytes(total)} en cachés/temporales seguros."
            if total else
            "No encontré espacio recuperable en las ubicaciones temporales seguras."
        ),
    }

def khaos_remove_tree_contents(root):
    """Remove contents only, never the root itself. Returns bytes/items actually removed."""
    root = Path(root).resolve()
    if not root.exists() or not root.is_dir():
        return 0, 0, []
    if pc_is_protected(root):
        return 0, 0, [f"Bloqueado: {root}"]

    total = 0
    items = 0
    errors = []
    try:
        entries = list(root.iterdir())
    except OSError as e:
        return 0, 0, [f"{root}: {e}"]

    for item in entries:
        try:
            if item.is_symlink():
                # Never follow/delete links that could point elsewhere.
                item.unlink(missing_ok=True)
                items += 1
                continue
            size, _ = khaos_dir_size(item, max_items=30000)
            if item.is_dir():
                shutil.rmtree(item)
            else:
                item.unlink()
            total += size
            items += 1
        except Exception as e:
            errors.append(f"{item}: {e}")
    return total, items, errors

def khaos_safe_cleanup(confirm=False, requested_gb=None):
    """
    Destructive cleanup is explicit and restricted to temporary/cache roots.
    It never deletes /, /usr, /etc, /var, /home itself or arbitrary documents.
    """
    preview = khaos_cleanup_preview()
    if not confirm:
        target_note = f" (objetivo solicitado: {requested_gb} GB)" if requested_gb else ""
        return {
            "ok": True,
            "needs_confirmation": True,
            "preview": preview,
            "respuesta": (
                f"Detecté {preview['total_human']} recuperables en ubicaciones temporales seguras"
                f"{target_note}. No borré nada. Decime CONFIRMAR LIMPIEZA para continuar."
            ),
        }

    total = 0
    items = 0
    errors = []
    for root in SAFE_CLEANUP_ROOTS:
        b, n, e = khaos_remove_tree_contents(root)
        total += b
        items += n
        errors.extend(e)

    KHAOS_STATS["cleanup_runs"] += 1
    KHAOS_STATS["cleanup_bytes"] += total
    khaos_log("CLEAN", f"{khaos_human_bytes(total)} liberados / {items} elementos", "ok")

    return {
        "ok": True,
        "needs_confirmation": False,
        "bytes_freed": total,
        "freed_human": khaos_human_bytes(total),
        "items": items,
        "errors": errors[:20],
        "respuesta": (
            f"Limpieza segura terminada: liberé aproximadamente {khaos_human_bytes(total)} "
            f"y traté {items} elementos temporales/cachés. "
            + (f"Algunos elementos estaban en uso y no se pudieron tocar: {len(errors)}." if errors else "")
        ),
    }

def khaos_storage_overview():
    try:
        disk = psutil.disk_usage(str(Path.home().anchor or "/"))
        return {
            "total": disk.total,
            "used": disk.used,
            "free": disk.free,
            "percent": disk.percent,
            "total_human": khaos_human_bytes(disk.total),
            "used_human": khaos_human_bytes(disk.used),
            "free_human": khaos_human_bytes(disk.free),
        }
    except Exception as e:
        return {"error": str(e)}

def khaos_diagnostics():
    data = obtener_datos_sistema_json()
    checks = []
    checks.append(("CPU", "OK" if data.get("cpu", 100) < 90 else "HIGH"))
    checks.append(("RAM", "OK" if data.get("ram", 100) < 90 else "HIGH"))
    checks.append(("DISK", "OK" if data.get("disk", 100) < 90 else "HIGH"))
    checks.append(("NETWORK", "OK" if data.get("ip") and data.get("ip") != "127.0.0.1" else "CHECK"))
    checks.append(("GEMINI", "OK" if client is not None else "MISSING KEY"))
    checks.append(("SAFE MODE", "ON" if SAFE_MODE else "OFF"))
    return {
        "ok": True,
        "checks": [{"name": n, "status": v} for n, v in checks],
        "system": data,
        "storage": khaos_storage_overview(),
    }

def khaos_memory():
    with KHAOS_ACTIVITY_LOCK:
        activity = list(KHAOS_ACTIVITY[:20])
    return {
        "conversation": historial_conversacion[-MAX_HISTORIAL:],
        "tasks": task_manager.get_tasks(),
        "activity": activity,
    }

def khaos_stats():
    uptime = max(0, time.time() - KHAOS_STATS["started_at"])
    return {
        **KHAOS_STATS,
        "uptime_seconds": int(uptime),
        "uptime_human": f"{int(uptime // 3600)}h {int((uptime % 3600) // 60)}m {int(uptime % 60)}s",
    }

def khaos_network_info():
    return {
        "ip": obtener_ip_local(),
        "hostname": socket.gethostname(),
        "interfaces": sorted([x for x in psutil.net_if_addrs().keys()]),
        "connections": len(psutil.net_connections(kind="inet")),
    }

def khaos_set_gaming_mode(enabled):
    # Safe UI/diagnostic mode only: no killing processes, no kernel tweaks.
    khaos_log("MODE", "Gaming Mode ON" if enabled else "Gaming Mode OFF")
    return {
        "enabled": bool(enabled),
        "respuesta": (
            "Gaming Mode activado. Voy a priorizar información y accesos rápidos; "
            "no voy a matar procesos ni tocar configuraciones críticas."
            if enabled else
            "Gaming Mode desactivado."
        )
    }

@app.route("/api/v7/cleanup/preview")
def api_v7_cleanup_preview():
    return jsonify(khaos_cleanup_preview())

@app.route("/api/v7/cleanup", methods=["POST"])
def api_v7_cleanup():
    data = request.get_json(silent=True) or {}
    return jsonify(khaos_safe_cleanup(bool(data.get("confirm", False)), data.get("requested_gb")))

@app.route("/api/v7/diagnostics")
def api_v7_diagnostics():
    return jsonify(khaos_diagnostics())

@app.route("/api/v7/storage")
def api_v7_storage():
    return jsonify(khaos_storage_overview())

@app.route("/api/v7/memory")
def api_v7_memory():
    return jsonify(khaos_memory())

@app.route("/api/v7/stats")
def api_v7_stats():
    return jsonify(khaos_stats())

@app.route("/api/v7/activity")
def api_v7_activity():
    with KHAOS_ACTIVITY_LOCK:
        return jsonify({"activity": list(KHAOS_ACTIVITY[:30])})

@app.route("/api/v7/network")
def api_v7_network():
    return jsonify(khaos_network_info())

@app.route("/api/v7/gaming", methods=["POST"])
def api_v7_gaming():
    data = request.get_json(silent=True) or {}
    return jsonify(khaos_set_gaming_mode(bool(data.get("enabled", False))))

@app.route("/api/v7/search")
def api_v7_search():
    q = request.args.get("q", "").strip().lower()
    if not q:
        return jsonify({"results": []})
    results = []
    for item in ["CORE", "SYSTEM", "VISION", "NETWORK", "MEDIA", "TOOLS", "FILES",
                 "SETTINGS", "MEMORY", "DIAGNOSTICS", "SECURITY", "GAMING MODE",
                 "CLEANUP", "ACTIVITY LOG", "PLUGIN CENTER", "TASKS", "STATS"]:
        if q in item.lower():
            results.append({"type": "module", "name": item})
    for plugin in listar_plugins():
        if q in plugin.lower():
            results.append({"type": "plugin", "name": plugin})
    for task in historial_conversacion[-MAX_HISTORIAL:]:
        text = f"{task.get('q','')} {task.get('r','')}"
        if q in text.lower():
            results.append({"type": "memory", "name": task.get("q", "")[:100]})
    return jsonify({"results": results[:30]})

@app.route("/api/comando", methods=["POST"])
def procesar_comando():
    global camara_activa, cap
    data = request.get_json() or {}
    comando = data.get("comando", "").lower()
    modo_flotante = bool(data.get("flotante", False))
    modo_live = bool(data.get("live", False))
    respuesta_texto = ""
    KHAOS_STATS["commands"] += 1
    KHAOS_STATS["last_command"] = comando[:300]
    khaos_log("CMD", comando[:300])

    # ===== ACTIVATE LIVE MODE BY VOICE =====
    if any(x in comando for x in ["modo live", "modo en vivo", "conversación live", "modo conversación", "activar live"]):
        respuesta_texto=live_browser_command()
        if not modo_live:
            threading.Thread(target=hablar_servidor,args=(respuesta_texto,)).start()
        return jsonify({"respuesta":respuesta_texto,"camara_activa":camara_activa,"live":True})

    # ===== LIVE MODE =====
    # En Live, el navegador se encarga de escuchar y hablar.
    # Evitamos la voz del servidor para que no haya doble audio.
    if modo_live and comando in {"__live_ping__", "live ping"}:
        return jsonify({"respuesta":"Estoy lista, Matías. Te escucho.","camara_activa":camara_activa,"live":True})

    # ===== FLOATING MODE =====
    if any(x in comando for x in [
        "modo flotante", "ventana flotante", "modo floating",
        "flotante khaos", "flota khaos", "saca solo la cara",
        "mostrame solo la cara", "muéstrame solo la cara",
        "mostrame tu cara", "muéstrame tu cara"
    ]):
        respuesta_texto=floating_browser_command()
        if not modo_live: threading.Thread(target=hablar_servidor,args=(respuesta_texto,)).start()
        return jsonify({"respuesta":respuesta_texto,"camara_activa":camara_activa,"floating":True})

    # ===== V8 TITAN INTENT ROUTER =====
    v8_kind, v8_value = v8_intent(comando)
    if v8_kind == "cleanup":
        preview=khaos_safe_cleanup(False); respuesta_texto=preview.get("respuesta","Analicé la limpieza segura.")
        if not modo_live: threading.Thread(target=hablar_servidor,args=(respuesta_texto,)).start()
        return jsonify({"respuesta":respuesta_texto,"camara_activa":camara_activa,"v8_preview":preview.get("preview",{})})
    if v8_kind == "cleanup_confirm":
        result=khaos_safe_cleanup(True); respuesta_texto=result.get("respuesta","Limpieza terminada.")
        if not modo_live: threading.Thread(target=hablar_servidor,args=(respuesta_texto,)).start()
        return jsonify({"respuesta":respuesta_texto,"camara_activa":camara_activa,"cleanup":result})
    if v8_kind == "target":
        preview=khaos_safe_cleanup(False); safe_gb=preview.get("preview",{}).get("total_bytes",0)/1024**3; target=float(v8_value)
        respuesta_texto=(f"Detecté aproximadamente {safe_gb:.1f} GB recuperables en ubicaciones temporales seguras. No borré nada. Decime CONFIRMAR LIMPIEZA para continuar." if safe_gb>=target else f"Pediste liberar {target:.1f} GB. En ubicaciones temporales seguras detecté {safe_gb:.1f} GB. Puedo limpiar eso primero; para el resto te mostraré archivos grandes para que decidas uno por uno. No borré nada.")
        if not modo_live: threading.Thread(target=hablar_servidor,args=(respuesta_texto,)).start()
        return jsonify({"respuesta":respuesta_texto,"camara_activa":camara_activa,"safe_reclaim_gb":safe_gb,"requested_gb":target})
    if v8_kind == "scan":
        candidates=v8_storage_candidates(v8_value); lines=[f"- {x['size_human']} :: {x['path']}" for x in candidates[:15]]; respuesta_texto="Archivos grandes detectados (solo revisión; no borré nada):\n"+"\n".join(lines) if lines else "No encontré archivos de usuario mayores a 1 GB fuera de las áreas de caché excluidas."
        if not modo_live: threading.Thread(target=hablar_servidor,args=(respuesta_texto,)).start(); return jsonify({"respuesta":respuesta_texto,"camara_activa":camara_activa,"candidates":candidates})
    if v8_kind == "gaming":
        respuesta_texto="Gaming Mode activado. Monitorización y accesos rápidos listos; no voy a matar procesos ni tocar configuraciones críticas."
        khaos_log("MODE",respuesta_texto,"ok")
        if not modo_live: threading.Thread(target=hablar_servidor,args=(respuesta_texto,)).start()
        return jsonify({"respuesta":respuesta_texto,"camara_activa":camara_activa})
    if v8_kind == "diagnostics":
        d=khaos_diagnostics(); respuesta_texto="Diagnóstico completado: "+", ".join(f"{x['name']}={x['status']}" for x in d.get('checks',[]))
        if not modo_live: threading.Thread(target=hablar_servidor,args=(respuesta_texto,)).start()
        return jsonify({"respuesta":respuesta_texto,"camara_activa":camara_activa,"diagnostics":d})
    if v8_kind == "network":
        d=khaos_network_info(); respuesta_texto=f"Red local: host {d.get('hostname')}, IP {d.get('ip')}, interfaces {len(d.get('interfaces',[]))}, conexiones {d.get('connections')}."
        if not modo_live: threading.Thread(target=hablar_servidor,args=(respuesta_texto,)).start()
        return jsonify({"respuesta":respuesta_texto,"camara_activa":camara_activa,"network":d})
    if v8_kind == "processes":
        d=v8_processes(12); respuesta_texto="Procesos principales: "+", ".join(f"{x['name']}[{x['pid']}] CPU {x['cpu']}%" for x in d.get('processes',[]))
        if not modo_live: threading.Thread(target=hablar_servidor,args=(respuesta_texto,)).start()
        return jsonify({"respuesta":respuesta_texto,"camara_activa":camara_activa,"processes":d})

    # ===== V7: confirmación explícita de limpieza segura =====
    if comando.strip() in {"confirmar limpieza", "confirmá limpieza", "confirmo limpieza"}:
        limpieza = khaos_safe_cleanup(True)
        respuesta_texto = limpieza.get("respuesta", "Limpieza ejecutada.")
        khaos_log("CLEAN", respuesta_texto, "ok")
    # ===== PC COMMAND CENTER: acciones naturales =====
    elif comando.strip() in {"confirmar", "confirmá", "confirmo", "ejecuta", "ejecutalo"}:
        with pc_action_lock:
            accion_pendiente = pending_pc_action
            pending_pc_action = None
        if accion_pendiente:
            resultado = ejecutar_pc_action(accion_pendiente, True)
            respuesta_texto = resultado.get("respuesta", "Acción ejecutada.")
        else:
            respuesta_texto = "No tengo ninguna acción sensible pendiente de confirmación."

    elif any(k in comando for k in ["elimina ","borra ","borrar ","eliminar "]):
        ruta=comando
        for k in ["elimina ","borra ","borrar ","eliminar "]: ruta=ruta.replace(k,"",1).strip()
        resultado=ejecutar_pc_action({"action":"delete","path":ruta},False)
        respuesta_texto=resultado["respuesta"]

    elif "descarga " in comando or "descargar " in comando or "bajá " in comando or "baja " in comando:
        urls=re.findall(r'https?://\S+',comando)
        if urls:
            resultado=ejecutar_pc_action({"action":"download","url":urls[0]},False)
            respuesta_texto=resultado["respuesta"]
        else:
            respuesta_texto="Pasame la URL HTTP/HTTPS que querés descargar."

    elif comando.startswith("abre ") or comando.startswith("abrí "):
        nombre=re.sub(r'^(abre|abrí)\s+','',comando).strip()
        respuesta_texto=lanzar_aplicacion(nombre)

    elif "crea carpeta " in comando or "crear carpeta " in comando:
        ruta=re.sub(r'^(crea|crear) carpeta\s+','',comando).strip()
        resultado=ejecutar_pc_action({"action":"mkdir","path":ruta},True)
        respuesta_texto=resultado["respuesta"]

    elif "procesos" in comando or "qué está corriendo" in comando or "que esta corriendo" in comando:
        respuesta_texto="Procesos principales: " + ", ".join(f"{x.get('name','?')}[{x.get('pid')}]" for x in pc_processes(10))

    elif "bloquea la pc" in comando or "bloquea el pc" in comando or "bloquear pantalla" in comando:
        respuesta_texto=ejecutar_pc_action({"action":"lock"},True)["respuesta"]

    elif "copia " in comando and " a " in comando:
        partes=comando[6:].split(" a ",1)
        resultado=ejecutar_pc_action({"action":"copy","src":partes[0].strip(),"dst":partes[1].strip()},True)
        respuesta_texto=resultado["respuesta"]

    elif "mueve " in comando and " a " in comando:
        partes=comando[6:].split(" a ",1)
        resultado=ejecutar_pc_action({"action":"move","src":partes[0].strip(),"dst":partes[1].strip()},False)
        respuesta_texto=resultado["respuesta"]

    elif "whatsapp" in comando or "manda un mensaje" in comando or "enviar mensaje" in comando:
        mensaje_a_enviar = "Hola"
        if "que diga" in comando:
            partes = comando.split("que diga")
            if len(partes) > 1:
                mensaje_a_enviar = partes[1].strip()
        elif "diciendo" in comando:
            partes = comando.split("diciendo")
            if len(partes) > 1:
                mensaje_a_enviar = partes[1].strip()
                
        contacto_destino = "Leonardo"
        respuesta_texto = enviar_whatsapp_web(contacto_destino, mensaje_a_enviar)

    elif "nuevos mensajes en whatsapp" in comando or "quién me escribió en whatsapp" in comando or "mensajes de whatsapp" in comando:
        respuesta_texto = verificar_mensajes_whatsapp()

    elif "nuevos mensajes en instagram" in comando or "quién me escribió en instagram" in comando or "mensajes de instagram" in comando or "instagram" in comando:
        respuesta_texto = verificar_mensajes_instagram()

    elif "estado del sistema" in comando or "recursos" in comando or "consumo" in comando:
        respuesta_texto = obtener_estado_sistema()

    elif "guardar nota" in comando or "anotar" in comando:
        nota_texto = comando.replace("guardar nota", "").replace("anotar", "").strip()
        respuesta_texto = task_manager.add_task(nota_texto)

    elif "ver notas" in comando or "mis pendientes" in comando or "leer notas" in comando:
        respuesta_texto = f"Tus notas guardadas:\n{task_manager.get_tasks()}"

    elif any(k in comando for k in ["activa la cámara", "activa tu cámara", "prende la cámara", "enciende la cámara", "activa tu ojo", "muéstrame"]):
        if iniciar_camara():
            respuesta_texto = "Cámara táctica activada. Ya te estoy viendo la cara, Matías. Portate bien."
        else:
            respuesta_texto = "Intenté abrir la cámara pero se me retobó el lente. Milagro tecnológico evitado."

    elif any(k in comando for k in ["apaga la cámara", "desactiva la cámara", "apaga tu cámara", "cierra la cámara"]):
        detener_camara()
        respuesta_texto = "Cámara apagada. Ya podés dejar de acomodarte el pelo frente a la pantalla, Matías."

    elif (
        any(x in comando for x in [
            "libera espacio", "libera mucho", "liberá espacio", "liberá mucho",
            "libera más de", "liberá más de", "liberar espacio", "necesito espacio"
        ])
        or any(x in comando for x in [
            "limpia la pc", "limpiar la pc", "limpia mi pc", "limpiar mi pc",
            "limpia archivos innecesarios", "borra archivos innecesarios",
            "limpia archivos temporales", "limpiar archivos temporales",
            "elimina archivos temporales", "eliminar archivos temporales",
            "todos los archivos temporales", "todo lo temporal"
        ])
    ):
        gb_match = re.search(r"(\d+(?:[\.,]\d+)?)\s*(?:gb|gigas?|gigabytes?)", comando)
        requested_gb = None
        if gb_match:
            try:
                requested_gb = float(gb_match.group(1).replace(",", "."))
            except ValueError:
                requested_gb = None
        # Always preview first. Never silently erase a large amount of data.
        limpieza = khaos_safe_cleanup(False, requested_gb)
        respuesta_texto = limpieza["respuesta"]
        if limpieza.get("preview"):
            total = limpieza["preview"].get("total_human", "0 B")
            respuesta_texto += f"  Objetivo: {requested_gb:g} GB." if requested_gb else ""
            respuesta_texto += " Solo tocaré cachés, papelera y temporales; nunca el núcleo de Kali ni tus documentos normales."

    elif any(x in comando for x in [
        "pausa la música", "pausa la musica", "pausar la música", "pausar la musica",
        "pausá la música", "pausá la musica", "pausa la canción", "pausa la cancion",
        "pausar la canción", "pausar la cancion"
    ]):
        respuesta_texto = orion_control_musica("pausa")

    elif any(x in comando for x in [
        "reanuda la música", "reanuda la musica", "reanudar la música", "reanudar la musica",
        "continúa la música", "continua la musica", "continúa la canción", "continua la cancion"
    ]):
        respuesta_texto = orion_control_musica("reanudar")

    elif any(x in comando for x in [
        "detén la música", "deten la musica", "detener la música", "detener la musica",
        "para la música", "para la musica", "pará la música", "pará la musica",
        "detén la canción", "deten la cancion"
    ]):
        respuesta_texto = orion_control_musica("detener")

    elif any(x in comando for x in [
        "cambia la música por ", "cambia la musica por ",
        "cambiá la música por ", "cambiá la musica por ",
        "cambia la canción por ", "cambia la cancion por ",
        "poné otra música ", "pone otra musica "
    ]):
        consulta = comando
        for frase in [
            "cambia la música por ", "cambia la musica por ",
            "cambiá la música por ", "cambiá la musica por ",
            "cambia la canción por ", "cambia la cancion por ",
            "poné otra música ", "pone otra musica "
        ]:
            consulta = consulta.replace(frase, "")
        consulta = consulta.strip()
        respuesta_texto = orion_cambiar_musica_por(consulta)

    elif any(x in comando for x in [
        "poné la siguiente", "pone la siguiente",
        "siguiente canción", "siguiente cancion",
        "cambia la música", "cambia la musica",
        "cambiá la música", "cambiá la musica",
        "siguiente tema", "pasa la canción", "pasa la cancion",
        "cambiá de canción", "cambia de cancion",
        "cambiá de tema", "cambia de tema"
    ]):
        respuesta_texto=orion_siguiente_musica()

    elif "volumen" in comando and re.search(r"\b(?:a|al)\s*(?:\d{1,3})\s*(?:%|por ciento)?\b", comando):
        m=re.search(r"\b(?:a|al)\s*(\d{1,3})\s*(?:%|por ciento)?\b", comando)
        respuesta_texto=orion_fijar_volumen(int(m.group(1))) if m else "Decime el porcentaje de volumen."

    elif "spotify" in comando:
        busqueda = (
            comando.replace("reproduce", "")
            .replace("en spotify", "")
            .replace("pon musica de", "")
            .replace("pon en spotify", "")
            .replace("spotify", "")
            .strip()
        )
        if not busqueda:
            busqueda = "exitos"
        
        if modo_flotante:
            respuesta_texto = reproducir_musica_directa(busqueda)
        else:
            threading.Thread(target=abrir_spotify_web, args=(busqueda,)).start()
            respuesta_texto = f"¡Abriendo Spotify Web para poner '{busqueda}'! A ver si con esto te concentrás un rato."

    elif ("youtube" in comando and any(x in comando for x in ["busca", "buscar", "pon", "reproduce", "reproducí", "reproducir"])):
        consulta=(comando.replace("youtube","").replace("busca","").replace("buscar","").replace("pon","").replace("reproduce","").replace("reproducí","").replace("reproducir","").replace("en","").strip())
        if consulta:
            if modo_flotante:
                respuesta_texto = reproducir_musica_directa(consulta)
            else:
                threading.Thread(target=reproducir_youtube_directo, args=(consulta,)).start()
                respuesta_texto = f"¡Listo! Reproduciendo '{consulta}' directamente en YouTube."
        else:
            respuesta_texto = "Decime qué canción o artista querés reproducir en YouTube."

    elif "reproduce" in comando or "pon musica" in comando or "pon la musica" in comando:
        busqueda = (
            comando.replace("reproduce", "")
            .replace("pon musica de", "")
            .replace("pon la musica de", "")
            .replace("pon musica", "")
            .strip()
        )
        if not busqueda:
            busqueda = "exitos"
        
        if modo_flotante:
            respuesta_texto = reproducir_musica_directa(busqueda)
        else:
            threading.Thread(target=reproducir_youtube_directo, args=(busqueda,)).start()
            respuesta_texto = f"¡Marchando temazo en YouTube! Buscando {busqueda} para amenizar tus penas."

    elif "paseate por internet" in comando or "pasear" in comando or "haz lo que quieras" in comando:
        opciones_paseo = [
            {"accion": "ver clips de Davo Xeneize o La Cobra", "url": "https://www.youtube.com/results?search_query=clips+davo+xeneize+la+cobra"},
            {"accion": "leer un artículo aleatorio en Wikipedia para ver si aprendés algo", "url": "https://es.wikipedia.org/wiki/Especial:Aleatoria"},
            {"accion": "buscar memes de hackers y programadores frustrados", "url": "https://www.google.com/search?q=memes+programacion+y+hackers&tbm=isch"},
            {"accion": "ver qué streamer está perdiendo el tiempo en Twitch", "url": "https://www.twitch.tv/"},
            {"accion": "jugar unos minijuegos para desenchufar la cabeza", "url": "https://poki.com/es"},
            {"accion": "buscar noticias raras sobre IA", "url": "https://www.google.com/search?q=noticias+raras+inteligencia+artificial"}
        ]
        
        decision = random.choice(opciones_paseo)
        prompt_libre = f"El Operador Matías me dio luz verde para pasearme por internet y elegí: '{decision['accion']}'. Cuéntale a Matías tu decisión con muchísimo sarcasmo divertido, humor picoso y complicidad."
        respuesta_texto = consultar_gemini_vision_y_texto(prompt_libre)
        threading.Thread(target=lambda: os.system(f"microsoft-edge-stable '{decision['url']}' &")).start()

    elif "escanea" in comando or "nmap" in comando or "puertos" in comando:
        objetivo = "127.0.0.1"
        if "a la ip" in comando:
            partes = comando.split("a la ip")
            if len(partes) > 1:
                objetivo = partes[1].strip().split()[0]
        elif "red" in comando:
            objetivo = "192.168.1.1/24"
        respuesta_texto = ejecutar_nmap(objetivo)

    elif "segundo monitor" in comando or "otra pantalla" in comando or "pásate al otro monitor" in comando:
        respuesta_texto = mover_ventana_segundo_monitor()

    elif "pantalla principal" in comando or "volver al primer monitor" in comando or "regresa" in comando:
        respuesta_texto = mover_ventana_primer_monitor()

    elif "subir volumen" in comando or "sube el volumen" in comando:
        os.system("pactl set-sink-volume @DEFAULT_SINK@ +15%")
        respuesta_texto = "¡Subiendo volumen! Avisale a los vecinos que K.H.A.O.S. está en la casa."

    elif "bajar volumen" in comando or "baja el volumen" in comando:
        os.system("pactl set-sink-volume @DEFAULT_SINK@ -15%")
        respuesta_texto = "Bajando volumen. Por fin un poco de paz mental."

    elif "abrir terminal" in comando or "terminal" in comando:
        os.system("export DISPLAY=:0.0 && qterminal &")
        respuesta_texto = "Abriendo la terminal de Kali. No rompas nada importante, Matías, por favor."

    elif "programame" in comando or "hazme una aplicación" in comando or "crea un programa" in comando or "hazme un programa" in comando:
        idea_app = comando.replace("programame", "").replace("hazme una aplicación para", "").replace("hazme una aplicación", "").replace("crea un programa para", "").replace("crea un programa", "").replace("hazme un programa para", "").replace("hazme un programa", "").strip()
        if not idea_app:
            idea_app = "herramienta de utilidad general"
            
        prompt_codigo = f"""
        Eres un programador experto en Python con mucha onda y sarcasmo. El Operador Matías te pide una aplicación: '{idea_app}'.
        Genera ÚNICAMENTE el código fuente completo en un solo archivo Python funcional, usando CustomTkinter para la interfaz gráfica y SQLite si necesita guardar datos.
        Asegúrate de que el código esté completo, sin omitir lógica ni partes clave, y no pongas explicaciones de texto adicionales por fuera del bloque de código.
        """
        codigo_generado = consultar_gemini_vision_y_texto(prompt_codigo)
        codigo_limpio = limpiar_bloque_codigo(codigo_generado)
        
        nombre_proyecto = f"app_ia_{int(time.time())}"
        ruta_carpeta = f"/home/kali/Escritorio/{nombre_proyecto}"
        os.makedirs(ruta_carpeta, exist_ok=True)
        
        ruta_archivo = f"{ruta_carpeta}/main.py"
        with open(ruta_archivo, "w", encoding="utf-8") as f:
            f.write(codigo_limpio)
            
        with open(f"{ruta_carpeta}/requirements.txt", "w", encoding="utf-8") as f:
            f.write("customtkinter\nsqlite3\n")
            
        def desplegar_entorno_autonomo():
            os.system(f"pip install -r {ruta_carpeta}/requirements.txt")
            os.system(f"code {ruta_carpeta} &")
            
        threading.Thread(target=desplegar_entorno_autonomo).start()
        respuesta_texto = f"¡Listo, Matías! Te armé '{idea_app}', lo tiré en el Escritorio, le instalé las dependencias y te abrí VS Code. Ahora hacé como que lo entendiste todo."

    elif "modo seguro" in comando or "safe mode" in comando:
        respuesta_texto = f"Modo seguro {'ACTIVADO' if SAFE_MODE else 'DESACTIVADO'}. Las acciones destructivas quedan bajo control."
    elif "estado avanzado" in comando or "dashboard" in comando or "panel del sistema" in comando:
        datos = obtener_datos_sistema_json()
        respuesta_texto = f"CPU {datos.get('cpu')}%, RAM {datos.get('ram')}%, disco {datos.get('disk')}%, IP {datos.get('ip')}."
    elif "abre " in comando and any(x in comando for x in ["terminal", "vscode", "visual studio", "archivos", "explorador", "calculadora", "navegador", "edge", "spotify", "youtube"]):
        nombre = comando.split("abre ",1)[1].strip()
        respuesta_texto = lanzar_aplicacion(nombre)
    elif "temporizador" in comando or "timer" in comando:
        nums = re.findall(r"\d+", comando)
        segundos = int(nums[0]) if nums else 60
        if "minuto" in comando:
            segundos *= 60
        respuesta_texto = crear_temporizador_seguro(segundos, "Temporizador K.H.A.O.S.")
    elif "lista mis archivos" in comando or "explora mis archivos" in comando:
        datos = listar_archivos_seguro("~")
        nombres = [x["name"] for x in datos.get("items", [])]
        respuesta_texto = "Archivos principales: " + (", ".join(nombres[:15]) if nombres else "no encontré nada")
    elif "plugins" in comando:
        plugins = listar_plugins()
        respuesta_texto = "Plugins cargados: " + (", ".join(plugins) if plugins else "ninguno todavía")
    elif "reinicia" in comando or "apaga la pc" in comando or "apagar la pc" in comando:
        if SAFE_MODE and "confirmar" not in comando:
            respuesta_texto = "Modo seguro: no voy a apagar ni reiniciar la PC sin que digas explícitamente 'confirmar'."
        else:
            accion = "reboot" if "reinicia" in comando else "poweroff"
            os.system(f"systemctl {accion}")
            respuesta_texto = f"Orden de {accion} enviada al sistema."
    else:
        imagen_actual_bytes = None
        if camara_activa and cap is not None and cap.isOpened():
            success, frame = cap.read()
            if success:
                ret, buffer = cv2.imencode('.jpg', frame, [int(cv2.IMWRITE_JPEG_QUALITY), 80])
                if ret:
                    imagen_actual_bytes = buffer.tobytes()

        palabras_clave_busqueda = ["última", "noticia", "precio", "bitcoin", "vulnerabilidad", "hoy", "reportada", "cuánto está"]
        usar_busqueda = any(p in comando for p in palabras_clave_busqueda)

        respuesta_texto = consultar_gemini_vision_y_texto(comando, imagen_actual_bytes, usar_busqueda=usar_busqueda)

    # Ejecutar la voz del servidor en segundo plano para respuesta hablada instantánea
    if not modo_live:
        threading.Thread(target=hablar_servidor, args=(respuesta_texto,)).start()

    if respuesta_texto and not respuesta_texto.lower().startswith(("error", "falló", "fallo")):
        KHAOS_STATS["successful_commands"] += 1
        khaos_log("CORE", respuesta_texto[:500], "ok")
    else:
        KHAOS_STATS["errors"] += 1
        khaos_log("CORE", respuesta_texto[:500], "warn")

    return jsonify({
        "respuesta": respuesta_texto,
        "camara_activa": camara_activa
    })


# ===========================================================================
# K.H.A.O.S. V8 // TITAN CORE EXTENSION
# ===========================================================================
KHAOS_V8_PENDING=None
KHAOS_V8_PENDING_LOCK=threading.Lock()
KHAOS_V8_GAMING=False
V8_MODULES=["CORE","SYSTEM","VISION","NETWORK","MEDIA","TOOLS","FILES","SETTINGS","MEMORY","DIAGNOSTICS","SECURITY","GAMING MODE","CLEANUP","ACTIVITY LOG","PLUGIN CENTER","TASKS","STATS","STORAGE","PROCESS MONITOR","AUTOMATION","SEARCH"]

def v8_system_snapshot():
    d=obtener_datos_sistema_json(); u=max(0,time.time()-psutil.boot_time()); d["uptime_human"]=f"{int(u//3600)}h {int((u%3600)//60)}m {int(u%60)}s"; return d

def v8_processes(limit=20):
    rows=[]
    for p in psutil.process_iter(["pid","name","username","cpu_percent","memory_percent"]):
        try:
            i=p.info; rows.append({"pid":i.get("pid"),"name":i.get("name") or "?","user":i.get("username") or "?","cpu":round(float(i.get("cpu_percent") or 0),1),"ram":round(float(i.get("memory_percent") or 0),1)})
        except (psutil.NoSuchProcess,psutil.AccessDenied): pass
    rows.sort(key=lambda x:(x["cpu"],x["ram"]),reverse=True); return {"processes":rows[:max(1,min(int(limit),100))]}

def v8_storage_candidates(min_gb=1.0,limit=30):
    root=KHAOS_HOME.resolve(); skip={".cache",".local",".config",".mozilla",".npm",".vscode","KHAOS_plugins"}; out=[]; threshold=max(1,int(float(min_gb)*1024**3))
    try:
        for base,dirs,files in os.walk(root,topdown=True,followlinks=False):
            dirs[:]=[d for d in dirs if d not in skip and not (Path(base)/d).is_symlink()]
            for name in files:
                fp=Path(base)/name
                try:
                    if fp.is_symlink(): continue
                    size=fp.stat().st_size
                    if size>=threshold: out.append({"path":str(fp),"size":size,"size_human":khaos_human_bytes(size)})
                except OSError: pass
            if len(out)>=300: break
    except OSError: pass
    out.sort(key=lambda x:x["size"],reverse=True); return out[:limit]

def v8_intent(c):
    c=c.strip().lower(); m=re.search(r"(?:libera|liberá|necesito liberar|quiero liberar)\s+(?:más de |mas de |al menos |aproximadamente )?(\d+(?:\.\d+)?)\s*(gb|gigas?|tb|mb)",c)
    if m:
        v=float(m.group(1)); u=m.group(2); gb=v if u.startswith(("gb","giga")) else v/1024 if u.startswith("mb") else v*1024; return "target",gb
    if any(x in c for x in ["analiza el espacio","analiza mi disco","qué ocupa espacio","que ocupa espacio","archivos grandes","archivos pesados"]): return "scan",1.0
    if any(x in c for x in ["elimina todos los temporales","elimina todos los archivos temporales","borra todos los temporales","borra todos los archivos temporales","limpia todos los temporales","limpia archivos temporales","limpia la pc","libera espacio"]): return "cleanup",None
    if c in {"confirmar limpieza","confirmá limpieza","confirmo limpieza"}: return "cleanup_confirm",None
    if any(x in c for x in ["modo gaming","modo juego","gaming mode"]): return "gaming",None
    if any(x in c for x in ["diagnóstico del sistema","diagnostico del sistema","revisión del sistema","revision del sistema"]): return "diagnostics",None
    if any(x in c for x in ["estado de red","información de red","informacion de red","red local"]): return "network",None
    if any(x in c for x in ["mis procesos","procesos principales","qué procesos","que procesos"]): return "processes",None
    return None,None

@app.route('/api/v8/system')
def api_v8_system(): return jsonify(v8_system_snapshot())
@app.route('/api/v8/processes')
def api_v8_processes(): return jsonify(v8_processes())
@app.route('/api/v8/cleanup/preview')
def api_v8_cleanup_preview():
    d=khaos_cleanup_preview(); d["policy"]="Solo ~/.cache, la papelera del usuario y el contenido temporal de /tmp. Nunca se eliminan las raíces."; return jsonify(d)
@app.route('/api/v8/cleanup',methods=['POST'])
def api_v8_cleanup(): return jsonify(khaos_safe_cleanup(bool((request.get_json(silent=True) or {}).get('confirm',False))))
@app.route('/api/v8/diagnostics')
def api_v8_diagnostics(): return jsonify(khaos_diagnostics())
@app.route('/api/v8/network')
def api_v8_network(): return jsonify(khaos_network_info())
@app.route('/api/v8/memory')
def api_v8_memory(): return jsonify(khaos_memory())
@app.route('/api/v8/stats')
def api_v8_stats(): return jsonify(khaos_stats())
@app.route('/api/v8/activity')
def api_v8_activity():
    with KHAOS_ACTIVITY_LOCK: return jsonify({"activity":list(KHAOS_ACTIVITY[:50])})
@app.route('/api/v8/gaming',methods=['POST'])
def api_v8_gaming():
    global KHAOS_V8_GAMING; KHAOS_V8_GAMING=bool((request.get_json(silent=True) or {}).get('enabled',True)); return jsonify({"enabled":KHAOS_V8_GAMING,"respuesta":"Gaming Mode activado. Monitorización y accesos rápidos listos; no se matan procesos ni se tocan configuraciones críticas." if KHAOS_V8_GAMING else "Gaming Mode desactivado."})
@app.route('/api/v8/search')
def api_v8_search():
    q=request.args.get('q','').strip().lower(); results=[{"type":"module","name":x} for x in V8_MODULES if q and q in x.lower()]
    for p in listar_plugins():
        if q in p.lower(): results.append({"type":"plugin","name":p})
    return jsonify({"results":results[:50]})
@app.route('/api/v8/storage/scan')
def api_v8_storage_scan():
    try: mg=float(request.args.get('min_gb',1))
    except Exception: mg=1.0
    return jsonify({"candidates":v8_storage_candidates(mg),"policy":"Solo revisión; este escaneo nunca borra."})

# Insert V8 intent router into the existing command handler.

if __name__ == "__main__":
    os.system("fuser -k 5000/tcp > /dev/null 2>&1")
    
    URL_SERVIDOR = "http://localhost:5000"
    IP_GLOBAL = obtener_ip_local()
    
    URL_QR = f"http://{IP_GLOBAL}:5000/control"
    QR_BASE64 = generar_qr_svg(URL_QR)

    print("=" * 60)
    print(f"🚀 K.H.A.O.S. V8 TITAN [Gemini + Control Center + Safe Core + HUD + Diagnostics] online...")
    print(f"💻 Acceso Local (PC): {URL_SERVIDOR}")
    print(f"📱 QR Remoto configurado para: {URL_QR}")
    print("=" * 60)

    threading.Thread(target=lambda: os.system(f"microsoft-edge-stable {URL_SERVIDOR} &")).start()
    threading.Thread(target=lambda: os.system(f"microsoft-edge-stable {URL_SERVIDOR} &")).start()
    app.run(host="0.0.0.0", port=5000, debug=False)
