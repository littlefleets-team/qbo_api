# QboApi

Ruby client for the QuickBooks Online API version 3.
- Built on top of the excellent Faraday gem. 
- JSON only support. 
  - Please don't ask about XML support. [Intuit has stated](https://github.com/ruckus/quickbooks-ruby/issues/257#issuecomment-126834454) that JSON is the primary data format for the QuickBooks API (v3 and beyond). This gem will specialize in JSON only. The [`quickbooks-ruby`](https://github.com/ruckus/quickbooks-ruby) gem has fantastic support for those who favor XML.
- Features specs built directly against a QuickBooks Online Sandbox via the VCR gem.
- Robust error handling.

## Tutorials and Screencasts
- <a href="http://minimul.com/introducing-a-new-ruby-quickbooks-online-client.html" target="_blank">Why qbo_api is a better choice </a> than the <a href="https://github.com/ruckus/quickbooks-ruby" target="_blank">quickbooks-ruby</a> gem.
- <a href="http://minimul.com/getting-started-with-the-modern-ruby-quickbooks-online-client-qbo_api-part-1.html" target="_blank">Part 1</a>: Learn how to spin up the <a href="https://github.com/minimul/qbo_api#spin-up-an-example">example app</a>.
- <a href="http://minimul.com/the-modern-ruby-quickbooks-client-part-2.html" target="_blank">Part 2</a>: <a href="https://github.com/minimul/qbo_api#running-the-specs">Running the specs</a> to aid you in understanding a QuickBooks API transaction. 
- <a href="http://minimul.com/the-modern-ruby-quickbooks-client-contributing.html" target="_blank">Part 3</a>: <a href="https://github.com/minimul/qbo_api#creating-new-specs-or-modifying-existing-spec-that-have-been-recorded-using-the-vcr-gem">Contributing to the gem</a>.

## Ruby >= 2.2.2 required

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'qbo_api'
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install qbo_api

## Usage

### Initialize
```ruby
  q = account.qbo_account # or wherever you are storing the OAuth creds
  qbo_api = QboApi.new(token: q.token,
                       token_secret: q.secret,
                       realm_id: q.companyid,
                       consumer_key: '*****',
                       consumer_secret: '********')
```

### Super fast way to use QboApi no matter your current tech stack as long as Ruby > 2.2.2 is installed
```
- cd ~/<local dir>
- git clone git@github.com:minimul/qbo_api.git && cd qbo_api
- bundle
- bin/console
- QboApi.production = true
- qboapi = QboApi.new(token: "qyprd2uvCOdRq8xzoSSiiiiii",
                      token_secret:"g8wcyQEtwxxxxxxm",
                      realm_id: "12314xxxxxx7",
                      consumer_key: "qyprdwzcxxxxxxbIWsIMIy9PYI",
                      consumer_secret: "CyDN4wpxxxxxxxPMv7hDhmh4")
- qboapi.get :customer, 1
```

