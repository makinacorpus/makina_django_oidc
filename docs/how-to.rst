How-to guides
=============


Display custom message on login/logout
--------------------------------------

This library provides a hook system to call custom code. Hooks are configured on a provider by provider basis. In this
guide we will setup two hook function that add login/logout messages using `Django's message system <https://docs.djangoproject.com/en/stable/ref/contrib/messages/>`_.

First, if you don't already have a Python module holding OIDC related code in your projet, create a file named ``oidc.py`` next to your settings.

Add in those two functions :

.. code-block:: python

    from django.contrib import messages

    def login_function(request, user):
        messages.success(request, f"Welcome '{user.username}', you have been logged in")


    def logout_function(request):
        messages.success(
            request, f"{request.user.username}, you have been logged out successfully"
        )


Next, we plug those functions in the library configuration. In your ``settings.py`` you should set the :ref:`LOGIN_FUNCTION` and :ref:`LOGOUT_FUNCTION` to point to those two functions.

If you used a provider, the best way to achieve that is by modifying the configuration value as such :

.. code-block:: python

    MAKINA_DJANGO_OIDC = {
        **my_project_provider.get_config(allowed_hosts=["myhost"]),
    }

    MAKINA_DJANGO_OIDC[my_project_provider.op_name]["LOGIN_FUNCTION"] = "<my_app>.oidc:login_function" # <- my_app is a placeholder, change it to your root module
    MAKINA_DJANGO_OIDC[my_project_provider.op_name]["LOGOUT_FUNCTION"] = "<my_app>.oidc:logout_function" # <- my_app is a placeholder, change it to your root module


If you configured your settings manually, juste add the LOGIN/LOGOUT function keys to your configuration. See :ref:`Hook settings` for more information on the function path syntax.

You should now see a message on login/logout ! 🎉

If not, make sure that you modified your template to display messages. See :func:`django:django.contrib.messages.get_messages` for more information.


Customize how token data is mapped to User attributes
-----------------------------------------------------

By default, this library only uses the email field in a userinfo token to retrieve/create users.

However you can implement more complex behaviour by specifying a :ref:`USER_FUNCTION` in your provider configuration. In this guide we will look at the ``groups`` attribute in a userinfo token and set the :attr:`is_staff <django.contrib.auth.models.User.is_staff>` attribute depending on the value.

First, if you don't already have a Python module holding OIDC related code in your projet, create a file named ``oidc.py`` next to your settings.

Add in a function that takes a two arguments : the *userinfo token* and the *id token*. It should return a :class:`User <django.contrib.auth.models.User>` instance.
We will start from this library default get_user function :

.. code-block:: python

    from django.contrib.auth import get_user_model

    def get_user(userinfo_token, id_token):
        User = get_user_model()
        user, created = User.objects.get_or_create(email=userinfo_token["email"])
        user.backend = "django.contrib.auth.backends.ModelBackend"
        return user

You can also print the *userinfo token* here. If you use Keycloak, you could have something like this :

.. code-block:: json

    {
      "sub": "40861311-0c53-4ad9-bc5c-d5fee81b0503",
      "email_verified": true,
      "name": "Admin User",
      "groups": [
        "basic-users",
        "default-roles-demo",
        "admins"
      ],
      "preferred_username": "admin",
      "given_name": "Admin",
      "family_name": "User",
      "email": "admin@example.com"
    }

We can see that we want to lookup the ``groups`` key and test if ``admins`` is in the list.

.. code-block:: python

    from django.contrib.auth import get_user_model

    def get_user(userinfo_token, id_token):
        User = get_user_model()
        user, created = User.objects.get_or_create(email=userinfo_token["email"])

        user.is_superuser = "admins" in userinfo_token["groups"]

        user.backend = "django.contrib.auth.backends.ModelBackend"
        user.save()
        return user


To have this function called instead of the default one, you need to modify your settings so that :ref:`USER_FUNCTION` points to the function that we just wrote.

The value of this setting should be : ``<my_app>.oidc:login_function`` (see :ref:`Hook settings` for more information on this syntax).

If you configured your settings manually (without using the providers system), you can add the key directly.

Using a provider, edith your configuration like this :

.. code-block:: python

    MAKINA_DJANGO_OIDC = {
        **my_project_provider.get_config(allowed_hosts=["myhost"]),
    }

    MAKINA_DJANGO_OIDC[my_project_provider.op_name]["USER_FUNCTION"] = "<my_app>.oidc:get_user" # <- my_app is a placeholder, change it to your root module



Add application-wide access control rules based on audiences
------------------------------------------------------------

Open ID Connect supports a system of audience which can be used to indicate the list of applications a user has access to.

