Development guide
=================

## Bare-metal

### Linux

In fact, all you need is described in the [production guide](Production-guide.md), **with the following exceptions**. You **don't** need:

- Nginx
- Systemd
- An `.env.production` file. If you need to set any environment variables, you can use an `.env` file
  - Use `LOCAL_HTTPS=false` if developing on the same machine
- To prefix any commands with `RAILS_ENV=production` since the default environment is "development" anyway
- Any cronjobs

The command to install Ruby project dependencies is the following:

    bundle install

Install JavaScript dependencies with this command:

    yarn install --pure-lockfile

By default the development environment wants to connect to a `mastodon_development` database on localhost using your user/ident to login to Postgres (i.e. not a md5 password)

To setup the `mastodon_development` database, run:

    bundle exec rails db:setup

You can then run Mastodon with:

    bundle exec rails server

Since 1.4, we are using Webpack, which in development environment needs to be started as well as the command above:

    ./bin/webpack-dev-server
    
Another, optional approach to managing the different processes starting (Rails, Webpack, Sidekiq, and the Streaming API) is to use the foreman tool.

    gem install foreman
    foreman start

Finally, open `http://localhost:3000` in your browser.

By default, your development environment will have an admin account created for you to use - the email address will be `admin@YOURDOMAIN` (e.g. admin@localhost:3000) and the password will be `mastodonadmin`.

You can run tests with:

    rspec

You can check localization status with:

    i18n-tasks health

And update localization files after adding new strings with:

    yarn manage:translations

You can check code quality with:

    rubocop

### OpenBSD

Follow the Linux setup as described above, but with these considerations:

- If you use a Ruby version manager (chruby, rbenv, rvm, etc.), you _must_
  configure Ruby with `CC=clang CXX=clang++`. This instructs Ruby to use that
  compiler when compiling native C gems.
- Many native C gems need to be told about `/usr/local`. You can do this by
  configuring a `build.gem_name` value using `bundle config`.
- Any C gem that uses mkmf.rb's `pkg_config` method might fail if the linker
  produces warnings, as happens when a library links with `sprintf(3)`. The
  `cld3` gem uses `pkg_config('protobuf')`; if you have protobuf installed but
  it cannot be found while building the gem, this is likely the problem. You
  will need to directly modify `mkmf.rb` to get this to install.

The bundle configuration as of Mastodon 2.0's Gemfile:

```sh
bundle config build.nokogiri --use-system-libraries --with-xml2-include=/usr/local/include/libxml2/ --with-opt-include=/usr/local/include --with-xslt-include=/usr/local/include/libxslt --with-exslt-include=/usr/local/include/libexslt --with-xml2-lib=/usr/local/lib
bundle config build.charlock_holmes --with-icu-dir=/usr/local --with-opt-dir=/usr/local
bundle config build.idn-ruby --with-idn-dir=/usr/local
```

Modify `mfmk.rb`:

```
@@ -655,7 +655,7 @@
   end
 
   def try_ldflags(flags, opts = {})
-    try_link(MAIN_DOES_NOTHING, flags, {:werror => true}.update(opts))
+    try_link(MAIN_DOES_NOTHING, flags, {:werror => false}.update(opts))
   end
 
   def append_ldflags(flags, *opts)
```

### Mac

These are self-contained instructions for setting up a development environment on a macOS system. It is assumed that you’ve cloned your fork of Mastodon to a local working directory and that you are in Terminal and in that directory.

#### Prerequisites

