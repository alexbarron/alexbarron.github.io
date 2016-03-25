---
layout: post
title:  "Building a Fantasy Basketball App with Rails"
date:   2016-03-24 16:14:21 -0800
categories: ruby rails
---
<iframe width="560" height="315" src="https://www.youtube.com/embed/KBvyz0Pf8j0" frameborder="0" allowfullscreen></iframe>

I’m excited to be on to the 3rd assessment project of Flatiron’s Learn-Verified course. The coding is all done and now I just owe a blog post and video walkthrough(see above).

For the 3rd project, we were tasked with building a Rails application. The project requirements were pretty extensive and it included things like have a “has_many through” relationship, nested form, nested resources, OmniAuth login, and more.

For my app, I decided to continue my basketball theme from the 1st assessment project by building a rudimentary fantasy basketball app. I had a blast building this and spent more time than I had planned working on it. There were just so many things I wanted to do with it.

NBA Fantasy had the following models: Users, Leagues, Teams, Players, and RosterSpots. Teams in this case refers to fantasy teams created by users, not real NBA teams. RosterSpots is a join model that tracks which teams have which players. A Player can be on multiple fantasy teams so I couldn’t use a simple “belongs_to :team” association.

Check out the associations below:

{% highlight ruby %}
class Player < ActiveRecord::Base
  has_many :roster_spots, dependent: :destroy
  has_many :teams, through: :roster_spots
end

class RosterSpot < ActiveRecord::Base
  belongs_to :team
  belongs_to :player
end

class Team < ActiveRecord::Base
  belongs_to :user
  belongs_to :league
  has_many :roster_spots, dependent: :destroy
  has_many :players, through: :roster_spots
end
{% endhighlight %}

In addition to fulfilling the project requirements, I made it a point to have skinny controllers, fat models, and low-logic views. I had built some small Rails apps before doing this course, and I had always made the common mistake of fat controllers, skinny models. I also committed the sin of having too much logic in my views. While building NBA Fantasy, this was something I wanted to correct.

I initially let myself write the code I normally would with lots of logic in views and controllers. After getting the core features set up, I reviewed my code and made it more “DRY.” This exercise of doing it the wrong way and then fixing it was really eye opening for why “DRY” is so important. Finding and fixing all the repetitive code was a time-consuming task. Now that I feel I have sufficiently tortured myself doing it the wrong way, I’ll be doing it the right way from the start from now on.

Another big takeaway from this clean up was the discovery of "polymorphic_paths" which allow you to use a single helper to dynamically generate paths for any model. I had links across NBA Fantasy for editing or deleting players, leagues, and teams. Instead of building all those links separately, I could dynamically build all of them with 1 helper ApplicationHelper#permitted_links(model).

Here are some of the partials, helpers, and model methods I used to slim down my views and controllers:

{% highlight ruby %}
# views/players/_player.html.erb
<tr>
  <td><%= link_to player.name, player %></td>
  <td><%= player.position %></td>
  <td><%= player.score %></td>
  <td><%= number_to_currency(player.salary, precision: 0) %></td>
  <td><%= player.value %></td>
  <td>On <%= pluralize(player.teams.count, 'team') %></td>
  <% if has_team_check? %>
    <td><%= add_player_column(player) %></td>
  <% end %>
</tr>

# helpers/application_helper.rb
module ApplicationHelper
  def permitted_links(model)
    if permitted?(model)
      simple_format(link_to("Edit #{model.name}", edit_polymorphic_path(model), class: "btn btn-primary")) + 
      link_to("Delete #{model.name}", polymorphic_path(model), method: :delete, data: {confirm: "Are you sure?"}, class: "btn btn-danger")
    end
  end
end

# helpers/leagues_helper.rb
module LeaguesHelper

  def join_leave_helper(league)
    if has_team_check?
      if league == current_user.team.league
        link_to "Leave League", leave_league_path(id: current_user.team), method: :post, class: "btn btn-danger btn-sm", data: {confirm: "Are you sure?"}
      elsif !!current_user.team.league
        link_to "Join League", join_league_path(id: current_user.team, league_id: @league.id), method: :post, class: "btn btn-success btn-sm", data: {confirm: "Are you sure? This will remove your team from your current league."}
      else
        link_to "Join League", join_league_path(id: current_user.team, league_id: @league.id), method: :post, class: "btn btn-success btn-sm"
      end
    else
      link_to "Create Team In This League", new_league_team_path(league)
    end
  end
end

