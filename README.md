About django-static
===================

What it does
------------

`django_static` is a Django app that enables as various template tags
for better serving your static content. It basically rewrites
references to static files and where applicable it does whitespace
optmization of the content. By making references to static content
unique (timestamp included in the name) you can be very aggressive
with your cache-control settings without ever having to worry about
upgrading your code and worrying about visitors using an older version.

The three template tags it enables are the following:

1. `staticfile` Takes the timestamp of the file, and makes a copy by
   symlinking as you define. You use it like this::
   
        <img src="{% staticfile "/images/foo.png" %}"/>
	
   and the following is rendered:
   
        <img src="/images/foo.123456789.png"/>
	
   ...assuming the epoch timestamp of the file is 123456789. 
   
2. `slimfile` Works the same as `staticfile` but instead of copying
   the file as a symlink it actually rewrites the file and compresses
   it through [slimmer](http://pypi.python.org/pypi/slimmer/). This of
   course only works for `.js` and `.css` files but it works
   wonderfully fast and is careful enough to not break things. The
   cool thing about doing this for `.css` files it finds all relative
   images inside and applies `staticfile` for all of them too. You use
   it just like `staticfile`:
   
        <script type="text/javascript"
          src="{% slimfile "/javascript/myscript.js" %}"></script>
	  
3. `slimcontent` is used to whitespace compress content right in the
   template. It requires a format paramter which can be `"js"`,
   `"css"` or `"html"`. So, for example for some inline CSS content
   you do this:
   
        <style type="text/css">
        {% slimcontent "css" %}
        h1, h2, h3 { 
	    font-face:'Trebuchet MS', Verdana, Arial; 
        }
        {% endslimcontent %}
        </style>
	
   ...and you get this:
   
        <style type="text/css">
        h1,h2,h3{font-face:'Trebuchet MS',Verdana,Arial}
        </style>
	
	
Configuration
-------------

`django_static` will be disabled by default. It's not until you set
`DJANGO_STATIC = True` in your settings module that it actually starts
to work for you. 

By default, when `django_static` slims files or makes symlinks with
timestamps in the filename, it does this into the same directory as
where the original file is. If you don't like that you can override
the save location by setting
`DJANGO_STATIC_SAVE_PREFIX = "/tmp/django-static"`

If you, for the sake of setting up your nginx/varnish/apache2, want
change the name the files get you can set
`DJANGO_STATIC_NAME_PREFIX = "/cache-forever"` as this will make it easier
to write a rewrite rule/regular expression that in
nginx/varnish/apache2 deliberately sets extra aggressive caching. 

Another option is to let django_static take care of setting your
`MEDIA_URL`. You could do this:

        <img src="{{ MEDIA_URL }}{% staticfile "/foo.png" %}"/>
	
But if you're feeling lazy and what django_static to automatically
take care of it set `DJANGO_STATIC_MEDIA_URL`. In settings.py:

        DJANGO_STATIC_MEDIA_URL = "http://static.example.com"
	
In your template:

        <img src="{% staticfile "/foo.png" %}"/>
	
And you get this result:

        <img src="http://static.example.com/foo.1247785534.png"/>


Advanced configuration with DJANGO_STATIC_FILE_PROXY
----------------------------------------------------

If you enable, in your settings, a variable called
`DJANGO_STATIC_FILE_PROXY` you can make all static URIs that
`django_static` generates go though one function. So that you, for
example, can do something with the information such as uploading to a
CDN. To get started set the config:

        DJANGO_STATIC_FILE_PROXY = 'mycdn.cdn_uploader_file_proxy'
        
This is expected to be the equivalent of this import statement:

        from mycdn import cdn_uploader_file_proxy
        
Where `mycdn` is a python module (e.g. `mycdn.py`) and
`cdn_uploader_file_proxy` is a regular python function. Here's the
skeleton for that function:

        def cdn_uploader_file_proxy(uri, **kwargs):
            return uri
            
Now, it's inside those keyword arguments that you get the juicy gossip
about what `django_static` has done with the file. These are the
pieces of information you will always get inside those keyword
argments:

        new = False
        checked = False
        changed = False
        notfound = False
        
The names hopefully speak for themselves. They become `True` depending
on what `django_static` has done. For example, if you change your
`foo.js` and re-run the template it's not `new` but it will be `checked`
and `changed`. The possibly most important keyword argument you might
get is `filepath`. This is set whenever `django_static` actually does
its magic on a static file. So, for example you might write a function
like this:

        on_my_cdn = {}

        def cdn_uploader_file_proxy(uri, filepath=None, new=False,
                                    changed=False, **kwargs):
            if filepath and (new or changed):
                on_my_cdn[uri] = upload_to_my_cdn(filepath)

            return on_my_cdn.get(uri, uri)


Custom converters
-----------------

CleverCSS can be integrated with django-static by adding the following in your
settings.py file:

	import clevercss
        DJANGO_STATIC_CONVERSIONS = {'.ccss': (clevercss, '.css')}

All `.ccss` files will be run through `clevercss.convert()` and the resulting
file extension will be changed to `.css`. Note that this currently works only
with `DJANGO_STATIC = True`.


Using Google Closure Compiler
-----------------------------

If you want to use the [Google Closure
Compiler](http://code.google.com/closure/compiler/) to optimize your
Javascript files you first have to download the compiler.jar file and
make sure your systam can run java. Suppose you download it in
/usr/local/bin, the set this variable in your settings.py file:

        DJANGO_STATIC_CLOSURE_COMPILER = '/usr/local/bin/compiler.jar'
        
If for some reason the compiler chokes on your Javascript it won't
halt the serving of the file but it won't be whitespace optimized and
the error will be inserted into the resulting Javascript file as a big
comment block.

Using the YUI Compressor
------------------------

The [YUI Compressor](http://developer.yahoo.com/yui/compressor/) is
both a Javascript and CSS compressor which requires a java runtime.
Just like the Google Closure Compiler, you need to download the jar
file and then set something like this in your settings.py:

        DJANGO_STATIC_YUI_COMPRESSOR = '/path/to/yuicompressor-2.4.2.jar'
	
If you configure the Google Closure Compiler **and** YUI Compressor,
the Google Closure Compiler will be first choice for Javascript
compression.

Using the slimmer
-----------------

[slimmer](http://pypi.python.org/pypi/slimmer/) is an all python
package that is capable of whitespace optimizing CSS, HTML, XHTML and
Javascript. It's faster than the YUI Compressor and Google Closure but
that speed difference is due to the start-stop time of bridging the
Java files.
	
How to hook this up with nginx
------------------------------

Read [this blog entry on
peterbe.com](http://www.peterbe.com/plog/serve-your-static-stuff-in-django-with-nginx)


	       
	
   
   