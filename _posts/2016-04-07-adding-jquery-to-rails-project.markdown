---
layout: post
title:  "Adding jQuery to Rails Project"
date:   2016-04-07 16:14:21 -0800
categories: ruby rails
---
<iframe width="560" height="315" src="https://www.youtube.com/embed/UEkgwQE-6II" frameborder="0" allowfullscreen></iframe>

My web development journey with Flatiron has been continuing nicely. I’m on to the 4th assessment project where we improved our Rails assessment project by adding dynamic jQuery features. I was very pleased to get to keep working on my fantasy NBA app because frankly it’s a fun thing to work on! 

The requirements for this project were heavily focused on building an internal API that could “get” and “post” data with JSON. In order to fulfill the requirements, I built out 3 new dynamic features.

<ol>
  <li>Dynamically load team rosters on team index pages</li>
  <li>Cycle through player show pages without page refresh</li>
  <li>Create players and dynamically display their information without page refresh</li>
</ol>

<strong>Dynamically load team rosters on team index pages</strong>

For this feature, I added an extra column to my team table partial that contained a “See Roster” button. By adding it to a partial, I was able to render this button across all the pages that contain a list of teams including the team index, league show, and team reports pages.

In addition to including a “See Roster” button, I also built a “Hide Roster” button that loads only after clicking “See Roster.” The current layout starts to look a little messy if a user clicks “See Roster” for multiple teams. To make it a little cleaner, users can hide whichever rosters they’re done looking at.

Loading the roster is accomplished with a get request that returns JSON data for the team. The JSON is serialized with help from the “active_model_serializer” gem. Now because the team data we’re actually interested in is their players’ data, I specified in my “TeamSerializer” file that a team has_many players. Then in a “TeamPlayerSerializer” file I defined that the only player data we needed in this case was their id and name.

Check out the code for this below:

{% highlight ruby %}
# views/teams/team_table.html.erb

<script type="text/javascript">
  $(document).ready(loadRoster);
  $(document).on('page:load', loadRoster);
  function loadRoster(){
    $(".js-roster").on("click", function() {
      var id = $(this).data("id");
      $.get('/teams/' + id + '.json', function(data){
        var players = data["team"]["players"];
        var rosterHTML = '<ol>'
        $.each(players, function(i, player){
          rosterHTML += '<li><a href="/players/' + player.id + '">' +  player.name + '</li>';
        })
        rosterHTML += '</ol><a href="#" class="js-hide-roster btn btn-primary btn-xs" data-id="' + id + '">Hide Roster</a>';
        $('#rosterCell-' + id).html(rosterHTML);
        hideRoster();
      })
    });
  };
  function hideRoster(){
    $(".js-hide-roster").on("click", function() {
      var id = $(this).data("id");
      $('#rosterCell-' + id).html('<a href="#" class="js-roster btn btn-primary btn-xs" data-id="' + id + '">See Roster</a>');
      loadRoster();
    });
  };
</script>

# serializers/team_serializer.rb

class TeamSerializer < ActiveModel::Serializer
  attributes :id, :name
  has_many :players, serializer: TeamPlayerSerializer
end

# serializers/team_player_serializer.rb

class TeamPlayerSerializer < ActiveModel::Serializer
  attributes :name, :id
end
{% endhighlight %}

<strong>Cycle through player show pages without page refresh</strong>

The next feature I’d like to highlight is located on the player show pages. I’ve added buttons for loading previous and next players in order of id. This way a user who loads Russell Westbrook’s profile page can quickly cycle through other player profile pages without any page refresh.

Loading the player data uses a JavaScript function that responds to either the previous or next button. If a user clicks the previous button, it passes the string of “previous” to the loadPlayer function. loadPlayer checks to see if the string passed is “previous” or “next” and then increments or decrements the current player’s id so the appropriate player data is loaded.

Similar to the team data, the player data is formatted using a serializer to make nice, clean JSON data. In this case, I wanted nearly all the player’s data including name, position, and all their stats. I also wanted some values that weren’t saved in the database such as what their “value” is and how many teams they’re on. The active_model_serializer gem enables you to set custom values to be passed so I created methods in the “PlayerSerializer” to fulfill these requests.

Check out the full code below:

{% highlight ruby %}
# views/players/show.html.erb

<a href="#" class="js-previous btn btn-primary" data-id="<%=@player.id%>">Load Previous Player</a>
<a href="#" class="js-next btn btn-primary" data-id="<%=@player.id%>">Load Next Player</a>

