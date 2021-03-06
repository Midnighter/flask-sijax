.. Flask-Sijax documentation master file, created by
   sphinx-quickstart on Sun Mar  6 02:01:07 2011.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

Flask-Sijax
===========

:Author: Slavi Pantaleev<s.pantaleev_REMOVE@ME_gmail.com>
:Version: |release|
:Source: github.com_
:Bug tracker: `github.com/issues <https://github.com/spantaleev/flask-sijax/issues>`_

Flask-Sijax helps you add Sijax_ support to your Flask_ applications.

Sijax is a Python/jQuery_ library that makes AJAX easy to use in web applications.

.. _github.com: https://github.com/spantaleev/flask-sijax
.. _Flask: http://flask.pocoo.org/
.. _jQuery: http://jquery.com/
.. _Sijax: http://pypi.python.org/pypi/Sijax

Installing Flask-Sijax
----------------------

Flask-Sijax is available on PyPI_ and can be installed using **easy_install**::

    easy_install flask-sijax

or using **pip**::

    pip install flask-sijax


.. _PyPI: http://pypi.python.org/pypi/Flask-Sijax/

Setting it up
-------------

Here's an example of how Flask-Sijax is typically initialized and configured::

    import os
    from flask import Flask, g
    import flask_sijax

    path = os.path.join('.', os.path.dirname(__file__), 'static/js/sijax/')

    app = Flask(__name__)
    app.config['SIJAX_STATIC_PATH'] = path
    app.config['SIJAX_JSON_URI'] = '/static/js/sijax/json2.js'
    flask_sijax.Sijax(app)


Configuration options
---------------------

**Flask-Sijax** is configured via the standard Flask config API.
Here are the available configuration options:

* **SIJAX_STATIC_PATH** - the static path where you want the Sijax javascript files to be mirrored.

Flask-Sijax takes care of keeping the Sijax javascript files ``sijax.js`` and ``json2.js``
up to date in this directory (even between version changes).

Don't put anything else in that directory - it should be dedicated to Sijax for hosting the files.


* **SIJAX_JSON_URI** - the URI to load the ``json2.js`` static file from (if needed).

Sijax uses JSON to pass data between the browser and server. This means that browsers either need to support
JSON natively or get JSON support from the ``json2.js`` file. Such browsers include IE <= 7.
If you've set a URI to ``json2.js`` and Sijax detects that the browser needs to load it, it will do so on demand.
The URI could be relative or absolute.


Making your Flask functions Sijax-aware
----------------------------------------------

Registering view functions with Flask is usually done using ``@app.route`` or ``@blueprint.route``.
Functions registered that way cannot provide Sijax functionality, because they cannot be accessed
using a POST method by default (and Sijax uses POST requests).

To make a view function capable of handling Sijax requests,
make it accessible via POST using ``@app.route('/url', methods=['GET', 'POST'])``
or use the ``@flask_sijax.route`` helper decorator like this::

    # Initialization code for Flask and Flask-Sijax
    # See above..

    # Functions registered with @app.route CANNOT use Sijax
    @app.route('/')
    def index():
        return 'Index'

    # Functions registered with @flask_sijax.route can use Sijax
    @flask_sijax.route(app, '/hello')
    def hello():
        # Every Sijax handler function (like this one) receives at least
        # one parameter automatically, much like Python passes `self`
        # to object methods.
        # The `obj_response` parameter is the function's way of talking
        # back to the browser
        def say_hi(obj_response):
            obj_response.alert('Hi there!')

        if g.sijax.is_sijax_request:
            # Sijax request detected - let Sijax handle it
            g.sijax.register_callback('say_hi', say_hi)
            return g.sijax.process_request()

        # Regular (non-Sijax request) - render the page template
        return _render_template()

