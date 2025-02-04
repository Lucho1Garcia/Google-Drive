# Archives-to-Google-Drive-To-YouTube
import os
import time
from tkinter import Tk, Label, Button, Text, Scrollbar, END, Frame, Toplevel, filedialog, simpledialog
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler
from googleapiclient.discovery import build
from googleapiclient.http import MediaFileUpload
from google_auth_oauthlib.flow import InstalledAppFlow
from google.auth.transport.requests import Request
import pickle

# Autenticación de la API de Google
def authenticate_google(scopes, token_file, credentials_file):
    creds = None
    if os.path.exists(token_file):
        with open(token_file, "rb") as token:
            creds = pickle.load(token)
    if not creds or not creds.valid:
        if creds and creds.expired and creds.refresh_token:
            creds.refresh(Request())
        else:
            flow = InstalledAppFlow.from_client_secrets_file(credentials_file, scopes)
            creds = flow.run_local_server(port=0)
        with open(token_file, "wb") as token:
            pickle.dump(creds, token)
    return creds

# Subir video a YouTube
def upload_to_youtube(youtube, file_path, log):
    try:
        title = os.path.basename(file_path)
        log(f"Iniciando la subida del archivo a YouTube: {file_path}")
        body = {
            "snippet": {
                "title": title,
                "description": "Video subido automáticamente.",
                "tags": ["automático", "YouTube API"],
                "categoryId": "22",
            },
            "status": {
                "privacyStatus": "private",
            },
        }
        media = MediaFileUpload(file_path, chunksize=-1, resumable=True)
        request = youtube.videos().insert(part="snippet,status", body=body, media_body=media)
        response = None
        while response is None:
            status, response = request.next_chunk()
            if status:
                log(f"Subiendo a YouTube: {int(status.progress() * 100)}% completado.")
        log(f"El video '{file_path}' se ha subido correctamente a YouTube.")
    except Exception as e:
        log(f"Error al subir a YouTube: {e}")

# Subir archivo a Google Drive
def upload_to_drive(drive, file_path, folder_id, log):
    try:
        file_metadata = {
            "name": os.path.basename(file_path),
            "parents": [folder_id]
        }
        media = MediaFileUpload(file_path, resumable=True)
        file = drive.files().create(body=file_metadata, media_body=media, fields="id").execute()
        log(f"Archivo '{file_path}' subido a Google Drive con ID: {file.get('id')}")
    except Exception as e:
        log(f"Error al subir a Google Drive: {e}")

# Monitorear la carpeta
class FolderWatcher(FileSystemEventHandler):
    def __init__(self, youtube, drive, drive_folder_id, log):
        self.youtube = youtube
        self.drive = drive
        self.drive_folder_id = drive_folder_id
        self.log = log

    def on_created(self, event):
        if event.is_directory:
            return
        _, ext = os.path.splitext(event.src_path)
        if ext.lower() in [".mp4", ".mov", ".avi", ".mkv"]:
            self.log(f"Nuevo archivo detectado: {event.src_path}")
            upload_to_youtube(self.youtube, event.src_path, self.log)
            upload_to_drive(self.drive, event.src_path, self.drive_folder_id, self.log)

