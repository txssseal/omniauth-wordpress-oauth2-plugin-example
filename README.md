omniauth-wordpress-oauth2-plugin-example
========================================

Example rails app demoing configuration.

## Steps
Remember to setup wordpress site, install the plugin oauth-server on wordpress by justin greer.  Set Permalinks to Post Name. 

Go to Settings -> Oauth Server -> clients

add a new client with the redirect like 

```
http://your-rails-site.com/users/auth/wordpress_hosted/callback
```

####1. Create new rails app.
  
```rails new omniauth-wordpress-oauth2-plugin-example . --database=postgres -T``` 

Grab the Key and Secret for devise config

####2. Add devise / omniauth gems to configuration file. `Gemfile`

```ruby
#authentication bits
gem 'devise'
gem 'omniauth'
gem 'omniauth-wordpress_hosted', github: 'txssseal/omniauth-wordpress-oauth2-plugin'
gem 'oauth2'
```
####3. Run bundle install

`bundle install`

####4. Run devise install / follow installation instructions post generator.

`rails g devise:install`

####5. Generate devise user

`rails g devise user`

run migrations

`rails db:migrate`


####7. Configure Devise / Omniauth provider information

Add provider to devise initializer `config/initializers/devise.rb`

```ruby
 config.omniauth :wordpress_hosted, 'APP_ID', 'APP_SECRET',
                  strategy_class: OmniAuth::Strategies::WordpressHosted,
                  client_options: { site: 'http://yourcustomwordpress.com' }
```

####8. Add routes configuration

Update routes `config/routes.rb` to add omniauth_callbacks controller

```ruby
devise_for :users, controllers: { omniauth_callbacks: 'omniauth_callbacks' }
```

####9. Create Callbacks Controller

Easiest to just create the class `app/controllers/omniauth_callbacks_controller.rb` instead of running generator.

```ruby
class OmniauthCallbacksController < Devise::OmniauthCallbacksController
  def wordpress_hosted
    #You need to implement the method below in your model (e.g. app/models/user.rb)
    @user = User.find_for_wordpress_oauth2(request.env["omniauth.auth"], current_user)

    if @user.persisted?
      flash[:notice] = I18n.t "devise.omniauth_callbacks.success", :kind => "Wordpress Oauth2"
      sign_in_and_redirect @user, :event => :authentication #this will throw if @user is not activated
    else
      session["devise.wordpress_oauth2_data"] = request.env["omniauth.auth"]
      redirect_to new_user_registration_url
    end
  end
end
```

####10. Update User Model

Update user to be omniauthable

```ruby
devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :trackable, :validatable, :omniauthable, :omniauthable, :omniauth_providers => [:wordpress_hosted]
```

Update User model to find users by oauth provider data:

```ruby
def self.find_for_wordpress_oauth2(oauth, signed_in_user=nil)

    #if the user was already signed in / but they navigated through the authorization with wordpress
    if signed_in_user

      #update / synch any information you want from the authentication service.
      if signed_in_user.email.nil? or signed_in_user.email.eql?('')
        signed_in_user.update_attributes(email: oauth['info']['email'])
      end

      return signed_in_user
    else
      #find user by id and provider.
      user = User.find_by_provider_and_uid(oauth['provider'], oauth['uid'])

      #if user isn't in our dabase yet, create it!
      if user.nil?
        user = User.create!(email: oauth['info']['email'], uid: oauth['uid'], provider: oauth['provider'],
                            nickname: oauth['extra']['user_login'], website: oauth['info']['urls']['Website'],
                            display_name: oauth['extra']['display_name'])
      end

      user
    end

  end
```

## License

The MIT License (MIT)

Copyright (c) 2013 Joel Wickard

Permission is hereby granted, free of charge, to any person obtaining a copy of
this software and associated documentation files (the "Software"), to deal in
the Software without restriction, including without limitation the rights to
use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of
the Software, and to permit persons to whom the Software is furnished to do so,
subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS
FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR
COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER
IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN
CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