Let's assume ``_render_template()`` renders the following page::

    <html>
    <head>
    <script type="text/javascript"
        src="{ URI to jQuery - not included with this project }"></script>
    <script type="text/javascript"
        src="/static/js/sijax/sijax.js"></script>
    <script type="text/javascript">
        {{ g.sijax.get_js()|safe }}
    </script>
    </head>
    <body>
        <a href="javascript://" onclick="Sijax.request('say_hi');">Click here</a>
    </body>
    </html>

Clicking on the link will fire a Sijax request (a special ``jQuery.ajax()`` request) to the server.

This request is detected on the server by ``g.sijax.is_sijax_request()``, in which case you let Sijax handle the request.

All functions registered using ``g.sijax.register_callback()`` (see :meth:`flask_sijax.Sijax.register_callback`) are exposed for calling from the browser.

Calling ``g.sijax.process_request()`` tells Sijax to execute the appropriate (previously registered) function and return the response to the browser.

To learn more on ``obj_response`` and what it provides, see :class:`sijax.response.BaseResponse`.


Setting up the client (browser)
-------------------------------

The browser needs to talk to the server and that's done using `jQuery`_ (``jQuery.ajax``)
and the Sijax javascript file (``sijax.js``).
You'll have to load those on each page that needs to use Sijax.

The ``sijax.js`` file is part of the `Sijax`_ project, but can be mirrored to a directory
of your choosing if you use the ``SIJAX_STATIC_PATH`` configuration option (see above).
There is no need to download Sijax separately and extract the file from it manually.

Once both files are loaded, you can put the javascript init code (``g.sijax.get_js()``) somewhere on the page.
That code is page-specific and needs to be executed after the ``sijax.js`` file has loaded.

Assuming you've used the above configuration here's the HTML markup you need to add to your template::

    <script type="text/javascript"
        src="{ URI to jQuery - not included with this project}"></script>
    <script type="text/javascript"
        src="/static/js/sijax/sijax.js"></script>
    <script type="text/javascript">
        {{ g.sijax.get_js()|safe }}
    </script>

You can then invoke a Sijax function using javascript like this::

    Sijax.request('function_name', ['argument 1', 150, 'argument 3']);

provided the function has been defined and registered with Sijax on the server-side::

    def function_name(obj_response, arg1, arg2, arg3):
        obj_response.alert('You called the function successfully!')

    g.sijax.register_callback('function_name', function_name)

To learn more on ``Sijax.request()`` see :ref:`Sijax:clientside-sijax-request`.

Learn more on how it all fits together from the **Examples**.

CSRF protection
---------------

Learn more about `cross-site request forgery <http://en.wikipedia.org/wiki/Cross-site_request_forgery>`_. In a nutshell, you want to ensure that you are indeed communicating with the same trusted user and not an imposter.

In the following, we will see how to implement basic CSRF protection for your
Sijax calls. The basic idea is that you send out a unique token when
communitcating with your unique, trusted user and expect the same token when that user communicates with your server.

There are a number of packages out there that can help you with token
generation and validation. You might be using some of them already so it would
be easy for you to implement this code.

* Flask-SeaSurf_
* Flask-WTF_
* Flask-Security_
* even Django_

.. _Flask-SeaSurf: https://flask-seasurf.readthedocs.org/en/latest/
.. _Flask-WTF: https://flask-wtf.readthedocs.org/en/latest/csrf.html
.. _Flask-Security: https://pythonhosted.org/Flask-Security/index.html
.. _Django: https://docs.djangoproject.com/en/1.7/ref/contrib/csrf/

In the following, we rely on some adapted code from a `Flask Snippet <http://flask.pocoo.org/snippets/3/>`_ and
`this gist <https://gist.github.com/danielrichman/5881046>`_. We do this as an
example here in order to not add more dependencies but it's probably better to
rely on the above packages to keep up with the development.

