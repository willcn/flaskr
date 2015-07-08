# Flaskr 2.0 Tutorial

*This tutorial had been modified for my personal usage. I want to teach some
people Flask, but they were running into problems setting up the development
environment as well as getting stuck on the SQL portions. So I forked the
tutorial and modified it for their benefit. Hope it helps.*

You want to develop an application with Python and Flask?  Here you have
the chance to learn by example.  In this tutorial, we will create a simple
microblogging application.  It only supports one user that can create
text-only entries and there are no feeds or comments, but it still
features everything you need to get started.  We will use Flask and Redis
as a database (which is included on Cloud9) so there is nothing else you need.

If you want the full source code in advance or for comparison, check out
the [example source](https://github.com/willfongqq/flaskr/).


## Introducing Flaskr

We will call our blogging application flaskr, but feel free to choose your own
less Web-2.0-ish name ;)  Essentially, we want it to do the following things:

1. Let the user sign in and out with credentials specified in the
   configuration.  Only one user is supported.
2. When the user is logged in, they can add new entries to the page
   consisting of a text-only title and some HTML for the text.  This HTML
   is not sanitized because we trust the user here.
3. The index page shows all entries so far in reverse chronological order
   (newest on top) and the user can add new ones from there if logged in.

We will be using Redis directly for this application because it's good
enough for an application of this size.  For larger applications, however,
it makes a lot of sense to use [SQLAlchemy](http://www.sqlalchemy.org/), as it handles database
connections in a more intelligent way, allowing you to target different
relational databases at once and more.  You might also want to consider
one of the popular NoSQL databases if your data is more suited for those.


## Step 0: Creating The Folders

Before we get started, let's create the folders needed for this application:

    /flaskr
      /static
      /templates

The flaskr folder is not a Python package, but just something where we drop 
our files. The files inside the `static` folder are available to users of the 
application via HTTP. This is the place where CSS and Javascript files go. 
Inside the `templates` folder, Flask will look for Jinja2 templates. The 
templates you create later on in the tutorial will go in this directory.


## Step 1: About The Redis Database

Redis is a key-value database that is pre-installed in the Cloud9 development 
environment. Redis can integrate transparently with Python, providing a 
permanent data store for Python lists and dicts. For the purpose of this Flask
tutorial, it's not necessary to learn how it works in complete detail. You can 
read more about it on their official [website](http://redis.io/).

In order to use Redis for our Flask application, you need to run 
`sudo service redis-server start` in the console. You should see something 
like this:

    willfongqq@flaskr:~/workspace (master) $ sudo service redis-server start
    Starting redis-server: redis-server.
    willfongqq@flaskr:~/workspace (master) $ 

The rest of this is a quick overview of Redis. 

** You can skip the rest of this step. **

But I wrote up this section for anyone who wants to know a little more about it.

----

Redis records are stored as a key / value pair:

    <key> | <value>
    
The keys and values can be almost any type of string you want. For example:

    username | willfongqq
    
The key is `username` and the value stored in that record is `willfongqq`. 

When you need a record from the database, you provide the key, `username`, 
and Redis will provide the value associated with it, `willfongqq`. 

To access Redis, you need to run `redis-cli`:

    willfongqq@flaskr:~/workspace (master) $ redis-cli 
    127.0.0.1:6379> 

To store a key, you run `SET <key> <value>`:

    127.0.0.1:6379> SET username willfongqq
    OK

To get the value of the key, you run: `GET <key>`:

    127.0.0.1:6379> GET username
    "willfongqq"

For this Flask tutorial, we're going to store a list. Redis has support for 
this as well. We use `RPUSH` to *push* a value to the list. Let's create a 
list of technology companies:

    127.0.0.1:6379> RPUSH companies apple
    (integer) 1
    127.0.0.1:6379> RPUSH companies google
    (integer) 2
    127.0.0.1:6379> RPUSH companies microsoft
    (integer) 3

You can see that each time we add a company, we get a number back of how many
records there are in the list. 

We can retrieve the list with the `LRANGE` command. For `LRANGE`, you will need
to specify the name of the list as well as the starting indexing and ending
index:

    LRANGE <list name> <starting index> <ending index>

To get all the companies, you would run a query like this:

    127.0.0.1:6379> LRANGE companies 0 -1
    1) "apple"
    2) "google"
    3) "microsoft"

