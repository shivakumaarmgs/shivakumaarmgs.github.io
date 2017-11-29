--- 
title: Rails Asset Pipeline 4.x
--- 
Ruby on Rails is a full-stack web development framework
written in Ruby. For a developer who just began working with Ruby on Rails, a
lot things will feel magical, like instance variables defined in controller
accessible by the views. Another such thing is the asset pipeline, but it
creates chaos at times especially when moving things to production. A simple
google search would yield a solution at that time, yet it wouldn’t feel
complete. This post it get a better understanding about the Rails asset
pipeline so it would feel complete.

## Understanding Rails Asset Pipeline

Code written inside `app/assets` and `vendor/assets` folders makes up for most of
the front-end in any Ruby on Rails application. Any javascript or css that we
write inside this folder is compiled and minified before it is served to the
browsers in production. All this is taken care automatically by the asset
pipeline which is shipped along with Rails by default. This asset pipeline is
implemented by the sprockets-rails gem. It even also allows us to write assets
in a different language or pre-processors, and compiles them to css and
javascript understandable by the browser.

A Ruby on Rails application at the time of creation is given with two manifest
files application.js and application.css. These two files by default contain
few lines of meaningful code which are deemed mandatory. Let us have a look at
the content of those files and try to figure out what are they for.

## Manifest files

```css 
/*
 * This is a manifest file that'll be compiled into application.css, which will include all the files
 * listed below.
 *
 * Any CSS and SCSS file within this directory, lib/assets/stylesheets, vendor/assets/stylesheets,
 * or vendor/assets/stylesheets of plugins, if any, can be referenced here using a relative path.
 *
 * You're free to add application-wide styles to this file and they'll appear at the top of the
 * compiled file, but it's generally better to create a new file per style scope.  
 *= require_self 
 *= require_tree .  
 */ 
```

This is the default css manifest file, which includes all the mentioned files
into a single application.css file. The require keyword here functions same as
the require keyword in ruby. It makes a file accessible within the file it is
called from. The require keyword looks for the mentioned files in app/assets/
folder first, if not found there it will look in the vendor/assets/ folder.

```
‘require_self’ and ‘require_tree .’
```

`require_self` loads all the content of that file where it is called from, in
our case the application.css into the line where its mentioned. `require_tree
.` loads all contents in the same folder where it is referenced into the line
where its mentioned. If we need to include all the files of a sub folder, use
`require_tree ./folder_name/`. This is helpful when using plain css. But if we
want to use sass/scss, to use global variables for colors etc, we need to
import the css file using scss method `@import` rather than `require`.

## Asset Paths

There is no big difference between `app/assets` and `vendor/assets` folder. By
convention it’s advised to use app/assets folder for custom css and javascript
files that we create and user vendor/assets folder for css and javascript of
any third party plugins.

The order in which assets are searched is as follows

``` 
/app/assets/images 
/app/assets/javascripts 
/app/assets/stylesheets
/vendor/assets/images 
/assets/javascripts 
/assets/stylesheet 
/[assets included by gems] 
```

## Asset Precompilation

To compile all the files and minify them we should run rake assets:precompile.
This command initiates a rake task which finds the files mentioned in
application.css, if they are written in scss/sass it is compile to css, then
adds them as a part of the application.css file. This application.css file is
linked to each page using `stylesheet_link_tag`, a rails view helper. Same
happens for application.js.

By default only `application.css` and `application.js` are the manifest files and
hence they are precompiled. But we might want to use one or more files as
manifest files for different layouts. In that case these files should be added
to the precompile array at config/initializers/asset.rb

```ruby
Rails.application.config.assets.precompile += ['dashboard.js', 'dashboard.css'] 
```

Here along with application.css and application.js files, dashboard.css and
dashboard.js files are considered manifest. So during precompile these files
are also precompiled. After precompilation these newly generated files are
placed in the public/assets folder. 

```
application-8cf9a6a42ecd8238b5d928c5ad7b4f4b440061daa2da0a0fe2016c7e92c9d955.css
application-ca682b893b6c32b8d053913db011cb78977866d8d1d6d95cf676e804adef35da.js
application-8cf9a6a42ecd8238b5d928c5ad7b4f4b440061daa2da0a0fe2016c7e92c9d955.css.gz
application-ca682b893b6c32b8d053913db011cb78977866d8d1d6d95cf676e804adef35da.js.gz
dashboard-fb0bd7de7060c10f28a74d784f046a519ef3b170645b00ee91a66953dc686554.css
dashboard-fb0bd7de7060c10f28a74d784f046a519ef3b170645b00ee91a66953dc686554.css.gz
```

### `public/assets` folder

This is the third folder where we can find assets in a rails application, but
adding assets here directly is not encouraged unless it’s a static asset and
doesn’t change at all. Also we won’t be able to reference any files placed
under this folder using rails url helpers. Assets are generally precompiled
only in production environment. If you wish to precompile assets in local run
the `rake assets:precompile RAILS_ENV=production` in local.

```
dashboard-fb0bd7de7060c10f28a74d784f046a519ef3b170645b00ee91a66953dc686554.css
dashboard-fb0bd7de7060c10f28a74d784f046a519ef3b170645b00ee91a66953dc686554.css.gz
```

If you take a look at the above file names they have random 64 characters
attached to them. This process is called fingerprinting, which makes the entire
filename depend on the content of the file. This technique is really helpful in
caching the assets. If the content of the file remains the same, the name of
files will also remain the same, hence it can be cached. If the content
changes, the name changes and so the cache.

## Sass Asset Url Helpers

Whenever we put some fonts, images in app/assets folder then try to access them
from css, scss files, it will work fine in development. But it most of the time
it breaks in production saying resource not found `/assets/bg-image.jpg`. This
happens especially for the fonts packages we include. This is because we would
have give the something like this for the background-image

```css
background-image: url('/images/header_background.jpg');
```

In production it will try to get the image from `/images/header_background.jpg`.
But there will be no files present in public folder as `header_background.jpg`
unless we place the file in public directory. We would normally place them in
app/assets or vendor/assets. So during precompilation it will be compiled and
fingerprint will be added to all the no js/css assets. Hence all image files,
font files will carry fingerprints in their name, hence this won’t work. 

```
header_background-5c2d30983dca95c015dda3f0a581ff39e9dbef809ec6290d1f8a771cc6a42e2b.png
company_logo-ad045423017cb12f23b6c2ccb746518586f0cb77080d7cb38b1800af6a4b4c73.png
```

Files compiled and added as shown above in the public folder. So to reference
this in the css we need to use asset path helpers provided by rails. Instead of
`url(‘/images/header_backgroung.png’)` use
`image-url(‘header_background.png’)`. After precompilation this will be added
seen as

```css 
background-image: url('/images/header_background-5c2d30983dca95c015dda3f0a581ff39e9dbef809ec6290d1f8a771cc6a42e2b.png');
```

I hope this post is informative enough about Rails asset pipeline. A better
understanding on asset pipeline will help you keep the front-end related code
more organized and maintainable. If you wish to explore further about asset
pipeline here is the link for official guide.
