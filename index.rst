===========
Flask-Login
===========
.. currentmodule:: flask.ext.login

Flask-Login 为 Flask 提供了用户会话管理。它处理了日常的登入，登出并且长时间记住用户的会话。

它会:

- 在会话中存储当前活跃的用户 ID，让你能够自由地登入和登出。
- 让你限制登入(或者登出)用户可以访问的视图。
- 处理让人棘手的 “记住我” 功能。
- 帮助你保护用户会话免遭 cookie 被盗的牵连。
- 可以与以后可能使用的 Flask-Principal 或其它认证扩展集成。

但是，它不会:

- 限制你使用特定的数据库或其它存储方法。如何加载用户完全由你决定。
- 限制你使用用户名和密码，OpenIDs，或者其它的认证方法。
- 处理超越 “登入或者登出” 之外的权限。
- 处理用户注册或者账号恢复。

.. contents::
   :local:
   :backlinks: none


配置你的应用
===============
对一个使用 Flask-Login 的应用最重要的一部分就是 `LoginManager` 类。你应该在你的代码的某处为应用创建一个，像这样::

    login_manager = LoginManager()

登录管理(login manager)包含了让你的应用和 Flask-Login 协同工作的代码，比如怎样从一个 ID 加载用户，当用户需要登录的时候跳转到哪里等等。

一旦实际的应用对象创建后，你能够这样配置它来实现登录::

    login_manager.init_app(app)


它是如何工作
=============
你必须提供一个 `~LoginManager.user_loader` 回调。这个回调用于从会话中存储的用户 ID 重新加载用户对象。它应该接受一个用户的 `unicode` ID 作为参数，并且返回相应的用户对象。比如::

    @login_manager.user_loader
    def load_user(userid):
        return User.get(userid)

如果 ID 无效的话，它应该返回 `None`(**而不是抛出异常***)。(在这种情况下，ID 会 被手动从会话中移除且处理会继续)

一旦用户通过验证，你可以使用 `login_user` 函数让用户登录。例如::

    @app.route("/login", methods=["GET", "POST"])
    def login():
        form = LoginForm()
        if form.validate_on_submit():
            # login and validate the user...
            login_user(user)
            flash("Logged in successfully.")
            return redirect(request.args.get("next") or url_for("index"))
        return render_template("login.html", form=form)

就这么简单。你可用使用 `current_user` 代理来访问登录的用户，在每一个模板中都可以使用 `current_user`::

    {% if current_user.is_authenticated() %}
      Hi {{ current_user.name }}!
    {% endif %}

需要用户登入 的视图可以用 `login_required` 装饰器来装饰::

    @app.route("/settings")
    @login_required
    def settings():
        pass

当用户要登出时::

    @app.route("/logout")
    @login_required
    def logout():
        logout_user()
        return redirect(somewhere)

他们会被登出，且他们会话产生的任何 cookie 都会被清理干净。

你的用户类
===============
你用来表示用户的类需要实现这些属性和方法:

`is_authenticated`
    当用户通过验证时，也即提供有效证明时返回 `True` 。（只有通过验证的用户会满足 `login_required` 的条件。）

`is_active`
    如果这是一个活动用户且通过验证，账户也已激活，未被停用，也不符合任何你 的应用拒绝一个账号的条件，返回 `True` 。不活动的账号可能不会登入（当然， 是在没被强制的情况下）。

`is_anonymous`
    如果是一个匿名用户，返回 `True` 。（真实用户应返回 `False` 。）

`get_id()`
    返回一个能唯一识别用户的，并能用于从 `~LoginManager.user_loader` 回调中加载用户的 `unicode` 。注意着 **必须** 是一个 `unicode` —— 如果 ID 原本是 一个 `int` 或其它类型，你需要把它转换为 `unicode` 。

要简便地实现用户类，你可以从 `UserMixin` 继承，它提供了对所有这些方法的默认 实现。（虽然这不是必须的。）

