# Trucker
Use Trucker to migrate legacy data into your Rails app.

## Installation

1. Add the trucker gem in to your Gemfile
  ```ruby
  gem 'trucker'
  ```

2. Bundle 
  ```bash
  bundle install
  ```

3. Generate the basic trucker files
  ```bash
  rails g truck
  ```

  This will do the following things:
  - Add legacy adapter to `database.yml`
  - Add `app/models/legacy` directory
  - Add `app/models/legacy` to `autoload_paths` in Rails Initializer config block
  - Add `app/models/legacy/legacy_base.rb` (from which legacy models will inherit)
  - Add legacy sub classes for all existing models
  - Generate sample migration task (using pluralized model names)
  
4. Update the legacy database adapter in `database.yml` with your legacy database info

  ```yaml
  legacy:
    adapter: mysql
    encoding: utf8
    database: app_legacy
    username: root
    password:
  ```

  By convention, we recommend naming your legacy database `APP_legacy`, just as your
  other databases might be named `APP_development`, `APP_production`, etc.

5. If the legacy database doesn't already exist, add it.
  ```bash
  rake db:create:all
  ```

6. Import your legacy data into the legacy database.
  ```bash
  mysql -u root app_legacy < old_database.sql
  ```
  
  If you're not using mysql, you should change this command as needed.
  
7. Custom your table name for each of your legacy models.
  ```ruby
  class LegacyPost < LegacyBase
    self.table_name =  "LEGACY_TABLE_NAME_GOES_HERE"
  end
  ```
  
  Since you're migrating data from an old database, your table names may not 
  follow Rails conventions for database table naming. If so, you will need to 
  set the `self.table_name = ` value for each of your legacy models to match the 
  name of table from which you will be importing data.
  
  For instance, in the example above, if your old posts were stored in an 
  `articles` table, you would customize `self.table_name = ` like so:
  
  ```ruby
  class LegacyPost < LegacyBase
    self.table_name =  "articles"
  end
  ```
  
8. Update legacy model field mappings.

  ```ruby
  class LegacyPost < LegacyBase
    self.table_name =  "LEGACY_TABLE_NAME_GOES_HERE"

    def map
      {
        :headline => self.title.squish,
        :body => self.long_text.squish
      }
    end
  end
  ```

  This is where you will connect your old database attributes with your new ones. 
  The map method is really just a hash which uses your new model attribute names
  as keys and your legacy model attributes as values.
  
  (aka `:new_field => self.legacy_field`)
  
  Note: make sure to add `self.` to each legacy attribute name.
    
9. Need to tweak some data? Just add some core ruby methods or add a helper method.

  ```ruby
  class LegacyPost < LegacyBase
    self.table_name =  "LEGACY_TABLE_NAME_GOES_HERE"

    def map
      {
        :headline => self.title.squish.capitalize, # <= Added capitalize method
        :body => tweak_body(self.long_text.squish) # <= Added tweak_body method
      }
    end
  
    # Insert helper methods as needed
    def tweak_body(body)
      body = body.gsub(/<br \//,"\n") # <= Convert break tags into normal line breaks
      body = body.gsub(/teh/, "the")  # <= Fix common typos
    end
  end
  ```
  
10. Start migrating!
  ```bash
  rake db:migrate:posts
  ```

## Migration command line options
Trucker supports a few command line options when migrating records:

  ```bash
  rake db:migrate:posts limit=100 (migrates 100 records)
  rake db:migrate:posts limit=100 offset=100 (migrates 100 records, but skip the first 100 records)
  ```

## Custom migration labels
You can tweak the default migration output generated by Trucker by using the `:label` option.

  ```bash
  rake db:migrate:posts
  => Migrating posts

  rake db:migrate:posts, :label => "blog posts"
  => Migrating blog posts
  ```

## Custom helpers
Trucker works great for migrating data from many legacy data sources such as apps built with PHP, Perl, Python, or even older versions of Rails (where upgrading an existing Rails code base is not practical). But, if you're migrating data from a large enterprise system, Trucker may not be your best choice.

That said, if you need to pull off a complex migration for a model, you can use a custom helper method to override Trucker's default migrate method in your rake task.

```ruby
namespace :db do
  namespace :migrate do
    ...
    desc 'Migrate pain_in_the_ass model'
    task :pain_in_the_ass => :environment do
      Trucker.migrate :pain_in_the_ass, :helper => pain_in_the_ass_migration
    end 
  end
end

def pain_in_the_ass_migration
  # Custom code goes here
end
```

If you don't want to write your custom migration method from scratch, you can copy trucker's migrate method method from [lib/trucker.rb](https://github.com/mokolabs/trucker/blob/master/lib/trucker.rb) and tweak accordingly.

As an example, here's a custom helper used to migrate join tables on a bunch of models.

```ruby
namespace :db do
  namespace :migrate do

    desc 'Migrates join tables'
    task :joins => :environment do
      migrate :joins, :helper => :migrate_joins  
    end

  end
end

def migrate_joins
  puts "Migrating #{number_of_records || "all"} joins #{"after #{offset_for_records}" if offset_for_records}"

  ["chain", "firm", "function", "style", "website"].each do |model|

    # Start migration
    puts "Migrating theaters_#{model.pluralize}"

    # Delete existing joins
    ActiveRecord::Base.connection.execute("TRUNCATE table theaters_#{model.pluralize}")

    # Tweak model ids and foreign keys to match model syntax
    if model == 'website'
      model_id = "url_id"
      send_foreign_key = "url_id".to_sym
    else
      model_id = "#{model}_id"
      send_foreign_key = "#{model}_id".to_sym
    end

    # Create join object class
    join = Object.const_set("Theaters#{model.classify}", Class.new(ActiveRecord::Base))

    # Set model foreign key
    model_foreign_key = "#{model}_id".to_sym

    # Migrate join (unless duplicate)
    "LegacyTheater#{model.classify}".constantize.find(:all, with(:order => model_id)).each do |record|
  
      unless join.find(:first, :conditions => {:theater_id => record.theater_id, model_foreign_key => record.send(send_foreign_key)})
        attributes = {
          model_foreign_key => record.send(send_foreign_key),
          :theater_id => record.theater_id
        }
    
        # Check if theater chain is current
        attributes[:is_current] = {'Yes' => 1, 'No' => 0, '' => 0}[record.current] if model == 'chain'
    
        # Migrate join
        join.create(attributes)
      end
    end
  end
end
```

## Sample application
Check out the [Trucker sample app](http://github.com/mokolabs/trucker_sample_app) for a working example of Trucker-based legacy data migration.


## Background
Trucker is based on a migration technique using legacy models first pioneered by Dave Thomas:
http://pragdave.blogs.pragprog.com/pragdave/2006/01/sharing_externa.html


## Note on patches/pull requests
- Fork the project.
- Make your feature addition or bug fix.
- Add tests for it. This is important so we don't break a future version unintentionally.
- Commit your changes, but do not mess with the rakefile, version, or history.
  (if you want to have your own version, that is fine but bump version in a commit by itself so we can ignore when we pull)
- Send a pull request. Bonus points for topic branches.


## Contributors
- [Patrick Crowley](https://github.com/mokolabs/)
- [Rob Kaufman](https://github.com/notch8/)
- [Jordan Fowler](https://github.com/thebreeze/)
- [Roel Bondoc](https://github.com/roelbondoc/)
- [Olivier Lacan](https://github.com/olivierlacan/)


## Copyright
Copyright (c) 2014 Patrick Crowley and Rob Kaufman. See [LICENSE](LICENSE) for details.