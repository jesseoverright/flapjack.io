## Developing

Clone the repository:

```
git clone https://github.com/flapjack/flapjack.git
```

Install development dependencies:

```
# Install Ruby dependencies
gem install bundler
bundle install
# Install Go dependencies and build binaries
./build.sh
```

You'll also need Redis installed:

```
# Mac OS X with Homebrew:

brew update
brew install redis

# ... and follow the instructions on adding redis as a launch agent (launchctl load etc)

# Debian based Linux:

apt-get update
apt-get install redis
```

Flapjack is, and will continue to be, well tested. Monitoring is like continuous
integration for production apps, so why shouldn't your monitoring system have tests?

Unit testing is done with RSpec, and unit tests live in `spec/`.

To run the unit tests, check out the code and run:

```
rake spec
```

The following environment variables will change the behaviour of the rspec tests:

- `SHOW_LOGGER_ALL` - if set, will print out all logger messages
- `SHOW_LOGGER_ERRORS` - if set, will print out all logger messages at ERROR or FATAL level
- `COVERAGE` - enables SimpleCov for code coverage reporting, see below

Integration testing is done with Cucumber, and integration tests live in `features/`.

To run the integration tests, check out the code and run:

```
rake features
```

NB, if the cucumber tests fail with a [spurious lexing error](https://github.com/cucumber/gherkin/issues/182) then try this:

```
cucumber -f fuubar features
```

If you have a failing scenario and would like to see the log leading up to the error, you can put in a line like the following above the failing line in the scenario:

```gherkin
  And show me the lovely log
```

You can use whatever adjective you like in there, so tune it to suit the mood of the moment.

API client integration tests are done with pact. To verify:

```
rake pact:verify
```

Code Coverage Reporting
-----------------------

To engage [SimpleCov](https://github.com/colszowka/simplecov) for a code coverage report, set the COVERAGE environment variable before running one (or both) of the test suites:

```
COVERAGE=x rake spec
COVERAGE=x rake features
open coverage/index.html
```

Note that SimpleCov will merge the results of the two test suite's measured code coverage if you run them both within a 10 minute timeout.

Startup and Shutdown
--------------------
Copy the example configuration file into place:

```
if [ ! -e etc/flapjack_config.yaml ] ; then
  cp etc/flapjack_config.yaml.example etc/flapjack_config.yaml
else
  echo "you've already got a config file at etc/flapjack_config.yaml, exiting"
fi
```

Ensure your local Redis server is running, and then to start:

```
FLAPJACK_ENV=development bundle exec bin/flapjack --config etc/flapjack_config.yaml server start
```
Stop:

```
FLAPJACK_ENV=development bundle exec bin/flapjack --config etc/flapjack_config.yaml
```

Get the status:

```
FLAPJACK_ENV=development  bundle exec bin/flapjack --config etc/flapjack_config.yaml server status
```

Flapjack can also be started in the foreground (non-daemonized) by adding `--no-daemonize` to the start command, eg:

```
FLAPJACK_ENV=development bundle exec bin/flapjack --config etc/flapjack_config.yaml server start --no-daemonize
```

When running, check you can access the web interface at [localhost:3080](http://localhost:3080/). The port for development can be modified in etc/flapjack_config.yaml under `development` - `gateways` - `web` - `port`.

Releasing
---------

Gem releases are handled with [Bundler](http://gembundler.com/rubygems.html).

Before building the gem for release, you need to do a bit of housekeeping:

- Update the flapjack version string

```
vi lib/flapjack/version.rb
```

- Update the changelog - add the new version and list each issue it addresses. The [Releases](https://github.com/flapjack/flapjack/releases) github page will help you discover which commits have been pushed to master since the last release.

```
vi CHANGELOG.md
```

- Update the bundle (travis (among others) will get upset if you don't):

```shell
bundle
```

- Run the tests (to be sure, to be sure)

```
bundle exec rake spec && \
bundle exec rake features && \
bundle exec rake pact:verify && \
cd src/flapjack && go test -v
```

- Fix the tests, or abort the release mission, if any tests are failing.
- Commit (use the actual new version string in the commit message below)

```
git commit -a -m 'prepare v0.0.0 release'
```

To build the gem, run:

```
bundle exec rake build
```

To push the gem to rubygems.org run:

```
bundle exec rake release
```

Once the gem has been released, you'll most likely be wanting to build and upload the [omnibus package](https://github.com/flapjack/omnibus-flapjack/) using the instructions [here](https://github.com/flapjack/omnibus-flapjack/blob/master/README.md)

You can then test the latest package with [vagrant-flapjack](https://github.com/flapjack/vagrant-flapjack):

```
git clone https://github.com/flapjack/vagrant-flapjack.git && cd vagrant-flapjack
flapjack_component=experimental distro_release=trusty vagrant up
```

Data Structures
---------------
See [Data Structures](../DATA_STRUCTURES)

RESTful API for input, output and actions
-----------------------------------------
See [API](../../jsonapi)

Importing via the command line
------------------------------
See [Importing](../../usage/IMPORTING)

Architecture
------------

FIXME document check data format, for writing new check receivers
