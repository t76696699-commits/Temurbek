1. app/__init__.py
SQLAlchemy obyekti tashqarida yaratiladi va create_app funksiyasi (factory) ichida db.init_app(app) orqali ilovaga bog'lanadi.

Python
from flask import Flask
from flask_sqlalchemy import SQLAlchemy

# SQLAlchemy obyektini yaratish
db = SQLAlchemy()

def create_app():
    app = Flask(__name__)
    
    # SQLite ma'lumotlar bazasi konfiguratsiyasi
    app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///app.db'
    app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
    app.config['SECRET_KEY'] = 'sizning_maxfiy_kalitingiz' # Form xavfsizligi uchun

    # Bazani ilova bilan bog'lash
    db.init_app(app)

    # Blueprint'larni ro'yxatdan o'tkazish
    from app.main.routes import main_bp
    from app.notes.routes import notes_bp

    app.register_blueprint(main_bp)
    app.register_blueprint(notes_bp, url_prefix='/notes')

    # Ma'lumotlar bazasi jadvallarini yaratish
    with app.app_context():
        db.create_all()

    return app
2. app/models.py
Talab etilgan Note modeli: id, title (String 200), body (Text), hamda avtomatik vaqt qo'yuvchi created_at maydonlari bilan.

Python
from app import db
from datetime import datetime, timezone

class Note(db.Model):
    __tablename__ = 'notes'

    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(200), nullable=False)
    body = db.Column(db.Text, nullable=False)
    created_at = db.Column(db.DateTime, default=lambda: datetime.now(timezone.utc))

    def __repr__(self):
        return f'<Note {self.title}>'
3. app/main/routes.py
Bosh sahifaga kirganda bazadagi notalar yangi-eski tartibda (order_by(Note.created_at.desc())) ko'rsatiladi.

Python
from flask import Blueprint, render_template
from app.models import Note

main_bp = Blueprint('main', __name__)

@main_bp.route('/')
def index():
    # Bazadan yangi yaratilgan eslatmalarni birinchi qilib olish (yangi-eski tartibi)
    recent_notes = Note.query.order_by(Note.created_at.desc()).all()
    return render_template('main_index.html', notes=recent_notes)
4. app/notes/routes.py
Bu yerda ham ro'yxat ko'rsatiladi, ham POST so'rovi orqali yangi nota qo'shiladi. Xato yuz berganda db.session.rollback() bajarilib, sessiya toza holatga keltiriladi.

Python
from flask import Blueprint, render_template, request, redirect, url_for, flash
from app import db
from app.models import Note

notes_bp = Blueprint('notes', __name__)

@notes_bp.route('/', methods=['GET', 'POST'])
def list_notes():
    if request.method == 'POST':
        title = request.form.get('title')
        body = request.form.get('body')

        if not title or not body:
            flash("Sarlavha va matn to'ldirilishi shart!", "error")
            return redirect(url_for('notes.list_notes'))

        new_note = Note(title=title, body=body)
        
        try:
            db.session.add(new_note)
            db.session.commit()
            flash("Yangi eslatma muvaffaqiyatli qo'shildi!", "success")
            return redirect(url_for('notes.list_notes'))
        except Exception as e:
            # Xatolik yuz berganda sessiyani orqaga qaytarish (rollback)
            db.session.rollback()
            flash(f"Xatolik yuz berdi bazaga yozilmadi: {str(e)}", "error")
            return redirect(url_for('notes.list_notes'))

    # GET so'rovi bo'lganda barcha notalarni yangi-eski tartibda chiqarish
    all_notes = Note.query.order_by(Note.created_at.desc()).all()
    return render_template('notes_list.html', notes=all_notes)
5. app/templates/base.html
Xabarlarni (Flash messages) chiqarish qismi qo'shilgan umumiy maket.

