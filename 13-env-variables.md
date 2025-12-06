# MongoDB Config Centralization & Environment Variable Setup (Docker + Node + Mongoose)

## 1. Problem: Hard‑coded Mongo URL in `index.js`

Your Express app originally had something like:

```js
mongoose.connect(
  'mongodb://sanjeev:myPassword@mongo:27017/mydb?authSource=admin'
);
```

Issues:
- Credentials hard‑coded.
- Host + port hard‑coded.
- Changing DB host or moving to cloud requires code changes.
- Violates 12‑factor principles.

Goal:
- Move all DB connection values into environment variables.
- Centralize access to them in a config module.
- Make Docker supply those values.

---

## 2. Create `config/config.js` — central home for env variables

```js
// config/config.js

const MONGO_IP = process.env.MONGO_IP || 'mongo';
const MONGO_PORT = process.env.MONGO_PORT || '27017';

const MONGO_USER = process.env.MONGO_USER;
const MONGO_PASSWORD = process.env.MONGO_PASSWORD;

module.exports = {
  MONGO_IP,
  MONGO_PORT,
  MONGO_USER,
  MONGO_PASSWORD,
};
```

Ideas:
- `mongo` resolves automatically via Docker DNS when both containers are on the same user‑defined network.
- Default values make local dev stable.
- No defaults for user/pass → fail fast if missing.

---

## 3. Build URL in `index.js`

```js
const mongoose = require('mongoose');
const {
  MONGO_IP,
  MONGO_PORT,
  MONGO_USER,
  MONGO_PASSWORD,
} = require('./config/config');

const MONGO_URL = `mongodb://${MONGO_USER}:${MONGO_PASSWORD}@${MONGO_IP}:${MONGO_PORT}/mydb?authSource=admin`;

mongoose
  .connect(MONGO_URL, {
    useNewUrlParser: true,
    useUnifiedTopology: true,
  })
  .then(() => console.log('✅ Successfully connected to database'))
  .catch((err) => console.error('❌ Failed to connect to database', err));
```

---

## 4. Missing env vars → authentication error

After updating the code, logs showed:

> Authentication failed

Reason:
- Node container didn’t receive `MONGO_USER` or `MONGO_PASSWORD`.

Solution:
Add env vars into the **dev compose file**, not the base one.

---

## 5. Add env vars in `docker-compose.dev.yaml`

```yaml
services:
  node_app:
    environment:
      - NODE_ENV=development
      - MONGO_IP=mongo
      - MONGO_PORT=27017
      - MONGO_USER=sanjeev
      - MONGO_PASSWORD=myPassword

    volumes:
      - ./:/app:ro
      - /app/node_modules
```

Mongo service itself:

```yaml
  mongo:
    image: mongo
    environment:
      - MONGO_INITDB_ROOT_USERNAME=sanjeev
      - MONGO_INITDB_ROOT_PASSWORD=myPassword
    volumes:
      - mongo-db:/data/db
```

---

## 6. Rebuild + verify

```bash
docker compose -f docker-compose.yaml -f docker-compose.dev.yaml down
docker compose -f docker-compose.yaml -f docker-compose.dev.yaml up -d --build
docker logs -f node_app
```

Expected:

```
✅ Successfully connected to database
```

---

## 7. Clean up Mongoose warnings

Add these options:

```js
mongoose.connect(MONGO_URL, {
  useNewUrlParser: true,
  useUnifiedTopology: true,
});
```

---

## 8. Recap

- Extracted Mongo connection fields into `config/config.js`.
- Docker dev compose now injects all DB env vars.
- Node app builds URL from env → no more hard-coded values.
- Rebuild required after adding new env vars.
- Mongoose warnings fixed using connection options.

Your application is now production‑ready from a configuration standpoint.
