# GOV.UK Prototype Kit for developers

The GOV.UK Prototype Kit (“the Kit”) is a great starting point for building prototypes in an Alpha project. It’s a common language spoken by interaction designers, developers, and sometimes content designers and service designers.

For developers wishing to build some functionality that’s more than just static pages, it can be helpful to understand how the Kit is put together. Especially: what extension points does it explicitly provide, and what extension points naturally exist by virtue of the ecosystem that the Kit is built with? 

This is written with reference to commit `9e19f81` of the Kit.

TODO copy over other notes (e.g. written)

## How it’s built

It’s built using [Express](https://expressjs.com/), which is a web framework for Node.js. Express is an _unopinionated_, _lightweight_ web framework - all it really does it provide tools for URL routing, view rendering, and middleware. All other decisions are up to the developers. So it’s interesting to understand what decisions have been made for the Kit.

### Directory structure

-   `docs` documentation for the kit
    -   `docs/views/templates` — copy from here to make new views in `app/views`
-   `app/config.js`
-   the `data` variable in views lets you get persisted data — the kit persists data with each request. (How?) Handles nested fields with e.g. `name="claimant[over-18]"` (How?)
    -   `app/data/session-data-defaults.js`
    -   outside of views it’s accessed as `req.session.data`, uses Express Session
    -   prefix input’s name with underscore and it won’t be stored (TODO how)
-   `/public` is auto-generated; put assets into `/app/assets`. Add to `app/assets/application.[css|js]`. Would be good to unpick what about those mean that GOV.UK stuff ends up being around. Linking to e.g. images, use href of `/public/images/user.png`. TODO what’s all the stuff that comes in `public` when you download the Kit? Oh, maybe there isn’t - it’s in gitignore after all.
-   `layout.html` would be what you update to e.g. change the header. It imports lots of stuff from the GOV.UK Frontend. `extends "govuk/template.njk"`. How does the template data get into it?
-   use routes to do branching — where? there’s an `app/routes.js` although I wonder how the default handling is done…
-   why does `render` not need `html` extension—Express does this?
-   the `serviceName` variable
-   their Notify tutorial tells us a couple of things—environment variables go in `.env`, and they seem to dump general JS in `routes.js`
-   it has a `lib` directory

### How some of the functionality is built

e.g.

- being able to do key paths in POST data (I’m guessing it comes from the [`body-parser`’s `extended` attribute](https://github.com/expressjs/body-parser#extended), which uses the `qs` library, which defines the data structure rules
- being able to inject extra locals into the views at a router level, using `res.locals` — that’s just [part of Express](http://expressjs.com/en/api.html#res.locals)

#### Auto-saving session, and `data` in the views

`server.js` `use`s `utils.autoStoreData`, and calls `utils.addCheckedFunction(nunjucksAppEnv)`

`autoStoreData` is a middleware that takes everything from `req.body` and `req.query` and stores it in `req.session.data`, and then it populates `res.locals.data`. (and incorporates `sessionDataDefaults`)

TODO `storeData` - has a bit of special handling of checkboxes. It’s here that session keys starting with `_` are ignored.

In `server.js` we configure Nunjucks with a config object, which we pass to `nunjucks.configure(appViews, nunjucksConfig)`. For some reason this includes an `express: app` object. This returns a `nunjucksAppEnv`, which we pass to `utils.addNunjucksFilters()`

TODO why `app.set('view engine', 'html')`?

TODO what’s `appViews` - seems to come from the extensions system

- `addCheckedFunction` calls `addGlobal` on the Nunjucks config. That looks for `this.ctx.data` (so I guess `ctx` provides the view’s locals?). It has (TODO) some logic about handling key paths and arrays and whatnot

- addNunjucksFilters - calls `env.addFilter`, again on the Nunjucks config. Does this for `lib/core_filters.js` and `app/filters.js`. The only one it comes with is `log`, which inserts a `<script>` tag to log something to the console

#### How do HTML pages end up getting served without an explicit route?

In `server.js` - has a regex `app.get`, with middleware `utils.matchRoutes`. This function tries to render a template with that name; retrying a couple of times if error has “template not found”, or trying to find the `index` page for the directory. (Doesn’t perform any filesystem access itself; leaves that to Express)

`server.js` also has a middleware which strips `.html` or `.htm` from the request path and redirects (not sure of use case)

There are some middlewares at the bottom of `server.js` (after all of the `matchRoutes`,  and `routes.js` stuff) to handle errors and 404

#### What exists for the documentation site?

TODO but I don’t care that much at the moment

### What the default `dependencies` do

TODO split into structural and utilities

- `acorn` :: this is a “peer dependency” (TODO what does that mean) of some other things, and if you don’t install it you get some warning on `npm install`. There are a few other such warnings though, anyway.

- `basic-auth` :: used by `lib/middleware/authentication/authentication.js` . Parses auth headers from a request
- `basic-auth-connect` :: I think not used any more - it was a middleware but considered deprecated. TODO PR to remove

- `body-parser` :: used in `server.js` - a middleware which we configure to parse JSON and URL encoded bodies

- `browser-sync` :: potentially used in `server.listen` call in `listen-on-port.js`. I think given a list of files to watch (`public` and `app/views`). Can be run as a server or as a proxy - we run it as a proxy. Used for syncing browsers or live reloading

- `client-sessions` (and `express-session`) :: used in `server.js` - a middleware which we use to store the session in cookie? Can be configured on /off; if off then instead uses `express-session`, not sure of difference but latter described as "in memory"

- `cookie-parser` :: used in `server.js`, a middleware which I think populates `req.cookies`. Then, `handleCookies` in `lib/utils.js` uses this property for the cookie banner

- `cross-spawn` :: used in `runGulp()` in `start.js` to run gulp

- `del` :: used in `gulp/clean.js` to somehow interact with gulp to delete the `public` directory

- `dotenv` :: used in `server.js` to load `.env` environment variables; use case is described in `docs/documentation/using-notify.md`

- `express` :: used in `server.js` to create the app, documentation app, etc. The only place that `express` is directly required.

- `express-writer` :: Not clear what used for; accidentally introduced, I think unused? Seems to be something that lets you auto-generate files by hitting routes. TODO remove

- `fancy-log` :: used in `gulp/nodemon.js` to write a single coloured log message

- `govuk-frontend` :: TODO
  - https://frontend.design-system.service.gov.uk/importing-css-assets-and-javascript/#import-css-assets-and-javascript
  - https://frontend.design-system.service.gov.uk/sass-api-reference/#sass-api-reference
  - `application.scss` has a couple of variables: `$govuk-global-styles: true` and `$govuk-assets-path` (?)
  - then `@import "lib/extensions/extensions";`, not really sure what that’s doing
  - `server.js` has some stuff for hosting `govuk-frontend` - perhaps superseded by the extensions stuff though
  - the `app/views/includes/head.html` loops through extensions and includes their stylesheets
  - but there's also a `<link href="/public/stylesheets/application.css">`
  - And ditto for `scripts.html` and `application.js`… hmm
  - I noticed that `scripts.html` also includes jquery, not sure why
  - CSS: I think that it is handled through the extensions system 
  - and then a separate question is how all this gets served up
  - `layout.html` has a bunch of Nunjucks imports

- `govuk-elements-sass`, `govuk_frontend_toolkit`, `govuk_template_jinja` - these are all deprecated, used to support the old GOV.UK design system. Seems to be mainly integrated via `gulp/sass.js` and `server.js` (if the `govuk-frontend` integration isn’t illuminating, take a look at these ones)

- `gulp` (and `gulp-nodemon`, `gulp-sass`, `gulp-sourcemaps`) :: Needs a deeper investigation. But configured by `gulp/*.js` files, and a `gulpfile.js`. And then `start.js` runs it. I think it’s a key part to understand

- `inquirer` :: For command-line user interfaces. Used by `findAvailablePort` in `lib/utils.js` for prompts and `lib/usage_data.js` for permission to track data

- `keypather` :: it's a library that allows you to access objects by key paths e.g. `get(obj, 'foo["bar"].qux')`. Used in `lib/utils.js` `exports.addCheckedFunction` to allow expressivity in the `checked()` view function

- `marked` :: Markdown parsing library, used by `matchMdRoutes` in `lib/utils.js`, which is used by the documentation app in `server.js`

- `notifications-node-client` :: this is the GOV.UK Notify client; explains how to use it in `docs/documentation/using-notify.md`. Not used anywhere by default

- `nunjucks` :: template engine. Configured in `server.js`

- `portscanner` :: the aforementioned `findAvailablePort` calls `findAPortNotInUse` on this

- `require-dir` :: “helper to `require()` directories” - used in `gulpfile.js` to recursively require `./gulp`

- `sync-request` :: used by getLatestRequest in `lib/utils.js` to fetch URL of latest release of the Kit. Library to let you make synchronous web requests. Surprised not part of Node (and surprised synchronous is so important to us) TODO we should remove this

- `universal-analytics` :: used in `lib/usage_data.js` to send - presumably OS / Kit / Node version data - to Google Analytics

- `uuid` :: used to generate a random UUID in `lib/usage_data.js`

TODO why gulp, cross-spawn not in dev dependencies?

TODO remove the Greenkeeper config in `package.json` - it's a bit like dependabot but been replaced by Snyk anyway

### Is there magic that happens just by virtue of a package being installed, or is everything explicit?

### How is the server started?

- in production: the `Procfile` does `node ./node_modules/gulp/bin/gulp generate-assets && node listen-on-port.js` 
- in dev: `node start.js`

`start.js` ultimately runs `gulp` (directly from `./node_modules/.bin`). This calls `listen-on-port.js`, using nodemon in `gulp/nodemon.js`. So gulp is like a long-running process?

(`nodemon` is like `node` but it restarts when your code changes)

And I guess that `gulp generate-assets` is generating some of the stuff that long-running `gulp` would generate otherwise.

So `listen-on-port.js` is, either way, the entry point to our service.

That gets the `server` (an Express app) from `server.js`, and then finds an available port and listens (possibly with `browserSync`, as mentioned elsewhere)

### How is the kit configured?

So some stuff is configured via `process.env`, and some stuff via `app/config.js`. It says that “prototype configuration” can be overridden using environment variables”.

Interesting thing I learned from config: sessions are stored in-memory by default, which is why they are lost after a restart. We can change it to a cookie store (with a max 4KB cookie limit to note)

### What kind of testing exists?

TODO Because there certainly are tests, e.g. `lib/middleware/authentication/authentication.test.js`

### What the default `devDependencies` do

- `glob`, `node-sass`, `supertest` :: used in `__tests__/spec/sanity-checks.js` 
- `jest` :: testing framework
- `standard` :: JS linter
- `supertest` :: a framework for testing HTTP servers - pass it an `http.Server`, or a function that takes a request and you can make calls and assertions about the response. Again, used in `sanity-checks.js`

## How the frontend code is built

TODO seems to be something to do with Gulp - a _task runner_ (I guess like Rake)

I think that the `gulpfile.js` defines the top-level _tasks_, which are composites of tasks defined in `gulp/*.js`.

Gulp has a small API - https://gulpjs.com/docs/en/api/concepts, with only a few concepts

The `generate-assets` task is probably our most important one. TODO how does it work?

The `config` referred to in the Gulp JS is `gulp/config.json`. This contains various paths e.g. `public: public/`, `nodeModules`, …

## How the frontend code is served

TODO

`server.js`:

```javascript
// Serve govuk-frontend in from node_modules (so not to break pre-extenstions prototype kits)
app.use('/node_modules/govuk-frontend', express.static(path.join(__dirname, '/node_modules/govuk-frontend')))
```

## How to add a front-end dependency

TODO understand the above two first

`app.locals.asset_path = '/public/'` in `server.js`

## How to build complicated formatting logic

a.k.a. “why don’t arrow functions work in Nunjucks expressions?”

## “Real code” functionality that might also be useful for prototypes

- automatic formatting

- The Kit is set up to use Standard for formatting JavaScript - run it with `npm lint`. See documentation in `docs/linting.md`.

## At what point might the Kit functionality not be sufficient?

## TODO

- how can we get text editors to recognize the `.html` files as being Nunjucks, and format appropriately?. I don’t yet understand how the template resolution works. Is `app.set('view engine', 'html')` actually doing anything (answer: yes; without it we get `No default engine was specified and no extension was provided.`)? And if not, then is it _Nunjucks_ that looks for `.html`?
- how to version prototypes after each sprint? 
- https://vickyteinaki.com/blog/more-efficient-prototyping-with-the-gov-uk-prototype-kit-step-by-step/#7-set-defaults-asnbspneeded
