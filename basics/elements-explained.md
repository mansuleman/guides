# CompoundJS starter guide

We are going to learn compoundjs core basics without using code generator.

## ./server.js

Let's start with a single file, call it **server.js**. Put two lines of code here:

    require('compound').createServer();
    if (!module.parent) app.listen(3000);

That's it - our first compoundjs project, which is regular express application. We
can run it using `node server.js` command. Let's start and try to load root url of
our web service.

We got an error 404: "Cannot GET /". It works! We can work with the express
application as usual:

    app.get('/', function (req, res) {
        res.send('hello');
    });

But wait. It's not just express, we should put things to the right places.

## ./config/environment.js

Let's configure our environment using file `config/environment.js` with following
contents:

    var express = require('express');

    app.configure(function () {
        app.use(express.static(app.root, {maxAge: 86400000}));
        app.set('view engine', 'ejs');
        app.set('view options', {complexNames: true});
        app.set('jsDirectory', '/javascripts/');
        app.set('cssDirectory', '/stylesheets/');
        app.use(express.bodyParser());
        app.use(express.cookieParser('secret'));
        app.use(express.session({secret: 'secret'}));
        app.use(express.methodOverride());
        app.use(app.router);
    });

Here we are setting up our middleware stack and do some basic configuration like
setting view engine, etc. Ok, but we may want to configure our application slightly
different for different envs (`NODE_ENV` environment variable), for example use
error reporter for development mode, or turn on assets pipelining for production,
or set any additional parameters depending on env. For that case we could create
`./config/environments/ENVNAME.js` file, where `ENVNAME` is name of env (test,
development, production, heroku, ec2staging)

Lets create two:

config/environments/development.js:

    app.configure('development', function () {
        app.disable('view cache');
        app.disable('model cache');
        app.disable('eval cache');
        app.enable('log actions');
        app.enable('env info');
        app.use(require('express').errorHandler({ dumpExceptions: true, showStack: true }));
    });

config/environments/production.js:

    app.configure('production', function () {
        app.enable('view cache');
        app.enable('model cache');
        app.enable('eval cache');
        app.enable('merge javascripts');
        app.enable('merge stylesheets');
        app.disable('assets timestamps');
        app.use(require('express').errorHandler());
        app.settings.quiet = true;
    });

Let me describe each option briefly:

- view cache: turns on view cache, which is useful for production but not necessary for development (it forces us to reload server when we need to modify view)
- model cache: same for models (files inside app/models/ directory), in development mode models will be reloaded before each request
- eval cache: same for controllers and any script evaluated in new context (models too)
- merge javascripts: join all js files into one
- merge stylesheets: join all css files into one
- log actions: turn on verbose request logging (show actions and filters in log)
- assets timestamps: adds timestamps to the end of js and css files
- quiet: put logging info to `./log/ENVNAME.log` file instead of STDOUT
- env info: enables backdoor for retriving information about environment (versions, packages, etc), this feature used in autogenerated `index.html` file when clicking a link:

    <a href="/compound/environment.json" id="show-env-info-link" data-remote="true" data-jsonp="load">Information about application environment</a>

## ./config/routes.js

Now we have successfully configured application and can focus on development. First
of all we may want to create some routes. We should put our routes to
`config/routes.js` file:

    module.exports = function (map) {
        map.root('dashboard#welcome');
    
        map.all(':controller/:action');
        map.all(':controller/:action/:id');
    };

Basically this module should export one function that accepts `map` argument. It
can also export `routes` function, result will be the same, but it may look nicer:

    exports.routes = function (map) {
        map.root('dashboard#welcome');

        map.all(':controller/:action');
        map.all(':controller/:action/:id');
    };

At the bottom of `routes` function we have couple of generic routes, which accept
all requests to any controller and any action - this is nice and powerful feature,
but also may create some security problems, so use it carefully.

We also created one root route that maps action `welcome` of `dashboard`
controller to '/' url. It's time to create controller.

## ./app/controllers/dashboard\_controller.js

