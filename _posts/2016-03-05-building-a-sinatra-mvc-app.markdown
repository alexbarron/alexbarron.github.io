---
layout: post
title:  "Building a Sinatra MVC App"
date:   2016-03-05 15:14:21 -0800
categories: ruby sinatra
---
<iframe width="560" height="315" src="https://www.youtube.com/embed/a90dFYhzNIw" frameborder="0" allowfullscreen></iframe>

I've been plugging away at the Learn-Verified program from Flatiron School. Since my last blog post about building a Ruby Gem, I've completed the sections for SQL, ORM, and Rack. The current section I'm finishing up is the Sinatra section. Similar to the OO Ruby section, the Sinatra section wraps up with a project where we build something of our own design and will be assessed on it.

For my Sinatra project, I built a simple wedding planning app. I'm in the midst of planning a wedding and wrangling guest lists and vendors has been tough to keep organized.

This app is certainly not an all-encompassing wedding planning app. It's primary focus is to help someone track their invited guests, RSVPd guests, and vendors.

In order for users to create a wedding in the app, the first thing they'll need to do is create an account. Sinatra makes it pretty easy to have user accounts and a login system. The login system uses sessions to ensure only approved users can see certain content and perform certain actions. Below is the code the login system:

{% highlight ruby %}
class ApplicationController < Sinatra::Base
  register Sinatra::ActiveRecordExtension
  set :views, Proc.new { File.join(root, "../views/") }
  set :method_override, true

  configure do
    enable :sessions
    set :session_secret, "password_security"
  end

  get "/" do
    erb :index
  end

  get "/signup" do
    if !logged_in?
      erb :signup
    else
      session[:message] = "You're already logged in!"
      redirect '/wedding'
    end
  end

  post "/signup" do
    if params[:username] != "" && params[:email] != "" && params[:password] != ""
      User.create(username: params[:username], email: params[:username], password: params[:password])
      session[:id] = user.id
      session[:message] = "Successfully signed up."
      redirect '/wedding'
    else
      session[:message] = "Sign up failed, try again."
      redirect '/signup'
    end
  end

  get "/login" do
    if !logged_in?
      erb :login
    else
      session[:message] = "You're already logged in!"
      redirect '/wedding'
    end
  end

  post "/login" do
    user = User.find_by(username: params[:username])
    if user && user.authenticate(params[:password])
      session[:id] = user.id
      session[:message] = "Successfully logged in."
      redirect '/wedding'
    else
      session[:message] = "Log in failed, try again."
      redirect "/login"
    end
  end

  get "/logout" do
    session.clear
    session[:message] = "Successfully logged out."
    redirect "/"
  end

  helpers do
    def logged_in?
      session[:id] ? true : false
    end

    def current_user
      User.find(session[:id])
    end

    def login_redirect
      session[:message] = "Please log in to see that."
      redirect '/login'
    end
  end
end
{% endhighlight %}

The helpers section at the end of the ApplicationController has some crucial methods to simplify controlling access. For example, the #login_redirect method handles all the situations when a user tries to access something before logging in or if they try to access content that doesn't belong to them. The cool thing about having this method is it provides a single point for editing how I want to handle these cases. I have 4 controllers using this #login_redirect method, and it would be highly error prone without a centralized method for handling access control.

Moving on to my models. A user is permitted to have 1 wedding. A wedding has many vendors and guests. 

Validations were an important part of this project and ActiveRecord makes it simple to ensure the models have the right data. Check out my models below:

{% highlight ruby %}
class Wedding < ActiveRecord::Base
  belongs_to :user
  has_many :guests, dependent: :destroy
  has_many :vendors, dependent: :destroy

  validates_presence_of :name

  def formatted_date
    self.date.strftime("%A %B %e, %Y")
  end

  def confirmed_guests
    self.guests.select {|guest| guest.rsvp }
  end

  def total_vendor_cost
    self.vendors.inject(0){|sum, vendor| sum + vendor.cost }
  end

  def avg_vendor_cost
    total_vendor_cost / self.vendors.count
  end
end

class Guest < ActiveRecord::Base
  belongs_to :wedding
  validates_presence_of :name
end

class Vendor < ActiveRecord::Base
  belongs_to :wedding
  validates_presence_of :name
  validates :cost, numericality: true
end
{% endhighlight %}

Another cool thing about ActiveRecord is the "dependent: :destroy." In my Wedding model, I used this function on the guests and vendors so whenever a wedding is deleted, all those guests and vendors are also deleted from the database. Without this, my database would quickly become cluttered with guests and vendors that don't belong to any wedding.

The next crucial part of this Sinatra app are the controllers for the wedding, guest, and vendor models.

Each of these controllers can handle showing content to the user, adding new content, editing existing content, and deleting content. For the most part, these 3 controllers all have similar code. Check them out below:

{% highlight ruby %}
class WeddingsController < ApplicationController

  configure do
    enable :sessions
    set :session_secret, "password_security"
  end

  get '/wedding' do
    if logged_in?
      if @wedding = current_user.wedding
        erb :'/wedding/show'
      else
        erb :'/wedding/no_wedding'
      end
    else
      login_redirect
    end
  end

  delete '/wedding/delete' do
    if logged_in?
      @wedding = current_user.wedding
      @wedding.destroy
      session[:message] = "Wedding deleted."
      redirect '/wedding'
    else
      login_redirect
    end
  end

  get '/wedding/new' do
    if logged_in? && !current_user.wedding
      erb :'/wedding/new'
    elsif current_user.wedding
      session[:message] = "You can't create more than 1 wedding per account. Please delete current wedding before creating another."
      redirect '/wedding'
    else
      login_redirect
    end
  end

  post '/wedding' do
    if logged_in? && params[:name] != ""
      Wedding.create(name: params[:name], location: params[:location], date: params[:date], user_id: current_user.id)
      redirect "/wedding"
    elsif params[:name] == ""
      session[:message] = "Name can't be empty."
      redirect "/wedding/new"
    else
      login_redirect
    end
  end

  get "/wedding/edit" do
    if logged_in?
      @wedding = current_user.wedding
      erb :'/wedding/edit'
    else
      login_redirect
    end
  end

  post "/wedding/edit" do
    if logged_in? && params[:name] != ""
      @wedding = current_user.wedding
      @wedding.update(name: params[:name], location: params[:location], date: params[:date])
      redirect "/wedding"
    elsif params[:name] == ""
      session[:message] = "Name can't be empty."
      redirect "/wedding/edit"
    else
      login_redirect
    end
  end

