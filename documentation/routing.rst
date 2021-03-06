.. _routing:

Routing and Url resolving
-----------------------------------------------
Crax Routing is simple and effective. You can describe your routes in several ways. for example

.. code-block:: python

    from crax.urls import Route, Url
    from my_app.handlers import BaseHandler

    urls = [Route(Url('/some/path'), handler=BaseHandler)]

The simplest example above shows how to create a route in Crax. First of all, we need to import two instances
from `crax.urls`. `Route` and` Url`.

**Route** Instance of `Route` takes two required arguments:
    **urls** - type of tuple of `Url` instances or single `Url` instance.

        .. code-block:: python

            from crax.urls import Route, Url
            from my_app.handlers import BaseHandler

            # This will work too
            urls = [Route(urls=(Url('/some/path'), Url('/another/path')), handler=BaseHandler)]

        In the example above, we told Crax that our BaseHandler should handle two different URLs.
        In case you have similar logic on two different uris, you can just serve it up with one handler.
        For example, you are going to write some kind of SPA (single page application), so you want to have routing in
        client (client) side. All you need is one index.html file generated by your JS framework and several
        static files. Let's do that.

        .. code-block:: python

            from crax.urls import Route, Url

            class APIView(TemplateView):
                template = "index.html"

            urls = [
                Route(
                    urls=(
                        Url("/"),
                        Url("/v1/customers"),
                        Url("/v1/discounts"),
                        Url("/v1/cart"),
                        Url("/v1/customer/<customer_id:int>"),
                        Url("/v1/discount/<discount_id:int>/<optional:str>/"),
                    ),
                    handler=APIView)
                ]

        We have done. No more code required to create SPA at the backend.

    **handler**
        Any callable that supports ASGI signature. Any handler you write according to documentation at :ref:`views` section can be passed as `handler` argument.