# models/team.rb
class Team < ActiveRecord::Base

  def starters
    self.players.select do |player|
      player if player.status(self) == "Starter"
    end
  end

  def benchwarmers
    self.players.select do |player|
      player if player.status(self) == "Benched"
    end
  end

  def status_sorted_roster
    self.starters + self.benchwarmers
  end

end

# models/player.rb
class Player < ActiveRecord::Base

  def self.most_valuable_players
    Player.all.max_by(10) do |player|
      player.value
    end
  end

  def self.most_popular_players
    Player.all.max_by(10) do |player|
      player.teams.count
    end
  end

  def self.highest_scoring_players
    Player.all.max_by(10) do |player|
      player.score
    end
  end

end
{% endhighlight %}

When I first sat down to build NBA Fantasy, I was just going to build a template for people to create teams and and manually add player stats. However, because of my experience building the NBA CLI GEM, I couldn’t justify not having real NBA stats in there so I re-purposed some of my NBA scraping code to get real stats in NBA Fantasy.

This way whenever someone wants to add a player, all they have to do is copy and paste their profile URL from basketballreference.com. The player#update_info method does all the rest of the work to finding and save that player’s information into the database.

player#update_info is also an important method because it’s the key to updating team’s scores. Fantasy team scores should be updated every day there’s an NBA game. player#update_info does more than just pull in the data once, it also checks to see if that player’s “score” has changed. If it has changed, it calls player#update_teams_score which updates the score on every team that player is on.

Players can be updated individually, but they can also be updated in mass in the class method player#update_all_players. For now, an admin user must initiate this request manually. If this was a real world application, I’d setup a Rake/Cron task to automatically do this every day.

Check out the player info code below:

{% highlight ruby %}
# models/player.rb
class Player < ActiveRecord::Base

  def self.update_all_players
    Player.all.each {|player| player.update_info}
  end

  def update_info
    old_score = self.score
    info_hash = self.get_player_info
    info_hash[:score] = self.calculate_new_score(info_hash)
    if old_score != info_hash[:score]
      self.update(info_hash)
      self.update_teams_score(old_score, self.score)
    end
  end

  def get_player_info
    url = self.player_url
    page = Nokogiri::HTML(open(url))

    name = page.css("h1").first.text
    
    salary = get_salary(page)

    season = page.css("table#totals tr.full_table").last
    stats_array = array_maker(season)
    info_hash = {
      points: stats_array[30].to_i, 
      assists: stats_array[25].to_i, 
      rebounds: stats_array[24].to_i, 
      blocks: stats_array[27].to_i, 
      steals: stats_array[26].to_i, 
      games_played: stats_array[6].to_i, 
      position: stats_array[5],
      salary: salary.to_i,
      name: name
    }
    return info_hash
  end

  def update_teams_score(old_score, new_score)
    difference = new_score - old_score
    score = 0
    self.roster_spots.each do |roster_spot|
      if roster_spot.starter
        score = roster_spot.team.score + difference
      else
        score = roster_spot.team.score + (difference / 2)
      end
      roster_spot.team.update(score: score)
    end
  end
end
{% endhighlight %}

One of the things that proved surprisingly difficult was enforcing the salary cap. NBA Fantasy restricts all teams to a maximum of $70,000,000 in player salaries. Adding a player to an existing team worked fine and NBA Fantasy would reject that addition if the team didn’t have enough salary remaining.

But I had some trouble with the new team form that had check boxes for players. If I created a new team and selected several players that had a cumulative salary exceeding $70,000,000, I was able to cheat the salary cap. My solution to this was to create a team#enforce_salary_cap method that would drop players from the team until the team was below the salary cap. This worked by identifying the least valuable player on the team to drop. “Valuable” in this case means the player with the worst ratio of score to salary.

Check out the enforce_salary_cap code below:

{% highlight ruby %}
# models/team.rb
class Team < ActiveRecord::Base

  def update_salaries
    self.update(salary_remaining: 70000000 - salary)
  end

  def enforce_salary_cap
    dropped = 0
    while self.salary > 70000000
      least_valuable = self.players.min_by {|player| player.value }
      self.drop_player(least_valuable.id)
      self.reload.update_salaries
      dropped += 1
    end
    return dropped
  end

end
{% endhighlight %}

All in all this was a super fun project that I certainly want to add to in the future.

If you’d like to see my code, see the GitHub repo here: <a href="https://github.com/alexbarron/nba_fantasy">https://github.com/alexbarron/nba_fantasy</a>

You can also clone the repo and play with the app yourself. Be sure to run “bundle” and the migrations. If you want some real players and stats, run “rake db:seed”.