---
layout: post
title:  "Re-factoring Gigglr: Optimizing View Rendering and Database Queries With New Relic"
date:   2016-05-16 07:58:21 -0800
categories: ruby rails newrelic optimization database
---

Re-visiting my Gigglr project that I built before Flatiron has been an eye-opening experience. While I built this only 7 months ago, it's amazing how obviously bad my code was back then. It's a little embarrassing looking back at this project, but I'm pleased with the growth I've made that enables me to see all the mistakes I made back then.

One of the mistakes that leaps off the page is how pages require redundant(N+1) database queries. When I initially built Gigglr, database calls felt free. Now I understand how excessive database calls can increase server costs and page load time. Increased production costs and slow performance is a lose-lose for both developers and users.

Scrolling through development server logs can provide a basic understanding of how many database queries are running for each page, but it's just not detailed or organize enough. That's where New Relic comes in.

<h3>New Relic</h3>

New Relic, a leader in application performance monitoring, offers a free Ruby gem for monitoring performance. Using the gem for a real production application will cost you, but we can get a lot of value out of it for free in development.

The New Relic gem works by monitoring each controller action for database queries, asset loading, load times, and more. These details are then saved in a report you can access at http://localhost:3000/newrelic or whatever URL you're running your local server at plus "/newrelic".

![Terminal Screenshot](/assets/new_relic_post/newrelic_main.png)

As you can see, it provides page response time, the URL, and how many SQL queries. Drilling down further into the Details gives us more information on that action with a pie chart and list of each SQL query.

![Terminal Screenshot](/assets/new_relic_post/pie_chart.png)

![Terminal Screenshot](/assets/new_relic_post/sql_list.png)

<h3>Optimizing Gigglr</h3>

Let's get to work on Gigglr. Looking over my report, there is one controller action that is embarassingly bad.

![Terminal Screenshot](/assets/new_relic_post/67_queries.png)

Ugh, 67 queries for the Comedians index! This is unacceptable. Let's get this down as much as possible.

Opening up the detailed SQL query list we see 20 queries that look like this:

{% highlight sql %}
SELECT COUNT(*) FROM "users" INNER JOIN "relationships" ON "users"."id" = "relationships"."user_id" WHERE "relationships"."comedian_id" = $1
{% endhighlight %}

Seems like this query is happening so we can display how many fans each comedian has. We're rendering 20 comedians so the current set up requires 20 database queries. I suspect this is the offending code:

{% highlight ruby %}
# views/comedians/comedian_box.html.erb
(<span id="fans-<%= comedian.id %>"><%= comedian.users.count %></span> <%= 'Fan'.pluralize(comedian.users.count) %>)
{% endhighlight %}

Just to confirm this is the offending code, let's remove this line from our view and see what happens.

![Terminal Screenshot](/assets/new_relic_post/77_queries.png)

Well that's surprising. Our SQL queries went up! There must be something else going on.

Another possible culprit is our Comedians Index controller action:

{% highlight ruby %}
# controllers/comedians_controller.rb
@comedians = Comedian.all.sort { |b,a| a.users.count <=> b.users.count }
{% endhighlight %}

This code sorts in Ruby which is generally a bad idea. We'll ignore that problem for now and outright delete the sorting.

{% highlight ruby %}
# controllers/comedians_controller.rb
@comedians = Comedian.all
{% endhighlight %}

Applying both these tweaks reduce our SQL queries to 47.

![Terminal Screenshot](/assets/new_relic_post/47_queries.png)

Now, the problem is we still want to sort comedians by how many fans they have, and we want to display their fan count. The easiest way to accomplish this without making too many queries is to simply add a "fan_count" column to the comedians table.

{% highlight ruby %}
class AddFanCountToComedians < ActiveRecord::Migration
  def change
    add_column :comedians, :fan_count, :integer
    update "UPDATE comedians SET fan_count = (SELECT COUNT(*) FROM relationships WHERE relationships.comedian_id = comedians.id)"
  end
end
{% endhighlight %}

Note that we're executing a SQL update statement to update all the comedians with their current fan count.

The next thing to do is set up the create and destroy relationships actions to update a comedian's fan count whenever a user follows or unfollows them.

{% highlight ruby %}
# models/user.rb
def follow(comedian)
  active_relationships.create(comedian_id: comedian.id)
  comedian.update(fan_count: comedian.fan_count + 1)
end

def unfollow(comedian)
  active_relationships.find_by(comedian_id: comedian.id).destroy
  comedian.update(fan_count: comedian.fan_count - 1)
end
{% endhighlight %}

Now we need to be re-implement sorting by fan count and displaying the fan count. 

{% highlight ruby %}
# views/comedians/comedian_box.html.erb
(<span id="fans-<%= comedian.id %>"><%= comedian.fan_count %></span> <%= 'Fan'.pluralize(comedian.fan_count) %>)

# controllers/comedians_controller.rb
@comedians = Comedian.order("fan_count DESC")
{% endhighlight %}

A quick test confirms that our sorting and fan count display work again and our SQL query count is still at 47. Perfect!

