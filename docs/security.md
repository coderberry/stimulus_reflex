---
description: How to secure your StimulusReflex application
---

# Security

StimulusReflex leans on [ActionCable for security](https://guides.rubyonrails.org/action_cable_overview.html#server-side-components-connections), but here's a jump-start to get you up-and-running quickly.

### Encrypted Session Cookies

If you're just trying to bootstrap a proof-of-concept application on your local workstation, you don't technically have to worry about giving ActionCable the ability to distinguish between multiple concurrent users. However, the moment you deploy to a host with more than one person accessing your app, you'll find that you're sharing a session and seeing other people's updates.

You can use your default Rails encrypted cookie-based sessions to isolate your users into their own sessions:

{% code-tabs %}
{% code-tabs-item title="app/controllers/application\_controller.rb" %}
```ruby
class ApplicationController < ActionController::Base
  before_action :set_action_cable_identifier

  private

  def set_action_cable_identifier
    cookies.encrypted[:session_id] = session.id
  end
end	end
```
{% endcode-tabs-item %}
{% endcode-tabs %}

{% code-tabs %}
{% code-tabs-item title="app/channels/application\_cable/connection.rb" %}
```ruby
module ApplicationCable
  class Connection < ActionCable::Connection::Base
    identified_by :session_id

    def connect
      self.session_id = cookies.encrypted[:session_id]
    end
  end
end
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### User-based Authentication

Most Rails apps use the current\_user convention to provide a session context. This gives access to the person logged in from all parts of your application, including ActionCable.

{% hint style="info" %}
This should work with authentication solutions like [Devise](https://github.com/plataformatec/devise).
{% endhint %}

{% code-tabs %}
{% code-tabs-item title="app/controllers/application\_controller.rb  " %}
```ruby
class ApplicationController < ActionController::Base
  before_action :set_action_cable_identifier

  private

  def set_action_cable_identifier
    cookies.encrypted[:user_id] = current_user&.id
  end
end
```
{% endcode-tabs-item %}
{% endcode-tabs %}

{% code-tabs %}
{% code-tabs-item title="app/channels/application\_cable/connection.rb " %}
```ruby
module ApplicationCable
  class Connection < ActionCable::Connection::Base
    identified_by :current_user

    def connect
      user_id = cookies.encrypted[:user_id]
      return reject_unauthorized_connection if user_id.nil?
      user = User.find_by(id: user_id)
      return reject_unauthorized_connection if user.nil?
      self.current_user = user
    end
  end
end
```
{% endcode-tabs-item %}
{% endcode-tabs %}

{% code-tabs %}
{% code-tabs-item title="app/reflexes/example\_reflex.rb" %}
```ruby
class ExampleReflex < StimulusReflex::Reflex
  delegate :current_user, to: :channel

  def do_suff
    current_user.first_name
  end
end
```
{% endcode-tabs-item %}
{% endcode-tabs %}