end

class VendorsController < ApplicationController

  configure do
    enable :sessions
    set :session_secret, "password_security"
  end

  get '/vendors' do
    if logged_in?
      @vendors = current_user.wedding.vendors
      erb :'/vendors/index'
    else
      login_redirect
    end
  end

  get "/vendors/new" do
    if logged_in?
      erb :'/vendors/new'
    else
      login_redirect
    end
  end

  get "/vendors/:id/edit" do
    if logged_in?
      @vendor = Vendor.find(params[:id])
      erb :'/vendors/edit'
    else
      login_redirect
    end
  end

  post "/vendors" do
    if logged_in? && params[:name] != "" && valid_cost(params[:cost])
      Vendor.create(name: params[:name], title: params[:title], cost: params[:cost].to_i, wedding_id: current_user.wedding.id)
      redirect "/vendors/#{Vendor.last.id}"
    elsif params[:name] == "" 
      session[:message] = "Name can't be empty."
      redirect "/vendors/new"
    elsif !valid_cost(params[:cost])
      session[:message] = "Cost must be a positive integer or 0."
      redirect "/vendors/new"
    else
      login_redirect
    end
  end

  post "/vendors/:id" do
    @vendor = Vendor.find(params[:id])
    if logged_in? && params[:name] != "" && valid_cost(params[:cost])
      @vendor.update(name: params[:name], title: params[:title], cost: params[:cost], wedding_id: current_user.wedding.id)
      redirect "/vendors/#{@vendor.id}"
    elsif params[:name] == ""
      session[:message] = "Name can't be empty."
      redirect "/vendors/#{@vendor.id}/edit"
    elsif !valid_cost(params[:cost])
      session[:message] = "Cost must be a positive integer or 0."
      redirect "/vendors/#{@vendor.id}/edit"
    else
      login_redirect
    end
  end

  get '/vendors/:id' do
    if logged_in?
      @vendor = Vendor.find(params[:id])
      erb :'/vendors/show'
    else
      login_redirect
    end
  end

  delete '/vendors/:id/delete' do
    @vendor = Vendor.find(params[:id])
    if logged_in? && @vendor.wedding.user == current_user
      @vendor.destroy
      session[:message] = "Vendor deleted."
      redirect '/vendors'
    else
      login_redirect
    end
  end

  helpers do
    def valid_cost(cost)
      if cost.to_i.to_s == cost && cost.to_i >= 0
        return true
      else
        return false
      end
    end
  end

end

class GuestsController < ApplicationController

  configure do
    enable :sessions
    set :session_secret, "password_security"
  end

  get '/guests' do
    if logged_in?
      @guests = current_user.wedding.guests
      erb :'/guests/index'
    else
      login_redirect
    end
  end

  get "/guests/new" do
    if logged_in?
      erb :'/guests/new'
    else
      login_redirect
    end
  end

  get "/guests/:id/edit" do
    if logged_in?
      @guest = Guest.find(params[:id])
      erb :'/guests/edit'
    else
      login_redirect
    end
  end

  post "/guests" do
    if logged_in? && params[:name] != ""
      Guest.create(name: params[:name], role: params[:role], rsvp: params[:rsvp], wedding_id: current_user.wedding.id)
      redirect "/guests"
    elsif params[:name] == ""
      session[:message] = "Name can't be empty."
      redirect "/guests/new"
    else
      login_redirect
    end
  end

  post "/guests/:id" do
    @guest = Guest.find(params[:id])
    if logged_in? && params[:name] != ""
      @guest.update(name: params[:name], role: params[:role], rsvp: params[:rsvp])
      redirect "/guests/#{@guest.id}"
    elsif params[:name] == ""
      session[:message] = "Name can't be empty."
      redirect "/guests/#{@guest.id}/edit"
    else
      login_redirect
    end
  end

  get '/guests/:id' do
    if logged_in?
      @guest = Guest.find(params[:id])
      erb :'/guests/show'
    else
      login_redirect
    end
  end

  delete '/guests/:id/delete' do
    @guest = Guest.find(params[:id])
    if logged_in? && @guest.wedding.user == current_user
      @guest.destroy
      session[:message] = "Guest deleted."
      redirect '/guests'
    else
      login_redirect
    end
  end

end
{% endhighlight %}

To complete this app, I just needed to put together some views to make all this content presentable to a user. Additionally, I added Bootstrap to make it a little more pretty and have a cleaner structure.

All in all this was a fun project that really hammered home HTTP methods and web application structure. I had built some Rails app prior to doing the Learn-Verified course, but there was always a lot of mystery with how things worked. Working through 48 lessons of Sinatra including this custom project has been great for illuminating how the nitty gritty of web applications work.

If you'd like to see the full code of this project, look up the GitHub repo here: <a href="https://github.com/alexbarron/wedding-planner">https://github.com/alexbarron/wedding-planner</a>
