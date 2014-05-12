# ENVied [![travis](https://secure.travis-ci.org/eval/envied.png?branch=master)](https://secure.travis-ci.org/#!/eval/envied)

### TL;DR ensure presence and type of your app's ENV-variables.

For applications that are configured via ENV-variables, this gem will provide:

* A fail-fast check for presence of required ENV-variables
* A fail-fast check for type of required ENV-variables
* Access to typed ENV-variables (instead of just strings)

### Current status

![](underconstruction.gif)

## Usage

### 1) Configure

Let's configure the ENV-variables we need:

```ruby
# e.g. config/application.rb
ENVied.configure do
  variable :FORCE_SSL, :Boolean
  variable :PORT, :Integer
end
```

### 2) Check for presence and type

```ruby
ENVied.require
```

This will throw an error if not:
* both `ENV['FORCE_SSL']` and `ENV['PORT']` are present
* both the values of `ENV['FORCE_SSL']` and `ENV['PORT']` can be coerced (to resp. Boolean or Integer).

### 3) Use typed variables

Variables accessed via ENVied have the configured type:

```ruby
ENVied.PORT # => 3001
ENVied.FORCE_SSL # => false
```

## Configuration

### Groups

Groups give you more flexibility to define when variables are needed.  
It's just like you Gemfile:

```ruby
ENVied.configure do
  variable :FORCE_SSL, :Boolean
  
  group :production do
    variable :NEW_RELIC_LICENSE_KEY
  end
end

# For local development you would typically do:
ENVied.require(:default) #=> Only ENV['FORCE_SSL'] is required
# On the server:
ENVied.require(:default, :production) #=> ...also ENV['NEW_RELIC_LICENSE_KEY'] is required

# BTW the following are equivalent:
ENVied.require
ENVied.require(:default)
ENVied.require('default')
ENVied.require(nil)
```

### Types

The following types are supported:

* `:String` (implied)
* `:Boolean` (e.g. '0'/'1', 'f'/'t', 'false'/'true', 'off'/'on', 'yes','no' for resp. true or false)
* `:Integer`
* `:Symbol`
* `:Date` (e.g. '2014-3-26')
* `:Time` (e.g. '14:00')

### Defaults

In order to let other developers easily bootstrap the application, you can assign defaults to variables.
Defaults can be a value or a `Proc` (see example below).

Note that 'easily bootstrap' is quite the opposite of 'fail-fast when not all ENV-variables are present'. Therefor you should explicitly state whén defaults are allowed:

```ruby
ENVied.configure(enable_defaults: ENV['RACK_ENV'] == 'development') do
  variable :FORCE_SSL, :Boolean, default: false
  variable :PORT, :Integer, default: proc {|envied| envied.FORCE_SSL ? 443 : 80 }
end
```

Please remember that ENVied only **reads** from ENV; it doesn't set ENV-variables.
Don't let setting a default for, say `RAILS_ENV`, give you or your team the impression that `ENV['RAILS_ENV']` is set.  
As a rule of thumb you should only use defaults:
* in a development-environment
* for ENV-variables that your application introduces (i.e. for `ENV['DEFAULT_SENDER']` not for `ENV['REDIS_URL']`)

### A more extensive example:

```ruby
# We allow defaults for local development (and local tests), but want our CI
# to mimic our production as much as possible.
# New developers that don't have RACK_ENV set, will in this way not be presented with a huge
# list of missing variables, as defaults are still enabled.
not_production_nor_ci = ->{ !(ENV['RACK_ENV'] == 'production' || ENV['CI']) }
ENVied.configure(enable_defaults: &not_production_nor_ci) do
  # Your code will likely not use ENVied.RACK_ENV (better check Rails.env),
  # we want it to be present though; heck, we're using it in this file!
  variable :RACK_ENV
  
  variable :FORCE_SSL, :Boolean, default: false
  variable :PORT, :Integer, default: 3000
  variable :PUBLIC_HOST_WITH_PORT, :String, default: proc {|envied| "localhost:#{envied.PORT}" }
  
  group :production do
    variable :MAIL_PAAS_USERNAME
    variable :DATABASE_URL, :Symbol
  end
  
  group :ci do
    # ci-only stuff
  end
  
  group :not_ci do
    # CI needs no puma-threads, and sidekiq-stuff etc.
    # Define that here:
    variable :MIN_THREADS, :Integer, default: 1
    # more...
  end
end

# Depending on our situation, we can now require the groups needed:
# At local machines:
ENVied.require(:default, :development, :not_ci) or
ENVied.require(:default, :test, :not_ci)

# At the server:
ENVied.require(:default, :production, :not_ci)

# At CI:
ENVied.require(:default, :test, :ci)

# All in one line:
ENVied.require(:default, ENV['RACK_ENV'], (ENV['CI'] ? :ci : :not_ci))
```


## Installation

Add this line to your application's Gemfile:

    gem 'envied'

And then execute:

    $ bundle

## Testing

```bash
bundle install --binstubs

bin/rspec
# or
bin/rake
```

## Developing

```bash
bin/pry --gem
```


## Contributing

1. Fork it ( http://github.com/eval/envied/fork )
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request
