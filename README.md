# GeoGuessr-like (tp_html)

Ce projet est un clone simplifié de GeoGuessr, construit sans framework web majeur, avec :
- Backend Deno + Oak 17.1.4 + oakCors 1.2.2
- Base SQLite (≥ 5 tables : users, games, rounds, guesses, sessions)
- Auth JWT + cookies, hash des mots de passe
- RESTful CRUD
- WebSockets pour le mode temps réel
- Middleware (routing, auth, CORS, validation)
- HTTPS (certificat auto-signé)
- Front séparé (Vue.js via CDN)
- Déployé sur cloud Polytech
- Optionnel : refresh/access tokens, CSP, CI/CD (lint, fmt…), pixel war temps réel

## Structure

```bash
tp_html
├─ backend
│  ├─ certs/
│  │  ├─ cert.pem      # généré par openssl
│  │  └─ key.pem
│  ├─ controllers/     # logique métier
│  ├─ routes/          # définition des routes REST
│  ├─ middleware/      # auth, CORS, erreurs…
│  ├─ models/          # ORM léger ou helpers SQLite
│  ├─ utils/           # fonctions utilitaires
│  ├─ websocket.ts     # gestion WebSockets
│  ├─ db.ts            # initialisation SQLite + migrations
│  └─ mod.ts           # point d’entrée Oak HTTPS
├─ frontend
│  ├─ index.html
│  ├─ main.js          # Vue.js + appels API + WS
│  └─ styles.css
└─ README.md
```

```typescript name=backend/mod.ts
import { Application } from "https://deno.land/x/oak@v17.1.4/mod.ts";
import { oakCors } from "https://deno.land/x/oakCors@v1.2.2/mod.ts";
import router from "./routes/index.ts";
import { errorHandler } from "./middleware/error.ts";
import { authMiddleware } from "./middleware/auth.ts";
import { initDb } from "./db.ts";
import { serveTLS } from "https://deno.land/std@0.180.0/http/server.ts";

// 1. Init DB (migrations, tables…)
await initDb();

// 2. Création de l’app
const app = new Application();

// 3. Middlewares globaux
app.use(oakCors({ origin: "http://localhost:8080", credentials: true }));
app.use(errorHandler);
app.use(authMiddleware.unprotected); // routes publiques (login, register)

// 4. Routes REST / WS
app.use(router.routes());
app.use(router.allowedMethods());

// 5. Lancement HTTPS
console.log("🚀 Serveur démarré en HTTPS sur https://localhost:8443");
await serveTLS(app.handle, {
  port: 8443,
  certFile: "./certs/cert.pem",
  keyFile: "./certs/key.pem",
});
```

```typescript name=backend/db.ts
import { DB } from "https://deno.land/x/sqlite/mod.ts";

const db = new DB("tp_html.sqlite");

export async function initDb() {
  // users, games, rounds, guesses, sessions
  db.query(`
    CREATE TABLE IF NOT EXISTS users (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      username TEXT UNIQUE,
      password_hash TEXT,
      role TEXT DEFAULT 'player',
      created_at DATETIME DEFAULT CURRENT_TIMESTAMP
    );
  `);
  db.query(`
    CREATE TABLE IF NOT EXISTS games (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      title TEXT,
      owner_id INTEGER,
      created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
      FOREIGN KEY(owner_id) REFERENCES users(id)
    );
  `);
  db.query(`
    CREATE TABLE IF NOT EXISTS rounds (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      game_id INTEGER,
      latitude REAL,
      longitude REAL,
      created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
      FOREIGN KEY(game_id) REFERENCES games(id)
    );
  `);
  db.query(`
    CREATE TABLE IF NOT EXISTS guesses (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      round_id INTEGER,
      user_id INTEGER,
      guessed_lat REAL,
      guessed_lng REAL,
      distance REAL,
      created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
      FOREIGN KEY(round_id) REFERENCES rounds(id),
      FOREIGN KEY(user_id) REFERENCES users(id)
    );
  `);
  db.query(`
    CREATE TABLE IF NOT EXISTS sessions (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      user_id INTEGER,
      jwt TEXT,
      created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
      expires_at DATETIME,
      FOREIGN KEY(user_id) REFERENCES users(id)
    );
  `);
}
```

```typescript name=backend/routes/index.ts
import { Router } from "https://deno.land/x/oak@v17.1.4/mod.ts";
import authRoutes from "./auth.ts";
import userRoutes from "./users.ts";
import gameRoutes from "./games.ts";
import wsRoutes from "../websocket.ts";

const router = new Router();

// Auth public
router.use("/api/auth", authRoutes.routes(), authRoutes.allowedMethods());

// CRUD Users (admin)
router.use("/api/users", userRoutes.routes(), userRoutes.allowedMethods());

// CRUD Games/Rounds/Guesses (auth required)
router.use("/api/games", gameRoutes.routes(), gameRoutes.allowedMethods());

// WS pour partie temps réel
router.get("/ws", wsRoutes);

export default router;
```

```typescript name=frontend/index.html
<!DOCTYPE html>
<html lang="fr">
<head>
  <meta charset="UTF-8" />
  <meta http-equiv="Content-Security-Policy" content="default-src 'self'; script-src 'self' https://unpkg.com; style-src 'self';">
  <title>GeoGuessr-like</title>
  <link rel="stylesheet" href="styles.css" />
</head>
<body>
  <div id="app">
    <h1>GeoGuessr-like</h1>
    <!-- login / register / lobby / game -->
  </div>

  <!-- Vue.js via CDN -->
  <script src="https://unpkg.com/vue@3.4.0/dist/vue.global.prod.js"></script>
  <script src="main.js"></script>
</body>
</html>
```

```javascript name=frontend/main.js
const { createApp } = Vue;
createApp({
  data() {
    return {
      user: null,
      token: null,
      games: [],
      socket: null,
    };
  },
  methods: {
    async login(username, password) {
      const res = await fetch("https://localhost:8443/api/auth/login", {
        method: "POST",
        credentials: "include",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ username, password }),
      });
      const data = await res.json();
      this.token = data.token;
      this.initSocket();
    },
    initSocket() {
      this.socket = new WebSocket("wss://localhost:8443/ws");
      this.socket.onmessage = e => console.log("WS:", JSON.parse(e.data));
    },
  },
}).mount("#app");
```

Ensuite :  
1. Écrire les controllers (auth, users, games…)  
2. Implémenter password hashing (bcrypt), JWT, cookies sécurisés.  
3. Ajouter middleware de protection sur /api/games…  
4. Coder websocket.ts (salles de jeu, broadcast).  
5. Finaliser front : lobby, carte interactive (Leaflet ou Google Maps), système de rounds.  
6. CI/CD (deno fmt, lint, check) et déploiement sur Polytech Cloud.  

Dis-moi si tu veux qu’on attaque un fichier en particulier !
