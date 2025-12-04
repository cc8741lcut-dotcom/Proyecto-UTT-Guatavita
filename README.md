#Glamping Guatavita

!pip install -q flask pyngrok pandas numpy matplotlib openpyxl

import os, time, threading, base64
from io import BytesIO
from datetime import datetime
from flask import Flask, request, render_template_string, redirect, url_for, send_file
from pyngrok import ngrok
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt


NGROK_TOKEN = "36MQupmX908XQQXvU1TAke9wzIq_yGQKGEXLq71sE7FnKFkN"
CSV_FILE = "visitantes.csv"
THEME_COLOR = "#0ea5a4"

if NGROK_TOKEN:
    try:
        ngrok.set_auth_token(NGROK_TOKEN)
    except Exception as e:
        print("⚠️ Advertencia al setear token ngrok:", e)


COLUMNS = ["id","nombre","edad","ciudad","dinero_gastado","glamping","tipo_viajero","fecha_visita","comentarios"]

def ensure_csv():
    if not os.path.exists(CSV_FILE):
        pd.DataFrame(columns=COLUMNS).to_csv(CSV_FILE, index=False)

def read_data():
    ensure_csv()
    try:
        return pd.read_csv(CSV_FILE, dtype=str).replace({np.nan: ""})
    except Exception as e:
        print("⚠️ Error leyendo CSV:", e)
        return pd.DataFrame(columns=COLUMNS)

def append_row(payload):
    df = read_data()
    try:
        next_id = int(df['id'].dropna().astype(int).max()) + 1 if not df.empty and df['id'].dropna().size>0 else 1
    except Exception:
        next_id = len(df) + 1
    row = {c: "" for c in COLUMNS}
    row['id'] = next_id
    for k in COLUMNS[1:]:
        row[k] = payload.get(k, "")
    df2 = pd.concat([df, pd.DataFrame([row])], ignore_index=True)
    df2.to_csv(CSV_FILE, index=False)
    return next_id

def delete_by_id(visitor_id):
    df = read_data()
    try:
        vid = str(int(visitor_id))
    except:
        return False, "ID inválido"
    if df.empty or 'id' not in df.columns:
        return False, "No hay registros"
    matches = df.index[df['id'].astype(str) == vid].tolist()
    if not matches:
        return False, f"No se encontró ID {vid}"
    df = df.drop(matches[0]).reset_index(drop=True)
    df.to_csv(CSV_FILE, index=False)
    return True, f"Visitante ID {vid} eliminado"


# GRÁFICAS

def fig_to_base64(fig):
    buf = BytesIO()
    fig.savefig(buf, format='png', bbox_inches='tight')
    plt.close(fig)
    buf.seek(0)
    return base64.b64encode(buf.getvalue()).decode()

def plot_visitors_by_glamping(df):
    fig, ax = plt.subplots(figsize=(6,3.2))
    counts = df['glamping'].replace("", "Sin glamping").value_counts()
    if counts.empty:
        ax.text(0.5,0.5,"No hay datos", ha='center')
    else:
        counts.plot(kind='bar', ax=ax)
    ax.set_title("Visitantes por Glamping"); ax.set_ylabel("Cantidad")
    plt.tight_layout()
    return fig_to_base64(fig)

def plot_visitors_by_tipo(df):
    fig, ax = plt.subplots(figsize=(6,3.2))
    counts = df['tipo_viajero'].replace("", "Sin tipo").value_counts()
    if counts.empty:
        ax.text(0.5,0.5,"No hay datos", ha='center')
    else:
        counts.plot(kind='bar', ax=ax)
    ax.set_title("Visitantes por Tipo de Viajero"); ax.set_ylabel("Cantidad")
    plt.tight_layout()
    return fig_to_base64(fig)

def plot_age_distribution(df):
    fig, ax = plt.subplots(figsize=(6,3.2))
    ages = pd.to_numeric(df['edad'], errors='coerce').dropna()
    if ages.empty:
        ax.text(0.5,0.5,"No hay datos de edad", ha='center')
    else:
        ax.hist(ages, bins=max(3,min(12,int(ages.nunique()))), edgecolor='black')
        ax.set_xlabel("Edad"); ax.set_ylabel("Frecuencia")
    ax.set_title("Distribución de Edades")
    plt.tight_layout()
    return fig_to_base64(fig)

