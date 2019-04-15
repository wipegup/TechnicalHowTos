# Setting Up a Simple D3 Graphic in Rails

This tutorial goes through the most basic steps of building a very simple [`d3`](https://d3js.org/) graphic.  

[All code may be found here](https://github.com/wipegup/SimpleD3). All commits are directly related to section headers below

The tutorial was built in Rails 5.1, and uses [`d3-rails`](https://github.com/iblue/d3-rails) and [`jquery`](https://jquery.com/).

[This more extensive](http://gregpark.io/blog/live-d3-rails-plot/) demonstration was modernized and simplified.  


## The Setup
In Rails 5.1, `jquery` is no longer a dependency, thus a gem must be installed, and it must be added to the `require`s in the `appliation.js` file.  

Similarly, a gem for `d3-rails` must be installed, and the attendant `d3` library must be added to the `require`s in the `application.js` file.  

```ruby
# Gemfile
# add:

gem 'd3-rails'
gem 'jquery-rails'
```
Run `bundle install`  


```javascript
// app/assets/javascripts/application.js
// add:

//= require d3
//= require jquery
//= require jquery_ujs
```

## The Database
Our visualization will like with our database, specific through a `clicks` table. `clicks` will not be built out in any meaningful way. Each row will simply be an `id` with `timestamps`.  

Run: `rails g migration CreateClicks`  

Edit resulting file to:

```ruby
# db/migrate <date/timestring>_create_clicks.rb

class CreateClicks < ActiveRecord::Migration[5.1]
  def change
    create_table :clicks do |t|
      t.timestamps
    end
  end
end
```

Create an associated model, adding a `total` class method:  

```ruby
# models/click.rb

class Click < ApplicationRecord
  def self.total
    [count]
  end
end

```  

Note above how `::total` returns an `Array` with a single element. This will be explained in the [Javascript](#javascript) section  

Run: `rake db:{drop,create,migrate}`

## The Controller

For our clicks we will specify two controller actions; `index`, and `create`. `index` will serve our page *and its data*. `create` will create a new click object.  

First, add to the `routes.rb` file:

```ruby
# config/routes.rb
# add:

resources :clicks, only: [:index, :create]
```
This yields a single `path helper` of `clicks_path`, where a `GET` routes to `index` and `POST` routes to `create`.  


In the Controller:  
```ruby
# app/controllers/clicks_controller.rb

class ClicksController < ApplicationController

  def index
    @clicks = Click.total
    respond_to do |format|
      format.html
      format.json { render json: @clicks }
    end
  end

  def create
    Click.create
    @clicks = Click.total
    respond_to do |format|
      format.html {redirect_to clicks_path}
      format.js
    end
  end

end
```


Note that above, both actions contain a [`respond_to`](https://apidock.com/rails/ActionController/MimeResponds/InstanceMethods/respond_to) block.  

The `respond_to` block defines *how the server should respond when a particular format is requested*.  

In the `index` action:
- In a conventional `GET` request routed to the index action, the `respond_to` block specifies that an `html` page should be served. Specifically, `Rails` will look for an `index.html[.erb]` file in `app/views/clicks`.
- In a `GET` request routed to the index action specifying `format: :json`, a different action; `{render json: @clicks}` is served. Thus, `Rails` would use its built-in `json` conversion on the `@clicks` object, and return that `json`. If that other block were not present, it would attempt to serve `index.json`  

In the `create` action, a two formats are specified, `.js` and `.html`.

Unlike with the formats in `index`, at no time will we [explicitly] specify, to serve `javascript`. Instead, this block will be triggered by using the `remote: true` option in a [`link_to`](https://apidock.com/rails/ActionView/Helpers/UrlHelper/link_to) helper.  

However, though the call is different, the resulting action is still the same, a file with the name of the action (`create` in this case) with the particular extension `.js` will be run.  

When executing the `create` action with a normal `html` `POST` request, the `.html` block will be executes; redirecting to the index.  

## The View

```html
<!-- app/views/clicks/index.html.erb -->

<p>Click Count:
  <span id="click-count"><%= @clicks[0] %></span>
</p>

<%= link_to 'Click Me, JS Refresh', clicks_path, remote: true, method: :post  %>
<br>
<%= link_to 'Click Me, Redirect Refresh', clicks_path, remote: false, method: :post  %>
<br>
<svg id="circle"></svg>

```

The above view consists of three parts:
- Header: Displays total number of clicks - Note how the zero-th element of `@clicks` is specified, as `::total` returns an array.
- Refresh Links: Same methods, and paths, `JS Refresh` has `remote: true`, `Redirect Refresh` has `remote: false` (the default for `link_to`)
- SVG tag: Currently empty, with id of `circle`.

## The Links  

Clicking on the `Click Me, JS Refresh` yields a `POST` request to `/clicks`, which enters the `create` action in the `clicks_controller`. Because `remote: true`, the format of the request is for `.js`.  

Thus the first two lines are run; a new `click` is created with `Click.create`, and `@clicks = Click.total`.  
Within the `respond_to` block, `format.js` is hit. At which point `Rails` searches for a `create.js` file

```javascript
// app/views/clicks/create.js.erb

$("#click-count").html("<%= @clicks[0] %>");
getData(redrawCircle);
```

This `create.js.erb` file performs two actions.  
- `$("#click-count").html("<%= @clicks[0] %>");`: Using `jquery`, find the element with the id of `click-count`. Replace the content of that element with the html[erb] of `"<%= @clicks[0] %>"`. Remember `@clicks` was already updated in the controller.
- Run the user defined function `getData()`. This will be covered more in the [Javascript](#javascript) section, but is used to refresh the graphic.  

Thus the page should be refreshed with the new


Clicking on the `Click Me, Redirect Refresh` yields a `POST` request to `/clicks`, which enters the `create` action in the `clicks_controller`. Because `remote: false`, the format of the request is for `.html`.

Again, the click is created and `@clicks` is updated. Within the `respond_to` block, a redirect to the index is hit. Thus the **entire** page is refreshed, not just the graphic and click count.

## Javascript
While some `javascript` was executed in `create.js.erb`, the majority is served from a file in the assets folder which contains multiple user-defined functions and instructions.  

First the function that loads our data
```javascript
// app/assets/javascripts/clicks.js

// ajax (using jquery) to load data
function getData(drawFunction){
               $.ajax({
                 type: 'GET',
                 contentType: 'application/json; charset=utf-8',
                 url: '/clicks',
                 dataType: 'json',
                 success: function(data){
                   drawFunction(data);
                 },
                 failure: function(result){
                   error();
                 }
               });
             };

// Error function for ajax failure
function error() {
   console.log("Error in Loading Data!");
}
```

`getData` is a function which takes a single parameter -- another function -- which is used to build the visualization of the data it retrieves.  

The `ajax` request (accomplished with `jquery` syntax), defines the type of request (`GET`), url (`/clicks`), the data type we expect in return `dataType: 'json'`, what to do in case of success, and what to do in case of failure. [Explination of content v. dataType](https://stackoverflow.com/questions/18701282/what-is-content-type-and-datatype-in-an-ajax-request)  

The data we receive through this request is exceedingly simple (an array of a single integer), this same request may be used to collect more complicated json.  

If this ajax call succeeds, the `data` passed to `drawFunction` would be `[50]` if there were 50 clicks.  


Below are the two functions that might be passed in as the `drawFunction`:  

```javascript
// app/assets/javascript/clicks.js

function drawCircle(data){
  d3.select('#circle')    // select element with id = 'circle'
  .append('circle')     // create a circle element at that selector
  .attr("cx", 50)       // define the x-location of the circle center
  .attr("cy", 50)       // define the y-location of the circle center
  .attr("r", data[0]);  // define the radius of the circle
}

function redrawCircle(data){
  d3.select('#circle')    // select element with id = 'circle'
    .select("circle")     // select the 'circle' element
    .transition()         // specify a transition should occur
    .attr("r", data[0]);  // specify which attributes should change
}
```

The `drawCircle` function finds our `svg` tag via its id (`#circle`), creates a new [svg circle](https://www.w3schools.com/graphics/svg_circle.asp), defining values for the `cx`, `cy`, and `r` `attr`ibutes.  

The `r`adius attribute is assigned the value of the first (and only) element in our data array. Note, this starts as a 0, so do not expect to see a circle appear until having clicked a number of times.  


The `redrawCircle` Finds the circle that is already on the page, readies itself for a [`transition()`](https://www.tutorialspoint.com/d3js/d3js_transition.htm), and defines which `attr`ibutes should be changed -- the `r`adius to the new value of the data.  


In `create.js.erb`, we saw where `redrawCircle` is passed/called to `getData`. However, the `drawCircle` function is passed/called within the `clicks.js` file, again using the  `jquery` [`.ready()`](https://www.w3schools.com/jquery/event_ready.asp) function, which will simply call `getData(drawCircle)` as soon as the page is loaded.

```javascript
// app/assets/javascript/clicks.js

// load data upon page load
//$(document).ready(function(){getData(drawCircle);}); is equivalent
$(function(){
  getData(drawCircle);
});
```

## Binding Data

For more complicated graphics, `d3`'s `.data()` function is useful. [Learn more here](https://medium.freecodecamp.org/learn-d3-js-in-5-minutes-c5ec29fb0725#53e2)

An extremely simple example may be found in the `binding_data` branch of the associated repo. The two things that are demonstrated are: passing data in with `.data()`, and accessing data via a function.
