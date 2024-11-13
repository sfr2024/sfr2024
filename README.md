- 👋 Hi, I’m @sfr2024
- 👀 I’m interested in ...
- 🌱 I’m currently learning ...
- 💞️ I’m looking to collaborate on ...
- 📫 How to reach me ...
- 😄 Pronouns: ...
- ⚡ Fun fact: ...

<!---
sfr2024/sfr2024 is a ✨ special ✨ repository because its `README.md` (this file) appears on your GitHub profile.
You can click the Preview link to take a look at your changes.
--->
# נתבסס על Flask עבור Python, עם MongoDB כבסיס נתונים.

# התקנות:
# pip install flask flask-jwt-extended pymongo flask-cors

from flask import Flask, request, jsonify
from flask_jwt_extended import JWTManager, create_access_token, jwt_required, get_jwt_identity
from pymongo import MongoClient
from flask_cors import CORS
import datetime

app = Flask(__name__)
CORS(app)
app.config['JWT_SECRET_KEY'] = 'your-secret-key'
jwt = JWTManager(app)

# חיבור ל-MongoDB
client = MongoClient('mongodb://localhost:27017/')
db = client['channels_db']
users_collection = db['users']
channels_collection = db['channels']

# רישום משתמש חדש
@app.route('/register', methods=['POST'])
def register():
    data = request.get_json()
    username = data.get('username')
    password = data.get('password')
    if users_collection.find_one({'username': username}):
        return jsonify({'msg': 'Username already exists'}), 400
    users_collection.insert_one({'username': username, 'password': password})
    return jsonify({'msg': 'User registered successfully'}), 200

# כניסת משתמש קיים
@app.route('/login', methods=['POST'])
def login():
    data = request.get_json()
    username = data.get('username')
    password = data.get('password')
    user = users_collection.find_one({'username': username, 'password': password})
    if not user:
        return jsonify({'msg': 'Invalid credentials'}), 401
    access_token = create_access_token(identity=str(user['_id']))
    return jsonify({'access_token': access_token}), 200

# יצירת ערוץ חדש
@app.route('/create_channel', methods=['POST'])
@jwt_required()
def create_channel():
    current_user = get_jwt_identity()
    data = request.get_json()
    channel_name = data.get('name')
    description = data.get('description')
    
    # יצירת ערוץ
    new_channel = {
        'name': channel_name,
        'description': description,
        'creator': current_user,
        'followers': [],
        'likes': 0,
        'created_at': datetime.datetime.now()
    }
    
    channels_collection.insert_one(new_channel)
    return jsonify({'msg': 'Channel created successfully'}), 200

# קבלת רשימת ערוצים של משתמש
@app.route('/my_channels', methods=['GET'])
@jwt_required()
def my_channels():
    current_user = get_jwt_identity()
    channels = list(channels_collection.find({'creator': current_user}))
    for channel in channels:
        channel['_id'] = str(channel['_id'])
    return jsonify({'channels': channels}), 200

# מעקב אחרי ערוץ
@app.route('/follow_channel', methods=['POST'])
@jwt_required()
def follow_channel():
    current_user = get_jwt_identity()
    data = request.get_json()
    channel_id = data.get('channel_id')
    channel = channels_collection.find_one({'_id': channel_id})
    
    if not channel:
        return jsonify({'msg': 'Channel not found'}), 404

    if current_user in channel['followers']:
        return jsonify({'msg': 'Already following this channel'}), 400

    channels_collection.update_one(
        {'_id': channel_id},
        {'$push': {'followers': current_user}}
    )
    
    return jsonify({'msg': 'Followed channel successfully'}), 200

# הוספת לייק לערוץ
@app.route('/like_channel', methods=['POST'])
@jwt_required()
def like_channel():
    data = request.get_json()
    channel_id = data.get('channel_id')
    channel = channels_collection.find_one({'_id': channel_id})
    
    if not channel:
        return jsonify({'msg': 'Channel not found'}), 404

    channels_collection.update_one(
        {'_id': channel_id},
        {'$inc': {'likes': 1}}
    )
    
    return jsonify({'msg': 'Channel liked successfully'}), 200

if __name__ == '__main__':
    app.run(debug=True)
