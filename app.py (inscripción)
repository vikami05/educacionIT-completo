from flask import Flask, render_template, request, redirect, url_for, session, jsonify, flash
from flask_mysqldb import MySQL
import MySQLdb.cursors
import hashlib

app = Flask(__name__)
app.secret_key = 'supersecretkey'

# Configuración MySQL
app.config['MYSQL_HOST'] = 'localhost'
app.config['MYSQL_USER'] = 'root'
app.config['MYSQL_PASSWORD'] = '1234'
app.config['MYSQL_DB'] = 'educacionit'
app.config['MYSQL_CURSORCLASS'] = 'DictCursor'

mysql = MySQL(app)

# --- UTIL ---
def hash_password(password: str) -> str:
    return hashlib.sha256(password.encode()).hexdigest()

# --- RUTAS ---
@app.route("/")
def seleccion_cursos():
    curso_id = request.args.get("id", "python")
    precio_curso = request.args.get("precio", 1000)

    usuarioYaInscripto = False
    user = None

    if session.get("user_id"):
        cursor = mysql.connection.cursor()
        cursor.execute("SELECT * FROM usuarios WHERE id_usuario=%s", (session["user_id"],))
        user = cursor.fetchone()
        if user:
            cursor.execute("SELECT COUNT(*) AS cnt FROM inscripciones WHERE id_usuario=%s", (session["user_id"],))
            cnt = cursor.fetchone()
            if cnt and cnt['cnt'] > 0:
                usuarioYaInscripto = True
        cursor.close()

    return render_template("seleccion_cursos2.html",
                           usuarioYaInscripto=usuarioYaInscripto,
                           curso_id=curso_id,
                           precio_curso=precio_curso,
                           user=user)

@app.route("/inscribirse", methods=["POST"])
def inscribirse():
    nombre = request.form["nombre"]
    apellido = request.form["apellido"]
    email = request.form["email"]
    fecha_nacimiento = request.form.get("fecha_nacimiento") or None
    dni = request.form.get("dni") or None
    telefono = request.form.get("telefono") or None
    password = request.form["password"]
    curso_str = request.form["curso"]
    precio = request.form.get("precio", 0)

    password_hash = hash_password(password)

    cursos_ids = {
        "python": 1, "java": 2, ".net": 3, "javascript": 4, "ia": 5, "chatgpt": 6,
        "ia-visual": 7, "ia-avanzada": 8, "community-manager": 9, "seo": 10,
        "publicidad-redes": 11, "marketing-contenido": 12, "uiux": 13, "bigdata": 14
    }
    curso_id = cursos_ids.get(curso_str, 1)

    cursor = mysql.connection.cursor()
    
    # Verificar si el usuario ya existe por email o DNI
    cursor.execute("SELECT id_usuario FROM usuarios WHERE email=%s OR dni=%s", (email, dni))
    usuario = cursor.fetchone()

    if usuario:
        # Usuario existente → usar su ID
        user_id = usuario[0]  # si fetchone() devuelve tupla
        flash("Usuario ya existente. Se usará la cuenta existente para esta inscripción.")
    else:
        # Usuario no existe → crear nuevo
        cursor.execute("""
            INSERT INTO usuarios (nombre, apellido, email, password, fecha_nacimiento, telefono, dni)
            VALUES (%s,%s,%s,%s,%s,%s,%s)
        """, (nombre, apellido, email, password_hash, fecha_nacimiento, telefono, dni))
        mysql.connection.commit()
        user_id = cursor.lastrowid

    # Insertar inscripción
    cursor.execute(
        "INSERT INTO inscripciones (id_usuario, id_curso) VALUES (%s,%s)",
        (user_id, curso_id)
    )
    mysql.connection.commit()

    # Guardamos user_id en session
    session['user_id'] = user_id
    session['email'] = email
    cursor.close()

    # Renderizamos la página de pago
    return render_template("pago_cursos2.html",
                           nombre=nombre, apellido=apellido,
                           fecha_nacimiento=fecha_nacimiento, email=email,
                           dni=dni, telefono=telefono,
                           curso_id=curso_str, precio_curso=precio)


# Ruta GET/POST para mostrar página de pago o simular pago
@app.route("/pago", methods=["GET", "POST"])
def pago():
    user_id = session.get("user_id")
    if not user_id:
        return redirect(url_for("seleccion_cursos"))

    cursor = mysql.connection.cursor()
    cursor.execute("SELECT nombre, apellido, email FROM usuarios WHERE id_usuario=%s", (user_id,))
    user = cursor.fetchone()
    cursor.close()

    if request.method == "POST":
        # Simulamos pago
        return "<h3>Pago simulado: aprobado ✅</h3><a href='/'>Volver</a>"
    else:
        # Mostramos la página de pago
        return render_template("pago_cursos2.html", user=user)

# LOGIN simple
@app.route("/login", methods=["GET", "POST"])
def login():
    if request.method == "POST":
        email = request.form['email']
        password = request.form['password']
        cursor = mysql.connection.cursor()
        cursor.execute("SELECT id_usuario, password, nombre FROM usuarios WHERE email=%s", (email,))
        u = cursor.fetchone()
        cursor.close()
        if u and u['password'] == hash_password(password):
            session['user_id'] = u['id_usuario']
            session['email'] = email
            flash("Inicio de sesión correcto", "success")
            return redirect(url_for('seleccion_cursos'))
        else:
            flash("Email o contraseña incorrectos", "danger")
            return redirect(url_for('login'))
    return render_template("login.html")

@app.route("/logout")
def logout():
    session.pop('user_id', None)
    session.pop('email', None)
    flash("Sesión cerrada", "info")
    return redirect(url_for('seleccion_cursos'))

# Endpoint AJAX opcional para chequear email
@app.route("/check_email", methods=["POST"])
def check_email():
    email = request.json.get("email")
    cursor = mysql.connection.cursor()
    cursor.execute("SELECT id_usuario FROM usuarios WHERE email=%s", (email,))
    found = cursor.fetchone() is not None
    cursor.close()
    return jsonify({"exists": found})

if __name__ == "__main__":
    app.run(debug=True)
