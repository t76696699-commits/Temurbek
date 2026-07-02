# Temurbek
1. Procfile va gunicorn masalasi
Render yoki boshqa platformalarda ilovangiz gunicorn orqali ishlashi shart.

Procfile faylini loyihangizning root (asosiy) papkasida yaratganingizga ishonch hosil qiling. Undagi mazmun quyidagicha bo‘lishi kerak:
web: gunicorn app:app (agar asosiy faylingiz app.py bo'lsa).

Agar requirements.txt ichida gunicorn yo'q bo'lsa, uni qo'shib, pip freeze > requirements.txt buyrug'i bilan yangilang.

2. .env va SECRET_KEY xavfsizligi
.env faylingiz .gitignore ichida ekanligini tekshiring:

Terminalda git check-ignore -v .env deb yozing. Agar natija qaytsa, demak u .gitignore ga qo'shilgan.

Kod ichida app.config['SECRET_KEY'] = os.getenv('SECRET_KEY') kabi ishlatganingizga ishonch hosil qiling. Hardcode qilingan (matn ko'rinishidagi) kalitlar bo'lmasligi kerak.

3. Production sozlamalari (DEBUG=False)
Ilovangizda debug rejimini config.py yoki app.py ichida os.getenv('DEBUG', 'False') == 'True' qilib belgilang.

Production muhitida (Render/Railway dashboardida) DEBUG o'zgaruvchisini False qilib belgilaganingizni tekshiring.

4. README.md fayli — Bu sizning yuzingiz
Tekshiruvchi birinchi navbatda shu faylni o'qiydi. Unda quyidagilar bo'lishi shart:

Lokal o'rnatish: Loyihani qanday qilib git clone qilish, virtual muhit yaratish va pip install -r requirements.txt qilish bo'yicha bosqichma-bosqich qo'llanma.

Deploy qadamlari: Loyihani qanday qilib Render/Railway'ga ulaganingiz haqida qisqacha ma'lumot.

Demo URL: Ishlayotgan saytingizning havolasini README'ning eng tepasiga, kattaroq shriftda yozing.

API Documentation: REST API ishlayotgan bo'lsa, qaysi endpoint-larga (/api/notes) qanday ma'lumot yuborish kerakligini (misol uchun curl yoki Postman misollari bilan) yozib chiqing.

5. Tekshiruv (Checklist)
Qayta topshirishdan oldin o‘zingizga savol bering:

Men bergan linkga kirganda 404 yoki 500 xatosi chiqyaptimi?

requirements.txt ichida hamma kutubxonalar bormi?

Github repozitoriyam "Public" holatdami?

Maslahat: Agar loyiha Render'da ishlamayotgan bo'lsa, Render dashboardidagi "Logs" bo‘limiga kiring. U yerda aynan qaysi qatorda xatolik chiqayotgani (masalan, modul topilmadi yoki baza ulanmadi) aniq yozilgan bo'ladi.

Ishingizda kamchilik bo'lsa, muammoning loglarini (xatolik matnini) menga yuboring, biz birgalikda uni tuzatamiz. Qaysi qismda qiynalayapsiz?
