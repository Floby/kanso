# Getting started

## Table of Contents

<ul class="toc">
    <li><a href="#installation">Installation</a></li>
    <li><a href="#starting_a_project">Starting a project</a></li>
    <li><a href="#deployment">Deployment</a></li>
    <li><a href="#describing_the_data">Describing the data</a></li>
    <li><a href="#querying_the_data">Querying the data</a></li>
    <li><a href="#rendering_pages">Rendering pages</a></li>
    <li><a href="#users_and_permissions">Users and permissions</a></li>
    <li><a href="#embedded_types">Embedded types</a></li>
    <li><a href="#adding_forms">Adding forms</a></li>
    <li><a href="#going_it_alone">Going it alone</a></li>
</ul>


<h2 id="installation">Installation</h2>

Install the most recent _stable_ version of
[node](http://nodejs.org/#download), then clone
[kanso](https://github.com/caolan/kanso) from GitHub.
Fetch the relevant submodules by doing the following in the cloned directory:

<pre><code class="no-highlight">git submodule init
git submodule update</code></pre>

You are then ready to install:

<pre><code class="no-highlight">make && sudo make install</code></pre>


### Using NPM

If you already have node.js installed, and you're using npm
(Node Package Manager), then you can install by simply doing the following:

<pre><code class="no-highlight">sudo npm install -g kanso</code></pre>


### Why does Kanso need node.js?

Node.js is used for the command-line tools only. With Kanso, the end result is
a pure [CouchApp](http://couchapp.org) you can host using CouchDB alone.
Using node to write the associated tools allows us to do some powerful things
by interpreting the JavaScript of your application.


### CouchDB version

As of release 0.0.7, Kanso **only supports CouchDB 1.1.0 or higher**. This is due
to a number of fixes in CouchDB 1.1.0 for CommonJS modules which make Kanso
faster and helps to keep the framework code clean. If you're running an older
version of CouchDB you'll want to upgrade it before continuing with this guide.


<h2 id="starting_a_project">Starting a project</h2>

For this tutorial we'll be making a fairly typical blog, with posts,
comments and users. Some familiarity with CouchDB is expected, but I'll
try to explain the concepts as we go.

To create a new project skeleton, enter the following command:

<pre><code class="no-highlight">kanso create myblog</code></pre>

This creates a number of files and directories representing a basic
project structure. We'll look more closely at the generated files later,
but for a brief overview, the directories fall into the following categories:

<pre><code class="no-highlight">myblog
  |- lib             CommonJS modules which define your app
  |- static          Static files such as jQuery and CSS
  |- templates       HTML templates used by the app
  |- kanso.json      Project configuration</code></pre>

This structure is a merely a guide and you are free to place files
wherever it makes sense, provided you update the configuration settings in
kanso.json.


<h2 id="deployment">Deployment</h2>

It might seem early in the tutorial to start talking about deployment, but
since your app needs to be hosted by CouchDB to run, we're going to cover it
now.

The first thing to mention is that <em>Kanso apps are just CouchApps</em>. This
means they can be hosted directly from your CouchDB instance, without any additional
tier or services.

For the rest of this tutorial we're going to assume you have CouchDB running
on <a href="http://localhost:5984">http://localhost:5984</a>
(the default settings). If you haven't got a local copy of CouchDB running,
go do that now.


### Pushing

The <code>push</code> command uploads your app to a CouchDB database. You can find
help on any of the commands used in this tutorial by typing <code>kanso help</code>.

Let's push the new project to your CouchDB instance, and check that everthing
works. To do this, enter the following command from your project directory:

<pre><code class="no-highlight">kanso push http://localhost:5984/myblog</code></pre>

<img src="images/deployment1.png" alt="pushing myblog" />

This creates a new database called 'myblog' and uploads your app to it.
If you now visit [http://localhost:5984/myblog/\_design/myblog/\_rewrite/](http://localhost:5984/myblog/_design/myblog/_rewrite/),
you should see the following welcome page:

<img src="images/deployment2.png" alt="pushing myblog" />

This is the location to use when testing your app. Don't worry about the
ugly URL, we can fix this later using
<a href="http://wiki.apache.org/couchdb/Virtual_Hosts">virtual hosts</a>.


<h2 id="describing_the_data">Describing the data</h2>

Now we've got our new project up and running, let's think about the data
structures this project requires. Because CouchDB is schemaless, we can make
lots of changes as we go without having to worry about doing ALTER TABLE.

The only problem is, it can quickly get complicated when manually validating
documents and checking permissions. Thankfully, kanso provides a powerful
document validation and permissions tool in the form of Type definitions.


### Creating types

In the skeleton project we created earlier, there's a file called
<code>lib/types.js</code>. This is the conventional place to export types.
Remember, the files in the <code>lib</code> directory are 
<a href="http://wiki.commonjs.org/wiki/Modules/1.1.1">CommonJS modules</a>.
This means anything added to the <code>exports</code> object in this file is
made visible to other modules.

Let's create a type for describing blog posts. Add the following to
<code>lib/types.js</code>.

<pre><code class="javascript">var Type = require('kanso/types').Type,
    fields = require('kanso/fields'),
    widgets = require('kanso/widgets');


exports.blogpost = new Type('blogpost', {
    fields: {
        created: fields.timestamp(),
        title: fields.string(),
        text: fields.string({
            widget: widgets.textarea({cols: 40, rows: 10})
        })
    }
});
</code></pre>

That should be fairly self-explanatory.
The first argument to the <code>Type</code> constructor is the type name
which must be unique, the second argument is an object describing fields
and other options.

Now we've added a basic type, let's push these changes to the server:

<img src="images/deployment1.png" alt="pushing myblog" />

<h3>The Admin App</h3>

As a useful way to play with data types, kanso provides a basic admin app.
You can push this app to the same database you pushed the 'myblog' project to
by using the following command:

<pre><code class="no-highlight">kanso pushadmin http://localhost:5984/myblog</code></pre>

<img src="images/describing_the_data1.png" alt="push the admin app" />

If you now visit
[http://localhost:5984/myblog/\_design/admin/\_rewrite/](http://localhost:5984/myblog/_design/admin/_rewrite/)
you should see the following page:

<img src="images/describing_the_data2.png" alt="admin app - apps list" />

Because you can have multiple apps running on a single database you will be
presented with a list of available applications. Click the 'myblog' app.

<img src="images/describing_the_data3.png" alt="admin app - app overview" />

Here, we are presented with a list of types. Currently, we just have the one
'blogpost' type. Clicking on this will show you a list of existing blog posts:

<img src="images/describing_the_data4.png" alt="admin app - type list" />

Of course, we haven't added any yet. Let's try adding one now. Click the
'Add blogpost' link.

<img src="images/describing_the_data5.png" alt="admin app - adding a type" />

As you can see, the admin app has read the field settings we defined earlier
and presented us with a sensible form for adding blog posts. You'll notice the
<code>timestamp</code> field is missing. This is because it's automatically
populated with the current time.

Create a blog post by filling out the form with some test data and clicking
the create button.

<img src="images/describing_the_data6.png" alt="admin app - type added" />

As you can see, kanso has automatically populated the 'type' and 'created'
fields. The 'type' field is automatically added, and reserved for use by
kanso so we can identify the definition to validate against.


<h2 id="querying_the_data">Querying the data</h2>

In CouchDB, you query your documents using map / reduce functions called
'views'. If you've never written a CouchDB view before, you can read up on
it in the <a href="http://guide.couchdb.org/draft/views.html">Definitive Guide</a>.

Adding a view to a kanso app couldn't be simpler. Just open up the
<code>lib/views.js</code> file and add the following example view:

<pre><code class="javascript">exports.blogposts_by_created = {
    map: function (doc) {
        if (doc.type === 'blogpost') {
            emit(doc.created, doc.title);
        }
    }
};</code></pre>

**Push the app**, then revisit the admin interface at
[http://localhost:5984/myblog/\_design/admin/\_rewrite/myblog](http://localhost:5984/myblog/_design/admin/_rewrite/myblog)
  you'll see the view we created now appears:

<img src="images/querying_the_data1.png" alt="admin app - now showing a view" />

Clicking on the view shows the current results. You should see the document
we created earlier keyed by its created at timestamp:

<img src="images/querying_the_data2.png" alt="admin app - view results" />

Clicking on a row will take you to its corresponding document.


<h2 id="rendering_pages">Rendering pages</h2>

In CouchDB, rendering custom representations of your data is achieved using
[List and Show functions](http://wiki.apache.org/couchdb/Formatting_with_Show_and_List).
Kanso is no different, although it will also run these functions client-side
wherever possible, giving users a more responsive interface while also
providing a fallback for search-engines.

Writing code in this way can be very efficient and shows where you might be
missing non-js support. However, at first it might seem a little odd. Work
through the examples and bear in mind that all list and show functions could
be run client-side or server-side at any time!


### List functions

List functions format the results of a view. We're going to use a list
function to show a list of blog posts on the homepage, ordered by the
date they were created. Add the following to <code>lib/lists.js</code>:

<pre><code class="javascript">var templates = require('kanso/templates');


exports.homepage = function (head, req) {

    start({code: 200, headers: {'Content-Type': 'text/html'}});

    // fetch all the rows
    var row, rows = [];
    while (row = getRow()) {
        rows.push(row);
    }

    // generate the markup for a list of blog posts
    var content = templates.render('blogposts.html', req, {
        rows: rows
    });

    if (req.client) {
        // being run client-side, update the current page
        $('#content').html(content);
        document.title = 'MyBlog';
    }
    else {
        // being run server-side, return a complete rendered page
        return templates.render('base.html', req, {
            content: content,
            title: 'MyBlog'
        });
    }

};</code></pre>

For simplicity, we're pre-fetching all the rows at once, on large views
you'll want to return content for each row before fetching the next.

You'll notice this list function detects if its running client-side by
checking the client property on the request object. When run client-side,
it will update the DOM instead of returning a new HTML document.

Since this is such a common pattern when writing Kanso apps, you can use a
convenient short-hand of returning an object with title and content properties,
Kanso will then do the appropriate thing for the current environment:

<pre><code class="javascript">var templates = require('kanso/templates');


exports.homepage = function (head, req) {

    start({code: 200, headers: {'Content-Type': 'text/html'}});

    // fetch all the rows
    var row, rows = [];
    while (row = getRow()) {
        rows.push(row);
    }

    // generate the markup for a list of blog posts
    var content = templates.render('blogposts.html', req, {
        rows: rows
    });

    return {title: 'MyBlog', content: content};

};</code></pre>

We referenced two templates in the first example: 'base.html' and
'blogposts.html'. The first was created as part of the project skeleton,
but the second we need to create ourselves.
Create a new template at <code>templates/blogposts.html</code> with the
following content:

<pre><code>&lt;h1&gt;My Blog&lt;/h1&gt;

{?rows}
  &lt;ul&gt;
  {#rows}
    &lt;li&gt;&lt;a href="{baseURL}/{id}"&gt;{value}&lt;/a&gt;&lt;/li&gt;
  {/rows}
    &lt;/ul&gt;
  {:else}
  &lt;p&gt;No blog posts&lt;/p&gt;
{/rows}</code></pre>

For more information on the template format used by kanso, see the
<a href="http://akdubya.github.com/dustjs/">Dust website</a>. Basically,
this will render a list of blog post titles.

One thing worth noting is the use of {baseURL} in the template. This variable
is automatically made available to templates by kanso, and refers to the
current 'root' level URL. That would currently mean
'/myblog/\_design/myblog/\_rewrite', but that may change if you use
a virtual host in the future. For now, be sure to prefix all
<strong>application</strong> URLs with the base URL.


### Rewrites

Next, we need to hook this view up to a URL. CouchDB uses
<a href="http://wiki.apache.org/couchdb/Rewriting_urls">rewrites</a>
to point arbitrary URLs at CouchDB functions and again, kanso is no exception.
Let's change the root URL from the welcome page to our new list of blog posts.
Edit <code>lib/rewrites.js</code> to look like the following:

<pre><code class="javascript">module.exports = [
    {from: '/static/*', to: 'static/*'},
    {from: '/', to: '_list/homepage/blogposts_by_created'},
    {from: '*', to: '_show/not_found'}
];</code></pre>

This uses our new list function in combination with the view query we created
earlier. <strong>Push the app</strong>, then open it in your browser:
<a href="http://localhost:5984/myblog/_design/myblog/_rewrite/">http://localhost:5984/myblog/\_design/myblog/\_rewrite/</a>.

<img src="images/rendering_pages1.png" alt="The new homepage" />

Of course, clicking on the blog post will give a 404 Not Found error, since
we've not implemented the view for a single blog post yet. Let's do that now.


### Show functions

Show functions are like list functions, only they display a single document
rather than the results of a view. We're going to create a show function
for our blog posts, open up <code>lib/shows.js</code> and take a look.
Currently, it will contain the welcome page we recently unhooked from our
rewrite rules. You can now remove that, and replace it with the following:

<pre><code class="javascript">var templates = require('kanso/templates');


exports.blogpost = function (doc, req) {
    return {
        title: doc.title,
        content: templates.render('blogpost.html', req, doc)
    };
};</code></pre>

You'll notice we're using the short-hand method used in the list function
earlier, by returning an object with title and content properties.
Next we need to add the <code>blogpost.html</code> template:

<pre><code>&lt;h1&gt;{title}&lt;/h1&gt;

&lt;p&gt;{text}&lt;/p&gt;</code></pre>

And update the rewrite rules:

<pre><code class="javascript">module.exports = [
    {from: '/static/*', to: 'static/*'},
    {from: '/', to: '_list/homepage/blogposts_by_created'},
    {from: '/:id', to: '_show/blogpost/:id'},
    {from: '*', to: '_show/not_found'}
];</code></pre>

__Push the app__ and try clicking the blog post link now.
You should see a page similar to the following:

<img src="images/rendering_pages2.png" alt="Single blog post page" />

If you check your JavaScript console in Chrome or Firebug, you should notice
that clicking on the blog post link on the homepage causes the next request to
be handled client-side:

<img src="images/rendering_pages3.png" alt="Single blog post page" />

However, turning off JavaScript should provide you with the same experience.
Just with your CouchDB server doing the work of rending pages.

As your sites become more complex you may want to make additional requests
to the server when a list or show function is run client-side. This means
we could show all blogposts with the same tag in a side-bar, or perhaps show
links to related posts. These features would then be available only to people
with JavaScript turned on, and browsers without JavaScript will fall-back to
the core content you see now.


<h2 id="users_and_permissions">Users and permissions</h2>

Currently, anyone can add a blog post to the site, even if they're not
logged-in. This probably isn't ideal, so let's look at protecting the ability
to add blog posts.

CouchDB provides its own users and roles system, which defines who has
access to which database. Kanso can also use this to authorize or
reject changes to the database on a document-level. The great thing about
this is that our authentication system is already set-up and ready to go!


### validate\_doc\_update

CouchDB authorizes changes to a document using the
<a href="http://guide.couchdb.org/draft/validation.html">validate\_doc\_update</a>
function defined in your app. This function accepts the new document,
the previous version of the document (if available), and the user's details as
arguments and throws an Error if the update should be rejected.

Kanso also supports this system as the underlying way to authorize
changes. However, this method is quite low-level. The simplicity of this approach
is both its attraction and its downfall. With complex applications comprising of
many types, each with it's own validation and authorization requirements, your
validate\_doc\_update function quickly becomes unwieldy.

Kanso allows you to mix-and-match, by providing powerful tools
on a type or even field-level, while still
supporting the simplicity of the validate\_doc\_update approach.

If you look at <code>lib/validate.js</code> in the project skeleton we
generated, you'll notice kanso has already added a
validate\_doc\_update function which just calls the kanso function
<code>types.validate_doc_update</code>. This runs the permissions
defined in your app's type and field definitions, so it's
important not to remove this line if you're using this feature.

<pre><code class="javascript">var types = require('kanso/types'),
app_types = require('./types');


module.exports = function (newDoc, oldDoc, userCtx) {
    types.validate_doc_update(app_types, newDoc, oldDoc, userCtx);
};</code></pre>

If you also want to add permissions directly into this validate\_doc\_update
function, you can do that too.


### End of the admin-party

By default, CouchDB has no users and everyone has the special "\_admin" role.
This makes playing around with CouchDB (and Kanso) easy when you first
install, but obviously isn't very secure. If you haven't done so already,
put an end to the
[admin party](http://guide.couchdb.org/draft/security.html#party)
by visiting the [Futon admin interface](http://localhost:5984/_utils/)
and adding a new admin user:

<img src="images/users_and_permissions2.png" alt="fixing admin party" />


### Type-level permissions

Currently, we want to protect the ability to add, remove and update the
blogpost type. To do this we simply update our type definition to include a
permissions section:

<pre><code class="javascript">var Type = require('kanso/types').Type,
    fields = require('kanso/fields'),
    widgets = require('kanso/widgets'),
    permissions = require('kanso/permissions');


exports.blogpost = new Type('blogpost', {
    permissions: {
        add:    permissions.hasRole('_admin'),
        update: permissions.hasRole('_admin'),
        remove: permissions.hasRole('_admin')
    },
    fields: {
        created: fields.timestamp(),
        title: fields.string(),
        text: fields.string({
            widget: widgets.textarea({cols: 40, rows: 10})
        })
    }
});</code></pre>

__Push the app__ (you may now be prompted for a username and
password), if you then __logout__ and go to the [blogposts page](http://localhost:5984/myblog/_design/admin/_rewrite/myblog/types/blogpost)
in the admin app. When you try to delete a blogpost, you should see the following:

<img src="images/users_and_permissions1.png" alt="admin app - role required" />

The same should happen when editing or adding new blog posts. Logging back in
as an admin user should allow you to complete all of these operations again.

A short note on pushing as a user: if you include a username or both a username
and password in the database URL, these credentials will be used to push the app.

<pre><code class="no-highlight">kanso push http://user:password@localhost:5984/dbname
kanso push http://user@localhost:5984/dbname</code></pre>

You may not want to expose the password in your shell history so I'd
recommend just putting the username in the URL, and kanso will know to
prompt you for a password before attempting to push.


<h2 id="embedded_types">Embedded types</h2>

Now we move on to one of Kanso's more advanced features: Embedded types.

Beacuse it often makes sense to 'de-normalize' your data in CouchDB you'll
often find yourself embedding one document directly into another.
Any Type system that's going to work properly will also need to support this
technique.

This can get quite tricky once you consider the permissions and validation
requirements involved. Consider a situation where you'd like to embed
comments on a blog post document. One user may be allowed to edit a blogpost,
while another may only be allowed to comment on it. However, both users need to be able to
update the document itself, since the comments are embedded within the blogpost.

Using embedded types, Kanso helps you handle these situations much more easily
and define exactly which parts of a document can be edited by a user.

Whether or not embedding one type within another document is a good idea
often depends on the specific circumstances. Comments on blog posts for
example, may be best added separately to the blog post document if you
expect there to be a lot of commenting activity, otherwise you run the
risk of conflicts, and CouchDB will have to write the whole list of
comments every time you add a new one.

For the sake of this example however, we're going to assume comments are
very infrequent and that we prefer to make them available on the blog post
document itself.


### Creating the comment type

Before we can embed comments within the blogpost type, we need to define
the comment type itself. Add the following to <code>lib/types.js</code>,
__BEFORE__ the blogpost type:

<pre><code class="javascript">exports.comment = new Type('comment', {
    permissions: {
        add: permissions.loggedIn(),
        update: permissions.usernameMatchesField('creator'),
        remove: permissions.usernameMatchesField('creator')
    },
    fields: {
        creator: fields.creator(),
        text: fields.string({
            widget: widgets.textarea({cols: 40, rows: 10})
        })
    }
});</code></pre>

You'll notice we're using some new permissions for this type,
first <code>loggedIn</code> simply checks the user is logged in (not
anonymous) before being allowed to add a comment. Then,
<code>usernameMatchesField</code> checks that the current user is the same
as the user defined in the creator field. This stops users from editing or
deleting other users comments.

The <code>creator</code> field is also new, and set's itself to the current
user's name when a new comment is added. After creation, this field becomes
uneditable.


### Embedding a type

To embed a list of comments in a blogpost, update the blogpost type definition
to include the following comments field:

<pre><code type="javascript">exports.blogpost = new Type('blogpost', {
    permissions: {
        add:    permissions.hasRole('_admin'),
        update: permissions.loggedIn(),
        remove: permissions.hasRole('_admin')
    },
    fields: {
        created: fields.timestamp(),
        title: fields.string({
            permissions: {
                update: permissions.hasRole('_admin')
            }
        }),
        text: fields.string({
            widget: widgets.textarea({cols: 40, rows: 10}),
            permissions: {
                update: permissions.hasRole('_admin')
            }
        }),
        comments: fields.embedList({
            type: exports.comment,
            required: false
        })
    }
});</code></pre>

You'll notice we've changed the permissions a bit. Previously the type had add,
update and remove permissions at the top-level which all checked for the '\_admin'
role. That meant you couldn't create, delete or make any changes to a blogpost
without being a CouchDB admin.

Since we're embedding comments within the blogpost document, we need to give update
access to any logged-in user, so they can add comments directly into it. However,
we still don't want them to change the title or text of the blogpost, so we've
added field-level permissions to protect those.

The <code>embedList</code> field will use the permissions of the comment Type
definition, meaning any logged-in user can add a comment, but you can only edit or
delete comments you created.

This is an example of the flexibility of the Kanso Types and permissions system.
De-normalizing your data often means you'll need some fine-grained permissions
to protect specific parts of a document. Because Kanso was written with CouchDB in
mind, we can design specifically for these kinds of problems.

__Push the app__, and visit the admin page for adding blogposts:
<a href="http://localhost:5984/myblog/_design/admin/_rewrite/myblog/types/blogpost/add">http://localhost:5984/myblog/\_design/admin/\_rewrite/myblog/types/blogpost/add</a>

<img src="images/embedded_types1.png" alt="new comments button" />

There is now an add a comment button, clicking on this will open a modal
dialog for embedding a comment.

<img src="images/embedded_types2.png" alt="adding comments" />

Continue to add a new blog post with some example comments, rembering that
you'll need to be logged in as an admin user to add new blog posts.

<img src="images/embedded_types3.png" alt="adding new blog post with comments" />


### Updating templates

Let's update the templates to handle the newly embedded comments.
Edit <code>templates/blogpost.html</code> to look like the following:

<pre><code>&lt;h1&gt;{title}&lt;/h1&gt;

&lt;p&gt;{text}&lt;/p&gt;

{?comments}
  &lt;h2&gt;Comments&lt;/h2&gt;

  &lt;ul&gt;
  {#comments}
    &lt;li&gt;
      &lt;strong&gt;{creator} says: &lt;/strong&gt;
      {text}
    &lt;/li&gt;
  {/comments}
  &lt;/ul&gt;
{/comments}</code></pre>

__Push the app__, and visit it's home page:
<a href="http://localhost:5984/myblog/_design/myblog/_rewrite/">http://localhost:5984/myblog/\_design/myblog/\_rewrite/</a>

<img src="images/embedded_types4.png" alt="app homepage" />

You should see the new blog post you created listed on the homepage.
Clicking on it should show it using the new template, complete with the
comments you added earlier:

<img src="images/embedded_types5.png" alt="showing blog post with comments" />


<h2 id="adding_forms">Adding forms</h2>

So far, we've been using the Kanso admin app to add documents, but let's
assume we want to write our own interface and forms. Kanso provides some useful
form-rendering code (also used by the admin app) which allows you to quickly
and easily create forms based on Type definitions.

First, we're going to create a form for adding new blog posts.
Add the following to the end of <code>templates/blogposts.html</code>:

    <p><a href="{baseURL}/add">Add new</a></p>

Then update <code>lib/rewrites.js</code> to look like this:

<pre><code class="javascript">module.exports = [
    {from: '/static/*', to: 'static/*'},
    {from: '/', to: '_list/homepage/blogposts_by_created'},
    {from: '/add', to: '_show/add_blogpost'},
    {from: '/:id', to: '_show/blogpost/:id'},
    {from: '*', to: '_show/not_found'}
];</code></pre>

Let's create a new show function to display the form. Add the following to 
<code>lib/shows.js</code>:

<pre><code class="javascript">exports.add_blogpost = function (doc, req) {
    // render the markup for a blog post form
    var content = templates.render('blogpost_form.html', req, {
        form_title: 'Add new blogpost'
    });

    return {title: 'Add new blogpost', content: content};
};</code></pre>

And add a new template for the page at <code>templates/blogpost_form.html</code>.

    <h1>{form_title}</h1>

__Push the app__, and visit it's home page:
<a href="http://localhost:5984/myblog/_design/myblog/_rewrite/">http://localhost:5984/myblog/\_design/myblog/\_rewrite/</a>

<img src="images/adding_forms1.png" alt="Check the add new link" />

Clicking 'Add new' should display the following page:

<img src="images/adding_forms2.png" alt="Check the add new rewrite" />

So far so good. Let's add a form to the page. In <code>lib/shows.js</code>
add the following new modules to the requires at the top:

<pre><code class="javascript">var templates = require('kanso/templates'),
    forms = require('kanso/forms'),
    types = require('./types');</code></pre>

Then update <code>exports.add_blogpost</code> to look like this:

<pre><code class="javascript">exports.add_blogpost = function (doc, req) {
    var form = new forms.Form(types.blogpost, null, {
        exclude: ['created', 'comments']
    });

    // render the markup for a blog post form
    var content = templates.render('blogpost_form.html', req, {
        form_title: 'Add new blogpost',
        form: form.toHTML(req)
    });

    return {title: 'Add new blogpost', content: content};
};</code></pre>

You'll notice we're using the new <code>forms</code> module to construct a
form object using the blogpost Type definition we created earlier. We are
excluding the created and comments fields, and just displaying the fields we need.

Next we render the form as HTML using <code>form.toHTML(req)</code>, and pass it
to the template. In <code>templates/blogpost_form.html</code> we need to add the
following:

    <h1>{form_title}</h1>

    <form method="POST" action="">
      <table>
        {form|s}
      </table>
      <input type="submit" value="Add" />
    </form>

__Push the app__, and visit the add blogpost page:
<a href="http://localhost:5984/myblog/_design/myblog/_rewrite/add">http://localhost:5984/myblog/\_design/myblog/\_rewrite/add</a>

<img src="images/adding_forms3.png" alt="Viewing the new form" />

Great, but if you try submitting the form, nothing happens! Thats because we need
to add an <a href="http://wiki.apache.org/couchdb/Document_Update_Handlers">update function</a> for saving changes to a document.

<h3>Update functions</h3>

Add the following to <code>lib/updates.js</code>:

<pre><code class="javascript">var templates = require('kanso/templates'),
    forms = require('kanso/forms'),
    utils = require('kanso/utils'),
    types = require('./types');


exports.add_blogpost = function (doc, req) {
    var form = new forms.Form(types.blogpost, null, {
        exclude: ['created', 'comments']
    });

    // parse the request data and check validation and permission functions
    form.validate(req);

    if (form.isValid()) {
        // the form is valid, save the document and redirect to the new page
        return [form.values, utils.redirect(req, '/' + form.values._id)];
    }
    else {
        // the form is not valid, so render it again with error messages
        var content = templates.render('blogpost_form.html', req, {
            form_title: 'Add new blogpost',
            form: form.toHTML(req)
        });
        // return null as the first argument so the document isn't saved
        return [null, {content: content, title: 'Add new blogpost'}];
    }
};</code></pre>

Update functions expect an array as the return value, the first argument is the
document to create or update, the second argument is the response to send to the
client. This function should look similar to the show function we created for
displaying the add blog post form, only with the addition of form validation.

After creating the form, we call <code>form.validate</code> with the request
object. This will populate the form with the request data and any relevant error
messages if the data is invalid.

We then check the validity of the form and either reject or accept the new document.
Note, that this is mostly to provide a nice response to the user and is not
required for security. Ultimately, an unacceptable document would be rejected by
validate\_doc\_update, even if this function were to accept it.

Next, we need to hook this update function up to a URL using rewrites. Edit
<code>lib/rewrites.js</code> to look like the following:

<pre><code class="javascript">module.exports = [
    {from: '/static/*', to: 'static/*'},
    {from: '/', to: '_list/homepage/blogposts_by_created'},
    {from: '/add', to: '_update/add_blogpost', method: 'POST'},
    {from: '/add', to: '_show/add_blogpost'},
    {from: '/:id', to: '_show/blogpost/:id'},
    {from: '*', to: '_show/not_found'}
];</code></pre>

We've added a new rewrite for <code>/add</code>, only this time we specify a HTTP
method to match. This means when we do <code>GET /add</code> we'll run the show
function, and when we do <code>POST /add</code> we'll run the update function.

Let's try submitting the form now. Be sure to login as an admin user using either
futon or the Kanso admin app (on the same hostname you're using with the app, no
good logging in on localhost if your accessing the app on 127.0.0.1).

__Push the app__, and visit the add blogpost page:
<a href="http://localhost:5984/myblog/_design/myblog/_rewrite/add">http://localhost:5984/myblog/\_design/myblog/\_rewrite/add</a>

After submitting the form you should be redirected to the list of blogposts and
you should see your new blogpost added to the end.


<h2 id="going_it_alone">Going it alone</h2>

Congratulations on finishing your first (basic) Kanso app! It's time for you
to strike out on your own, and explore the possibilities of a whole new way
to write web-apps. If you create something cool, please share it on the
<a href="https://github.com/caolan/kanso/wiki/Sites-using-Kanso">Kanso wiki</a>.

You can <strong>report issues</strong> using the
<a href="https://github.com/caolan/kanso/issues">GitHub issues page</a>.

Don't forget to check out the <a href="../api/index.html">Docs</a> and other
available <a href="index.html">Guides</a>.
