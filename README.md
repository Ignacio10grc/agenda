# agenda
from flask import Flask, request, jsonify, render_template
import datetime
import time
import requests
from threading import Thread

app = Flask(__name__)
tasks = []
FCM_SERVER_KEY = "TU_CLAVE_DE_SERVIDOR_DE_FIREBASE"  # Reemplázala con tu clave de Firebase
FCM_URL = "https://fcm.googleapis.com/fcm/send"
devices = []  # Lista de tokens de dispositivos registrados

def validate_time(time_str):
    try:
        datetime.datetime.strptime(time_str, "%H:%M")
        return True
    except ValueError:
        return False

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/add_task', methods=['POST'])
def add_task():
    data = request.json
    task_text = data.get('task_text')
    task_time = data.get('task_time')
    
    if task_text and validate_time(task_time):
        tasks.append({"task": task_text, "time": task_time})
        return jsonify({"message": "Tarea agregada", "tasks": tasks})
    return jsonify({"error": "Ingrese una tarea válida y un horario en formato HH:MM"})

@app.route('/complete_task', methods=['POST'])
def complete_task():
    data = request.json
    task_index = data.get('task_index')
    
    if 0 <= task_index < len(tasks):
        completed_task = tasks.pop(task_index)
        return jsonify({"message": f"Tarea completada: {completed_task['task']}", "tasks": tasks})
    return jsonify({"error": "Índice de tarea inválido"})

@app.route('/register_device', methods=['POST'])
def register_device():
    data = request.json
    token = data.get("token")
    if token and token not in devices:
        devices.append(token)
        return jsonify({"message": "Dispositivo registrado"})
    return jsonify({"error": "Token inválido o ya registrado"})

def send_notification(title, message):
    headers = {
        "Authorization": f"key={FCM_SERVER_KEY}",
        "Content-Type": "application/json"
    }
    payload = {
        "registration_ids": devices,
        "notification": {
            "title": title,
            "body": message,
            "click_action": "/"
        }
    }
    requests.post(FCM_URL, json=payload, headers=headers)

def check_notifications():
    while True:
        now = datetime.datetime.now().strftime("%H:%M")
        for task in tasks:
            if task["time"] == now:
                send_notification("Recordatorio de Tarea", task["task"])
        time.sleep(60)
if __name__ == '__main__':
    notification_thread = Thread(target=check_notifications, daemon=True)
    notification_thread.start()
    app.run(debug=True)
