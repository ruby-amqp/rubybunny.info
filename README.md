# Bunny Documentation

This is a documentation site for [Bunny](http://rubybunny.info), a dead easy
to use RabbitMQ client for Ruby.


## Install Dependencies

To run the site generator yourself, you'll need to first have Ruby installed.
Python is also required, as syntax-highlighting of code blocks is handled by pygments.

If installing Ruby from source, a prerequisite is the libyaml dev package.

To install dependencies with Bundler:

    bundle install --binstubs

Then install Pygments via your OS-specific package installer, or else using `pip` (assuming you've installed pip):

    pip install pygments


## How to run a development server

    ./bin/jekyll --auto --server


## How to regenerate the site

In order to modify contents and launch dev environment, run:

    ./bin/jekyll


## License & Copyright

Copyright (C) 2012-2013 Michael S. Klishin.

Distributed under the MIT License.
