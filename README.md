# Tuto-Flask
Projet reprenant le tutoriel disponible depuis la page de documentation du projet : https://flask.palletsprojects.com/en/2.0.x/tutorial/


# Contents:
- Application Setup
- Define and Access the Database
- Blueprints and Views


# Application Setup
Instead of creating a [Flask](https://flask.palletsprojects.com/en/2.0.x/api/#flask.Flask) instance globally, you will create it inside a function. This function is known as the ___application factory___. Any configuration, registration, and other setup the application needs will happen inside the function, then the application will be returned.


## The Application Factory
The ```__init__```.py serves double duty: it will contain the application factory, and it tells Python that the flaskr directory should be treated as a package.

```flaskr/__init__.py```
1. ```app = Flask(__name__, instance_relative_config=True)``` creates the Flask instance.

    ```__name__``` is the name of the current Python module. The app needs to know where it’s located to set up some paths, and ```__name__``` is a convenient way to tell it that.

    ```instance_relative_config=True``` tells the app that configuration files are relative to the [instance folder](https://flask.palletsprojects.com/en/2.0.x/config/#instance-folders). The instance folder is located outside the ```flaskr``` package and can hold local data that shouldn’t be committed to version control, such as configuration secrets and the database file.

2. [app.config.from_mapping()](https://flask.palletsprojects.com/en/2.0.x/api/#flask.Config.from_mapping) sets some default configuration that the app will use:

    [```SECRET_KEY```](https://flask.palletsprojects.com/en/2.0.x/config/#SECRET_KEY) is used by Flask and extensions to keep data safe. It’s set to ```"dev"``` to provide a convenient value during development, but it should be overridden with a random value when deploying.

    ```DATABASE``` is the path where the SQLite database file will be saved. It’s under [app.instance_path](https://flask.palletsprojects.com/en/2.0.x/api/#flask.Flask.instance_path), which is the path that Flask has chosen for the instance folder. You’ll learn more about the database in the next section.

3. [app.config.from_pyfile()](https://flask.palletsprojects.com/en/2.0.x/api/#flask.Config.from_pyfile) overrides the default configuration with values taken from the ```config.py``` file in the instance folder if it exists. For example, when deploying, this can be used to set a real ```SECRET_KEY```.

    ```test_config``` can also be passed to the factory, and will be used instead of the instance configuration. This is so the tests you’ll write later in the tutorial can be configured independently of any development values you have configured.

4. [os.makedirs()](https://docs.python.org/3/library/os.html#os.makedirs) ensures that [app.instance_path](https://flask.palletsprojects.com/en/2.0.x/api/#flask.Flask.instance_path) exists. Flask doesn’t create the instance folder automatically, but it needs to be created because your project will create the SQLite database file there.

5. [@app.route()](https://flask.palletsprojects.com/en/2.0.x/api/#flask.Flask.route) creates a simple route so you can see the application working before getting into the rest of the tutorial. It creates a connection between the URL ```/hello``` and a function that returns a response, the string ```'Hello, World!'``` in this case.


## Run the Application
Now you can run your application using the ```flask``` command. From the terminal, tell Flask where to find your application, then run it in development mode. Remember, you should still be in the top-level ```flask-tutorial``` directory, not the ```flaskr``` package.

Development mode shows an interactive debugger whenever a page raises an exception, and restarts the server whenever you make changes to the code. You can leave it running and just reload the browser page as you follow the tutorial.

1. Set FLASK_APP to "flaskr"
2. Set FLASK_ENV to "development"
3. ```flask run```

You’ll see output similar to this:
```powershell
* Serving Flask app "flaskr"
* Environment: development
* Debug mode: on
* Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
* Restarting with stat
* Debugger is active!
* Debugger PIN: 855-212-761
```

Visit http://127.0.0.1:5000/hello in a browser and you should see the “Hello, World!” message. Congratulations, you’re now running your Flask web application!

If another program is already using port 5000, you’ll see ```OSError: [Errno 98]``` or ```OSError: [WinError 10013]``` when the server tries to start. See [Address already in use](https://flask.palletsprojects.com/en/2.0.x/server/#address-already-in-use) for how to handle that.


# Define and Access the Database
The application will use a [SQLite](https://sqlite.org/about.html) database to store users and posts. Python comes with built-in support for SQLite in the [sqlite3](https://docs.python.org/3/library/sqlite3.html#module-sqlite3) module.

SQLite is convenient because it doesn’t require setting up a separate database server and is built-in to Python. However, if concurrent requests try to write to the database at the same time, they will slow down as each write happens sequentially. Small applications won’t notice this. Once you become big, you may want to switch to a different database.

The tutorial doesn’t go into detail about SQL. If you are not familiar with it, the SQLite docs describe the [language](https://sqlite.org/lang.html).

# Connect to the Database
The first thing to do when working with a SQLite database (and most other Python database libraries) is to create a connection to it. Any queries and operations are performed using the connection, which is closed after the work is finished.

In web applications this connection is typically tied to the request. It is created at some point when handling a request, and closed before the response is sent.

```flaskr/db.py```
1. [g](https://flask.palletsprojects.com/en/2.0.x/api/#flask.g) is a special object that is unique for each request. It is used to store data that might be accessed by multiple functions during the request. The connection is stored and reused instead of creating a new connection if ```get_db``` is called a second time in the same request.

2. [current_app](https://flask.palletsprojects.com/en/2.0.x/api/#flask.current_app) is another special object that points to the Flask application handling the request. Since you used an application factory, there is no application object when writing the rest of your code. ```get_db``` will be called when the application has been created and is handling a request, so [current_app](https://flask.palletsprojects.com/en/2.0.x/api/#flask.current_app) can be used.

3. [sqlite3.connect()](https://docs.python.org/3/library/sqlite3.html#sqlite3.connect) establishes a connection to the file pointed at by the ```DATABASE``` configuration key. This file doesn’t have to exist yet, and won’t until you initialize the database later.

4. [sqlite3.Row](https://docs.python.org/3/library/sqlite3.html#sqlite3.Row) tells the connection to return rows that behave like dicts. This allows accessing the columns by name.

5. ```close_db``` checks if a connection was created by checking if ```g.db``` was set. If the connection exists, it is closed. Further down you will tell your application about the ```close_db``` function in the application factory so that it is called after each request.


## Create the Tables


In SQLite, data is stored in ___tables___ and ___columns___. These need to be created before you can store and retrieve data. Flaskr will store users in the ```user``` table, and posts in the ```post``` table. Create a file with the SQL commands needed to create empty tables:

```flaskr/schema.sql```

Add the Python functions that will run these SQL commands to the db.py file:

``` flaskr/db.py```

1. [open_resource()](https://flask.palletsprojects.com/en/2.0.x/api/#flask.Flask.open_resource) opens a file relative to the ```flaskr``` package, which is useful since you won’t necessarily know where that location is when deploying the application later. ```get_db``` returns a database connection, which is used to execute the commands read from the file.

2. [click.command()](https://click.palletsprojects.com/en/8.0.x/api/#click.command) defines a command line command called ```init-db``` that calls the ```init_db``` function and shows a success message to the user. You can read [Command Line Interface](https://flask.palletsprojects.com/en/2.0.x/cli/) to learn more about writing commands.

3. [with_appcontext](https://flask.palletsprojects.com/en/2.0.x/api/#flask.cli.with_appcontext) wraps a callback so that it’s guaranteed to be executed with the script’s application context. If callbacks are registered directly to the ```app.cli``` object then they are wrapped with this function by default unless it’s disabled.


## Register with the Application
The ```close_db``` and ```init_db_command``` functions need to be registered with the application instance; otherwise, they won’t be used by the application. However, since you’re using a factory function, that instance isn’t available when writing the functions. Instead, write a function that takes an application and does the registration.

```flaskr/db.py```

1. [app.teardown_appcontext()](https://flask.palletsprojects.com/en/2.0.x/api/#flask.Flask.teardown_appcontext) tells Flask to call that function when cleaning up after returning the response.

2. [app.cli.add_command()](https://click.palletsprojects.com/en/8.0.x/api/#click.Group.add_command) adds a new command that can be called with the ```flask``` command.

Import and call this function from the factory. Place the new code at the end of the factory function before returning the app.

```flaskr/__init__.py```


## Initialize the Database File
Now that ```init-db``` has been registered with the app, it can be called using the ```flask``` command, similar to the run command from the previous page.

>Note:
>
>If you’re still running the server from the previous page, you can either stop the server, or run this command in a new terminal. If you use a new terminal, remember to change to your project directory and activate the env as described in Installation. You’ll also need to set ```FLASK_APP``` and ```FLASK_ENV``` as shown on the previous page.

Run the ```init-db``` command:

```flask init-db```

You’ll see output similar to this:
```powershell
Initialized the database.
```

There will now be a ```flaskr.sqlite``` file in the ```instance``` folder in your project.


# Blueprints and Views
A view function is the code you write to respond to requests to your application. Flask uses patterns to match the incoming request URL to the view that should handle it. The view returns data that Flask turns into an outgoing response. Flask can also go the other direction and generate a URL to a view based on its name and arguments.


## Create a Blueprint
A [Blueprint](https://flask.palletsprojects.com/en/2.0.x/api/#flask.Blueprint) is a way to organize a group of related views and other code. Rather than registering views and other code directly with an application, they are registered with a blueprint. Then the blueprint is registered with the application when it is available in the [factory](https://refactoring.guru/design-patterns/factory-method) function.

Flaskr will have two blueprints, one for authentication functions and one for the blog posts functions. The code for each blueprint will go in a separate module. Since the blog needs to know about authentication, you’ll write the authentication one first.

```flaskr/auth.py```

This creates a [Blueprint](https://flask.palletsprojects.com/en/2.0.x/api/#flask.Blueprint) named ```'auth'```. Like the application object, the blueprint needs to know where it’s defined, so ```__name__``` is passed as the second argument. The ```url_prefix``` will be prepended to all the URLs associated with the blueprint.

Import and register the blueprint from the factory using [app.register_blueprint()](https://flask.palletsprojects.com/en/2.0.x/api/#flask.Flask.register_blueprint). Place the new code at the end of the factory function before returning the app.

```flaskr/__init__.py```

The authentication blueprint will have views to register new users and to log in and log out.


## The First View: Register
When the user visits the ```/auth/register``` URL, the ```register``` view will return [HTML](https://developer.mozilla.org/docs/Web/HTML) with a form for them to fill out. When they submit the form, it will validate their input and either show the form again with an error message or create the new user and go to the login page.

For now you will just write the view code. On the next page, you’ll write templates to generate the HTML form.

```flaskr/auth.py```

Here’s what the ```register``` view function is doing:
1. [@bp.route](https://flask.palletsprojects.com/en/2.0.x/api/#flask.Blueprint.route) associates the URL ```/register``` with the ```register``` view function. When Flask receives a request to ```/auth/register```, it will call the ```register``` view and use the return value as the response.

2. If the user submitted the form, [request.method](https://flask.palletsprojects.com/en/2.0.x/api/#flask.Request.method) will be ```'POST'```. In this case, start validating the input.

3. [request.form](https://flask.palletsprojects.com/en/2.0.x/api/#flask.Request.form) is a special type of [dict](https://docs.python.org/3/library/stdtypes.html#dict) mapping submitted form keys and values. The user will input their ```username``` and ```password```.

4. Validate that ```username``` and ```password``` are not empty.

5. If validation succeeds, insert the new user data into the database.
    * [db.execute] takes a SQL query with ```?``` placeholders for any user input, and a tuple of values to replace the placeholders with. The database library will take care of escaping the values so you are not vulnerable to a ___SQL injection attack___.
    * For security, passwords should never be stored in the database directly. Instead, [generate_password_hash()](https://werkzeug.palletsprojects.com/en/2.0.x/utils/#werkzeug.security.generate_password_hash) is used to securely hash the password, and that hash is stored. Since this query modifies data, [db.commit()](https://docs.python.org/3/library/sqlite3.html#sqlite3.Connection.commit) needs to be called afterwards to save the changes.
    * An [sqlite3.IntegrityError](https://docs.python.org/3/library/sqlite3.html#sqlite3.IntegrityError) will occur if the username already exists, which should be shown to the user as another validation error.

6. After storing the user, they are redirected to the login page. [url_for()](https://flask.palletsprojects.com/en/2.0.x/api/#flask.url_for) generates the URL for the login view based on its name. This is preferable to writing the URL directly as it allows you to change the URL later without changing all code that links to it. [redirect()](https://flask.palletsprojects.com/en/2.0.x/api/#flask.redirect) generates a redirect response to the generated URL.

7. If validation fails, the error is shown to the user. [flash()](If validation fails, the error is shown to the user. flash() stores messages that can be retrieved when rendering the template.) stores messages that can be retrieved when rendering the template.

8. When the user initially navigates to ```auth/register```, or there was a validation error, an HTML page with the registration form should be shown. [render_template()](https://flask.palletsprojects.com/en/2.0.x/api/#flask.render_template) will render a template containing the HTML, which you’ll write in the next step of the tutorial.


## Login
This view follows the same pattern as the ```register``` view above.

```flaskr/auth.py```

There are a few differences from the ```register``` view:

1. The user is queried first and stored in a variable for later use.

    [fetchone()](https://docs.python.org/3/library/sqlite3.html#sqlite3.Cursor.fetchone) returns one row from the query. If the query returned no results, it returns ```None```. Later, [fetchall()](https://docs.python.org/3/library/sqlite3.html#sqlite3.Cursor.fetchall) will be used, which returns a list of all results.

2. [check_password_hash()](https://werkzeug.palletsprojects.com/en/2.0.x/utils/#werkzeug.security.check_password_hash) hashes the submitted password in the same way as the stored hash and securely compares them. If they match, the password is valid.

3. [session](https://flask.palletsprojects.com/en/2.0.x/api/#flask.session) is a [dict](https://docs.python.org/3/library/stdtypes.html#dict) that stores data across requests. When validation succeeds, the user’s ```id``` is stored in a new session. The data is stored in a ___cookie___ that is sent to the browser, and the browser then sends it back with subsequent requests. Flask securely ___signs____ the data so that it can’t be tampered with.

Now that the user’s ```id``` is stored in the [session](https://flask.palletsprojects.com/en/2.0.x/api/#flask.session), it will be available on subsequent requests. At the beginning of each request, if a user is logged in their information should be loaded and made available to other views.

```flaskr/auth.py```

[bp.before_app_request()](https://flask.palletsprojects.com/en/2.0.x/api/#flask.Blueprint.before_app_request) registers a function that runs before the view function, no matter what URL is requested. load_logged_in_user checks if a user id is stored in the [session](https://flask.palletsprojects.com/en/2.0.x/api/#flask.session) and gets that user’s data from the database, storing it on g.user, which lasts for the length of the request. If there is no user id, or if the id doesn’t exist, [g.user](https://flask.palletsprojects.com/en/2.0.x/api/#flask.g) will be None.


## Logout
To log out, you need to remove the user id from the [session](https://flask.palletsprojects.com/en/2.0.x/api/#flask.session). Then ```load_logged_in_user``` won’t load a user on subsequent requests.

```flaskr/auth.py```


## Require Authentification in Other Views
Creating, editing, and deleting blog posts will require a user to be logged in. A ___decorator___ can be used to check this for each view it’s applied to.

```flaskr/auth.py```

This decorator returns a new view function that [wraps](https://docs.python.org/3/library/functools.html#functools.wraps) the original view it’s applied to. The new function checks if a user is loaded and redirects to the login page otherwise. If a user is loaded the original view is called and continues normally. You’ll use this decorator when writing the blog views.


## Endpoints and URLs
The [url_for()](https://flask.palletsprojects.com/en/2.0.x/api/#flask.url_for) function generates the URL to a view based on a name and arguments. The name associated with a view is also called the ___endpoint____, and by default it’s the same as the name of the view function.

For example, the ```hello()``` view that was added to the app factory earlier in the tutorial has the name ```'hello'``` and can be linked to with ```url_for('hello')```. If it took an argument, which you’ll see later, it would be linked to using ```url_for('hello', who='World')```.

When using a blueprint, the name of the blueprint is prepended to the name of the function, so the endpoint for the ```login``` function you wrote above is ```'auth.login'``` because you added it to the ```'auth'``` blueprint.


# Templates
You’ve written the authentication views for your application, but if you’re running the server and try to go to any of the URLs, you’ll see a ```TemplateNotFound``` error. That’s because the views are calling [render_template()](https://flask.palletsprojects.com/en/2.0.x/api/#flask.render_template), but you haven’t written the templates yet. The template files will be stored in the ```templates``` directory inside the flaskr package.

Templates are files that contain static data as well as placeholders for dynamic data. A template is rendered with specific data to produce a final document. Flask uses the [Jinja](https://jinja.palletsprojects.com/templates/) template library to render templates.

In your application, you will use templates to render [HTML](https://developer.mozilla.org/docs/Web/HTML) which will display in the user’s browser. In Flask, Jinja is configured to ___autoescape___ any data that is rendered in HTML templates. This means that it’s safe to render user input; any characters they’ve entered that could mess with the HTML, such as ```<``` and ```>``` will be ___escaped___ with ___safe___ values that look the same in the browser but don’t cause unwanted effects.

Jinja looks and behaves mostly like Python. Special delimiters are used to distinguish Jinja syntax from the static data in the template. Anything between ```{{``` and ```}}``` is an expression that will be output to the final document. ```{%``` and ```%}``` denotes a control flow statement like ```if``` and ```for```. Unlike Python, blocks are denoted by start and end tags rather than indentation since static text within a block could change indentation.


## The Base Layout
Each page in the application will have the same basic layout around a different body. Instead of writing the entire HTML structure in each template, each template will ___extend___ a base template and override specific sections.

```flaskr/templates/base.html```

[g](https://flask.palletsprojects.com/en/2.0.x/api/#flask.g) is automatically available in templates. Based on if ```g.user``` is set (```from load_logged_in_user```), either the username and a log out link are displayed, or links to register and log in are displayed. [url_for()](https://flask.palletsprojects.com/en/2.0.x/api/#flask.url_for) is also automatically available, and is used to generate URLs to views instead of writing them out manually.

After the page title, and before the content, the template loops over each message returned by [get_flashed_messages()](https://flask.palletsprojects.com/en/2.0.x/api/#flask.get_flashed_messages). You used [flash()](https://flask.palletsprojects.com/en/2.0.x/api/#flask.flash) in the views to show error messages, and this is the code that will display them.

There are three blocks defined here that will be overridden in the other templates:

    ```{% block title %}``` will change the title displayed in the browser’s tab and window title.

    ```{% block header %}``` is similar to title but will change the title displayed on the page.

    ```{% block content %}``` is where the content of each page goes, such as the login form or a blog post.

The base template is directly in the ```templates``` directory. To keep the others organized, the templates for a blueprint will be placed in a directory with the same name as the blueprint.


## Register

```flaskr/templates/auth/register.html```

```{% extends 'base.html' %}``` tells Jinja that this template should replace the blocks from the base template. All the rendered content must appear inside ```{% block %}``` tags that override blocks from the base template.

A useful pattern used here is to place ```{% block title %}``` inside ```{% block header %}```. This will set the title block and then output the value of it into the header block, so that both the window and page share the same title without writing it twice.

The ```input``` tags are using the ```required``` attribute here. This tells the browser not to submit the form until those fields are filled in. If the user is using an older browser that doesn’t support that attribute, or if they are using something besides a browser to make requests, you still want to validate the data in the Flask view. It’s important to always fully validate the data on the server, even if the client does some validation as well.


## Log In
This is identical to the register template except for the title and submit button.

```flaskr/templates/auth/login.html```


## Register A User
Now that the authentication templates are written, you can register a user. Make sure the server is still running (```flask run``` if it’s not), then go to http://127.0.0.1:5000/auth/register.

Try clicking the “Register” button without filling out the form and see that the browser shows an error message. Try removing the ```required``` attributes from the ```register.html``` template and click “Register” again. Instead of the browser showing an error, the page will reload and the error from [flash()](https://flask.palletsprojects.com/en/2.0.x/api/#flask.flash) in the view will be shown.

Fill out a username and password and you’ll be redirected to the login page. Try entering an incorrect username, or the correct username and incorrect password. If you log in you’ll get an error because there’s no ```index``` view to redirect to yet.


# Static Files
The authentication views and templates work, but they look very plain right now. Some [CSS](https://developer.mozilla.org/docs/Web/CSS) can be added to add style to the HTML layout you constructed. The style won’t change, so it’s a ___static___ file rather than a template.

Flask automatically adds a ```static``` view that takes a path relative to the ```flaskr/static``` directory and serves it. The ```base.html``` template already has a link to the ```style.css``` file:

```{{ url_for('static', filename='style.css') }}```

Besides CSS, other types of static files might be files with JavaScript functions, or a logo image. They are all placed under the ```flaskr/static``` directory and referenced with ```url_for('static', filename='...')```.

This tutorial isn’t focused on how to write CSS, so you can just copy the following into the ```flaskr/static/style.css``` file:

```flaskr/static/style.css```

You can read more about CSS from Mozilla’s documentation. If you change a static file, refresh the browser page. If the change doesn’t show up, try clearing your browser’s cache.