def plot_avg_spent_by_glamping(df):
    fig, ax = plt.subplots(figsize=(6,3.2))
    df['dinero_gastado'] = pd.to_numeric(df['dinero_gastado'], errors='coerce')
    mean_spent = df.groupby('glamping')['dinero_gastado'].mean().dropna()
    if mean_spent.empty:
        ax.text(0.5,0.5,"No hay datos de gasto", ha='center')
    else:
        mean_spent.plot(kind='bar', ax=ax)
        ax.set_ylabel("Promedio gastado")
    ax.set_title("Gasto promedio por glamping")
    plt.tight_layout()
    return fig_to_base64(fig)

# ---------------------------
# FRONTEND TEMPLATE (Jinja) - NO f-strings, safe for CSS/JS
# ---------------------------
TEMPLATE = """
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width,initial-scale=1">
   <img src="https://live.staticflickr.com/90/266629056_c5f7107f09_h.jpg"
         style="width:100%; height:180px; object-fit:cover;">
  <title>Glampings Guatavita — Dashboard</title>
  <link href="https://fonts.googleapis.com/css2?family=Inter:wght@300;400;600&display=swap" rel="stylesheet">
  <style>
    :root{ --primary: {{ theme_color }}; --bg:#75eb93; --card:#3b7a5c; --muted:#94a3b8; --accent:#06b6d4; --radius:12px; }
    *{box-sizing:border-box}
    body{font-family:Inter,system-ui,-apple-system,Segoe UI,Roboto,Helvetica,Arial; margin:0; background:linear-gradient(180deg,#071126 0%,#071a2a 100%); color:#e6eef6; padding:18px;}
    .wrap{max-width:1200px;margin:0 auto}
    header{display:flex;justify-content:space-between;align-items:center;margin-bottom:18px}
    h1{margin:0;font-size:20px;color:var(--accent)}
    .sub{color:var(--muted); font-size:13px}
    .layout{display:grid;grid-template-columns:320px 1fr;gap:16px}
    .card{background:linear-gradient(180deg, rgba(255,255,255,0.02), rgba(255,255,255,0.01)); padding:16px;border-radius:10px; box-shadow: 0 6px 30px rgba(2,6,23,0.6)}
    .menu-item{padding:10px;border-radius:8px;display:block;color:var(--muted);text-decoration:none;margin-bottom:6px}
    label{font-size:13px;color:var(--muted);display:block;margin-bottom:6px}
    input, select, textarea{width:100%;padding:10px;border-radius:8px;border:1px solid rgba(255,255,255,0.04);background:transparent;color:#fff}
    .btn{background:var(--accent); color:#022; border:none;padding:10px 12px;border-radius:8px;cursor:pointer;font-weight:600}
    table{width:100%;border-collapse:collapse;margin-top:8px}
    th,td{padding:8px;text-align:left;border-bottom:1px solid rgba(255,255,255,0.03);font-size:13px}
    pre{background:transparent;border:0;color:var(--muted);white-space:pre-wrap;font-size:13px}
    .charts{display:grid;grid-template-columns:repeat(2,1fr);gap:12px;margin-top:12px}
    img.chart{width:100%;border-radius:8px;border:1px solid rgba(255,255,255,0.03);background:#071326}
    footer{margin-top:12px;color:var(--muted);font-size:13px}
    @media(max-width:900px){ .layout{grid-template-columns:1fr} .charts{grid-template-columns:1fr} }
  </style>
</head>
<body>
  <div class="wrap">
    <header>
      <div>
        <h1>🏕️ Glampings Guatavita — Dashboard</h1>
        <div class="sub">Registro, análisis y exportación local</div>
      </div>
      <div style="text-align:right">
        <div class="sub">Archivo local: <strong>{{ csv_file }}</strong></div>
        <div style="height:8px"></div>
        <a href="/" style="text-decoration:none"><button class="btn">Recargar</button></a>
      </div>
    </header>

    <div class="layout">
      <!-- SIDEBAR -->
      <aside>
        <div class="card">
          <h3 style="margin-top:0">Registrar visitante</h3>
          <form action="/add" method="post">
            <label>Nombre</label><input name="nombre" required placeholder="Nombre completo">
            <label>Edad</label><input type="number" name="edad" min="0" placeholder="Ej: 28">
            <label>Ciudad</label><input name="ciudad" placeholder="Ciudad">
            <label>Dinero gastado</label><input type="number" step="any" name="dinero_gastado" placeholder="0">
            <label>Glamping</label>
            <select name="glamping" required>
              <option value="">-- Seleccione --</option>
              <option>Glamping La Laguna</option>
              <option>Glamping Mirador</option>
              <option>Glamping Bosque Mágico</option>
              <option>Glamping Estrella</option>
            </select>
            <label>Tipo de viajero</label>
            <select name="tipo_viajero" required>
              <option value="">-- Seleccione --</option>
              <option>Aventurero</option><option>Familiar</option><option>Pareja</option>
              <option>Amigos</option><option>Lujo</option><option>Ecoturista</option>
            </select>
            <label>Fecha</label><input type="date" name="fecha_visita">
            <label>Comentarios</label><textarea name="comentarios" rows="2" placeholder="Notas..."></textarea>
            <div style="display:flex;gap:8px;margin-top:10px">
              <button class="btn" type="submit">➕ Guardar</button>
              <a href="/export"><button type="button" class="btn" style="background:#fff;color:#022">Exportar</button></a>
            </div>
          </form>

          <hr style="border:none;border-top:1px solid rgba(255,255,255,0.03);margin:12px 0">

          <h4 style="margin:0 0 8px 0">Eliminar por ID</h4>
          <form method="post" action="/delete" style="display:flex;gap:8px">
            <input name="delete_id" placeholder="ID a eliminar" style="width:100px">
            <button class="btn" type="submit">Eliminar</button>
          </form>
        </div>
      </aside>

      <!-- MAIN -->
      <main>
        <div class="card">
          <div style="display:flex;justify-content:space-between;align-items:center">
            <h3 style="margin:0">📋 Registros</h3>
            <form method="get" action="/" style="display:inline">
              <div style="display:flex;gap:8px;align-items:center">
                <select name="filter_glamping"><option value="">Todos glampings</option>{% for opt in glamping_options %}<option value="{{ opt }}">{{ opt }}</option>{% endfor %}</select>
                <select name="filter_tipo"><option value="">Todos tipos</option>{% for opt in tipo_options %}<option value="{{ opt }}">{{ opt }}</option>{% endfor %}</select>
                <button class="btn" type="submit">Filtrar</button>
              </div>
            </form>
          </div>

          <div style="margin-top:12px">
            <pre id="tablePre">{{ table_text }}</pre>
          </div>
        </div>

        <div class="card" style="margin-top:12px">
          <h3 style="margin:0 0 8px 0">📊 Gráficas & Estadísticas</h3>
          <div style="display:flex;gap:12px;align-items:center">
            <div style="flex:1">
              <div class="charts">
                <img class="chart" src="data:image/png;base64,{{ g_glamping }}">
                <img class="chart" src="data:image/png;base64,{{ g_tipo }}">
                <img class="chart" src="data:image/png;base64,{{ g_age }}">
                <img class="chart" src="data:image/png;base64,{{ g_money }}">
              </div>
            </div>
            <div style="width:260px">
              <h4 style="margin:0">Estadísticas</h4>
              <pre>{{ stats_text }}</pre>
            </div>
          </div>
        </div>

        <footer style="margin-top:10px" class="card">
          <div style="display:flex;justify-content:space-between;align-items:center">
            <div class="sub">Archivo local: <strong>{{ csv_file }}</strong></div>
            <div class="sub">Ngrok: <strong>{{ public_url }}</strong></div>
          </div>
        </footer>
      </main>
    </div>
  </div>
</body>
</html>
"""

