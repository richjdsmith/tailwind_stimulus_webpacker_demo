# README
This should be a good place for you to get an idea on how the heck you integrate Tailwind StimulusJS and Webpacker together on a new Rails 5.2 app. It also gives direction on removing the Asset Pipeline from it.

You should be able to run it by simple cloning it to your computer, running  `yarn` then `rails db:create` locally. 

I have pasted the follow-along guide below. 
# Ruby on Rails 5.2 with Webpacker, Tailwind, StimulusJS and removing the Asset Pipeline

Alright, because there is nothing online that I could find showing people how to do this, I decided to record it myself so I could get back to this later. 

## 1. New Rails App

Generate the new rails app
`$rails new <app_name> --webpack=stimulus --skip-sprockets -T --database=postgresql`


## 2. Clean out Asset Pipeline - Part 1, Replace Javascript with Webpacker Managed Javascript

This part is a pain and was a challenge to sort out. Lets see how this pays off. First, we're going to install the required Node Modules to replace the Rails ones and set up webpacker.

`yarn add rails-ujs activestorage turbolinks`

Then in **./app/javascript/packs/application.js**, replace what's in there with the following:
```
console.log('Great news! Webpacker is working up to this point!')
// app/javascript/initializers/rails_ujs.js
import Rails from 'rails-ujs'
Rails.start()

// app/javascript/initializers/activestorage.js
import * as ActiveStorage from 'activestorage'
ActiveStorage.start()

// app/javascript/initializers/turbolinks.js
import Turbolinks from 'turbolinks'
Turbolinks.start()
Turbolinks.setProgressBarDelay(50)

// StimulusJS
import {
  Application
} from "stimulus"
import {
  definitionsFromContext
} from "stimulus/webpack-helpers"
const application = Application.start()
const context = require.context("controllers", true, /.js$/)
application.load(definitionsFromContext(context))

```

Because I like to use the *webpacker-dev-server*, now's a good time to go update the Content Security Policy found in **./config/initializers/content_security_policy.rb** with:

```
...
Rails.application.config.content_security_policy do |policy|
#   policy.default_src :self, :https
#   policy.font_src    :self, :https, :data
#   policy.img_src     :self, :https, :data
#   policy.object_src  :none
#   policy.script_src  :self, :https
#   policy.style_src   :self, :https
  policy.connect_src :self, :https, "http://localhost:3035", "ws://localhost:3035" if Rails.env.development?
# Specify URI for violation reports
#   policy.report_uri "/csp-violation-report-endpoint"
 end

# If you are using UJS then enable automatic nonce generation
 Rails.application.config.content_security_policy_nonce_generator = -> request { SecureRandom.base64(16) }

...
```


Last step to replacing our Asset Pipeline with Webpack is removing the `javascript_include_tag` in **./app/views/layouts/application.html.erb** with:
```
-- <%= javascript_pack_tag 'application', 'data-turbolinks-track': 'reload' %>
++ <%= javascript_include_tag 'application', 'data-turbolinks-track': 'reload' %>
```


