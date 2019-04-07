# Using the "Errors" Feature of the Rail's Application Record

The below code snippets should give context sufficient for starting to use `.errors` in Rails to provide substantive error messages when validating form information.

Used [this article for reference on custom validations](https://blog.bigbinary.com/2016/05/03/rails-5-adds-a-way-to-get-information-about-types-of-failed-validations.html)

## The Simple Model

```
class Item < ApplicationRecord
  validates_presence_of :name, :price
end
```

## The Controller

```
class ItemsController < ApplicationController
  def new
    @item = Item.new
  end

  def create
    @item = Item.new(item_info)

    if @item.valid?
      @item.save
      redirect_to items_path
    else
      render :new
    end
  end

  private

  def item_info
    params.require(:item).permit(:name,:price)
  end
end
```  

## The Form
```
# ./app/views/items/new.html.erb
# In routes.rb:
#   resources :items, only: [:new, :create]
<% if @item.errors.any? %>
<ul>
  <% @item.errors.full_messages.each do |msg| %>
  <li><%= msg %></li>
  <% end %>
</ul>
<% end %>

<%= form_for @item do |f| %>
  <%= f.label :name %>
  <%= f.text_field :name %>
  <br>
  <%= f.label :price %>
  <%= f.number_field :price %>

  <%= f.submit %>
<% end %>

```

## The Test
```
RSpec.describe 'Repopulate new form and display errors' do
  visit new_item_path

  fill_in "Name", with: "Item 1"

  click_button "Create Item"

  expect(page).to have_field("Name", with:"Item 1")
  expect(page).to have_content("Price can't be blank")

end

```
After clicking the `"Create Item"` button on the new form, the `create` method is entered into in the `ItemController`.  

In the `create` method, after running:  
```
@item = Item.new(item_info)
```

The `@item` object has a number of useful properties:  

- `@item.valid?` will evaluate to `false`
- `@item.errors.any?` will evaluate to `true`
- `@item.errors.details` has the value of `{:price => [{:error =>:blank}]}`
- `@item.errors.messages` has the value of `{:price => ["can't be blank]`
- `@item.errors.full_messages` has the value of `['Price can't be blank']`

Additionally, **because it was assigned to the instance variable `@item`** when `render :new` is called, it will re-populate the new form with all of the correct information (in this case only the name) *and* will have access to all of the `errors` attributes described above

# Custom Validations

As should be clear by now, these errors depend upon the validations within the Model.  

While many validations are available, custom validations may also be created:   

```
class Item < ApplicationRecord

# ...More Code...

  validate :name_doesnt_contain_Bob

  def name_doesnt_contain_Bob
    if name.index("Bob") != nil
      errors.add(:name, "may not contain the string 'Bob'")
    end
  end
end
```
