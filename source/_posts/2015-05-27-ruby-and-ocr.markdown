---
layout: post
title: "Ruby &amp; OCR"
date: 2015-05-27 12:45:41 -0700
comments: true
categories: ruby,api,sinatra
---

#### (Adapted from [@yburyug's](http://www.github.com/ybur-yug) original Python guide)

## Why?
A) Obviously it just sounds fun as shit.

B) Real world use. I mean, we do have to go paperless somehow right?
OCR has become common. With the advent of libraries such as Tesseract, Ocrad, and more people
have built hundreds of bots and libraries that use them in interesting ways. A trivial example is Meme Reading
bots on Reddit. Extracting text from images on sites like Tumblr or Pinterest that have overlay text commonly
also can be used in order to further add natural language analysis data into models to predict it.

<!-- more -->

## Beginning steps

There are two potential options for building this in the simplest scale. We will go through building the server
layer here as a simple means to have an API resource we can hit from a frontend framework in order to make
and application based around this. The other is to generate the HTML serverside. We will take the former approach.

First, we have to install some dependencies. As always, configuring your environment is 90% of the fun.

> This post has been tested on Ubuntu version 14.04 but it should work for 12.x and 13.x versions as well. If you're running OSX, you can use [VirtualBox](http://osxdaily.com/2012/03/27/install-run-ubuntu-linux-virtualbox/) or a droplet on [DigitalOcean](https://www.digitalocean.com/) (recommended!) to create the appropriate environment.


### Downloading dependencies

