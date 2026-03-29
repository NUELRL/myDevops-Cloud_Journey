# MEAN Stack Deployment to Ubuntu in AWS

![Node.js](https://img.shields.io/badge/Node.js-339933?style=for-the-badge&logo=nodedotjs&logoColor=white)
![MongoDB](https://img.shields.io/badge/MongoDB-13AA52?style=for-the-badge&logo=mongodb&logoColor=white)
![Express.js](https://img.shields.io/badge/Express.js-000000?style=for-the-badge&logo=express&logoColor=white)
![Angular](https://img.shields.io/badge/Angular-DD0031?style=for-the-badge&logo=angular&logoColor=white)
![AWS](https://img.shields.io/badge/Amazon%20AWS-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)

## Overview

In this project, I built a simple Book Register web form using the MEAN stack and deployed it on an Ubuntu Server in AWS. My goal was to get a feel for how MongoDB, Express, Angular, and Node.js all work together in a real server setup.

---

## Step 1: Install Node.js

I started by updating my Ubuntu system and installing Node.js.

```bash
sudo apt update
```
![sudo update](./images/task%204%20sudo%20update.jpg)

```bash
sudo apt upgrade
```
![sudo upgrade](./images/task%204%20sudo%20upgrade.jpg)

```bash
sudo apt -y install curl dirmngr apt-transport-https lsb-release ca-certificates
```
![install certs](./images/task%204%20cert%202.jpg)

```bash
curl -sL https://deb.nodesource.com/setup_20.x | sudo -E bash -
```
![nodesource setup](./images/task%204%20cert%201.jpg)

```bash
sudo apt install -y nodejs
```

---

## Step 2: Install MongoDB

Next, I set up MongoDB to store my book records. MongoDB saves data in a flexible, JSON-like format which works really well with Node.js.

First, I installed `gnupg` which is needed to handle security keys:

![install gnupg](./images/task%204%20gnupg%20-i.jpg)

Then I added the MongoDB security key:

```bash
curl -fsSL https://pgp.mongodb.com/server-7.0.asc | \
sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg \
--dearmor
```

After that, I told my system where to find MongoDB packages:

```bash
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list
```
![mongo list](./images/task%204%20mongo%20list.jpg)

Then I updated the package list and installed MongoDB:

```bash
sudo apt-get update
sudo apt-get install -y mongodb-org
```
![install mongo](./images/task%204%20--i%20mongo.jpg)

I checked that MongoDB was up and running:

```bash
sudo systemctl status mongod
```
![mongo running](./images/task%204%20mongo%20success.jpg)

Next, I installed `body-parser`, a package that helps the server read data sent from the browser:

```bash
sudo npm install body-parser
```
![install body-parser](./images/task%204%20i%20body-parser.jpg)

Then I created a project folder and set it up as an npm project:

```bash
mkdir Books && cd Books
npm init
```
![npm init](./images/task%204%20all-npm.jpg)

---

## Step 3: Set Up Express and Routes

I installed Express (the web server framework) and Mongoose (which helps Node.js talk to MongoDB):

```bash
sudo npm install express mongoose
```

Then I created a folder for my app logic and set up a routes file that handles GET, POST, and DELETE requests for books:

```bash
mkdir apps && cd apps
vi routes.js
```

Here's the code I added to `routes.js`:

```javascript
var Book = require('./models/book');

module.exports = function(app) { 
  app.get('/book', function(req, res) { 
    Book.find({}, function(err, result) { 
      if ( err ) throw err; 
      res.json(result); 
    }); 
  });  
  app.post('/book', function(req, res) { 
    var book = new Book( { 
      name:req.body.name, 
      isbn:req.body.isbn, 
      author:req.body.author, 
      pages:req.body.pages 
    }); 
    book.save(function(err, result) { 
      if ( err ) throw err; 
      res.json( { 
        message:"Successfully added book", 
        book:result 
      }); 
    }); 
  }); 
  app.delete("/book/:isbn", function(req, res) { 
    Book.findOneAndRemove({ isbn: req.params.isbn }, function(err, result) { 
      if ( err ) throw err; 
      res.json( { 
        message: "Successfully deleted the book", 
        book: result 
      }); 
    }); 
  }); 
  var path = require('path'); 
  app.get('*', function(req, res) { 
    res.sendfile(path.join(__dirname + '../public/index.html')); 
  }); 
}; 
```
![routes file](./images/task%204%20u-routes.jpg)

I also created the database model in `models/book.js`. This file describes what a "book" looks like in the database:

```javascript
var mongoose = require('mongoose');

var dbHost = 'mongodb://localhost:27017/test'; 
mongoose.connect(dbHost); 
mongoose.connection; 
mongoose.set('debug', true); 
var bookSchema = mongoose.Schema( { 
  name: String, 
  isbn: {type: String, index: true}, 
  author: String, 
  pages: Number 
}); 
var Book = mongoose.model('Book', bookSchema); 
module.exports = mongoose.model('Book', bookSchema);
```
![book model](./images/task%204%20book.jpg)

---

## Step 4: Set Up the Main Server File

Back in the `Books` folder, I created `server.js`, which is the entry point for the whole app:

```bash
vi server.js
```

```javascript
var express = require('express');
var bodyParser = require('body-parser'); 

var app = express(); 
app.use(express.static(__dirname + '/public')); 
app.use(bodyParser.json()); 
require('./apps/routes')(app); 
app.set('port', 3300); 
app.listen(app.get('port'), function() { 
    console.log('Server up: http://localhost:' + app.get('port')); 
});
```
![server.js](./images/task%204%20vi%20server.jpg)

---

## Step 5: Create the Frontend with AngularJS

I built the frontend using AngularJS, which handles what the user sees and does in the browser.

```bash
mkdir public && cd public
vi script.js
```

I added the AngularJS controller code to `script.js`. I also added a few extra lines to make the UI feel more responsive and dynamic:

```javascript
var app = angular.module('myApp', []);

app.controller('myCtrl', function($scope, $http) { 
  $http( { 
    method: 'GET', 
    url: '/book' 
  }).then(function successCallback(response) { 
    $scope.books = response.data; 
  }, function errorCallback(response) { 
    console.log('Error: ' + response); 
  }); 
  $scope.del_book = function(book) { 
    $http( { 
      method: 'DELETE', 
      url: '/book/' + book.isbn, 
      params: {'isbn': book.isbn} 
    }).then(function successCallback(response) { 
      console.log(response); 
    }, function errorCallback(response) { 
      console.log('Error: ' + response); 
    }); 
  }; 
  $scope.add_book = function() { 
    var body = '{ "name": "' + $scope.Name +  
    '", "isbn": "' + $scope.Isbn + 
    '", "author": "' + $scope.Author +  
    '", "pages": "' + $scope.Pages + '" }'; 
    $http({ 
      method: 'POST', 
      url: '/book', 
      data: body 
    }).then(function successCallback(response) { 
      console.log(response); 
    }, function errorCallback(response) { 
      console.log('Error: ' + response); 
    }); 
  }; 
});
```
![script.js](./images/task%204%20script%20-u.jpg)

Then I created the HTML file that the user actually sees:

```bash
vi index.html
```

```html
<!doctype html>
<html ng-app="myApp" ng-controller="myCtrl"> 
  <head> 
    <script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.6.4/angular.min.js"></script>
    <script src="script.js"></script> 
  </head> 
  <body> 
    <div> 
      <table> 
        <tr> 
          <td>Name:</td> 
          <td><input type="text" ng-model="Name"></td> 
        </tr> 
        <tr> 
          <td>Isbn:</td> 
          <td><input type="text" ng-model="Isbn"></td> 
        </tr> 
        <tr> 
          <td>Author:</td> 
          <td><input type="text" ng-model="Author"></td> 
        </tr> 
        <tr> 
          <td>Pages:</td> 
          <td><input type="number" ng-model="Pages"></td> 
        </tr> 
      </table> 
      <button ng-click="add_book()">Add</button> 
    </div> 
    <hr> 
    <div> 
      <table> 
        <tr> 
          <th>Name</th> 
          <th>Isbn</th> 
          <th>Author</th> 
          <th>Pages</th> 
        </tr> 
        <tr ng-repeat="book in books"> 
          <td>{{book.name}}</td> 
          <td>{{book.isbn}}</td> 
          <td>{{book.author}}</td> 
          <td>{{book.pages}}</td> 
          <td><input type="button" value="Delete" data-ng-click="del_book(book)"></td>
        </tr> 
      </table> 
    </div> 
  </body> 
</html>
```

The original HTML looked a bit plain and outdated, so I updated it to give it a more modern look:

![updated UI](./images/task%204%20ui.jpg)

---

## Step 6: Launch and Test the App

Before starting the server, I opened port `3300` in my AWS security group settings so the app could be reached from the internet:

![security group](./images/task%204%20change%20roles.jpg)

Then I went back to the `Books` folder and started the server:

```bash
cd ..
node server.js
```

And here's the finished product running in the browser:

![final result](./images/task%204%20finalleeee.jpg)

---

## What I Learned

Through this project, I got hands-on experience with each part of the MEAN stack:

- **MongoDB** — I used it to store book records (name, ISBN, author, and page count) as documents in a database.
- **Express** — I used it to build the backend, handling requests to read, add, and delete books.
- **AngularJS** — I used it on the frontend to send and receive data from the server and update the page without reloading.
- **Node.js** — It runs the whole server, listening for requests and sending back responses.