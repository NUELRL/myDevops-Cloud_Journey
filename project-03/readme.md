# MERN Stack To-Do Application on AWS

![MERN Stack](https://img.shields.io/badge/Stack-MERN-blue?style=for-the-badge)
![MongoDB](https://img.shields.io/badge/MongoDB-Atlas-green?style=for-the-badge&logo=mongodb)
![Express](https://img.shields.io/badge/Express.js-Backend-black?style=for-the-badge&logo=express)
![React](https://img.shields.io/badge/React-Frontend-61DAFB?style=for-the-badge&logo=react)
![Node.js](https://img.shields.io/badge/Node.js-Runtime-339933?style=for-the-badge&logo=node.js)
![AWS](https://img.shields.io/badge/AWS-EC2-FF9900?style=for-the-badge&logo=amazonaws)

A full-stack To-Do web application built and deployed on an AWS EC2 instance using the MERN stack — MongoDB, ExpressJS, ReactJS, and Node.js. This documents the exact steps I followed to get it working, including the mistakes I ran into and how I fixed them.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Step 1 – Backend Configuration](#step-1--backend-configuration)
- [Step 2 – Setting Up Routes and Models](#step-2--setting-up-routes-and-models)
- [Step 3 – MongoDB Atlas Database Setup](#step-3--mongodb-atlas-database-setup)
- [Step 4 – Connecting the Database to the App](#step-4--connecting-the-database-to-the-app)
- [Step 5 – Testing the API with Postman](#step-5--testing-the-api-with-postman)
- [Step 6 – Frontend Creation with React](#step-6--frontend-creation-with-react)
- [Step 7 – Running the Full Application](#step-7--running-the-full-application)
- [Mistakes I Made & How I Fixed Them](#mistakes-i-made--how-i-fixed-them)

---

## Architecture Overview

The application follows a classic three-tier architecture:

```
React (Browser) ←→ ExpressJS + Node.js (Server) ←→ MongoDB Atlas (Database)
```

The React frontend runs in the browser and communicates with an Express/Node.js backend via REST API calls. The backend reads and writes data to a MongoDB Atlas cluster.

---

## Step 1 – Backend Configuration

### Update and Upgrade Ubuntu

After SSH-ing into my EC2 instance, I started by updating the package lists and upgrading existing packages:

```bash
sudo apt update
sudo apt upgrade
```
![alt text](./images/task%203%20sudo%20apt%20update.jpg)
![alt text](./images/task%203%20sudo%20apt%20upgrade.jpg)


### Install Node.js

I fetched the Node.js 20.x setup script from the NodeSource repository and installed it:

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs
```
![alt text](./images/task%203%20curl%20-fsSL.jpg)
![alt text](./images/task%203%20sudo%20apt-get%20install%20-y%20nodejs.jpg)



This single command installs both `node` and `npm`. I verified both were installed correctly:

```bash
node -v
npm -v
```

I created a project directory called `Todo` and initialized a Node.js project inside it:

```bash
mkdir Todo
cd Todo
npm init
```

![alt text](./images/task%203%20cd%20todo.jpg)


I followed the prompts, pressing Enter to accept defaults, and typed `yes` when asked to write out `package.json`. I confirmed the file was created with:

```bash
ls
```

### Install Express and dotenv

```bash
npm install express
npm install dotenv
touch index.js
```
![alt text](./images/task%203%20npm%20i%20dolven.jpg)
![alt text](./images/task%203%20express%20i.jpg)



### Create the Initial index.js

I opened `index.js` with `vim index.js` and pasted the following code:

```js
const express = require('express');
require('dotenv').config();

const app = express();
const port = process.env.PORT || 5000;

app.use((req, res, next) => {
  res.header("Access-Control-Allow-Origin", "*");
  res.header("Access-Control-Allow-Headers", "Origin, X-Requested-With, Content-Type, Accept");
  next();
});

app.use((req, res, next) => {
  res.send('Welcome to Express');
});

app.listen(port, () => {
  console.log(`Server running on port ${port}`)
});
```

I saved with `:w` and exited with `:qa`.

![alt text](./images/task%203%20vim%20indexjs.jpg)

I started the server to verify it was working:

```bash
node index.js
```

Then i opened port `5000`

![alt text](./images/task%203%20roles.jpg)


I then visited `http://<EC2-Public-IP>:5000` in my browser and saw **"Welcome to Express"**.

![alt text](./images/task%203%20welcome%20to%20e.jpg)

---

## Step 2 – Setting Up Routes and Models

### Create the Routes Directory

Still in the `Todo` directory, I created a `routes` folder and an `api.js` file inside it:

```bash
mkdir routes
cd routes
touch api.js
vim api.js
```
![alt text](./images/task%203%20routes.jpg)


I wrote the following initial route structure:

```js
const express = require ('express');
const router = express.Router();

router.get('/todos', (req, res, next) => {});
router.post('/todos', (req, res, next) => {});
router.delete('/todos/:id', (req, res, next) => {});

module.exports = router;
```
![alt text](./images/task%203%20vim%20config.jpg)



### Create the Models Directory and Todo Schema

I navigated back to `Todo` and installed Mongoose, then created the model:

```bash
cd ..
npm install mongoose
mkdir models && cd models && touch todo.js
vim todo.js
```
![alt text](./images/task%203%20mongoose%20i.jpg)

![alt text](./images/task%203%20todojs.jpg)


I wrote the following schema definition:

```js
const mongoose = require('mongoose');
const Schema = mongoose.Schema;

const TodoSchema = new Schema({
  action: {
    type: String,
    required: [true, 'The todo text field is required']
  }
});

const Todo = mongoose.model('todo', TodoSchema);
module.exports = Todo;
```

![alt text](./images/task%203%20vim%20todojs.jpg)

### Update api.js with Full Route Logic

I went back into `routes/api.js` (using `:%d` in vim to clear it first) and replaced the content with the full implementation that connects to the Todo model:

```js
const express = require ('express');
const router = express.Router();
const Todo = require('../models/todo');

router.get('/todos', (req, res, next) => {
  Todo.find({}, 'action')
    .then(data => res.json(data))
    .catch(next)
});

router.post('/todos', (req, res, next) => {
  if(req.body.action){
    Todo.create(req.body)
      .then(data => res.json(data))
      .catch(next)
  } else {
    res.json({ error: "The input field is empty" })
  }
});

router.delete('/todos/:id', (req, res, next) => {
  Todo.findOneAndDelete({"_id": req.params.id})
    .then(data => res.json(data))
    .catch(next)
});

module.exports = router;
```
![alt text](./images/task%203%20updRoutes.jpg)


---

## Step 3 – MongoDB Atlas Database Setup

I signed up for a free MongoDB Atlas account and created a new cluster.

**Steps I followed in Atlas:**

1. Created a new project and a free-tier **Sandbox** cluster on AWS (selected a region close to me)
2. Under **Database Access**, created a new database user with a username and password
3. Under **Network Access**, clicked **Allow Access from Anywhere** (`0.0.0.0/0`) — I made sure to change the deletion time from **6 Hours to 1 Week**
4. Clicked **Connect** on my cluster → **Connect your application** → selected **Node.js** driver → copied the connection string

![alt text](./images/task%203%20mongo%201.jpg)
![alt text](./images/task%203%20mongo%202%20crtclus.jpg)
![alt text](./images/task%203%20mongo%203.jpg)
![alt text](./images/task%203%20mongo%20net%20acesss.jpg)



---

## Step 4 – Connecting the Database to the App

### Create the .env File

Back in the `Todo` directory, I created a `.env` file to store my MongoDB connection string securely:

```bash
touch .env
vi .env
```

I added the connection string, replacing the placeholders with my actual Atlas credentials:

```
DB = 'mongodb+srv://<username>:<password>@<network-address>/<dbname>?retryWrites=true&w=majority'
```

![alt text](./images/task%203%20mongo%206.jpg)


### Update index.js to Connect to MongoDB

I opened `index.js` again, cleared it with `:%d`, and replaced the content with the full version that connects to the database:

```js
const express = require('express');
const bodyParser = require('body-parser');
const mongoose = require('mongoose');
const routes = require('./routes/api');
const path = require('path');
require('dotenv').config();

const app = express();
const port = process.env.PORT || 5000;

mongoose.connect(process.env.DB, { useNewUrlParser: true, useUnifiedTopology: true })
  .then(() => console.log(`Database connected successfully`))
  .catch(err => console.log(err));

mongoose.Promise = global.Promise;

app.use((req, res, next) => {
  res.header("Access-Control-Allow-Origin", "*");
  res.header("Access-Control-Allow-Headers", "Origin, X-Requested-With, Content-Type, Accept");
  next();
});

app.use(bodyParser.json());
app.use('/api', routes);

app.use((err, req, res, next) => {
  console.log(err);
  next();
});

app.listen(port, () => {
  console.log(`Server running on port ${port}`)
});
```

![alt text](./images/task%203%20mongo%207.jpg)


I started the server again:

```bash
node index.js
```
![alt text](./images/task%203%20mongo%208.jpg)


---

## Step 5 – Testing the API with Postman

With the backend running, I tested all three API endpoints using Postman before building the frontend.

### POST Request – Add a Task

- Method: `POST`
- URL: `http://<EC2-Public-IP>:5000/api/todos`
- Header: `Content-Type: application/json`
- Body (raw JSON):

```json
{
  "action": "################"
}
```
![alt text](./images/task%203%20postman%20test.jpg)

### GET Request – Retrieve All Tasks

- Method: `GET`
- URL: `http://<EC2-Public-IP>:5000/api/todos`

![alt text](./images/task%203%20postman%20get%20req.jpg)


### DELETE Request – Remove a Task

- Method: `DELETE`
- URL: `http://<EC2-Public-IP>:5000/api/todos/id
---

## Step 6 – Frontend Creation with React

### Scaffold the React App

From the `Todo` directory:

```bash
npx create-react-app client
```
![alt text](./images/task%203%20react-i.jpg)


### Install Dev Dependencies

Still in the `Todo` directory:

```bash
npm install concurrently --save-dev
npm install nodemon --save-dev
```
![alt text](./images/task%203%20front%201.jpg)


### Update package.json Scripts

I opened `package.json` in the `Todo` directory and updated the `"scripts"` section:

```json
"scripts": {
  "start": "node index.js",
  "start-watch": "nodemon index.js",
  "dev": "concurrently \"npm run start-watch\" \"cd client && npm start\""
},
```
![alt text](./images/task%203%20front%202.jpg)


### Configure Proxy in Client package.json

```bash
cd client
vi package.json
```

I added this line inside the JSON object:

```json
"proxy": "http://localhost:5000"
```
![alt text](./images/task%203%20front%203.jpg)

This lets the React app make API calls to `/api/todos` without needing to specify the full backend URL.

### Install Axios

```bash
cd ..
npm install axios
```

### Create React Components

```bash
cd client/src
mkdir components
cd components
touch Input.js ListTodo.js Todo.js
```
![alt text](./images/task%203%20mkcomponent.jpg)


**Input.js** – Handles adding new todos:

![alt text](./images/task%203%20front-input.jpg)


**ListTodo.js** – Renders the list:

![alt text](./images/task%203%20list%20todo.jpg)


### Update App.js, App.css, and index.css

In `client/src/`, I updated `App.js` to use the Todo component, and styled `App.css` and `index.css` with the dark theme matching the final application design shown in the project guide.

![alt text](./images/task%203%20app.jpg)

![alt text](./images/task%203%20front%20css.jpg)

![alt text](./images/task%203%20front%20css%202.jpg)

![alt text](./images/task%203%20appcss.jpg)




---

## Step 7 – Running the Full Application

From the **`Todo` directory** (not `client`!), I ran:

```bash
npm run dev
```

This starts both the backend (port 5000) and the React frontend (port 3000) concurrently. I visited:

```
http://<EC2-Public-IP>:3000
```
Had errors so i remebered to open port `:3000`
![alt text](./images/task%203%20front%204.jpg)
![alt text](./images/task%203%20finalllleeeee.jpg)

---

## Mistakes I Made & How I Fixed Them

###  Running `npm start` Instead of `npm run dev`

When I ran `npm start`, only the backend Node server started — the React frontend never launched. I sat there confused about why the UI wasn't loading. The fix was simple: I needed to run `npm run dev` from the `Todo` directory, which uses `concurrently` to start **both** the backend watcher and the React dev server at the same time.

```bash
# Wrong
npm start

# Correct
npm run dev
```

---

###  Running `npm run dev` from the `client` Folder Instead of `Todo`

Another mistake I made was `cd`-ing into `client` and trying to run `npm run dev` from there. That doesn't work because the `dev` script which wires up both servers — only exists in the **root `Todo` package.json**, not the client one. I kept getting errors until I realized I was in the wrong directory.

```bash
# Wrong — running from Todo/client
cd client
npm run dev

# Correct — run from the root Todo directory
cd ~/Todo
npm run dev
```

---

###  Difficulty Connecting to MongoDB Atlas

Connecting to Atlas gave me headaches for a while. Here's what was tripping me up and how I solved each issue:

IP not whitelisted – My connection to Atlas kept failing because I never clicked "Allow Access from Anywhere" under Network Access. Once I clicked that button and confirmed, the connection went through immediately.

Once it was sorted, the terminal showed:

```
Database connected successfully
Server running on port 5000
```

---
