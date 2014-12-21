##Controller Guide

Phoenix controllers act as intermediary modules. Their functions - called actions - are invoked from the router in response to HTTP requests. The actions, in turn, gather all the necessary data and perform all the necessary steps before - in a typical case - invoking the view layer to render a template.

A newly generated Phoenix app will have a single controller, the `PageController`, which looks like this.

```elixir
defmodule HelloPhoenix.PageController do
  use Phoenix.Controller

  plug :action

  def index(conn, _params) do
    render conn, "index.html"
  end
end
```
The first line below the module definition invokes the `__using__/1` macro of the `Phoenix.Controller` module, which imports some useful modules.

The second line `plug :action` has been recently added with the 0.5.0 release of Phoenix. `plug/1` is a macro defined in the `Phoenix.Controller.Pipeline` module. Its purpose is to insert a new piece of middleware into the stack of middlewares which will be executed during a request cycle. Here, it is inserting `:action` which handles dispatching to the correct controller module and action according to the routes defined in the router.

In addition, the `PageController` gives us the index action to display the Phoenix welcome page associated with the default route Phoenix defines in the router.

###Actions
Controller actions are just functions. We can name them anything we like as long as they follow Elixir's naming rules. The only requirement we must fulfill is that the action name matches a route defined in the router.

For example, we could change the action name in the default route that Phoenix gives us in a new app from index:

```elixir
get "/", HelloPhoenix.PageController, :index
```

To test:

```elixir
get "/", HelloPhoenix.PageController, :test
```

As long as we change the action name in the `PageController` to `test` as well, the welcome page will load as before.

```elixir
def test(conn, _params) do
 render conn, "index.html"
end
```