### Configuration options
- By default this client runs against a QBO sandbox. To run against the production QBO API URL do:
```ruby
QboApi.production = true
```
- Logging:
```ruby
QboApi.log = true
```
- To change logging target from `$stdout` e.g.
```ruby
QboApi.logger = Rails.logger
```
- To run all requests with unique [RequestIds](https://developer.intuit.com/hub/blog/2015/04/06/15346).
```ruby
QboApi.request_id = true
```
- To run individual requests with a RequestId then do something like this:
```ruby
  resp = qbo_api.create(:bill, payload: bill_hash, params: { requestid: qbo_api.uuid })
  # Works with .get, .create, .update, .query methods
```
- To run all requests with a [Minor version](https://developer.intuit.com/docs/0100_quickbooks_online/0200_dev_guides/accounting/minor_versions).
```ruby
QboApi.minor_version = 8
```
- To run individual requests with a minor version then do something like this:
```ruby
  resp = qbo_api.get(:item, 8, params: { minorversion: 8 })
  # Works with .get, .create, .update, .query methods
```


### Create
```ruby
  invoice = {
    "Line": [
      {
        "Amount": 100.00,
        "DetailType": "SalesItemLineDetail",
        "SalesItemLineDetail": {
          "ItemRef": {
            "value": "1",
            "name": "Services"
          }
        }
      }
    ],
    "CustomerRef": {
      "value": "1"
    }
  }
  response = qbo_api.create(:invoice, payload: invoice)
  p response['Id'] # => 65
```

### Update
```ruby
  customer = { 
    DisplayName: 'Jack Doe',
    PrimaryPhone: {
      FreeFormNumber: "(415) 444-1234"
    }
  }
  response = qbo_api.update(:customer, id: 60, payload: customer)
  p response.fetch('PrimaryPhone').fetch('FreeFormNumber') # => "(415) 444-1234"
```

### Delete (only works for transaction entities)
```ruby
  response = qbo_api.delete(:invoice, id: 145)
  p response['status'] # => "Deleted"
```

### Deactivate (only works for name list entities)
```ruby
  response = qbo_api.deactivate(:employee, id: 55)
  p response['Active'] # => false
```

### Search with irregular characters
```ruby
  name = qbo_api.esc "Amy's Bird Sanctuary"
  response = qbo_api.query(%{SELECT * FROM Customer WHERE DisplayName = '#{name}'})
```
### Get an entity by its id
```ruby
  response = qbo_api.get(:customer, 5)
  p response['DisplayName'] # => "Dukes Basketball Camp"
```

### Change data capture (CDC) query
```ruby
  response = api.cdc(entities: 'estimate', changed_since: '2011-10-10T09:00:00-07:00')
  # You can also send in a Time object e.g. changed_since: Time.now 
  expect(response['CDCResponse'].size).to eq 1
  ids = response['CDCResponse'][0]['QueryResponse'][0]['Estimate'].collect{ |e| e['Id'] }
  p ids
```

### Batch operations (limit 30 operations in 1 batch request)
```ruby
  payload = {
      "BatchItemRequest":
      [
        {
          "bId": "bid1",
          "operation": "create",
          "Vendor": {
            "DisplayName": "Smith Family Store"
          }
        }, {
          "bId": "bid2",
          "operation": "delete",
          "Invoice": {
            "Id": "129",
            "SyncToken": "0"
          }
        }
      ]
  }
  response = api.batch(payload)
  expect(response['BatchItemResponse'].size).to eq 2
  expect(batch_response.detect{ |b| b["bId"] == "bid1" }["Vendor"]["DisplayName"]).to eq "Smith Family Store"
```

### Reports

```ruby
        params = { start_date: '2015-01-01', end_date: '2015-07-31', customer: 1, summarize_column_by: 'Customers' }
        response = api.reports(name: 'ProfitAndLoss', params: params)
        p response["Header"]["ReportName"]) #=> 'ProfitAndLoss'
```

### Reconnect

See [docs](https://developer.intuit.com/docs/0100_quickbooks_online/0100_essentials/0085_develop_quickbooks_apps/0004_authentication_and_authorization/oauth_management_api#/Reconnect)
```ruby
        response = qbo_api.reconnect
        #=> if response['ErrorCode'] == 0
        #=>   p response['OAuthToken'] #=> rewq23423424afadsdfs==
        #=>   p response['OAuthTokenSecret'] #=> ertwwetu12345312005343453yy=Fg
```

### Disconnect

See [docs](https://developer.intuit.com/docs/0100_quickbooks_online/0100_essentials/0085_develop_quickbooks_apps/0004_authentication_and_authorization/oauth_management_api#/Disconnect)
```ruby
        response = qbo_api.disconnect
        #=> if response['ErrorCode'] == 0
        #=>   # Successful disconnect
```


### Respond to an error
```ruby
  customer = { DisplayName: 'Weiskopf Consulting' } 
  begin
    response = qbo_api.create(:customer, payload: customer)
  rescue QboApi::BadRequest => e
    if e.message =~ /Another customer already exists with this name/
      # Query for Id using DisplayName
      # Do an qbo_api.update instead
    end
  end
```

### Import/retrieve all
```ruby
  # retrieves all active customers
  qbo_api.all(:customers) do |c|
    p "#{c['Id']} #{c['DisplayName']}"
  end

  # retrieves all active or inactive employees
  qbo_api.all(:employees, inactive: true) do |e|
    p "#{e['Id']} #{e['DisplayName']}"
  end

  # retrieves all vendors by groups of 5
  qbo_api.all(:vendor, max: 5) do |v|
    p v['DisplayName']
  end
  
  # retrieves all customers by groups of 2 using a custom select query
  where = "WHERE Id IN ('5', '6', '7', '8', '9', '10')"
  qbo_api.all(:customer, max: 2, select: "SELECT * FROM Customer #{where}") do |c|
    p c['DisplayName']
  end
```

### What kind of QuickBooks entity?
```ruby
  p qbo_api.is_transaction_entity?(:invoice) # => true
  # Plural is supported as well
  p qbo_api.is_transaction_entity?(:invoices) # => true
  p qbo_api.is_transaction_entity?(:customer) # => false
  p qbo_api.is_name_list_entity?(:vendors) # => true
```

## Spin up an example
- <a href="http://minimul.com/getting-started-with-the-modern-ruby-quickbooks-online-client-qbo_api-part-1.html" target="_blank">Check out this tutorial and screencast on spinning up an example</a>.
- `git clone git://github.com/minimul/qbo_api && cd qbo_api`
- `bundle`
- Create a `.env` file
  - If needed create an account at [https://developer.intuit.com](https://developer.intuit.com)
  - Click `Get started coding`
  - Create an app with both the `Accounting` & `Payments` selected.
  - Go to the `Development` tab and copy and paste the consumer key and secret into the `.env` file.
```ruby
export QBO_API_CONSUMER_KEY=<Your QuickBooks apps consumer key>
export QBO_API_CONSUMER_SECRET=<Your QuickBooks apps consumer secret>
```
- Note: the `.env` file will be automatically loaded after you run the next step.
- Start up the example app
  - `ruby example/app.rb`
- Goto `http://localhost:9393`
- Use the `Connect to QuickBooks` button to connect to your QuickBooks sandbox, which you receive when signing up at [https://developer.intuit.com](https://developer.intuit.com).
- After successfully connecting to your sandbox run:
  - `http://localhost:9393/customer/5`
  - You should see "Dukes Basketball Camp" displayed
- Checkout [`example/app.rb`](https://github.com/minimul/qbo_api/blob/master/example/app.rb) to see what is going on under the hood.

## Webhooks
- <a href="http://minimul.com/getting-started-with-quickbooks-online-webhooks.html" target="_blank">Check out this tutorial and screencast on handling a webhook request</a>. Also checkout [`example/app.rb`](https://github.com/minimul/qbo_api/blob/master/example/app.rb) for the request handling code.

## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/minimul/qbo_api.

#### Running the specs
- <a href="http://minimul.com/the-modern-ruby-quickbooks-client-part-2.html" target="_blank">Check out this tutorial and screencast on running the specs</a>
- `git clone git://github.com/minimul/qbo_api && cd qbo_api`
- `bundle`
- Create a `.env` file
  - If you are just running the specs then at minimum your `.env` needs to look like the following:
```ruby
export QBO_API_CONSUMER_KEY=
export QBO_API_CONSUMER_SECRET=
export QBO_API_ACCESS_TOKEN=
export QBO_API_ACCESS_TOKEN_SECRET=
export QBO_API_COMPANY_ID=12345
```
- `bundle exec rspec spec/`
  
#### Creating new specs or modifying existing spec that have been recorded using the VCR gem.
- All specs that require interaction with the API must be recorded against your personal QuickBooks sandbox. More coming on how to create or modifying existing specs against your sandbox.
- <a href="http://minimul.com/the-modern-ruby-quickbooks-client-contributing.html" target="_blank">Check out this tutorial and screencast on contributing to qbo_api</a>.

#### Protip: Once your .env file is completely filled out you can use the console to play around in your sandbox
```
bin/console test
>> q = QboApi.new(creds.to_h)
>> q.get :customer, 1
```

## License

The gem is available as open source under the terms of the [MIT License](http://opensource.org/licenses/MIT).

