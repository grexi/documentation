# Deploying a Ruby on Rails application

In this tutorial we're going to show you how to migrate an existing Rails
application to the [cloudControl] platform. You can find the [source code on Github][example-app]
and check out the [Ruby buildpack][ruby buildpack] for supported features. The
application in question is a fork of Michael Hartl's [Rails tutorial]'s Sample
App and is a Twitter clone.

## The Rails Application Explained

### Get the App

To start, first clone the application from the previously mentioned git repository:
~~~bash
$ git clone git://github.com/cloudControl/ruby-rails-example-app.git
$ cd ruby-rails-example-app
~~~

Now you have the original version of the app. This version should work locally
on you machine, but is still not ready to be deployed on the platform.

### Dependency Tracking

The Ruby buildpack tracks dependencies with [RubyGems]. Those are defined in
the Gemfile which is placed in the root directory of the project. To install
them execute the `bundle install` command.

### Testing

The app has an exhaustive set of tests. Check that all the tests are
passing locally.

~~~bash
$ bundle exec rake db:migrate
$ bundle exec rake db:test:prepare
$ bundle exec rspec spec/
~~~

All the tests should pass at this point, so  you can now run a server locally
to get acquainted with the app.
~~~bash
$ rails s
~~~

Now that the app is working, it's time to prepare it for deployment on the
platform.


## Preparing the App

### Creating the App

Choose a unique name to replace the `APP_NAME` placeholder for your application
and create it on the cloudControl platform. Be sure that you're inside the git
repository when running the command:
~~~bash
$ cctrlapp APP_NAME create ruby
~~~

### Process Type Definition

cloudControl uses a [Procfile] to know how to start your processes.

Create a file called `Procfile` with the following content:
~~~
web: bundle exec rails s -p $PORT
~~~

Left from the colon we specified the **required** process type called `web`
followed by the command that starts the app and listens on the port specified
by the environment variable `$PORT`.

### Configuring the asset pipeline

Add the following code into the `Application` class defined in the `config/application.rb`:
~~~ruby
module SampleApp
  class Application < Rails::Application

    ...

    # Do not initialize on precompile in the build process
    # as this can fail, e.g. if database is being accessed in the process
    # and there is no benefit in doing it in build process anyway
    config.assets.initialize_on_precompile = false if ENV['BUILDPACK_RUNNING']

  end
end
~~~

### Production database

By default, Rails 3 uses SQLite for all the databases, even the production one.
However, it is not possible to use SQLite in the production environment on the
platform, reason being the [Non Persistent Filesystem][filesystem]. To use a
database you should choose an Add-on from [Data Storage category][data-storage-addons].

In this  tutorial we use PostgresSQL with the [ElephantSQLAdd-on][postres-addon].
Modify the `Gemfile` by moving the `sqlite3` line to ":development, :test"
block and add a new ":production" group with 'pg' and 'cloudcontrol-rails'
gems.

The `Gemfile` should now have the following content:
~~~ruby
source 'http://rubygems.org'

gem 'rails', '3.1.10'
gem 'gravatar_image_tag', '1.0.0.pre2'
gem 'will_paginate', '3.0.4'

group :development do
  gem 'annotate', '2.4.0'
  gem 'faker', '0.3.1'
end

group :test do
  gem 'webrat', '0.7.1'
  gem 'spork', '0.9.0.rc8'
  gem 'factory_girl_rails', '1.0'
end

group :development, :test do
  gem 'rspec-rails', '2.6.1'
  gem 'sqlite3', '1.3.4'
end

group :production do
  gem 'pg'
  gem 'cloudcontrol-rails', '0.0.6'
end

group :assets do
  gem 'sass-rails'
  gem 'coffee-rails'
  gem 'uglifier'
end

gem 'jquery-rails'
~~~

Don't forget to run the `bundle install` command after you saved your changes.
Note that this might fail in certain operating systems due to missing
dependencies. In that case you can ignore installing the production gems
locally by executing `bundle install --without production`.

