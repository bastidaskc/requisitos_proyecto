# app.py
import os
import uuid
import json
import datetime as dt
from pathlib import Path

from flask import Flask, request, Response, redirect, url_for
from werkzeug.utils import secure_filename
import requests

# ---------------------------
# Config
# ---------------------------
ROBOFLOW_MODEL   = os.getenv("ROBOFLOW_MODEL", "tu-workspace/tu-modelo")
ROBOFLOW_VERSION = os.getenv("ROBOFLOW_VERSION", "1")
ROBOFLOW_API_KEY = os.getenv("ROBOFLOW_API_KEY", "TU_API_KEY")
ALLOWED_EXTS     = {"jpg", "jpeg", "png"}

BASE_DIR     = Path(__file__).parent
STATIC_DIR   = BASE_DIR / "static"
UPLOADS_DIR  = STATIC_DIR / "uploads"
UPLOADS_DIR.mkdir(parents=True, exist_ok=True)

HISTORY_FILE = BASE_DIR / "history.json"

ROBOFLOW_URL = f"https://detect.roboflow.com/{ROBOFLOW_MODEL}/{ROBOFLOW_VERSION}?api_key={ROBOFLOW_API_KEY}"

app = Flask(__name__, static_folder=str(STATIC_DIR))

# ---------------------------
# Helpers
# ---------------------------
def allowed_file(filename: str) -> bool:
    return "." in filename and filename.rsplit(".", 1)[1].lower() in ALLOWED_EXTS

def load_history() -> list:
    if HISTORY_FILE.exists():
        try:
            return json.loads(HISTORY_FILE.read_text(encoding="utf-8"))
        except Exception:
            return []
    return []

def save_history(items: list) -> None:
    HISTORY_FILE.write_text(json.dumps(items, ensure_ascii=False, indent=2), encoding="utf-8")

def add_history_entry(filename_saved: str, pred_json: dict) -> str:
    items = load_history()
    entry_id = str(uuid.uuid4())
    items.insert(0, {  # insert al inicio (lo más reciente primero)
        "id": entry_id,
        "file": filename_saved,           # ruta relativa dentro de /static/uploads
        "timestamp": dt.datetime.now().isoformat(timespec="seconds"),
        "prediction": pred_json
    })
    save_history(items)
    return entry_id

def html_page(content: str, selected_id: str | None = None) -> str:
    # panel izquierdo: historial
    items = load_history()
    hist_links = []
    for it in items:
        active = "style='font-weight:600;'" if it["id"] == selected_id else ""
        hist_links.append(
            f"<li><a {active} href='/pred/{it['id']}' title='{it['timestamp']}'>{it['timestamp']} — {os.path.basename(it['file'])}</a></li>"
        )
    hist_html = "<ul>" + ("\n".join(hist_links) if hist_links else "<li>Sin registros</li>") + "</ul>"

    return f"""<!doctype html>
<html lang="es">
<head>
  <meta charset="utf-8">
  <title>Demo Roboflow + Historial</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <style>
    :root {{ --bg:#0b1220; --card:#111827; --muted:#94a3b8; --text:#e5e7eb; }}
    * {{ box-sizing:border-box }}
    body {{ margin:0; font-family: system-ui, -apple-system, Segoe UI, Roboto, Ubuntu, Arial, sans-serif; background:var(--bg); color:var(--text); }}
    .wrap {{ display:flex; min-height:100vh; }}
    aside {{ width:320px; border-right:1px solid #1f2937; padding:16px; }}
    main {{ flex:1; padding:24px; }}
    h1 {{ margin:0 0 12px 0; font-size:22px }}
    .card {{ background:var(--card); border:1px solid #1f2937; border-radius:12px; padding:16px; }}
    form .row {{ margin:10px 0 }}
    input[type=file] {{ padding:6px; width:100% }}
    button {{ padding:10px 14px; border-radius:10px; border:1px solid #2563eb; background:#2563eb; color:#fff; cursor:pointer }}
    ul {{ list-style:none; padding:0; margin:8px 0 0 0 }}
    li {{ margin:6px 0; }}
    a {{ color:#93c5fd; text-decoration:none }}
    a:hover {{ text-decoration:underline }}
    .hint {{ color:var(--muted); font-size:13px }}
    img.preview {{ max-width:100%; border-radius:10px; display:block; margin:10px 0 }}
    pre {{ background:#0f172a; padding:12px; border-radius:10px; overflow:auto }}
  </style>
</head>
<body>
  <div class="wrap">
    <aside>
      <h1>Historial</h1>
      <div class="card">{hist_html}</div>
      <div class="hint" style="margin-top:10px">
        Se guarda en <code>history.json</code>. Archivos en <code>/static/uploads</code>.
      </div>
    </aside>
    <main>
      {content}
    </main>
  </div>
</body>
</html>"""

# ---------------------------
# Rutas
# ---------------------------
@app.get("/")
def home():
    content = f"""
      <h1>Subir imagen → Predicción Roboflow</h1>
      <div class="card">
        <form action="/predict" method="post" enctype="multipart/form-data">
          <div class="row">
            <input type="file" name="file" accept=".jpg,.jpeg,.png" required>
          </div>
          <div class="row">
            <button type="submit">Enviar</button>
          </div>
        </form>
        <p class="hint">Configura <b>ROBOFLOW_MODEL</b>, <b>ROBOFLOW_VERSION</b> y <b>ROBOFLOW_API_KEY</b> como variables de entorno.</p>
      </div>
    """
    return Response(html_page(content), mimetype="text/html")

@app.post("/predict")
def predict():
    if "file" not in request.files:
        return {"error": "Falta 'file'."}, 400
    f = request.files["file"]
    if f.filename == "":
        return {"error": "Archivo vacío."}, 400
    if not allowed_file(f.filename):
        return {"error": "Formato no permitido. Usa JPG/PNG."}, 400
    if "TU_API_KEY" in ROBOFLOW_API_KEY or "tu-modelo" in ROBOFLOW_MODEL:
        return {"error": "Configura credenciales de Roboflow."}, 400

    # Guardar copia persistente para re-visualizar
    filename = secure_filename(f.filename)
    saved_name = f"{uuid.uuid4()}_{filename}"
    saved_path = UPLOADS_DIR / saved_name
    f.save(saved_path.as_posix())

    # Enviar a Roboflow
    try:
        with open(saved_path, "rb") as img:
            resp = requests.post(ROBOFLOW_URL, files={"file": img}, timeout=60)
        data = resp.json()
    except Exception as e:
        return {"error": "Fallo Roboflow", "detail": str(e)}, 502

    # Guardar en historial
    entry_id = add_history_entry(f"uploads/{saved_name}", data)

    # Redirigir a la vista de esa predicción
    return redirect(url_for("view_pred", entry_id=entry_id), code=303)

@app.get("/pred/<entry_id>")
def view_pred(entry_id: str):
    items = load_history()
    item = next((x for x in items if x["id"] == entry_id), None)
    if not item:
        return {"error": "No existe el id"}, 404

    img_url = url_for('static', filename=item["file"])
    pretty_json = json.dumps(item["prediction"], ensure_ascii=False, indent=2)
    content = f"""
      <h1>Predicción — {item['timestamp']}</h1>
      <div class="card">
        <img class="preview" src="{img_url}" alt="preview">
        <pre>{pretty_json}</pre>
        <p class="hint">Archivo: {os.path.basename(item['file'])}</p>
        <p><a href="/">← Nueva predicción</a></p>
      </div>
    """
    return Response(html_page(content, selected_id=entry_id), mimetype="text/html")

# ---------------------------
# Run
# ---------------------------
if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=True)
