# redis_token

API tokens redis store

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'redis_token'
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install redis_token

## Usage

```ruby
before_action :create_service

def auth
  client = Client.find_by_email(params[:email])

  if client.password == params[:password]
    token = @redis_token.create(owner: "client.#{client.id}", payload: { source: :native })
    json(access_token: token)

    # ...
  end

  # ...
end

def secured_method
  value = @redis_token.get(params[:access_token])
  return unathorized unless value

  client = Client.find_by_id(value[:owner])
  payload = value[:payload]

  # ...
end

def client_tokens
  @tokens = @redis_token.owned_by("client.#{client.id}").map do |token, value|
    { token: token, value: value }
  end
end

private

def create_service
  @redis_token ||= RedisToken.new(ttl: 30.days)
end
```

### RedisToken creation

Implicit Redis instance creation:

```ruby
# Redis.new with no arguments:
r = RedisToken.new(ttl: 30.days)

# Redis.new(host: '192.168.1.1', port: 33421)
r = RedisToken.new(host: '192.168.1.1', port: 33421)
```

Explicit Redis instance injection:

```ruby
redis = Redis.new
r = RedisToken.new(redis, ttl: 30.days)
r = RedisToken.new(redis)
```

### Create token

```ruby
client_token = r.create(owner: "client.#{client.id}")
# => "eca431add3b1f0bbc6cfc73980b68708"

# Redefine default ttl:
user_token = r.create(owner: "u:#{current_user.id}", ttl: 15.hours)
# => "548a5d54eaf474c750bf83ed04bd242a"

# Create token with payload:
mobile_token = r.create(owner: "c.#{client.id}", payload: { source: :native })
# => "865249d6b87c4e6dd8f6b0796ace7fa0"

# Save exist token:
r.create(owner: 'api', token: SecureRandom.uuid)
# => "aed6e179-14b4-4a8c-9a1b-6b0f9150ede3"

# Token without an owner:
r.create
# => "9dfb0d52280965fe4dd95d21447941c2"
```

### Get token

```ruby
value = r.get('865249d6b87c4e6dd8f6b0796ace7fa0')
# => {:owner=>"c.555", :at=>1503327773, :payload=>{:source=>:native}}

# Each get request slides ttl by default:
r.ttl('865249d6b87c4e6dd8f6b0796ace7fa0')
# => 1209493
r.get('865249d6b87c4e6dd8f6b0796ace7fa0')
# => {:owner=>"c.555", :at=>1503327773, :payload=>{:source=>:native}}
r.ttl('865249d6b87c4e6dd8f6b0796ace7fa0')
# => 1209598

# To prevent ttl sliding set slide_expire to false:
r.get('865249d6b87c4e6dd8f6b0796ace7fa0', slide_expire: false)

# Token without an owner:
rt.get('9dfb0d52280965fe4dd95d21447941c2')
# => {:at=>1503327773}
```

### Get all tokens owned by someone

```ruby
5.times { r.create(owner: 'u.555') }

r.owned_by('u.555').each { |t, v| p "#{t}: #{v}" }
# "5e8c661c11955b062d8c512c201734dd: {:owner=>\"u.555\", :at=>1503327773}"
# "5607a698763a6164975b9ffc06528513: {:owner=>\"u.555\", :at=>1503327773}"
# "7ef74cb761c1595dd33058e78a125720: {:owner=>\"u.555\", :at=>1503327773}"
# "621d2a10c34e92d7f0b4fc3b00be62af: {:owner=>\"u.555\", :at=>1503327773}"
# "17f7cb676e67d6b53c48e51f1c1beeb1: {:owner=>\"u.555\", :at=>1503327773}"
# => nil
```

### Deletion

Single token:
```ruby
r.delete('865249d6b87c4e6dd8f6b0796ace7fa0')
# => true
```

All tokens of an owner:
```ruby
r.delete_owned_by('client.1')
# => 8
```

All tokens without an owner:
```ruby
r.delete_without_owner
# => 1
```

All tokens:
```ruby
r.delete_all
# => 99
```

### Serialization

redis_token uses native Marshal class for data serialization by default. You can override it like this:

```ruby
require 'msgpack'

class MsgPackSerializer
  def pack(value)
    MessagePack.pack(value)
  end

  def unpack(value)
    MessagePack.unpack(value)
  end
end

r = RedisToken.new(serializer_class: MsgPackSerializer)
# Or
r = RedisToken.new
r.use(MsgPackSerializer)
```