Additionally you have to change the "production" section of
`config/database.yml` file to:
~~~
production:
  adapter: postgresql
  encoding: utf8
  pool: 5
  host:
  port:
  database:
  username:
  password:
~~~

That's all, your app is now ready to use the PostgreSQL database.

## Pushing and Deploying your App

Push your code to the application's repository, which triggers the deployment image build process:
~~~bash
$ cctrlapp APP_NAME/default push
Counting objects: 5, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 296 bytes, done.
Total 3 (delta 1), reused 0 (delta 0)

-----> Receiving push
-----> Installing dependencies using Bundler version 1.2.1
       Running: bundle install --without development:test --path vendor/bundle --binstubs bin/ --deployment
       Using rake (10.1.0)
       Using multi_json (1.2.0)
       ...
       Using will_paginate (3.0.4)
       Your bundle is complete! It was installed into ./vendor/bundle
       Cleaning up the bundler cache.
-----> Preparing app for Rails asset pipeline
       Running: rake assets:precompile
       /usr/bin/ruby1.9.1 /srv/tmp/builddir/vendor/bundle/ruby/1.9.1/bin/rake assets:precompile:nondigest RAILS_ENV=production RAILS_GROUPS=assets
       Asset precompilation completed (3.73s)
-----> Rails plugin injection
       Injecting rails_log_stdout
       Injecting rails3_serve_static_assets
-----> Building image
-----> Uploading image (8.7M)

To ssh://APP_NAME@cloudcontrolled.com/repository.git
 * [new branch]      master -> master
~~~

Now add the free option of the ElephantSQL Add-on to the "default" deployment
and then deploy it:
~~~bash
$ cctrlapp APP_NAME/default addon.add elephantsql.turtle
~~~

Finally, you need to run the migrations on the database - to do so, use the [run command]:
~~~bash
$ cctrlapp APP_NAME/default run "rake db:migrate"
~~~

Congratulations, you should now be able to reach the app at http://APP_NAME.cloudcontrolled.com.

For additional information take a look at [Ruby on Rails notes][rails-notes] and
other [ruby-specific documents][ruby-guides].

[Ruby on Rails]: http://rubyonrails.org/
[RubyGems]: http://rubygems.org/
[cloudControl]: http://www.cloudcontrol.com
[cloudControl-doc-user]: https://www.cloudcontrol.com/dev-center/Platform%20Documentation#user-accounts
[cloudControl-doc-cmdline]: https://www.cloudcontrol.com/dev-center/Platform%20Documentation#command-line-client-web-console-and-api
[ruby buildpack]: https://github.com/cloudControl/buildpack-ruby
[procfile]: https://www.cloudcontrol.com/dev-center/Platform%20Documentation#buildpacks-and-the-procfile
[git]: https://help.github.com/articles/set-up-git
[bundler]: http://gembundler.com/
[filesystem]: https://www.cloudcontrol.com/dev-center/Platform%20Documentation#non-persistent-filesystem
[data-storage-addons]: https://www.cloudcontrol.com/dev-center/Add-on%20Documentation/Data%20Storage/
[mysqls]: https://www.cloudcontrol.com/dev-center/Add-on%20Documentation/Data%20Storage/MySQLs
[gem itself]: http://rubygems.org/gems/cloudcontrol-rails
[database-conf]: https://www.cloudcontrol.com/dev-center/Guides/Ruby/Read%20configuration#adding-relational-databases
[example-app]: https://github.com/cloudControl/ruby-rails-example-app
[rails-notes]: https://www.cloudcontrol.com/dev-center/Guides/Ruby/Ruby%20on%20Rails%20notes
[Rails tutorial]: http://ruby.railstutorial.org/
[postres-addon]: https://www.cloudcontrol.com/dev-center/Add-on%20Documentation/Data%20Storage/ElephantSQL
[ruby-guides]: https://www.cloudcontrol.com/dev-center/Guides/Ruby
[run command]: https://www.cloudcontrol.com/dev-center/Guides/Ruby/SSH%20session
