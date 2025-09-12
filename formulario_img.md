# app.py
import os
import uuid
import json
import tempfile
from pathlib import Path
from flask import Flask, request, Response

import requests
from werkzeug.utils import secure_filename

# -----------------------------
# Config
# -----------------------------
# Define estos 3 valores como variables de entorno o cámbialos aquí.
ROBOFLOW_MODEL = os.getenv("ROBOFLOW_MODEL", "tu-workspace/tu-modelo")  # p.ej. "miusuario/fruit-detector"
ROBOFLOW_VERSION = os.getenv("ROBOFLOW_VERSION", "1")                    # p.ej. "3"
ROBOFLOW_API_KEY = os.getenv("ROBOFLOW_API_KEY", "TU_API_KEY")

# Extensiones permitidas
ALLOWED_EXTS = {"jpg", "jpeg", "png"}

# URL Roboflow (modelo hosted)
ROBOFLOW_URL = f"https://detect.roboflow.com/{ROBOFLOW_MODEL}/{ROBOFLOW_VERSION}?api_key={ROBOFLOW_API_KEY}"

app = Flask(__name__)

# -----------------------------
# Helpers
# -----------------------------
def allowed_file(filename: str) -> bool:
    return "." in filename and filename.rsplit(".", 1)[1].lower() in ALLOWED_EXTS


# -----------------------------
# Rutas
# -----------------------------
@app.get("/")
def form():
    # Formulario HTML simple en una sola ruta
    return Response(
        """
<!doctype html>
<html lang="es">
  <head>
    <meta charset="utf-8">
    <title>Subir imagen a Roboflow</title>
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <style>
      body{font-family:system-ui,-apple-system,Segoe UI,Roboto,Ubuntu,Arial,sans-serif;max-width:680px;margin:40px auto;padding:0 16px}
      .card{border:1px solid #e5e7eb;border-radius:12px;padding:20px}
      .row{margin:12px 0}
      input[type=file]{padding:8px}
      button{padding:10px 14px;border-radius:10px;border:1px solid #111827;background:#111827;color:#fff;cursor:pointer}
      code,pre{background:#f3f4f6;padding:8px;border-radius:8px;display:block;white-space:pre-wrap}
      .hint{color:#6b7280;font-size:14px}
    </style>
  </head>
  <body>
    <h1>Predicción con Roboflow (demo)</h1>
    <div class="card">
      <form action="/predict" method="post" enctype="multipart/form-data">
        <div class="row">
          <label for="file">Elige una imagen (JPG/PNG):</label><br>
          <input id="file" name="file" type="file" accept=".jpg,.jpeg,.png" required>
        </div>
        <div class="row">
          <button type="submit">Enviar a Roboflow</button>
        </div>
      </form>
      <p class="hint">Configura <b>ROBOFLOW_MODEL</b>, <b>ROBOFLOW_VERSION</b> y <b>ROBOFLOW_API_KEY</b> en variables de entorno.</p>
    </div>
  </body>
</html>
        """,
        mimetype="text/html",
    )


@app.post("/predict")
def predict():
    if "file" not in request.files:
        return {"error": "No se envió archivo 'file'."}, 400

    f = request.files["file"]
    if f.filename == "":
        return {"error": "Archivo sin nombre."}, 400

    if not allowed_file(f.filename):
        return {"error": "Formato no permitido. Usa JPG/PNG."}, 400

    # Validar configuración
    if "TU_API_KEY" in ROBOFLOW_API_KEY or "tu-modelo" in ROBOFLOW_MODEL:
        return {
            "error": "Configura ROBOFLOW_MODEL, ROBOFLOW_VERSION y ROBOFLOW_API_KEY."
        }, 400

    # Guardar temporalmente y enviar a Roboflow
    filename = secure_filename(f.filename)
    tmp_dir = Path(tempfile.gettempdir()) / "uploads"
    tmp_dir.mkdir(parents=True, exist_ok=True)
    tmp_path = tmp_dir / f"{uuid.uuid4()}_{filename}"
    f.save(tmp_path.as_posix())

    try:
        with open(tmp_path, "rb") as img:
            files = {"file": img}
            resp = requests.post(ROBOFLOW_URL, files=files, timeout=60)

        # Responder JSON crudo de Roboflow (o error)
        try:
            data = resp.json()
        except Exception:
            return {"error": "Respuesta no-JSON de Roboflow", "raw": resp.text}, 502

        return Response(json.dumps(data, ensure_ascii=False, indent=2), mimetype="application/json", status=resp.status_code)
    except requests.RequestException as e:
        return {"error": "Fallo al conectar con Roboflow", "detail": str(e)}, 502
    finally:
        # Limpieza del archivo temporal
        try:
            tmp_path.unlink(missing_ok=True)
        except Exception:
            pass


if __name__ == "__main__":
    # Ejecuta:  python app.py
    # Luego visita: http://127.0.0.1:5000/
    app.run(host="0.0.0.0", port=5000, debug=True)
