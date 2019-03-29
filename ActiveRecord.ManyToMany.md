# Many-to-Many Active Record Basics in Rails

For the purposes of these Examples I will assume that we have two `ApplicationRecord` classes created; `Widget` and `Maker`.  

Each `Maker` produces multiple widgets, and each `Widget` may be produced by multiple makers.

## Migrations

In order to link these two classes in a Database, we will use three migrations;  
- `CreateMakers`
- `CreateWidgets`
- `CreateMakerWidgets`  

**Simple Migrations**
Each Migration may be initialized by running:  
```
rails g[enerate] migration Create<migration_name>
```
e.g. To create our `Maker` migration we would run:  
```
rails g migration CreateMakers
```
or
```
rails generate migration CreateMakers
```

The migrations for `Maker` and `Widget` are fairly straight forward. Each widget might have a `string` "name" and an `integer` price. Maybe each `Maker` has a `string` name. Thus, for example the `Widget` migration should appear similar to:  

```
class CreateWidgets < ActiveRecord::Migration[5.1]
  def change
    create_table :widget do |t|
      t.string :name
      t.integer :price

      t.timestamps
    end
  end
end
```
In a file named: `<date+time>_create_widgets.rb` In the `db/migrate` folder.  

**Linking Migration**

More importantly let's discuss the creation of the `CreateMakerWidgets` migration.  

The *Maker - Widget* migration is what will eventually enable to Many-to-Many relationships between Maker and Widget described above.  

The `rails g migration Create<name> ...` offers some useful shortcuts for defining relationships.  

In one-line, the *Maker - Widget* migration may be created via:  
```
rails g migration CreateMakerWidgets maker:references widgets:references  
```
Yielding a file named: `<date+time>_create_maker_widgets.rb`.  

The `rails generate migration` shortcuts need not be utilized. Ultimately the migration file should appear similar to:  

```
class CreateMakerWidgets < ActiveRecord::Migration[5.1]
  def change
    create_table :maker_widgets do |t|
      t.references :maker, foreign_key: true
      t.references :widget, foreign_key: true
    end
  end
end
```

**Running Migrations**

With those three migration files created, running `rake db:create` and `rake db:migrate` should yield a database with a many-to-many relationship between `Maker` and `Widget`.  


## ApplicationRecord Models  

Next, within the `app/models` folder, three files, must be generated:  
- `widget.rb`
- `maker.rb`
- `maker_widget.rb`  

While you may specify what fields the model `validates_presence_of` below will discuss how to LINK the two models.  

There are two strategies which might be employed to define this *Many-to-Many* relationship. Using [`has_and_belongs_to_many`](https://guides.rubyonrails.org/association_basics.html#the-has-and-belongs-to-many-association) is possible, however, below will describe using [`has_many :through`](https://guides.rubyonrails.org/association_basics.html#the-has-many-through-association). [A discussion of the two may be found in the rails docs](https://guides.rubyonrails.org/association_basics.html#choosing-between-has-many-through-and-has-and-belongs-to-many)  

First, the `maker_widget.rb` file.  

The `MakerWidget` model is used to create a link between the `Maker`s and the `Widget`s. Thus, each record in `MakerWidget` is linked to (as was defined in the migration) one `Widget` and one `Maker`. Thus the `maker_widget.rb` should look like:  

```
class MakerWidget < ApplicationRecord
  belongs_to :maker
  belongs_to :widget
end
```  

That part is simple enough, and, as a bonus, having defined that link we can start to define our links between `Maker` and `Widget` with `through:`.  

Moving onto the `widget.rb` file we should have the following:  

```
class Widget < ApplicationRecord
  has_many :maker_widgets
  has_many :makers, through: :maker_widgets
end
```

As described above, the  `Widget` may be associated with multiple `MakerWidget`s. Then, by virtue of those multiple `MakerWidget`s, the `Widget` may be associated with multiple `Maker`s.  

The `maker.rb` will look very similar.
```
class Maker < ApplicationRecord
  has_many :maker_widgets
  has_many :widgets, through: :maker_widgets
end
```

Of note, the order of these two `has_many` statements matters. Were the `through:` line be placed above the `:maker_widgets` line, the `through:` line would not yet "know" about the presence (or future presence) of `maker_widgets`.  

## Creating Objects with a Many-to-Many Relationship  

The final piece of the puzzle involves actually creating `Widgets` with multiple `Makers` and vice-a-versa.  

First we can create some `Widget`s:

```
w1 = Widget.create(name: "w1", price: 1)
w2 = Widget.create(name: "w2", price: 2)
w3 = Widget.create(name: "w3", price: 3)
```
Hopefully that looks familiar.  

Next we will create a `Maker` with a single widget; via a couple of methods:  

```
# Way 1
m1 = Maker.create(name: "m1", widgets: [w1])  

# Way 2
m1 = Maker.create(name: "m1")
m1.widget_ids = [w1.id]

# Way 3
m1 = Maker.create(name: "m1")
MakerWidget.create(widget:w1, maker:m1)

# Way 4
m1 = w1.makers.create(name:"m1")
```

Finally, we can create a Maker with a couple of widgets.

```
# Way 1
m2 = Maker.create(name: "m2", widgets: [w2, w3])  

# Way 2
m2 = Maker.create(name: "m2")
m2.widget_ids = [w2.id, w3.id]

# Way 3
m2 = Maker.create(name: "m2")
MakerWidget.create(widget:w2, maker:m2)
MakerWidget.create(widget:w3, maker:m2)

# Way 4
m2 = w2.makers.create(name:"m2")
m2 << w3
```