<script type="text/javascript">
  $(document).ready(setListeners);
  $(document).on('page:load', setListeners);
  function setListeners(){
    $(".js-next").on("click", function() {
      loadPlayer("next");
    });
    $(".js-previous").on("click", function() {
      loadPlayer("previous");
    });
  }
  function loadPlayer(direction){
    var id;
    if(direction === "next"){
      id = parseInt($(".js-next").attr("data-id")) + 1;
    } else if (direction === "previous"){
      id = parseInt($(".js-previous").attr("data-id")) - 1;
    }
    
    $.get("/players/" + id + ".json", function(data) {
      $(".js-next").attr("data-id", data.player.id);
      $(".js-previous").attr("data-id", data.player.id);
      // Set top information
      $("h1").text(data.player.name + '(' + data.player.position + ')');
      $("h3").text('Score: ' + data.player.score);
      $("#salary").text('Salary: $' + data.player.salary.toLocaleString());
      $("#value").text('Score Per Million: ' + data.player.value);
      if(data.player.team_count === 1){
        $("#team_count").text('Num of Teams: ' + data.player.team_count + " team");
      } else {
        $("#team_count").text('Num of Teams: ' + data.player.team_count + " teams");
      }
      // Set stats
      $("#points").text(data.player.points);
      $("#assists").text(data.player.assists);
      $("#rebounds").text(data.player.rebounds);
      $("#blocks").text(data.player.blocks);
      $("#steals").text(data.player.steals);
      $("#games_played").text(data.player.games_played);
    });
  }
</script>

# serializers/player_serializer.rb

class PlayerSerializer < ActiveModel::Serializer
  attributes :id, :name, :position, :score, :salary, :points, :rebounds, :assists, :blocks, :steals, :games_played, :value, :team_count

  def value
    object.value
  end

  def team_count
    object.teams.count
  end
end
{% endhighlight %}

<strong>Create players and dynamically display their information without page refresh</strong>

The last feature to describe is creating players and dynamically displaying their data on the new player page. 

The new player form is very simple and just requires the user to input a Basketball Reference player profile page. The dynamic creation works by sending a POST request to the server with the Basketball Reference URL. The controller then takes in that URL, creates a new player, and then saves. The Player model has an “after_save” method that does the scraping work of getting the name, salary, and stats of the new player.

Once the player’s data has been fully scraped and committed to the database, the controller passes back the player’s JSON data to the new player page. Using jQuery, I reveal a previously hidden div with the id of playerResult that will take in and display all the player’s data in a nice table similar to the player show page.

Here’s the code for this feature:

{% highlight ruby %}
# views/players/new.html.erb

<%= render 'form' %>

<h3 id="status"></h3>

<div id="playerResult">
  
  <p id="score"></p>
  <p id="salary"></p>
  <p id="value"></p>

  <table class="table table-striped table-condensed">
    <tr>
      <th>Points</th>
      <th>Assists</th>
      <th>Rebounds</th>  
      <th>Blocks</th>
      <th>Steals</th>
      <th>Games Played</th>
    </tr>
    <tr>
      <td id="points"><%= @player.points %></td>
      <td id="assists"><%= @player.assists %></td>
      <td id="rebounds"><%= @player.rebounds %></td>
      <td id="blocks"><%= @player.blocks %></td>
      <td id="steals"><%= @player.steals %></td>
      <td id="games_played"><%= @player.games_played %></td>
    </tr>
  </table>
</div>

<script type="text/javascript">
  $(function () {
    $('form').submit(function(event) {
      $("#status").text("Loading new player...");
      event.preventDefault();
      
      var player_url = $(this).serialize();
      var posting = $.post('/players', player_url);
      posting.done(function(data){
        $("#playerResult").show();
        $("#status").text("Successfully added " + data.player.name);
        $("#score").text('Score: ' + data.player.score);
        $("#salary").text('Salary: $' + data.player.salary.toLocaleString());
        $("#value").text('Score Per Million: ' + data.player.value);
        $("#points").text(data.player.points);
        $("#assists").text(data.player.assists);
        $("#rebounds").text(data.player.rebounds);
        $("#blocks").text(data.player.blocks);
        $("#steals").text(data.player.steals);
        $("#games_played").text(data.player.games_played);
      })
    });
  });
</script>

# controllers/players_controller.rb

def create
    @player = Player.find_or_create_by(player_params)
    if @player.save
      render json: @player, status: 201
    else
      flash[:alert] = @player.errors.full_messages.first
      render :new
    end
  end

{% endhighlight %}

It's been a great learning experience how to integrate these dynamic features into a Rails app with JSON and jQuery. The next section is all about Angular and seems like we'll eventually be integrating both Angular and Rails which should be really interesting.

If you’d like to see my code for this project, see the GitHub repo here: <a href="https://github.com/alexbarron/nba_fantasy_js">https://github.com/alexbarron/nba_fantasy_js</a>