![header](https://filebox.tymoon.eu/file/TVRFNE9RPT0=)  
It's been a good while since I [last worked on Radiance](https://blog.tymoon.eu/article/308). Unfortunately I can't claim that this was because Radiance was completely finished, quite far from it to be truthful. However, it has been stable and running well enough that I didn't have to tend to it either. In fact, you're reading this entry on a site served by Radiance right now. Either way, thanks to some nudging by my good friend Janne, I've now returned to it.

I did some rather serious refactoring and cleanup in the past week that I'm certain brought Radiance a good chunk closer to being publishable. For me, a project needs to fulfil a lot of extra criteria in order to be publishable, even if it already works well enough in terms of the code itself just doing what it's supposed to. So, while it'll be a while before Radiance is ready for public use and biting critique, the overall architecture of it is set in stone. I thought it would be a good idea to elaborate on that part in this entry here, to give people an idea of what sets Radiance apart.

Radiance is, in some sense of the term, a web framework. However, it is sufficiently different from what usually qualifies as that for me to not consider the term an apt description. I instead opted for calling it a "web application environment". If you're familiar with web development in Lisp, you might also know of [Clack](https://github.com/fukamachi/clack), which also adopted that term after I introduced it in a [lightning talk at ELS'15](https://www.youtube.com/watch?v=VVNbn99jGRk). Clack comes closer to what Radiance is than most web frameworks, but it's still rather different.

So what is it then that sets Radiance apart? Well, that is explained by going into what Radiance really does, which are only two things: interface management and routing. Naturally it provides some other things on the side as well, but those are by far the biggest components that influence everything.

Back when I was a young bab and had ambition, I decided to write my own websites from scratch. Things developed and evolved into a framework of sorts over the course of a multitude of rewrites. One of the things I quickly became concerned with was the question of how to handle potential user demands. Let's say that I'm an administrator and would like to set up a blog software that uses Radiance. I have some requirements, such as what kind of database to use, and perhaps I already have a webserver running too. Maybe I would also like to pick a certain method of authenticating users and so forth. This means that the actual framework and blog software code needs to be sufficiently decoupled in order to allow switching out/in a variety of components.

This is what interfaces are for. Radiance specifies a set of interfaces that each outline a number of signatures for functions, macros, variables, and any kind of definable thing. As an application writer, you then simply say that you depend on a particular interface and write your code against the interface's symbols. As an administrator you simply configure Radiance and tell it which implementation to use for which interface. When the whole system is then loaded, the application's interface dependencies are resolved according to the configuration and the specified implementation then provides the actual functions, macros, etc that make the interface work.

This also allows you to easily write an implementation for a specific interface should you have particular demands that aren't already filled by implementations that are provided out of the box. Since most applications will be written against these interfaces, everything will 'just work' without having to change a single line of code and without having to write your application to be especially aware of any potential implementation. The modularity also means that not every interface needs to have an implementation loaded if you don't need it at all, avoiding the monolith problem a lot of Frameworks pose.

Unlike other systems that use dynamic deferring to provide interfaces, Radiance's way of doing things means that there is zero additional overhead to calling an interface function and that macros also simply work as you would expect them to. While interfaces are not that novel of an idea, I would say that this, coupled with the fact that Radiance provides interfaces on a very fine level (there's interfaces for authentication, sessions, users, rate limiting, banning, cache, etc), makes it distinct enough to be considered a new approach.

Let's look at an example in the hopes that it will make things feel a bit more practical. Here's the definition for the cache interface:

```common-lisp
(define-interface cache
  (defun get (name))
  (defun renew (name))
  (defmacro with-cache (name-form test-form &body request-generator)))
```

This definition creates a new package called `cache` with the functions `get` and `renew` and the macro `with-cache` in it. `get` is further specified to retrieve the cached content, `renew` will throw the cache out, and `with-cache` will emit the cached data if available and otherwise execute the body to generate the data, stash it away, and return it.

There's a couple of different ways in which the cache could be provided. You could store it in memory, save it to file, throw it into a database, etc. Whatever you want to do, you can get the behaviour by writing a module that implements this interface. For example, here's an excerpt from the `simple-cache` default implementation:

