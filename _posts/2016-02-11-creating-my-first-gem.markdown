---
layout: post
title:  "Creating My First Gem"
date:   2016-02-11 19:20:21 -0800
categories: ruby gems nba
---
As part of the Learn-Verified course at Flatiron, we were assigned a project of creating a Ruby gem. The requirements were to pull in a set of data through scraping or an API, and develop a command line interface for drilling down into that data.

This was an excellent challenge because it required us to utilize a number of key skills from the course including object oriented design, mass assignment, manipulating data, scraping, and more. The other big challenge of this project was to figure out how to package a gem. This proved to not be that technically difficult, but it was very mentally rewarding. As a novice Ruby developer, the idea of contributing a gem seemed very far away. Certainly only developers with years of experience could contribute gems, right? Wrong. While my gem is not all that technically complicated, I'm proud to have a published gem out in the world for anyone to use.

I'm a huge basketball nerd so when I saw we had to create a CLI gem for scraping and accessing data, it was a no brainer to do something with NBA stats. My goal was to create a CLI gem where anyone could get a list of current NBA teams, get any team's current roster, and display any player's stats.

The first task I wanted to tackle on this project was scraping the team list. I find scraping really fun so I simply had to start there. I used basketball-reference.com to get the list of teams. Getting the list proved to be pretty simple, but organizing teams by Western and Eastern conferences was a little tougher. I decided to put this off until later because 1 alphabetized list of all 30 teams would be fine to start. After getting everything else working, I figured out how to filter teams by conference as shown in the final code below:

{% highlight ruby %}
class NbaStats::Scraper

  def self.open_page(url)
    Nokogiri::HTML(open(url))
  end

  def self.get_teams
    page = open_page("http://www.basketball-reference.com/leagues/NBA_2016_standings.html")

    east_teams = page.css("table#E_standings td a")
    west_teams = page.css("table#W_standings td a")

    assign_teams(west_teams, "West") + assign_teams(east_teams, "East")
  end

  def self.assign_teams(teams, conference)
    assigned_teams = []
    teams.each do |team|
      name = team.text
      team_url = "http://www.basketball-reference.com"+ team["href"]
      hash = {name: name, team_url: team_url, conference: conference}
      assigned_teams << hash unless assigned_teams.include? hash
    end
    assigned_teams
  end
...
end
{% endhighlight %}

As you can see above, I put all my scraping code in a Scraper class separate from my Team class. I knew I would be scraping data for my Player class eventually so I decided it'd be best to have all my scraping methods within one class.

The next challenge was getting all the player names for any team. I didn't want to scrape all 30 teams to get the roster for one team so in my previous team scraper, I saved the URL for each team page. This way if someone requested the Golden State Warriors, I already had their team page and could scrape just their roster.

Similarly, I didn't want to have to scrape stats for all 12-15 players on a team. I implemented the same solution of saving all the player URLs so I had just what I need to get the stats of any requested player.

Fortunately, all the roster lists and player stats were nicely organized in tables on Basketball Reference. Getting the roster just meant selecting each row in the roster table and assigning a row to a player. Each player's data such as their name, number, years of experience, and height was packaged up in a hash. A full team roster was an array of these player data hashes.

For the player stats, I just grabbed the last row from the Per Game stats table. I only wanted the last row becauase this row contained their 2015-16 season stats. Just like the player data hash from before, I assigned all the player's stats to a hash.

Check out the code below for getting the roster and player stats:

{% highlight ruby %}
  def self.get_roster(team)
    page = open_page(team.team_url)
    players_array = []
    players = page.css("table#roster tr")
    players.drop(1).each do |player|
      data_array = player.text.split("\n").map {|x| x.strip}
      number = data_array[1]
      name = data_array[2]
      position = data_array[3]
      height = data_array[4]
      data_array[8] == "R" ? experience = "Rookie" : experience = data_array[8] + " Years"

      player_url = player.css("a").map {|element| element["href"]}.first
      player_url = "http://www.basketball-reference.com" + player_url.to_s

      hash = {name: name, number: number, position: position, height: height, experience: experience, player_url: player_url}
      players_array << hash
    end
    players_array
  end

  def self.get_player_stats(player)
    page = open_page(player.player_url)

    season = page.css("table#per_game tr.full_table").last
    stats_array = season.text.gsub("\n", "").strip.split("   ")

    stats_hash = {
      points_pg: stats_array[29], 
      assists_pg: stats_array[24], 
      rebounds_pg: stats_array[23], 
      blocks_pg: stats_array[26], 
      steals_pg: stats_array[25], 
      minutes_pg: stats_array[7], 
      fg_percentage: stats_array[10], 
      three_percentage: stats_array[13], 
      ft_percentage: stats_array[20]
    }
  end
{% endhighlight %}

Now that I had all this team and player data, I wanted to display it nicely in the terminal. Rather than reinvent a way to cleanly display data in a terminal, I sought out a gem to help me do this. All it took was 1 Google search to turn up the terminal-table gem. After installing this gem, I just had to map my roster and player data to arrays that terminal-table could convert into a clean looking table. See the results below along with the code to accomplish this:

![Terminal Screenshot](/assets/nba-stats-terminal.png)

{% highlight ruby %}
  def display_roster(requested_team)
    team = NbaStats::Team.all.detect {|team| team.name == requested_team}
    team.add_players if team.players.empty?
    puts team.name + " roster:"
    rows = [["Number", "Name", "Position", "Height", "Experience"]]
    team.players.each do |player|
      rows << [player.number, player.name, player.position, player.height, player.experience]
    end
    puts Terminal::Table.new rows: rows
  end

  def display_player_stats(requested_player)
    player = NbaStats::Player.all.detect {|player| player.name == requested_player}
    stats_hash = NbaStats::Scraper.get_player_stats(player)
    player.add_player_stats(stats_hash)
    rows = [["Points/Game", "Assists/Game", "Rebounds/Game", "Blocks/Game", "Steals/Game", "FG%", "3P%", "FT%", "Minutes/Game",]]
    rows << [player.points_pg, player.assists_pg, player.rebounds_pg, player.blocks_pg, player.steals_pg, player.fg_percentage, player.three_percentage, player.ft_percentage, player.minutes_pg]
    puts "Here are #{player.name}'s 2015-16 stats: "
    puts Terminal::Table.new rows: rows
  end
{% endhighlight %}


Now that I had a working command line interface for getting NBA player stats, it was time to package everything up as a gem.

I won't lie, I was a little nervous about screwing this up, but I was super excited. Turns out there was no need to be nervous because there are so many resources out there for how to create and publish your own gem. The RubyGems.org guides were obviously fantastic. The most useful resource was just looking at a couple examples of how other people did it. Once I had seen some real world examples, I configured my own gemspec file, built it, and published it to RubyGems.org.

I then eagerly fired up a terminal on my other Mac and entered 'gem install nba-stats'. It all installed and I thought I was done. I got an error when I ran it though! Turns out I overlooked the crucial step of adding dependencies in my gemspec file. Even though I had required the terminal-table gem in my lib directory, my other Mac didn't have the terminal-table gem installed so it crashed when I tried to run the gem. After adding the required gem dependencies to the gemspec file, it all worked swimmingly!

If you'd like to play around with my gem or see the code, look up my gem <a href="https://rubygems.org/gems/nba-stats">here.</a>
