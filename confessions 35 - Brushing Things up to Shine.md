![header](http://shinmera.tymoon.eu/public/46115712_p9.jpg)
Continuing on from [last time](http://blog.tymoon.eu/article/292), we'll now take a look at users, sessions and authentication. Starting out the most obvious deficiency in our twatter application is that anyone could post as anyone.

Fixing this required an authentication mechanism (which implicitly requires a session and users mechanism). Luckily, Radiance provides an interface for all of those too. Start out by adding `(:interface :auth)` to your system and quickloading it again to get the dependencies in. If you started a new session and radiance isn't loaded or running yet, don't worry. Just proceed as above and everything will load properly. You can then call `radiance:startup` again. If Radiance was already running when you loaded the dependencies, you'll have to call `(radiance:trigger 'db:connected)` to ensure the modules are properly setup.

The default configuration will now pull in `r-simple-users`, `r-simple-sessions` and `r-simple-auth`. These are all rather straightforward implementations of the interfaces that come with the radiance base. Now let's see how to utilise this.

First we'll want to make sure that the API page is secured by removing the `user` argument and requiring a logged in user.

```commonlisp
(define-api twatter/status/create (text &optional client) (:access (twatter status create))
  (unless (<= 1 (length text) 140)
    (error 'api-argument-invalid :argument 'text :message "Text must be between 1 and 140 characters."))
  (db:insert 'twatter-statuses
             `((user . ,(user:username (auth:current)))
               (text . ,text)
               (time . ,(get-universal-time))))
  (when client (redirect (referer)))
  (api-output "Ok"))
```

In order to enforce authentication we add the `:access` option. This not only checks for an active user session, but also checks for a certain permission in order to let the query go through. Also note the change of the user field to `(user:username (auth:current))`. `auth:current` returns the `user` object currently associated with the session.

Permissions in Radiance work through a mechanism called 'branches', which is basically a sequence of tokens that get more and more specific. Each user has a list of branches they have access to. If they have access to, say `foo bar baz` then `foo bar baz` and `foo bar baz fab` and so on will all pass the check, but things like `foo bar zab` or `bla` won't. You can specify branches through either a list of symbols or as a dot-separated string.

However, as it is currently we would lock out all users from posting and that's probably not what we want. Of course we could manually assign the posting permission for every user, but that's kind of dumb. So instead, we'll just assume that by default everyone should be able to post anyway. If that's not the case, the sysop in question can change the default permissions. In order to do this, we'll add the following form.

```commonlisp
(user:add-default-permission '(twatter status))
```

This'll add a default permission. Whenever a new user is created now, it'll have the above branch available. Since we specialised on `create` in our API, he should have access to that as well. If you try to create a new status now you'll see that access will be denied.

Next we need to fix our front page to only show anything if it is a page for an existing user and only the form if the current viewer is the user in question.

```commonlisp
(define-page twatter-user #@"/user/([a-zA-Z]+)" (:uri-groups (username) :lquery (template "user.ctml"))
  (let ((user (user:get username)))
    (if user
        (r-clip:process
         T
         :user user
         :statuses (dm:get 'twatter-statuses (db:query :all)
                           :amount 25 :sort '((time :DESC))))
        (error 'request-not-found))))
```

The template also needs some changes:

```html
&lt;!DOCTYPE html&gt;
&lt;html xmlns="http://www.w3.org/1999/xhtml"&gt;
  &lt;head&gt;
    &lt;meta charset="utf-8"/&gt;
    &lt;title&gt;&lt;c:splice lquery="(text (user:username user))" /&gt; - Twatter Profile&lt;/title&gt;
    &lt;link rel="stylesheet" type="text/css" href="/static/twatter/default.css" /&gt;
  &lt;/head&gt;
  &lt;body&gt;
    &lt;h1 lquery="(text (user:username user))"&gt;USER&lt;/h1&gt;
    &lt;c:when test="(eq (auth:current) user)"&gt;
      &lt;h3&gt;Create a new Status&lt;/h3&gt;
      &lt;form action="/api/twatter/status/create" method="post"&gt;
        &lt;textarea name="text" maxlength="140" required&gt;&lt;/textarea&gt;
        &lt;input type="hidden" name="client" value="true" /&gt;
        &lt;input type="submit" value="Update!" /&gt;
      &lt;/form&gt;
    &lt;/c:when&gt;
    &lt;h3&gt;Status Updates&lt;/h3&gt;
    &lt;ul iterate="statuses"&gt;
      &lt;li&gt;
        &lt;article&gt;
          &lt;header&gt;
            &lt;span class="username" lquery="(text user)"&gt;USER&lt;/span&gt;
            &lt;span class="timestamp" lquery="(text (twatter::format-time time))"&gt;1900-01-01 00:00:00&lt;/span&gt;
          &lt;/header&gt;
          &lt;blockquote lquery="(text text)"&gt;
            TEXT
          &lt;/blockquote&gt;
        &lt;/article&gt;
      &lt;/li&gt;
    &lt;/ul&gt;
  &lt;/body&gt;
&lt;/html&gt;
```

Now that we're getting error pages galore, we'll need to actually set up a user for us to play around with. `r-simple-auth` offers a login and registration page. For now we'll set up our user account manually though.

```commonlisp
(user:get "radguy" :if-does-not-exist :create)
```

In order for sessions to work properly, we however need to move away from `localhost`, as browsers will not work nicely with cookies on that domain. To circumvent this, add something like `127.0.0.1 radiance.test` to your `hosts` file. We'll use this from now on to test. 

As soon as you visit [radiance.test:8080](http://radiance.test:8080) you should receive a `radiance-session` cookie that ties you to a session. This happens regardless of whether you're actually logged in or not and serves as a way to preserve state between calls. 

You can list all existing sessions with `session:list`. This will most likely show you a bunch of sessions now, which were created during your experimentation with the localhost domain -- and failed to persist. In order to figure out which one is your current, you can either end all of the current sessions using `(mapcar #'session:end (session:list))` then refresh your page and check again, or you can change the logging level to `:debug`, which should show which session is being resumed once you load a page.

To tie the user to the session, you first need to grab the session. For this, copy out the UUID and call `auth:associate`:

```commonlisp
(auth:associate (user:get "radguy") (session:get "99AA97A2-6684-446C-BEDF-39FF2507B20A"))
```

You can now visit [radiance.test/user/radguy](http://radiance.test:8080/user/radguy) and see that it'll show you the profile and allow you to post new statuses. Neat. You can create a second user and see that you won't be able to post on their profile and vice-versa if you change the association again.

That's it for now. This is shorter than I'd like, but if I started including more of what I've got planned it would most likely end up too long. So next time: User profiles, administration, and routing.
