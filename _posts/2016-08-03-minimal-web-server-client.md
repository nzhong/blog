---
layout: post
title:  "Minimal Web Server & Client"
date: 2016-08-03 06:25:06 -0700
comments: true
---


NodeJS Server
=============

A friend was asking me, "how to setup a minimal web server, so I can have some simple web pages, and interact with the backend?"

#### Pre-Req

- NodeJS should be installed
- npm install express
- npm install body-parser

&nbsp;

#### Starter

The simplest code I can think of, would be NodeJS based. Here is a working 10-line example:

```
var express = require('express');
var app = express();
var server = app.listen(8000, function () {
  console.log("REST API listening on ", JSON.stringify(server.address()))
});

app.get('/', function (req, res) {
  console.log('Server received GET')
  res.sendStatus(200);
})
```

If we save the above as a file `server.js`, run the command `node server` in the command line, and load `http://127.0.0.1:8000` in the browser, we should see an `OK` in the browser, and `Server received GET` in the command line console window.


&nbsp;

#### Add static file handler

Our goal here is to have some simple web pages, so it would be helpful to be able to serve static files. To do this, we need to add one extra line at the bottom of the above server.js:

```
var express = require('express');
var app = express();
var server = app.listen(8000, function () {
  console.log("REST API listening on ", JSON.stringify(server.address()))
});

app.get('/', function (req, res) {
  console.log('Server received GET')
  res.sendStatus(200);
})

app.use('/public', express.static('static'));
```

The line `app.use('/public', express.static('static'));` tells NodeJS/Express that whenever we receive a request of the pattern `http://...:8000/public/*`, serve static files from the folder `static`.

Now save the above to server.js, create a folder `static` (in the same folder as server.js), and add a file `static/test.html`. You can put whatever content in there. Now load `http://127.0.0.1:8000/public/test.html` in the browser, we should see the static content we just put in the file.


&nbsp;

#### Handle Form Submit

We may want to handle form submit. Let's first write a simple form:

```
<form action="/formHandler" method="post">
  <input type="text" name="fieldA" value="">
  <input type="submit">
</form>
```

Change the `static/test.html` created in the previous step to these 4 lines. Now load `http://127.0.0.1:8000/public/test.html` and we should see a simple one-line form.

To hanlde the form submit (read the submitted value), we can use the `body-parser` library.

```
var express = require('express');
var bodyParser = require('body-parser');
var app = express();
var server = app.listen(8000, function () {
  console.log("REST API listening on ", JSON.stringify(server.address()))
});

app.get('/', function (req, res) {
  console.log('Server received GET')
  res.sendStatus(200);
})

app.use('/public', express.static('static'));

app.use( bodyParser.json() );
app.use( bodyParser.urlencoded({extended: true}) );
app.post('/formHandler', function(req, res) {
  console.log( 'Form Submit Value = ' + req.body.fieldA );
  res.sendStatus(200);
});
```

(The second line and the last block were added this round)

Now if we load `http://127.0.0.1:8000/public/test.html` and fill that one-line form with some value, and click [Submit], we should see in the browser window that the URL is changed to `http://127.0.0.1:8000/formHandler` with content 'OK'. In the command line console window we should see `Form Submit Value = ...` with the value we just submitted.


&nbsp;

#### REST API & Client Side Dynamic Data

To wrap this up, let's have the server send some JSON data via a REST API:

```
var express = require('express');
var bodyParser = require('body-parser');
var app = express();
var server = app.listen(8000, function () {
  console.log("REST API listening on ", JSON.stringify(server.address()))
});

app.get('/', function (req, res) {
  console.log('Server received GET')
  res.sendStatus(200);
})

app.use('/public', express.static('static'));

app.use( bodyParser.json() );
app.use( bodyParser.urlencoded({extended: true}) );
app.post('/formHandler', function(req, res) {
  console.log( 'Form Submit Value = ' + req.body.fieldA );
  res.sendStatus(200);
});

var myObj = {
  name: 'John Doe',
  age: 21,
  interest: ['Chess', 'Music', 'Tennis']
}
app.get('/data', function (req, res) {
  res.setHeader('Content-Type', 'application/json');
  res.send( JSON.stringify(myObj) );
})
```

(The last block was added this round)

Here we added a `(GET) /data` handler. Load `http://127.0.0.1:8000/data` in the browser we should see `{"name":"John Doe","age":21,"interest":["Chess","Music","Tennis"]}` .

For a really dumb way to do client side dynamic data, we could change `static/test.html` to the following content:

```
<script src="https://code.jquery.com/jquery-3.1.0.min.js"
  integrity="sha256-cCueBR6CsyA4/9szpPfrX3s49M9vUU5BgtiJj06wt/s="
  crossorigin="anonymous"></script>

<form action="/formHandler" method="post">
  <input type="text" name="fieldA" value="">
  <input type="submit">
</form>

Name: <div id='name'></div>
Age: <div id='age'></div>
Interest: <div id='interest'></div>
<script>
  $.ajax({
    url: "http://127.0.0.1:8000/data",
    success: function(data, status, jqXHR) {
      document.getElementById('name').innerHTML=data.name;
      document.getElementById('age').innerHTML=data.age;
      document.getElementById('interest').innerHTML=data.interest;
    },
    error: function(jqXHR, status, errorThrown) {
      console.log(status);
    }
  });
</script>
```

Here we use jQuery to do a GET on `http://127.0.0.1:8000/data`, and fill in the placeholders with the retrieved (thus dynamic) value.

Load `http://127.0.0.1:8000/public/test.html` we should see

```
Name:
John Doe
Age:
21
Interest:
Chess,Music,Tennis
```

And the values are really dynamically loaded from the backend!


GitHub Repo
===========

I have put the above code into a simple github repo: <a href="https://github.com/nzhong/minimal-web-server-client">https://github.com/nzhong/minimal-web-server-client</a>. There are really just three files

* <a href="https://github.com/nzhong/minimal-web-server-client/blob/master/server.js">server.js</a>
* <a href="https://github.com/nzhong/minimal-web-server-client/blob/master/static/test.html">static/test.html</a>
* <a href="https://github.com/nzhong/minimal-web-server-client/blob/master/package.json">package.json</a> specifies our dependencies: express & body-parser

To try it out, you can do the following:

* git clone https://github.com/nzhong/minimal-web-server-client
* cd minimal-web-server-client
* npm install
* node server

Open up your chrome browser, and load "http://127.0.0.1:8000/public/test.html"



<br/><br/><br/>
<br/><br/><br/>