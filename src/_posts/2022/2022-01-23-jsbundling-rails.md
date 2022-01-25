---
layout: post
title:  "Migrating to jsbundling-rails"
tags: rails, ruby, esbuild, webpacker
date:   2022-01-23 11:00:00 -0500
categories:
  - rails
  - webpacker
  - javascript
blurb: |
  Now that Webpacker is riding off into the sunset, many of us find ourselves switching over to using one of the solutions offered by jsbuilding-rails.
---

Now that [Webpacker is riding off into the sunset](https://github.com/rails/webpacker/commit/16bba5d6ca862b950a002f19c10d12e4bf51a87b#diff-b335630551682c19a781afebcf4d07bf978fb1f8ac04c6bf87428ed5106870f5), many of us find ourselves switching over to
using one of the solutions offered by [jsbuilding-rails](https://github.com/rails/jsbundling-rails).

With the move comes a much simplier javascript story, which is a welcome change for many of us,
but making the move can quickly lead to a few gotchas. Below I'll outline the few that I've run into
and how you can avoid them so you can ensure a smooth transition to using esbuild.

## Gotcha #1 - Importing CSS in your Javascript

This has been one of the most common problems I've noticed folks having. Say you're using a javascript
package that also includes CSS assets, esbuild does allow you to import those, but will name
the output file the same name as the root javascript file in `app/javascript` which in most cases will be `application.js`.

The result of that being two files output to `app/assets/build` , `application.js` and `application.css` - and that's where the hang-up occurs. If you are also using [cssbundling-rails](https://github.com/rails/cssbundling-rails) it will by default name it's output the same thing - `application.css`, causing a naming collision that will inevitably cause you to bang your head against the wall if you aren't sure what's going on.

There are two solutions here, one is renaming the output of your `cssbundling-rails` file, or renaming your `app/javascript/application.js` to something else, which in my case is what I did, making it `application-esbuild.js`. Now, esbuild will process your javascript and CSS and place two files named `application-esbuild.js` and `application-esbuild.css` into `app/assets/build`. Then all you need to do is make sure you include those files in the head of your `application.html.erb` file:

```erb
<!-- app/views/layout/application.html.erb -->

<%= stylesheet_link_tag "application", media: :all, data: { turbo_track: "reload" } %>

<%= stylesheet_link_tag "application-esbuild", media: :all, data: { turbo_track: "reload" } %>
<%= javascript_include_tag "application-esbuild", data: { turbo_track: "reload" } %>
```

## Gotcha #2 - Having Two Javascript Directories
If you're not starting fresh in Rails 7 there's good chance you've got files in `app/assets/javascript`.
If so you'll also want to make sure you've got sprockets setup to serve those assets. This can be done in `app/assets/config/manifest.js`, just make sure you've got the following line in the config:

```javascript
// app/assets/config/manifest.js

//= link_tree ../images
//= link_tree ../fonts
//= link_tree ../builds

// this is the key addition, referring to the file app/assets/javascript/application.js
//= link application.js
```
Of course you could also use `link_tree` or any of the other methods that sprockets provides in order
to properly include the necessary javascript files in `app/assets/javascript/`, but I think explicitly
including just one file helps keep things clean and easy to understand.

Also note that doing this also assumes that you've renamed your esbuild output file to `application-esbuild.js`
as mentioned above, otherwise you'd run into similar naming collision issues that Gotcha #1 covers.




