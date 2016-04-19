---
layout: post
title:  "Combining AngularJS and Rails"
date:   2016-04-19 11:31:21 -0800
categories: ruby rails angularjs javascript
---
<iframe width="560" height="315" src="https://www.youtube.com/embed/oqjOT1LgGog" frameborder="0" allowfullscreen></iframe>

For the final project of the Flatiron School's Learn-Verified program, we were challenged to build an app with a Rails backend API and an AngularJS front end. Going through the AngularJS part of the Learn-Verified program was the most challenging part of the course so I had no doubt this project would also be the most challenging. OO Ruby, Sinatra, and Rails all made sense to me very quickly. AngularJS, however, tripped me up a lot. I still have a lot to learn and I fully intend to supplement my AngularJS studying with more studying and building. Building this project was a big confidence booster though.

Here were the requirements for the project:

<li>Must use an Angular Front-End that includes at least 5 pages</li>
<li>Must contain some sort of nested views</li>
<li>Must contain some sort of searching as well as filtering based on some criteria. Ex: All items in the "fruit" category, or all tasks past due</li>
<li>Must contain at least one page that allows for dynamic updating of a single field of a resource. Ex: Allow changing of quantity in a shopping cart</li>
<li>Links should work correctly. Ex: Clicking on a product in a list, should take you to the show page for that product</li>
<li>Data should be validated in Angular before submission</li>
<li>Must talk to the Rails backend using $http and Services.</li>

I'm a big air travel nerd so I decided to make an app for storing and displaying information about flights, airlines, and airports. The app is called "Hiko" which means flight in Japanese. Hiko is pronounced "he-ko" in Japanese.

The first thing I built for this app were all the Rails models, controllers, and serializers. I created 3 models and they associated with each other as such:

{% highlight ruby %}
# models/airport.rb
class Airport < ActiveRecord::Base
  has_many :departures, foreign_key: :origin_id, class_name: "Flight"
  has_many :arrivals, foreign_key: :destination_id, class_name: "Flight"
  geocoded_by :name
  after_validation :geocode
end

# models/airline.rb
class Airline < ActiveRecord::Base
  has_many :flights
end

# models/flight.rb
class Flight < ActiveRecord::Base
  belongs_to :origin, foreign_key: :origin_id, class_name: "Airport"
  belongs_to :destination, foreign_key: :destination_id, class_name: "Airport"
  belongs_to :airline
  has_many :passengers

  ...
end
{% endhighlight %}

The Flight and Airport models have a slightly complicated set up because it was important for flights to know their origin and destination airports. Similarly, airports should know their departing and arriving flights. This way I can use calls like "airport.departures" or "flight.destination" to quickly access that information. This was enabled by having "origin_id" and "destination_id" attributes on the Flight model. "origin_id" and "destination_id" are actually just airport ids.

Another thing worth mentioning is that I installed the Geocoder ruby gem to geocode airports. This automatically grabs and stores the latitude and longitude coordinates for an airport. We'll need this for rendering flight path maps with the Google Maps API. More on this later.

Controllers for these models were also built of course, but it's not worth posting all that code. All you need to know is the controllers are only set up to render JSON data that the Angular app will consume.

<h3>Angular & Rails API Service</h3>

In the Angular part of my app, the first thing I wanted to get working was the API service to pull data from the Rails backend. I accomplished this by creating a service called 'BackendService.js'. This service handles all the API calls for flights, airports, and airlines. Another option for this could have been building individual factories for each model, but I figured having one service was a nice way to consolidate and save code.

The BackendService doesn't use separate functions for each model. Instead, it takes in a method parameter for which resource we're accessing. So if I wanted to get all the flights in my database I would call "BackendService.allRecords("flights")" or call "BackendService.getRecord("flights", id)" to get an individual flight.

Check out the code for BackendService.js as well as an example use case below:

{% highlight javascript %}
// services/BackendService.js
function BackendService($http){
  this.allRecords = function(resource){
    return $http.get('http://localhost:3000/' + resource + '.json');
  }
  this.getRecord = function(resource, id){
    return $http.get('http://localhost:3000/' + resource + '/' + id + '.json');
  }

  this.createRecord = function(resource, params){
    return $http.post('http://localhost:3000/' + resource, params);
  }

  this.updateRecord = function(resource, params){
    return $http.put('http://localhost:3000/' + resource + '/' + params.id, params);
  }

}

