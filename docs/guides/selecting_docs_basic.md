
# Introduction

In the getting started guide, we walked through the construction of a
simple weblog application. In this guide, we'll extend that application -- 
n the process introducing some additional widgets, some additional methods of
referring to external documents, and some ways to take more control over the
presentation of your data in Kanso.

We'll start by recalling the example application that was introduced in the
getting started guide. For purposes of brevity, we've intentionally omitted data
related to permissions and security. The application containes two types: a
blogpost type -- which is comprised of a timestamp, title, and body of text --
and a comment type, which is embedded directly inside of the blogpost type using
an embedList field.

<pre><code class="javascript">var Type = require('kanso/types').Type,
    fields = require('kanso/fields'),
    widgets = require('kanso/widgets');

exports.comment = new Type('comment', {
    fields: {
        creator: fields.creator(),
        text: fields.string({
            widget: widgets.textarea({cols: 40, rows: 10})
        })
    }
});

exports.blogpost = new Type('blogpost', {
    fields: {
        created: fields.createdTime(),
        title: fields.string(),
        text: fields.string({
            widget: widgets.textarea({cols: 40, rows: 10})
        }),
        comments: fields.embedList({
            type: exports.comment
        })
    }
});
</code></pre>

If you haven't yet completed the getting started guide, we recommend that
you do so before proceeding. To simplify this guide, we'll be using Kanso's
administration application to work with our type definitions. The administration
application generates simple add, update, and delete screens, based upon the types
you define in your application's lib/types.js file. After every change to types.js,
we'll push the application and administrative interface to CouchDB:

<pre>
<code class="no-highlight">kanso push myblog</code><br />
<code class="no-highlight">kanso pushadmin myblog</code>
</pre>

The type definitions above model a denormalized version of a one-to-many
relationship: a blogpost contains potentially many comments, and each comment
belongs to exactly one blogpost. By definition, comments cannot be shared
between posts, and updating a single comment requires that the blogpost
containing it be updated well.

This arrangement is likely suitable for low-volume blog comments: comments are
likely to be unique from one another, don't typically need to be counted or
grouped, and don't have any particular utility outside of the context of the
blogpost they're attached to.

Suppose that we're frequent travellers, and that we write frequently as we're
visiting places away from home. We'd like to have the option of tagging a blog
post with a location. In this case, the same location will almost certainly be
shared between several blogposts, and there's some substantial utility to be had
in normalizing this relationship -- names of places are easily misspelled, we
may want to keep extra information about the places we've visited, and it's
quite useful to be able to count the number of posts made from a particular
location, or group posts by location. We'll explore this feature in the next few
sections.


# Dissecting the {embedList} Field Type

Before we start extending the tutorial application, it's worth delving a bit
deeper to examine how the embedList type works. When you instansiate an
embedList type without specifying any options, it automatically fills in some
default values that drive the data presentation and editing process.

When you write:

<pre><code class="javascript">exports.blogpost = new Type('blogpost', {
    fields: {
        ...,
        comments: fields.embedList({
            type: exports.comment
        })
    }
});
</code></pre>

...Kanso fills in some additional options automatically, effectively expanding
the full type definition to this:


<pre><code class="javascript">exports.blogpost = new Type('blogpost', {
    fields: {
        ...,
        comments: fields.embedList({
            type: exports.comment,
            widget: widgets.embedList({
                sortable: false,
                widget: widgets.defaultEmbedded(),
                actions: {
                    add: {
                        module: 'kanso/actions',
                        callback: 'modalDialog',
                        options: {
                            widget: widgets.embedForm({
                                type: exports.comment
                            })
                        }
                    },
                    edit: [
                        'kanso/actions', 'modalDialog', {
                            widget: widgets.embedForm({
                                type: exports.comment
                            })
                        }
                    ],
                    del: function () {
                        /* No default action */
                    }
                }
            })
        }),
    }
});
</code></pre>

The above definition looka a bit complicated, so we'll walk through each of the
options above one-by-one.

The embedList field type, like all other types in Kanso, uses a widget to
control its presentation. The default widget for the embedList field type is,
unsurprisingly, called the embedList widget. This arrangement, though appearing
redundant, allows Kanso developers to create alternate ways of working with
lists -- without necessarily affecting the way data is stored.

The embedList widget (two lines below the comments label, above) employs yet
another separate widget to draw each list item. This second "inner" widget
(defaultEmbedded, unless modified) is invoked repeatedly by the "outer"
embedList widget, and emits an HTML representation of a single list item.  The
inner widget is not responsible for drawing controls related to management of
the list itself. The embedList widget manages the list's add, edit, delete, and
reorder events, and contains logic to dispatch these events to an appropriate
handler.

Though this "inner" widget is free to expose different functionality based upon
its embedding context, the widget API is designed so that widgets can be used in
a list without any modification. The embed widget -- which is now simply an
alias for a single-element version of embedList -- uses the same approach as
embedList to draw it list item. 