```common-lisp
(in-package #:modularize-user)
(define-module simple-cache
  (:use #:cl #:radiance)
  (:implements #:cache))
(in-package #:simple-cache)

(defun cache:get (name)
  (read-data-file (cache::file name)))

(defun cache:renew (name)
  (delete-file (cache::file name)))

(defmacro cache:with-cache (name test &body request-generator)
  (let ((cache (gensym "CACHE")))
    `(let ((,cache ,name))
       (if (or (not (cache::exists ,cache))
               ,test)
           (cache::output ,cache ((lambda () ,@request-generator)))
           (cache:get ,cache)))))
```

We define a module, which is basically a fancy package definition with extra metadata. We then simply overwrite the functions and macros of the interface with definitions that do something useful. You'll also notice that there's references to internal symbols in the `cache` interface. This is how implementations can provide implementation-dependant extra functionality through the same interface. The actual definitions of these functions are omitted here for brevity.

Now that we have an interface and an implementing module, all we need to do is create an ASDF system for it and tell radiance to load it if someone needs the interface.

```common-lisp
(setf (radiance:mconfig :radiance :interfaces :cache) "simple-cache")
```

Finally, if we want to use this interface in an application we simply use the interface functions like any other lisp code, and add `(:interface :cache)` as a dependency in the system definition. When you quickload the application, simple-cache will automatically be loaded to provide the interface functionality.

Radiance provides one last thing in order to make the orchestra complete: environments. It is not too far out there to imagine that someone might want multiple, separate Radiance setups running on the same machine. Maybe the implementations used in production and development are different and you'd like to test both on the same machine. Perhaps you're developing different systems. Either way, you need to multiplex the configuration somehow and that's what the environment is for.

The environment is a variable that distinguishes where all of the configuration files for your radiance system are stored, including the configuration and data files of potential applications/modules like a forum or a blog.

So, to start up a particular setup, you just set the proper environment and launch Radiance as usual.

-------

The second part, routing, is something that every web application must do in some way. However, Radiance again provides a different spin on it. In this case, it is something that I have never seen before anywhere else. This may well be due to ignorance on my behalf, but for now we'll proceed in the vain hope that it isn't.

Routing is coupled to one of the initial reasons why Radiance even came to be. Namely, I wanted to have a website that had a couple of separate components like a forum, a gallery, and a blog, but all of them should be able to use the same user account and login mechanism. They should also be able to somehow share the "URL address space" without getting into conflicts with each other as to who has which part of the URL. Coupling this with the previous problem of the webmaster having a certain number of constraints for their setup such as only having a single subdomain available, or having to be in a subfolder, things quickly become difficult.

The first part of the problem is kind solved partially by interfaces, and the rest could be solved if we, say, gave each module of the whole system its own subdomain to work on. Then the URL path could be done with as each part requires it to be. This approach stops working once we consider the last problem, namely that the system cannot presume to have the luxury of infinite subdomains, or the luxury of really having much of any control over the URL layout.

This issue gets worse when you consider the emitted data. Not only do we need to know how to resolve a request to the server to the appropriate module, we also need to make sure to resolve each URL in the HTML we emit back again so that the link won't point to nowhere. Compounding on that is the question of cross-references. Most likely your forum will want to provide a login button that should take them to the pages of the authentication module that then handles the rest.

To make this more concrete, let's imagine an example scenario. You're the webmaster of `humble-beginnings.free.home-dns-resolver.com`. Since this is a free hoster that just gives you a domain that refers to your home server, you can't make any subdomains of your own. Worse, you don't have root access on your machine, so you can't use port `80`. You've already set up a simple file server to provide some intro page and other files that you would like to keep at the root. You'd now like to set up a forum on `/forum/` and a blog on `/personal/blog/`. This might seem far-fetched, but I don't deem it too much of a stretch. Most of the constraints are plausible ones that do occur in the wild.

So now as an application writer for the forum or blog software you have a problem. You don't know the domain and you don't know the path either. Hell, you don't even know the port. So how can you write anything to work at all?

The answer is to create two universes and make Radiance take care of them. There's an "external" environment, which is everything the actual HTTP server gets and sends, and thus the user reading a site sees and does. And then there's an "internal" environment, which is everything an application deals with. As an application writer you can now merrily define your endpoints in a sane and orderly fashion, and the admin can still make things work the way he wants them to work. This is what Radiance's routing system allows.

To this effect, Radiance defines a URI object that is, on a request, initialised to the address the server gets. It is then transformed by mapping routes, and then dispatched on. Each application then creates URIs for the links it needs and transforms them by the reversal routes before baking them into the actual HTML.

Again, I think a specific code example will help make things feel more tangible. Let's first define some stubs for the forum and the blog software:

```common-lisp
(define-page forum "forum/" ()
  (format NIL "<html><head><title>Forum</title></head>~
               <body>Welcome to the forum! Check out the <a href=~s>blog</a> too!</body></html>"
          (uri-to-url "blog/" :representation :external)))
  
