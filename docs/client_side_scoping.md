## ActiveRecord Scope Enhancement

When the client receives notification that a record has changed Synchromesh finds the set of currently rendered scopes that might be effected, and requests them to be updated from the server.  

To give you control over this process Synchromesh adds some features to the ActiveRecord `scope` and `default_scope` macros.  Note you must use the `scope` macro (and not class methods) for things to work with Synchromesh.

The additional features are accessed via the `:joins` and `:client` options.

The `:joins` option tells the synchromesh client which models are joined with the scope.  *You must add a `:joins` option if the scope has any data base join operations in it, otherwise if a joined model changes, synchromesh will not know to update the scope.*

The `:client` option provides the client a way to update scopes without having to contact the server.  Unlike the `:joins` option this is an optimization and is not required for scopes to work.

```ruby
class Todo < ActiveRecord::Base

  # Standard ActiveRecord form:
  # the proc will be evaluated as normal on the server, and as needed updates
  # will be requested from the clients

  scope :active, -> () { where(completed: true) }

  # In the simple form the scope will be reevaluated if the model that is
  # being scoped changes, and if the scope is currently being used to render data.

  # If the scope joins with other data you will need to specify this by
  # passing a relationship or array of relationships to the `joins` option.

  scope :with_recent_comments,
        -> { joins(:comments).where('comment.created_at >= ?', Time.now-1.week) },
        joins: ['comments'] # or joins: 'comments'

  # Now with_recent_comments will be re-evaluated whenever a Todo record, or a Comment
  # joined with a Todo change.

  # Normally whenever synchromesh detects that a scope may be effected by a changed
  # model, it will request the scope be re-evaluated on the server.  To offload this
  # computation to the client provide a client side scope method:

  scope :with_recent_comments,
        -> { joins(:comments).where('comment.created_at >= ?', Time.now-1.week) },
        joins: ['comments']
        client: -> { comments.detect { |comment| comment.created_at >= Time.now-1.week }

  # The client proc is executed on each candidate record, and if it returns true the record
  # will be added to the scope.

  # The client proc can take a single argument, and if present, the proc will receive the
  # the entire proposed new scope, which can then be filtered or sorted as needed.

  scope :sort_by_created_at,
        -> { order('created_at DESC') }
        client: -> (collection) { collection.sort { |a, b| b.created_at <=> a.created_at }}

  # To keep things tidy you can specify the server scope proc with the :server option

  scope :completed,
        server: -> { where(complete: true) }
        client: -> { complete }

  # The expressions in the joins array can be arbitrary sequences of relationships and
  # scopes such as 'comments.author'.  

  scope :with_managers_comments,
        server: -> { ... }
        joins: ['comments.author', 'owner']
        client: -> { comments.detect { |comment| comment.author == owner.manager }}}

  # You can also use the client, server, and joins option with the default_scope macro

  default_scope server: -> { where(deleted: false).order('updated_at DESC') }
                client: ->(c) { c.select { |r| !r.deleted }.sort { |a, b| b <=> a }

  # NOTE: it is highly recommend to provide a client proc with default_scopes.  Otherwise
  # every change is going to require a server interaction regardless of what other client procs
  # you provide.

end
```

#### How it works

Consider this scope on the Todo model

```ruby
scope :with_managers_comments,
      server: -> -> { joins(owner: :manager, comments: :author).where('managers_users.id = authors_comments.id').distinct },
      client: -> { comments.detect { |comment| comment.author == owner.manager }}
      joins: ['comments.author', 'owner']
```

The joins 'comments.author' relationship is inverted so that we have User 'has_many' Comments which 'belongs_to' Todos.

Thus we now know that whenever a User or a Comment changes this may effect our with_managers_comments scope

Likewise 'owner' becomes User 'has_many' Todos.

Lets say that a user changes teams and now has a new manager.  This means according to the relationships that the
User model will change (i.e. there will be a new manager_id in the User model) and thus all Todos belonging to that
User are subject to evaluation.

While the server side proc efficiently delivers all the objects in the scope, the client side proc just needs to incrementally update the scope.