- Get [Xcode](https://developer.apple.com/xcode/) Command Line Tools:

	```
	xcode-select --install
	```

- Get [Homebrew](https://brew.sh) and use it to install the other dependencies:

	```
	brew install imagemagick ffmpeg yarn postgresql redis rbenv nodejs protobuf libidn
	```

- Configure Rbenv:

	```
	rbenv init
	rbenv install 2.4.1
	```

- Install/configure bundler to use your local rbenv:

	```
	gem update --system
	gem install bundler
	rbenv rehash
	```

- Configure [PostgreSQL](https://www.postgresql.org):

	```
	initdb /usr/local/var/postgres -E utf8
	createdb
	export PGDATA=/usr/local/var/postgres
	/usr/local/bin/postgres
	/usr/local/bin/psql
	```

	In the prompt:

	```
	CREATE USER mastodon CREATEDB;
	\q
	```

#### Installation

```
bundle install --with development
yarn install --pure-lockfile
gem install foreman --no-ri --no-rdoc
bundle exec rails db:setup
bin/rails assets:precompile
```

#### Running

In separate Terminal windows/tabs:

1. Start PostgreSQL: `/usr/local/bin/postgres`
2. Start Redis: `redis-server`
3. Start Mastodon (from the Mastodon folder): `foreman start`

Go to http://localhost:3000 to see your development instance.

Admin account is `admin@localhost:3000`. Password is `mastodonadmin`.

## Docker

You need Docker and Docker Compose.

- Build

```
docker-compose -f docker-compose.development.yml build
```

- Install Ruby project dependencies

```
docker-compose -f docker-compose.development.yml run --rm web bundle install
```

- Install JavaScript dependencies

```
docker-compose -f docker-compose.development.yml run --rm web yarn install --pure-lockfile
```

- Setup the database

```
docker-compose -f docker-compose.development.yml run --rm web ./bin/rails db:setup
```

- Run

```
docker-compose -f docker_compose.development.yml up
```

Go to `http://127.0.0.1` to see your development instance.

Admin account is `admin@localhost:3000`. Password is `mastodonadmin`.

### Federation

You can test federation by setting up multiple instances and sharing a network.

In this section, a procedure to set up these instances will be described:

|URI for Web browser|Domain for federation|
|-|-|
|`https://127.0.0.1/`|`mastodon1.localdomain`|
|`https://127.0.0.2/`|`mastodon2.localdomain`|

Note that HTTPS must be enabled. You will be asked by your browser if add an
exception of invalid certificates.

It is advisable to name your repository directory like `mastodon1`. The
directory will be devoted for `mastodon1.localdomain`.

- Create a new network

This network will be used as fake fediverse.

```
docker network create mastodon
```

- Create network configuration file

Save the following content as `docker-compose.development.network.yml`.

```YAML
version: '3'
networks:
  federation:
    external:
      name: mastodon
```

Make sure the network name (`mastodon` in this case) is the same which you
specified in the previous step.

- Create `.env` file

We will need several environment variables. `.env` automatically sets up those
variables so you don't have to type them again and again.

Save the following lines as `.env`.

```
COMPOSE_FILE=docker-compose.development.yml:docker-compose.development.network.yml
FRONTEND_DOMAIN=127.0.0.1
LOCAL_DOMAIN=mastodon1.localdomain
LOCAL_FEDERATION=mastodon
LOCAL_HTTPS=true
```

Make sure `COMPOSE_FILE` is pointing to `docker-compose.development.yml` and
the file you previously created (`docker-compose.development.network.yml`)

`FRONTEND_DOMAIN` is the IP address you access with a Web browser.

`LOCAL_DOMAIN` is the domain name used for federation. It must be suffixed
`.localdomain`, or a TLS authentication among instances will fail.

- Copy the instance

Now, copy the repository directory. You may name the destination directory
`mastodon2`.

- Edit `.env` of the new instance

Open `mastodon2/.env` and replace `127.0.0.1` with `127.0.0.2` and
`mastodon1.localdomain` with `mastodon2.localdomain`.

- Run

Run the following command in each directory (`mastodon1` and `mastodon2`):

```
docker-compose up
```

Go to `https://127.0.0.1` and `https://127.0.0.2` to see your development
instances. Again, `HTTPS` must be used.

Permanent links used in emails, toot links, media, and many others will not work
with this configuration because they uses the federation domains. If you want to
make them valid, add the following lines to your `/etc/hosts` (or a counterpart
if you are not on a Unix variant.)

```
127.0.0.1 mastodon1.localdomain
127.0.0.2 mastodon2.localdomain
```

## Development tips

You can use a localhost->world tunneling service like [ngrok](https://ngrok.com) if you want to test federation with remote instances, **however** that should not be your primary mode of operation. If you want to have a permanently federating server, set up a proper instance on a VPS with a domain name, and simply keep it up to date with your own fork of the project while doing development on localhost.

Ngrok and similar services give you a random domain on each start up. This is good enough to test how the code you're working on handles real-world situations. But as soon as your domain changes, for everybody else concerned you're a different instance than before.

When you are testing with a disposable instance you are polluting the databases of the real servers you're testing against, usually not a big deal but can be annoying. Instead of disposing multiple instances, record the exchanges from its web interface to create fixtures and test suites and work with them. Local federation with Docker is also a good alternative.

I advise to study the existing code and the RFCs before trying to implement any federation-related changes. It's not *that* difficult, but I think "here be dragons" applies because it's easy to break.

If your development environment is running remotely (e.g. on a VPS or virtual machine), setting the `REMOTE_DEV` environment variable will swap your instance from using "letter opener" (which launches a local browser) to "letter opener web" (which collects emails and displays them at /letter_opener ).
