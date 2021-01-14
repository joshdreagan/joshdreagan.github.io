---
title: Connecting FeedHenry To Fuse REST Quickstart
cover: /2015/05/08/connecting_feedhenry_to_fuse_rest_quickstart/post-bg.png
thumbnail: /2015/05/08/connecting_feedhenry_to_fuse_rest_quickstart/post-bg.png
date: 2015-05-08 00:00:00
tags:
- feedhenry
- fuse
- karaf
- fabric8
- camel
- cxf
- openshift
---

This guide demonstrates step-by-step how to connect a FeedHenry Hello World application to a JBoss Fuse REST quickstart backend running on Red Hat's OpenShift Online.
<!-- more -->

Here are a few assumptions before you begin:

- {% post_link "getting_started__feedhenry_hello_world" "You've already completed the guide in my previous blog about creating a FeedHenry Hello World project" %}
- {% post_link "getting_started__fuse_rest_quickstart_on_openshift" "You've already completed the guide in my previous blog about creating a Fuse REST quickstart project on OpenShift" %}

Once you've completed the prerequisite blog guides, connecting the two is a simple matter of modifying a few files. You can even use the browser based editor so you can do everything online.

Log into your FeedHenry domain and find your previously created Hello World project.

![Explore Project](explore_project.png)

# Step 1: Modify the Hello World Cloud App

First, you'll modify the cloud app files. Click on the "__Cloud App__" panel shown below.

![Explore Project](explore_project_cloud_code.png)

Next, click on the "__Editor__" button in the column on the left.

![Explore Project](explore_project_cloud_code_editor_button.png)

Edit the `/package.json` file and add a dependency for "_xml2js_" version "_~0.4.8_" to the end of the _dependencies_ section. If you'd like, you can copy/paste the entire contents from the text below.

{% codeblock lang:json %}
{
  "name": "fh-app",
  "version": "0.2.0",
  "dependencies": {
    "body-parser": "~1.0.2",
    "cors": "~2.2.0",
    "express": "~4.0.0",
    "fh-mbaas-api": "~4.9.0",
    "mocha": "^2.1.0",
    "request": "~2.40.0",
    "xml2js": "~0.4.8"
  },
  "devDependencies": {
    "grunt-contrib-watch": "latest",
    "grunt-nodemon": "0.2.0",
    "grunt-concurrent": "latest",
    "supertest": "0.8.2",
    "should": "2.1.1",
    "proxyquire": "0.5.3",
    "grunt-shell": "0.6.4",
    "istanbul": "0.2.7",
    "grunt-env": "~0.4.1",
    "load-grunt-tasks": "~0.4.0",
    "time-grunt": "~0.3.1",
    "grunt-node-inspector": "~0.1.5",
    "grunt-open": "~0.2.3",
    "grunt-plato": "~1.0.0"
  },
  "main": "application.js",
  "scripts": {
    "test": "grunt test",
    "debug": "grunt serve"
  },
  "license": "mit"
}
{% endcodeblock %}

Next, edit the `/lib/hello.js` file and replace its contents with the text below. __Make sure to replace the _uri_ field in both the _get_ and _post_ methods with the URI to your Fuse REST quickstart that you created in the previous guide.__

{% codeblock lang:javascript %}
var express = require('express');
var bodyParser = require('body-parser');
var cors = require('cors');
var request = require('request');
var xml2js = require('xml2js');

function helloRoute() {
  var hello = new express.Router();
  hello.use(cors());
  hello.use(bodyParser());

  hello.get('/', function(req, res) {
    console.log(new Date(), 'In hello route GET / req.query=', req.query);
    var id = req.query && req.query.hello ? req.query.hello : '000';
    request({uri: 'http://restchild-joshdreagan.rhcloud.com/cxf/crm/customerservice/customers/' + id,
             method: "GET"}, function (error, response, body) {
        xml2js.parseString(body, function (err, result) {
          var name = result.Customer.name[0];
          res.json({msg: 'Hello ' + name});
        });
    });
  });

  hello.post('/', function(req, res) {
    console.log(new Date(), 'In hello route POST / req.body=', req.body);
    var id = req.body && req.body.hello ? req.body.hello : '000';
    request({uri: 'http://restchild-joshdreagan.rhcloud.com/cxf/crm/customerservice/customers/' + id,
             method: "GET"}, function (error, response, body) {
        xml2js.parseString(body, function (err, result) {
          var name = result.Customer.name[0];
          res.json({msg: 'Hello ' + name});
        });
    });
  });

  return hello;
}

