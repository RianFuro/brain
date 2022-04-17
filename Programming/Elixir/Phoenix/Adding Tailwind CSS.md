# Add package management to `assets`

To facilitate tailwind's css packaging we need a few tools, so we might as well use a package manager to handle dependencies. The `package.json` should at least look like this:
```json
{  
 "name": "home",  
 "scripts": { 
  "deploy": "NODE_ENV=production tailwindcss --postcss --minify -i css/app.css -o ../priv/static/assets/app.css"  
 },  
 "devDependencies": {  
 "autoprefixer": "^10.3.7",  
 "postcss": "^8.3.11",  
 "postcss-import": "^14.0.2",  
 "tailwindcss": "^2.2.17"  
 }  
}
```

To handle automatic package installation on setup we can adapt the respective alias in `mix.exs`:
```diff
 defp aliases do
   [
-    setup: ["deps.get", "ecto.setup"],
+    setup: ["deps.get", "ecto.setup", "cmd --cd assets npm install"],
```

# Configuration

Some necessary configuration to setup css compilation by tailwind:
- `postcss.config.js`:
  ```js
  module.exports = {  
   plugins: {  
   'postcss-import': {},  
   tailwindcss: {},  
   autoprefixer: {},  
   }  
  }
  ```
- `tailwind.config.js`:
  ```js
  module.exports = {  
    mode: 'jit',  
    purge: [  
      './js/**/*.js',  
      '../lib/*_web/**/*.*ex'  
    ],  
    theme: {  
    },  
    variants: {  
      extend: {},  
    },  
    plugins: [],  
  }
  ```

# Tailwind magic

To let tailwind take care of compilation, we need to hook it into phoenix's bootstrapping at various points.
- For automatic recompilation on changes, we need to add a watcher in `config/dev.exs`:
  ```diff
   watchers: [
     # Start the esbuild watcher by calling Esbuild.install_and_run(:default, args)
     esbuild: {Esbuild, :install_and_run, [:default, ~w(--sourcemap=inline --watch)]},
  +  npx: [
  +    "tailwindcss",
  +    "--input=css/app.css",
  +    "--output=../priv/static/assets/app.css",
  +    "--postcss",
  +    "--watch",
  +    cd: Path.expand("../assets", __DIR__)
  +  ]
   ]
  ```

- To prepare the css for production, we add to the `assets.deploy` alias in `mix.exs`:
  ```diff
  -"assets.deploy": ["esbuild default --minify", "phx.digest"]
  +"assets.deploy": ["cmd --cd assets npm run deploy", "esbuild default --minify", "phx.digest"]
  ```
  Note, that we just call the `deploy` script defined in the `package.json` file before.
  
Lastly, we can remove the css import in `app.js`. Since tailwind is now taking care of bundling, we don't need to manually import it to be taken up by the build pipeline.
```diff
-// We import the CSS which is extracted to its own file by esbuild.
-// Remove this line if you add a your own CSS build pipeline (e.g postcss).
-import "../css/app.css"
```

# Start using it
You can now start using tailwind like usual. At a minimum, add these to your `app.css`:
```css
@import "tailwindcss/base";  
@import "tailwindcss/components";  
@import "tailwindcss/utilities";
```