const express = require('express');
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const cors = require('cors');
require('dotenv').config();

const app = express();
app.use(express.json());
app.use(cors());
app.set('view engine', 'ejs');
app.use(express.static('public')); // For CSS/JS files

// Connect to MongoDB
mongoose.connect(process.env.MONGO_URI, { useNewUrlParser: true, useUnifiedTopology: true });

// Schemas
const PoemSchema = new mongoose.Schema({
  title: String,
  content: String, // Marathi text
  image: String, // Optional image URL
  createdAt: { type: Date, default: Date.now }
});
const CommentSchema = new mongoose.Schema({
  poemId: mongoose.Schema.Types.ObjectId,
  name: String,
  comment: String,
  createdAt: { type: Date, default: Date.now }
});
const LikeSchema = new mongoose.Schema({
  poemId: mongoose.Schema.Types.ObjectId,
  ip: String // Track likes by IP to prevent spam
});

const Poem = mongoose.model('Poem', PoemSchema);
const Comment = mongoose.model('Comment', CommentSchema);
const Like = mongoose.model('Like', LikeSchema);

// Middleware for owner authentication
const authenticateOwner = (req, res, next) => {
  const token = req.header('Authorization');
  if (!token) return res.status(401).json({ error: 'Access denied' });
  try {
    const verified = jwt.verify(token, process.env.JWT_SECRET);
    req.owner = verified;
    next();
  } catch (err) {
    res.status(400).json({ error: 'Invalid token' });
  }
};

// Routes
// Public: View all poems
app.get('/', async (req, res) => {
  const poems = await Poem.find().sort({ createdAt: -1 });
  res.render('index', { poems });
});

// Public: View single poem with comments and likes
app.get('/poem/:id', async (req, res) => {
  const poem = await Poem.findById(req.params.id);
  const comments = await Comment.find({ poemId: req.params.id });
  const likes = await Like.countDocuments({ poemId: req.params.id });
  res.render('poem', { poem, comments, likes });
});

// Owner login
app.post('/login', async (req, res) => {
  const { password } = req.body;
  if (password !== process.env.OWNER_PASSWORD) return res.status(400).json({ error: 'Invalid password' });
  const token = jwt.sign({ id: 'owner' }, process.env.JWT_SECRET);
  res.json({ token });
});

// Owner: Post new poem
app.post('/poems', authenticateOwner, async (req, res) => {
  const { title, content, image } = req.body;
  const poem = new Poem({ title, content, image });
  await poem.save();
  res.json({ message: 'Poem posted' });
});

// Public: Add comment
app.post('/comment/:id', async (req, res) => {
  const { name, comment } = req.body;
  const newComment = new Comment({ poemId: req.params.id, name, comment });
  await newComment.save();
  res.redirect(`/poem/${req.params.id}`);
});

// Public: Like poem
app.post('/like/:id', async (req, res) => {
  const ip = req.ip;
  const existing = await Like.findOne({ poemId: req.params.id, ip });
  if (existing) return res.json({ message: 'Already liked' });
  const like = new Like({ poemId: req.params.id, ip });
  await like.save();
  res.json({ message: 'Liked' });
});

// Start server
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
Environment Variables (.env file):

Copy code
MONGO_URI=your_mongodb_connection_string
JWT_SECRET=your_secret_key_here
OWNER_PASSWORD=your_secure_password
