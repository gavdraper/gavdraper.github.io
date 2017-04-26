---
layout: post
title: Rails Continuous Testing
date: '2014-03-11 07:57:00'
---

Coming from the .Net world where continuous testing is expensive on both the wallet and processing power seeing what Ruby has to offer in this area is pretty incredible. 

![alt text](/content/images/autotestGrowl.gif "Animation of Growl Notifications")

You can pretty much bring in a few gems and have continuous testing with awesome notifications out of the box. For the below example I'm using Ruby on Rails to create a new project and enable continuous testing but similar would be possible just using Ruby.

To start with switch to your terminal and fire off the following commands to create a new Rails app and scaffold some stuff ready for testing....

```language-bash
rails new awesomeApp
cd awesomeApp
rails generate scaffold user name:string password:string date_added:datetime
rake db:migrate
rake test
```

You now have a new Rails application with an initialized test database. We then need to add the following to gemFile file in the root of your application...

```language-ruby
group :development, :test do
	gem 'autotest'
    gem 'autotest-rails'   
end
```

Once you have done that run the following to download those gems if you don't already have them.

```language-bash
bundle install
```

Then run autotest at the terminal to start the test runner.

You should see the output from the tests the Rails scaffolder created. Try going in and changing one of the tests or controllers and you'll see in the terminal window that the unit tests are automatically re run. That's pretty cool, you could stick the terminal window on a second screen if you wanted to get feedback t a glance every time you change a file.

Even better though for OSX users if you install "Growl" from the app store you can configure the Rails project to give you Growl notifications. This means you then can minimize the terminal window and still see when you break or fix broken tests. To enable this once you have installed Growl run the following command in a terminal window...

```language-bash
echo "require 'autotest/growl'" >> .autotest
```

This tells autotest to use the growl notification gem. For that to work we need to add the growl gem to gemFile. Just add the following line into the :development/:test group you created above

```language-bash
gem 'autotest-growl'
```

If you then run autotest again you should see notification balloons pop up on your screen when tests break or go from broken to fixed. 