**Url** instance that defines uri and and it's parameters. Can take arguments:
    **path**:
        String type. Required argument describing uri. You should always start with a forward slash. The closing slash is not
        mandatory. How to write URLs (with or without the addition of a forward slash) is up to you. Crax will work any way.

    **name**:
        String type. An optional argument specifying the short name of the current uri. It must be unique. this is
        needed to generate URLs if you have, for example, the same paths in two applications or if some string parameters
        can be recognized by Crax as the same.



        .. code-block:: python

                from crax.urls import Route, Url

                Url(r"/cabinet/settings/")
                Url(r"/cabinet/(?P<username>\w{0,30})/", type="re_path")
                # It is different uris but user with username "settings" will never get cabinet
                # User with username "settings" will always receive the settings page instead of cabinet.

                # This will generate the correct URL only during template rendering
                # But the user still cannot access the account
                Url(r"/cabinet/settings/", name="cabinet_settings")
                Url(r"/cabinet/(?P<username>\w{0,30})/", name="cabinet", type="re_path")


        The predefined templated function url () looks at the name parameter first. If not, general resolution
        will be activated. The generic `Url` resolving works like this: url resolver iterates over your list of urls
        this is defined in your project settings. See: ref: `crax_settings` for details. Iteration starts from the first URL
        until the last, and the first match will be processed. So the first match for a user with username is
        "Settings" is "cabinet/settings/".

        .. code-block:: python

                from crax.urls import Route, Url

                # Better way
                Url(r"/user_cabinet/settings/")
                Url(r"/cabinet/(?P<username>\w{0,30})/", type="re_path")

        So where should we use the `name` argument. As mentioned above, this is for the url () template function.
        This function takes Url.name as the first argument, if it matches, the url will be processed. So if you
        have unique names and you are confident that your templates will always display the correct URLs.

    **scheme**:
        List or string type. Determines what type of request processing should be managed. The default is ["http", "http.request"].
        If you want to set the handler as a `websocket` handler, you must set this argument to "websocket".

    **type**:
        String type. `Url` can be one of two types:
            `path`:
                The default "path" value for the `path` argument. In this case, the Crax url resolver will treat the given uri as a simple path.

                .. code-block:: python

                    # Url defined as simple "path"
                    Url("/v1/customer/<customer_id:int>/<discount_name:str>/")

                Here we have defined a path that takes two parameters customer_id and Discount_name. So we move on to
                `http://some_host.org/v1/customer/10/Xmas/` we will get in our Request object
                parameters `{"customer_id": "10", "Discount_name": "Xmas"}`.

                .. code-block:: python

                    Url("/v1/customer/<customer_id:int>/<discount_name:str>/")

                    # Assuming that the Customer view is handler for the url above
                    # So Assertion is right
                    class CustomerView(BaseView):
                        methods = ['GET', 'POST']

                        async def get(self):
                            assert self.request.params == {"customer_id": "10", "discount_name": "Xmas"}

                As you can see, the values ​​have not been cast to specific types. It's not a mistake. Definitions of this type
                not for typecasting during URL resolution and is not actually required.

                .. code-block:: python

                    # It will work too
                    Url("/v1/customer/<customer_id>/<discount_name>/")

                So why did we define it. This is just for Swagger's online documentation. If you are not going
                to create an OpenAPI schema, you can choose not to define types. For everything about Swagger see :ref:`openapi`.

            `re_path`:
                This type of `Url` will process given path with regular expressions.

                .. code-block:: python

                    from crax.urls import Route, Url

                    Url(r"/cabinet/(?P<username>\w{0,30})/", type="re_path")

                If you want to describe some complex URLs, you can use the "re_path" type of `Url` type. For example, we want
                define a url that accepts a floating (optional) parameter. We want Crax to match this url if it contains optional
                parameter and it is not.

                .. code-block:: python

                    from crax.urls import Route, Url

                    Url(r"/cabinet/(?P<username>\w{0,30})/(?:(?P<optional>\w+))?", type="re_path")

                    class CustomerView(BaseView):
                        methods = ['GET', 'POST']

                        async def get(self):
                            assert self.request.params == {"customer_id": "10", "optional": "Optional value"}
                            # if optional is not in the url
                            assert self.request.params == {"customer_id": "10", "optional": None}

                Both cases will work fine. In the :ref:`intro` section, we checked if the optional parameter is None
                and, as a result, performed different actions.

                .. code-block:: python

                    # Here "department_id" is optional

                    async def post(self):
                        params = self.request.params
                        if params['department_id'] is not None:
                            await Department.query.insert(values=json.loads(self.request.post))
                            return TextResponse(self.request, 'Created')
                        else:
                            await Company.query.insert(values=json.loads(self.request.post))
                            return TextResponse(self.request, 'Created')

    **masquerade** Crax `Url` can be switched to the `Masquerade mode`. What does it mean.

        .. code-block:: python

            class MasqueradedView(TemplateView):
                template = "index.html"
                scope = ['template_one.html', 'template_two.html', 'template_three.html']

            Route(Url(r"/documentation/", masquerade=True), handler=MasqueradedView)

        Take a look at the above example. We only have one handler that we want to use in Masqueraded mode.
        Why? Maybe we have some templates like Sphinx documentation or some directory and we don't want to do anything
        but renders a bunch of templates. Of course we can create many urls and associate them with one handler,
        but we'll have a terrible list of urls and a lot of code. Thus, we can prevent this by using Masqueraded mode.
        We just need to tell Crax that this handler will handle all templates defined in its scope parameter.
        URLs at `http://some_host.org/documentation/template_?.html` will be successfully processed by one handler.

    **tag**:
        String type. An argument that binds the current URL to the scope with the given tag name in the OpenAPI schema.
        See :ref:`openapi` for details.

    **methods**:
        List type. Uppercase HTTP methods. These are not the same as handler's "methods" argument. The `methods` parameter
        of handlers tells Crax which methods this handler will serve. See: ref: `views` for details.
        The Url's `methods` argument, tells Swagger which methods to declare for this view. Details on
        :ref:`openapi` section.

    Good. We now have an idea of ​​how to work with Crax routing. But parameters could be passed not only through url parameters.
    We must be able to work with query parameters. And it's as simple as all of the above. You can transfer any number
    query parameters and access them in your handler.

    .. code-block:: python

        import requests
        requests.get('http://some_host.org/?param_1=1&param_2=2&param_3=3')
        class CompanyDetails(BaseView):

            async def get(self):
                assert self.request.query == {'param_1': ['1'], 'param_2': ['2'], 'param_3': ['3']}

    Thus, all parameters passed as URI parameters can be accessed via `request.params` and all parameters are specified
    since the query parameters are available via `request.query`. Please note that Crax does not currently support Basic Authentication.
    Thus, you cannot pass credentials as URI parameters.


.. toctree::
   :maxdepth: 3
   :caption: Contents:

.. index::
   Routing