47 is still way too many so let's find the next thing to optimize. Here's a snapshot of our current SQL query list:

![Terminal Screenshot](/assets/new_relic_post/shows_queries.png)

Looks like we're pulling a lot one-by-one from the shows table:

{% highlight sql %}
SELECT "shows".* FROM "shows" INNER JOIN "bookings" ON "shows"."id" = "bookings"."show_id" WHERE "bookings"."comedian_id" = $1 AND (showtime > '2016-05-13 19:53:25.059987') ORDER BY showtime ASC LIMIT 3
{% endhighlight %}

The obvious suspect here is the code below that pulls the next 3 shows for each comedian.

{% highlight ruby %}
# views/comedians/comedian_box.html.erb
<h2>Upcoming Shows:</h2> 
<% comedian.future_shows.limit(3).each do |show| %>
  <p><%= link_to "#{show.short_date}", show %></p>
<% end %>

# models/comedian.rb
def future_shows
  self.shows.where("showtime > ?", Time.now).order("showtime ASC")
end
{% endhighlight %}

Sure enough, removing this code from our view drops the SQL query count to 27. 

![Terminal Screenshot](/assets/new_relic_post/27_queries.png)

This is a classic case of the N+1 Problem where as we iterate over each comedian, we're also querying their shows. To solve this, we'll need to use "includes" on our comedians query like so:

{% highlight ruby %}
# controllers/comedians_controller.rb
@comedians = Comedian.order("fan_count DESC").includes(:shows).where("shows.showtime > ?", Time.now ).references(:shows)

# views/comedians/comedian_box.html.erb
<h2>Upcoming Shows:</h2> 
<% comedian.shows.each do |show| %>
  <p><%= link_to "#{show.short_date}", show %></p>
<% end %>
{% endhighlight %}

Note that we're including a condition that we only want shows with a showtime after today's date. We also need to pass a shows reference whenever a condition is used. If we wanted all the shows, we could just use:

{% highlight ruby %}
# controllers/comedians_controller.rb
@comedians = Comedian.order("fan_count DESC").includes(:shows)
{% endhighlight %}

Okay, we're displaying our shows again, and we're all the way down to 22 queries! Here's what the SQL log looks like now:

![Terminal Screenshot](/assets/new_relic_post/relationships_queries.png)

There's still one more redundant query we need to eliminate:

{% highlight sql %}
SELECT 1 AS one FROM "comedians" INNER JOIN "relationships" ON "comedians"."id" = "relationships"."comedian_id" WHERE "relationships"."user_id" = $1 AND "comedians"."id" = $2 LIMIT 1
{% endhighlight %}

The guilty party is our helper method that relies on a User method called fan_of? to render the appropriate form for following or unfollowing comedians.

{% highlight ruby %}
# views/comedians/comedian_box.html.erb
<%= follow_unfollow(comedian) %>

# helpers/comedians_helper.rb
def follow_unfollow(comedian)
  if !!current_user && current_user.fan_of?(comedian)
    render partial: 'unfollow', locals: { comedian: comedian }
  elsif !!current_user
    render partial: 'follow', locals: { comedian: comedian }
  else
    button_to 'Follow', new_user_session_path, method: :get, class: 'btn btn-primary', params: { :com_id => comedian.id }
  end
end

# models/user.rb
def fan_of?(comedian)
  comedians.include?(comedian)
end

{% endhighlight %}

The fan_of? method is the real culprit here. The fan_of? method has valid uses in other parts of the application so I don't want to change it right now. A simple solution for this problem exists anyway.

To resolve the issue, I'll do 2 things.

1. Add users to our includes call in the comedians controller
2. Update the comedians helper to use include? instead of fan_of?

{% highlight ruby %}
# controllers/comedians_controller.rb
@comedians = Comedian.order("fan_count DESC").includes(:users, :shows).where("shows.showtime > ?", Time.now ).order("shows.showtime ASC").references(:shows)

# helpers/comedians_helper.rb
def follow_unfollow(comedian)
  if !!current_user && comedian.users.include?(current_user)
    render partial: 'unfollow', locals: { comedian: comedian }
  elsif !!current_user
    render partial: 'follow', locals: { comedian: comedian }
  else
    button_to 'Follow', new_user_session_path, method: :get, class: 'btn btn-primary', params: { :com_id => comedian.id }
  end
end
{% endhighlight %}

With those 2 changes, our SQL count drops to 6!

![Terminal Screenshot](/assets/new_relic_post/6_queries.png)

We could certainly optimize this further, but I'm satisfied with 6 calls for now. Besides, the Shows Index action is now the most query heavy with 18 SQL queries. Better work on reducing that before we further optimize the Comedians index action.

<h3>Conclusion</h3>

It's a truly eye-opening experience seeing how easy it is to make excessive SQL queries. I love Rails, but the downside of it is that the magic of it presents a lot of situations where the N+1 Problem rears its ugly face. Now that I know how to use tools like New Relic, I'll be sure to keep these in check in the future.