The embedList widget, abstractly, implements either a set data structure or a
list data structure, depending upon the options provided to it. If the sortable
option is set to true, the embedList widget becomes a list, and displays
controls suitable for adjusting the list order. If sortable is false, the widget
becomes a set -- though the order of items is currently preserved, this may not
always be the case.

When a user asks Kanso to modify an embedList -- e.g. by pressing the add
button, by deleting an item, or by modifying the list order -- the embedList
widget uses a list of actions to decide how to handle the user's request. This
list of actions is in essence just a list of callbacks, each of which is
paired with the name of an action. Though ordinary functions are accepted, there
are two additional syntaxes for describing actions that ae typically easier to
use:

  * The first syntax, shown for the 'add' action above, is an object notation.
    When kanso encounters an object inside of a list of actions it (i) loads the
    commonjs module specified by the object's {module} property; (ii) using that
    module, looks up the exported method referred to by the object's {callback}
    property (a string); and then (iii) invokes that method, providing the
    object's {options} property to the method as its first argument.

  * The second syntax, shown for the 'edit' action above, is simply an
    abbreviated form of the first syntax. In this form, the action is specified
    as a three-item array -- element zero is the {module} property from above,
    element one is the {callback} peoperty, and element two is the {options}
    property.

Kanso provides a number of built-in actions that can be used with the embedList
widget; the default is {modalDialog}.

The {modalDialog} action uses the jQuery simplemodal library to display a modal
pop-up "window" (actually a regular page element). This space is typically used
to solicit information from the user -- typically information that's required to
continue with the requested action. The {modalDialog} action handles the
display, updating, and dismissal of the dialog element, but leaves the contents
of the dialog to be drawn by -- you guessed it, a widget. This is the inner-most
'widget' option in the above type definition.

By default, the {modalDialog} action uses the {embedForm} widget to ask the user
for information about an instance of an embedded type. The embedForm widget
requires a single parameter named 'type', which should be a Kanso type
definition -- exactly like the one accepted by fields.embedList. This type is
(and, in most typical cases, should remain) set to the same type definition that
the embedList uses for its 'type' parameter. After all, embedForm is providing
an interface to edit the very items in that list.

Thankfully, most of these defaults are filled in for you by Kanso, allowing you
to quickly create workable embedded lists. In the next section, we'll change
some of these options to more closely control the type's default presentation.


# Widget: Document Selector

The document selector widget -- known as {documentSelector} inside of Kanso -- is
a built-in widget that allows the user to select from (and optionally store
references to) external documents returned from a CouchDB view. The
documentSelector provides functionality to implement some of the more
relationally-normalized schema patterns you may be familiar with.

We'll now shift our focus back to extending the tutorial application, first by
describing our idea of a location using a type definition:

<pre><code class="javascript">exports.location = new Type('location', {
    fields: {
        name: fields.string({ required: true }),
        city: fields.string({ required: false }),
        visits: fields.number({ required: false })
    }
}</code></pre>

We've decided to our locations using simple names, but also track a bit of
information about the locations themselves -- the city each location can be
found in, and a count of the number of times we've made a trip to each.

After saving this new type definition and pushing the administrative application
to CouchDB, we can create a few locations that we frequent. The results of our
editing session is shown below in JSON:

[
    { name: 'Coffee Shop', city: 'Portland', visits: 39 },
    { name: 'Fountain', city: 'Chicago', visits: 3 },
    { name: 'Museum', city: 'Paris', visits: 2 },
    { name: 'Home' },
    { name: 'Art Museum', city: 'Portland', visits: 3 },
    { name: 'Train', city: 'San Francisco', visits: 4 },
    { name: 'Hockey Game', city: 'Vancouver', visits: 6 },
    { name: 'Pioneer Square', city: 'Portland', visits: 12 },
    { name: 'Farmers Market', city: 'Austin', visits: 8 },
    { name: 'Cafe', city: 'Amsterdam' },
    { name: 'Space Needle', city: 'Seattle', visits: 1 }
]

We'll now go ahead and use the documentSelector, in combination with the embed
widget, to reference a single location from each blogpost. Note that this
differs from the default behaviour of embed, in that there is a master list of
locations stored independently of the blogpost type:

<pre><code class="javascript">
exports.blogpost = new Type('blogpost', {
    fields: {
        ...
        location: fields.embed({
            type: exports.location,
            widget: widgets.documentSelector({
                db: 'myblog', viewName: 'locations'
            }),
            actions: {
                add: false, edit: false, del: false
            }
        })
    }
});
</code></pre>

The {viewName} option tells our widget how to retrieve a list of locations. In
this case, we'll use the results from a view named 'locations'.  The {db}
option tells the documentSelector where to find the 'locations' view --
specifying this option isn't strictly necessary, but will ensure that the
widget works from within the administrative application.

Before previewing our changes, we'll create a simple view for the
documentSelector to use by adding the following to our application's
lib/views.js file:

<pre><code class="javascript">exports.locations = { 
    map: function (doc) { 
        if (doc.type === 'location') { 
            emit(doc._id, doc.name); 
        } 
    } 
};
</code></pre>

Previewing our changes in the administrative application, we now notice a
drop-down box, which allows us to select from the list of locations we have on
file.

If you rename a location via the admininstraitv application, however, you'll
notice that this change doesn't immediately affect the value of the new blogpost
field. When enclosed in an embed field, the documentSelector embeds the entire
location by value -- that is, a copy of the entire location record is stored
within the blogpost, and the blogpost must be updated to modify the location's
name. The document selector will display the new name when the post is edited,
but the location stored within the blogpost document will not change until the
post is re-saved.

To remedy this, we can change our type definition to store a *reference* to a
location, rather than a copy of the entire location record:

<pre><code class="javascript">
exports.blogpost = new Type('blogpost', {
    fields: {
        ...
        /* Changed data type to 'string' */
        location: fields.string({
            widget: widgets.documentSelector({
                db: 'myblog', viewName: 'locations',
                /* Addition of two new options */
                useJSON: false, storeEntireDocument: false
            })
        })
    }
});
</code></pre>

With the above modifications, Kanso will now save only the CouchDB identifier of
the location, and will store it in a string field. Since this stores a reference
of a location instead of a copy, renaming a location will immediately rename it
everywhere, including inside of each blogpost instance. If you're coming to
CouchDB from a relational database system, these are probably the semantics that
are most familiar.

Kanso also has a built-in reference type that can provide some additional flexibility
in more advanced use cases. We can achieve an equivalent effect by writing:

<pre><code class="javascript">exports.blogpost = new Type('blogpost', {
    fields: {
        ...
        /* Changed outer data type and widget to 'embed'.
           Changed inner data type to a 'reference' type,
           which is constructed to refer to a 'location' */

        location: fields.embed({
            type: types.reference({
                type: exports.location
            }),
            widget: widgets.embedList({
                singleton: true,
                widget: widgets.documentSelector({
                    db: 'myblog', viewName: 'locations',
                    storeEntireDocument: false,
                    /* Modified one option */
                    useJSON: true
                }),
                actions: {
                    add: false, edit: false, del: false
                }
            })
        })
    }
});
</code></pre>

In this case the reference is stored as a special object, but is not
functionally different from the string case.


# Using {documentSelector} With {embedList}

Suppose that rather than keeping track of a single location per post, we'd like
to track a whole itinerary for each post -- i.e. an ordered list of all of the
places we travelled while drafting the post. Suppose that we also didn't
question the sanity of simultaneous walking and blogging.

We can accomplish this with a relatively simple modification of our {blogpost} type
definition.

<pre><code class="javascript">exports.blogpost = new Type('blogpost', {
    fields: {
        ...
        /* Changed field name to 'itinerary';
           Changed field type to an 'embedList' */

        itinerary: fields.embedList({
            type: types.reference({
                type: exports.location
            }),
            widget: widgets.embedList({
                /* Added sortable option */
                sortable: true,
                widget: widgets.documentSelector({
                    db: 'myblog', viewName: 'locations',
                    storeEntireDocument: false,
                    useJSON: true
                }),
                actions: {
                    add: false, edit: false, del: false
                }
            })
        })
    }
});
</code></pre>

Finally, here's an example that puts together many of the things we've learned
so far. The following type definition stores a list of locations by value, allowing
the user to edit each one in place using the same documentSelector control.
After you have the following example working, try allowing the user to edit the
details of an embedded location -- employing an {embedForm} in place of the
{documentSelector} in the 'edit' action.

<pre><code class="javascript">exports.blogpost = new Type('blogpost', {
    fields: {
        ...
        itinerary: fields.embedList({
            /* Changed type back to by-value storage */
            type: exports.location,
            widget: widgets.embedList({
                sortable: true,
                widget: widgets.documentSelector({
                    db: 'myblog', viewName: 'locations',
                    storeEntireDocument: true, useJSON: true
                }),
                /* Added 'add' and 'edit' actions */
                actions: {
                    add: [ 'kanso/actions', 'modalDialog', {
                        widget: widgets.documentSelector({
                            db: 'myblog', viewName: 'locations',
                            storeEntireDocument: true, useJSON: true
                        })
                    } ],
                    edit: [ 'kanso/actions', 'modalDialog', {
                        widget: widgets.documentSelector({
                            db: 'myblog', viewName: 'locations',
                            storeEntireDocument: true, useJSON: true
                        })
                    } ],
                    del: false
                }
            })
        })
    }
});
</code></pre>

Though this last example is functional, there's considerable repetition in the
type definition. Further, we'd like the 'Add' and 'Edit' buttons to have some
additional functionality to offer -- ideally, they'd have the ability to edit
the currently-selected document via the _id reference.

We'll explore these issues in the next guide.