class App:
    def __init__(self, root):
        self.root = root
        self.root.title("Subidor de Videos Automático")

        # Frame principal
        frame = Frame(self.root)
        frame.pack(pady=10)

        # Botones
        self.start_button = Button(frame, text="Iniciar Monitoreo", command=self.start_monitoring, width=20)
        self.start_button.grid(row=0, column=0, padx=10)

        self.stop_button = Button(frame, text="Detener Monitoreo", command=self.stop_monitoring, width=20, state="disabled")
        self.stop_button.grid(row=0, column=1, padx=10)

        self.settings_button = Button(frame, text="⚙️ Ajustes", command=self.open_settings, width=20)
        self.settings_button.grid(row=0, column=2, padx=10)

        # Área de texto para el log
        self.log_text = Text(self.root, height=20, width=70, state="disabled", wrap="word")
        self.log_text.pack(pady=10)
        scrollbar = Scrollbar(self.root, command=self.log_text.yview)
        self.log_text.configure(yscrollcommand=scrollbar.set)
        scrollbar.pack(side="right", fill="y")

        # Configuración de monitoreo
        self.folder_to_watch = r"C:\Users\lucia\Desktop\Carpeta YouTube"
        self.drive_folder_id = "1jlKCjiZxgriEsYv2fHBPiM6qOSwMB36Y"
        self.observer = None
        self.youtube = None
        self.drive = None

        # Autenticación
        self.authenticate()

    def authenticate(self):
        SCOPES = ["https://www.googleapis.com/auth/youtube.upload", "https://www.googleapis.com/auth/drive.file"]
        CREDENTIALS_FILE = "config/client_secrets.json"
        YOUTUBE_TOKEN = "youtube_token.pickle"
        DRIVE_TOKEN = "drive_token.pickle"

        self.log("Autenticando con Google...")
        self.youtube = authenticate_google(SCOPES[:1], YOUTUBE_TOKEN, CREDENTIALS_FILE)
        self.youtube = build("youtube", "v3", credentials=self.youtube)
        self.drive = authenticate_google(SCOPES[1:], DRIVE_TOKEN, CREDENTIALS_FILE)
        self.drive = build("drive", "v3", credentials=self.drive)
        self.log("Autenticación completada.")

    def log(self, message):
        self.log_text.config(state="normal")
        self.log_text.insert(END, f"{message}\n")
        self.log_text.see(END)
        self.log_text.config(state="disabled")

    def open_settings(self):
        settings_window = Toplevel(self.root)
        settings_window.title("Ajustes")
        settings_window.geometry("300x250")

        Label(settings_window, text="Opciones de Ajustes", font=("Arial", 14)).pack(pady=10)

        Button(settings_window, text="Iniciar sesión en YouTube", command=lambda: self.reauthenticate("youtube"), width=25).pack(pady=5)
        Button(settings_window, text="Iniciar sesión en Drive", command=lambda: self.reauthenticate("drive"), width=25).pack(pady=5)
        Button(settings_window, text="Cambiar carpeta monitoreada", command=self.change_folder, width=25).pack(pady=5)
        Button(settings_window, text="Cambiar carpeta de Drive", command=self.change_drive_folder, width=25).pack(pady=5)

    def reauthenticate(self, service):
        if service == "youtube":
            SCOPES = ["https://www.googleapis.com/auth/youtube.upload"]
            CREDENTIALS_FILE = "config/client_secrets.json"
            YOUTUBE_TOKEN = "youtube_token.pickle"
            self.youtube = authenticate_google(SCOPES, YOUTUBE_TOKEN, CREDENTIALS_FILE)
            self.youtube = build("youtube", "v3", credentials=self.youtube)
            self.log("Reautenticado en YouTube.")
        elif service == "drive":
            SCOPES = ["https://www.googleapis.com/auth/drive.file"]
            CREDENTIALS_FILE = "config/client_secrets.json"
            DRIVE_TOKEN = "drive_token.pickle"
            self.drive = authenticate_google(SCOPES, DRIVE_TOKEN, CREDENTIALS_FILE)
            self.drive = build("drive", "v3", credentials=self.drive)
            self.log("Reautenticado en Drive.")

    def change_drive_folder(self):
        new_folder_id = simpledialog.askstring("Cambiar carpeta de Drive", "Ingresa el ID de la nueva carpeta o el enlace completo:")
        if new_folder_id:
            # Extraer el ID si se ingresó un enlace completo
            if "drive.google.com" in new_folder_id and "/folders/" in new_folder_id:
                new_folder_id = new_folder_id.split("/folders/")[1].split("?")[0]
            self.drive_folder_id = new_folder_id
            self.log(f"Carpeta de Drive actualizada. Nueva ID: {self.drive_folder_id}")

    def change_folder(self):
        new_folder = filedialog.askdirectory(title="Seleccionar Carpeta para Monitorear")
        if new_folder:
            self.folder_to_watch = new_folder
            self.log(f"Carpeta monitoreada cambiada a: {self.folder_to_watch}")

    def start_monitoring(self):
        if not os.path.exists(self.folder_to_watch):
            os.makedirs(self.folder_to_watch)
            self.log(f"Carpeta creada: {self.folder_to_watch}")
        self.log(f"Monitoreando la carpeta: {self.folder_to_watch}")

        self.start_button.config(state="disabled")
        self.stop_button.config(state="normal")

        self.observer = Observer()
        event_handler = FolderWatcher(self.youtube, self.drive, self.drive_folder_id, self.log)
        self.observer.schedule(event_handler, self.folder_to_watch, recursive=False)
        self.observer.start()

    def stop_monitoring(self):
        if self.observer:
            self.observer.stop()
            self.observer.join()
            self.observer = None
            self.log("Monitoreo detenido.")

        self.start_button.config(state="normal")
        self.stop_button.config(state="disabled")

def main():
    root = Tk()
    app = App(root)
    root.mainloop()

if __name__ == "__main__":
    main()