定制登入过程
=============================
默认情况下，当未登录的用户尝试访问一个 `login_required` 装饰的视图，Flask-Login 会闪现一条消息并且重定向到登录视图。(如果未设置登录视图，它将会以 401 错误退出。)

登录视图的名称可以设置成 `LoginManager.login_view`。例如::

    login_manager.login_view = "users.login"

默认的闪现消息是 ``Please log in to access this page.``。要自定义该信息，请设置 `LoginManager.login_message`::

    login_manager.login_message = u"Bonvolu ensaluti por uzi tio pa臐o."

要自定义消息分类的话，请设置 `LoginManager.login_message_category`::

    login_manager.login_message_category = "info"

当重定向到登入视图，它的请求字符串中会有一个 ``next`` 变量，其值为用户之前访问的页面。

如果你想要进一步自定义登入过程，请使用 `LoginManager.unauthorized_handler` 装饰函数::

    @login_manager.unauthorized_handler
    def unauthorized():
        # do stuff
        return a_response


使用授权头(Authorization header)登录
======================================

.. Caution::
   该方法将会被弃用，使用下一节的 `~LoginManager.request_loader` 来代替。

Sometimes you want to support Basic Auth login using the `Authorization`
header, such as for api requests. To support login via header you will need
to provide a `~LoginManager.header_loader` callback. This callback should behave
the same as your `~LoginManager.user_loader` callback, except that it accepts
a header value instead of a user id. For example::

    @login_manager.header_loader
    def load_user_from_header(header_val):
        header_val = header_val.replace('Basic ', '', 1)                                
        try:                                                                        
            header_val = base64.b64decode(header_val)                                       
        except TypeError:                                                     
            pass
        return User.query.filter_by(api_key=header_val).first()
        
By default the `Authorization` header's value is passed to your
`~LoginManager.header_loader` callback. You can change the header used with
the `AUTH_HEADER_NAME` configuration.


使用 Request Loader 定制登录
=================================
Sometimes you want to login users without using cookies, such as using header
values or an api key passed as a query argument. In these cases, you should use
the `~LoginManager.request_loader` callback. This callback should behave the
same as your `~LoginManager.user_loader` callback, except that it accepts the
Flask request instead of a user_id.

For example, to support login from both a url argument and from Basic Auth
using the `Authorization` header::

    @login_manager.request_loader
    def load_user_from_request(request):

        # first, try to login using the api_key url arg
        api_key = request.args.get('api_key')
        if api_key:
            user = User.query.filter_by(api_key=api_key).first()
            if user:
                return user

        # next, try to login using Basic Auth
        api_key = request.headers.get('Authorization')
        if api_key:
            api_key = api_key.replace('Basic ', '', 1)                                
            try:                                                                        
                api_key = base64.b64decode(api_key)                                       
            except TypeError:                                                     
                pass
            user = User.query.filter_by(api_key=api_key).first()
            if user:
                return user

        # finally, return None if both methods did not login the user
        return None
        

匿名用户
===============
By default, when a user is not actually logged in, `current_user` is set to
an `AnonymousUserMixin` object. It has the following properties and methods:

- `is_active` and `is_authenticated` are `False`
- `is_anonymous` is `True`
- `get_id()` returns `None`

If you have custom requirements for anonymous users (for example, they need
to have a permissions field), you can provide a callable (either a class or
factory function) that creates anonymous users to the `LoginManager` with::

    login_manager.anonymous_user = MyAnonymousUser