We need [Tesseract](http://en.wikipedia.org/wiki/Tesseract_%28software%29) and all of its dependencies, which includes [Leptonica](http://www.leptonica.com/), as well as some other packages that power these two for sanity checks to start. Now, I could just give you a list of magic commands that I know work in the env. But, lets explain things a bit first.

> NOTE: You can also use the [_run.sh](link) shell script to quickly install the dependencies along with Leptonica and Tesseract and the relevant English language packages. If you go this route, skip down to the [Web-server time!](link) section. But please consider manually building these libraries if you have not before. It is chicken soup for the hacker's soul to play with tarballs and make. However, first are our regular apt-get dependencies (before we get fancy). 

```sh
$ sudo apt-get update
$ sudo apt-get install autoconf automake libtool
$ sudo apt-get install libpng12-dev
$ sudo apt-get install libjpeg62-dev
$ sudo apt-get install g++
$ sudo apt-get install libtiff4-dev
$ sudo apt-get install libopencv-dev libtesseract-dev 
$ sudo apt-get install git 
$ sudo apt-get install cmake 
$ sudo apt-get install build-essential
$ sudo apt-get install libleptonica-dev
$ sudo apt-get install liblog4cplus-dev
$ sudo apt-get install libcurl3-dev
$ sudo apt-get install python2.7-dev
$ sudo apt-get install tk8.5 tcl8.5 tk8.5-dev tcl8.5-dev
$ sudo apt-get build-dep python-imaging --fix-missing
```

We run `sudo apt-get update` is short for 'make sure we have the latest package listings.
`g++` is the GNU compiled collection.
We also get a bunch of libraries that allow us to toy with images. ie `libtiff` `libpng` etc. 
We also get `git`, which if you lack famliiarity with but have found yourself here, you may want to read [The Git Book](link).
Beyond this, we also ensure we have `Python 2.7`, our programming language of choice.
We then get the `python-imaging` library set up for interaction with all these pieces.

Speaking of images: now, we'll need [ImageMagick](http://www.imagemagick.org/) as well if we want to toy with images 
before we throw them in programmatically, now that we have all the libraries needed to understand and parse them in..

```sh
$ sudo apt-get install imagemagick
```

### Building Leptonica

Now, time for Leptonica, finally! (Unless you ran the shell scripts and for some reason are seeing this. In which case proceed to the [Webserver Time!](link) section. 

```sh
$ wget http://www.leptonica.org/source/leptonica-1.70.tar.gz
$ tar -zxvf leptonica-1.70.tar.gz
$ cd leptonica-1.70/
$ ./autobuild
$ ./configure
$ make
$ sudo make install
$ sudo ldconfig
```

If this is your first time playing with tar, heres what we are doing:

- get the binary for Leptonica (`wget`)
- unzip the tarball  and  (`x` for extract, `v` for verbose...etc. For a detailed explanation: `man tar`)
- `cd` into our new unpacked directory
- run `autobuild` and `configure` bash scripts to set up the application
- use `make` to build it
- install it with `make` after the build
- create necessary links with `ldconfig`

Boom, now we have Leptonica. On to Tesseract!

### Building Tesseract

And now to download and build Tesseract...

```sh
$ cd ..
$ wget https://tesseract-ocr.googlecode.com/files/tesseract-ocr-3.02.02.tar.gz
$ tar -zxvf tesseract-ocr-3.02.02.tar.gz
$ cd tesseract-ocr/
$ ./autogen.sh
$ ./configure
$ make
$ sudo make install
$ sudo ldconfig
```
The process here mirrors the Leptonica one almost perfectly. So for explanation, I'll keep this DRY and just say see above.

We need to set up an environment variable to source our Tesseract data, so we'll take care of that now:

```sh
$ export TESSDATA_PREFIX=/usr/local/share/
```

Now, lets get the Tesseract english language packages that are relevant:

```
$ cd ..
$ wget https://tesseract-ocr.googlecode.com/files/tesseract-ocr-3.02.eng.tar.gz
$ tar -xf tesseract-ocr-3.02.eng.tar.gz
$ sudo cp -r tesseract-ocr/tessdata $TESSDATA_PREFIX
```

BOOM! We now have Tesseract. We can use the CLI. Feel free to read the [docs](https://code.google.com/p/tesseract-ocr/) if you want to play. However, we need a Python wrapper to truly achieve our end goal. So the next step is setting
up a Flask server that will allow us to easily build an API that we will POST requests to with a link to an image, and it will run the character recognition on them.

## Web-server time!

Now, on to the fun stuff. First, we will need to build a way to interface with Tesseract via Ruby. We COULD use `popen` 
but that just is wrong. The prolific [Meh](gh link) has created a very robust Ruby wrapper for tesseract that we will
be utilizing for this. It has the most features of any wrapper that I have found thus far, and is a joy to use. In this
initial guide we will go over simply using its basic API, but will dive in further in a later post.

Building this server shouldn't be much of a challenge, but let's assume you might have never had to do so before. I prefer
to use [Sinatra](http://www.sinatrarb.com) for simple API's like this one. Since we have a blank slate, we may as well take
the TDD approach.

```sh
mkdir ocr_api
cd ocr_api
mkdir lib spec
touch spec/integration_spec.rb
```

Now, run

`bundle init`

so we have a Gemfile, and we can open that up with 

`vim Gemfile`

(I use vim, but replace that with any editor of your choice's unix command)

```ruby
...
gem 'sinatra'
gem 'tesseract-ocr'

group :development, :test do
  gem 'pry'
  gem 'rspec'
end
...
```

and now in `integration_spec.rb`

```ruby
require 'rack/test'
require 'rspec'
require './lib/server'

require File.expand_path '../../my-app.rb', __FILE__

ENV['RACK_ENV'] = 'test'

module RSpecMixin
  include Rack::Test::Methods
  def app() Sinatra::Application end
end

RSpec.configure { |c| c.include RSpecMixin }
Spec::Runner.configure { |c| c.include RSpecMixin }

# The above gets our rack env set up for sinatra testing

describe "Our Sinatra App" do
  it "should allow access to the root route" do
    get "/"
    last_response.should be_ok
  end
end
```

now we can actually make a file to pass this test...

`vim lib/server.rb`

```ruby
require 'sinatra'

get "/" do
  "hello world"
end
```

Now, let's create a simple class to get our image and get text for it.

`lib/server.rb`

```ruby
require 'open-uri'
...
class OcrEngine
  def initialize
    @engine = Tesseract::Engine.new { |e| e.language = :eng }
  end

  def parse_image url
    @engine.text_for(get_image(image))
  end

  private

  get_image img_url
    open(img_url).read
  end
end
...
```

And now if we define our route to take a POST request instead of a GET request:

```
post '/' do
  if params['image_url']
    OcrEngine.new.parse_image(params['image_url']
  else
    "Please supply an `image_url` parameter
  end
end
```


# This is super ghetto. I'm gonna refine it in the coming day(s). Don't worry
