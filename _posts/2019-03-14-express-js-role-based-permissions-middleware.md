---
layout: post
title: 'Express.js role-based permissions middleware'
date: 2019-03-14 13:40:00 +0700
categories: [nodejs]
tags: [nodejs, express, authentication]
image: place_holder.jpeg
---

## App.js

---

```javascript
// the main app file
import express from 'express';
import loadDb from './loadDb'; // dummy middleware to load db (sets request.db)
import authenticate from './authentication'; // middleware for doing authentication
import permit from './permission'; // middleware for checking if user's role is permitted to make request

const app = express(),
  api = express.Router();

// first middleware will setup db connection
app.use(loadDb);

// authenticate each request
// will set `request.user`
app.use(authenticate);

// setup permission middleware,
// check `request.user.role` and decide if ok to continue
app.use('/api/private', permit('admin'));
app.use(['/api/foo', '/api/bar'], permit('owner', 'employee'));

// setup requests handlers
api.get('/private/whatever', (req, res) => res.json({ whatever: true }));
api.get('/foo', (req, res) => res.json({ currentUser: req.user }));
api.get('/bar', (req, res) => res.json({ currentUser: req.user }));

// setup permissions based on HTTP Method

// account creation is public
api.post('/account', (req, res) => res.json({ message: 'created' }));

// account update & delete (PATCH & DELETE) are only available to account owner
api.patch('/account', permit('owner'), (req, res) =>
  res.json({ message: 'updated' }),
);
api.delete('/account', permit('owner'), (req, res) =>
  res.json({ message: 'deleted' }),
);

// viewing account "GET" available to account owner and account member
api.get('/account', permit('owner', 'employee'), (req, res) =>
  res.json({ currentUser: req.user }),
);

// mount api router
app.use('/api', api);

// start 'er up
app.listen(process.env.PORT || 3000);
```

## Authentication.js

---

```javascript
// middleware for authentication
export default async function authorize(request, _response, next) {
  const apiToken = request.headers['x-api-token'];

  // set user on-success
  request.user = await request.db.users.findByApiKey(apiToken);

  // always continue to next middleware
  next();
}
```

## loadDB.js

---

```javascript
// dummy middleware for db (set's request.db)
export default function loadDb(request, _response, next) {

  // dummy db
  request.db = {
    users: {
      findByApiKey: async token => {
        switch {
          case (token == '1234') {
            return {role: 'owner', id: 1234};
          case (token == '5678') {
            return {role: 'employee', id: 5678};
          default:
            return null; // no user
        }
      }
    }
  };

  next();
}
```

## permission.js

---

```javascript
// middleware for doing role-based permissions
export default function permit(...allowed) {
  const isAllowed = role => allowed.indexOf(role) > -1;

  // return a middleware
  return (request, response, next) => {
    if (request.user && isAllowed(request.user.role)) next();
    // role is allowed, so continue on the next middleware
    else {
      response.status(403).json({ message: 'Forbidden' }); // user is forbidden
    }
  };
}
```
