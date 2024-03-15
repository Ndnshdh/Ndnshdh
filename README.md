from flask import Flask, render_template, request, redirect, url_for
from flask_socketio import SocketIO, emit, join_room, leave_room
import os

app = Flask(__name__)
app.config['SECRET_KEY'] = os.urandom(24)
socketio = SocketIO(app)

# Oda ve kullanıcı listeleri
rooms = {}
users = {}

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/chat')
def chat():
    username = request.args.get('username')
    if username:
        return render_template('chat.html', username=username)
    else:
        return redirect(url_for('index'))

@socketio.on('connect')
def handle_connect():
    print('Bir kullanıcı bağlandı.')

@socketio.on('disconnect')
def handle_disconnect():
    print('Bir kullanıcı ayrıldı.')

@socketio.on('join')
def handle_join(data):
    username = data['username']
    room = data['room']
    join_room(room)
    if room not in rooms:
        rooms[room] = []
    rooms[room].append(username)
    users[username] = room
    emit('user_update', {'users': rooms[room]}, room=room)
    emit('message', {'username': 'System', 'message': f'{username} odaya katıldı.'}, room=room)

@socketio.on('leave')
def handle_leave(data):
    username = data['username']
    room = users[username]
    leave_room(room)
    rooms[room].remove(username)
    del users[username]
    emit('user_update', {'users': rooms[room]}, room=room)
    emit('message', {'username': 'System', 'message': f'{username} odadan ayrıldı.'}, room=room)

@socketio.on('message')
def handle_message(data):
    username = data['username']
    message = data['message']
    room = users[username]
    emit('message', {'username': username, 'message': message}, room=room)

if __name__ == '__main__':
    socketio.run(app, debug=True)
