---
layout: post
title: Using the VCR gem to mock external requests in Rails
tags: [blog]
---
The problem

It is common for a Rails application to make HTTP requests — for instance to an external API. It is important that your test suite accounts for this request, the way it is made and how the response is handled. However, for a variety of reasons (speed, rate limits, security), it is often desirable that your tests don’t really go and make this HTTP request every time.

There are a number of approaches to this, most of which have an intrinsic level of artifice (i.e. you’re somehow mocking or faking the request). The approach I’m going to outline below uses the [VCR gem](https://github.com/vcr/vcr). I haven’t dwelled on some of the initial setup for VCR, which is covered in their [repo](https://github.com/vcr/vcr).

The VCR gem allows you to make a real, properly structured request. Both this and the ensuing response are then ‘recorded’ and stored in your test ecosystem. From that point on your tests shouldn’t make a real HTTP request, however the structure will be akin to as if it had done so. The fragility here (somewhat mitigated by VCR’s re-recording facility) is that the external service may change. Whilst remaining mindful of this disconnect, it’s still a very reasonable sacrifice compared to making real HTTP requests.

## Testing the Request

Below is a simpleExternalServiceFetcher class. It takes a number as an argument, builds a URL to the external service and then makes a HTTP request.

```ruby
class ExternalServiceFetcher
  def initialize(request_id)
    @request_id = request_id
  end

  def make_request
    begin
      HTTParty.get(build_url)
    rescue => e
      Rails.logger.error { "Encountered an error when making request {url}. #{e.message} #{e.backtrace.join("\n")}" }
    end
  end

  def build_url
    ENV['EXTERNAL_REQUEST_ENDPOINT'] + "/#{@request_id}" + '?client_id=' + ENV['CLIENT_ID'] + '&client_secret=' + ENV['CLIENT_SECRET']
  end
end
```

Our class does not have any responsibility for the content of the response. The only thing this class needs to do is to form the request correctly. Because in this case it’s building an endpoint from different variables & parameters there might be some merit in checking that the request is structured as expected.

One immediate thing to note is that this external request uses sensitive information. The request needs to be made successfully but you also don’t want to commit these values to source control. There are different ways to approach this problem and one is to use VCR’s in-built configuration that matches sensitive information and substitutes it for a benign replacement.

```ruby
  VCR.configure do |c|
    ...
    c.filter_sensitive_data('my-client-id') { ENV['CLIENT_ID'] }
    c.filter_sensitive_data('my-client-secret') { ENV['CLIENT_SECRET'] }
    c.filter_sensitive_data('https://some.api.endpoint') { ENV['EXTERNAL_REQUEST_ENDPOINT'] }
  end
```

What this ensures, is that although when the cassette is record it will use the real environment variables that it depends on to authenticate, it won’t commit those to the cassette.

## Making the Request

The next step is to record the cassette. Assuming the necessary configuration is in place, you can wrap the test which will be making an external call with the VCR block and a descriptive string. Then it’s a case of setting the test’s expectation as normal (in this case, that a request is made with a URL of a certain structure). In order to make and record the HTTP request we want to .and_call_original at the end of the expectation.

```ruby
RSpec.describe ExternalServiceFetcher do
  subject { ExternalServiceFetcher.new(1) }

  it 'builds the correct API request' do
    VCR.use_cassette('my_external_service_request') do
      expect(HTTParty).to receive(:get).with('https://some.api.endpoint/1?client_id=my-client-id&client_secret=my-client-secret').and_call_original
      subject.make_request
    end
  end
end
```

The first time you run this test, assuming all went well, a new file will appear located atspec/cassettes/my_external_service_request.yml. This file contains lots of interesting detail — the full response, headers and so on. Assuming the url that you passed to the HTTP request was formed correctly, then the test should pass.

## Testing the response

In most cases, once your class has made an HTTP request, it is then likely to want do something with the response. Sometimes the ExternalServiceFetcher might undertake some further work with the response, but for now let’s assume you just want to leave a test in place that confirms the structure of the response.

```ruby
class ExternalServiceFetcher
  ...
  def make_request
    begin
      api_response = HTTParty.get(build_url)
    rescue => e
      Rails.logger.error { "Encountered an error when making request {url}. #{e.message} #{e.backtrace.join("\n")}" }
    end
  end
  ....
end
```

Because this test is wrapped in a VCR instruction pointing a recorded cassette the ExternalFetcherService class can go ahead any make the HTTParty.get request that it’s expecting to do so. VCR now steps in and prevents an external call being made.

We can now test that the response from the HTTP request is of the structure that we expect

```ruby
  RSpec.describe ExternalServiceFetcher do
    subject { ExternalServiceFetcher.new(1) }
    let(:api_response) {
      a_property: 'a value',
      another_property: 'another_value'
    }
    ...

    it 'passes the API response to the parser class' do
      VCR.use_cassette('my_external_service_request') do
        response = subject.make_request
        expect(JSON.parse(response.body)).to eq(api_response)
      end
    end
  end
```

## Conclusion

There are a wide variety of approaches to testing HTTP requests in a Rails application. In some cases the above may be too much work, in same cases it may be insufficient. I’m generally comfortable with the approach above because

1. You’ve made a full ‘end to end’ request one time. You therefore know things work as expected…at least once.

1. The recorded response will create a test object that’s indistinguishable from the real response (providing the service is static). Giving you more confidence in the behaviour that depends on that response

1. From that point on your tests are fast and readable, there’s as little mocking or stubbing going on as you can get away with.
