---
layout: post
title: "RewriteRule examples and explanations"
date: 2014-01-09 19:52
comments: true
categories: apache, htaccess, mod_rewrite
---

####1. example

Our goals in this example:  

- Remove .php extensions. For example `filkor.org/hello.php` becomes `filkor.org/hello/`
- Add trailing slash to the given URL (if it's not there already)
- Redirect to /404.php if the file (resource) does not exist

You can find the solution below with a step-by-step explanation. 
Also, look at the  <a href="http://htaccessexample.filkor.org" target="_blank"><strong>DEMO</strong></a> I made for this example. 
Try out different URL-s to see what happens.

<pre class="prettyprint">
RewriteEngine On
RewriteBase /

# 1 
RewriteCond %{ENV:REDIRECT_END}  =1
RewriteRule ^                    -					  [L]

# 2 - remove php file extension on GETs
RewriteCond %{REQUEST_FILENAME} !-d
RewriteCond %{REQUEST_METHOD}   =GET
RewriteRule (.*)\.php$          $1/                      [L,R=301]

# 3 - add trailing slash
RewriteCond %{REQUEST_FILENAME} !-f
RewriteRule ^.*[^/]$            $0/                      [L,R=301]

# 4 - terminate if file exists. 
RewriteCond %{REQUEST_FILENAME} -f
RewriteRule ^                   -                        [L,E=END:1]

# 5 - terminate if directory/index.php exists.  Could be $1index.php, too, cause "/" already exists between them.
RewriteCond %{REQUEST_FILENAME}    -d
RewriteCond %{REQUEST_FILENAME}index.php    -f
RewriteRule ^(.*)(/?)$             $1/index.php          [L,E=END:1]

# 6 - resolve urls to matching php files 
RewriteCond %{DOCUMENT_ROOT}/$1.php  -f
RewriteRule ^(.*?)/?$              $1.php                [L,E=END:1]

# 7 - Anything else redirect to the 404 script.
RewriteRule ^                      404.php              [L,E=END:1]
</pre>

#### Solution explanation
As you can see I divided up the solution into 7 parts and now I will go through them one by one...

<!--more-->

<pre class="prettyprint">
# 1
RewriteCond %{ENV:REDIRECT_END}  =1
RewriteRule ^                    -					  [L]
</pre>

This is the #1. part. In a nutshell these two lines say: if the `REDIRECT_END` environment variable is set to 1 then completely stop the rewriting process.  

The second line here basically says "match the whole URL, don't rewrite anything" and because the [L] flag is present it says "stop here", so don't process any more RewriteCond or RewriteRule directives.   
Where does this `REDIRECT_END` variable come from? It's a bit tricky. I will tell you in a moment where does it come from, and why we need this particular variable.


<pre class="prettyprint">
# 2 - remove php file extension on GETs
RewriteCond %{REQUEST_FILENAME} !-d
RewriteCond %{REQUEST_METHOD}   =GET
RewriteRule (.*)\.php$          $1/                      [L,R=301]
</pre>

This is the part where we actually start manipulating the URL.  
The first condition says that the `%{REQUEST_FILENAME}` can't be a directory.   
`%{REQUEST_FILENAME}` is the full local filesystem path to the file or script matching the request. ([source](http://httpd.apache.org/docs/current/mod/mod_rewrite.html))  

The next condition says we don't do any rewrites if it's not a `GET` request. This condition is not necessary, but I like to put it in. If we request with `POST` for example, the .htacces file won't do anything (it's not clear at first but believe me. It stops at part 4).     
Let's say we request the URL `filkor.org/hello.php`. The rewritten URL will be  `filkor.org/hello/`.  
Now one more important thing here: we got 2 flags **[L,R=301]**.  

Within mod_rewrite (the Apache module) we differentiate two types of "redirections" (or "rewrites"):  
- **External redirection**: When you see the `[R]` flag (like above), then it means we are doing an external redirection. The URL is going to change in your browser. The .htaccess will be processed again from the beginning (with the new URL).

-**Internal redirection**: The main difference here is that the URL is *not* going to change in your address bar. However the .htaccess rules will be processed again (applying the new URL now). The URL will change only "internally" inside the Apache server, so to speak.   
We know that using an **[L]** flag it causes to stop processing the rule set (no more RewriteRule processing in this round), *moreover*, and this is the important part the **[L]** flag also causes an internal redirection. This means when you use an **[L]** flag it will restart processing the .htaccess rules with the new URL (until no actual rewrite happens).  

If this is not clear please write a comment and I will clarify this a bit more:) The most important thing about .htaccess to understand is when a rewite happens and there is an **[L]** flag then the whole process restarts from the beginning (part #1), now applying the new (rewritten) URL.


Sources: ([R flag](http://httpd.apache.org/docs/2.4/rewrite/flags.html#flag_r)), ([L flag](http://httpd.apache.org/docs/2.4/rewrite/flags.html#flag_l)), ([External, internal rewrites](http://httpd.apache.org/docs/current/rewrite/remapping.html))

<pre class="prettyprint">
# 3 - add trailing slash
RewriteCond %{REQUEST_FILENAME} !-f
RewriteRule ^.*[^/]$            $0/                      [L,R=301]
</pre>
Part #3 is pretty straightforward. We add a trailing slash if it's not there already and if `REQUEST_FILENAME` is not a file.  
We don't wanna add trailing slashes to the `.jpg` (or any static) files for example. `filkor.org/kitty.jpg/` would be fatal.  
From the flags you can see we're doing a permanent redirect here. Which means we start the processing again from the beginning with the new URL. 
Note that **[R]** should be used in conjunction with the **[L]** flag. 
See ([source](http://httpd.apache.org/docs/2.4/rewrite/flags.html#flag_r)).  


<pre class="prettyprint">
# 4 - terminate if file exists. 
RewriteCond %{REQUEST_FILENAME} -f
RewriteRule ^                   -                        [L,E=END:1]
</pre>

Terminate if file exists. For example if we requesting a `.jpg` file, or any static file.  
Look at the flags. We got an `[L]` flag and this causes an internal redirect. In the second flag we set up an environment variable called `END` and set it's value to 1.  
Now the tricky part: after the internal redirect, this variable get renamed by having "REDIRECT_" prepended to it's name. Yeah, thats annoying. And you couldn't find docs about this.  
This is where we arrived to part #1 again and to the `%{ENV:REDIRECT_END}` variable.  
If we'are having an `[L,E=END:1]` flag from now on, this means "stop rewriting the URLs, stop .htaccess processing". 

Further reading: [What causes the variable name to be prefixed with “REDIRECT_”?](http://stackoverflow.com/questions/3050444/when-setting-environment-variables-in-apache-rewriterule-directives-what-causes)


<pre class="prettyprint">
# 5 - terminate if directory/index.php exists.
RewriteCond %{REQUEST_FILENAME}    -d
RewriteCond %{REQUEST_FILENAME}index.php    -f
RewriteRule ^(.*)(/?)$             $1/index.php          [L,E=END:1]
</pre>

Nothing fancy in this part. If we request `filkor.org/mydir/` then serve `filkor.org/mydir/index.php` (while the URL remains `filkor.org/mydir/`).
Thats not an important part but it could be useful to be here. You see the `[L,E=END:1]` flag here with means terminate the rewriting process when we've served index.php. 

<pre class="prettyprint">
# 6 - resolve urls to matching php files 
RewriteCond %{DOCUMENT_ROOT}/$1.php  -f
RewriteRule ^(.*?)/?$              $1.php                [L,E=END:1]
</pre>

What we are doing here is to change the URL back *internally* to `hello.php` then terminate.  
So while you see in your address bar `filkor.org/about/` we serve `filkor.org/about.php` internally. Nice, isn't it?  

The RewriteCond is really interesting now.  
It might help to know that the RewriteCond is NOT processed first.  
Processing order is:

1. evaluate the RewriteRule ^(.*?)/?$ pattern  to see if "true"
2. if true, the ^(.*?)/?$ creates the **$1** backreference (which essentially holds the requested "filename"),
3. evaluate the RewriteCond; which checks if the "filename".php is really a file, 
4. redirect to a new URL internally (RewriteRule), then terminate (via the flags)

(One [further example](http://www.webmasterworld.com/apache/4330396.htm) for this feature)


<pre class="prettyprint">
# 7 - Anything else redirect to the 404 script.
RewriteRule ^                      404.php              [L,E=END:1]
</pre>

Internally redirects to `404.php`, what means no URL changing, just serve `404.php` while you see `filkor.org/whatever/` inside your browser's address bar.  
Maybe you will need a leading slash: [When is the leading slash (/) needed in mod_rewrite patterns?](http://webmasters.stackexchange.com/questions/27118/when-is-the-leading-slash-needed-in-mod-rewrite-patterns) 
depending on your Apache version.