module.exports = helloRoute;
{% endcodeblock %}

Finally, edit the `/README.md` file and change the sample request body to "_123_" instead of "_world_". If you'd like, you can copy/paste the entire contents from the text below. _This step is optional, but is nice to do since it will allow you to test directly from the "__Docs__" page._

{% codeblock %}
# FeedHenry Hello World MBaaS Server

This is a blank 'hello world' FeedHenry MBaaS. Use it as a starting point for building your APIs.

# Group Hello World API

# hello [/hello]

'Hello world' endpoint.

## hello [POST]

'Hello world' endpoint.

+ Request (application/json)
    + Body
            {
              "hello": "123"
            }

+ Response 200 (application/json)
    + Body
            {
              "msg": "Hello world"
            }
{% endcodeblock %}

Now, save all of the files and click on the "__Deploy__" button in the column on the left.

![Deploy Project](cloud_code_deploy_button.png)

That should bring you to the "__Deploy__" page. Click on the "__Deploy Cloud App__" button.

![Deploy Project](cloud_code_deploy_page.png)

Deployment should take less than 1 minute. You will be shown a progress bar and status text box.

![Deploy Project](cloud_code_deploy_in_progress.png)

When the deploy process is complete, you will see a green progress bar and no errors in the status text box.

![Deploy Project](cloud_code_deploy_success.png)

Once everything deploys successfully, you can click on the "__Docs__" button in the column on the left.

![Test Project](cloud_code_docs_button.png)

This will take you to the test page for your app. It should look like the picture below.

![Test Project](cloud_code_test_page.png)

Click on the "__Try it!__" button to expand the _request_ & _response_ text boxes.

![Test Project](cloud_code_test_page_request.png)

Click on the "__Submit__" button to invoke the service. You should see a successful response like the one pictured below.

![Test Project](cloud_code_test_page_response.png)

# Step 2: Modify The Hello World Cordova Light App

Now, you'll modify the Cordova application files. Click on the "__Cordova Light App__" panel shown below.

![Explore Project](explore_project_apps.png)

Next, click on the "__Editor__" button in the column on the left.

![Explore Project](explore_project_apps_editor_button.png)

Edit the `/www/index.html` file and replace the placeholder text for the input box with "_Enter Your ID Here._". If you'd like, you can copy/paste the entire contents from the text below.

{% codeblock lang:html %}
<!DOCTYPE html>
  <head>
    <meta charset="utf-8">

    <meta name="viewport" content="width=device-width, initial-scale=1, user-scalable=no">

    <title>Hello World</title>

    <link rel="stylesheet" href="css/app.css">
  </head>

  <body>
  <header class="header-footer-bg">
      <div class="center">
          <h3>FeedHenry
              <small>QuickStart - HTML5</small>
          </h3>
      </div>
  </header>

  <div id="count" class="">
      <div id="formWrapper">
          <p id="description">This is a basic Javascript App that can take in your name, send it to a cloud app and display the response.</p>
          <br>

          <div class="input-div">
              <input id="hello_to" type="text" class="input-text" placeholder="Enter Your ID Here."/>
          </div>

          <br>

          <button id="say_hello" type="button" class="say-hello-button">Say Hello From The Cloud</button>
          <div id="cloudResponse" class="cloudResponse"></div>
      </div>
  </div>

      <footer class="header-footer-bg">
          <div>
              <small class="right">
                  <!-- v.&nbsp;{{ version }} -->
              </small>
          </div>
      </footer>

    <script src="feedhenry.js"></script>
    <script src="hello.js"></script>
  </body>
</html>
{% endcodeblock %}

Don't forget to save the file. The simulator on the right should automatically update. Don't worry if it doesn't. Sometimes it needs a page refresh. However, since the changes we made were purely cosmetic, it doesn't really matter if you see the update.

![Test Project](apps_test_sim.png)

Enter "_123_" into the input box on the simulator and click "__Say Hello From The Cloud__".

![Test Project](apps_test_sim_request.png)

If everything was successful, you should see a response like the one pictured below.

![Test Project](apps_test_sim_response.png)

Now you've successfully wired your FeedHenry Hello World project to a backend Fuse REST quickstart running on OpenShift.
