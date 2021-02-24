# Redis stores for Ruby on Rails

__`redis-rails`__ provides a full set of stores (*Cache*, *Session*, *HTTP Cache*) for __Ruby on Rails__. See the main [redis-store readme](https://github.com/redis-store/redis-store) for general guidelines.

## A quick note about Rails 5.2

Rails 5.2.0 [includes a Redis cache store out of the
box](https://github.com/rails/rails/pull/31134), so you don't really
need this gem anymore if you just need to store the fragment cache in
Redis. Maintenance on the
[redis-activesupport](https://github.com/redis-store/redis-activesupport)
gem will continue for security and compatibility issues, but we are no
longer accepting new features. We are still actively maintaining all
other gems in the redis-store family, such as
[redis-actionpack](https://github.com/redis-store/redis-actionpack)
for session management, and
[redis-rack-cache](https://github.com/redis-store/redis-rack-cache)
for HTTP cache storage.

## Installation

Add the following to your Gemfile:

```ruby
gem 'redis-rails'
```

## Usage

`redis-rails` packages storage drivers for Redis which implement the
ActiveSupport fragment caching and ActionDispatch / Rack session
storage APIs. The following section(s) explain how to configure each
store:

### Rails Fragment Cache

Configure the fragment cache store in **config/environments/production.rb** like so:

```ruby
config.cache_store = :redis_store, "redis://localhost:6379/0/cache", { expires_in: 90.minutes }
```

The `ActiveSupport::Cache::Store` implementation assumes that your
backend store (Redis, Memcached, etc) will be available at boot time. If
you cannot guarantee this, you can use the `raise_errors: false` option
to rescue connection errors.

You can also provide a hash instead of a URL:

```ruby
config.cache_store = :redis_store, {
  host: "localhost",
  port: 6379,
  db: 0,
  password: "mysecret",
  namespace: "cache"
}, {
  expires_in: 90.minutes
}
```

### Session Storage

If you need session storage, consider directly using
[redis-actionpack](https://github.com/redis-store/redis-actionpack)
instead.

You can also store your session data in Redis, keeping user-specific
data isolated, shared, and highly available. Built upon [redis-rack](https://github.com/redis-store/redis-rack),
we present the session data to the user as a signed/encrypted cookie,
but we persist the data in Redis.

Add the following to your **config/initializers/session_store.rb** to
use Redis as the session store.

```ruby
MyApplication::Application.config.session_store :redis_store,
  servers: ["redis://localhost:6379/0/session"],
  expire_after: 90.minutes,
  key: "_#{Rails.application.class.parent_name.downcase}_session",
  threadsafe: true,
  secure: true
```

A brief run-down of these options...

- **servers** is an Array of Redis servers that we will attempt to find
  data from. This uses the same syntax as `:redis_store`
- **expire_after** is the default TTL of session keys. This is also set
  as the expiry time of any cookies generated by the session store.
- **key** is the name of the cookie on the client side
- **threadsafe** is for applications that run on multiple instances. Set
  this to `false` if you want to disable the global mutex lock on
  session data. It's `true` by default, meaning the mutex will be
  enabled.
- **signed** uses signed/encrypted cookies to store the local session on
  a client machine, preventing a malicious user from tampering with its
  contents.
- **secure** ensures HTTP cookies are transferred from server to client
  on a secure (HTTPS) connection
- **httponly** ensures that all cookies have the HttpOnly flag set to true

### HTTP Caching

We also provide [an adapter](https://github.com/redis-store/redis-rack-cache) for
[Rack::Cache](http://rtomayko.github.io/rack-cache/) that lets you store HTTP
caching data in Redis. To take advantage of this, add the following to
Gemfile:

```ruby
group :production do
  gem 'redis-rack-cache'
end
```

Then, add the following to **config/environments/production.rb**:

```ruby
# config/environments/production.rb
config.action_dispatch.rack_cache = {
  metastore: "redis://localhost:6379/1/metastore",
  entitystore: "redis://localhost:6379/1/entitystore"
}
```

### Usage with Redis Sentinel

You can also use [Redis Sentinel](https://redis.io/topics/sentinel) to manage a cluster of Redis servers
for high-availability data access. To do so, configure the sentinel
servers like so:

```ruby
sentinel_config = {
  url: "redis://mymaster/0",
  role: "master",
  sentinels: [{
    host: "127.0.0.1",
    port: 26379
  },{
    host: "127.0.0.1",
    port: 26380
  },{
    host: "127.0.0.1",
    port: 26381
  }]
}
```

You can then include this in your cache store configuration within
**config/environments/production.rb**:

```ruby
config.cache_store = :redis_store, sentinel_config.merge(
  namespace: "cache",
  expires_in: 1.days
)
config.session_store :redis_store, {
  servers: [
    sentinel_config.merge(
      namespace: "sessions"
    )
  ],
  expire_after: 2.days
}
```

## Usage with Redis Cluster

You can also specify only a subset of the nodes, and the client will discover the missing ones using the [CLUSTER NODES](https://redis.io/commands/cluster-nodes) command.

```ruby
config.cache_store = :redis_store, { cluster: %w[redis://127.0.0.1:6379/0/] }
```

## Running tests

```shell
gem install bundler
git clone git://github.com/redis-store/redis-rails.git
cd redis-rails
RAILS_VERSION=5.0.1 bundle install
RAILS_VERSION=5.0.1 bundle exec rake
```

If you are on **Snow Leopard**, run `env ARCHFLAGS="-arch x86_64" bundle exec rake`

## Status

[![Gem Version](https://badge.fury.io/rb/redis-rails.svg)](http://badge.fury.io/rb/redis-rails)
[![Build Status](https://secure.travis-ci.org/redis-store/redis-rails.svg?branch=master)](http://travis-ci.org/redis-store/redis-rails?branch=master)
[![Code Climate](https://codeclimate.com/github/redis-store/redis-rails.svg)](https://codeclimate.com/github/redis-store/redis-rails)

## Copyright

2009 - 2018 Luca Guidi - [http://lucaguidi.com](http://lucaguidi.com), released under the MIT license