The `0` and `-1` are going to be a little confusing, so let's explain that a 
bit. 

The index of a entry is the *position* of the value in the list. This is going
to be a little confusing at first, but in Redis, the first entry has an index 
of 0. For example, `apple` is the first entry in our list, so it has an index
of `0`. `google` is the second entry in the list, so it has an index of `1`.
So, to get the first item `apple`, we would use `0` and `0` for the index
positions:
    
    127.0.0.1:6379> LRANGE companies 0 0
    1) "apple"

And for the third item, `microsoft`, we would use `2` and `2`:

    127.0.0.1:6379> LRANGE companies 2 2
    1) "microsoft"

A negative number would tell Redis to count from the end of the list. In our
original example, we had `-1`, which meant "stop at the last entry of the 
list". A value of `-2` would mean "stop at the second to the last entry of the
list", in this case, would have been `google`. 

Confusing? I'm sorry about that...


## Step 2: Application Setup Code

The first file will be the application module. Let's call it `flaskr.py`. 
We will place this file inside the `flaskr` folder. We will begin by adding 
the imports we need and by adding the config
section.  For small applications, it is possible to drop the configuration
directly into the module, and this is what we will be doing here. However,
a cleaner solution would be to create a separate `.ini` or `.py` file,
load that, and import the values from there.

First, we add the imports in :file:`flaskr.py`::

    # all the imports
    import os
    from redis_collections import List
    from flask import Flask, request, session, g, redirect, url_for, abort, \
         render_template, flash

Next, we can create our actual application and initialize it with the
config from the same file in :file:`flaskr.py`::

    # create our little application :)
    app = Flask(__name__)
    app.config.from_object(__name__)

    # Load default config and override config from an environment variable
    app.config.update(dict(
        DATABASE=os.path.join(app.root_path, 'flaskr.db'),
        SECRET_KEY='development key',
        USERNAME='admin',
        PASSWORD='default'
    ))
    app.config.from_envvar('FLASKR_SETTINGS', silent=True)

The :class:`~flask.Config` object works similarly to a dictionary so we
can update it with new values.

.. admonition:: Database Path

    Operating systems know the concept of a current working directory for
    each process.  Unfortunately, you cannot depend on this in web
    applications because you might have more than one application in the
    same process.

    For this reason the ``app.root_path`` attribute can be used to
    get the path to the application.  Together with the ``os.path`` module,
    files can then easily be found.  In this example, we place the
    database right next to it.

    For a real-world application, it's recommended to use
    :ref:`instance-folders` instead.

Usually, it is a good idea to load a separate, environment-specific
configuration file.  Flask allows you to import multiple configurations and it
will use the setting defined in the last import. This enables robust
configuration setups.  :meth:`~flask.Config.from_envvar` can help achieve this.

.. code-block:: python

   app.config.from_envvar('FLASKR_SETTINGS', silent=True)

Simply define the environment variable :envvar:`FLASKR_SETTINGS` that points to
a config file to be loaded.  The silent switch just tells Flask to not complain
if no such environment key is set.

In addition to that, you can use the :meth:`~flask.Config.from_object`
method on the config object and provide it with an import name of a
module.  Flask will then initialize the variable from that module.  Note
that in all cases, only variable names that are uppercase are considered.

The ``SECRET_KEY`` is needed to keep the client-side sessions secure.
Choose that key wisely and as hard to guess and complex as possible.

We will also add a method that allows for easy connections to the
specified database. This can be used to open a connection on request and
also from the interactive Python shell or a script.  This will come in
handy later.  We create a simple database connection through SQLite and
then tell it to use the :class:`sqlite3.Row` object to represent rows.
This allows us to treat the rows as if they were dictionaries instead of
tuples.

::

    def connect_db():
        """Connects to the specific database."""
        rv = sqlite3.connect(app.config['DATABASE'])
        rv.row_factory = sqlite3.Row
        return rv

With that out of the way, you should be able to start up the application
without problems.  Do this with the following command::

    flask --app=flaskr --debug run

