
# **DEPRECATED, NO LONGER USED**

# Help System

One of the problems with our help links is they often times get out of date. We used to have html files, then we changed to jsps. We had

to update all of the links everywhere. It was a laborious thing to do.

[[https://bugzilla.redhat.com/show_bug.cgi?id=458413]]

A better mechanism would be to have a level of indirection where the help links would have just a key name that can be looked up in a map by a help system. When we need to change urls, we simply change them in a single place.
