---
title: "Using new @tailwindcss/jit in Ruby on Rails"
date: 2021-04-05 09:00:20 +0300
categories: rails tailwindcss
tags: rails tailwindcss
---

[Tailwindcss](https://tailwindcss.com/) is a utility-first CSS framework that gives you full controll of how you design and build your web application.

### :link: What is @tailwindcss/jit?

- @tailwindcss/jit is a new tool(or mode) of tailwindcss that compiles your CSS on demand via. Before, all CSS classes has to generated upfront which lead to large file size during development.

- Large CSS files leads to buggy chrome devtools and webpack-dev-server takes a long time

### :link: How to use it Rails?

> There is an excellent blogpost on [Evil Martins' site - Set up Tailwind CSS JIT in a Rails project to compile styles 20x faster](https://evilmartians.com/chronicles/set-up-tailwind-css-jit-in-a-rails-project-to-compile-styles-20x-faster) that addresses TailwindCSS JIT using `webpacker`, and uses *Tailwindcss 2.1*.



This involves running tailwindcss via node pipeline directly instead of webpacker



1. :point_right: Install @tailwindcss and @tailwindcss/jit as a devDependencies

   ```bash
   $ yarn install -D @tailwindcss @tailwindcss/jit \
   	postcss postcss-cli postcss-import postcss-nested \
   	autoprefixer 
   ```

   - `@tailwindcss` and `@tailwindcss/jit` are the main packages
   - `postcss` and `postcss-cli` are required since tailwindcss in a *postcss plugin*
   - `postcss-nested` allows using *sass-like nesting* in the CSS files and `post-import` is required if you need to *componentize CSS logic into classes*
   - `autoprefixer` adds *vendor prefixes* to generated css files

2. :point_right: Add *build* and *dev* scripts in `package.json` file

   ```json
   // package.json
   {
     // ..
     "script": {
      	"build": "TAILWIND_MODE=build postcss ./app/assets/stylesheets/tailwind.scss -o ./app/assets/stylesheets/tailwind-build.css --verbose",
       "dev": "TAILWIND_MODE=watch postcss ./app/assets/stylesheets/tailwind.scss -o ./app/assets/stylesheets/tailwind-build.css -w --verbose"
     }
     // ...
   }
   ```

3. :point_right: Create `tailwind.scss` in *app/assets/stylesheets/* directory, and add the following

```scss
// app/assets/stylesheets/tailwind.scss
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer components {
  // your custom css with @apply
}
```

4. :point_right: The output of *dev script* created `tailwind-build.css` in the directory. You need to register the output in *application.css* manifest file. Add `stub tailwind` and `require tailwind-build` as shown below

```css
/* app/assets/stylesheet/application (manifest file) */
/* comments... */

 *= require_tree .
 *= stub tailwind
 *= require tailwind-build
 *= require_self

```

5. :point_right: Next, in `tailwind.config.js`, (generated via `npx tailwindcss init`), you need to provide a list of *directories* and *files* to be purged. Tailwindcss watches those files and rebuild the css files

```js
// tailwind.config.js
module.exports = {
  purge: [
    "./app/views/**/*.erb",
    "./app/javascript/controllers/**/*.js",
    "./app/helpers/**/*.rb",
    "./app/components/**/*.rb",
    "./app/components/**/*.erb"
  ],
  darkMode: false, // or 'media' or 'class'
  theme: {
    extend: {},
  },
  variants: {
    extend: {},
  }
};

```

This code purges all files in *app/views*, *stimulus controller* folder, *helper* and *view_component'' components*

Add any *files* or *directory* that contains *tailwind-classes*

6. :point_right: Ignore the build `tailwindcss-build.css` files in your *.gitignore*

   ```
   # .gitignore
   
   …
   /node_modules
   npm-debug.log
   /app/assets/stylesheets/tailwind-build.css
   ```

   

### :link: Starting Development

To kick of, just run `yarn run dev` in one terminal and `bin/rails server` in another terminal 

:rocket: And there you go

However having to run :two: terminal is not ideal. You can use `foreman` to manage application processes

#### Using foreman during development

- :point_right: Install `foreman` gem in globally

```shell
$ gem install foreman
```

- :point_right: Create a `Procfile.dev` in the root directory. (Heroku uses `Procfile`), hence creating a dev one eradicates contradiction if we plan to deploy to heroku which we will do.

```ruby
# Procfile.dev
web: bin/rails server
yarn-dev: yarn run dev

```

- :point_right: Now starting the application is as simple as running *foreman* with a custom *procfile*. Foreman uses *port 5000* using `-p 3000` allows using [localhost:3000](localhost:3000) as standard rails.

```shell
$ foreman start -f Procfile.dev -p 3000
```

### :link: Deploying to Production

Technically, the `tailwindcss-build.css` file should be available or else *asset pipeline* will complain about missing file. We must compile it before starting the server in production environment.

To deploy to *Heroku*, (assuming you have everything setup for Heroku)

1. :point_right: Add *Nodejs Heroku builpack*

   ```shell
   $ heroku buildpacks:add --index 1 heroku/nodejs
   ```

   Adding `-index 1` will cause the *build* script to run (by default) which will create `tailwindcss-build.css` files require in *application* css manifest file.

   You could also add 

   ```shell
   $ heroku buildpacks:add --index 2 heroku/ruby
   ```

   but my deployment worked without it anyway!

2. :point_right: This assumes you have your `Procfile`

   ```ruby
   # Procfile
   release: bundle exec rails db:prepare
   web: bundle exec puma -C config/puma.rb
   #... other stuffs eg sidekiq
   ```

   

3. :point_right: Deploy your app (*assumes your branch is main instead of **master***)

```shell
$ git push heroku main
```

### :link: Caveats

- The downsize of *tailwind css jit* is that you cannot develop via devtools, meaning if a class say `mt-2` is not loaded you cant use it in devtools as it's not available.
- But trust me you can bear with me rather than dealing dealing with *chrome devtools* and *4GM* RAM laptop with *Intel Celeron* processor and having to start *bin/rails server*, *elasticsearch*, *redis-server*, *postgreslq* and *sidekiq*. My laptop once hanged and I had to force restart!

### :link: Resources

- [Just-In-Time: The Next Generation of Tailwind CSS](https://blog.tailwindcss.com/just-in-time-the-next-generation-of-tailwind-css) - Official tailwindcss blog

- [Tailwind CSS JIT + Rails without Webpacker](https://githubmemory.com/repo/domchristie/tailwindcss-jit-rails#tailwind-css-jit--rails-without-webpacker) - the first repo that inspired me to author this post
- [Set up Tailwind CSS JIT in a Rails project to compile styles 20x faster](https://evilmartians.com/chronicles/set-up-tailwind-css-jit-in-a-rails-project-to-compile-styles-20x-faster) - Usinf Tailwind CSS JIT with webpacker