At this point, your Gemfile should be pretty sparse. This is what mine looks like (notice how there's no Sass!):

```
source 'https://rubygems.org'
git_source(:github) { |repo| "https://github.com/#{repo}.git" }

ruby '2.5.1'

# Bundle edge Rails instead: gem 'rails', github: 'rails/rails'
gem 'rails', '~> 5.2.1'
# Use postgresql as the database for Active Record
gem 'pg', '>= 0.18', '< 2.0'
# Use Puma as the app server
gem 'puma', '~> 3.11'
# Transpile app-like JavaScript. Read more: https://github.com/rails/webpacker
gem 'webpacker'
# Build JSON APIs with ease. Read more: https://github.com/rails/jbuilder
gem 'jbuilder', '~> 2.5'
# Use Redis adapter to run Action Cable in production
# gem 'redis', '~> 4.0'
# Use ActiveModel has_secure_password
# gem 'bcrypt', '~> 3.1.7'

# Use ActiveStorage variant
# gem 'mini_magick', '~> 4.8'

# Use Capistrano for deployment
# gem 'capistrano-rails', group: :development

# Reduces boot times through caching; required in config/boot.rb
gem 'bootsnap', '>= 1.1.0', require: false

group :development, :test do
  # Call 'byebug' anywhere in the code to stop execution and get a debugger console
  gem 'byebug', platforms: [:mri, :mingw, :x64_mingw]
end

group :development do
  # Access an interactive console on exception pages or by calling 'console' anywhere in the code.
  gem 'web-console', '>= 3.3.0'
  gem 'listen', '>= 3.0.5', '< 3.2'
  # Spring speeds up development by keeping your application running in the background. Read more: https://github.com/rails/spring
  gem 'spring'
  gem 'spring-watcher-listen', '~> 2.0.0'
end


# Windows does not include zoneinfo files, so bundle the tzinfo-data gem
gem 'tzinfo-data', platforms: [:mingw, :mswin, :x64_mingw, :jruby]
```

This is what my package.json looks like - again, pretty sparse. Perfect.:

```
{
  "name": "thegoodvine_app",
  "private": true,
  "dependencies": {
    "@rails/webpacker": "3.5",
    "activestorage": "^5.2.1",
    "rails-ujs": "^5.2.1",
    "stimulus": "^1.1.0",
    "turbolinks": "^5.2.0"
  },
  "devDependencies": {
    "webpack-dev-server": "2.11.2"
  }
}
```

### Smoke Test

Now, before we move on we're going to generate a pages controller with an index so we can see what the hell is going on. Before we do that, lets make sure our generator doesn't go and do something stupid. Jump into **./config/application.rb** and make a couple changes. Essentially, we're making sure that our generators don't bother with generating any css or js for us. Because nobody's got time for coffeescript. I've included the whole file, just to make sure there's no sneaky business (I got caught in this file before..)

```
require_relative 'boot'

require "rails"
# Pick the frameworks you want:
require "active_model/railtie"
require "active_job/railtie"
require "active_record/railtie"
require "active_storage/engine"
require "action_controller/railtie"
require "action_mailer/railtie"
require "action_view/railtie"
require "action_cable/engine"
# require "sprockets/railtie"
# require "rails/test_unit/railtie"

# Require the gems listed in Gemfile, including any gems
# you've limited to :test, :development, or :production.
Bundler.require(*Rails.groups)

module ThegoodvineApp
  class Application < Rails::Application
    # Initialize configuration defaults for originally generated Rails version.
    config.load_defaults 5.2

    config.generators do |g|
      g.test_framework  false
      g.stylesheets     false
      g.javascripts     false
    end

    # Settings in config/environments/* take precedence over those specified here.
    # Application configuration can go into files in config/initializers
    # -- all .rb files in that directory are automatically loaded after loading
    # the framework and any gems in your application.

    # Don't generate system test files.
    config.generators.system_tests = nil
  end
end

```

Perfect. Now we can generate that controller with: `$rails g controller pages index --no-assets` At this point, I updated the generated **pages/index.html.erb** to look like this so I could make sure that Webpacker was all wired up to this point.

```
<div data-controller="hello">
  <h1>Oh Hey There!</h1>
  <p>This is a big test to see if stimulus works. Lets see: <span data-target="hello.output"></span></p>
</div>
```

Run `./bin/webpack` to precompile your assets (its a good chance to make sure nothing is funny) ,then go ahead and start your server (`rails s`). Did it work? If so, great! You can go ahead and delete that **./assets/javascript/** folder. Or don't. It may break things down the road? Tough to tell. Let's move on. If not... well, I dunno.

At this point I also spun up a heroku server, just to see how it was behaving on a remote server in a different environment than my local one (linux machine vs localhosting on my Mac), but you certainly don't need to do that. 

### Bonus Test for Stimulus
Thanks to [Andy You](https://gist.github.com/andyyou/834e82f5723fec9d2dc021fb7b819517) for this example.

In **./app/views/pages/index.html.erb**, add:
```
...
<hr>
<div data-controller="clipboard members dashboard">
  PIN
  <input type="text" data-target="clipboard.source" value="1234" readonly>
  <button data-action="clipboard#copy" class="clipboard-button">
    Copy to Clipboard
  </button>
</div>
```

and create **./app/javascript/controllers/clipboard_controller.js**
```
import { Controller } from 'stimulus'

export default class extends Controller {
  static targets = ['source']
  initialize() {
    console.log('clipboard initialize')
  }
  connect() {
    console.log('clipboard connect')
    if (document.queryCommandSupported('copy')) {
      this.element.classList.add('clipboard--supported')
    }
  }
  copy(e) {
    e.preventDefault()
    this.sourceTarget.select()
    document.execCommand('copy')
  }
}
```
Boom! Even better!

## 3. Clean out Asset Pipeline - Part 2, Replace CSS with Tailwind.

Alright, lets start by installing Tailwind itself. We'll start first by adding it to our project with `yarn add tailwindcss`. Right. That's it! Jkjk, there's still some things we need to do. 

Next, let's create a home for our stylesheets in our Webpacker Managed Javascript folder with `$ mkdir -p app/javascript/stylesheets/components` and then we'll run the Tailwind Config file generator with `$ ./node_modules/.bin/tailwind init app/javascript/stylesheets/tailwind.js`

Perfect. That's the file that you will use to config all the fun Tailwind settings and what-nots. I'll let you [read their documentation](https://tailwindcss.com/docs/installation) for more details. 

**This was a pain to figure out. Massive thanks to [Chris (@excid3)](https://twitter.com/excid3)**

First, let's create our application's primary stylesheet. We can do that with `$ touch app/javascript/stylesheets/application.scss`. Note the **.scss**. It makes importing components later way the hell easier.

 Inside our **application.scss** file, you're going to paste the recommended code found on the Tailwind Website. The below code is was up to date as of November 13, 2018. If you're reading this and it's been a while, [go get fresh stuff](https://tailwindcss.com/docs/installation#3-use-tailwind-in-your-css).

```
/**
 * This injects Tailwind's base styles, which is a combination of
 * Normalize.css and some additional base styles.
 *
 * You can see the styles here:
 * https://github.com/tailwindcss/tailwindcss/blob/master/css/preflight.css
 *
 * If using `postcss-import`, use this import instead:
 *
 * @import "tailwindcss/preflight";
 */
@tailwind preflight;

/**
 * This injects any component classes registered by plugins.
 *
 * If using `postcss-import`, use this import instead:
 *
 * @import "tailwindcss/components";
 */
@tailwind components;

/**
 * Here you would add any of your custom component classes; stuff that you'd
 * want loaded *before* the utilities so that the utilities could still
 * override them.
 *
 * Example:
 *
 * .btn { ... }
 * .form-input { ... }
 *
 * Or if using a preprocessor or `postcss-import`:
 *
 * @import "components/buttons";
 * @import "components/forms";
 */

/**
 * This injects all of Tailwind's utility classes, generated based on your
 * config file.
 *
 * If using `postcss-import`, use this import instead:
 *
 * @import "tailwindcss/utilities";
 */
@tailwind utilities;

/**
 * Here you would add any custom utilities you need that don't come out of the
 * box with Tailwind.
 *
 * Example :
 *
 * .bg-pattern-graph-paper { ... }
 * .skew-45 { ... }
 *
 * Or if using a preprocessor or `postcss-import`:
 *
 * @import "utilities/background-patterns";
 * @import "utilities/skew-transforms";
 */


/* Import Components */
@import "./components/button";
 ```
Did you notice that line at the bottom? Whenever you want to spin off Tailwind components, you can do so by adding them as either css or scss files in your **./app/javascript/stylesheets/components** folder then referencing them at the bottom of your **application.scss** file.


Awesome. Lastly, lets go ahead and create a button component for your reference inside
**./app/javascript/stylesheets/application.scss** by first creating the file with `$ touch ./app/javascript/stylesheets/components/button.scss`

Add the following to our new button.scss file:

```
.btn {
  @apply .font-bold .py-2 .px-4 .rounded-sm;

  &.btn-blue {
    @apply .bg-purple .text-white;
  }

  &:hover {
    @apply .bg-purple-light;
  }
}
```
And just to try it out, somewhere on your **index.html.erb**, add a button that references the code with `<button class="btn btn-blue">Press Me</button>`


 If you try and run your server right now, you'll notice nothing's changed. What the hell? No worries, we just have to tell Webpacker where to find Tailwind and what to do with it! Let's start by adding the following to the **.postcssrc.yml** that was added to your project root folder by Webpacker I think? Or in this case, with the initial Rails New.

 ```
 plugins:
  postcss-import: {}
  postcss-cssnext: {}
++tailwindcss: './app/javascript/stylesheets/tailwind.js'
```
There. That last line is all we added. Easy right? All that's doing is telling our PostCss where to find our Tailwind config file. We also need to tell Webpack where to find our CSS file so it can be included as a stylesheet pack. Let's do that by adding the following line to **./app/javascript/packs/application.js**

```
...
const context = require.context("controllers", true, /.js$/)
application.load(definitionsFromContext(context))

// Import our project's application stylesheet, which happens to contain our Tailwind stuff.
++import "../stylesheets/application.scss"
```

Alright, lets jump into our **./app/views/layouts/application.erb.html** and make the following slight change:

```
--<%= stylesheet_link_tag    'application', media: 'all', 'data-turbolinks-track': 'reload' %>
++<%= stylesheet_pack_tag 'application', 'data-turbolinks-track': 'reload' %>
```

Nice! If you run ./bin/webpack then start your server, your *pages#index* should be looking pretty different! Lets just confirm this by making a couple tiny changes to it. Here's mine:

```
<div data-controller="hello" class="m-4">
  <h1 class="text-orange text-xl ">Oh Hey There!</h1>
  <p class="text-orange-dark">This is a big test to see if stimulis works. Lets see: <span class="text-red font-bold" data-target="hello.output"></span></p>
</div>
<hr>
<div data-controller="clipboard" class="m-4">
  PIN
  <input type="text" class="border rounded p-2" data-target="clipboard.source" value="1234" readonly>
  <button class="text-grey-lighter bg-orange-darker p-2 rounded-sm shadow hover:bg-red-darker hover:text-grey" data-action="clipboard#copy" class="clipboard-button">
    Copy to Clipboard
  </button>
</div>

<div class="m-4">
  <button class="btn btn-blue">
    Press Me
  </button>
</div>
```

It ain't pretty, but it should look like something! It also means we're nearly there! 

🚨Heads up: If you haven't changed anything in **./config/environments/production** and you try to run your app in production mode with something like `$ rails s -e production` you're not going to see any assets. Don't panic. It's just due to the line  that says `config.public_file_server.enabled = ENV['RAILS_SERVE_STATIC_FILES'].present?`  This is because normally static assets are handled by NGINX or Apache or wherever you're serving files from. Just run `$ RAILS_SERVE_STATIC_FILES=true rails s -e production` and you should be running along as fit as a fiddle. 

## Cleaning up CSS

Our last step now is going to be adding a couple PostCSS tools so that our final CSS file isn't a bajillion lines long. This section is thanks to [nerdcave](https://gist.github.com/nerdcave/ade26d701060d93e95ec83ddf58784b5) for being a great guy and sharing his stuff. Lets start by installing a few tools:

`yarn add glob-all purgecss-webpack-plugin`

First we're going to set it up is based on the recommendation of Tailwind. That one would be Purgecss. It's critical to reduce the CSS that tailwind creates from ~400kb uncompressed down to as little as 5kb.

I have it set up only in my Production environment (it's slow and unnessecary in development), but you do you. All you're going to do is add the following lines of code to either **./config/webpack/environment.js** or **./config/webpack/production.js** depending on if you want it in just the production environment, or in all environments. This is what it looks like in the **production.js** file:

```
process.env.NODE_ENV = process.env.NODE_ENV || 'production'

const environment = require('./environment')

/*
  PurgeCSS configuration for Rails 5 + Webpacker + Tailwind CSS 
  Optionally, put this in production.js if you only want this to apply to production.
  For example, your app is large and you want to optimize dev compilation speed.
*/
const path = require('path')
const PurgecssPlugin = require('purgecss-webpack-plugin')
const glob = require('glob-all')

// ensure classes with special chars like -mt-1 and md:w-1/3 are included
class TailwindExtractor {
  static extract(content) {
    return content.match(/[A-z0-9-:\/]+/g);
  }
}

environment.plugins.append('PurgecssPlugin', new PurgecssPlugin({
  paths: glob.sync([
    path.join(__dirname, '../../app/javascript/**/*.js'),
    path.join(__dirname, '../../app/views/**/*.erb'),
  ]),
  extractors: [ // if using Tailwind
    {
      extractor: TailwindExtractor,
      extensions: ['html', 'js', 'erb']
    }
  ]
}));

module.exports = environment.toWebpackConfig()
```


Done that? Well then I think we can go ahead and do the last big move, deleting the **./app/assets/stylesheets** folder - because baby, you're cooking with Webpacker now! 👨‍🍳🥘🔥

## 4. Clean out Asset Pipeline - Part 3, Replace /assets/images with javascript/images/

This step is the easy one! We saved the easiest for last. We can actually 
go the full nine yards and rip everything left in our **./app/assets/ folder right now. Lets start by doing exactly that with `$ rm -rf ./app/assets`

Done that? Scary right? 🌝

Turns out, Webpacker already has this built in for you. Nice eh? They kind of work similarly to stylesheets in that you will need to import them into your javascript before you can use them.

So first, let's give them a place to live. Create an images folder with `$ mkdir ./app/javascript/images`

Now, the technical way to add images would be to add the image to our images folder, then import each image individually into our project by adding it to our **./app/javascript/packs/application.js**

That's not fun. Instead, lets be a bit lazy. 

Add the following to the end of your **./app/javascript/packs/application.js**

```
require.context('../images/', true, /\.(gif|jpg|png|svg)$/i)
// Now within the views, you can call images using
// <%= image_tag asset_pack_path('images/abc.svg') %>
```

As I added in the comment, its now easy to add an image using the good ol' image tag and using an `asset_pack_path('images/image_name_here.jpg')`

Awesome! Oh, and just so you're aware, I'm pretty sure you can reference your images in your CSS/Sass and Webpacker will automatically hook them up for you. Generally using something like *images/imagename.jpg* should work as the root path will be the entry point to your CSS so you shouldn't need to use absolute paths or anything crazy.


## 5. Using .erb in JS

You may be thinking, no way, webpacker can do that too?! Or you may not be. Either way, yep, we can also add support for ERB files from within our **app/javascript** workflow. It's actually dead simple. Just  run `bundle exec rails webpacker:install:erb` on any ol' Rails app already setup with Webpacker. In this case, if you've been following along - this one! The config that Webpacker sets up for us gets us 99% of the way there, the only thing we need to change is we need to make sure you can use erb with StimulusJS files as well. To make that change, open **./app/javascript/packs/application.js**

and change this line:
```
...
const application = Application.start()
--const context = require.context("controllers", true, /.js$/)
++const context = require.context("controllers", true, /\.js(\.erb)?$/)
...

```
Done. Lets make sure it's working. Add this to your **./app/views/layouts/application.html.erb**:
`<%= javascript_pack_tag 'hello_erb' %>`

If all goes well, when you recompile, you should be able to check the browser console and see something new!





## Done?

Yep. I think so! Or at least I'm done writing. 