HTML
<!DOCTYPE html>
<html lang="uz">
<head>
    <meta charset="UTF-8">
    <title>Flask DB Loyihasi</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 40px; background-color: #f4f4f4; }
        nav { margin-bottom: 20px; padding: 10px; background: #fff; border-radius: 5px; }
        nav a { margin-right: 15px; text-decoration: none; color: #007BFF; font-weight: bold; }
        .container { background: #fff; padding: 20px; border-radius: 5px; box-shadow: 0 0 10px rgba(0,0,0,0.1); }
        .note-card { border-left: 4px solid #28a745; padding: 10px; margin-bottom: 15px; background: #fafafa; }
        .note-date { font-size: 0.8em; color: #777; }
        .form-group { margin-bottom: 15px; }
        .form-group label { display: block; margin-bottom: 5px; font-weight: bold; }
        .form-group input, .form-group textarea { width: 100%; padding: 8px; box-sizing: border-box; }
        button { padding: 10px 15px; background: #007BFF; color: white; border: none; border-radius: 3px; cursor: pointer; }
        .flash { padding: 10px; margin-bottom: 15px; border-radius: 3px; }
        .flash.success { background: #d4edda; color: #155724; }
        .flash.error { background: #f8d7da; color: #721c24; }
    </style>
</head>
<body>

    <nav>
        <a href="{{ url_for('main.index') }}">Bosh sahifa (Yangi-Eski)</a>
        <a href="{{ url_for('notes.list_notes') }}">Eslatmalar paneli (Qo'shish)</a>
    </nav>

    <div class="container">
        <!-- Flash xabarlarni chiqarish -->
        {% with messages = get_flashed_messages(with_categories=true) %}
          {% if messages %}
            {% for category, message in messages %}
              <div class="flash {{ category }}">{{ message }}</div>
            {% endfor %}
          {% endif %}
        {% endwith %}

        {% block content %} {% endblock %}
    </div>

</body>
</html>
6. app/templates/main_index.html
Bosh sahifada faqat bazadagi eslatmalar tartiblangan ro'yxati chiqadi.

HTML
{% extends 'base.html' %}

{% block content %}
    <h1>Bosh sahifa: Eng yangi eslatmalar</h1>
    <p>Bu yerda barcha yozuvlar yaratilgan vaqtiga ko'ra teskari tartibda (yangi-eski) ko'rsatiladi.</p>
    <hr>

    {% if notes %}
        {% for note in notes %}
            <div class="note-card">
                <h3>{{ note.title }}</h3>
                <p>{{ note.body }}</p>
                <div class="note-date">Yaratilgan vaqti: {{ note.created_at.strftime('%Y-%m-%d %H:%M') }}</div>
            </div>
        {% endfor %}
    {% else %}
        <p>Hozircha eslatmalar yo'q. <a href="{{ url_for('notes.list_notes') }}">Birinchisini qo'shing!</a></p>
    {% endif %}
{% endblock %}
7. app/templates/notes_list.html
Yangi nota qo'shish uchun HTML formasi (POST) va uning ostida mavjud notalar ro'yxati.

HTML
{% extends 'base.html' %}

{% block content %}
    <h2>Yangi eslatma qo'shish</h2>
    <!-- POST formasi -->
    <form action="{{ url_for('notes.list_notes') }}" method="POST">
        <div class="form-group">
            <label for="title">Sarlavha (Title):</label>
            <input type="text" id="title" name="title" required maxlength="200">
        </div>
        <div class="form-group">
            <label for="body">Matn (Body):</label>
            <textarea id="body" name="body" rows="4" required></textarea>
        </div>
        <button type="submit">Saqlash</button>
    </form>

    <br><hr><br>

    <h2>Mavjud eslatmalar ro'yxati</h2>
    {% for note in notes %}
        <div class="note-card" style="border-left-color: #007BFF;">
            <h3>{{ note.title }}</h3>
            <p>{{ note.body }}</p>
            <div class="note-date">Vaqt: {{ note.created_at.strftime('%Y-%m-%d %H:%M') }}</div>
        </div>
    {% endfor %}
{% endblock %}
8. run.py
Python
from app import create_app

app = create_app()

if __name__ == '__main__':
    app.run(debug=True)
Seed: Flask Shell orqali kamida 5 ta boshlang'ich nota qo'shish
Ilovani brauzerda ochishdan oldin ma'lumotlar bazasini boshlang'ich ma'lumotlar bilan to'ldirish (seed) uchun terminalda quyidagi amallarni bajaring:

Terminalda loyiha papkasiga kiring va Flask shellni yoqing:

Bash
flask shell
Shell ichiga quyidagi Python kodlarini navbatma-navbat nusxalab kiriting (har bir qatordan keyin Enter bosing):

Python
from app import db
from app.models import Note

# 5 ta eslatma obyektini yaratish
n1 = Note(title="Birinchi eslatma", body="Bu SQLite bazasidagi eng birinchi yozuv.")
n2 = Note(title="Flask-SQLAlchemy", body="Factory pattern bilan db obyekti to'g'ri ulandi.")
n3 = Note(title="SQLite ma'lumotlari", body="SQLite lokal instance papkasida app.db fayliga yoziladi.")
n4 = Note(title="Shell orqali Seed", body="Ushbu ma'lumotlar terminal (flask shell) orqali kiritildi.")
n5 = Note(title="Teskari tartib testi", body="Ushbu nota ro'yxatning eng tepasida chiqishi kerak.")

# Obyektlarni sessiyaga qo'shish va saqlash
db.session.add_all([n1, n2, n3, n4, n5])
db.session.commit()

# Shelldan chiqish
exit()
