# Heroku Django Deployment



1. Inside the project-directory add a `.env`, `.gitignore` and `Profile` file. Make sure they are in the project outer directory where the `manage.py` file is found:

```bash
touch .env .gitignore Procfile
```

2. From the Terminal, with a pipenv virtual environment running, after installing `django` and `psycopg2-binary`, add the other packages needed for deployment:

```bash
pipenv install whitenoise dj-database-url gunicorn django-cors-headers
```

3. If you haven't already initialized your project for git, run `git init`.

4. Open the project in the editor with `code .`. Add this one line to your `Procfile`. Make sure to replace the example_django with the name of your **project root**.

```
# Replace example_django with your project root name:
web: gunicorn example_django.wsgi
```

5. Add the following to your `.gitignore` file:

```
.env
*.log
*.pot
*.pyc
__pycache__/
local_settings.py
db.sqlite3
db.sqlite3-journal
media

# General MacOS files to ignore
.DS_Store
.DocumentRevisions-V100
.fseventsd
.Spotlight-V100
.TemporaryItems
.Trashes
.VolumeIcon.icns
.com.apple.timemachine.donotpresent
```

6. Update the `settings.py` file: 

```python
# At the top of the settings.py file add:

import os
import dj_database_url

```

> You'll notice that the `settings.py` contains a warning not to leave your secret key in production so copy it into your `.env` file and replace it with the following: 

```python
# SECURITY WARNING: keep the secret key used in production secret! 

# Copy the entire SECRET_KEY line into your .env file and then change it to read:
SECRET_KEY = os.environ['SECRET_KEY']
```

> Note the message to not run debug in production. Instead, we'll create an environment variable called MODE and set it to 'dev' in our .env file locally and 'production' in the Heroku configvars. We can use a ternary operator to set the DEBUG to True or False based on the environment variable.

```python
# SECURITY WARNING: don't run with debug turned on in production! 

# Replace the DEBUG = True with:
DEBUG = True if os.environ['MODE'] == 'dev' else False
```
> The allowed hosts for your app depends on what front end apps you want to allow to connect to your back end.  This can prevent others from using your API, so you can add your deployed front end app url to the list or you can use a `'*'` to allow anyone to connect to your app.

```python
# Change this according to your needs:
ALLOWED_HOSTS = ['*']
```
> If you haven't already done so, add `corsheaders` to the installed apps list:

```python
INSTALLED_APPS = [
    ...
    'corsheaders',
    ...
]
```

> Configure whitenoise and cors by updating the settings.py file by updating the MIDDLEWARE list. **The order matters here!**:

```python
MIDDLEWARE = [
# Add whitenoise right AFTER 'django.middleware.security.SecurityMiddleware',
'whitenoise.middleware.WhiteNoiseMiddleware',
...
'corsheaders.middleware.CorsMiddleware',
# Add corsheader right BEFORE 'django.middleware.common.CommonMiddleware',
...
]
```

> Configure the corsheaders package by adding **one** of the following.  For more details about what these options do, take a look at the [Configuration](https://github.com/adamchainz/django-cors-headers#configuration) section in the doc. **Make sure you [clear your browser cache](https://support.piktochart.com/article/243-clear-browser-cache) after setting up CORS in your back end to test if it is working.**:

```python
# To prevent access to your API from other applications add the
# CORS_ALLOW_ORIGINS list and include only your front end app's
# URLs (localhost and deployed).  This list prevents a front end 
# from connecting to your back end unless it comes from a listed origin:

CORS_ALLOWED_ORIGINS = [
    "https://example.com",
    "http://localhost:3000",
    "http://127.0.0.1:3000"
]
```
OR

```python
CORS_ALLOW_ALL_ORIGINS = True
```



> The dj-database-url package parses a database URL to configure the connection string values (i.e., the database name, port, and root user and password) automatically. This allows us to configure our database connection string using the same format that Heroku uses when we create a Postgres database for our deployed app. Heroku automatically adds the DATABASE_URL to our config vars, which it reads as environment variables. The benefit for us is that we will be able to easily push to Heroku without having to make changes to our settings.py file. **Make sure to replace the entire `DATABASES` constant in your file:

```python
DATABASES = {
  'default': dj_database_url.config(conn_max_age=600)
}
```

> The whitenoise package we installed is going to help us out with serving static files for our app such as `css` or image files.  Even though you may be using Django REST Framework exclusively and didn't add any files of this type to the project yourself, the Django Admin site and Browseable API sites utilize static files like these extensively.  If you don't tell Django where to store these files, your app won't deploy properly, so add a *new* constant to the bottom of your `settings.py` file: 

```python
STATIC_ROOT=os.path.join(BASE_DIR, "static/")
```

7. Add your environment variables to the `.env` file.  You should already have copied over your `SECRET_KEY` from the `settings.py` file.

> Add a MODE variable. We want the DEBUG value to be True when we're working locally, so set the value to `dev` here:

```
MODE=dev
```

> Configure your database URL so that it is in the same format as the one that Heroku sets by adding a DATABASE_URL variable in the following format `DATABASE_URL=postgres://USER:PASSWORD@HOST:PORT/NAME`.  Replace the USER, PASSWORD and database NAME with your project specific values.  We'll use the default port for PostgreSQL on PORT 5432:

```
# DATABASE_URL=postgres://USER:PASSWORD@HOST:PORT/NAME
# Example: DATABASE_URL=postgres://petsuser:pets@localhost:5432/pets
```

8. Back in the Terminal, make sure **ALL** of your virtual environments are completely exited using the keyboard shortcut: <kbd>^</kbd> + <kbd>D</kbd> (control + D) or by typing `exit`.  This is important because the environment variables you created are only loaded when you initially launch the environment.  Once you've completely exited the shell, restart it with `pipenv shell` and test that your app is working locally (`python3 manage.py runserver`).  If it is, proceed to the next steps:


9. Run the Heroku commands to create your app.  Make sure you're in the directory where the `manage.py` file is in your project when you run these commands:

> Login:

```bash
heroku login
```

> Create an app:


```bash
heroku create
```

> Add Postgres

```bash
heroku addons:create heroku-postgresql:hobby-dev
```

> Set your config vars on Heroku. Remember that the DATABASE_URL is already generated in the last step, so you'll just need to add the other environment variables from your .env file. Also don't forget to set MODE to prod (or production, something that is NOT called dev) and replace your secret key with your actual key :slightly_smiling_face: :

```bash
heroku config:set MODE=prod SECRET_KEY='your secret key'
```

10. Run collectstatic in your virtual environment (this can take a few minutes the first time):

```bash
python3 manage.py collectstatic
```

11. Add, commit and push:

```bash
git add .
git commit -m "Your message"
git push heroku master
```

12. If your build is successful:

> Run your migrations on Heroku.

```bash
heroku run python manage.py migrate
```

> Create a superuser on your Heroku DB.

```bash
heroku run python manage.py createsuperuser

```



