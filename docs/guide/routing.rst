.. _guide.routing:

URI routing
===========
`URI routing` is the process of taking the requested URI and deciding which
application handler will handle the current request. For this, we initialize
the :class:`WSGIApplication` defining a list of `routes`: each `route`
analyses the current request URI and, if it matches certain criterias,
returns the handler and optional variables extracted from the URI.

webapp2 offers a powerful and extensible system to match and build URIs,
which is explained in details in this section.


Simple routes
-------------
The simplest form of URI route in webapp2 is a tuple ``(regex, handler)``,
where `regex` is a regular expression to match the requested URI path and
`handler` is a :class:`webapp2.RequestHandler` to handle the request.
This routing mechanism is fully compatible with App Engine's webapp framework.

This is how it works: a list of routes is registered in the WSGI application.
When the application receives a request, it tries to match each one in order
until one matches, and then call the corresponding handler. Here, for example,
we define three handlers and register three routes that pointing to those
handlers::

    class HomeHandler(webapp2.RequestHandler):
        def get(self):
            self.response.write('This is the HomeHandler.')

    class ProductListHandler(webapp2.RequestHandler):
        def get(self):
            self.response.write('This is the ProductListHandler.')

    class ProductHandler(webapp2.RequestHandler):
        def get(self, product_id):
            self.response.write('This is the ProductHandler. '
                'The product id is %s' % product_id)

    app = webapp2.WSGIApplication([
        (r'/', HomeHandler),
        (r'/products', ProductListHandler),
        (r'/products/(\d+)', ProductHandler),
    ])

When a request comes in, the application will match the request path to find
the corresponding handler. If no route matches, an ``HTTPException`` is raised
with status code 404, and the WSGI application can handle it accordingly (see
:ref:`guide.exceptions`).

The `regex` part is an ordinary regular expression (see the :py:mod:`re`
module) that can define groups inside parentheses. The matched group values are
passed to the handler as positional arguments. In the example above, the last
route defines a group, so the handler will receive the matched value when the
route matches (one or more digits in this case).

The `handler` part is a callable as explained in :ref:`guide.handlers`.
One additional feature compared to webapp is that the handler can also be
defined as a string in dotted notation to be lazily imported when needed.
This is useful to avoid loading all modules when the app is initialized: we
can define handlers in different modules without needing to import all of them
to initialize the app. This is not only convenient but also speeds up the
application startup.

Our previous example could be rewritten using strings instead of handler
classes and splitting our handlers in two files, ``handlers.py`` and
``products.py``::

    app = webapp2.WSGIApplication([
        (r'/', 'handlers.HomeHandler'),
        (r'/products', 'products.ProductListHandler'),
        (r'/products/(\d+)', 'products.ProductHandler'),
    ])

When one of these routes matches, the handler will be imported by the routing
system if needed.

Simple routes are easy to use and enough for a lot of cases but don't support
keyword arguments, URI building, domain and subdomain matching, automatic
redirection and other useful features. For this, webapp2 offers the extended
routing mechanism that we'll see next.


Extended routes
---------------
webapp2 introduces a routing mechanism that extends the webapp model to provide
additional features:

- **URI building:** the registered routes can be built when needed, avoiding
  hardcoded URIs in the app code and templates. If you change the route
  definition in a compatible way during development, all places that use that
  route will continue to point to the correct URI. This is less error prone and
  easier to maintain.
- **Keyword arguments:** handlers can receive keyword arguments from the
  matched URIs. This is easier to use and also more maintanable than positional
  arguments.
- **Nested routes:** routes can be extended to match more than the request
  path. We will see below a route class that can also match domains and
  subdomains.

And several other features and benefits.

The concept is similar to the simple routes we saw before, but instead of a
tuple ``(regex, handler)``, we define each route using the class
:class:`webapp2.Route`. Let's remake our previous routes using it::

    app = webapp2.WSGIApplication([
        webapp2.Route(r'/', handler=HomeHandler, name='home'),
        webapp2.Route(r'/products', handler=ProductListHandler, name='product-list'),
        webapp2.Route(r'/products/<product_id:\d+>', handler=ProductHandler, name='product'),
    ])

The first argument in the routes above is a regex template, the second
argument is the handler to be used, and the third is a name used to build
a URI for that route. We already know what the handler is for, so let's
explain the other two.

Check :meth:`webapp2.Route.__init__` in the API reference for the other
parameters accepted by the ``Route`` constructor.

The regex template
~~~~~~~~~~~~~~~~~~
The regex template can have variables enclosed by ``<>`` that define a name, a
regular expression or both. Examples:

=================  ==================================
Format             Example
=================  ==================================
``<name>``         ``'/blog/<year>/<month>'``
``<:regex>``       ``'/blog/<:\d{4}>/<:\d{2}>'``
``<name:regex>``   ``'/blog/<year:\d{4}>/<month:\d{2}>'``
=================  ==================================

