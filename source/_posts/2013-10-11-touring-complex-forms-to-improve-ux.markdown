---
layout: post
title: "Touring complex forms to improve UX"
date: 2013-10-11 13:11
author: Dan Burton
comments: true
categories: 
---

The web is hard. Let's make it easier.

(This post is in the works; it's security by obscurity, so shhh,
don't tell!)

<!-- more -->

When providing a web service with lots of features,
it is often the case that you want to deliver a lot of power
to your user without a lot of unnecessary clicking.
This sometimes leads to very dense configuration pages.
The user, when first confronted with so many options,
will surely be overwhealmed. She may be able to find just the right
nobs and frobs that she needs for some particular task,
but will surely have a hard time comprehending everything
that the page allows her to control. Features may be staring her in
the face, but such dense and confusing pages may cause her to miss
that which is right in front of her.

There are a few solutions to this problem.
We can simplify her experience by limiting her options.
Sometimes less is more. But sometimes not.
Another solution is to train her in person.
You can point at the screen and say, "you can use this option to do this,"
and "when you want to do that, go here, and then click that."
For various reasons, this isn't always possible. It certainly becomes
a huge bottleneck when you want many people to spontaneously try out
your awesome web service.

Our goal today is to simulate the "personal tour" eperience mechanically.
We do this by:

* Darkening and disabling the page
* Highlighting features on the screen, one at a time
* Explaining how to use that feature
* Lightening and re-enabling the page

I'm going to step you through the changes to a rails project,
using haml, sass, and coffeescript. However, note that the heart
of this tutorial really has nothing to do with those technologies,
and you can do everything I show here with a trivial translation to
plain old html, css, javascript, and your web server of choice.

Although this post isn't *about* rails, I do cover, in detail,
all of the little steps necessary to get things done, which I hope
can incidentally also serve as a nice little rails primer for those
who are interested.

## Initial setup

Let's whip up a new rails project that uses haml.

```bash
rails new tour-example && cd tour-example && echo "gem 'haml'" >> Gemfile && bundle install
```

We'll start off with a simple page.

```haml
-# app/views/application/index.haml

#example-page
  .one
    Part one
  .two
    Part two
  .three
    Part three
```

With some style.

```sass
// app/assets/stylesheets/index.sass

.one, .two, .three
  margin: 5em
  padding: 5em
  background-color: lightblue
```

And we'll configure our application to serve this page at the root.

```diff
diff --git a/app/controllers/application_controller.rb b/app/controllers/application_controller.rb
index d83690e..99ee3ea 100644
--- a/app/controllers/application_controller.rb
+++ b/app/controllers/application_controller.rb
@@ -2,4 +2,7 @@ class ApplicationController < ActionController::Base
   # Prevent CSRF attacks by raising an exception.
   # For APIs, you may want to use :null_session instead.
   protect_from_forgery with: :exception
+
+  def index
+  end
 end
diff --git a/config/routes.rb b/config/routes.rb
index e92c058..43ccfc7 100644
--- a/config/routes.rb
+++ b/config/routes.rb
@@ -3,7 +3,7 @@ TourExample::Application.routes.draw do
   # See how all your routes lay out with "rake routes".
 
   # You can have the root of your site routed with "root"
-  # root 'welcome#index'
+  root 'application#index'
 
   # Example of regular route:
   #   get 'products/:id' => 'catalog#view'
```

If you're following along, go ahead and make sure everything is working so far.

```bash
rails server
```

Visit http://localhost:3000 and you should see that page we just added,
appropriately styled.

{% img /images/tour-1.png %}

## Darken the screen

During the tour, we want everything to be darkened,
so that the user's focus will be on the features being showcased
during the tour. Let's start there.
All that we need to accomplish this
is a fixed element that covers the page,
and has a black background and high (but not total) opacity.

```diff
diff --git a/app/views/application/index.haml b/app/views/application/index.haml
index e336443..92a257d 100644
--- a/app/views/application/index.haml
+++ b/app/views/application/index.haml
@@ -1,5 +1,7 @@
 -# app/views/application/index.haml
 
+#cover
+
 #example-page
   .one
     Part one
```

```sass
// app/assets/stylesheets/tour.sass

#cover
  position: fixed
  z-index: 1
  top: 0
  left: 0
  right: 0
  bottom: 0

  opacity: 0.8
  background-color: black
```


Check http://localhost:3000 and see how that looks now.
The whole page should remain dark, even when you scroll
up and down.

{% img /images/tour-2.png %}

## Annotating the page with tour information

Then we add legs of the tour by wrapping them
up with some additional tagging information.
In this example, I've annotated "part one" and
"part three" with tour information.
The whole thing gets wrapped in a `.tour` div,
the additional tour text is written in the `.callout` div,
and finally, the the highlighted feature is simply
nested inside the `.feature` div.

```haml
-# app/views/application/index.haml

#cover

#example-page
  .one
    .tour
      .callout
        This is part one
      .feature
        Part one
  .two
    Part two.
  .three
    .tour
      .callout
        This is part three
      .feature
        Part three
```

And we'll give it some style.

```sass
// append to app/assets/stylesheets/tour.sass

.tour
  position: relative
  background-color: inherit

  .callout
    position: absolute
    z-index: 2
    top: 0
    right: 0

    padding: 2em
    border: 1px solid white
    background-color: white
    color: black

  .feature
    position: relative
    z-index: 2

    border-radius: 5px
    background-color: inherit

```

Notice how we've cleverly
positioned the `.callout` absolutely in reference to the `.tour`.
This means that the `.feature` div will be rendered exactly
in the same position where it would have been without these
extra elements.
Also note how `.tour` and `.feature` inherit the background color
of the parent. This helps them pop out a little better.

Let's see how that looks so far.

{% img /images/tour-3.png %}

Well, it's not bad, but not great.
Notice how the callout overlaps the feature.
We'll improve the placement of the callout
when we get to scripting. In other words, now.

## One step at a time

The tour should take our user through features
one at a time. Let's throw in some scripting to
get that done.

```coffeescript
# app/assets/javascripts/tour.coffee

class @Tour
```

Using `class @Tour` is a little bit of sugar
that lets me use `new Tour()` in the haml file by attaching
the `Tour` class to the `window` object.
There are better namespacey ways to do this which are
beyond the scope of this blog post.

I've defined a few helpers in this class.
First, we want to be able to scroll the user gently
to a given element. This is easy with jQuery.

```coffeescript
  # smoothly scroll an element to the top
  prettyScroll = (e) ->
    $('html, body').animate {
      scrollTop: $(e).offset().top
    }, 500
```

Next, we need a way to initiate and terminate
each portion of the tour.

```coffeescript
  # embark on a particular leg of the tour
  summon = (tour) ->
    feature = $(tour).find '.feature'
    callout = $(tour).find '.callout'

    callout.show()
    feature.addClass 'on-tour'

    prettyScroll feature


  # complete a leg of the tour
  dismiss = (tour) ->
    feature = $(tour).find '.feature'
    callout = $(tour).find '.callout'

    callout.hide()
    feature.removeClass 'on-tour'
```

Pretty straightforward.
Now, we just need to give the user a way to actually
interact with the tour, and move from one leg to the next.
I chose to set this up by appending a link to each callout.

As an additional item of pre-processing, I've used javascript
to move the callout just below the feature, by calculating
that feature's height, and setting the callout css to be just below.

```coffeescript
  constructor: (cover, scope) ->
    scope ||= $('body')
    tours = $(scope).find('.tour')

    # attach 'next/okay' button to each leg of the tour
    tours.each (i, tour) ->
      callout = $(tour).find '.callout'
      feature = $(tour).find '.feature'
      upNext = tours[i + 1]

      advance = $('<a href="#"></a>')
      advance.addClass 'tour-next-button'
      advance.html(if upNext then 'next' else 'okay')
      advance.click (e) ->
        e.preventDefault()
        dismiss tour

        if upNext
          summon upNext
        else
          $('html, body').animate {
            scrollTop: 0
          }, 2000
          $(cover).fadeOut 2000

      callout.append $('<br />')
      callout.append $('<br />')
      callout.append advance

      # move the callout to just below the feature
      callout.css
        top: feature.height()

    # begin the tour
    $(cover).show()
    summon tours[0]
```

Now we can invoke this in the haml.

```haml
-# append to app/views/application/index.haml

:javascript
  $(function() { new Tour('#cover', '#example-page'); });
```

And that's just about it. Let's tweak the style so that
it leverages the `on-tour` class, and have the cover and
the callouts be hidden by default.

```diff
diff --git a/app/assets/stylesheets/tour.sass b/app/assets/stylesheets/tour.sass
index acb3466..e65b811 100644
--- a/app/assets/stylesheets/tour.sass
+++ b/app/assets/stylesheets/tour.sass
@@ -1,6 +1,8 @@
 // app/assets/stylesheets/tour.sass
 
 #cover
+  display: none
+
   position: fixed
   z-index: 1
   top: 0
@@ -16,6 +18,8 @@
   background-color: inherit
 
   .callout
+    display: none
+
     position: absolute
     z-index: 2
     top: 0
@@ -26,7 +30,7 @@
     background-color: white
     color: black
 
-  .feature
+  .feature.on-tour
     position: relative
     z-index: 2
```

## Nasty bug, easy fix

What happens if you can interact with the features
on tour? I'll tell you what can happen: nasty things.
Here's an example.

```diff
diff --git a/app/views/application/index.haml b/app/views/application/index.haml
index 6991ff1..447b87f 100644
--- a/app/views/application/index.haml
+++ b/app/views/application/index.haml
@@ -8,7 +8,10 @@
       .callout
         This is part one
       .feature
-        Part one
+        %p Part one
+
+        %p
+          %a{href: 'javascript:oops();'} A wild link has appeared!
   .two
     Part two.
   .three
@@ -20,3 +23,7 @@
 
 :javascript
   $(function() { new Tour('#cover', '#example-page'); });
+
+  function oops() {
+    $('.three').remove();
+  }
```

Now click the wild link, and try to complete the tour.
Whoops, you can't, because the final leg of the tour was removed
from the DOM!

CSS has a really cool and easy solution for this.

```diff
diff --git a/app/assets/stylesheets/tour.sass b/app/assets/stylesheets/tour.sass
index e65b811..86b61b2 100644
--- a/app/assets/stylesheets/tour.sass
+++ b/app/assets/stylesheets/tour.sass
@@ -31,8 +31,11 @@
     color: black
 
   .feature.on-tour
+    pointer-events: none
+
     position: relative
     z-index: 2
 
     border-radius: 5px
     background-color: inherit
```

Now no pointer events are allowed for a feature on tour,
and the continuity of the tour is saved. Neat.

## Exercises for the reader

* You may wish to lead the tour in a different order than the default
DOM traversal order discovered by jQuery. Change the coffeescript and add metadata
to the haml so that the tour order is configurable.
* If you nest one legs of the tour inside of the feature of another
leg of the tour, you will probably get weird behavior. Explain why this is,
and provide a solution.
