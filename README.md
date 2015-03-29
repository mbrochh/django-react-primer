# Django React.js Primer

A primer for building anything with [Django](https://www.djangoproject.com) and
[React](http://facebook.github.io/react/).

## Status

This is a work in progress. This repository is basically a stream of
consciousness and a collection of links as I venture into React.js land and try
to figure out how to make it work for my existing Django toolchain.

## Table of Contents
- [Author's Note](#authors-note)
- [Part 1: Status Quo](#status-quo)
  - [1.1: The Stack](#the-stack)
  - [1.2: Project Structure](#project-structure)
  - [1.3: Shortcomings](#shortcomings)
    - [1.3.1: Too Many Partials](#too-many-partials)
    - [1.3.2: Too Many Requests](#too-many-requests)
    - [1.3.3: Too Much jQuery Spaghetti Code](#too-much-jquery-spaghetti-code)
    - [1.3.4: Unmaintainable CSS](#unmaintainable-css)
    - [1.3.5: No Realtime](#no-realtime)
- [Part 2: Possible Solution](#possible-solution)
  - [2.1: New Stack](#new-stack)
  - [2.2: New Project Structure](#new-project-structure)
  - [2.3: Results](#results)
- [Part 3: The Toolchain](#the-toolchain)

---

## Authors Note

My Javascript knowledge is limited to the usual work with jQuery and some DOM
manipulation. Therefore, getting into React and it's vast ecosystem is a major
hurdle for me. After reading blog posts, tutorials and Github repositories
for the past few weeks I feel that I need to write something down in order to
make any progress.

There are so many things that confuse me, I keep ending my days with my brain
hurting because as a Python/Django developer, there are so many new things to
learn that it feels just overwhelming:

* What is ES6, babel, 6to5, transpilers, do I need this? Can I use this?
* Grunt or Gulp?
* Browserify or Webpack?
* What is CommonJS, AMD, ES6 modules?
* Which Flux implementation do I need? Why the fuck are there 20 different?
* What are promises, immutability, isomorphic apps?
* Which testing library is the best?
* How to work with the npm ecosystem?

And most importantly:

How can I build generic reusable components? Every tutorial out there seems to
assume that you write all your components from scratch for each project. I want
to publish all my components on npm and re-use them in all my projects. How
should I structure my npm modules and how should I provide static assets such
as images and styles?

These are the questions that I will hopefully find answers for in the next
few weeks.

By the way: The structure of this repository is inspired by
[Mike Chau's react-primer-draft](https://github.com/mikechau/react-primer-draft)

## Status Quo

We have built everything from small to large Django applications during the
last four years. We are a small team of three full stack developers. Designs
are usually handed to us as PSD files and we take care about the rest.

### The Stack

Our stack usually consists of the following tools:

[nginx](http://nginx.org), [uwsgi](http://uwsgi-docs.readthedocs.org/en/latest/),
[Django](https://www.djangoproject.com), [django-cms](https://www.django-cms.org),
[django-filer](http://django-filer.readthedocs.org/en/latest/),
[django-debug-toolbar](http://django-debug-toolbar.readthedocs.org),
[Bootstrap](http://getbootstrap.com), [less](http://lesscss.org),
[Fabric](http://www.fabfile.org), [Memcached](http://www.memcached.org) and
[solr](http://lucene.apache.org/solr/)

All the above is essentially wrapped up in the following boilerplate
repositories:

[django-project-template](https://github.com/bitmazk/django-project-template),
[django-reusable-app-template](https://github.com/bitmazk/django-reusable-app-template) and
[django-development-fabfile](https://github.com/bitmazk/django-development-fabfile)

### Project Structure

We try to stick to the DRY principle as best as possible, even across different
projects. Therefore, we release almost everything as reusable Django apps on
Github and re-use apps in all our projects whenever possible.

A project usually only consists of the following major parts:

1. A settings module which has a lot of reusable apps in the `INSTALLED_APPS`
   setting.
2. A `urls.py` which is hooks up all those reusable apps.
3. A `requirements.txt` file which allows new developers to setup a local
   development environment quickly.
4. A fabfile which allows developers to setup a local database, run the
   Python unit-tests, measure code coverage, lint the code and run
   automated deployments.
5. A templates folder which contains a `base.html` and lots of overrides for
   all the templates of all reusable apps.
6. A static folder which contains the bootstrap styles, our own style
   overrides, images, third party .css and .js files, usually for jQuery
   plugins.

### Shortcomings

The above stack and project structure works extremely well for our small team.
Everyone knows about this stack and structure very well and therefore everyone
is able to quickly setup any project and commit fixes.

We consider our products to be extremely well crafted and highly maintainable
but as with most projects, there are some shortcomings that we have realized
over time but never really cared to solve (because things work good enough).

#### Too Many Partials

We try to split our Django templates into the smallest possible units - we call
them "partials". This makes for great reusability, but it comes with a hefty
performance penalty. Once we reach a certain amount of includes in a template,
requests become painfully slow, even when keeping all (un-rendered) templates
in cache. 

Django's [template fragment caching](https://docs.djangoproject.com/en/1.7/topics/cache/#template-fragment-caching)
can often alleviate this problem, but it ads complexity to the code and is not
always an option. The added complexity means that developers must now remember
which fragments are cached and must decide which actions should invalidate
those caches. Since the concerns are separated far away from each other (cache
block in the template vs. data manipulation in views, forms or models), it is
virtually impossible to document and remember when caches need to be
invalidated.

Django also makes it very easy to use templatetags and filters in the templates
and the temptation to have SQL queries in those tags/filters is almost
irresistible. Now all you need is to put a certain templatetag into a nested
for-loop and you end up with potentially hundreds of SQL queries to render
a template.

Again, Django's `select_related` and `prefetch_related` and
[django-cachalot](https://github.com/BertrandBordage/django-cachalot) can fix
this but this requires very experienced developers who understand SQL very well
and a lot of debugging and refactoring in order to identify bottlenecks and
then pass along pre-fetched instances. If one developer adds a `.filter()` to
one of the pre-fetched querysets, all the hard work will be for nothing, the
performance bottleneck will re-surface and it will be very difficult to find
out where the problem happens.

#### Too Many Requests

We try to use
[django-compressor](https://github.com/django-compressor/django-compressor) but
sometimes, reusable Django apps have [form assets](https://docs.djangoproject.com/en/1.7/topics/forms/media/).
Those assets cannot be easily added to one big compressed CSS or JS file
because they only appear on the views that use those forms. Therefore, calling
`{{ form.media }}` in a sub-template seems to be an anti-pattern. The only way
around this seems to be to dive into the third-party-app's code and see which
assets are being used and then add them to the compress-blocks in `base.html`.
We never bothered to do this.

As a result, our pages usually trigger a multitude of HTTP requests,
potentially resulting in lower SEO rankings. [django-sekizai](http://django-sekizai.readthedocs.org/en/latest/)
tries to address this problem but we don't like this solution at all. It does
make sure that even deep down in the deepest partial template you will be
adding assets to the one-and-only sekizai-blocks in your base template but this
would mean that different views will genrate different compressed assets -
which means that users would re-download large parts of the code just because
some small additional script is needed on a certain view.

If we compress files, they must know about *everything* that the whole project
will ever need and include all of it.

#### Too Much jQuery Spaghetti Code

As projects grow, developers tend to keep adding third party libraries, some of
them in the page `HEAD`, some of them before the closing `BODY` tag.

Even worse, all kinds of initialization for various jQuery plugins happens
all over the place, sometimes near the end of `base.html`, sometimes in a
`general.js` file which is compressed, sometimes in `{% block extra_script %}`
blocks deep down in partial templates (which is not compressed).

Sometimes, JS snippets need to get some values from Django (i.e. URLs for AJAX
calls), so we start adding global variables, again they appear all over the
place.

Our UIs got much more robust when we stopped using IDs and class names and
started using data-attributes for selecting DOM objects via `$(...)` but it
remains very hard to understand maintain complex user interfaces.

#### Unmaintainable CSS

We have optimized the way we [manage Twitter Bootstrap](http://martinbrochhaus.com/bootstrap.html)
in a unique way that allows us to keep Bootstrap up to date very easily (just
run `git pull` on the git-submodule) and make use of Bootstrap's
`variables.less` and even override existing variables with our own values. This
is leaps and bounds better than just downloading `bootstrap.min.js` and then
add our own `styles.css` with project specific overrides, but it still results
in one gigantic project specific `styles.less` which tries to style everything,
from our own global site structure down to the tinyest element of some third
party app.

As a project grows and changes, developers usually don't dare to delete
existing styles because they don't know if they are still in use somewhere, so
we keep adding more and more styles all the time.

On the other hand, if we use the full power of less, obey the DRY principle
and put everything into variables and re-use those variables (and whole blocks
of CSS), we end up not being sure if a certain change can be made safely
because we don't know how many other elements would be affected by this (and
where to see those elements on the page).

A gigantic styleguide-view that contains every single element that is ever used
in the project would be a solution for this, but especially when facing tight
budgets or deadlines, developers tend to skip maintaining that styleguide.
Maintenance of that file can also become very tricky because often a certain
partial can look very different depending on the data given to it. This means
you would have to include several versions of that partial into the
styleguide-view.

Sometimes, creating fixtures of data just to include a given partial can
be so tricky that nobody bothers even to try it (i.e. when the partial needs
an object with complex fields and attributes and then some stuff from the
request and the session - good luck with that).

We wrote a [load_context templatetag](https://github.com/bitmazk/django-libs/blob/master/docs/libs_tags.rst#load_context)
to solve this problem, but as mentioned, creating correct fixtures remains
tricky and keeping them up to date is even trickier.

#### No Realtime

I know that it is possible to do fancy realtime stuff with Django. I have tried
that a while ago and it was a weird and painful experience. I haven't tried
[Swampdragon](http://swampdragon.net), yet but it looks promising and I keep
hearing good things about it.

But: [Node.js](https://nodejs.org) has taken the crown when it comes to
anything related to [Socket.io](http://socket.io). So there is this tool that
everyone already uses for realtime stuff and it seems to work extremely well
for a lot of people out there - why should I even bother to wrestle with
Tornado and gevent and greenlet and all these hacks?


## Possible Solution 

Right now, this chapter is pretty much wishful thinking and some ideas that I
have gotten during the last weeks while soaking all available online-knowledge
about React.js. This chapter will probably change as I learn more about the
possibilities and limitations of the new stack.

### New Stack

#### For New Projects

Django will no longer serve any templates. It will only provide a JSON API via
[django-rest-framework](http://www.django-rest-framework.org). Spike Brehm
of airbnb has shared some nice ideas about
[isomorphic javascript](http://nerds.airbnb.com/isomorphic-javascript-future-web-apps/).
If you scroll to the second image of that post, you will get a good idea on
how the new stack should work. Admins will still be able to use the awesome
Django admin in order to maintain the database and we will still be able to
use the awesome ORM (and it's caching solutions) to make queries that any
developer can easily understand, but we will no longer use Django's views,
forms and templating language. To be fair, the APIViews of
django-rest-framework are similar to Django's views and forms, so we are really
just giving up the templating engine.

[Node.js](https://nodejs.org) will serve the whole bundled app to the client.
This is the isomorphic part. We will have a `server.js` file which consists
of an [express](http://expressjs.com) application that listens to the requests
and responds with the matching rendered React component. This way, we can
pre-render the requests on the server, so that search engines won't get an
empty page. Once the client recieves the response, everything else happens on
the client side (just calling API endpoints, re-rendering parts of the page).

Nginx will serve the Django API at `example.com/api/v1/` (via uwsgi) and the
Node.js server at `example.com`. Nginx will also serve static files (as usual).

[react-router](https://github.com/rackt/react-router) will be used as a
replacement of our beloved Django `urls.py` files. It defines which URL should
map to which React component and has some cool ways to transition between
"views", handle redirects and even pickup interrupted intents (i.e. when you
try to access a restricted area and need to login first).

[react-bootstrap](http://react-bootstrap.github.io/index.html) shall be used
to replace vanilla bootstrap.

[newforms-bootstrap](https://github.com/insin/newforms-bootstrap) could be used
to render forms. It has an API that was inspired by Django, so the code for
this will look very familiar to us. It will handle frontend-validation nicely.
Unfortunately, we will still have to define the same validation in 
django-rest-framework (always validate on the server), so there is a possible
duplication of code. It might be a better idea to always let
django-rest-framework do the validation and then just display the returned
error messages (if any).

[react-style](https://github.com/js-next/react-style) could be used to create
styles right there in the files where the components are implemented. By also
passing in styles via props, this would allow us to override the implemented
default styles by passing in a big StyleSheet into the ContainerComponent.
ReactNative uses something with the same syntax. I'm not sure if this would be
compatible with newforms-bootstrap, though.

[Webpack](http://webpack.github.io) will be used to bundle the app. It should
result in one or more vendors files (i.e. for react.js itself) and one file
that contains our own whole app. I'm not even sure, if it makes sense to
separate the vendors files. If the total size of the bundle would be somewhere
at 200kb, I guess everything could just be included into one big bundle.

[Gulp](http://gulpjs.com) will be used to trigger certain build tasks (similar
to our Django fabfile).

[react-starter-kit](https://github.com/kriasoft/react-starter-kit) shall be
used as a basis to start new projects, until we know enough to host our own
version (this one is lacking some things, like react-router, and a server
implementation for isomorphic apps). Overall it seems to be a great starting
point and seems to be the only one that has BrowserSync in it's stack.

#### For Existing Projects

TODO

### New Project Structure 

TODO

### Results

TODO

## The Toolchain

TODO: This chapter should describe in detail how the [2. Possible
Solution](#possible-solution) can be implemented.
