# -AI-Powered-Notes-App-like-Notion-ChatGPT-(NotedAI — Your Smart, Minimalist Note Companion}

AI-Powered Notes App — a Notion-style editor that integrates ChatGPT-like features using the MERN stack!

# Features:
-Markdown editor with autosave
-Authenticated user notes
-AI content generation using OpenAI API
-Note sharing via public links
-Full-text search

## Project Structure (Folder Layout)

noted-ai/
├── client/                     # React frontend
│   ├── public/
│   ├── src/
│   │   ├── components/         # Editor, NoteCard, Sidebar, etc.
│   │   ├── pages/              # NotesPage, Login, Register
│   │   ├── context/            # AuthContext, NotesContext
│   │   ├── hooks/              # Custom hooks (e.g. useAuth, useNote)
│   │   ├── services/           # Axios calls to backend
│   │   ├── App.jsx
│   │   └── index.js
│   └── package.json
├── server/                     # Node + Express backend
│   ├── config/                 # DB config, middleware
│   ├── controllers/            # Auth + Notes + AI logic
│   ├── models/                 # User, Note schemas
│   ├── routes/                 # authRoutes, noteRoutes, aiRoutes
│   ├── utils/                  # JWT, OpenAI wrapper
│   ├── server.js
│   └── package.json
├── .env
└── README.md

## Core Features

# User Authentication

JWT-based login/register

Password hashing (bcrypt)

Protected routes

# Notes Editor

Rich text or Markdown support

Auto-save with debounce

Full CRUD (Create, Read, Update, Delete)

# AI Integration

ChatGPT API: Suggest titles, summarize notes, expand ideas

Commands like:

/summarize

/expand

/bullet

# Notes Management

Tagging, sorting, and search

Pin/favorite notes

Auto-generated titles

# Cloud Storage
MongoDB (title, content, tags, timestamps, user ID)

Optional: image upload with Cloudinary

# Tech Stack

Frontend: React + TailwindCSS + React Router + Axios

Backend: Express + MongoDB + Mongoose

Auth: JWT + bcrypt

AI: OpenAI API (chat completions)

Deployment: Vercel (frontend) + Render (backend) + MongoDB Atlas

## plan

 # Core Setup
 
 Setup server + MongoDB + client with React Router

 User auth (register/login/logout with JWT)

 Create note model and RESTful API

# Notes UI + AI Feature

 Build rich text/markdown editor

 Add note CRUD logic

 Integrate OpenAI /summarize + /expand

# UX Polish + Deployment

 Implement tagging + search

 Toasts, loading states, error handling

 Deploy + polish UI for mobile/tablet

 ## code

// === server/models/userModel.js ===
import mongoose from 'mongoose';
import bcrypt from 'bcryptjs';

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
}, { timestamps: true });

userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

userSchema.methods.matchPassword = async function(enteredPassword) {
  return await bcrypt.compare(enteredPassword, this.password);
};

const User = mongoose.model('User', userSchema);
export { User };


// === server/models/noteModel.js ===
import mongoose from 'mongoose';

const noteSchema = new mongoose.Schema({
  user: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  title: { type: String, default: 'Untitled' },
  content: { type: String, required: true },
  tags: [String],
}, { timestamps: true });

// Avoid model overwrite error in development
const Note = mongoose.models.Note || mongoose.model('Note', noteSchema);
export { Note };


// === server/controllers/noteController.js ===
// Removed invalid dynamic path import for ESM
import { Note } from '../models/noteModel.js';
import { Configuration, OpenAIApi } from 'openai';

const configuration = new Configuration({ apiKey: process.env.OPENAI_API_KEY });
const openai = new OpenAIApi(configuration);

export const getNotes = async (req, res) => {
  const notes = await Note.find({ user: req.user.id });
  res.json(notes);
};

export const createNote = async (req, res) => {
  const note = new Note({ ...req.body, user: req.user.id });
  const saved = await note.save();
  res.status(201).json(saved);
};

export const summarizeNote = async (req, res) => {
  try {
    const { content } = req.body;
    const response = await openai.createChatCompletion({
      model: 'gpt-4',
      messages: [
        { role: 'system', content: 'You summarize notes into bullet points.' },
        { role: 'user', content }
      ]
    });
    res.json({ summary: response.data.choices[0].message.content });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
};

export const updateNote = async (req, res) => {
  const note = await Note.findById(req.params.id);
  if (!note) return res.status(404).json({ message: 'Note not found' });
  if (note.user.toString() !== req.user.id) return res.status(401).json({ message: 'Not authorized' });
  note.title = req.body.title;
  note.content = req.body.content;
  note.tags = req.body.tags;
  const updated = await note.save();
  res.json(updated);
};

export const deleteNote = async (req, res) => {
  const note = await Note.findById(req.params.id);
  if (!note) return res.status(404).json({ message: 'Note not found' });
  if (note.user.toString() !== req.user.id) return res.status(401).json({ message: 'Not authorized' });
  await note.deleteOne();
  res.json({ message: 'Note deleted' });
};


// === client/src/pages/NotesPage.jsx ===
import { useEffect, useState } from 'react';
import axios from 'axios';
import dynamic from 'next/dynamic';
const ReactQuill = dynamic(() => import('react-quill'), { ssr: false });
import 'react-quill/dist/quill.snow.css';

export default function NotesPage() {
  const [notes, setNotes] = useState([]);
  const [newNote, setNewNote] = useState('');
  const [summary, setSummary] = useState('');

  useEffect(() => {
    axios.get('/api/notes').then(res => setNotes(res.data));
  }, []);

  const handleCreate = async () => {
    const res = await axios.post('/api/notes', { content: newNote });
    setNotes([...notes, res.data]);
    setNewNote('');
  };

  const handleSummarize = async () => {
    const res = await axios.post('/api/notes/summarize', { content: newNote });
    setSummary(res.data.summary);
  };

  return (
    <div className="p-4">
      <h2 className="text-xl font-bold mb-4">My Notes</h2>
      <ReactQuill theme="snow" value={newNote} onChange={setNewNote} className="mb-2" />
      <div className="flex space-x-2">
        <button onClick={handleCreate} className="bg-blue-600 text-white px-4 py-2 rounded">Save Note</button>
        <button onClick={handleSummarize} className="bg-green-600 text-white px-4 py-2 rounded">Summarize</button>
      </div>
      {summary && (
        <div className="mt-4 p-3 bg-gray-100 border rounded">
          <h4 className="font-semibold">AI Summary</h4>
          <p className="text-sm text-gray-800 whitespace-pre-line">{summary}</p>
        </div>
      )}
      <div className="mt-4 space-y-3">
        {notes.map((note) => (
          <div key={note._id} className="p-3 border rounded bg-white shadow">
            <h3 className="font-semibold">{note.title || 'Untitled'}</h3>
            <p className="text-sm text-gray-700 whitespace-pre-line">{note.content}</p>
          </div>
        ))}
      </div>
    </div>
  );
}

