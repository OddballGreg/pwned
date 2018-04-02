# Pwned

An easy, Ruby way to use the Pwned Passwords API.

[![Gem Version](https://badge.fury.io/rb/pwned.svg)](https://rubygems.org/gems/pwned) [![Build Status](https://travis-ci.org/philnash/pwned.svg?branch=master)](https://travis-ci.org/philnash/pwned) [![Maintainability](https://codeclimate.com/github/philnash/pwned/badges/gpa.svg)](https://codeclimate.com/github/philnash/pwned/maintainability) [![Inline docs](https://inch-ci.org/github/philnash/pwned.svg?branch=master)](https://inch-ci.org/github/philnash/pwned)

[API docs](https://philnash.github.io/pwned/) | [GitHub repo](https://github.com/philnash/pwned)

## About

Troy Hunt's [Pwned Passwords API V2](https://haveibeenpwned.com/API/v2#PwnedPasswords) allows you to check if a password has been found in any of the huge data breaches.

`Pwned` is a Ruby library to use the Pwned Passwords API's [k-Anonymity model](https://www.troyhunt.com/ive-just-launched-pwned-passwords-version-2/#cloudflareprivacyandkanonymity) to test a password against the API without sending the entire password to the service.

The data from this API is provided by [Have I been pwned?](https://haveibeenpwned.com/). Before using the API, please check [the acceptable uses and license of the API](https://haveibeenpwned.com/API/v2#AcceptableUse).

Here is a blog post I wrote on [how to use this gem in your Ruby applications to make your users' passwords better](https://www.twilio.com/blog/2018/03/better-passwords-in-ruby-applications-pwned-passwords-api.html).

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'pwned'
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install pwned

## Usage

There are a few ways you can use this gem:

1. [Plain Ruby](#plain-ruby)
2. [Rails](#activerecord-validator)
3. [Rails and Devise](#devise)

### Plain Ruby

To test a password against the API, instantiate a `Pwned::Password` object and then ask if it is `pwned?`.

```ruby
password = Pwned::Password.new("password")
password.pwned?
#=> true
password.pwned_count
#=> 3303003
```

You can also check how many times the password appears in the dataset.

```ruby
password = Pwned::Password.new("password")
password.pwned_count
#=> 3303003
```

Since you are likely using this as part of a signup flow, it is recommended that you rescue errors so if the service does go down, your user journey is not disturbed.

```ruby
begin
  password = Pwned::Password.new("password")
  password.pwned?
rescue Pwned::Error => e
  # Ummm... don't worry about it, I guess?
end
```

Most of the times you only care if the password has been pwned before or not. You can use simplified accessors to check whether the password has been pwned, or how many times it was pwned:

```ruby
Pwned.pwned?("password")
#=> true
Pwned.pwned_count("password")
#=> 3303003
```

#### Advanced

You can set options and headers to be used with `open-uri` when making the request to the API. HTTP headers must be string keys and the [other options are available in the `OpenURI::OpenRead` module](https://ruby-doc.org/stdlib-2.5.0/libdoc/open-uri/rdoc/OpenURI/OpenRead.html#method-i-open).

```ruby
password = Pwned::Password.new("password", { 'User-Agent' => 'Super fun new user agent' })
```

### ActiveRecord Validator

There is a custom validator available for your ActiveRecord models:

```ruby
class User < ApplicationRecord
  validates :password, not_pwned: true
  # or
  validates :password, not_pwned: { message: "has been pwned %{count} times" }
end
```

#### I18n

You can change the error message using I18n (use `%{count}` to interpolate the number of times the password was seen in the data breaches):

```yaml
en:
  errors:
    messages:
      not_pwned: has been pwned %{count} times
      pwned_error: might be pwned
```

#### Threshold

If you are ok with the password appearing a certain number of times before you decide it is invalid, you can set a threshold. The validator will check whether the `pwned_count` is greater than the threshold.

```ruby
class User < ApplicationRecord
  # The record is marked as valid if the password has been used once in the breached data
  validates :password, not_pwned: { threshold: 1 }
end
```

#### Network Error Handling

By default the record will be treated as valid when we cannot reach the [haveibeenpwned.com](https://haveibeenpwned.com/) servers. This can be changed with the `:on_error` validator parameter:

```ruby
class User < ApplicationRecord
  # The record is marked as valid on network errors.
  validates :password, not_pwned: true
  validates :password, not_pwned: { on_error: :valid }

  # The record is marked as invalid on network errors
  # (error message "could not be verified against the past data breaches".)
  validates :password, not_pwned: { on_error: :invalid }

  # The record is marked as invalid on network errors with custom error.
  validates :password, not_pwned: { on_error: :invalid, error_message: "might be pwned" }

  # We will raise an error on network errors.
  # This means that `record.valid?` will raise `Pwned::Error`.
  # Not recommended to use in production.
  validates :password, not_pwned: { on_error: :raise_error }

  # Call custom proc on error. For example, capture errors in Sentry,
  # but do not mark the record as invalid.
  validates :password, not_pwned: {
    on_error: ->(record, error) { Raven.capture_exception(error) }
  }
end
```

#### Custom Request Options

You can configure network requests made from the validator using `:request_options` (see [OpenURI::OpenRead#open](http://ruby-doc.org/stdlib-2.5.0/libdoc/open-uri/rdoc/OpenURI/OpenRead.html#method-i-open) for the list of available options, string keys represent custom network request headers, e.g. `"User-Agent"`):

```ruby
  validates :password, not_pwned: {
    request_options: { read_timeout: 5, open_timeout: 1, "User-Agent" => "Super fun user agent" }
  }
```

### Devise

If you are using Devise I recommend you use the [devise-pwned_password extension](https://github.com/michaelbanfield/devise-pwned_password) which is now powered by this gem.

## Development

After checking out the repo, run `bin/setup` to install dependencies. Then, run `rake spec` to run the tests. You can also run `bin/console` for an interactive prompt that will allow you to experiment.

To install this gem onto your local machine, run `bundle exec rake install`. To release a new version, update the version number in `version.rb`, and then run `bundle exec rake release`, which will create a git tag for the version, push git commits and tags, and push the `.gem` file to [rubygems.org](https://rubygems.org).

## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/philnash/pwned. This project is intended to be a safe, welcoming space for collaboration, and contributors are expected to adhere to the [Contributor Covenant](http://contributor-covenant.org) code of conduct.

## License

The gem is available as open source under the terms of the [MIT License](https://opensource.org/licenses/MIT).

## Code of Conduct

Everyone interacting in the Pwned project’s codebases, issue trackers, chat rooms and mailing lists is expected to follow the [code of conduct](https://github.com/philnash/pwned/blob/master/CODE_OF_CONDUCT.md).