// controllers/AirportIndexController.js
function AirportIndexController(airports, $filter, BackendService, $location){
  ...

  ctrl.createAirport = function(){
    BackendService.createRecord("airports", ctrl.airport).success(function(data){
      ctrl.filteredList.unshift(data.airport);
      ctrl.airport = {};
      $location.path('airports');
    });
  };

  ...
}
{% endhighlight %}

<h3>Rendering Nested Views</h3>

I implemented nested views in a few places. One of them was nesting the airline show and new form views in the airline index view. The table for displaying the full list of airlines is only 2 columns so I decided to make it a sidebar rather than a full content page. I stuffed the list of all airlines in a Bootstrap "col-md-4" column and then left the remaining space blank until a user clicked an airline's name or clicked the 'Add Airline' button. Then the white space would be filled by the HTML for showing an airline's information or creating a new airline.

{% highlight html %}
// views/airlines/index.html
<div class="row">
  <div class="col-md-4">
    <h3>Airlines <a href="" ui-sref="airlines.new" class="btn btn-primary">Add Airline</a></h3>
    <table class="table table-striped table-condensed">
      <tr>
        <th>Airline Name</th>
        <th>Airline Code</th>
      </tr>
      <tr ng-repeat="airline in ctrl.airlines | orderBy: 'name'">
        <td><a href="" ui-sref="airlines.airline({id: airline.id})">{{ airline.name }}</a></td>
        <td>{{ airline.code }}</td>  
      </tr>
    </table>
  </div>
  <div class="col-md-8">
    <div ui-view></div>
  </div>
</div>

// views/airlines/show.html
<h3>
  Flights on {{ ctrl.airline.name }} ({{ctrl.airline.code}}) 
  <a href="" ui-sref="airlines.airline.edit" class="btn btn-primary">Edit</a>
</h3>

<div ui-view></div>

<input ng-model="ctrl.search" ng-change="ctrl.refilter()" placeholder="Search Flights"/>

<table class="table table-striped table-condensed">
  <tr>
    <th></th>
    <th>Flight Number</th>
    <th>Departure Time</th>
    <th>Departure Airport</th>
    <th>Arrival Time</th>  
    <th>Arrival Airport</th>
  </tr>
  <tr ng-repeat="flight in ctrl.filteredList | orderBy: 'departure'">
    <td><a href="" ui-sref="flight({id: flight.id})" class="btn btn-primary btn-xs">Details</a></td>
    <td>{{AirlineShow.airline.code}} {{ flight.id }}</td>
    <td>{{ flight.departure | date:'short' }}</td>
    <td>{{ flight.origin }}</td>
    <td>{{ flight.arrival | date:'short' }}</td>  
    <td>{{ flight.destination }}</td>
  </tr>
</table>

// views/airlines/new.html
<h2>Add Airline</h2>
<form name="form" ng-submit="form.$valid && ctrl.createAirline()">
    <ng-include src="'views/airlines/_form.html'"></ng-include>
  <input type="submit" value="Create Airline" class="btn btn-success btn-sm">
</form>
{% endhighlight %}

The forms for adding airports and flights also use nested views that pop up only after a user clicks the appropriate button.

<h3>Searching & Filtering</h3>
Another requirement of this project was to be able to search and filter lists. For the filter, we were tasked with allowing users to filter based on some condition like a category of food, past due tasks, or in my case, flight dates. Hiko will store both historical and future flight information so I set up buttons for users to dynamically view all flights, past flights, or future flights on the flight index page.

Each filter button on the flight index page is bound to a FlightIndexController function called "filterDates" that can take in a function parameter of how the user wants to filter the flights. Originally, "filterDates" did all the work itself, but in order to make search work nicely with date filtering, I ended up delegating the work to another function called "makeList".

"makeList" works by checking how the user wants to filter the flights and then pushes each matching flight into an array called "filteredList". For example, if the user wants past flights the function will check to see if a flight's departure date was before or after the current time. If it's before, it pushes that into "filteredList".

{% highlight javascript %}
// controllers/FlightIndexController.js
function FlightIndexController(flights, $filter, BackendService, $location){
  ...

  ctrl.makeList = function(direction){
    var now = new Date;
    var flights = ctrl.flights;
    ctrl.filteredList = [];
    if (direction === "all"){
      ctrl.filteredList = ctrl.flights;
      filterStatus = "all";
    } else if (direction === "past"){
        for(var i = 0; i < flights.length; i++){
          if (new Date(flights[i].departure) < now){
            ctrl.filteredList.push(flights[i]);
          } 
        }
        filterStatus = "past";
    } else{
        for(var i = 0; i < flights.length; i++){
          if (new Date(flights[i].departure) > now){
            ctrl.filteredList.push(flights[i]);
          }
        }
        filterStatus = "future";
    }
    return ctrl.filteredList;   
  }

  ctrl.filterDates = function(direction){
    ctrl.search = "";
    return ctrl.makeList(direction);
  }

  ...
}

