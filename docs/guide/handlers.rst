.. _guide.handlers:

Request handlers
================
In the webapp2 vocabulary, `handler` is a common term that refers to the
callable that contains the application logic to handle a request. This sounds
a lot abstract, but we will explain everything in details in this section.


Handlers 101
------------
A handler is equivalent to the `Controller` in the
`MVC <http://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller>`_
terminology: in a simplified manner, it is where you process the request,
manipulate data and define a response to be returned to the client: HTML,
JSON, XML, files or whatever the app requires.

Normally a handler is a class that extends :class:`webapp2.RequestHandler`
or, for compatibility purposes, ``webapp.RequestHandler``. Here is a simple
one::

    class ProductHandler(webapp2.RequestHandler):
        def get(self, product_id):
            self.response.write('You requested product %r.' % product_id)

    app = webapp2.WSGIApplication([
        (r'/products/(\d+)', ProductHandler),
    ])

This code defines one request handler, ``ProductHandler``, and a WSGI
application that maps the URI ``r'/products/(\d+)'`` to that handler.
When the application receives an HTTP request to a path that matches this
regular expression, it instantiates the handler and calls the corresponding
HTTP method from it. The handler above can only handle ``GET`` HTTP requests,
as it only defines a ``get()`` method. To handle ``POST`` requests,
it would need to implement a ``post()`` method, and so on.

The handler method receives a ``product_id`` extracted from the URI, and
sets a simple message containing the id as response. Not very useful, but this
is just to show how it works. In a more complete example, the handler would
fetch a corresponding record from a database and set an appropriate response
-- HTML, JSON or XML with details about the requested product, for example.

For more details about how URI variables are defined, see :ref:`guide.routing`.


View functions
--------------
In some Python frameworks, handlers are called `view functions` or simply
`views`. In Django, for example, `views` are normally simple functions that
handle a request. Our examples use mostly classes, but webapp2 handlers can
also be normal functions equivalent to Django's `views`.

To be precise, a webapp2 handler can be any callable that receives
(:class:`webapp2.Request`, :class:`webapp2.Response`) as arguments. This
means that, differently from webapp, ordinary functions can also be used to
handle requests in webapp2, and not only classes. The following example
demonstrates it::

    def display_product(request, response):
        product_id = request.route_args[0]
        response.write('You requested product %r.' % product_id)

    app = webapp2.WSGIApplication([
        (r'/products/(\d+)', display_product),
    ])

Here, our handler is a simple function, and the positional route variables are
available in the request attribute ``.route_args``. Similarly, if the route
defined named variables, they would be available in ``.route_kwargs``.

Functions are an alternative for those that prefer their simplicity or think
that handlers don't benefit that much from the power and flexibility provided
by classes: inheritance, attributes, grouped methods, descriptors, metaclasses,
etc. An app can have mixed handler classes and functions.

.. note::
   We avoid using the term `view` because it is often confused with the `View`
   definition of the classic `MVC` pattern. Django prefers to call its `MVC`
   implementation `MTV` (model-template-view), so `view` may make sense in
   their terminology. Still, we think that the term can cause unnecessary
   confusion and prefer to use `handler` instead, as in webapp. In essence,
   though, they are synonyms.


Custom methods
--------------
TBD

The RequestHandler class
------------------------
TBD

Overriding ``__init__()``
~~~~~~~~~~~~~~~~~~~~~~~~~
If you want to override the :meth:`webapp2.RequestHandler.__init__` method,
you must call :meth:`webapp2.RequestHandler.initialize` at the beginning and
:meth:`webapp2.RequestHandler.dispatch` at the end of the method. This is
because in webapp2 the handler dispatches the requested method on construction.
If you call the parent class at the beginning of the method, any other
initializations you do after it will be lost because the requested method will
have been dispatched already.

Here's a correct way to override ``__init__()``::

    class MyHandler(webapp2.RequestHandler):
        def __init__(self, request, response):
            # Set self.request, self.response and self.app.
            self.initialize(request, response)

            # ... add your custom initializations here ...
            # ...

            # Dispatch the requested method.
            self.dispatch()


Overriding ``dispatch()``
~~~~~~~~~~~~~~~~~~~~~~~~~
One of the advantadges of webapp2 over webapp is that you can wrap the
dispatching process of :class:`webapp2.RequestHandler` to perform actions
before and/or after the requested method is dispatched. You can do this
overriding the :meth:`webapp2.RequestHandler.dispatch` method. This can be
useful, for example, to test if requirements were met before actually
dispatching the requested method, or to perform actions in the response object
after the method was dispatched. Here's an example::

    class MyHandler(webapp2.RequestHandler):
        def dispatch(self):
            # ... check if requirements were met ...
            # ...

            if requirements_were_met:
                # Parent class will call the method to be dispatched
                # -- get() or post() or etc.
                super(MyHandler, self).dispatch()
            else:
                self.abort(403)

In this case, if the requirements were not met, the method won't ever be
dispatched and a "403 Forbidden" response will be returned instead.

There are several possibilities to explore overriding ``dispatch()``, like
performing common checkings, setting common attributes or post-processing the
response.