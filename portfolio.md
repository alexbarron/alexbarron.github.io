---
layout: page
title: Portfolio
permalink: /portfolio/
---
<p>Here's a quick overview of what I've built. My full Github account is here: {% include icon-github.html username="alexbarron" %}</p>

<h2><a href="http://github.com/alexbarron/hiko">Hiko - Flight Information App (AngularJS & Rails)</a></h2>
![Terminal Screenshot](/assets/hiko-map.png)
<h4>
  April 2016<br />
  Hiko is a tool for air travel nerds to look up flight, airport, and airline information.
</h4>
<ul>
  <li>Built front-end interface with AngularJS to dynamically create, load, and update content</li>
  <li>Built back-end API using Ruby on Rails</li>
  <li>Integrated Devise authentication gem with AngularJS and Rails to enable user registration</li>
  <li>Used Bootstrap to create basic styling</li>
  <li>Created custom Angular directive to display list of flights and their information</li>
  <li>Designed database schema and ActiveRecord associations for 5 Rails models</li>
  <li>Configured ActiveRecord joins association and database table to track users and the flights they’ve bought</li>
  <li>Used Active Model Serializers gem to package Rails back-end data into JSON</li>
  <li>Created custom Angular filter for displaying flights based on departure times</li>
  <li>Implemented Google Maps API in a custom directive to display geodesic flight paths</li>
  <li>Used Geocoder gem to geocode airports and support marker creation with the Google Maps API</li>
</ul>

<h2><a href="http://github.com/alexbarron/nba_fantasy_js">Fantasy NBA Game (Rails & jQuery)</a></h2>
![Terminal Screenshot](/assets/nba_fantasy.png)
<h4>
  March - April 2016<br />
 A fantasy basketball game with leagues, salary caps, real players, and more.
</h4>
<ul>
  <li>Used Sidekiq gem to process bulk updating of player stats and team scores</li>
  <li>Set up Facebook OmniAuth registration and log in</li>
  <li>Scraped Basketball Reference with Nokogiri</li>
  <li>Built Rails API and packaged JSON data with Active Model Serializers gem</li>
  <li>Used jQuery for dynamic content creation and loading</li>
  <li>Used HandlebarsJS to organize HTML templates in JavaScript</li>
  <li>Configured Devise for user authentication and registration</li>
  <li>Created player stat charts with ChartJS</li>
  <li>Maintained skinny controllers and views with Rails model and helper methods</li>
  <li>Built nested forms to create parent and child models within 1 form</li>
  <li>Configured CanCanCan gem to control permissions for admins, users, and visitors</li>
</ul>

<h2><a href="http://github.com/alexbarron/wedding-planner">Wedding Planning App (Sinatra)</a></h2>
![Terminal Screenshot](/assets/wedding-planner.png)
<h4>
  February - March 2016<br />
  A simple app for engaged couples to track wedding guests, vendors, and expenses.
</h4>
<ul>
  <li>Used Sinatra to build MVC web application</li>
  <li>Built user authentication flow with Bcrypt gem</li>
  <li>Restricted content access for correct users only with custom helper methods</li>
  <li>Used Boostrap to structure content</li>
  <li>Used Rack and MethodOverride to utilize PUT and DELETE methods in Sinatra forms</li>
</ul>

<h2><a href="http://github.com/alexbarron/nba-stats-cli-gem">NBA Stats CLI Ruby Gem (Ruby)</a></h2>
![Terminal Screenshot](/assets/nba-stats-terminal.png)
<h4>
  February 2016<br />
  A command line Ruby gem for quickly accessing up-to-date NBA player stats. Run "gem install nba-stats" in your terminal to install.
</h4>
<ul>
  <li>Built command line interface in Ruby</li>
  <li>Designed object-oriented Ruby structure to organize teams and players</li>
  <li>Packaged and published on rubygems.org</li>
  <li>Used Terminal Tables gem to present data to user</li>
  <li>Scraped Basketball Reference with Nokogiri</li>
</ul>

<h2><a href="http://github.com/alexbarron/gigglr">Gigglr - Comedy Show Tracker (Rails)</a></h2>
![Terminal Screenshot](/assets/gigglr.png)
<h4>
  October - November 2015(Before Flatiron School)<br />
  Gigglr helps stand up comedy fans never miss when their favorite comedians perform near them.
</h4>
<ul>
  <li>Used Ruby on Rails to build MVC structure</li>
  <li>Wrote Rspec and Capybara tests for unit and feature tests</li>
  <li>Used FactoryGirl and Faker gem for spoofing Rails models in tests</li>
  <li>Configured VCR and Webmock gems to mock API calls in Rspec tests</li>
  <li>Used Geocoder gem to geocode comedy club locations and users’ location</li>
  <li>Configured Delayed Jobs gem and Action Mailer to email users when a comedian adds a show near them</li>
  <li>Used Simple Form gem to create forms</li>
  <li>Implemented user registration and authentication with Devise gem</li>
  <li>Set up Paperclip gem to handle file upload of comedian pictures</li>
  <li>Used session variables to re-direct users to requested page after log in or registration</li>
  <li>Built and designed schema for Postgres database</li>
  <li>Created 2 join tables to track comedians’ followers and shows</li>
</ul>