{% endhighlight %}

As for search, it proved a little tricky to get search and filters working well together. My original search implementation ignored any filter the user clicked and just searched all the flights. This was obviously not good because if the user clicked 'Past Flights' and then searched, they should only get search results that are considered historical flights.

In order to solve this, I kept track of what filter the user applied in a variable called "filterStatus". By default, "filterStatus" is set to "future". Each time the user clicked a filter, I updated this variable. See the above code in the "makeList" function for this.

The function that does the search work is called "searchList" and it calls on the "makeList" function while passing in the "filterStatus" variable. The "makeList" function returns a list of flights that is then searched/filtered based on the current search query. 

{% highlight javascript %}
// controllers/FlightIndexController.js
function FlightIndexController(flights, $filter, BackendService, $location){
  ...

  var filterStatus = "future";

  ctrl.searchList = function(){
    ctrl.filteredList = $filter('filter')(ctrl.makeList(filterStatus), ctrl.search);
  };

  ...
}

{% endhighlight %}

<h3>Google Maps Flight Paths</h3>

![Terminal Screenshot](/assets/google-maps-path.png)

The last thing I should discuss is how the Google Maps flights paths work. This wasn't a requirement, but I though it'd be a fun thing to include. The <a href="https://developers.google.com/maps/documentation/javascript/examples/geometry-headings">Google Maps API</a> is very well documented so building this really just involved customizing their documentation examples to my needs.

The first thing I tried with the Google Maps API was getting a generic map to render. For Angular, this involved storing a Google Maps object in $scope.map that took in some attributes like where the map should be centered and how far it should be zoomed in by default.

The next task was getting the map markers to show where the origin and destination airports are on the map. This is done by creating a new Google Maps Marker, telling the marker which map it belongs to, and setting latitude and longitude coordinates. I created 2 markers and saved them in "marker1" and "marker2".

Next, we need to create a line or "Polyline" as Google calls them. Google Maps can't do this by default so you must include the "geometry" library when you load the Google Maps API. Once you have the geomety library, you can create a new Google Maps Polyline that has customizable attributes like color, opacity, weight, which map it belongs to, and a boolean geodesic. Google Polylines normally draw a straight path as if the world is flat, however, flight paths are generally displayed by how they actually move around the curvature of the Earth so we need to set geodesic to true to get the nice arc.

Finally, we create the line by using the "setPath" function that belongs to a Google Maps Polyline object. This function takes in 2 pairs of coordinates and then draws the line on our map. 

Check out the full code below which is stored in an Angular directive:

{% highlight javascript %}
// directives/Map
function Map() {
  return {
    templateUrl: 'views/flights/map.html',
    scope: {
      orglat: '=',
      orglong: '=',
      destlat: '=',
      destlong: '='
    },
    controllerAs: 'map',
    controller: function($scope){
      var center = {
        lat: ($scope.orglat + $scope.destlat) / 2,
        long: ($scope.orglong + $scope.destlong) / 2
      }

      $scope.map = new google.maps.Map(document.getElementById('map'), {
          zoom: 4,
          center: new google.maps.LatLng(center.lat, center.long),
          mapTypeId: google.maps.MapTypeId.TERRAIN
      });

      var marker1 = new google.maps.Marker({
        map: $scope.map,
        position: new google.maps.LatLng($scope.orglat, $scope.orglong)
      });

      var marker2 = new google.maps.Marker({
        map: $scope.map,
        position: new google.maps.LatLng($scope.destlat, $scope.destlong)
      });

      var geodesicPoly = new google.maps.Polyline({
        strokeColor: '#CC0099',
        strokeOpacity: 1.0,
        strokeWeight: 3,
        geodesic: true,
        map: $scope.map
      });

      var path = [marker1.getPosition(), marker2.getPosition()];
      geodesicPoly.setPath(path);
    }
  }
}

{% endhighlight %}

<h3>Conclusion</h3>

As I mentioned before, Angular was a very challenging thing to learn, but building this app taught me a ton and boosted my confidence. I still feel like there's so much to learn though. To learn more about Angular, I've bought a couple books to go through and will also come back and improve this app. Some things I would like to add to this app would be user log in, mock flight purchasing, and flights having passengers.

If you’d like to see the full code for this project, see the GitHub repo here: <a href="https://github.com/alexbarron/hiko">https://github.com/alexbarron/hiko</a>