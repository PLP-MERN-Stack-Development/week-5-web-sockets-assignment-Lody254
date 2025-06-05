// ✅ BACKEND FILES

// server/server.js
const express = require('express');
const http = require('http');
const { Server } = require('socket.io');
const cors = require('cors');
require('dotenv').config();

const app = express();
const server = http.createServer(app);
const io = new Server(server, {
  cors: {
    origin: '*',
    methods: ['GET', 'POST']
  }
});

app.use(cors());
app.use(express.json());

const users = {};

io.on('connection', socket => {
  console.log(`User connected: ${socket.id}`);

  socket.on('join', ({ username }) => {
    users[socket.id] = username;
    io.emit('userList', Object.values(users));
    socket.broadcast.emit('message', { user: 'System', text: `${username} joined the chat.` });
  });

  socket.on('sendMessage', ({ message }) => {
    const user = users[socket.id];
    io.emit('message', { user, text: message });
  });

  socket.on('disconnect', () => {
    const username = users[socket.id];
    delete users[socket.id];
    io.emit('userList', Object.values(users));
    socket.broadcast.emit('message', { user: 'System', text: `${username} left the chat.` });
  });
});

const PORT = process.env.PORT || 5000;
server.listen(PORT, () => console.log(`Server running on port ${PORT}`));


// ✅ FRONTEND FILES

// client/src/App.jsx
import React, { useState, useEffect } from 'react';
import io from 'socket.io-client';

const socket = io('http://localhost:5000');

function App() {
  const [username, setUsername] = useState('');
  const [message, setMessage] = useState('');
  const [messages, setMessages] = useState([]);
  const [users, setUsers] = useState([]);
  const [joined, setJoined] = useState(false);

  useEffect(() => {
    socket.on('message', data => setMessages(prev => [...prev, data]));
    socket.on('userList', userList => setUsers(userList));

    return () => {
      socket.off('message');
      socket.off('userList');
    };
  }, []);

  const handleJoin = () => {
    if (username) {
      socket.emit('join', { username });
      setJoined(true);
    }
  };

  const handleSend = () => {
    if (message) {
      socket.emit('sendMessage', { message });
      setMessage('');
    }
  };

  return (
    <div className="p-4">
      {!joined ? (
        <div>
          <input type="text" value={username} onChange={e => setUsername(e.target.value)} placeholder="Enter username" className="border p-2" />
          <button onClick={handleJoin} className="ml-2 p-2 bg-blue-500 text-white">Join</button>
        </div>
      ) : (
        <div>
          <h2 className="font-bold">Users Online:</h2>
          <ul>{users.map((u, i) => <li key={i}>{u}</li>)}</ul>
          <div className="mt-4">
            {messages.map((m, i) => (
              <div key={i}><strong>{m.user}:</strong> {m.text}</div>
            ))}
          </div>
          <div className="mt-4">
            <input type="text" value={message} onChange={e => setMessage(e.target.value)} placeholder="Type a message" className="border p-2 w-64" />
            <button onClick={handleSend} className="ml-2 p-2 bg-green-500 text-white">Send</button>
          </div>
        </div>
      )}
    </div>
  );
}

export default App;


// client/src/index.js
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';
import './index.css';

const root = ReactDOM.createRoot(document.getElementById('root'));
root.render(<App />);


// ✅ .env (in server folder, not committed to GitHub)
PORT=5000


// ✅ To install dependencies:
// On server:
// npm install express socket.io cors dotenv

// On client:
// npm install socket.io-client react react-dom

// ✅ To run:
// In server folder: node server.js
// In client folder: npm start