(define-page blog "blog/" ()
  (format NIL "<html><head><title>Blog</title></head>~
               <body><a href=~s>Log in</a> to write an entry.</body></html>"
          (uri-to-url (resource :auth :page :login) :representation :external)))
```

Radiance by convention requires each separate module to inhabit its own subdomain. You also see that in the forum page we include a link to the blog by externalising the URI for its page and turning it into a URL. In the blog page we include a link to the login page by asking the resource system for the page in the authentication interface. We can do this because the authentication interface specifies that the page resource must be resolvable and must return the appropriate URI object.

Next we need to tell Radiance about our external top-level domain so that it doesn't get confused and try to see subdomains where there are none.

```common-lisp
(add-domain "humble-beginnings.free.home-dns-resolver.com")
```

By the way, you can have as many different domains, ports, and whatevers as you want simultaneously and the system will still work just fine.

Finally we need the actual routes that do the transformations. Funnily enough, despite the apparent complexity, this current setup allows us to use the simplest route definition macro. You can do much, much more complicated things.

```common-lisp
(define-string-route forum :mapping "/forum/(.*)" "forum/\\1")
(define-string-route forum :reversal "forum/(.*)" "/forum/\\1")
(define-string-route blog :mapping "/personal/blog/(.*)" "blog/\\1")
(define-string-route blog :reversal "blog/(.*)" "/personal/blog/\\1")
```

And that's all. Radiance will now automatically rewrite URIs in both directions to resolve to the proper place.

For an example of a more complicated route, Radiance provides a default [`virtual-module` route](https://github.com/Shirakumo/radiance/blob/master/defaults.lisp#L180) that allows you to use the `/!/foo/..` path to simulate a lookup to the `foo` subdomain. This may seem simple at first, but it needs to do a bit of extra trickery in order to ensure that all external URIs are also using the same prefix, but only if you already got to the page by using that prefix. 

But it just works. The route system neatly confines that complexity to a single place with just two definitions and ensures complete agnosticism for all applications that reside in Radiance. No matter what kind of weird routes you might have to end up using, everything will resolve properly automagically provided you set up the routes right and provided you do use the URI system.

---

And that's, really, mostly all that Radiance's core does. It shoves off all of the functionality that is not strictly always needed to interfaces and their various possible implementations. In the end, it just has to manage those, and take care of translating URIs and dispatching to pages.

I don't want to toot my own horn much here, mostly because I despise bragging of any kind and I'm much too insecure to know if what I've done is really noteworthy. On the other hand, I think that this should illustrate some of the benefits that Radiance can give you and present a brief overview on how things fit together.

As I mentioned at the beginning, Radiance is not yet ready for public consumption, at least according to my own criteria. Lots of documentation is missing, many parts are not quite as well thought out yet, and in general there's some idiosyncrasies that I desperately want to get rid of. So, unless you feel very brave and have a lot of time to spend learning by example, I would not advise using Radiance. I'm sure I won't forget to blog once I do deem it ready though. If you are interested, you'll simply have to remain patient.
