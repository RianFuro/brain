Live Views are a nice way to seamlessly interact with a server from a website, while leaving UI updates to the server-side template engine.
Events are pushed to the server via socket connection and the server pushes the updated template back to the client.
To me, this makes it sound like _elm_, but with the mvu part on the server.

# Setup

The Live View files live inside a separate folder `live` inside the `abc_web` folder. Both the equivalent of the `View` module, as well as the template go there.
At a minimum, there must be a `LiveView` module whose name ends in `Live`:
```elixir
# live/my_thing_live.ex
defmodule HomeWeb.MyThingLive do  
  use Phoenix.LiveView
end
```

This module can either have a `render/2` function, or there must be a template in the same folder, with the same base name (`my_thing_live.html.heex` in our example).

The router then takes the definition not via a http-verb helper, but with the `live/2` function:
```elixir
scope "/", SomeWeb do  
 pipe_through :browser  
  
 get "/", PageController, :index  
 live "/thing", MyThingLive  
end
```

# Communication

There is a `socket` parameter passed through everything, which handles client connection and has a bag of parameters assigned as a "state".

To setup that state, you can define a `mount/3` function which takes the route's paramters, the session and the socket:
```elixir
def mount(_params, _session, socket) do  
  {:ok, socket}  
end
```

When the client sends an update, you can handle it by defining a `handle_event/3` function, which takes the event name, parameters and the socket. It's job is to handle the event, potentially updating the socket's assigns in the process:
```elixir
def handle_event(ev, data, socket) do
  {:noreply, assign(socket, newthing: data.someparameter)}
end
```

The website can send updates to the server via predefined bindings defined [here](https://hexdocs.pm/phoenix_live_view/Phoenix.LiveView.html#module-bindings)