## **Launch the project**

Create Rails project
```shell
    $ rails new project_name
    $ cd project_name
    $ stt
    $ rails db:create db:migrate
```
Add the gem 'messenger-bot' but the MatthiasRMS version
```ruby
    gem 'messenger-bot', :git => 'git://github.com/MatthiasRMS/messenger-bot-rails'
    
    $ bundle install
```
 **Facebook Configuration** 

- Create a FaceBook page

 [https://www.facebook.com/pages/create/](https://www.facebook.com/pages/create/) 

- Create a FaceBook app :

 [https://developers.facebook.com/](https://developers.facebook.com/) 

- Add a Messenger Product to your App
- Messenger / Settings : Generate a Token by selecting your page on the dropdown

![](https://static.notion-static.com/4d0b0b7683d14853aa6f3aaae5a1037b/Screen_Shot_2017-11-18_at_02.35.11.png)

- Suscribe your page to your webhook (https ngrok url), ex: [https://172dfb36.ngrok.io/webhook](https://172dfb36.ngrok.io/webhook) with the token you just created

![](https://static.notion-static.com/7f8eb0ef047242ee9a5e352e56f7609d/Screen_Shot_2017-11-18_at_02.57.37.png)

- Subscribe to the following events :

![](https://static.notion-static.com/b277f9d90d244f42befc347c4a056f06/Screen_Shot_2017-11-18_at_02.52.07.png)

- Select a page to subscribe your webhook to the page events

![](https://static.notion-static.com/d542941d87ce4b45b799382ff3c59fe8/Screen_Shot_2017-11-18_at_02.53.16.png)

- Report the credentials in the `application.yml` : Page Token and App Secret Key
```ruby
    development:
    FB_PAGE_KEY: 'EAAB2lF1nr5ABAIZCIU.....etw8IqCqzh2kPDCJAu'
    FB_SECRET_PASS: '8337f0cd.....192c45c'
    
    staging:
    FB_PAGE_KEY: 'EAAB2lF1nr5ABAIZCIU.....etw8IqCqzh2kPDCJAu'
    FB_SECRET_PASS: '8337f0cd.....192c45c'
    
    production:
    FB_PAGE_KEY: 'EAAB2lF1nr5ABAIZCIU.....etw8IqCqzh2kPDCJAu'
    FB_SECRET_PASS: '8337f0cd.....192c45c'
```
 
**LET'S CODE THE BACKEND** 

- Create a `messenger_bot.rb` in the initializers
- Configure the initializer :
```ruby
    Messenger::Bot.config do |config|
    	config.access_token = ENV['FB_PAGE_KEY']
    	config.validation_token = ENV['FB_PAGE_KEY']
    	config.secret_token = ENV['FB_SECRET_PASS']
    end
 ```
- Add those routes :
```ruby
    mount Messenger::Bot::Space => "/webhook"
```
- Create a new controller `MessengerBotController` :
```ruby
    class MessengerBotController < ActionController::Base
    
        def message(event, sender)
    		 # profile = sender.get_profile(field) # default field [:locale, :timezone, :gender, :first_name, :last_name, :profile_pic]
            sender.reply({ text: "Reply: #{event['message']['text']}" })
        end
    
        def delivery(event, sender)
        end
    	
    	def optin(event, sender)
        end
    
        def postback(event, sender)
            payload = event["postback"]["payload"]
            case payload
            when :something
            #ex: process sender.reply({text: "button click event!"})
            end
        end
    end
```
 [https://developers.facebook.com/docs/messenger-platform/webhook#subscribe](https://developers.facebook.com/docs/messenger-platform/webhook#subscribe) 

- NB : if you are using devise you need to skip the authentication for this controller :
```ruby
    class MessengerBotController < ActionController::Base
    	skip_before_action :authenticate_user!
    	skip_before_action :verify_authenticity_token
    	....
    end
```
## **How to test the chatbot ?**

We need a server running not locally but accessible on the web

 `ngrok` will help us to expose our localhost on the web

https://ngrok.com/

Save the `ngrok` file in the same folder as your app and launch it with the commande : `../ngrok http 3000` 

then run your server : `rails s` 

 _NB: if you make changes on your code never stop the ngrok tunnel but relaunch your server_ 

 **What we received from messenger ?** 

Event : json informations about the event 

Sender : informations about the sender (use `sender.get_profile` to have the details)
```ruby
    sender_id = event["sender"]["id"].to_i
    first_name = sender.get_profile[:body]["first_name"]
    last_name = sender.get_profile[:body]["last_name"]
    name = (first_name + last_name).downcase
    
    profile_pic = sender.get_profile[:body]["profile_pic"]
    msg = event["message"]["text"]
```
## **Some usefull function to add in the controller**

 **Find or Create a user** 

each time your backend receive a message you should find in your database if the user exist to assign the conversation or create the user
```ruby
    def find_or_create_user(first_name, last_name, sender_id)
    	if User.find_by(facebook_id: sender_id)
    		User.find_by(facebook_id: sender_id)
    	else
    		User.create(email: "#{sender_id}@fake_email.com",first_name: first_name, last_name: last_name,password: sender_id,facebook_id:              sender_id)
    	end
    end
```
 **Find or Create a session** 

each time your backend receive a message you want to create or update the user session (cf session concept bellow)
```ruby
    def find_or_create_session(user_id)#, max_age: 59.minutes)
    	if Session.find_by("user_id = ?", user_id )# AND last_exchange >= ? ", max_age.ago)
    	 session = Session.find_by("user_id = ?", user_id )# AND last_exchange >= ? ", user_id, max_age.ago)
     session.update(last_exchange: Time.now)
    	 return session
    	else
    	 session = Session.create(context: {}, previous_context: {}, user_id: user_id, last_exchange: Time.now)
    	 return session
    	end
    end
```
## Let's template things

 [https://developers.facebook.com/docs/messenger-platform/send-messages/templates](https://developers.facebook.com/docs/messenger-platform/send-messages/templates) 

The gem manage several templates :

1. button.rb
2. file.rb
3. generic.rb
4. image.rb
5. list.rb
6. quick_reply.rb
7. receipt.rb
8. text.rb

## Asynchronous system : How to manage that ? creating a Session !

By saving the state in the database

We will create a table Sessions to save the context and the previous context
```ruby
    rails g migration CreateSessions
```

5 fields :
```ruby
    create_table :sessions do |t|
    	t.jsonb :context
    	t.jsonb :previous_context
        t.references :user
        t.datetime :last_exchange
        t.timestamps
     end

    rails db:migrate
```
## Send info to user
Create a SendRequest function on apps/services/send_request.rb
```ruby
    module SendRequest
      def self.send(message, sender_id)
        token = ENV['FB_PAGE_KEY']
        url = ENV['FB_URL']
        request_params =  {
          recipient: {id: sender_id},
          message: message,
          access_token: token
        }

        RestClient.post url, request_params.to_json, :content_type => :json, :accept => :json
      end
    end
```
Add the environment variable `FB_URL` in `application.yml`:
```ruby
    FB_URL: "https://graph.facebook.com/me/messages?access_token="
```
To send a message to a user without chatting with him you can use the `SendRequest` services. Ex:
```ruby
    def optin(event, sender)
      message_json = {
        "text": "♥️ from Antoine"
        }
      SendRequest.send(message_json,sender.sender_id)
   end
```
## FaceBook Approval

FB should approve your bot. Your need to follow the process down in the messenger product page. You need the 2 first approval.

![](https://static.notion-static.com/326176abcb534760b30d2dfce29c3c75/Screen_Shot_2017-11-18_at_03.02.36.png)

## Send to Messenger Plugin

The "Send to Messenger" plugin is used to trigger an authentication event to your webhook. You can pass in data to know which user and transaction was tied to the authentication event, and link the user on your back-end.

 [https://developers.facebook.com/docs/messenger-platform/discovery/send-to-messenger-plugin](https://developers.facebook.com/docs/messenger-platform/discovery/send-to-messenger-plugin) 

![](https://static.notion-static.com/1440523fec574c09ba2133d89d3d924b/Screen_Shot_2017-11-19_at_23.08.13.png)

Add this script after the `body` of the `application.html.erb` 
```javascript
    <script>
    	window.fbAsyncInit = function() {
     FB.init({
     appId: <%= ENV['FB_APP_ID'] %>,
     xfbml: true,
     version: "v2.6"
     });
    
     FB.Event.subscribe('send_to_messenger', function(e) {
     // callback for events triggered by the plugin
     });
    
     };
    
     (function(d, s, id){
     var js, fjs = d.getElementsByTagName(s)[0];
     if (d.getElementById(id)) { return; }
     js = d.createElement(s); js.id = id;
     js.src = "//connect.facebook.net/en_US/sdk.js";
     fjs.parentNode.insertBefore(js, fjs);
     }(document, 'script', 'facebook-jssdk'));
    </script>
```
Add this to your HTML Page where you want to display the button
```html
    <div class="fb-send-to-messenger"
     messenger_app_id=<%= ENV['FB_APP_ID'] %>
     page_id= <%= ENV['FB_PAGE_ID'] %>
     data-ref= "Hello"
     color="blue"
     size="standard">
    </div>
```
 **Event Subscription** 

Subscribe to plugin events.
```javascript
    <script>
    
     FB.Event.subscribe('send_to_messenger', function(e) {
     // callback for events triggered by the plugin
     
     });
    
    </script>
```

**Callback** 

This triggers the  [opt-in callback](https://developers.facebook.com/docs/messenger-platform/webhook-reference/optins) .
