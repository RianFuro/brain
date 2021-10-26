Phoenix's template engine doesn't rely on explicitly extending from base layouts, so finding out the page is assembled is a bit of a wild goose chase. The defaults work as follows:

- `root.html.heex`:
  Phoenix provides a `Plug` to set an explicit root layout for your page in `Phoenix.Controller` called `put_root_layout`. By default, this is called in the browser pipeline in `router.ex`:
  ```elixir
  plug :put_root_layout, {HomeWeb.LayoutView, :root}
  ```
- `app.html.heex`:
  This is a tricky one. It's not actually referenced in the bootstrapped user code, but instead set as the layout when the Controller module is used; `phoenix/lib/phoenix/controller.ex`:
  ```elixir
  defmacro __using__(opts) do  
    quote bind_quoted: [opts: opts] do  
      import Phoenix.Controller  

      # TODO v2: No longer automatically import dependencies  
      import Plug.Conn  

      use Phoenix.Controller.Pipeline  

      if Keyword.get(opts, :put_default_views, true) do  
        plug :put_new_layout, {Phoenix.Controller.__layout__(__MODULE__, opts), :app}  
        plug :put_new_view, Phoenix.Controller.__view__(__MODULE__)  
      end  
    end
  end
  ```
  The default behavor can be disabled by setting `put_default_views` to `false` when using the module in the controller bootstrapping function in your main `ABC_web.ex#controller`:
  ```diff
   def controller do  
     quote do  
       use Phoenix.Controller, 
         namespace: HomeWeb,
  +      put_default_views: false
       import Plug.Conn  
       import HomeWeb.Gettext  
       alias HomeWeb.Router.Helpers, as: Routes  
     end  
   end
  ```
  However, this also disables the default view selection, so the `render` function won't have a view to use for rendering unless you specifically set one.
- `index.html.heex`:
  This leaves only the actual content, which is actually the easiest to trace, since it's explicitly mentioned in the controller itself.
  Phoenix does perform a bit of magic at this point though, which is documented very well [here](https://hexdocs.pm/phoenix/views.html#understanding-template-compilation).
  
# Overriding defaults

Defaults can easily be overridden at any time before the `render` function is called. There's a couple of functions on the controller module to help with that:
- `put_root_layout/2`
- `put_layout/2`
- `put_view/2`

Since all of these adhere to the `Plug`-spec, they can also be used with `plug` to set them at any point in the controller or pipeline.