from flask import render_template_string

# ---------------------------
# FLASK APP
# ---------------------------
app = Flask(__name__, static_url_path='/static')
app.config['PROPAGATE_EXCEPTIONS'] = True
app.debug = True
def escape_html(s):
    import html
    return html.escape(s)

def format_table(df):
    if df.empty:
        return "No hay registros."
    max_rows = 200
    if len(df) > max_rows:
        df_sh = df.tail(max_rows)
        txt = df_sh.to_string(index=False)
        txt += f"\n\n(Se muestran últimos {max_rows} registros de {len(df)})"
        return txt
    return df.to_string(index=False)

def compute_stats(df):
    if df.empty:
        return "Sin datos."
    total = len(df)
    ages = pd.to_numeric(df['edad'], errors='coerce').dropna()
    avg_age = f"{ages.mean():.2f}" if not ages.empty else "N/A"
    money = pd.to_numeric(df['dinero_gastado'], errors='coerce').dropna()
    avg_money = f"{money.mean():.2f}" if not money.empty else "N/A"
    top_glamping = df['glamping'].replace("", np.nan).dropna().value_counts().idxmax() if not df['glamping'].replace("", np.nan).dropna().empty else "N/A"
    top_tipo = df['tipo_viajero'].replace("", np.nan).dropna().value_counts().idxmax() if not df['tipo_viajero'].replace("", np.nan).dropna().empty else "N/A"
    return f"Total visitantes: {total} \n\nEdad promedio: {avg_age}\n\nGasto promedio: {avg_money}\n\nGlamping más frecuente: {top_glamping}\n\nTipo de viajero más frecuente: {top_tipo}"