记住我
===========
"Remember Me" functionality can be tricky to implement. However, Flask-Login
makes it nearly transparent - just pass ``remember=True`` to the `login_user`
call. A cookie will be saved on the user's computer, and then Flask-Login
will automatically restore the user ID from that cookie if it is not in the
session. The cookie is tamper-proof, so if the user tampers with it (i.e.
inserts someone else's user ID in place of their own), the cookie will merely
be rejected, as if it was not there.

That level of functionality is handled automatically. However, you can (and
should, if your application handles any kind of sensitive data) provide
additional infrastructure to increase the security of your remember cookies.


Alternative Tokens
------------------
Using the user ID as the value of the remember token is not necessarily
secure. More secure is a hash of the username and password combined, or
something similar. To add an alternative token, add a method to your user
objects:

`get_auth_token()`
    Returns an authentication token (as `unicode`) for the user. The auth
    token should uniquely identify the user, and preferably not be guessable
    by public information about the user such as their UID and name - nor
    should it expose such information.

Correspondingly, you should set a `~LoginManager.token_loader` function on the
`LoginManager`, which takes a token (as stored in the cookie) and returns the
appropriate `User` object.

The `make_secure_token` function is provided for creating auth tokens
conveniently. It will concatenate all of its arguments, then HMAC it with
the app's secret key to ensure maximum cryptographic security. (If you store
the user's token in the database permanently, then you may wish to add random
data to the token to further impede guessing.)

If your application uses passwords to authenticate users, including the
password (or the salted password hash you should be using) in the auth
token will ensure that if a user changes their password, their old
authentication tokens will cease to be valid.


Fresh Logins
------------
When a user logs in, their session is marked as "fresh," which indicates that
they actually authenticated on that session. When their session is destroyed
and they are logged back in with a "remember me" cookie, it is marked as
"non-fresh." `login_required` does not differentiate between freshness, which
is fine for most pages. However, sensitive actions like changing one's
personal information should require a fresh login. (Actions like changing
one's password should always require a password re-entry regardless.)

`fresh_login_required`, in addition to verifying that the user is logged
in, will also ensure that their login is fresh. If not, it will send them to
a page where they can re-enter their credentials. You can customize its
behavior in the same ways as you can customize `login_required`, by setting
`LoginManager.refresh_view`, `~LoginManager.needs_refresh_message`, and
`~LoginManager.needs_refresh_message_category`::

    login_manager.refresh_view = "accounts.reauthenticate"
    login_manager.needs_refresh_message = (
        u"To protect your account, please reauthenticate to access this page."
    )
    login_manager.needs_refresh_message_category = "info"

Or by providing your own callback to handle refreshing::

    @login_manager.needs_refresh_handler
    def refresh():
        # do stuff
        return a_response

To mark a session as fresh again, call the `confirm_login` function.


Cookie Settings
---------------
The details of the cookie can be customized in the application settings.

=========================== =================================================
`REMEMBER_COOKIE_NAME`      The name of the cookie to store the "remember me"
                            information in. **Default:** ``remember_token``
`REMEMBER_COOKIE_DURATION`  The amount of time before the cookie expires, as
                            a `datetime.timedelta` object.
                            **Default:** 365 days (1 non-leap Gregorian year)
`REMEMBER_COOKIE_DOMAIN`    If the "Remember Me" cookie should cross domains,
                            set the domain value here (i.e. ``.example.com``
                            would allow the cookie to be used on all
                            subdomains of ``example.com``).
                            **Default:** `None`
`REMEMBER_COOKIE_PATH`      Limits the "Remember Me" cookie to a certain path.
                            **Default:** ``/``
=========================== =================================================


Session Protection
==================
While the features above help secure your "Remember Me" token from cookie
thieves, the session cookie is still vulnerable. Flask-Login includes session
protection to help prevent your users' sessions from being stolen.

You can configure session protection on the `LoginManager`, and in the app's
configuration. If it is enabled, it can operate in either `basic` or `strong`
mode. To set it on the `LoginManager`, set the
`~LoginManager.session_protection` attribute to ``"basic"`` or ``"strong"``::

    login_manager.session_protection = "strong"

Or, to disable it::

    login_manager.session_protection = None

By default, it is activated in ``"basic"`` mode. It can be disabled in the
app's configuration by setting the `SESSION_PROTECTION` setting to `None`,
``"basic"``, or ``"strong"``.

When session protection is active, each request, it generates an identifier
for the user's computer (basically, the MD5 hash of the IP address and user
agent). If the session does not have an associated identifier, the one
generated will be stored. If it has an identifier, and it matches the one
generated, then the request is OK.

