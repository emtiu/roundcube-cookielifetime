# patching Roundcube webmail for configurable login cookie lifetime

There's [been](https://github.com/roundcube/roundcubemail/issues/5050) [considerable](https://github.com/roundcube/roundcubemail/pull/7709) [interest](https://github.com/roundcube/roundcubemail/issues/7865) [over](https://github.com/roundcube/roundcubemail/issues/7251) [the](https://packagist.org/packages/texxasrulez/persistent_login) [years](http://lists.roundcube.net/pipermail/dev/2007-August/005317.html) in configuring the lifetime of Roundcube's login cookies. However, such a feature is [not going into Roundcube anytime soon](https://github.com/roundcube/roundcubemail/issues/7865#issuecomment-770343039).
## How Roundcube's default session lifetime management works
*(incomplete description, considering only aspects affected by this patch)*

The `session_lifetime` config option defines how many minutes may pass without a user having Roundcube open/reachable before the server considers their session expired and the user is logged out.

Also, Roundcube sets a cookie named `roundcube_sessauth` (by default, the name is configurable by the `session_auth_name` config option). If the browser does not present this cookie when accessing Roundcube, the user is logged out.

Roundcube hardcodes the lifetime of this cookie as 0, meaning the browser considers it a *"session cookie"*. Browsers delete session cookies on quit/exit.

All of this means that after login, the following situations lead to a user being logged out automatically:
- The user has closed Roundcube/lost the connection, and the number of minutes defined in the `session_lifetime` config option has passed without the user opening Roundcube again.
- The user's browser quits, deleting the `roundcube_sessauth` cookie.
## What's the problem? / Why this patch?
Some users want their session to stay alive even when they close their browsers. Also, consider that smartphone browsers don't actually ever *quit* in any predictable way, which means that users of Roundcube on a smartphone may get logged out unpredictably at any time.

Setting a lifetime on the `roundcube_sessauth` solves this by giving the browser a defined time until which to keep the cookie, regardless of if or when the browser quits. There are two ways to achieve this:
 1. **Manipulate the lifetime of the `roundcube_sessauth` cookie in the browser, preventing it from getting deleted.** (This works without changing Roundcube's code, but has to be done manually after each login, because Roundcube sets `roundcube_sessauth` as a session cookie on each login.)
 2. **Use this patch to make the `roundcube_sessauth` cookie's lifetime configurable in Roundcube.**
## How it works
**This patch adds a config option named `cookie_lifetime`, defined in minutes.** When Roundcube sets the `roundcube_sessauth` cookie on login, it adds this number of minutes to the current time to set the expiration time of the cookie. When this time is reached, the browser deletes the cookie, effectively logging the user out. (The same lifetime is set on the `roundcube_sessid` cookie holding the session id.)

If set to 0, the cookie is set as a session cookie. This is the default value of this new config option, and mirrors Roundcube's unpatched default behavior.

Note that whatever the `cookie_lifetime` set for the browser, sessions always expire after closing Roundcube when the `session_lifetime` (as tracked by the server) has expired. Therefore, it makes no sense to set a `cookie_lifetime` longer than the `session_lifetime`.

With this patch, a user will be automatically logged out in the following situations:
- The number of minutes defined in `cookie_lifetime` has passed since the last login.
- The user has closed Roundcube/lost the connection for more than the number of minutes defined in `session_lifetime`.
### Some examples:
1. ***`cookie_lifetime` set to 14 days, `session_lifetime` set to 2 days:*** Any browser that logs in to Roundcube stays logged in for 14 days at most, but is logged out automatically after 2 days have passed without opening Roundcube. **--> If Roundcube is accessed at least every 2 days, the user will only need to re-login every 14 days.**
2. ***`cookie_lifetime` set to 5 days, `session_lifetime` set to 5 days:*** Any browser that logs in to Roundcube stays logged in for 5 days, no matter how often Roundcube is opened. **--> The user will only need to re-login every 5 days.**
3. ***`cookie_lifetime` set to 0***: Roundcube's unpatched default behavior (see above).
## Caution!
A long-lived authentication cookie poses a security risk. Whoever has the auth cookie (and the session id) is considered logged in without entering a password. Use this patch only if you know what this means and are willing to accept it.