In order to implement access control based on the audience, you need to hook the :ref:`USER_FUNCTION` to add your own logic.

In this guide, we will start from what we did in :ref:`Customize how token data is mapped to User attributes` and add audience based access control.

By the specification, the audience in a token is a list of string, so let's just check that our client id is in this list. Since we already defined our client ID in the settings, we fetch it from there ! This example assumes that your provider is named `keycloak`.

.. code-block:: python

    from django.contrib.auth import get_user_model
    from django.core.exceptions import PermissionDenied
    from django.conf import settings

    def get_user(userinfo_token, id_token):

        audiences = id_token["aud"]

        # Perform audience check
        if settings.MAKINA_DJANGO_OIDC["keycloak"]["CLIENT_ID"] not in audiences:
            raise PermissionDenied("You do not have access to this application")

        User = get_user_model()
        user, created = User.objects.get_or_create(email=userinfo_token["email"])
        user.is_superuser = "admins" in userinfo_token["groups"]
        user.backend = "django.contrib.auth.backends.ModelBackend"
        user.save()

        return user


Use the Django permission system with OIDC
------------------------------------------

Django provides a rich authentication system that handles groups and permissions.

In this guide we will map Keycloak groups to Django groups. This allows one to manage group level permissions using Django system, will keeping all the advantages of an Identity Provider to manage a user base.


In order to add user to groups on login, you need to hook the :ref:`USER_FUNCTION`.

We will start from what we did in :ref:`Customize how token data is mapped to User attributes` and add audience based access control.

In the *userinfo token* we can expect to find a 'groups' key (if available) and use it to query Django Groups models.

Here is how to do it :

.. code-block:: python

    from django.contrib.auth import get_user_model

    def get_user(userinfo_token, id_token):


        User = get_user_model()
        user, created = User.objects.get_or_create(email=userinfo_token["email"])

        if "groups" in userinfo_token:
            for group_name in userinfo_token["groups"]:
                group, _ = Group.objects.get_or_create(name=group_name)
                group.user_set.add(user)
                group.save()

        user.is_superuser = "admins" in userinfo_token["groups"]
        user.backend = "django.contrib.auth.backends.ModelBackend"
        user.save()

        return user

And that's it. Groups will be created on the fly as your users connect to your application. Then, you can grant group level permissions and it will be applied to your users.

.. note::
    For the sake of simplicity, in this tutorial users are only added to groups. However you might also want to remove user from groups depending on your use case.

Redirect the user after login
------------------------------

To redirect the user after login, you can provide the http query parameter ``next`` with an URI.
If the user login is sucessful it then will be redirected to this URI.

Here is an example of a login button redirecting the user to the page named "profile" :

.. code-block:: python

    import urllib

    from django.urls import reverse
    from django.views import View

    class RedirectDemo(View):
        http_method_names = ["get"]

        def get(self):
            # From : https://realpython.com/django-redirects/#passing-parameters-with-redirects
            base_url = reverse("keycloak_1-login")
            query_string = urllib.parse.urlencode({"next": reverse("profile")})
            return redirect(f"{base_url}?{query_string}")

However you will need to tweak the settings according to your use-case. You should take a look at  :ref:`REDIRECT_REQUIRES_HTTPS` and :ref:`REDIRECT_ALLOWED_HOSTS`.

Use multiple identity providers
-------------------------------

This library natively supports multiples identity providers.

You already have to specify a provider name when you configure your settings (either automatically by using a provider, or :ref:`manually <Providers settings>`).

In a multi-provider setup, the settings look like this :

.. code-block:: python

    MAKINA_DJANGO_OIDC = {
        'provider_name_1' : {
            'CLIENT_ID' : '' # <- provider 1 settings here
        }
        'provider_name_2' : {
            'CLIENT_ID' : '' # <- provider 2 settings here
        }
     }

If you are using our premade providers configuration, your ``settings.py`` will look like this :


.. code-block:: python

    from .oidc_providers import provider_1, provider_2

    MAKINA_DJANGO_OIDC = {
        **provider_1.get_config(allowed_hosts=["app.local:8082"]),
        **provider_2.get_config(allowed_hosts=["app.local:8082"]),
     }

Then you have to include all your provider url configuration in your ``urlpatterns``. Since view names includes the identity provider name, they should not collide.

Here is an example of such a configuration :

.. code-block:: python
    :caption: urls.py

    from .oidc import my_project_provider

    urlpatterns = [
        path("auth", include(provider_1.get_urlpatterns())),
        path("auth", include(provider_2.get_urlpatterns())),
    ]


You can then use those view names to redirect a user to one or the other provider.

Since settings are local to a provider, you can also provider different :ref:`USER_FUNCTION` to implement custom behaviour based on which identity provider a user is coming from.
