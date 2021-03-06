# ajax-datatables-rails

[![Build Status](https://travis-ci.org/antillas21/ajax-datatables-rails.svg?branch=master)](https://travis-ci.org/antillas21/ajax-datatables-rails)
[![Gem Version](https://badge.fury.io/rb/ajax-datatables-rails.svg)](http://badge.fury.io/rb/ajax-datatables-rails)

### Versions

[Datatables](http://datatables.net) recently released version 1.10 and deprecated version 1.9 which includes a new API and features.

If you have dataTables 1.9 in your project and want to keep using it, please use this gem's version `0.1.x` in your `Gemfile`:

```ruby
# specific version number
gem 'ajax-datatables-rails', '0.1.2'

# or, support on datatables 1.9
gem 'ajax-datatables-rails', git: 'git://github.com/antillas21/ajax-datatables-rails.git', branch: 'legacy'
```

If you have dataTables 1.10 in your project, then use the gem's latest version, or point to the `master` branch.



## Description

Datatables is a nifty jquery plugin that adds the ability to paginate, sort, and search your html tables. When dealing with large tables (more than a couple hundred rows) however, we run into performance issues. These can be fixed by using server-side pagination, but this breaks some datatables functionality.

`ajax-datatables-rails` is a wrapper around datatable's ajax methods that allow synchronization with server-side pagination in a rails app. It was inspired by this [Railscast](http://railscasts.com/episodes/340-datatables). I needed to implement a similar solution in a couple projects I was working on so I extracted it out into a gem.

## ORM support

Currently `AjaxDatatablesRails` only supports `ActiveRecord` as ORM for performing database queries.

Adding support for `Sequel`, `Mongoid` and `MongoMapper` is a planned feature for this gem. If you'd be interested in contributing to speed development, please [open an issue](https://github.com/antillas21/ajax-datatables-rails/issues/new) and get in touch.

## Installation

Add these lines to your application's Gemfile:

    gem 'jquery-datatables-rails', git: 'git://github.com/rweng/jquery-datatables-rails.git', branch: 'master'
    gem 'ajax-datatables-rails'

And then execute:

    $ bundle

The `jquery-datatables-rails` gem is listed as a convenience, to ease adding
jQuery dataTables to your Rails project. You can always add the plugin assets
manually via the assets pipeline. If you decide to use the `jquery-datatables-rails` gem, please refer to its installation instructions [here](https://github.com/rweng/jquery-datatables-rails).

## Usage
*The following examples assume that we are setting up ajax-datatables-rails for an index of users from a `User` model*

### Generate
Run the following command:

    $ rails generate datatable User


This will generate a file named `user_datatable.rb` in `app/datatables`. Open the file and customize in the functions as directed by the comments.

Take a look [here](#generator-syntax) for an explanation about the generator syntax.

### Customize
```ruby
# uncomment the appropriate paginator module,
# depending on gems available in your project.
# include AjaxDatatablesRails::Extensions::Kaminari
# include AjaxDatatablesRails::Extensions::WillPaginate
# include AjaxDatatablesRails::Extensions::SimplePaginator

def sortable_columns
  # list columns inside the Array in string dot notation.
  # Example: 'users.email'
  @sortable_columns ||= []
end

def searchable_columns
  # list columns inside the Array in string dot notation.
  # Example: 'users.email'
  @searchable_columns ||= []
end
```

* For `paginator options`, just uncomment the paginator you would like to use, given
the gems bundled in your project. For example, if your models are using `Kaminari`, uncomment `AjaxDatatablesRails::Extensions::Kaminari`. You may remove all commented lines.
  * `SimplePaginator` is the most basic of them all, it falls back to passing `offset` and `limit` at the database level (through `ActiveRecord` of course, as that is the only ORM supported for the time being).

* For `sortable_columns`, assign an array of the database columns that correspond to the columns in our view table. For example `[users.f_name, users.l_name, users.bio]`. This array is used for sorting by various columns.

* For `searchable_columns`, assign an array of the database columns that you want searchable by datatables. For example `[users.f_name, users.l_name]`

This gives us:
```ruby
include AjaxDatatablesRails::Extensions::Kaminari

def sortable_columns
  @sortable_columns ||= ['users.f_name', 'users.l_name', 'users.bio']
end

def searchable_columns
  @searchable_columns ||= ['users.f_name', 'users.l_name']
end

```

### Map data
```ruby
def data
  records.map do |record|
    [
        # comma separated list of the values for each cell of a table row
        # example: record.attribute,
      ]
  end
end
```

This method builds a 2d array that is used by datatables to construct the html table. Insert the values you want on each column.

```ruby
def data
  records.map do |record|
    [
      record.f_name,
      record.l_name,
      record.bio
    ]
  end
end
```

[See here](#using-view-helpers) if you need to use view helpers in the returned 2d array, like `link_to`, `mail_to`, `resource_path`, etc.

#### Get Raw Records
```ruby
def get_raw_records
  # insert query here
end
```

This is where your query goes.

```ruby
def get_raw_records
  User.all
end
```

Obviously, you can construct your query as required for the use case the datatable is used. Example: `User.active.with_recent_messages`.
__IMPORTANT:__ Make sure to return an `ActiveRecord::Relation` object as the end product of this method. Why? Because the result from
this method, will be chained (for now) to `ActiveRecord` methods for sorting, filtering and pagination.

#### Associated and nested models
The previous example has only one single model. But what about if you have some associated nested models and in a report you want to show fields from these tables.

Take an example that has an `Event, Course, Coursetype, Allocation, Teacher, Contact, Competency and CompetencyType` models. We want to have a datatables report which has the following column:
```ruby
        'coursetypes.name',
        'courses.name',
        'events.title',
        'events.event_start',
        'events.event_end',
        'contacts.full_name',
        'competency_types.name',
        'events.status'
```
We want to sort and search on all columns of the list. The related definition would be:
```ruby

  def sortable_columns
    @sortable_columns ||= [
        'coursetypes.name',
        'courses.name',
        'events.title',
        'events.event_start',
        'events.event_end',
        'contacts.last_name',
        'competency_types.name',
        'events.status'
    ]
  end

  def searchable_columns
    @searchable_columns ||= [
        'coursetypes.name',
        'courses.name',
        'events.title',
        'events.event_start',
        'events.event_end',
        'contacts.last_name',
        'competency_types.name',
        'events.status'
    ]
  end

  def get_raw_records
     Event.joins(
      { course: :coursetype },
      { allocations: {
          teacher: [:contact, {competencies: :competency_type}]
        }
      }).distinct
  end
```

__Some comments for the above code:__

1. In the list we show `full_name`, but in `sortable_columns` and `searchable_columns` we use `last_name` from the `Contact` model. The reason is we can use only database columns as sort or search fields and the full_name is not a database field.

2. In the `get_raw_records` method we have quite a complex query having one to many and may to many associations using the joins ActiveRecord method. The joins will generate INNER JOIN relations in the SQL query. In this case we do not include all event in the report if we have events which is not associated with any model record from the relation.

3. To have all event records in the list we should use the `.includes` method, which generate LEFT OUTER JOIN relation of the SQL query. __IMPORTANT:__ Make sure to append `.references(:related_model)` with any associated model. That forces the eager loading of all the associated models by one SQL query, and the search condition for any column works fine. Otherwise the `:recordsFiltered => filter_records(get_raw_records).count(:all)` will generate 2 SQL queries (one for the Event model, and then another for the associated tables). The `:recordsFiltered => filter_records(get_raw_records).count(:all)` will use only the first one to return from the ActiveRecord::Relation object in `get_raw_records` and you will get an error message of __Unknown column 'yourtable.yourfield' in 'where clause'__ in case the search field value is not empty.

So the query using the `.includes()` method is:

```ruby
  def get_raw_records
     Event.includes(
      { course: :coursetype },
      { allocations: {
          teacher: [:contact, { competencies: :competency_type }]
        }
      }
      ).references(:course).distinct
  end
```

### Controller
Set up the controller to respond to JSON

```ruby
def index
  respond_to do |format|
    format.html
    format.json { render json: UserDatatable.new(view_context) }
  end
end
```

Don't forget to make sure the proper route has been added to `config/routes.rb`.

### View
* Set up an html `<table>` with a `<thead>` and `<tbody>`
* Add in your table headers if desired
* Don't add any rows to the body of the table, datatables does this automatically
* Add a data attribute to the `<table>` tag with the url of the JSON feed

The resulting view may look like this:

```erb
<table id="users-table", data-source="<%= users_path(format: :json) %>">
  <thead>
    <tr>
      <th>First Name</th>
      <th>Last Name</th>
      <th>Brief Bio</th>
    </tr>
  </thead>
  <tbody>
  </tbody>
</table>
```

### Javascript
Finally, the javascript to tie this all together. In the appropriate `js.coffee` file:

```coffeescript
$ ->
  $('#users-table').dataTable
    processing: true
    serverSide: true
    ajax: $('#users-table').data('source')
    pagingType: 'full_numbers'
    # optional, if you want full pagination controls.
    # Check dataTables documentation to learn more about
    # available options.
```

or, if you're using plain javascript:
```javascript
// users.js

jQuery(document).ready(function() {
  $('#users-table').dataTable({
    "processing": true,
    "serverSide": true,
    "ajax": $('#users-table').data('source')
    "pagingType": "full_numbers",
    // optional, if you want full pagination controls.
    // Check dataTables documentation to learn more about
    // available options.
  });
});
```

### Additional Notes

#### Using view helpers

Sometimes you'll need to use view helper methods like `link_to`, `h`, `mailto`, `edit_resource_path` in the returned JSON representation returned by the `data` method.

To have these methods available to be used, this is the way to go:

```ruby
class MyCustomDatatable < AjaxDatatablesRails::Base
  # either define them one-by-one
  def_delegator :@view, :link_to
  def_delegator :@view, :h
  def_delegator :@view, :mail_to

  # or define them in one pass
  def_delegators :@view, :link_to, :h, :mailto, :edit_resource_path, :other_method

  # now, you'll have these methods available to be used anywhere
  # example: mapping the 2d jsonified array returned.
  def data
    records.map do |record|
      [
        link_to(record.fname, edit_resource_path(record)),
        mail_to(record.email),
        # other attributes
      ]
    end
  end
end
```

#### Options

An `AjaxDatatablesRails::Base` inherited class can accept an options hash at initialization. This provides room for flexibility when required. Example:

```ruby
class UnrespondedMessagesDatatable < AjaxDatatablesRails::Base
  # customized methods here
end

datatable = UnrespondedMessagesDatatable.new(view_context,
  { :foo => { :bar => Baz.new }, :from => 1.month.ago }
)
```
So, now inside your class code, you can use those options like this:


```ruby
# let's see an example
def from
  @from ||= options[:from].beginning_of_day
end

def to
  @to ||= Date.today.end_of_day
end

def get_raw_records
  Message.unresponded.where(received_at: from..to)
end
```

#### Generator Syntax

Also, a class that inherits from `AjaxDatatablesRails::Base` is not tied to an existing model, module, constant or any type of class in your Rails app. You can pass a name to your datatable class like this:


```
$ rails generate datatable users
# returns a users_datatable.rb file with a UsersDatatable class

$ rails generate datatable contact_messages
# returns a contact_messages_datatable.rb file with a ContactMessagesDatatable class

$ rails generate datatable UnrespondedMessages
# returns an unresponded_messages_datatable.rb file with an UnrespondedMessagesDatatable class
```


In the end, it's up to the developer which model(s), scope(s), relationship(s) (or else) to employ inside the datatable class to retrieve records from the database.


## Contributing

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Added some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request
