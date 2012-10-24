# Restforce [![travis-ci](https://secure.travis-ci.org/ejholmes/restforce.png)](https://secure.travis-ci.org/ejholmes/restforce) [![Code Climate](https://codeclimate.com/badge.png)](https://codeclimate.com/github/ejholmes/restforce)

Restforce is a ruby gem for the [Salesforce REST api](http://www.salesforce.com/us/developer/docs/api_rest/index.htm).
It's meant to be a lighter weight alternative to the [databasedotcom gem](https://github.com/heroku/databasedotcom).

It attempts to solve a couple of key issues that the databasedotcom gem has been unable to solve:

* Support for interacting with multiple users from different orgs.
* Support for parent-to-child relationships.
* Support for aggregate queries.
* Remove the need to materialize constants.
* Support for the Streaming API
* Support for blob data types.
* Support for GZIP compression.
* A clean and modular architecture using [Faraday middleware](https://github.com/technoweenie/faraday)
* Support for decoding [Force.com Canvas](http://www.salesforce.com/us/developer/docs/platform_connectpre/canvas_framework.pdf) signed requests. (NEW!)

[Documentation](http://rubydoc.info/gems/restforce/frames)

## Installation

Add this line to your application's Gemfile:

    gem 'restforce'

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install restforce

## Usage

Restforce is designed with flexibility and ease of use in mind. By default, all api calls will
return [Hashie::Mash](https://github.com/intridea/hashie/tree/v1.2.0) objects,
so you can do things like `client.query('select Id, (select Name from Children__r) from Account').Children__r.first.Name`.

### Initialization

Which authentication method you use really depends on your use case. If you're
building an application where many users from different orgs are authenticated
through oauth and you need to interact with data in their org on their behalf,
you should use the OAuth token authentication method.

If you're using the gem to interact with a single org (maybe you're building some
salesforce integration internally?) then you should use the username/password
authentication method.

#### OAuth token authentication

```ruby
client = Restforce.new :oauth_token => 'oauth token',
  :instance_url  => 'instance url'
```

Although the above will work, you'll probably want to take advantage of the
(re)authentication middleware by specifying a refresh token, client id and client secret:

```ruby
client = Restforce.new :oauth_token => 'oauth token',
  :refresh_token => 'refresh token',
  :instance_url  => 'instance url',
  :client_id     => 'client_id',
  :client_secret => 'client_secret'
```

#### Username/Password authentication

If you prefer to use a username and password to authenticate:

```ruby
client = Restforce.new :username => 'foo',
  :password       => 'bar',
  :security_token => 'security token'
  :client_id      => 'client_id',
  :client_secret  => 'client_secret'
```

#### Sandbox Orgs

You can connect to sandbox orgs by specifying a host. The default host is
'login.salesforce.com':

```ruby
client = Restforce.new :host => 'test.salesforce.com'
```

#### Global configuration

You can set any of the options passed into Restforce.new globally:

```ruby
Restforce.configure do |config|
  config.client_id     = ENV['SALESFORCE_CLIENT_ID']
  config.client_secret = ENV['SALESFORCE_CLIENT_SECRET']
end
```

* * *

### query(soql)

Performs a soql query and returns the result. The result will be a
[Restforce::Collection][], which can be iterated over.

```ruby
accounts = client.query("select Id, Something__c from Account where Id = 'someid'")
# => #<Restforce::Collection >

account = records.first
# => #<Restforce::SObject >

account.sobject_type
# => 'Account'

account.Id
# => "someid"

account.Name = 'Foobar'
account.save
# => true

account.destroy
# => true
```

_See also: http://www.salesforce.com/us/developer/docs/api_rest/Content/dome_query.htm_

* * *

### search(sosl)

Performs a sosl query and returns the result. The result will be a
[Restforce::Collection][].

```ruby
# Find all occurrences of 'bar'
client.search('FIND {bar}')
# => #<Restforce::Collection >

# Find accounts match the term 'genepoint' and return the Name field
client.search('FIND {genepoint} RETURNING Account (Name)').map(&:Name)
# => ['GenePoint']
```

_See also: http://www.salesforce.com/us/developer/docs/api_rest/Content/dome_search.htm_

* * *

### create(sobject, attrs)

_Alias: insert_

Takes an sobject name and a hash of attributes to create a record. Returns the
Id of the newly created reocrd if the record was successfully created.

```ruby
# Add a new account
client.create('Account', Name: 'Foobar Inc.')
# => '0016000000MRatd'
```

_See also: http://www.salesforce.com/us/developer/docs/api_rest/Content/dome_sobject_create.htm_

* * *

### update(sobject, attrs)

Takes an sobject name and a hash of attributes to update a record. The
'Id' field is required to update. Returns true if the record was successfully
updated.

```ruby
# Update the Account with Id '0016000000MRatd'
client.update('Account', Id: '0016000000MRatd', Name: 'Whizbang Corp')
# => true
```

_See also: http://www.salesforce.com/us/developer/docs/api_rest/Content/dome_update_fields.htm_

* * *

### upsert(sobject, field, attrs)

Takes an sobject name, an external id field, and a hash of attributes and
either inserts or updates the record depending on the existince of the record.
Returns true if the record was updated or the Id of the record if the record was
created.

```ruby
# Update the record with external ID of 12
client.upsert('Account', 'External__c', External__c: 12, Name: 'Foobar')
```

_See also: http://www.salesforce.com/us/developer/docs/api_rest/Content/dome_upsert.htm_

### destroy(sobject, id)

Takes an sobject name and an Id and deletes the record. Returns true if the
record was successfully deleted.

```ruby
# Delete the Account with Id '0016000000MRatd'
client.destroy('Account', '0016000000MRatd')
# => true
```

_See also: http://www.salesforce.com/us/developer/docs/api_rest/Content/dome_delete_record.htm_

* * *

### describe(sobject)

If no parameter is given, it will return the global describe. If the name of an
sobject is given, it will return the describe for that sobject.

```ruby
# get the global describe for all sobjects
client.describe
# => { ... }

# get the describe for the Account object
client.describe('Account')
# => { ... }
```

_See also: http://www.salesforce.com/us/developer/docs/api_rest/Content/dome_describeGlobal.htm, http://www.salesforce.com/us/developer/docs/api_rest/Content/dome_sobject_describe.htm_

* * *

### authenticate!

Performs an authentication and returns the response. In general, calling this
directly shouldn't be required, since the client will handle authentication for
you automatically. This should only be used if you want to force
an authentication before using the streaming api, or you want to get some
information about the user.

```ruby
response = client.authenticate!
# => #<Restforce::Mash access_token="..." id="https://login.salesforce.com/id/00DE0000000cOGcMAM/005E0000001eM4LIAU" instance_url="https://na9.salesforce.com" issued_at="1348465359751" scope="api refresh_token" signature="3fW0pC/TEY2cjK5FCBFOZdjRtCfAuEbK1U74H/eF+Ho=">

# Get the user information
info = client.get(response.id).body
info.user_id
# => '005E0000001eM4LIAU'
```

* * *

### File Uploads

Using the new [Blob Data](http://www.salesforce.com/us/developer/docs/api_rest/Content/dome_sobject_insert_update_blob.htm) api feature (500mb limit):

```ruby
client.create 'Document', FolderId: '00lE0000000FJ6H',
  Description: 'Document test',
  Name: 'My image',
  Body: Restforce::UploadIO.new(File.expand_path('image.jpg', __FILE__), 'image/jpeg'))
```

Using base64 encoded data (37.5mb limit):

```ruby
client.create 'Document', FolderId: '00lE0000000FJ6H',
  Description: 'Document test',
  Name: 'My image',
  Body: Base64::encode64(File.read('image.jpg'))
```

_See also: http://www.salesforce.com/us/developer/docs/api_rest/Content/dome_sobject_insert_update_blob.htm_

* * *

### Streaming

Restforce supports the [Streaming API](http://wiki.developerforce.com/page/Getting_Started_with_the_Force.com_Streaming_API), and makes implementing
pub/sub with Salesforce a trivial task:

```ruby
# Restforce uses faye as the underlying implementation for CometD. I recommend
# using faye 0.8.3.
require 'faye'

# Initialize a client with your username/password/oauth token/etc.
client = Restforce.new

# Force an authentication request.
client.authenticate!

# Create a PushTopic for subscribing to Account changes.
client.create! 'PushTopic', {
  ApiVersion: '23.0',
  Name: 'AllAccounts',
  Description: 'All account records',
  NotifyForOperations: 'All',
  NotifyForFields: 'All',
  Query: "select Id from Account"
}

EM.run {
  # Subscribe to the PushTopic.
  client.subscribe 'AllAccounts' do |message|
    puts message.inspect
  end
}
```

Boom, you're now receiving push notifications when Accounts are
created/updated.

_See also: http://www.salesforce.com/us/developer/docs/api_streaming/index.htm_

* * *

### Caching

The gem supports easy caching of GET requests (e.g. queries):

```ruby
# rails example:
client = Restforce.new cache: Rails.cache

# or
Restforce.configure do |config|
  config.cache = Rails.cache
end
```

If you enable caching, you can disable caching on a per-request basis by using
.without_caching:

```ruby
client.without_caching do
  client.query('select Id from Account')
end
```

* * *

### Logging/Debugging

You can easily inspect what Restforce is sending/receiving by setting
`Restforce.log = true`.

```ruby
Restforce.log = true
client = Restforce.new.query('select Id, Name from Account')
```

Another awesome feature about restforce is that, because it is based on
Faraday, you can insert your own middleware. For example, if you were using
Restforce in a rails app, you can setup custom logging using
ActiveSupport::Notifications:

```ruby
client = Restforce.new
client.middleware.insert_after Restforce::Middleware::InstanceURL, FaradayMiddleware::Instrumentation

# config/initializers/notifications.rb
ActiveSupport::Notifications.subscribe('request.faraday') do |name, start, finish, id, payload|  
  Rails.logger.debug(['notification:', name, "#{(finish - start) * 1000}ms", payload[:status]].join(" "))  
end
```

## Contributing

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Added some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request

[Restforce::Collection]: https://github.com/ejholmes/restforce/blob/master/lib/restforce/collection.rb "Restforce::Collection"
