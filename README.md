Loyiha Tuzilishi (Directory Structure)
Loyihangiz papkalar ierarxiyasi quyidagicha bo'lishi kerak:

Plaintext
flask_app/
│
├── app/
│   ├── __init__.py          # App factory shu yerda joylashadi
│   ├── main/
│   │   └── routes.py        # Main blueprint yo'llari
│   ├── notes/
│   │   └── routes.py        # Notes blueprint yo'llari va xotira-ichidagi ma'lumotlar
│   └── templates/
│       ├── base.html        # Umumiy shablon (baza)
│       ├── main_index.html  # Asosiy sahifa
│       └── notes_list.html  # Eslatmalar ro'yxati sahifasi
│
└── run.py                   # Loyihani ishga tushiruvchi fayl
Kodlar
1. app/__init__.py (Application Factory)
Bu yerda Flask ilovasi yaratiladi va Blueprint'lar ro'yxatdan o'tkaziladi. notes_bp uchun talab qilinganidek /notes prefiksi o'rnatilgan.

Python
from flask import Flask

def create_app():
    app = Flask(__name__)

    # Blueprint'larni import qilish
    from app.main.routes import main_bp
    from app.notes.routes import notes_bp

    # Blueprint'larni ro'yxatdan o'tkazish
    app.register_blueprint(main_bp)
    app.register_blueprint(notes_bp, url_prefix='/notes')

    return app
2. app/main/routes.py
Asosiy sahifa uchun Blueprint va uning yo'li (route).

Python
from flask import Blueprint, render_template

main_bp = Blueprint('main', __name__)

@main_bp.route('/')
def index():
    return render_template('main_index.html')
3. app/notes/routes.py
Bu yerda kamida 5 ta eslatmadan iborat Python ro'yxati (list) xotirada saqlanadi va sahifaga uzatiladi.

Python
from flask import Blueprint, render_template

notes_bp = Blueprint('notes', __name__)

# Xotira ichidagi eslatmalar ro'yxati (Kamida 5 ta)
NOTES_DATA = [
    {"id": 1, "title": "Bozorlik ro'yxati", "content": "Sut, non, sariyog' va mevalar sotib olish."},
    {"id": 2, "title": "Flask o'rganish", "content": "Blueprint va Application Factory mavzularini takrorlash."},
    {"id": 3, "title": "Ertangi rejalar", "content": "Ertalab soat 7:00 da yugurish va kitob o'qish."},
    {"id": 4, "title": "Loyihani topshirish", "content": "Flask loyihasini Github-ga yuklash va linkini yuborish."},
    {"id": 5, "title": "Shaxsiy eslatma", "content": "Dasturlash bo'yicha yangi videolarni tomosha qilish."}
]

@notes_bp.route('/')
def list_notes():
    return render_template('notes_list.html', notes=NOTES_DATA)
4. app/templates/base.html
Barcha sahifalar uchun umumiy bo'lgan HTML maketi. Undagi linklar url_for orqali dinamik yaratilgan.

HTML
<!DOCTYPE html>
<html lang="uz">
<head>
    <meta charset="UTF-8">
    <title>Flask Blueprint Loyihasi</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 40px; background-color: #f4f4f4; }
        nav { margin-bottom: 20px; padding: 10px; background: #fff; border-radius: 5px; }
        nav a { margin-right: 15px; text-decoration: none; color: #007BFF; font-weight: bold; }
        .container { background: #fff; padding: 20px; border-radius: 5px; box-shadow: 0 0 10px rgba(0,0,0,0.1); }
        .note-card { border-left: 4px solid #007BFF; padding: 10px; margin-bottom: 10px; background: #fafafa; }
    </style>
</head>
<body>

    <nav>
        <!-- url_for('blueprint.funksiya') formati -->
        <a href="{{ url_for('main.index') }}">Bosh sahifa</a>
        <a href="{{ url_for('notes.list_notes') }}">Eslatmalar</a>
    </nav>

    <div class="container">
        {% block content %} {% endblock %}
    </div>

</body>
</html>
5. app/templates/main_index.html
Asosiy sahifa kontenti.

HTML
{% extends 'base.html' %}

{% block content %}
    <h1>Xush kelibsiz!</h1>
    <p>Bu Flask Application Factory va Blueprint arxitekturasi asosida yaratilgan loyiha.</p>
    <p>Eslatmalarni ko'rish uchun yuqoridagi menyudan foydalaning yoki <a href="{{ url_for('notes.list_notes') }}">bu yerga bosing</a>.</p>
{% endblock %}
6. app/templates/notes_list.html
Xotiradagi eslatmalarni aylanib chiquvchi (loop) shablon.

HTML
{% extends 'base.html' %}

{% block content %}
    <h1>Mening Eslatmalarim</h1>
    <hr>
    {% for note in notes %}
        <div class="note-card">
            <h3>{{ note.title }}</h3>
            <p>{{ note.content }}</p>
        </div>
    {% endfor %}
{% endblock %}
7. run.py
Loyihani ishga tushirish uchun asosiy fayl.

Python
from app import create_app

app = create_app()

if __name__ == '__main__':
    app.run(debug=True)
Loyihani ishga tushirish:
Terminalda loyiha joylashgan asosiy papkaga (flask_app/) kiring.

Quyidagi buyruq orqali loyihani ishga tushiring:

Bash
python run.py
Brauzerda [http://127.0.0.1:5000/](http://127.0.0.1:5000/) manziliga kirsangiz Bosh sahifa, [http://127.0.0.1:5000/notes/](http://127.0.0.1:5000/notes/) manziliga kirsangiz esa eslatmalar ro'yxati ochiladi. Shuningdek, menyudagi barcha havolalar url_for orqali to'g'ri bog'langan.
