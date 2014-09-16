æsh-site
============

Source of [Æsh Website](http://aeshell.github.io)


Prerequisites
-------------

The site is built using awestruct, and its dependencies are best installed using the ruby gem `bundler`. 

1. Make sure you have a modern, sane Ruby install. The best way to install Ruby is to use RVM from <https://rvm.io/>
2. Install the ruby gem `bundler` 
        
        gem install bundler
3. Install the necessary gems for the site. At the root of the site directory, run
        
        bundle install

Developing the site
-----------------

If you run Awestruct locally, it will watch for changes to your sources and regenerate the site. Run

    bundle exec awestruct -d

and browse to <http://localhost:4242>.
