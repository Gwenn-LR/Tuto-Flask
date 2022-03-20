# Tuto-Flask
Projet reprenant le tutoriel disponible depuis la page de documentation du projet : https://flask.palletsprojects.com/en/2.0.x/tutorial/


# Contents:
- Application Setup


# Application Setup
Instead of creating a [Flask](https://flask.palletsprojects.com/en/2.0.x/api/#flask.Flask) instance globally, you will create it inside a function. This function is known as the ___application factory___. Any configuration, registration, and other setup the application needs will happen inside the function, then the application will be returned.


## The Application Factory
The ```__init__```.py serves double duty: it will contain the application factory, and it tells Python that the flaskr directory should be treated as a package.

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
3. flask run

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