If the identifiers do not match in `basic` mode, or when the session is
permanent, then the session will simply be marked as non-fresh, and anything
requiring a fresh login will force the user to re-authenticate. (Of course,
you must be already using fresh logins where appropriate for this to have an
effect.)

If the identifiers do not match in `strong` mode for a non-permanent session,
then the entire session (as well as the remember token if it exists) is
deleted.


本地化
============
By default, the `LoginManager` uses ``flash`` to display messages when a user
is required to log in. These messages are in English. If you require
localization, set the `localize_callback` attribute of `LoginManager` to a
function to be called with these messages before they're sent to ``flash``,
e.g. ``gettext``. This function will be called with the message and its return
value will be sent to ``flash`` instead.


API 文档
=================
This documentation is automatically generated from Flask-Login's source code.


配置登录
-----------------
.. autoclass:: LoginManager
   
   .. automethod:: setup_app
   
   .. automethod:: unauthorized
   
   .. automethod:: needs_refresh
   
   .. rubric:: General Configuration
   
   .. automethod:: user_loader
   
   .. automethod:: header_loader
   
   .. automethod:: token_loader
   
   .. attribute:: anonymous_user
   
      A class or factory function that produces an anonymous user, which
      is used when no one is logged in.
   
   .. rubric:: `unauthorized` Configuration
   
   .. attribute:: login_view
   
      The name of the view to redirect to when the user needs to log in. (This
      can be an absolute URL as well, if your authentication machinery is
      external to your application.)
   
   .. attribute:: login_message
   
      The message to flash when a user is redirected to the login page.
   
   .. automethod:: unauthorized_handler
   
   .. rubric:: `needs_refresh` Configuration
   
   .. attribute:: refresh_view
   
      The name of the view to redirect to when the user needs to
      reauthenticate.
   
   .. attribute:: needs_refresh_message
   
      The message to flash when a user is redirected to the reauthentication
      page.
   
   .. automethod:: needs_refresh_handler


登录机制
----------------
.. data:: current_user

   A proxy for the current user.

.. autofunction:: login_fresh

.. autofunction:: login_user

.. autofunction:: logout_user

.. autofunction:: confirm_login


保护视图
----------------
.. autofunction:: login_required

.. autofunction:: fresh_login_required


用户对象助手
-------------------
.. autoclass:: UserMixin
   :members:

.. autoclass:: AnonymousUser
   :members:


工具
---------
.. autofunction:: login_url

.. autofunction:: make_secure_token


信号
-------
See the `Flask documentation on signals`_ for information on how to use these
signals in your code.

.. data:: user_logged_in

   Sent when a user is logged in. In addition to the app (which is the
   sender), it is passed `user`, which is the user being logged in.

.. data:: user_logged_out

   Sent when a user is logged out. In addition to the app (which is the
   sender), it is passed `user`, which is the user being logged out.

.. data:: user_login_confirmed

   Sent when a user's login is confirmed, marking it as fresh. (It is not
   called for a normal login.)
   It receives no additional arguments besides the app.

.. data:: user_unauthorized

   Sent when the `unauthorized` method is called on a `LoginManager`. It
   receives no additional arguments besides the app.

.. data:: user_needs_refresh

   Sent when the `needs_refresh` method is called on a `LoginManager`. It
   receives no additional arguments besides the app.

.. data:: session_protected

   Sent whenever session protection takes effect, and a session is either
   marked non-fresh or deleted. It receives no additional arguments besides
   the app.

.. _Flask documentation on signals: http://flask.pocoo.org/docs/signals/