The same template can mix parts with name, regular expression or both.

If the name is set, the value of the matched regular expression is passed as
keyword argument to the handler. Otherwise it is passed as positional argument.

If only the name is set, it will match anything except a slash. So these
routes are equivalent::

    Route('/<user_id>/settings', handler=SettingsHandler, name='user-settings')
    Route('/<user_id:[^/]+>/settings', handler=SettingsHandler, name='user-settings')

.. note::
   The handler only receives ``*args`` if no named variables are
   set. Otherwise, the handler only receives ``**kwargs``. This
   allows you to set regular expressions that are not captured:
   just mix named and unnamed variables and the handler will
   only receive the named ones.

Building URIs
~~~~~~~~~~~~~
Because our routes now have a ``name``, we can use it to build URIs whenever
we need to reference those resources inside the application. This is done
using the method :meth:`webapp2.RequestHandler.uri_for` in a handler.

For example, if you have these routes defined for the application::

    app = webapp2.WSGIApplication([
        webapp2.Route('/', handler='handlers.HomeHandler', name='home'),
        webapp2.Route('/wiki', handler=WikiHandler, name='wiki'),
        webapp2.Route('/wiki/<page>', handler=WikiHandler, name='wiki-page'),
    ])

Here are some examples of how to generate URIs inside a handler::

    # /
    url = self.url_for('home')
    # http://localhost:8080/
    url = self.url_for('home', _full=True)
    # /wiki
    url = self.url_for('wiki')
    # http://localhost:8080/wiki
    url = self.url_for('wiki', _full=True)
    # http://localhost:8080/wiki#my-heading
    url = self.url_for('wiki', _full=True, _fragment='my-heading')
    # /wiki/my-first-page
    url = self.url_for('wiki-page', page='my-first-page')
    # /wiki/my-first-page?format=atom
    url = self.url_for('wiki-page', page='my-first-page', format='atom')

Check :meth:`webapp2.Router.build` in the API reference for a complete
explanation of the parameters used to build URIs.


Domain and subdomain routing
----------------------------
The routing system can also handle domain and subdomain matching. This is done
using a special route class provided in the ``webapp2_extras.routes`` module:
the ``DomainRoute``. This is a class that is initialized with a pattern to
match the current host name and a list of nested :class:`webapp2.Route` that
will only be tested if the domain or subdomain matches.

For example, to restrict routes to a subdomain of the appspot domain::

    app = webapp2.WSGIApplication([
        routes.DomainRoute('<subdomain>.app-id.appspot.com', [
            webapp2.Route('/', handler=SubdomainHomeHandler, name='subdomain-home'),
        ]),
        webapp2.Route('/', handler=HomeHandler, name='home'),
    ])

In the example above, we define a template ``'<subdomain>.app-id.appspot.com'``
for the domain matching. When a request comes in, only if the request server
name matches that pattern, the nested rules will be tested. Otherwise the
routing system will test the next rule until one matches. So the first route
``/foo`` will only match when a subdomain of the ``app-id.appspot.com`` domain
is accessed: when a subdomain is accessed and the path is ``/``, the handler
``SubdomainHomeHandler`` will be used, but when no subdomain is accessed (or
the domain is different) the ``HomeHandler`` will be used instead.

The template follows the same syntax used by :class:`webapp2.Route` and
must define named groups if any value must be added to the match results.
In the example above, an extra `subdomain` keyword is passed to the handler,
but if the regex didn't define any named groups, nothing would be added.

Matching only www, or anything except www
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
A common need it to set some routes for the main subdomain (``www``) and
different routes for other submains. The webapp2 routing system can handle
this easily.

To match only the ``www`` subdomain, simple set the domain template to a fixed
value::

    routes.DomainRoute('www.mydomain.com', [
        webapp2.Route('/', handler=HomeHandler, name='home'),
    ])

To match any subdomain expect the ``www`` subdomain, set a regular expression
that excludes ``www``::

    routes.DomainRoute(r'<subdomain:(?!www\.)[^.]+>.mydomain.com', [
        webapp2.Route('/', handler=HomeHandler, name='home'),
    ])

Any subdomain that matches and is not ``www`` will be passed as a parameter
``subdomain`` to the handler.

Similarly, you can restrict matches to the main ``appspot`` domain **or**
a ``www`` domain from a custom domain::

    routes.DomainRoute(r'<:app-id\.appspot\.com|www\.somedomain\.com>', [
        webapp2.Route('/', handler=HomeHandler, name='home'),
    ])

And then have a route that matches subdomains of the main ``appspot`` domain
**or** from a custom domain, except ``www``::

    routes.DomainRoute(r'<subdomain:(?!www\.)[^.]+>.<:app-id\.appspot\.com|mydomain\.com>', [
        webapp2.Route('/', handler=HomeHandler, name='home'),
    ])