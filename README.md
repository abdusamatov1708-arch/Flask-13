# Flask-13
from flask import Flask, render_template, request, redirect, url_for, jsonify
from flask_sqlalchemy import SQLAlchemy
from flask_cors import CORS

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///api.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
db = SQLAlchemy(app)

# Boshqa frontend'lar ulanishi uchun CORS sozlamasi
CORS(app, resources={r"/api/*": {"origins": "*"}})


# Model tuzilishi
class Note(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(120), nullable=False)
    body = db.Column(db.Text, default='')

    def to_dict(self):
        return {'id': self.id, 'title': self.title, 'body': self.body}


# Ma'lumotlar bazasini yaratish
with app.app_context():
    db.create_all()


# 1. WEB UI - CRUD (Avvalgi darsdagi HTML interfeysi ishlashda davom etadi)


@app.route('/')
def index():
    notes = Note.query.all()
    return render_template('index.html', notes=notes)


@app.route('/create', methods=['POST'])
def web_create():
    title = request.form.get('title')
    body = request.form.get('body', '')
    if title:
        n = Note(title=title, body=body)
        db.session.add(n)
        db.session.commit()
    return redirect(url_for('index'))


@app.route('/delete/<int:id>', methods=['POST'])
def web_delete(id):
    n = Note.query.get_or_404(id)
    db.session.delete(n)
    db.session.commit()
    return redirect(url_for('index'))


# 2. JSON API ENDPOINT'LARI (Vazifa talablari)

# GET /api/notes — barcha notlarni JSON formatda olish (200 OK)
@app.route('/api/notes', methods=['GET'])
def list_notes():
    notes = Note.query.all()
    return jsonify([n.to_dict() for n in notes]), 200


# GET /api/notes/<id> — bitta notni olish (200 OK yoki 404 Not Found)
@app.route('/api/notes/<int:id>', methods=['GET'])
def get_note(id):
    n = Note.query.get_or_404(id)  # Topilmasa avtomatik 404 qaytaradi
    return jsonify(n.to_dict()), 200


# POST /api/notes — yangi not yaratish (201 Created)
@app.route('/api/notes', methods=['POST'])
def create_note():
    data = request.get_json()
    if not data or not data.get('title'):
        return jsonify({'error': 'title shart'}), 400

    n = Note(title=data['title'], body=data.get('body', ''))
    db.session.add(n)
    db.session.commit()
    return jsonify(n.to_dict()), 201


# DELETE /api/notes/<id> — notni o'chirish (204 No Content yoki 404 Not Found)
@app.route('/api/notes/<int:id>', methods=['DELETE'])
def delete_note(id):
    n = Note.query.get_or_404(id)
    db.session.delete(n)
    db.session.commit()
    return '', 204


# Global xatolik handler (404 xatosi chiroyli JSON ko'rinishida chiqishi uchun)
@app.errorhandler(404)
def not_found(e):
    return jsonify({'error': 'Topilmadi'}), 404


if __name__ == '__main__':
    app.run(debug=True)
