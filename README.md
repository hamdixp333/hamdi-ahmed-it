from flask import Flask, render_template, request, redirect, url_for, flash
from flask_sqlalchemy import SQLAlchemy
from flask_login import LoginManager, UserMixin, login_user, login_required, logout_user, current_user
import os
import datetime

app = Flask(__name__)
app.config['SECRET_KEY'] = 'your_secret_key_here' # **تأكد من تغيير هذا في بيئة الإنتاج بمفتاح سري قوي**
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///database.db' # مسار ملف قاعدة البيانات
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False # لتجنب رسائل التحذير من Flask-SQLAlchemy
db = SQLAlchemy(app) # تهيئة كائن قاعدة البيانات

login_manager = LoginManager() # تهيئة مدير تسجيل الدخول
login_manager.init_app(app) # ربط مدير تسجيل الدخول بالتطبيق
login_manager.login_view = 'login' # توجيه المستخدم لصفحة 'login' في حال محاولة الوصول لصفحة محمية بدون تسجيل دخول

# --- نماذج قاعدة البيانات (Database Models) ---
# تمثل هذه الفئات الجداول في قاعدة البيانات
class User(UserMixin, db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    password = db.Column(db.String(200), nullable=False) # يجب تخزين كلمات مرور مشفرة (hashed passwords)
    is_admin = db.Column(db.Boolean, default=False) # لتحديد ما إذا كان المستخدم مسؤولاً

    def __repr__(self):
        return f'<User {self.username}>'

class NewsArticle(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(200), nullable=False)
    content = db.Column(db.Text, nullable=False)
    image_url = db.Column(db.String(500)) # رابط الصورة (اختياري)
    source = db.Column(db.String(100)) # مصدر الخبر (مثل BBC, Reuters)
    published_at = db.Column(db.DateTime, default=datetime.datetime.now) # تاريخ ووقت النشر
    category = db.Column(db.String(50)) # فئة الخبر (مثل رياضة, سياسة, اقتصاد)

    def __repr__(self):
        return f'<NewsArticle {self.title}>'

# --- دالة تحميل المستخدم لـ Flask-Login ---
# تُستخدم هذه الدالة بواسطة Flask-Login لإعادة تحميل كائن المستخدم من معرف المستخدم (ID) المخزن في الجلسة
@login_manager.user_loader
def load_user(user_id):
    return User.query.get(int(user_id))

# --- المسارات (Routes) - منطق الصفحات ---

# الصفحة الرئيسية: تعرض أحدث المقالات الإخبارية
@app.route('/')
def index():
    # جلب آخر 10 مقالات إخبارية مرتبة تنازليًا حسب تاريخ النشر
    articles = NewsArticle.query.order_by(NewsArticle.published_at.desc()).limit(10).all()
    return render_template('index.html', articles=articles)

# صفحة تفاصيل الخبر: تعرض المحتوى الكامل لخبر محدد
@app.route('/news/<int:news_id>')
def news_detail(news_id):
    # جلب المقال بالمعرف الخاص به، أو إرجاع خطأ 404 إذا لم يتم العثور عليه
    article = NewsArticle.query.get_or_404(news_id)
    return render_template('news_detail.html', article=article)

# صفحة تسجيل المستخدمين الجدد
@app.route('/register', methods=['GET', 'POST'])
def register():
    # إذا كان المستخدم مسجل دخول بالفعل، قم بتوجيهه للصفحة الرئيسية
    if current_user.is_authenticated:
        return redirect(url_for('index'))
    if request.method == 'POST':
        username = request.form.get('username')
        password = request.form.get('password') # **تنبيه: يجب تشفير كلمة المرور هنا قبل الحفظ!**
                                              # استخدم مكتبة مثل Werkzeug.security.generate_password_hash
        existing_user = User.query.filter_by(username=username).first()
        if existing_user:
            flash('اسم المستخدم موجود بالفعل. الرجاء اختيار اسم آخر.', 'error')
        else:
            # مثال على تشفير كلمة المرور (تحتاج لاستيراد generate_password_hash من werkzeug.security)
            # from werkzeug.security import generate_password_hash
            # hashed_password = generate_password_hash(password, method='sha256')
            # new_user = User(username=username, password=hashed_password)
            new_user = User(username=username, password=password) # مثال بسيط بدون تشفير (للتطوير فقط)
            db.session.add(new_user)
            db.session.commit()
            flash('تم التسجيل بنجاح! الرجاء تسجيل الدخول.', 'success')
            return redirect(url_for('login'))
    return render_template('register.html')

# صفحة تسجيل الدخول
@app.route('/login', methods=['GET', 'POST'])
def login():
    # إذا كان المستخدم مسجل دخول بالفعل، قم بتوجيهه للصفحة الرئيسية
    if current_user.is_authenticated:
        return redirect(url_for('index'))
    if request.method == 'POST':
        username = request.form.get('username')
        password = request.form.get('password')
        user = User.query.filter_by(username=username).first()
        # **تنبيه: يجب التحقق من كلمة المرور المشفرة هنا!**
        # استخدم مكتبة مثل Werkzeug.security.check_password_hash
        # if user and check_password_hash(user.password, password):
        if user and user.password == password: # مثال بسيط بدون تشفير (للتطوير فقط)
            login_user(user) # تسجيل دخول المستخدم
            flash('تم تسجيل الدخول بنجاح!', 'success')
            next_page = request.args.get('next') # التوجيه للصفحة التي كان يحاول الوصول إليها قبل تسجيل الدخول
            return redirect(next_page or url_for('index'))
        else:
            flash('اسم المستخدم أو كلمة المرور غير صحيحة.', 'error')
    return render_template('login.html')

# تسجيل الخروج
@app.route('/logout')
@login_required # تتطلب أن يكون المستخدم مسجلاً للدخول للوصول لهذا المسار
def logout():
    logout_user() # إنهاء جلسة المستخدم
    flash('لقد تم تسجيل خروجك.', 'message')
    return redirect(url_for('index'))

# صفحة المسؤول: إضافة خبر جديد (متاح للمسؤولين فقط)
@app.route('/admin/add_news', methods=['GET', 'POST'])
@login_required # تتطلب تسجيل الدخول
def add_news():
    # التحقق مما إذا كان المستخدم الحالي مسؤولاً
    if not current_user.is_admin:
        flash('ليس لديك صلاحية للوصول إلى هذه الصفحة.', 'error')
        return redirect(url_for('index'))
    if request.method == 'POST':
        title = request.form.get('title')
        content = request.form.get('content')
        image_url = request.form.get('image_url')
        source = request.form.get('source')
        category = request.form.get('category')

        if not title or not content:
            flash('العنوان والمحتوى مطلوبان.', 'error')
        else:
            new_article = NewsArticle(title=title, content=content,
                                      image_url=image_url, source=source,
                                      category=category)
            db.session.add(new_article)
            db.session.commit()
            flash('تمت إضافة المقال الإخباري بنجاح!', 'success')
            return redirect(url_for('index')) # يمكنك التوجيه لصفحة إدارة الأخبار في لوحة التحكم لاحقًا
    return render_template('admin/add_news.html')

# وظيفة البحث عن الأخبار
@app.route('/search')
def search():
    query = request.args.get('q', '') # الحصول على كلمة البحث من معلمات URL
    if query:
        # البحث في العنوان أو المحتوى (بحث بسيط، يمكن تحسينه لنتائج أفضل)
        results = NewsArticle.query.filter(
            (NewsArticle.title.contains(query)) |
            (NewsArticle.content.contains(query))
        ).order_by(NewsArticle.published_at.desc()).all()
    else:
        results = []
    return render_template('search_results.html', query=query, results=results)

# --- تهيئة قاعدة البيانات عند أول تشغيل ---
# هذه الدالة تُنفذ مرة واحدة فقط قبل أول طلب للموقع
@app.before_first_request
def create_tables():
    db.create_all() # إنشاء جميع الجداول المعرفة في نماذج SQLAlchemy

    # إنشاء مستخدم مسؤول افتراضي إذا لم يكن موجودًا بالفعل
    # **مهم: غيّر 'admin_password' في بيئة الإنتاج!**
    if not User.query.filter_by(username='admin').first():
        admin_user = User(username='admin', password='admin_password', is_admin=True) # يجب تشفير كلمة المرور هنا!
        db.session.add(admin_user)
        db.session.commit()
        print("************************************************************")
        print("تم إنشاء مستخدم مسؤول افتراضي:")
        print("اسم المستخدم: admin")
        print("كلمة المرور: admin_password (الرجاء تغييرها فوراً في الإنتاج!)")
        print("************************************************************")


# تشغيل التطبيق
if __name__ == '__main__':
    # debug=True مفيد أثناء التطوير لأنه يعيد تحميل الخادم تلقائيًا عند حفظ التغييرات
    # ويعرض رسائل الأخطاء في المتصفح. اجعله False في بيئة الإنتاج للأمان.
    app.run(debug=True)
<!DOCTYPE html>
<html lang="ar" dir="rtl"> {# لغة عربية واتجاه من اليمين لليسار #}
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}أخبار العالم{% endblock %}</title>
    <link rel="stylesheet" href="{{ url_for('static', filename='css/style.css') }}">
</head>
<body>
    <header>
        <nav>
            <div class="logo">
                <a href="{{ url_for('index') }}">مركز أخبار العالم</a>
            </div>
            <ul class="nav-links">
                <li><a href="{{ url_for('index') }}">الرئيسية</a></li>
                {# عرض روابط مختلفة بناءً على حالة تسجيل الدخول #}
                {% if current_user.is_authenticated %}
                    <li>مرحبًا، {{ current_user.username }}!</li>
                    {% if current_user.is_admin %}
                        <li><a href="{{ url_for('add_news') }}">إضافة خبر</a></li>
                    {% endif %}
                    <li><a href="{{ url_for('logout') }}">تسجيل الخروج</a></li>
                {% else %}
                    <li><a href="{{ url_for('login') }}">تسجيل الدخول</a></li>
                    <li><a href="{{ url_for('register') }}">التسجيل</a></li>
                {% endif %}
            </ul>
            <div class="search-bar">
                <form action="{{ url_for('search') }}" method="GET">
                    <input type="text" name="q" placeholder="ابحث عن الأخبار..." value="{{ request.args.get('q', '') }}">
                    <button type="submit">بحث</button>
                </form>
            </div>
        </nav>
    </header>

    <main>
        {# لعرض رسائل الفلاش (مثل رسائل النجاح أو الخطأ) #}
        {% with messages = get_flashed_messages(with_categories=true) %}
            {% if messages %}
                <ul class="flash-messages">
                    {% for category, message in messages %}
                        <li class="{{ category }}">{{ message }}</li>
                    {% endfor %}
                </ul>
            {% endif %}
        {% endwith %}
        {# هنا سيتم وضع محتوى كل صفحة فرعية #}
        {% block content %}{% endblock %}
    </main>

    <footer>
        <p>&copy; 2025 مركز أخبار العالم. جميع الحقوق محفوظة.</p>
    </footer>

    <script src="{{ url_for('static', filename='js/script.js') }}"></script>
</body>
</html>
{% extends 'base.html' %} {# يرث من القالب الأساسي #}

{% block title %}آخر أخبار العالم{% endblock %} {# يحدد عنوان الصفحة #}

{% block content %} {# محتوى هذه الصفحة سيظهر هنا #}
    <h1>آخر الأخبار</h1>
    <div class="news-grid">
        {% if articles %}
            {# حلقة تكرارية لعرض كل مقال #}
            {% for article in articles %}
                <div class="news-card">
                    {% if article.image_url %}
                        <img src="{{ article.image_url }}" alt="{{ article.title }}">
                    {% endif %}
                    <h2><a href="{{ url_for('news_detail', news_id=article.id) }}">{{ article.title }}</a></h2>
                    <p class="news-meta">
                        المصدر: {{ article.source | default('غير معروف') }} |
                        النشر: {{ article.published_at.strftime('%Y-%m-%d %H:%M') }}
                    </p>
                    <p>{{ article.content[:150] }}...</p> {# عرض جزء من المحتوى #}
                    <a href="{{ url_for('news_detail', news_id=article.id) }}" class="read-more">قراءة المزيد</a>
                </div>
            {% endfor %}
        {% else %}
            <p>لا توجد مقالات إخبارية متاحة في الوقت الحالي.</p>
        {% endif %}
    </div>
{% endblock %}
{% extends 'base.html' %}

{% block title %}{{ article.title }}{% endblock %}

{% block content %}
    <div class="news-detail">
        <h1>{{ article.title }}</h1>
        <p class="news-meta">
            المصدر: {{ article.source | default('غير معروف') }} |
            النشر: {{ article.published_at.strftime('%Y-%m-%d %H:%M') }} |
            الفئة: {{ article.category | default('عام') }}
        </p>
        {% if article.image_url %}
            <img src="{{ article.image_url }}" alt="{{ article.title }}" class="detail-image">
        {% endif %}
        <div class="news-content">
            <p>{{ article.content }}</p>
        </div>
        {# يمكنك إضافة قسم التعليقات هنا في تطبيق حقيقي #}
    </div>
{% endblock %}
{% extends 'base.html' %}

{% block title %}تسجيل الدخول{% endblock %}

{% block content %}
    <div class="auth-form">
        <h1>تسجيل الدخول</h1>
        <form method="POST">
            <label for="username">اسم المستخدم:</label>
            <input type="text" id="username" name="username" required>
            <label for="password">كلمة المرور:</label>
            <input type="password" id="password" name="password" required>
            <button type="submit">تسجيل الدخول</button>
        </form>
        <p>ليس لديك حساب؟ <a href="{{ url_for('register') }}">سجل هنا</a>.</p>
    </div>
{% endblock %}
{% extends 'base.html' %}

{% block title %}التسجيل{% endblock %}

{% block content %}
    <div class="auth-form">
        <h1>التسجيل</h1>
        <form method="POST">
            <label for="username">اسم المستخدم:</label>
            <input type="text" id="username" name="username" required>
            <label for="password">كلمة المرور:</label>
            <input type="password" id="password" name="password" required>
            <button type="submit">تسجيل</button>
        </form>
    </div>
{% endblock %}
{% extends 'base.html' %}

{% block title %}إضافة مقال جديد{% endblock %}

{% block content %}
    <div class="admin-form">
        <h1>إضافة مقال إخباري جديد</h1>
        <form method="POST">
            <label for="title">العنوان:</label>
            <input type="text" id="title" name="title" required>

            <label for="content">المحتوى:</label>
            <textarea id="content" name="content" rows="15" required></textarea>

            <label for="image_url">رابط الصورة (اختياري):</label>
            <input type="url" id="image_url" name="image_url">

            <label for="source">المصدر (اختياري):</label>
            <input type="text" id="source" name="source">

            <label for="category">الفئة (اختياري):</label>
            <input type="text" id="category" name="category">

            <button type="submit">إضافة المقال</button>
        </form>
    </div>
{% endblock %}
{% extends 'base.html' %}

{% block title %}نتائج البحث عن "{{ query }}"{% endblock %}

{% block content %}
    <h1>نتائج البحث عن "{{ query }}"</h1>
    <div class="news-grid">
        {% if results %}
            {% for article in results %}
                <div class="news-card">
                    {% if article.image_url %}
                        <img src="{{ article.image_url }}" alt="{{ article.title }}">
                    {% endif %}
                    <h2><a href="{{ url_for('news_detail', news_id=article.id) }}">{{ article.title }}</a></h2>
                    <p class="news-meta">
                        المصدر: {{ article.source | default('غير معروف') }} |
                        النشر: {{ article.published_at.strftime('%Y-%m-%d %H:%M') }}
                    </p>
                    <p>{{ article.content[:150] }}...</p>
                    <a href="{{ url_for('news_detail', news_id=article.id) }}" class="read-more">قراءة المزيد</a>
                </div>
            {% endfor %}
        {% else %}
            <p>لم يتم العثور على مقالات تطابق بحثك عن "{{ query }}".</p>
        {% endif %}
    </div>
{% endblock %}
/* إعادة تعيين أساسية */
* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
}

body {
    font-family: 'Arial', sans-serif;
    line-height: 1.6;
    background-color: #f4f4f4;
    color: #333;
    direction: rtl; /* اتجاه النص من اليمين لليسار */
    text-align: right; /* محاذاة النص لليمين */
}

/* لا تستخدم .container إذا كنت لا تستخدمه في HTML */

/* الرأس وشريط التنقل */
header {
    background: #333;
    color: #fff;
    padding: 1rem 0;
    border-bottom: #0779e4 3px solid;
}

nav {
    display: flex;
    justify-content: space-between;
    align-items: center;
    width: 90%;
    margin: auto;
    flex-wrap: wrap; /* للسماح للعناصر بالانتقال لأسفل في الشاشات الصغيرة */
}

.logo a {
    color: #fff;
    text-decoration: none;
    font-size: 1.8rem;
    font-weight: bold;
}

.nav-links {
    list-style: none;
    display: flex;
    padding-right: 0; /* لإزالة أي حشوة افتراضية للقوائم في RTL */
}

.nav-links li {
    margin-right: 20px; /* المسافة بين عناصر القائمة في RTL */
    margin-left: 0;
}

.nav-links a {
    color: #fff;
    text-decoration: none;
    font-weight: bold;
    transition: color 0.3s ease;
}

.nav-links a:hover {
    color: #0779e4;
}

.search-bar {
    display: flex;
}

.search-bar input[type="text"] {
    padding: 8px;
    border: none;
    border-radius: 0 5px 5px 0; /* تغيير ترتيب الزوايا لـ RTL */
    outline: none;
    text-align: right; /* محاذاة النص داخل حقل البحث لليمين */
}

.search-bar button {
    background: #0779e4;
    color: #fff;
    border: none;
    padding: 8px 12px;
    border-radius: 5px 0 0 5px; /* تغيير ترتيب الزوايا لـ RTL */
    cursor: pointer;
    transition: background 0.3s ease;
}

.search-bar button:hover {
    background: #055ba3;
}

/* المحتوى الرئيسي */
main {
    padding: 20px 0;
    width: 90%;
    margin: auto;
}

/* رسائل الفلاش (Flash Messages) */
.flash-messages {
    list-style: none;
    margin-bottom: 20px;
    padding: 10px;
    border-radius: 5px;
    padding-right: 20px; /* لتبدأ النقاط من اليمين في RTL */
}

.flash-messages li {
    padding: 8px;
    margin-bottom: 5px;
    border-radius: 3px;
    color: #fff;
    text-align: right;
}

.flash-messages li.success {
    background-color: #28a745; /* أخضر */
}

.flash-messages li.error {
    background-color: #dc3545; /* أحمر */
}

.flash-messages li.message { /* للرسائل العامة */
    background-color: #007bff; /* أزرق */
}

/* شبكة الأخبار (News Grid) */
.news-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
    gap: 20px;
}

.news-card {
    background: #fff;
    border: 1px solid #ddd;
    border-radius: 8px;
    overflow: hidden;
    box-shadow: 0 2px 5px rgba(0,0,0,0.1);
    transition: transform 0.2s ease;
    text-align: right;
}

.news-card:hover {
    transform: translateY(-5px);
}

.news-card img {
    width: 100%;
    height: 200px;
    object-fit: cover;
    display: block;
}

.news-card h2 {
    font-size: 1.5rem;
    margin: 15px;
}

.news-card h2 a {
    text-decoration: none;
    color: #333;
}

.news-card p {
    margin: 0 15px 15px;
}

.news-meta {
    font-size: 0.9em;
    color: #666;
    margin: 10px 15px;
}

.news-card .read-more {
    display: inline-block;
    background: #0779e4;
    color: #fff;
    padding: 8px 15px;
    border-radius: 5px;
    text-decoration: none;
    margin: 0 15px 15px;
    transition: background 0.3s ease;
}

.news-card .read-more:hover {
    background: #055ba3;
}

/* تفاصيل الخبر (News Detail) */
.news-detail {
    background: #fff;
    padding: 30px;
    border-radius: 8px;
    box-shadow: 0 2px 5px rgba(0,0,0,0.1);
    text-align: right;
}

.news-detail h1 {
    margin-bottom: 15px;
    font-size: 2.5rem;
    color: #222;
}

.news-detail .news-meta {
    margin-bottom: 20px;
}

.news-detail .detail-image {
    max-width: 100%;
    height: auto;
    margin-bottom: 20px;
    border-radius: 5px;
}

.news-content p {
    margin-bottom: 15px;
    font-size: 1.1rem;
    line-height: 1.8;
}

/* نماذج المصادقة (تسجيل الدخول/التسجيل) ونموذج المسؤول */
.auth-form, .admin-form {
    background: #fff;
    padding: 30px;
    border-radius: 8px;
    box-shadow: 0 2px 5px rgba(0,0,0,0.1);
    max-width: 500px;
    margin: 40px auto;
    text-align: right;
}

.auth-form h1, .admin-form h1 {
    text-align: center;
    margin-bottom: 25px;
    color: #333;
}

.auth-form label, .admin-form label {
    display: block;
    margin-bottom: 8px;
    font-weight: bold;
}

.auth-form input[type="text"],
.auth-form input[type="password"],
.auth-form input[type="url"],
.admin-form input[type="text"],
.admin-form input[type="url"],
.admin-form textarea {
    width: calc(100% - 20px); /* Adjusting for padding */
    padding: 10px;
    margin-bottom: 15px;
    border: 1px solid #ddd;
    border-radius: 5px;
    font-size: 1rem;
    text-align: right; /* محاذاة النص داخل الحقول لليمين */
}

.auth-form button, .admin-form button {
    display: block;
    width: 100%;
    padding: 12px;
    background: #0779e4;
    color: #fff;
    border: none;
    border-radius: 5px;
    font-size: 1.1rem;
    cursor: pointer;
    transition: background 0.3s ease;
}

.auth-form button:hover, .admin-form button:hover {
    background: #055ba3;
}

.auth-form p {
    text-align: center;
    margin-top: 20px;
}

.auth-form p a {
    color: #0779e4;
    text-decoration: none;
}

/* التذييل (Footer) */
footer {
    text-align: center;
    padding: 20px;
    background: #333;
    color: #fff;
    margin-top: 30px;
}