The :option:`--debug` flag enables or disables the interactive debugger.  *Never
leave debug mode activated in a production system*, because it will allow
users to execute code on the server!

You will see a message telling you that server has started along with
the address at which you can access it.

When you head over to the server in your browser, you will get a 404 error
because we don't have any views yet.  We will focus on that a little later,
but first, we should get the database working.

    Want your server to be publicly available?  Check out the
    :ref:`externally visible server <public-server>` section for more
    information.


## Step 3: Database Connections

We have created a function for establishing a database connection with
`connect_db`, but by itself, that's not particularly useful.  Creating and
closing database connections all the time is very inefficient, so we want
to keep it around for longer.  Because database connections encapsulate a
transaction, we also need to make sure that only one request at the time
uses the connection. How can we elegantly do that with Flask?

This is where the application context comes into play, so let's start
there.

Flask provides us with two contexts: the application context and the
request context.  For the time being, all you have to know is that there
are special variables that use these.  For instance, the
:data:`~flask.request` variable is the request object associated with
the current request, whereas :data:`~flask.g` is a general purpose
variable associated with the current application context.  We will go into
the details of this a bit later.

For the time being, all you have to know is that you can store information
safely on the :data:`~flask.g` object.

So when do you put it on there?  To do that you can make a helper
function.  The first time the function is called, it will create a database
connection for the current context, and successive calls will return the
already established connection::

    def get_db():
        """Opens a new database connection if there is none yet for the
        current application context.
        """
        if not hasattr(g, 'sqlite_db'):
            g.sqlite_db = connect_db()
        return g.sqlite_db


So now we know how to connect, but how do we properly disconnect?  For
that, Flask provides us with the :meth:`~flask.Flask.teardown_appcontext`
decorator.  It's executed every time the application context tears down::

    @app.teardown_appcontext
    def close_db(error):
        """Closes the database again at the end of the request."""
        if hasattr(g, 'sqlite_db'):
            g.sqlite_db.close()

Functions marked with :meth:`~flask.Flask.teardown_appcontext` are called
every time the app context tears down.  What does this mean?
Essentially, the app context is created before the request comes in and is
destroyed (torn down) whenever the request finishes.  A teardown can
happen because of two reasons: either everything went well (the error
parameter will be ``None``) or an exception happened, in which case the error
is passed to the teardown function.

Curious about what these contexts mean?  Have a look at the
:ref:`app-context` documentation to learn more.

If you've been following along in this tutorial, you might be wondering
where to put the code from this step and the next.  A logical place is to
group these module-level functions together, and put your new
``get_db`` and ``close_db`` functions below your existing
``connect_db`` function (following the tutorial line-by-line).

