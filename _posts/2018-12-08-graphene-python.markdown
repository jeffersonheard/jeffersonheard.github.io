---
layout: post
title:  "GraphQL in Python with Graphene"
comments: true
date:   2018-12-08 13:54:00 -0500
categories: python graphql
---

[Graphene](https://graphene-python.org/) is probably the best library for creating GraphQL endpoints in Python.  It's actively developed. It has pretty complete helper libraries for [SQLAlchemy], [Django]'s ORM, and [MongoDB].  It's relatively easy to get something simple going. That said, Graphene's documentation leaves a *lot* to be desired. It's easy to get something simple going in GraphQL using their documentation, but getting something hardened, production ready, and capable is another story. 

If you're starting out in Graphene, go to their website and get through the tutorial. I'll not repeat it here. But if you're wondering about more advanced topics like:

* Integrating Graphene with Flask and a SQL database
* Pagination
* Handling permissions and authentication.
* Filtering of results, not just structure.

then stay here. For this post I'll be using [Flask] and SQLAlchemy. Django is relatively simple to get going, and SQLAlchemy is a bit more involved, so I think it'll be more illustrative. 

# Why GraphQL?

For me, GraphQL lets my backend and frontend developers work together with less friction about data contracts. You still have to have them, and you still have to optimize for them, but there is an opportunity to "start with the kitchen sink." and iterate. You'll begin with a non-performant but correct query and work toward a performant one that exposes the data that is needed by your apps and integration partners. It's just a smoother process in general. 

I personally don't use GraphQL for mutations, so I won't be writing about them in this post. I still use REST and RPC-style endpoints for those. Our developers don't like the syntax for mutations, and to them it *seems* easier to write buggy code with mutations vs. REST. If you want to use mutations, I'd suggest you start with [Marc-Andre Giroux](https://medium.com/@__xuorig__/graphql-mutation-design-anemic-mutations-dd107ba70496)'s post on graphql mutation design and branch out from there.

# I got through the initial tutorial. Now what?

The Graphene tutorial gets you through creating a really basic schema and querying it. But first thing's first. GraphQL is supposed to be about interoperation! We want to expose it on a web-based API. Flask is relatively simple to get going, and if all you want is an API, Django's got a lot more "built in" than you really need. Although it's hard to find, luckily for us there's a [Flask+SQLAlchemy tutorial](https://docs.graphene-python.org/projects/sqlalchemy/en/latest/tutorial/) on the main Graphene website. 

There's a bug in it, by the way. Well, there's a bug in Graphene-SQLAlchemy. Rename your `EmployeeConnection` class to anything but `EmployeeConnection` and it will work. Turns out Graphene-SQLAlchemy automatically creates a bunch of connection classes when introspecting relationships, and you will create a name clash if you name it the same thing that Flask-SQLAlchemy does. Other than the bug, follow that tutorial and then come back here. Your first question is probably, "Wait, what are `Connection`s, `relay.Node` and why does everything in my code look different than it did in the basic tutorial?" At least that was *my* first question.

> From here on out, I recommend you use the standalone electron client for GraphiQL instead of the one built into Flask. If you have Homebrew and Cask installed on a Mac, you can simply type `brew cask install graphiql` and you're off to the races. [Otherwise, follow the instructions for your platform here](https://electronjs.org/apps/graphiql).

# Connections and Nodes

Shortly after GraphQL came out, Relay became a popular way to structure GraphQL schemas. Relay organizes GraphQL results as nodes and edges. It's a bit more involved to parse, but it's useful for us because it also defines a standard, automatic and efficient way to paginate results. 

At the end of the tutorial, your first query looks like this:

{% highlight graphql %}
{
  allEmployees {
    edges {
      node {
        id
        name
        department {
          name
        }
      }
    }
  }
}
{% endhighlight %}

If you enter that into GraphiQL, you'll get something like this:

{% highlight javascript %}
{
  "data": {
    "allEmployees": {
      "edges": [
        {
          "node": {
            "id": "RW1wbG95ZWU6MQ==",
            "name": "Peter",
            "department": {
              "name": "Engineering"
            }
          }
        },
        {
          "node": {
            "id": "RW1wbG95ZWU6Mg==",
            "name": "Roy",
            "department": {
              "name": "Engineering"
            }
          }
        },
        {
          "node": {
            "id": "RW1wbG95ZWU6Mw==",
            "name": "Tracy",
            "department": {
              "name": "Human Resources"
            }
          }
        }
      ]
    }
  }
}
{% endhighlight %}

Let's dissect that. First off, the "id" is definitely *not* your primary key. It's a "node id", which is a unique id based on both the node classname and the primary key, which means that for your entire API it should uniquely identify the object itself, not merely the object with respect to the collection it's in.  This can be important, depending on your application.

First, how would we get an employee object with that id? It's not obvious. At first glance, I'd try to filter the collection by id with a parameter like so:

{% highlight graphql %}
{
  allEmployees(id:"RW1wbG95ZWU6MQ==") {
    edges {
      node {
        id
        name
        department {
          name
        }
      }
    }
  }
}
{% endhighlight %}

The `id` field isn't available to query on, though! We just get an error. We have to add the Employee node to the Query class:

{% highlight python %}
class Query(graphene.ObjectType):
    node = relay.Node.Field()

    employee = relay.Node.Field(Employee)
    # Allows sorting over multiple columns, by default over the primary key
    all_employees = SQLAlchemyConnectionField(EmployeeConn)
    # Disable sorting over this field
    all_departments = SQLAlchemyConnectionField(DepartmentConn, sort=None)
{% endhighlight %}

Then we can change our GraphQL query to hit employee directly:

{% highlight graphql %}
{
  employee(id:"RW1wbG95ZWU6MQ==") {
    name
    department {
      name
    }
  }
}
{% endhighlight %}

yields

{% highlight json %}
{
  "data": {
    "employee": {
      "name": "Peter",
      "department": {
        "name": "Engineering"
      }
    }
  }
}
{% endhighlight %}

just like we expect!

What if we want a department instead, and a list of employees?  Add `department` to the Query class the same way we did `employee`. Whereas the many-to-one relationship above does not the department node encapsulated in an "edges" structure, the one-to-many relationship will. This means that we can paginate inner results too!

{% highlight graphql %}
{
  department(id:"RGVwYXJ0bWVudDox") {
    name
    employees {
      edges {
        node {
          name
          hiredOn
        }
      }
    }
  }
}
{% endhighlight %}

yields 

{% highlight json %}
{
  "data": {
    "department": {
      "name": "Engineering",
      "employees": {
        "edges": [
          {
            "node": {
              "name": "Peter",
              "hiredOn": "2018-12-08T20:38:30"
            }
          },
          {
            "node": {
              "name": "Roy",
              "hiredOn": "2018-12-08T20:38:30"
            }
          }
        ]
      }
    }
  }
}
{% endhighlight %}

# Pagination

By default, a Graphene query will return *all* rows. This is a *fantastic* way to DOS yourself. In fact, you should set up server side limiting of the number of GraphQL results - something else that is non-obvious about setting up Graphene. But first, let's at least let the client be nice to us and understand how the client would use pagination.

So how do we paginate? First we have to get an opaque cursor, and then we have to use it. Cursors and pagination are supported not by the `Node` subclass but by the `Connection` subclass. Anywhere you have a `Connection`, you can use pagination, even in the nested portion of a query! It helps to understand the `pageInfo` structure. Let's go back to our `allEmployees` example and add `cursor` and `pageInfo` to it:

{% highlight graphql %}
{
  allEmployees {
    pageInfo {
      startCursor
      endCursor
      hasNextPage
      hasPreviousPage
    }
    edges {
      cursor
      node {
        id
        name
        department {
          name
        }
      }
    }
  }
}
{% endhighlight %}

Note the new `pageInfo` section and the `cursor` inside `edges`. These give us what we need to figure out where we are in the result set. Here are the results:

{% highlight json %}
{
  "data": {
    "allEmployees": {
      "pageInfo": {
        "startCursor": "YXJyYXljb25uZWN0aW9uOjA=",
        "endCursor": "YXJyYXljb25uZWN0aW9uOjI=",
        "hasNextPage": false,
        "hasPreviousPage": false
      },
      "edges": [
        {
          "cursor": "YXJyYXljb25uZWN0aW9uOjA=",
          "node": {
            "id": "RW1wbG95ZWU6MQ==",
            "name": "Peter",
            "department": {
              "name": "Engineering"
            }
          }
        },
        {
          "cursor": "YXJyYXljb25uZWN0aW9uOjE=",
          "node": {
            "id": "RW1wbG95ZWU6Mg==",
            "name": "Roy",
            "department": {
              "name": "Engineering"
            }
          }
        },
        {
          "cursor": "YXJyYXljb25uZWN0aW9uOjI=",
          "node": {
            "id": "RW1wbG95ZWU6Mw==",
            "name": "Tracy",
            "department": {
              "name": "Human Resources"
            }
          }
        }
      ]
    }
  }
}
{% endhighlight %}

Every `Connection` gives us the ability to add `cursor` and `pageInfo` objects to our queries, and allows us to use them by supplying arguments to the `Connection` in the GraphQL query. The arguments we get are `before` and `after` which take a cursor string, and `first` and `last` which are integers that supply the length of the page. If you're iterating forward through the collection, you would supply the last cursor string in the previous page to the `after` argument. Here's an example:

{% highlight graphql %}
{
  allEmployees(after:"YXJyYXljb25uZWN0aW9uOjA=", first:1) {
    pageInfo {
      startCursor
      endCursor
      hasNextPage
      hasPreviousPage
    }
    edges {
      cursor
      node {
        id
        name
        department {
          name
        }
      }
    }
  }
}
{% endhighlight %}

which yields: 

{% highlight json %}
{
  "data": {
    "allEmployees": {
      "pageInfo": {
        "startCursor": "YXJyYXljb25uZWN0aW9uOjE=",
        "endCursor": "YXJyYXljb25uZWN0aW9uOjE=",
        "hasNextPage": true,
        "hasPreviousPage": false
      },
      "edges": [
        {
          "cursor": "YXJyYXljb25uZWN0aW9uOjE=",
          "node": {
            "id": "RW1wbG95ZWU6Mg==",
            "name": "Roy",
            "department": {
              "name": "Engineering"
            }
          }
        }
      ]
    }
  }
}
{% endhighlight %}

which is the second node in the set. Voila! 

But Jeff... ANYONE CAN READ MY DATA!!!

# Authentication and permissions checking

The most basic case is that authenticated users can see everything in your GraphQL API. If you are this lucky, I envy you. The following snippet courtesy GitHub user Microldea is [buried in Issue #17 on GraphQL-Flask's](https://github.com/graphql-python/flask-graphql/issues/17#issuecomment-363346190) GitHub will use "traditional" auth: 

{% highlight python %}
def auth_required(fn):
    def wrapper(*args, **kwargs):
        session = request.headers.get(AUTH_HEADER, '')
        # Do some authentication here maybe...
        return fn(*args, **kwargs)
    return wrapper

def graphql_view():
    view = GraphQLView.as_view(
        'graphql',
        schema=schema,
        graphiql=True,
        context={
            'session': DBSession,
        }
    )
    return auth_required(view)

app = Flask(__name__)
app.debug = True
app.add_url_rule(
    '/graphql',
    view_func=graphql_view()
)
{% endhighlight %}

There is a lot of support in the GraphQL community for using JWT tokens for auth. There are a [ton](http://cryto.net/~joepie91/blog/2016/06/13/stop-using-jwt-for-sessions/) [of](http://cryto.net/~joepie91/blog/2016/06/19/stop-using-jwt-for-sessions-part-2-why-your-solution-doesnt-work/) [articles](https://dzone.com/articles/stop-using-jwts-as-session-tokens) on why you shouldn't do this for security reasons, but it is straightforward and for internal applications it may be appropriate in some cases. If you choose to do this, look through the thread I linked above and there are plenty of examples. 

The most flexible way to do this is with the `flask_login` extension. This will let you use whatever auth system you want. I use OAuth2.

{% highlight python %}
from flask_login import login_required

def graphql_view():
    view = GraphQLView.as_view('graphql', schema=schema, context={'session': db.session},
                               graphiql=True)
    return login_required(view)

{% endhighlight %}

Now at least your endpoint will require authentication to work properly. GraphQL purists will note however that a failed auth with this method will give you an HTTP 401. I don't really consider this a problem, but strict GraphQL standards adherence would require that you return the error as a JSON string inside of a 200 response (because we in browser-land have enough layers in our protocols to make an Austrian pastry chef blush). You could achieve this with a Graphene middleware:

{% highlight python %}
class AuthorizationMiddleware(object):
    def resolve(self, next, root, info, **args):
        if current_user.is_anonymous:
            raise werkzeug.exceptions.Unauthorized()
        else:
            return next(root, info, **args)

app.add_url_rule(
    '/graphql',
    view_func=GraphQLView.as_view(
        'graphql',
        schema=schema,
        graphiql=True, # for having the GraphiQL interface
        middleware=[AuthorizationMiddleware()]
    )
)
{% endhighlight %}

The really nice thing about this is that it can be extended in the case where some queries or fields are unavailable to the logged in user:

{% highlight python %}
class AuthorizationMiddleware(object):
    def resolve(self, next, root, info, **kwargs):
        if current_user.is_anonymous:
            raise werkzeug.exceptions.Unauthorized()
        elif info.field_name == 'allEmployees' and not some_permission_predicate():
            raise werkzeug.exceptions.Forbidden()
        else:
            return next(root, info, **kwargs)
{% endhighlight %}

That may be alright for many people -- you *could* structure your queries so that all the results of a query are safe for the users of that query. But for very complex sets of permissions that may not work. In that case, your middleware can set permissions and the acting user for the query, and you can consume that by overriding the Connection's get_query() method or a node's get_node() method.

{% highlight python %}
class FlaskAuthorizationMiddleware(object):
    def resolve(self, next, root, info, **kwargs):
        if current_user.is_anonymous:
            raise werkzeug.exceptions.Unauthorized()
        elif info.field_name == 'allEmployees' and not some_permission_predicate():
            raise werkzeug.exceptions.Forbidden()
        else:
            context = info.context
            context.query_user = current_user
            return next(root, info, **kwargs)

# in the node class
class Employee(relay.Node):
    class Meta:
        node = Employee

    def resolve_name(self, info):
        if can_view_names(info.context.query_user):
            return self.name
        else:
            return None

{% endhighlight %}

Why not just use `current_user` inside `get_query()`? You could, and if you never expect to call your GraphQL query in any way other than through the web application context, then by all means. However, if you are using GraphQL as a means to communicate between microservices you may want to be flexible on the way you obtain the user object. Something like Celery, for example, doesn't have a notion of `current_user` and will have to have a different way to construct it.

# Filtering results

So it turns out that Graphene and Graphene-SQLAlchemy has no way out of the box to filter the records that are returned by issuing a query.  How do you do it? By adding arguments to the query and handling those arguments in the Connection's `get_query()` method. You can take three approaches: 

* Handle arguments on a Connection by Connection basis, leading each query to be custom, and you have to enforce consistency through standards
* Introspect the fields on SQLAlchemy tables and use a base class or a metaclass to construct arguments for you for every node type. 
* Create a single argument, which accepts some kind of a DSL string (probably JSON) and constructs a filter from the string.

Each have their advantages. The first is easier to implement, at the cost of more development time for each new query, and it allows for precise control over how arguments are interpreted and what's exposed. The second, once you have it set up makes all queries act similar to each other, provides for a predictable interface for your users, and allows a medium level of control over how to filter. Don't forget to add whitelisting, blacklisting. and requirement of fields or you will allow your users to spam your servers with inefficient queries. 

The last one is the most flexible, but requires careful attention to design or your users will be baffled about how to construct filters and coding against your API will be error prone. Forcing your users to learn a SQL-Like or MongoDB Query-Like JSON syntax is generally a bad idea, especially since it should be a bit more limited to prevent inefficient queries from consuming excess server resources. 

The first system is the only one that can be explained in a short section of a blog post, so let's focus on that. Because the get_query method is a part of `SQLAlchemyConnectionField`, we forego our `relay.Connection` subclass for subclassing that. It changes the structure a bit:

{% highlight python %}
class DepartmentConn(SQLAlchemyConnectionField):
    @classmethod
    def get_query(cls, model, info, **args):
        print(args)
        print
        if 'name' in args:
            return model.query.filter_by(name=args['name'])
        return model.query

    
class Query(graphene.ObjectType):
    node = relay.Node.Field()

    department = relay.Node.Field(Department)
    employee = relay.Node.Field(Employee)
    # Allows sorting over multiple columns, by default over the primary key
    all_employees = SQLAlchemyConnectionField(EmployeeConn)
    # Disable sorting over this field
    all_departments = DepartmentConn(
        Department, sort=None, args={'name': graphene.Argument(graphene.String)})
{% endhighlight %}

However now when you reload GraphiQL, you'll see `name` show up as a parameter, and if you pass in "Engineering" you'll only get the engineering department and not the others. That's the simplest case, obviously. Once you define args and process then in the get_query method, you can pretty much construct any arbitrary filters you like. 

# Final notes

GraphQL via Graphene is self-documenting, and if you remember in your connections to add `description` arguments, and you add `doc=` args to your models, you'll get fully fleshed out, type-safe documentation about your GraphQL queries and node types. Don't underestimate the power of that.

Graphene changes fairly rapidly still. If you notice that there has been some drift on the validity of the code constructs in this post, definitely reach out to me and let me know. 

This should get you most of what you need to get to a production-ready GraphQL API via Graphene and Python. Obviously there are other frameworks out there than Flask, and in particular uses of GraphQL over RabbitMQ are interesting for microservice architectures. The same goes for Tornado and async GraphQL. If you try something new, feel free to send me a note. I'd love to see the other ideas people have on how to use Graphene. 