The first step is to generate a unique token. The following function illustrates
a simple way of doing this. Keep in mind that in order to use the session, the
Flask application needs to have its secret key set.::

    import os
    import hmac
    from hashlib import sha1
    from flask import session

    @app.template_global('csrf_token')
    def csrf_token():
        """
        Generate a token string from bytes arrays. The token in the session is user
        specific.
        """
        if "_csrf_token" not in session:
            session["_csrf_token"] = os.urandom(128)
        return hmac.new(app.secret_key, session["_csrf_token"],
                digestmod=sha1).hexdigest()



Due to the decorator you can now use this function in any of your templates.
There are at least four ways to embed the token in your page. Say you have a
Jinja2 template, then you either put it in a meta tag,::

    <meta name="csrf-token" content="{{ csrf_token() }}">

as part of a form into a hidden field,::

    <input type="hidden" name="csrf-token" value="{{ csrf_token() }}">

directly store it in a Javascript variable::

    <script type="text/javascript">
        var csrfToken = "{{ csrf_token() }}"
    </script>

or directly as part of your Sijax call::

    <a href="javascript://" onclick="Sijax.request('say_hello', ['John', 'Greg'], { data: { csrf_token: '{{ csrf_token() }}' } }">

In the first two cases, you will have to use Javascript to extract the token
from the HTML element.

Sijax merges any object of the `data` field of the third argument to
`Sijax.request` with the form data of the request. We can retrieve the token
from there and check its consistency. You can automatically do this for any
request to a view function using the following piece of code::

    @app.before_request
    def check_csrf_token():
        """Checks that token is correct, aborting if not"""
        if request.method in ("GET",): # not exhaustive list
            return
        token = request.form.get("csrf_token")
        if token is None:
            app.logger.warning("Expected CSRF Token: not present")
            abort(400)
        if not safe_str_cmp(token, csrf_token()):
            app.logger.warning("CSRF Token incorrect")
            abort(400)

This will check for the presence of a token for any request that is not `GET`
and compare the delivered with the known token.

Examples
--------

We have provided complete examples which you can run directly from the source distribution (if you've installed the dependencies - Flask_ and Sijax_ - both available on PyPI).

* **Hello** - a Hello world project

 - `Hello Code <https://github.com/spantaleev/flask-sijax/blob/master/examples/hello.py>`_
 - `Hello Template (jinja2) <https://github.com/spantaleev/flask-sijax/blob/master/examples/templates/hello.html>`_


* **Chat** - a simple Chat/Shoutbox

 - `Chat Code <https://github.com/spantaleev/flask-sijax/blob/master/examples/chat.py>`_
 - `Chat Template (jinja2) <https://github.com/spantaleev/flask-sijax/blob/master/examples/templates/chat.html>`_


* **Comet** - a demonstration of the :ref:`Comet Plugin <sijax:comet-plugin>`

 - `Comet Code <https://github.com/spantaleev/flask-sijax/blob/master/examples/comet.py>`_
 - `Comet Template (jinja2) <https://github.com/spantaleev/flask-sijax/blob/master/examples/templates/comet.html>`_


* **Upload** - a demonstration of the :ref:`Upload Plugin <sijax:upload-plugin>`

 - `Upload Code <https://github.com/spantaleev/flask-sijax/blob/master/examples/upload.py>`_
 - `Upload Template (jinja2) <https://github.com/spantaleev/flask-sijax/blob/master/examples/templates/upload.html>`_


Sijax documentation
-------------------

The `documentation for Sijax <http://packages.python.org/Sijax/>`_ is very exhaustive and even though you're using the Flask-Sijax extension, which hides some of the details for you, you can still learn a lot from it.

* :ref:`Using Sijax <sijax:usage>`
* :ref:`Comet Plugin <sijax:comet-plugin>`
* :ref:`Upload Plugin <sijax:upload-plugin>`
* :ref:`Sijax.request() <sijax:clientside-sijax-request>`
* :ref:`Sijax.getFormValues() <sijax:clientside-sijax-get-form-values>`
* :ref:`FAQ <sijax:faq>`


API
---

.. autofunction:: flask_sijax.route
.. autoclass:: flask_sijax.Sijax
   :members:

