# Temurbek
Tushunarli. Capstone loyihangiz "Отклонено" (68/100) deb qaytarilgan bo'lsa, demak, asosiy texnik talablar yoki hujjatlashtirishda (README) kamchilik bor. Keling, ushbu ro'yxatni sinchkovlik bilan tekshirib chiqamiz:

1. Texnik talablar tekshiruvi (Checklist)
Tirik demo URL: Sizning saytingizga tashqi odam kirganda sahifa ochilyaptimi?

Tekshiring: Render yoki Railway dashboardida "Deployment successful" degan yashil belgi bormi?

.env va secret_key:

Tekshiring: GitHub repozitoriyangizni oching. U yerda .env fayli ko'rinib turibdimi? Agar ko'rinsa, bu katta xato! U mutlaqo bo'lmasligi kerak.

Ishlab chiqarish (Production) muhiti: Render yoki boshqa hosting sozlamalarida "Environment Variables" bo'limiga SECRET_KEY va boshqa kalitlarni kiritganmisiz?

Procfile:

Tekshiring: Asosiy papkada Procfile (kengaytmasiz) fayli bormi? Ichida web: gunicorn app:app (yoki main:app) yozuvi bormi?

DEBUG=False:

Tekshiring: app.run(debug=True) kodi kod bazasida qolib ketmadimi? Ishlab chiqarish muhitida debug parametri har doim False bo'lishi kerak.

2. README.md - Muhim qism
Agar texnik qism to'g'ri bo'lsa, tekshiruvchi sizning README faylingizni yetarli emas deb topgan bo'lishi mumkin. README quyidagi tuzilishga ega bo'lishi shart:

Notlar Ilovasi
Demo URL: [Havolani shu yerga qo'ying]

Lokal o'rnatish
Repozitoriyani yuklab oling: git clone ...

Virtual muhitni yoqing va kutubxonalarni o'rnating: pip install -r requirements.txt

.env faylini yarating va SECRET_KEY=... ni yozing.

Ishga tushiring: python app.py

Deploy qadamlari
(Bu yerda loyihani qaysi platformaga va qanday o'tkazganingizni qisqacha yozing)