While we can name our actions whatever we like, there are conventions for action names which we should follow whenever possible. We went over these in the [Routing Guide](http://www.phoenixframework.org/docs/routing), but we'll take another quick look here.

- index   - renders a list of all items of the given resource type
- show    - renders an individual item by id
- new     - renders a form for creating a new item
- create  - receives params for one new item and saves it in a datastore
- edit    - retrieves and individual item by id and displays it in a form for editing
- update  - receives params for one edited item and saves it to a datastore
- destroy - receives an id for an item to be deleted and deletes it from a datastore

Each of these actions takes two parameters, which will be provided by Phoenix behind the scenes.

The first parameter is always `conn`, a struct which holds information about the request such as the host, path elements, port, query string, and much more. `conn`, comes to Phoenix via Elixir's plug middleware framework. More detailed info about `conn` can be found in [plug's documentation](http://hexdocs.pm/plug/Plug.Conn.html).

The second parameter is `params`. Not surprisingly, this is a map which holds any parameters passed along in the HTTP request. It is a good practice to pattern match against params in the function signature to provide data in a simple package we can pass on to rendering. We saw this in the [Adding Pages guide](http://www.phoenixframework.org/docs/adding-pages) when we added a messenger parameter to our `show` route.

```elixir
def show(conn, %{"messenger" => messenger}) do
  render conn, "show.html", messenger: messenger
end
```
In some cases - often in `index` actions, for instance - we don't care about parameters because our behavior doesn't depend on them. In those cases, we don't use the incoming params, and simply prepend the variable name with an underscore, `_params`. This will keep the compiler from complaining about the unused variable while still keeping the correct arity.

###Gathering Data
While Phoenix does not ship with its own data access layer, the Elixir project [Ecto](http://hexdocs.pm/ecto) provides a very nice solution for those using the [Postgres](http://www.postgresql.org/) relational database. (There are plans to offer other adapters for Ecto in the future.) We cover how to use Ecto in a Phoenix project in the [Ecto Models Guide](http://www.phoenixframework.org/docs/ecto-models).

Of course, there are many other data access options. [Ets](http://www.erlang.org/doc/man/ets.html) and [Dets](http://www.erlang.org/doc/man/ets.html) are key value data stores built into [OTP](http://www.erlang.org/doc/). OTP also provides a relational database called [mnesia](http://www.erlang.org/doc/man/mnesia.html) with its own query language called QLC. Both Elixir and Erlang also have a number of libraries for working with a wide range of popular data stores.

The data world is your oyster, but we won't be covering these options in these guides.

###Flash Messages

There are times when we need to communicate with users during the course of an action. Maybe there was an error updating a model. Maybe we just want to welcome them back to the application. For this, we have flash messages.

In order to use flash messages, we first alias the `Phoenix.Controller.Flash` module in the controller we want to set messages in.

```elixir
defmodule HelloPhoenix.PageController do
  use Phoenix.Controller
  alias Phoenix.Controller.Flash
. . .
```
Now we can use `Flash.put/3` to set flash messages on `conn` for this request cycle. We could change the `PageController` `index` action to set a notice and an error.

```elixir
def index(conn, _params) do
  conn
  |> Flash.put(:notice, "Welcome to Phoenix, from a flash notice!")
  |> Flash.put(:error, "Let's pretend we have an error.")
  |> render "index.html"
end
```

The `Phoenix.Controller.Flash` module is not particular about the keys we use. As long as we are internally consistent, all will be well. `notice`, `error`, and `alert`, however, are common.

In order to see our flash messages, we need to be able to pull them off of the `conn` and display them all in a template/layout. One way to do the first part is with `get_all/2` which takes `conn` and the key we care about and returns a list of values for that key.

Let's put these blocks in our application layout. They are designed to work if we have one or many messages set on each key.

```html
<%= for error <- Flash.get_all(@conn, :error) do %>
      <div class="flash_error">
        <div class="row">
          <p><%= error %></p>
        </div>
      </div>
 <% end %>

 <%= for notice <- Flash.get_all(@conn, :notice) do %>
       <div class="flash_notice">
         <div class="row">
           <p><%= notice %></p>
         </div>
       </div>
  <% end %>
```

When we reload the page, our messages should appear.

Besides `put/3` and `get_all/2`, the `Phoenix.Controller.Flash` module has some other useful functions worth looking into. The doc strings embedded in the source code have examples.

`persist/2` takes `conn` and a key, and allows us to save flash messages for that key in the session so that they can persist beyond the current request cycle.

`get/2` also takes `conn` and a key, but only returns a single value.

`clear/1` takes only `conn` and removes any flash messages in the session.

`pop_all/2` also takes `conn` and a key, and returns a tuple containing a list of values and `conn`.

### Rendering
Controllers have several ways of rendering content. The simplest is to render some plain text using the `text/2` function.

Let's say we have a show action which receives an id from the params map, and all we want to do is return some text with the id. For that, we could do the following.

```elixir
def show(conn, %{"id" => id}) do
  text conn, "Showing id #{id}"
end
```
Assuming we had a route for `get "/our_path/:id"` mapped to this show action, going to `/our_path/15` in your browser should display `Showing id 15` as plain text without any HTML.

A step beyond this is rendering pure json. Phoenix provides the `json/2` function for this. We must pass in something that the Poison library can parse into JSON such as a map.

```elixir
def show(conn, %{"id" => id}) do
  json conn, %{id: id}
end
```
If we again visit `our_path/15` in the browser, we should see a block of JSON with the key `id` mapped to the number `15`.

```elixir
{
  id: 15
}
```
Phoenix controllers can also render HTML without a template. As you may have already guessed, the `html/2` function does just that. This time, we implement the show action like this.

```elixir
def show(conn, %{"id" => id}) do
  html conn, """
     <html>
       <head>
          <title>Passing an Id</title>
       </head>
       <body>
         <p>You sent in id #{id}</p>
       </body>
     </html>
    """
end
```

Hitting `/our_path/15` this time generates the HTML document as we created the string for in the action, except that the value `15` will be interpolated into the page. Note that what we wrote in the action is not an eex document. It's a multi-line string, so we interpolate the `id` variable like this `#{id}` instead of this `<%= id %>`.

It is worth noting that the `text/2`, `json/2`, and `html/2` functions require neither a Phoenix view, nor a template to render.

The `json/2` function is obviously useful for writing APIs, and the other two may come in handy, but rendering a template into a layout with values we pass in is a very common case.

For this, Phoenix provides the `render/3` function.

Interestingly, `render/3` is defined in the `Phoenix.View` module instead of `Phoenix.Controller`, but it is aliased in `Phoenix.Controller` for convenience.

We have already seen the render function in the [Adding Pages Guide](http://www.phoenixframework.org/docs/adding-pages). Our show action there looked like this.

```elixir
defmodule HelloPhoenix.HelloController do
  use Phoenix.Controller

  plug :action

  def show(conn, %{"messenger" => messenger}) do
    render conn, "show.html", messenger: messenger
  end
end
```

The controller must have the same root name as the individual view as the directory name where `show.html.eex` lives in order for this `render/3` call to work properly. `render/3` will also pass the value which the show action received for messenger from the params hash into the template for interpolation.

There is also a shortcut for rendering which uses a plug for rendering. Here's how.

The first thing we need to do is to add a `plug :render` line to our controller.

```elixir
defmodule HelloPhoenix.PageController do
  use Phoenix.Controller

  plug :action
  plug :render

. . .
```

After that we can omit the render call in our actions as long as we return `conn` either directly or as the return value of a function.

In a new Phoenix app, try plugging render and then replacing the index action with this.

```elixir
def index(conn, _params) do
  conn
end
```
After restarting our local server and viewing the root route in our browser, we should see the "Welcome to Phoenix!" page exactly as we did when we explicitly rendered the index template.

Just as `plug :action` inserted a plug for dispatching to the correct module/function in a controller's plug stack, `plug :render` inserts a rendering plug in the stack for us. In essence, we've just moved rendering to a different layer.

If we need to pass values into the template when using `plug :render`, that's easy. We can use `Plug.Conn.assign/3`, which conveniently returns `conn`.

```elixir
def index(conn, _params) do
  assign(conn, :message, "Welcome Back!")
end
```
We can access this message in our `index.html.eex` template like this `<%= @message %>`, anywhere we like.

The `Phoenix.Controller` module imports `Plug.Conn`, so shortening the call to `assign/3` works just fine.

Passing more than one value in to our template is as simple as connecting `assign/3` functions together in a pipeline.

```elixir
def index(conn, _params) do
  conn
  |> assign(:message, "Welcome Back!")
  |> assign(:name, "Dweezil")
end
```
With this, both `@message` and `@name` will be available in the `index.html.eex` template.

What if we want to plug render, but only some of our actions should actually call `render/2`? For example, some actions might need to do a redirect, or return JSON.

By default, the render plug takes over all rendering for all actions in a controller. Once we plug render, if we try to use any rendering-style function in any action - `text/2`, `json/2`, `html/2`, `render/2` or even `redirect/2`- we will get errors if that action is dispatched to. Even if the application appears to do the right thing, the server will be throwing errors.

Phoenix offers a solution to this by letting us specify which actions `plug :render` should be applied to. If we only wanted `plug :render` to work on the `index` and `show` actions, we could do this.

```elixir
defmodule HelloPhoenix.PageController do
  use Phoenix.Controller

  plug :action
  plug :render  when action in [:index, :show]

. . .
```
Then we are free to call any of the rendering-style functions in any other actions besides `index` and `show` in `HelloPhoenix.PageController` without generating errors.

Rendering does not end with the template, though. By default, the results of the template render will be inserted into a layout, which will also be rendered.

[Templates and layouts](http://www.phoenixframework.org/docs/templates) have their own guide, so we won't spend much time on them here. What we will look at is how to assign a different layout, or none at all, inside a controller action.

### Assigning Layouts

Layouts are just a special subset of templates. They live in `/web/templates/layout`. Phoenix created one for us when we generated our app. It's called `application.html.eex`, and it is the layout into which all templates will be rendered by default.

Since layouts are really just templates, they need a view to render them. This is the `LayoutView` module defined in `/web/views/layout_view.ex`. Since Phoenix generated this view for us, we won't have to create a new one as long as we put the layouts we want to render inside the `/web/templates/layout` directory.

Before we create a new layout, though, let's do the simplest possible thing and render a template with no layout at all.

The `Phoenix.Controller.Connection` module provides the `put_layout/2` function for us to switch layouts with. (Note: in release 0.4.1 and earlier, this was `assign_layout/2`.) This takes `conn` as its first argument and a string for the basename of the layout we want to render as the second. Another clause of the fuction will match on the atom `:none` for the second argument, and that's how we will render the Phoenix welcome page with no layout at all.

In a freshly generated Phoenix app, edit the index action of the `PageController` module to look like this.

```elixir
def index(conn, params) do
  conn
  |> put_layout(false)
  |> render "index.html"
end
```
When you start the application and view [http://localhost:4000/](http://localhost:4000/), you should see a very different page, one with no title, logo image, or css styling at all.

Very Important! For function calls in the middle of a pipeline, like `put_layout/2` here, it is critical to use parenthesis around the arguments because the pipe operator binds very tightly. This leads to parsing problems and very strange results.

If you ever get a stack trace that looks like this,

```text
**(FunctionClauseError) no function clause matching in Plug.Conn.get_resp_header/2

Stacktrace

    (plug) lib/plug/conn.ex:353: Plug.Conn.get_resp_header(false, "content-type")
```

where your argument replaces `conn` as the first argument, one of the first things to check is whether there are parens in the right places.

This is fine.

```elixir
def index(conn, params) do
  conn
  |> put_layout(false)
  |> render "index.html"
end
```

This won't work.

```elixir
def index(conn, params) do
  conn
  |> put_layout false
  |> render "index.html"
end
```

Now let's actually create another layout and render the index template into it. As an example, let's say we had a different layout for the admin section of our application which didn't have the logo image. To do this, let's copy the existing `application.html.eex` to a new file `admin.html.eex`. Then remove the line in it that displays the logo.

```html
<span class="logo"></span> <!-- remove this line -->
```

Then, pass the basename of the new layout into `put_layout/2` in our index action.

```elixir
def index(conn, params) do
  conn
  |> put_layout("admin.html")
  |> render "index.html"
end
```
Reload the page, and we should be rendering the admin layout with no logo.

### Overriding Rendering Formats

Rendering HTML through a template is fine, but what if we need to change the rendering format on the fly? Let's say that sometimes we need HTML, sometimes we need plain text, and sometimes we need JSON. Then what?

Phoenix allows us to change formats on the fly with the `format` query string parameter. To make this  happen, Phoenix requires an appropriately named view and an appropriately named template in the correct directory.

Let's take the `PageController` index action from a newly generated app as an example. Out of the box, this has the right view, `PageView`, the right templates directory, `/web/templates/page`, and the right template for rendering HTML, `index.html.eex`.

```elixir
def index(conn, _params) do
  render conn, "index.html"
end
```
What it doesn't have is a new template for rendering text. Let's add one at `/web/templates/page/index.text.eex`.

Here is our example `index.text.eex` template.

```elixir
"OMG, this is actually some text."
```
There are just few more thing we need to do to make this work. We need to tell our router that it should accept the `text` format. We do that by adding to the list of accepted formats in the `:before` pipeline. Let's open up `web/router.ex` and change the `plug :accepts` line like this.

```elixir
defmodule HelloPhoenix.Router do
  use Phoenix.Router

  pipeline :browser do
    plug :accepts, ~w(html, text)
    plug :fetch_session
  end
. . .
```
And also we need to tell controller to render a template with the same format as the one found in conn.params["format"].

```elixir
def index(conn, _params) do
  render conn, :index
end
```
If we go to [http://localhost:4000/?format=text](http://localhost:4000/?format=text), we will see `OMG, this is actually some text.`

Of course, we can pass data into our template as well. Let's change our action to take in a message parameter.

```elixir
def index(conn, params) do
  render conn, "index.text", message: params["message"]
end
```

And let's add a bit to out text template.

```elixir
"OMG, this is actually some text." <%= @message %>
```
Now if we go to `http://localhost:4000/?format=text&message=CrazyTown`, we will see "OMG, this is actually some text. CrazyTown"

### Setting the Content Type

Analogous to the `format` query string param, we can render any sort of format we want by modifying the HTTP Accepts Header and providing the appropriate template. If we wanted to render an xml version of our index action, we might implement the action like this.

```elixir
def index(conn, _params) do
  conn
  |> put_resp_content_type("text/xml")
  |> render "index.xml", content: some_xml_content
end
```
We would then need to provide an `index.xml.eex` template which created valid xml, and we would be done.

For a list of valid content mime-types, please see the documentation from the plug middleware framework: [https://github.com/elixir-lang/plug/blob/master/lib/plug/mime.types](https://github.com/elixir-lang/plug/blob/master/lib/plug/mime.types)

### Setting the HTTP Status

We can also set the HTTP status code of a response similarly to the way we set the content type. The `Plug.Conn` module, imported into all controllers, has a `put_status/2` function to do this.

`put_status/2` takes `conn` and either an integer or a "friendly name" used as an atom for the status code we want to set. Here is the list of supported [https://github.com/elixir-lang/plug/blob/master/lib/plug/conn/status.ex#L7-L63](friendly names).

Let's change the status in our `PageController` `index` action.

```elixir
def index(conn, _params) do
  conn
  |> put_status(202)
  |> render "index.html"
end
```
The status code we provide must be valid - Cowboy, the web server Phoenix runs on, will throw an error on invalid codes. If we look at our development logs (which is to say, the iex session), or use our browser's web inspection network tool, we will see the status code being set as we reload the page.

If the action sends a response - either renders or redirects - changing the code will not change the behavior of the response. If, for example, we set the status to 404 or 500 and then `render "index.html"`, we do not get an error page. Similarly, no 300 level code will actually redirect. (It wouldn't know where to redirect to, even if the code did affect behavior.)

This implementation of the `HelloPhoenix.PageController` `index` action, for example, will _not_ render the default `not_found` behavior.

```elixir
def index(conn, _params) do
  conn
  |> put_status(:not_found)
  |> render "index.html"
end
```

### Redirection

Often, we need to redirect to a new url in the middle of a request. A successful create action, for instance, will usually redirect to the show action for the model we just created. Alternately, it could redirect to the index action to show all the things of that same type. There are plenty of other cases where redirection is useful as well.

Whatever the circumstance, Phoenix controllers provide the handy `redirect/2` function to make redirection easy. Phoenix differentiates between redirecting to a path within the application and redirecting to a url external to our application.

In order to try out `redirect/2`, let's create a new route.

```elixir
get "/redirect_test", HelloPhoenix.PageController, :redirect_test, as: :redirect_test
```

Then we'll change the `index` action to do nothing but redirect to our new route.

```elixir
def index(conn, _params) do
  redirect conn, to: "/redirect_test"
end
```

Finally, let's define the action we redirect to, which simply renders the text `Redirect!`.

```elixir
def redirect_test(conn, _params) do
  text conn, "Redirect!"
end
```
When we reload our Welcome page at the root route, we see that we've been redirected to `/redirect_test` which has rendered the text `Redirect!`. It works!

If we care to, we can open up our developer tools, click on the network tab, and visit our root route again. We see two main requests for this page - a get to `/` with a status of `302`, and a get to `/redirect_test` with a status of `200`.

Notice that the redirect function takes `conn` as well as a string representing a relative path within our application. It can also take `conn` and a string representing a fully-qualified url.

```elixir
def index(conn, _params) do
  redirect conn, external: "http://elixir-lang.org/"
end
```
We can also make use of the path helpers we learned about in the Routing Guide. It's useful to alias the helpers in order to shorten the expression.

```elixir
defmodule HelloPhoenix.PageController do
  use Phoenix.Controller
  alias HelloPhoenix.Router.Helpers

  def index(conn, _params) do
    redirect conn, to: Helpers.redirect_test_path(:redirect_test)
  end
end
```
Note that we can't use the url helper here because `redirect/2` using the atom `:to`, expects a path. For example, the following will fail.

```elixir
def index(conn, _params) do
  redirect conn, to: Helpers.redirect_test_path(:redirect_test) |> Helpers.url
end
```
If we want to use the url helper to pass a full url to `redirect/2`, we must use the atom `:external`. Note that the url does not have to be truly external to our application to use `:exernal`, as we see in this example.

```elixir
def index(conn, _params) do
  redirect conn, external: Helpers.redirect_test_path(:redirect_test) |> Helpers.url
end
```