@app.route("/", methods=["GET"])
def index():
    df = read_data()
    fg = request.args.get('filter_glamping','') or ''
    ft = request.args.get('filter_tipo','') or ''
    dff = df.copy()
    if fg:
        dff = dff[dff['glamping'] == fg]
    if ft:
        dff = dff[dff['tipo_viajero'] == ft]

    table_text = escape_html(format_table(dff))
    stats_text = escape_html(compute_stats(dff))

    g_glamping = plot_visitors_by_glamping(dff)
    g_tipo = plot_visitors_by_tipo(dff)
    g_age = plot_age_distribution(dff)
    g_money = plot_avg_spent_by_glamping(dff)

    glamping_vals = sorted(df['glamping'].replace("", np.nan).dropna().unique().tolist()) if not df.empty else []
    tipo_vals = sorted(df['tipo_viajero'].replace("", np.nan).dropna().unique().tolist()) if not df.empty else []

    # public_url will be set after ngrok connect; if not yet, show pending
    public_url = getattr(app, "public_url", "(ngrok pending)")

    return render_template_string(
        TEMPLATE,
        theme_color=THEME_COLOR,
        csv_file=CSV_FILE,
        table_text=table_text,
        stats_text=stats_text,
        g_glamping=g_glamping,
        g_tipo=g_tipo,
        g_age=g_age,
        g_money=g_money,
        glamping_options=glamping_vals,
        tipo_options=tipo_vals,
        public_url=public_url
    )

@app.route("/add", methods=["POST"])
def add():
    payload = {
        'nombre': request.form.get('nombre','').strip(),
        'edad': request.form.get('edad',''),
        'ciudad': request.form.get('ciudad',''),
        'dinero_gastado': request.form.get('dinero_gastado',''),
        'glamping': request.form.get('glamping','').strip(),
        'tipo_viajero': request.form.get('tipo_viajero','').strip(),
        'fecha_visita': request.form.get('fecha_visita') or datetime.now().strftime('%Y-%m-%d'),
        'comentarios': request.form.get('comentarios','').strip()
    }
    if not payload['nombre'] or not payload['glamping'] or not payload['tipo_viajero']:
        return "<h3>Faltan campos obligatorios (nombre, glamping, tipo_viajero).</h3><a href='/'>Volver</a>"
    append_row(payload)
    return redirect(url_for('index'))

@app.route("/delete", methods=["POST"])
def delete():
    vid = request.form.get('delete_id','')
    ok, msg = delete_by_id(vid)
    return f"<h3>{msg}</h3><a href='/'>Volver</a>"

@app.route("/export", methods=["GET"])
def export_excel():
    df = read_data()
    buf = BytesIO()
    df.to_excel(buf, index=False, engine='openpyxl')
    buf.seek(0)
    return send_file(buf, as_attachment=True, download_name="visitantes.xlsx", mimetype="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet")

# ---------------------------
# Run Flask + ngrok (threaded, safe order)
# ---------------------------
def run_app():
    app.run(host="0.0.0.0", port=5000, debug=False, use_reloader=False)

ensure_csv()

# kill old tunnels and start flask thread
try:
    ngrok.kill()
except Exception:
    pass

threading.Thread(target=run_app, daemon=True).start()
time.sleep(1)

# create ngrok tunnel (simple, compatible)
try:
    tunnel = ngrok.connect(5000, proto="http")
    public_url = str(tunnel)
    app.public_url = public_url
    print("🔗 Tu app pública via ngrok:", public_url)
except Exception as e:
    print("⚠️ ngrok connect fallo:", e)
    app.public_url = "(ngrok failed)"

print("✅ App iniciada — abre la URL pública en otra pestaña.")