Controller file by default executed in new context that has shortcuts for most
often used response methods, such as `send`, `render`, `redirect` and others. It
means that you shouldn't accept `req` and `res` params in each action, you can just
call `send` instead of `res.send`, or `render` instead of `res.render`.

We need controller with one `welcome` action:

    action(function welcome() {
        send('hi');
    });

Alternative syntax (coffee-friendly):

    action('welcome', function () {
        send('hi');
    });

It allows us to write controller action in coffeescript:

    action 'welcome', ->
        send 'hi'

This is a common coding conventions in compoundjs controllers: all utility methods
may accept one argument - named function, or two arguments - name and function.

## ./app/views

What if we want to render a view? By default all views are located in `app/views`
directory. We can call `render` without params and it will render view with the
same name as action located within subdirectory with the name of controller, i.e.

`dashboard_controller.js`:

    action 'welcome', ->
        render()

will render `app/views/dashboard/welcome.ejs`. We also can pass params to view
using two ways. First - pass object as first or second param of `render` method.

    action(function welcome() {
        render({user: {name: 'Anatoliy'}});
    });

Second - using context object of action:

    action(function welcome() {
        this.user = {name: 'Anatoliy'};
        render();
    });

If we want to render a different view, we can specify its name as the first param
of `render` method.

Our `welcome.ejs` would be rendered as is, but if we need to render it within any
layout we should create layout view inside `app/views/layouts`. By default (when
layout is not specified manually using `layout` method) the following logic of
layout selection is used:

1. Use layout with the name of controller (if exists), i.e. `app/views/layouts/dashboard.ejs`
2. Use default layout `app/views/layouts/application.ejs`

Let's proceed with the last option, here are the contents of our layout:

    <!DOCTYPE html>
    <html lang="en">
        <head>
            <title><%= title %></title>
            <%- stylesheet_link_tag('bootstrap', 'bootstrap-responsive') %>
            <%- javascript_include_tag('http://ajax.googleapis.com/ajax/libs/jquery/1.7.1/jquery.min.js', 'bootstrap', 'rails', 'application') %>
            <%- csrf_meta_tag() %>
        </head>
        <body>
            <div class="navbar">
                <div class="navbar-inner">
                    <div class="container">
                        <a class="brand" href="#">Project name</a>
                    </div>
                </div>
            </div>

            <div class="container">
                <% var flash = request.flash('info').pop(); if (flash) { %>
                    <div class="alert alert-info">
                        <a class="close" data-dismiss="alert">×</a>
                        <%- flash %>
                    </div>
                <% } %>

                <% flash = request.flash('error').pop(); if (flash) { %>
                    <div class="alert alert-error">
                        <a class="close" data-dismiss="alert">×</a>
                        <%- flash %>
                    </div>
                <% }; %>

                <%- body %>

            </div>
        </body>
    </html>

This layout used bootstrap and jquery, it also loads some javascripts and defines
CSRF protection meta tags. Within containers we are printing out flash messages
(if any), and then print contents of our view (which is saved in `body` variable).

## ./app/models

The last thing we should talk about - models. Basically we have two kind of models
in compound: persistent and not persistent. Persistent models defined in database
schema. Not persistent models defined directly in `app/models/*.js` files (these
models should be exported from this file as `module.exports = MyModel`

### ./db/schema.js

Let's say we have an User model with `name` property:

    define('User', function () {
        property('name', String);
    });

Now `User` variable available globally in application and also stored in
`app.models.User`.

But where is it persisted? We have to select some database engine. For configuring
database settings we should create `config/database.json` file. Structure of this
file:

{ environmentName: { driver: 'name of driver', database: '', username: '', password: '', host: 'localhost', port: '' } }

for example we want redis in development env:

{ development: { driver: 'redis' } }

One more thing we need at this point - install jugglingdb ORM, using npm:

    npm install jugglingdb

When you try to start your server using `node server.js` or `compound server` you
will get error message explaining which database driver you should install. In our
case it's `redis`:

    npm install redis
