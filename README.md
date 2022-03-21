Objectives
-

- Learn Serialization Customizations
- Handling CORS
- Filtering and Ordering the index action
- Stateless authentication for the API

Due Date:
-

**March 24, 2022** by the end of your lab session!

Important Note
===============

In this lab, you have a total of **4 checkpoints** to validate with one of the teaching team members during the lab session.

Checkpoints will be graded as follows:

- Checkpoint 1: 30 points
- Checkpoint 2: 20 points
- Checkpoint 3: 25 points
- Checkpoint 4: 25 points

**If you have any issues with swagger docs not working properly - clear your cache. In Google Chrome, a hard reload should work.**

## Part 1 - API Custom Serialization

1. In Lab 8, last week, we created the barebone API for ChoreTracker and documenting it using Swagger-docs and Swagger-UI. The Basic API as well as the SAwagger-docs are already provided to you in this starter code. But, there are a lot more things you can do to improve it and make it more usable. One main thing is **serialization**, which is how Rails converts a Child/Task/Chore model object to JSON. With serializers, you can truly customize how you want these objects to show up in your API. One good example of this is to display all the chores that are tied to a child when viewing the show action of a child. While we could use [active_models_serializers](https://www.rubydoc.info/gems/active_model_serializers), for this lab we are going to use [fast_jsonapi](https://github.com/jsonapi-serializer/jsonapi-serializer), which is much faster than `active_models_serializers`.

First of all, add the gem to your Gemfile: `gem 'fast_jsonapi'` and run `bundle install`.

2. Now, you can actually generate some boilerplate code for your serializer, by running `rails g serializer <model_name>` so for example, `rails g serializer child` will create a new file called `child_serializer.rb` in the serializers folder in app. **Please, generate serializer files for each controller** (child, chore, and task).

3. By default for the child serializer, you should just see the following. This means that when serializing a child object to JSON it will only display the attributes listed after "attributes".
    
  ```ruby
  class ChildSerializer
    include FastJsonapi::ObjectSerializer
    attributes 
  end
  ```

4. Let's start off by adding what we want to the ChildSerializer. In this case, we want to display the id, name of the child, whether or not it is active, and the list of chores that it has. To do this, after the `:id`, also add `:name` and `:active`. The reason that name works even though the Child Model doesn't have a name attribute (only first_name and last_name) is that we had already defined a method called `name` in the Child Model that combines the first and last names. Next, we need to get all the chores that is related to this child. To do so, we need to add a relationship, just like with the model, by writing `has_many :chores`. Your ChildSerializer should look something like the following. Now, look at the output on Swagger Docs and try to view all children (Note: run the rails server, then go to localhost:<your port>/api  to see the documentation).
    
  ```ruby
  class ChildSerializer
    include FastJsonapi::ObjectSerializer
    attributes :id, :name, :active
    has_many :chores
  end
  ``` 

5. You have probably noticed that Swagger Docs is showing much more information than what we specified in the serializer. For example, when we try to get the information of one child, we see additional attributes such as `created_on`. The reason for this is that we are not actually calling the serializers on our objects. Go to the `children_controller` and change the show action so that it looks like following. Make this similar change in all other children_controller actions (index, create, update) that render JSON objects. Check Swagger Docs to verify that it worked.

  ```ruby
  def show
    render json: ChildSerializer.new(@child)
  end 
  ```

6. Let's go onto fixing the TaskSerializer. For this follow the same idea, but note that we only need to display the **id**, **name**, **points** that its worth, and whether or not it is **active**. 

7. After that, you should go onto adding serialization to the ChoreSerializer, which should include the **id**, **child_id**, **task_id**, and **due_on**.

We also want to add an extra **Completed** attribute to our serializer. We want this attribute to show as "completed" or "pending" (not just true or false). For implementing `:completed` to be not a geeky True/False in the API, we can add the following method to our serializer that takes advantage of our model method:

  ```ruby
  attribute :completed do |object|
    object.status
  end
  ```

8. Verify that your serializers worked properly by using the SwaggerDocs. Make sure that you restarted your server before doing so.

9. At this point, you have just very standard serialization for each of these models. Let's make ChildSerializer more interesting! It would probably be useful to include the total number of points that the child has earned (good thing we wrote the `points_earned` function already in the model). Include that as an attribute of the ChildSerializer (as specified in the code below). 

Next, it probably makes more sense to break up the chores list into completed and unfinished chores for each child. You will need to write a custom method to do this and won't need the relationship to chores. In this case, the variable ```object``` will always represent the current object that you are trying to serialize, so we are getting all the chores tied to the specific child and running the done and pending scopes on it. You can imagine ```object``` to be similar to ```self```. After getting each of the relations, we still need to manually serialize each one using the ChoreSerializer class.

  ```ruby
    class ChildSerializer
      include FastJsonapi::ObjectSerializer
      attributes :id, :name, :points_earned, :active

      attribute :completed_chores do |object|
        object.chores.done.map do |chore|
          ChoreSerializer.new(chore)
        end
      end

      attribute :pending_chores do |object|
        object.chores.pending.map do |chore|
          ChoreSerializer.new(chore)
        end
      end

    end
  ```

10. To make another improvement to the serialization is to actually allow users to preview what task the chore entails instead of just an id. This is a little bit more complex since we can't just have one serializer for tasks. We want one serializer that shows all information about a task when we hit the index action of the tasks controller, and we want another serializer that previews the task with just the **id** and the **name**. To do this we need another serializer! 

11. Make another file in the serializers folder and call it ```chore_task_serializer.rb``` and make a new class called ```ChoreTaskSerializer```. Have this serializer give back only the `id` and `name` of the object.

12. Now, go to your chore_serializer and instead of displaying task_id, have it display `:task` by writing a custom serialization method called `task`. In this method, all you need to return is the preview version of the serialized task which includes the `:id` and `name` of the task (from the ChoreTaskSerializer). Call over a CA if you are having trouble with this concept! Now test if it worked by going to the /children endpoint! It should display the task id and name instead of just the task id.

  ```ruby
  attribute :task do |object|
    ChoreTaskSerializer.new(object.task)
  end
  ```


# <span class="mega-icon mega-icon-issue-opened"></span> Stop
**Checkpoint 1**
Show a CA that you have properly serialized JSON objects in the ChoreTrackerAPI!
* * *

## Part 2 - CORS

CORS stands for Cross Origin Resource Sharing. Most web applications have CORS disabled, which means that the web app prevents JavaScript from making requests that are outside of the domain. For example, let's say that the web application is hosted on ```cmuis.net```, if CORS is disabled then Javascript code located on another domain (```testdomain.com```) can't make a request to cmuis.net. This is meant to protect malicious Javascript from making requests to your web application.

However, for the purposes of an API, we want CORS to be enabled since we would like code from other domains to access our API. The only reason that the swagger docs works in hitting the API's endpoints is that the swagger docs is located in the same domain. To demonstrate that CORS isn't enabled right now, please create another HTML file (let's call it ```cors_test.html```) and copy paste the code below:

To create this html file, type `touch cors_test.html` in the terminal. Then, you will see the file appearing in the explorer (below config.ru).
Make sure you replace 3000 with your port, in the url line in the code below.

```html
<!DOCTYPE html>
<html>
<head>
  <title>CORS Test</title>
  <script src="https://code.jquery.com/jquery-3.2.1.min.js"></script>
  <script type="text/javascript">
    $.ajax({
      method: "GET",
      url: "http://localhost:3000/children",
      success: function(res) {
        alert("Success");
        console.log(res);
      },
      error: function(res) {
        alert("Error");
        console.log(res);
      }
    })
  </script>
</head>
<body>
  <h1>CORS Test</h1>
</body>
</html>
```

After creating this new simple HTML page , just open it up. After it tries to make an AJAX request to the ```/children``` endpoint, it should alert out "Error" and if you look in the Console (Right click -> Inspect Element -> Console) there should be an error message saying "No 'Access-Control-Allow-Origin' header is present". This basically means that the API located at ```localhost:3000/children``` doesn't allow for CORS access.

1. In order to fix this for your Chore Tracker API, you will need this new gem called `rack-cors` (Read more about it here: https://github.com/cyu/rack-cors). Add `gem 'rack-cors'` to your Gemfile and bundle install.

2. Next go to the ```config/application.rb``` file and add the following code to it within the Application class at the end. Notice the code ```:methods => [:get, :post, :put]```, this is how rack-cors will be able to whitelist certain types of request. For example, if you don't want anyone from another domain to make post requests (or create new things) to your API, then remove that. If you want to allow them to make delete requests, then add it in like this: ```:methods => [:get, :post, :put, :delete]```.

    ```
    module ChoreTrackerAPI
      class Application < Rails::Application
        # some other code
        # ...

        config.middleware.insert_before 0, Rack::Cors do
          allow do
            origins '*'
            resource '*', :headers => :any, :methods => [:get, :post, :put]
          end
        end

      end
    end
    ```

3. Restart your server and try to refresh the ```cors_test.html``` page. Now it should alert "Success" and if you look at the console again, the original error should be gone and the actual children JSON objects will be displayed.


# <span class="mega-icon mega-icon-issue-opened"></span> Stop
**Checkpoint 2**
Show a CA that your API now allows for Cross Origin Requests!
* * *

# Part 3 - Filtering and Ordering

One thing that we will be improving upon in this lab is filtering and ordering. This is mainly for the the index action of each controller and allows users of your API to filter out and order the list of objects. For example, if you want to get all the active tasks right now, you will have to hit the `/tasks` endpoint to get all the tasks back and then filter out the inactive ones manually using Javascript. However, a better option would be to pass in an active parameter that states what you want. So for the example, `/tasks?active=true` will get you all the active tasks, `/tasks?active=false` will get you all the inactive tasks, and `/tasks` will get you all the tasks. With the format, you can concat different filters and ordering together.

1. Let's first add this feature to the children endpoint! Open up the `child.rb` model file first and notice the scopes that are present (:active and :alphabetical). :active is a filtering scope and :alphabetical is a ordering scope. In this case, we will probably need another scope called `:inactive` to be the opposite of the :active filtering scope. Add the following :inactive scope to the child model file:

    ```ruby
    scope :inactive, -> {where(active: false)}
    ```

2. Now go the ChildrenController and let's add this new active filter to the index action. In this case, the `:active` param will be the one that triggers the filter and do nothing if the param isn't present. Also the only reason that we are checking if it's equal to the string "true" is that params are all treated as strings. Copy the following code into the index action (ask a TA for help if you don't understand the logic here):

    ```ruby
    def index
      @children = Child.all
      if(params[:active].present?)
        @children = params[:active] == "true" ? @children.active : @children.inactive
      end

      render json: ChildSerializer.new(@children)
    end
    ```

3. Since there was also an `:alphabetical` ordering scope, we will need to add that to the index action too. In this case, it will behave slightly different than the filtering scope. This is because it will only alphabetically order the children if the `:alphabetical` param is present and true. Add the following right after the active filter param in the index action (**Note:** The reasons why we are checking if the param is equal to the string "true" rather than the boolean is because all params are interpreted as strings initially):

    ```ruby
    if params[:alphabetical].present? && params[:alphabetical] == "true"
      @children = @children.alphabetical
    end
    ```

4. Now before we test this out, we will need to add the proper params to the swagger docs. Add the following ```:query``` params to the ChildrenController's swagger docs' index action and test it out (**Note**: don't forget to run ```rails swagger:docs``` afterward):

    ```ruby
    param :query, :active, :boolean, :optional, "Filter on whether or not the child is active"
    param :query, :alphabetical, :boolean, :optional, "Order children by alphabetical"
    ```

5. After you tested everything out for children with swagger docs, we will move on to doing the same thing for tasks and chores. Since Tasks is basically the same as Children, you will be completing the ```:active``` and ```:alphabetical``` filtering/ordering scopes on your own. (**Note**: Make sure you add the necessary scopes to the task model.)

6. Chores is a bit more complicated but not that much. All the necessary scopes are there for you. You will be creating the filtering params on your own for ```:done``` and ```:upcoming``` (where ```:pending``` and ```:past``` are the opposite scopes respectively). Also, you will be creating the ordering params ```:chronological``` and ```:by_task```. 

7. Make sure you add all the appropriate swagger docs to the index actions of each controller and test out the filtering/ordering params.


# <span class="mega-icon mega-icon-issue-opened"></span>Stop
**Checkpoint 3**
Show a CA that you have all the filtering and ordering params working for all the controllers. 
* * *

# Part 4 - Token Authentication


1. Now we will tackle authentication for API's since we don't want just anyone modifying the chores. (It would be a hoot to let the children just mark off their own chores regardless if they were actually done or not. Even better would be to allow children to reassign chores to siblings!) This will be slightly different from authentication for regular Rails applications mainly because the authentication will be stateless and we will be using a token (instead of an email and password). For this to work, we will first need to create a User model! Follow the specifications below and generate a new User model and run ```rails db:migrate```. Note that there is still an email and password because we still want there to be a way later on for users to retrieve their authentication token (if they forgot it) by authentication through email and password.
    - User
        - email (string)
        - password_digest (string)
        - api_key (string)
        - active (boolean)

2. For now let's fill the User model with some validations. This is pretty standard and we have already done something similar before, so just copy paste the code below to your User model.

    ``` ruby
    class User < ApplicationRecord
      has_secure_password
    
      validates_presence_of :email
      validates_uniqueness_of :email, allow_blank: true
      validates_presence_of :password, on: :create 
      validates_presence_of :password_confirmation, on: :create 
      validates_confirmation_of :password, message: "does not match"
      validates_length_of :password, minimum: 4, message: "must be at least 4 characters long", allow_blank: true
    end
    ```

3. So the general idea of the `api_key` is so that when someone sends a GET/POST/etc. request to your API, they will also need to provide the token in a header. Your API will then try to authenticate with that token and see what authorization that user has. This means that the `api_key` needs to be unique so we will not be allowing users to change/create the api_key. Instead, we will be generating a random api_key for each user when it is created. Therefore we will write a new callback function in the model code for creating the api_key. The following is the new model code. **Please understand it before continuing, or else everything will be rather confusing!** (**Note:** Don't forget to add the ```gem 'bcrypt'``` to the Gemfile for passwords and run bundle install).

    ``` ruby
    class User < ApplicationRecord
      has_secure_password
    
      validates_presence_of :email
      validates_uniqueness_of :email, allow_blank: true
      validates_presence_of :password, on: :create 
      validates_presence_of :password_confirmation, on: :create 
      validates_confirmation_of :password, message: "does not match"
      validates_length_of :password, minimum: 4, message: "must be at least 4 characters long", allow_blank: true
      validates_uniqueness_of :api_key
    
      before_create :generate_api_key
    
      def generate_api_key
        begin
          self.api_key = SecureRandom.hex
        end while User.exists?(api_key: self.api_key)
      end
    end
    ```

4. Now we should create the User controller and the Swagger Docs for the controller. This should be quick since you have done this already for all the other controllers. (**Note:** make sure that the user_params method only permits these parameters because we don't want them creating the api_key: `params.permit(:email, :password, :password_confirmation, :active))` After you are done, verify that it is the same as below and make sure the create documentation has the right form parameters. **Also add the user resources to the routes.rb and run ```rails swagger:docs```**

    ``` ruby
    class UsersController < ApplicationController
      # This is to tell the gem that this controller is an API
      swagger_controller :users, "Users Management"
    
      # Each API endpoint index, show, create, etc. has to have one of these descriptions
    
      # This one is for the index action. The notes param is optional but helps describe what the index endpoint does
      swagger_api :index do
        summary "Fetches all Users"
        notes "This lists all the users"
      end
    
      # Show needs a param which is which user id to show.
      # The param defines that it is in the path, and that it is the User's ID
      # The response params here define what type of error responses can be returned back to the user from your API. In this case, the error responses are 404 not_found and not_acceptable.
      swagger_api :show do
        summary "Shows one User"
        param :path, :id, :integer, :required, "User ID"
        notes "This lists details of one user"
        response :not_found
        response :not_acceptable
      end
    
      # Create doesn't take in the user id, but rather the required fields for a user (namely first_name and last_name)
      # Instead of a path param, this uses form params and defines them as required
      swagger_api :create do
        summary "Creates a new User"
        param :form, :email, :string, :required, "Email"
        param :form, :password, :password, :required, "Password"
        param :form, :password_confirmation, :password, :required, "Password Confirmation"
        param :form, :active, :boolean, :required, "active"
        response :not_acceptable
      end
    
      # Lastly destroy is just like the rest and just takes in the param path for user id. 
      swagger_api :destroy do
        summary "Deletes an existing User"
        param :path, :id, :integer, :required, "User Id"
        response :not_found
        response :not_acceptable
      end
    
    
      # Controller Code
    
      before_action :set_user, only: [:show, :update, :destroy]
    
      # GET /users
      def index
        @users = User.all
    
        render json: @users
      end
    
      # GET /users/1
      def show
        render json: @user
      end
    
      # POST /users
      def create
        @user = User.new(user_params)
    
        if @user.save
          render json: @user, status: :created, location: @user
        else
          render json: @user.errors, status: :unprocessable_entity
        end
      end
    
      # DELETE /users/1
      def destroy
        @user.destroy
      end
    
      private
        # Use callbacks to share common setup or constraints between actions.
        def set_user
          @user = User.find(params[:id])
        end
    
        # Only allow a trusted parameter "white list" through.
        def user_params
          params.permit(:email, :password, :password_confirmation, :active)
        end
    end
    ```

5. We should also create a new serializer for users since we really don't want to display the password_digest, but we do need to show the api_key. **Make sure to call the serializers in the controller actions.**

6. We can now start up the rails server and test out whether or not our user model creation worked! Create a new user using Swagger and **save the api_key from the response**, this is **very important** for the next steps!

7. Next we need to actually implement the authentication with the tokens so that nobody can modify anything in the system without having a proper token. You will need to add the following to the ApplicationController. This uses the built-in ```authenticate_with_http_token``` method which checks if it is a valid token and if anything fails, it will just render the Bad Credentials JSON. How it works is that every request that comes through has to have an Authorization header with the specified token and that is what rails will check in order to authenticate. Also for simplicity, we authenticated for all actions in all controllers by putting a before_action in the ApplicationController.

    ```ruby
    class ApplicationController < ActionController::API
      include ActionController::HttpAuthentication::Token::ControllerMethods
    
      before_action :authenticate
    
      protected
    
      def authenticate
        authenticate_token || render_unauthorized
      end
    
      def authenticate_token
        authenticate_with_http_token do |token, options|
          @current_user = User.find_by(api_key: token)
        end
      end
    
      def render_unauthorized(realm = "Application")
        self.headers["WWW-Authenticate"] = %(Token realm="#{realm.gsub(/"/, "")}")
        render json: {error: "Bad Credentials"}, status: :unauthorized
      end
    end
    ```

8. If you restart the server now and try to use Swagger to test out any of the endpoints in any controller, you will be faced with the Bad Credentials message. To fix this we need to change the swagger docs so that it will pass along the token in the headers of every request. There are two ways to do this, one way is to add another header param to every single endpoint; another way is to add a setup method for swagger docs to pick up. In order to do this, all we need to do is write this singleton class within in the ApplicationController class so that it will affect all of the other controllers. This code goes to all the subclasses of ApplicationController and then adds the header param to each of the actions. (**Note:** Make sure you put this within the ApplicationController class)

    ```ruby
    class << self
      def inherited(subclass)
        super
        subclass.class_eval do
          setup_basic_api_documentation
        end
      end
    
      private
      def setup_basic_api_documentation
        [:index, :show, :create, :update, :delete].each do |api_action|
          swagger_api api_action do
            param :header, 'Authorization', :string, :required, 'Authentication token in the format of: Token token=<token>'
          end
        end
      end
    end
    ```

9. Make sure you run ```rails swagger:docs```, start up the server and check out the swagger docs. For each endpoint, there should be a header param. In order to successfully hit any of the endpoints, you will need to fill out this param too. This is a little bit more complicated as before since rails has its own format/way to do things. In the input box, enter ```Token token=<api_key>``` and replace `<api_key>` with the key from the user you created before. Now check that the API works with the token authentication!

10. Now that you have the token authentication implemented for each of the endpoints, there's no way for someone to access the API if they forgot their token. However, a user will most likely remember their email/password and forget their api_key, which is why we will need to create one more endpoint where users will be able to retrieve their token with their correct email/password. Let's call this endpoint ```/token```. **Ask a TA if you don't understand the purpose of this new endpoint!** First, you will need to add a helper method to your user model (`user.rb`) that authenticates the user by email and password:

    ```ruby
    # login by email address
    def self.authenticate(email, password)
      find_by_email(email).try(:authenticate, password)
    end
    ```

11. Now add the following to the ```application_controller.rb``` file/class along with all the other authentication code (**Note:** You will need to replace the the before_action code with the one below). We are using something called Basic Http Authentication which is provided by rails, that authenticates with email and password. This `/token` endpoint will not be authenticated with the api_key, but rather with email and password. As mentioned before, this endpoint will return the user JSON, which contains the api_key. **Once someone enters their email/password and uses this endpoint to retrieve their api_key, they can then use the api_key to authenticate with all the other endpoints.**

    ```ruby
    include ActionController::HttpAuthentication::Basic::ControllerMethods

    before_action :authenticate, except: [:token]

    # A method to handle initial authentication
    def token
      authenticate_username_password || render_unauthorized
    end

    protected

    def authenticate_username_password
      authenticate_or_request_with_http_basic do |email, password|
        user = User.authenticate(email, password)
        if user
          render json: user
        end
      end
    end
    ```

12. After adding this don't forget to add all the token action to the ```routes.rb``` file. (GET request to `/token` will use the `application#token` action) 

13. Now that you have an endpoint in the application controller, you will need to add swagger docs to it as well!

    ```ruby
    swagger_controller :application, "Application Management"

    swagger_api :token do |api|
      summary "Authenticate with email and password to get token"
      param :header, "Authorization", :string, :required, "Email and password in the format of: Basic {Base64.encode64('email:password')}"
    end
    ```

14. **Please understand the following before continuing.** Here's an explanation of the format of how you would interact with the `/token` endpoint, since we will need to do everything the Rails way. Just like with Token auth, the way they intake the email/password is through the ```Authorization``` header and it needs to be in the format of `Basic <Base64.encode64('email:password')>`. Let's say that the email and password for a user is "test@example.com" and "password" respectively (and the encoded email:password is `dGVzdEBleGFtcGxlLmNvbTpwYXNzd29yZA==\n`) then the full header should be `Authorization: Basic dGVzdEBleGFtcGxlLmNvbTpwYXNzd29yZA==\n`


15. After running ```rails swagger:docs``` you can test out the token endpoint to see if you are able to get the user's token with the email and password. (**Note**: as mentioned before the format of the Authorization header is supposed to be ```Basic <encoded_base64_value>``` (without the <>)). You should use irb or rails console to get the Base64 value: 


```ruby
require 'base64'
#put in your user's email and password
Base64.encode64('email:password')
```


# <span class="mega-icon mega-icon-issue-opened"></span>Stop
**Checkpoint 4**
Show a CA that you have all the filtering and ordering params working for all the controllers. 
* * *