If you need a moment to find your bearings, take a look at how the [example
source](https://github.com/willfongqq/flaskr) is organized.  In Flask, you 
can put all of your application code
into a single Python module.  You don't have to, and if your app :ref:`grows
larger <larger-applications>`, it's a good idea not to.


## Step 4: Creating The Database

As outlined earlier, Flaskr is a database powered application, and more
precisely, it is an application powered by a relational database system.  Such
systems need a schema that tells them how to store that information.
Before starting the server for the first time, it's important to create
that schema.

Such a schema can be created by piping the ``schema.sql`` file into the
`sqlite3` command as follows::

    sqlite3 /tmp/flaskr.db < schema.sql

The downside of this is that it requires the ``sqlite3`` command to be
installed, which is not necessarily the case on every system.  This also
requires that we provide the path to the database, which can introduce
errors.  It's a good idea to add a function that initializes the database
for you to the application.

To do this, we can create a function and hook it into the :command:`flask`
command that initializes the database.  Let me show you the code first.  Just
add this function below the `connect_db` function in :file:`flaskr.py`::

    def init_db():
        db = get_db()
        with app.open_resource('schema.sql', mode='r') as f:
            db.cursor().executescript(f.read())
        db.commit()

    @app.cli.command('initdb')
    def initdb_command():
        """Initializes the database."""
        init_db()
        print 'Initialized the database.'

The ``app.cli.command()`` decorator registers a new command with the
:command:`flask` script.  When the command executes, Flask will automatically
create a application context for us bound to the right application.
Within the function, we can then access :attr:`flask.g` and other things as
we would expect.  When the script ends, the application context tears down
and the database connection is released.

We want to keep an actual functions around that initializes the database,
though, so that we can easily create databases in unit tests later on.  (For
more information see :ref:`testing`.)

The :func:`~flask.Flask.open_resource` method of the application object
is a convenient helper function that will open a resource that the
application provides.  This function opens a file from the resource
location (your ``flaskr`` folder) and allows you to read from it.  We are
using this here to execute a script on the database connection.

The connection object provided by SQLite can give us a cursor object.
On that cursor, there is a method to execute a complete script.  Finally, we
only have to commit the changes.  SQLite3 and other transactional
databases will not commit unless you explicitly tell it to.

Now, it is possible to create a database with the :command:`flask` script::

    flask --app=flaskr initdb
    Initialized the database.

If you get an exception later on stating that a table cannot be found, check
that you did execute the ``initdb`` command and that your table names are
correct (singular vs. plural, for example).


## Step 5: The View Functions

Now that the database connections are working, we can start writing the
view functions.  We will need four of them:

### Show Entries

This view shows all the entries stored in the database.  It listens on the
root of the application and will select title and text from the database.
The one with the highest id (the newest entry) will be on top.  The rows
returned from the cursor look a bit like tuples because we are using
the :class:`sqlite3.Row` row factory.

The view function will pass the entries as dictionaries to the
:file:`show_entries.html` template and return the rendered one::

    @app.route('/')
    def show_entries():
        db = get_db()
        cur = db.execute('select title, text from entries order by id desc')
        entries = cur.fetchall()
        return render_template('show_entries.html', entries=entries)

### Add New Entry

This view lets the user add new entries if they are logged in.  This only
responds to ``POST`` requests; the actual form is shown on the
`show_entries` page.  If everything worked out well, we will
:func:`~flask.flash` an information message to the next request and
redirect back to the `show_entries` page::

    @app.route('/add', methods=['POST'])
    def add_entry():
        if not session.get('logged_in'):
            abort(401)
        db = get_db()
        db.execute('insert into entries (title, text) values (?, ?)',
                     [request.form['title'], request.form['text']])
        db.commit()
        flash('New entry was successfully posted')
        return redirect(url_for('show_entries'))

Note that we check that the user is logged in here (the `logged_in` key is
present in the session and ``True``).


### Login and Logout

These functions are used to sign the user in and out.  Login checks the
username and password against the ones from the configuration and sets the
`logged_in` key for the session.  If the user logged in successfully, that
key is set to ``True``, and the user is redirected back to the `show_entries`
page.  In addition, a message is flashed that informs the user that he or
she was logged in successfully.  If an error occurred, the template is
notified about that, and the user is asked again::

    @app.route('/login', methods=['GET', 'POST'])
    def login():
        error = None
        if request.method == 'POST':
            if request.form['username'] != app.config['USERNAME']:
                error = 'Invalid username'
            elif request.form['password'] != app.config['PASSWORD']:
                error = 'Invalid password'
            else:
                session['logged_in'] = True
                flash('You were logged in')
                return redirect(url_for('show_entries'))
        return render_template('login.html', error=error)

The `logout` function, on the other hand, removes that key from the session
again.  We use a neat trick here: if you use the :meth:`~dict.pop` method
of the dict and pass a second parameter to it (the default), the method
will delete the key from the dictionary if present or do nothing when that
key is not in there.  This is helpful because now we don't have to check
if the user was logged in.

    @app.route('/logout')
    def logout():
        session.pop('logged_in', None)
        flash('You were logged out')
        return redirect(url_for('show_entries'))


## Step 6: The Templates

Now we should start working on the templates.  If we were to request the URLs
now, we would only get an exception that Flask cannot find the templates.  The
templates are using `Jinja2`_ syntax and have autoescaping enabled by
default.  This means that unless you mark a value in the code with
:class:`~flask.Markup` or with the ``|safe`` filter in the template,
Jinja2 will ensure that special characters such as ``<`` or ``>`` are
escaped with their XML equivalents.

We are also using template inheritance which makes it possible to reuse
the layout of the website in all pages.

Put the following templates into the :file:`templates` folder:

.. _Jinja2: http://jinja.pocoo.org/docs/templates

### layout.html

This template contains the HTML skeleton, the header and a link to log in
(or log out if the user was already logged in).  It also displays the
flashed messages if there are any.  The ``{% block body %}`` block can be
replaced by a block of the same name (``body``) in a child template.

The :class:`~flask.session` dict is available in the template as well and
you can use that to check if the user is logged in or not.  Note that in
Jinja you can access missing attributes and items of objects / dicts which
makes the following code work, even if there is no ``'logged_in'`` key in
the session:

    <!doctype html>
    <title>Flaskr</title>
    <link rel=stylesheet type=text/css href="{{ url_for('static', filename='style.css') }}">
    <div class=page>
      <h1>Flaskr</h1>
      <div class=metanav>
      {% if not session.logged_in %}
        <a href="{{ url_for('login') }}">log in</a>
      {% else %}
        <a href="{{ url_for('logout') }}">log out</a>
      {% endif %}
      </div>
      {% for message in get_flashed_messages() %}
        <div class=flash>{{ message }}</div>
      {% endfor %}
      {% block body %}{% endblock %}
    </div>

### show_entries.html

This template extends the :file:`layout.html` template from above to display the
messages.  Note that the ``for`` loop iterates over the messages we passed
in with the :func:`~flask.render_template` function.  We also tell the
form to submit to your `add_entry` function and use ``POST`` as HTTP
method:

    {% extends "layout.html" %}
    {% block body %}
      {% if session.logged_in %}
        <form action="{{ url_for('add_entry') }}" method=post class=add-entry>
          <dl>
            <dt>Title:
            <dd><input type=text size=30 name=title>
            <dt>Text:
            <dd><textarea name=text rows=5 cols=40></textarea>
            <dd><input type=submit value=Share>
          </dl>
        </form>
      {% endif %}
      <ul class=entries>
      {% for entry in entries %}
        <li><h2>{{ entry.title }}</h2>{{ entry.text|safe }}
      {% else %}
        <li><em>Unbelievable.  No entries here so far</em>
      {% endfor %}
      </ul>
    {% endblock %}

### login.html

This is the login template, which basically just displays a form to allow
the user to login:

    {% extends "layout.html" %}
    {% block body %}
      <h2>Login</h2>
      {% if error %}<p class=error><strong>Error:</strong> {{ error }}{% endif %}
      <form action="{{ url_for('login') }}" method=post>
        <dl>
          <dt>Username:
          <dd><input type=text name=username>
          <dt>Password:
          <dd><input type=password name=password>
          <dd><input type=submit value=Login>
        </dl>
      </form>
    {% endblock %}


## Step 7: Adding Style

Now that everything else works, it's time to add some style to the
application.  Just create a stylesheet called :file:`style.css` in the
:file:`static` folder we created before:

    body            { font-family: sans-serif; background: #eee; }
    a, h1, h2       { color: #377ba8; }
    h1, h2          { font-family: 'Georgia', serif; margin: 0; }
    h1              { border-bottom: 2px solid #eee; }
    h2              { font-size: 1.2em; }

    .page           { margin: 2em auto; width: 35em; border: 5px solid #ccc;
                      padding: 0.8em; background: white; }
    .entries        { list-style: none; margin: 0; padding: 0; }
    .entries li     { margin: 0.8em 1.2em; }
    .entries li h2  { margin-left: -1em; }
    .add-entry      { font-size: 0.9em; border-bottom: 1px solid #ccc; }
    .add-entry dl   { font-weight: bold; }
    .metanav        { text-align: right; font-size: 0.8em; padding: 0.3em;
                      margin-bottom: 1em; background: #fafafa; }
    .flash          { background: #cee5F5; padding: 0.5em;
                      border: 1px solid #aacbe2; }
    .error          { background: #f0d6d6; padding: 0.5em; }


## Bonus: Testing the Application

Now that you have finished the application and everything works as
expected, it's probably not a bad idea to add automated tests to simplify
modifications in the future.  The application above is used as a basic
example of how to perform unit testing in the :ref:`testing` section of the
documentation.  Go there to see how easy it is to test Flask applications.
