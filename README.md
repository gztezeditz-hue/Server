{
  "name": "darpanwears-server",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "cors": "^2.8.5",
    "express": "^4.18.2",
    "multer": "^1.4.5-lts.1",
    "nanoid": "^4.0.0"
  }
}// server.js
// Simple products API storing products in products.json and images in uploads/
const express = require('express');
const cors = require('cors');
const multer = require('multer');
const fs = require('fs');
const path = require('path');
const { nanoid } = require('nanoid');

const UPLOAD_DIR = path.join(__dirname, 'uploads');
const DB_FILE = path.join(__dirname, 'products.json');

if (!fs.existsSync(UPLOAD_DIR)) fs.mkdirSync(UPLOAD_DIR);
if (!fs.existsSync(DB_FILE)) fs.writeFileSync(DB_FILE, JSON.stringify([],{spaces:2}));

const storage = multer.diskStorage({
  destination: (req,file,cb) => cb(null, UPLOAD_DIR),
  filename: (req,file,cb) => cb(null, Date.now() + '_' + file.originalname.replace(/\s+/g,'_'))
});
const upload = multer({ storage });

const app = express();
app.use(cors());
app.use(express.json());
app.use('/uploads', express.static(UPLOAD_DIR)); // images public

function readDB(){ return JSON.parse(fs.readFileSync(DB_FILE,'utf8')||'[]'); }
function writeDB(data){ fs.writeFileSync(DB_FILE, JSON.stringify(data, null, 2)); }

// GET /products
app.get('/products', (req,res)=>{
  const list = readDB();
  res.json(list);
});

// POST /products  (multipart: image + fields)
app.post('/products', upload.single('image'), (req,res)=>{
  try{
    const { name, price, category, description, sizes, material } = req.body;
    if(!name || !price) return res.status(400).json({error:'name & price required'});
    const imgPath = req.file ? `/uploads/${path.basename(req.file.path)}` : '';
    const id = nanoid();
    const product = { id, name, price, category, description, sizes, material, image: imgPath, createdAt: Date.now() };
    const db = readDB();
    db.unshift(product);
    writeDB(db);
    res.json(product);
  }catch(e){ console.error(e); res.status(500).json({error:e.message}); }
});

// PUT /products/:id  (optional file)
app.put('/products/:id', upload.single('image'), (req,res)=>{
  try{
    const id = req.params.id;
    const db = readDB();
    const i = db.findIndex(x=>x.id===id);
    if(i===-1) return res.status(404).json({error:'not found'});
    const p = db[i];
    const { name, price, category, description, sizes, material } = req.body;
    if(name) p.name = name;
    if(price) p.price = price;
    if(category) p.category = category;
    if(description) p.description = description;
    if(sizes) p.sizes = sizes;
    if(material) p.material = material;
    if(req.file){
      // delete old file if exists
      if(p.image){
        const old = path.join(__dirname, p.image.replace(/^\/+/,'')); // strip leading slash
        if(fs.existsSync(old)) try{ fs.unlinkSync(old); }catch(e){}
      }
      p.image = `/uploads/${path.basename(req.file.path)}`;
    }
    p.updatedAt = Date.now();
    db[i] = p;
    writeDB(db);
    res.json(p);
  }catch(e){ console.error(e); res.status(500).json({error:e.message}); }
});

// DELETE /products/:id
app.delete('/products/:id', (req,res)=>{
  try{
    const id = req.params.id;
    const db = readDB();
    const i = db.findIndex(x=>x.id===id);
    if(i===-1) return res.status(404).json({error:'not found'});
    const [removed] = db.splice(i,1);
    // remove image file
    if(removed.image){
      const p = path.join(__dirname, removed.image.replace(/^\/+/,''));
      if(fs.existsSync(p)) try{ fs.unlinkSync(p); }catch(e){}
    }
    writeDB(db);
    res.json({ ok:true });
  }catch(e){ console.error(e); res.status(500).json({error:e.message}); }
});

// Simple health
app.get('/', (req,res)=>res.send('DarpanWears API'));

const PORT = process.env.PORT || 3000;
app.listen(PORT, ()=>console.log('Server running on', PORT));
