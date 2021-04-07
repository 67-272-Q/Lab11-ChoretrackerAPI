#Objectives

- Creating the API itself
- Documenting the API with swagger docs
- Serialization Customizations
- Handling CORS

--

# Implementing the ChoreTracker API with Documentation and Serialization

## Part 1 - Building the API

1. We will not be using any starter code for this application since everything will be built from scratch to help you understand the full process. First of all create a new rails application using the api flag and call it ChoreTrackerAPI. The api flag allows rails to know how the application is intended to be used and will make sure to set up the right things in order to make the application RESTful.

  ```
  $ rails new ChoreTrackerAPI --api
  ```

2. Just like the ChoreTracker app that you have built before, there will be 3 main entities to the ChoreTracker application, please review the old lab if you want any clarifications on the ERD. The following are the data dictionaries for the 3 models. Based on these specifications, please generate all the models with all the proper fields and then run ```rails db:migrate```. This step should be the same as if you were building a regular rails application. (ex. `rails generate model Child first_name:string last_name:string active:boolean`)
    - Child
        - first_name (string)
        - last_name (string)
        - active (boolean)
    - Task
        - name (string)
        - points (integer)
        - active (boolean)
    - Chore
        - child (references)
        - task (references)
        - due_on (date)
        - completed (boolean)

3. Now add in the model code below for each of the models. The code here is exactly the same as what you have done in the old ChoreTracker Application, which is why we won't require you to rewrite everything! (**Note: don't forget to add the validates_timeliness gem to the Gemfile, run** `bundle install` **and the run the** `rails generate validates_timeliness:install` **command in order to get some of the validations to work.**)

  ```ruby
    class Child < ApplicationRecord
      has_many :chores
      has_many :tasks, through: :chores
    
      validates_presence_of :first_name, :last_name
    
      scope :alphabetical, -> { order(:last_name, :first_name) }
      scope :active, -> {where(active: true)}
    
      def name
        return first_name + " " + last_name
      end
    
      def points_earned
        self.chores.done.inject(0){|sum,chore| sum += chore.task.points}
      end 
    end
  ```
  &nbsp;
  ```ruby
    class Task < ApplicationRecord
      has_many :chores
      has_many :children, through: :chores
    
      validates_presence_of :name
      validates_numericality_of :points, only_integer: true, greater_than_or_equal_to: 0
    
      scope :alphabetical, -> { order(:name) }
      scope :active, -> {where(active: true)}
    end
  ```
  &nbsp;
  ```ruby
    class Chore < ApplicationRecord
      belongs_to :child
      belongs_to :task
    
      # Validations
      validates_date :due_on
      
      # Scopes
      scope :by_task, -> { joins(:task).order('tasks.name') }
      scope :chronological, -> { order('due_on') }
      scope :done, -> { where('completed = ?', true) }
      scope :pending, -> { where('completed = ?', false) }
      scope :upcoming, -> { where('due_on >= ?', Date.today) }
      scope :past, -> { where('due_on < ?', Date.today) }
      
      # Other methods
      def status
        self.completed ? "Completed" : "Pending"
      end
    end
  ```

4. Now we will be starting to build out the controllers for the models that we just made. First, let's create a file called `children_controller.rb` in the controllers folder (you can also run `rails generate controller Children`), define the class `class ChildrenController < ApplicationController ... end` and follow along below!

5. As you remember, we will not be building any views since literally all user output from a RESTful API is just JSON (no need for HTML/CSS/JS). First, let's go through the process of creating the controller for the Child model and then you will need to **create the controllers for the other 2 models**. So unlike in a normal Rails application, in a RESTful one, you will only need 5 (**index, show, create, update, and destroy**) actions instead of 7. We won't be needing the new or edit action since those were only used to display the form, and with only JSON responses, the form will no longer be needed.  (Note: One thing to note here is the idea of the status code. This is especially important when developing a RESTful API to tell users of it what happened. All success type codes (ok, created, etc.) are in the 200 number ranges, and generally other error statuses are either in the 400 or 500 ranges.)

6. Index Action (responds to GET) is used to display all of the children that exist and its information/fields. So in this case all you need is to render all of the children objects as json.
  
7. Show Action (responds to GET) just like before, given a child id from the url path, it will display the information for just that child. This uses the `set_child` method to set the instance variable @child before rendering it.
  
8. Create Action (responds to a POST) actually creates a new child given the proper params. Using the ```child_params``` method it gets all the whitelisted params and tries to create a new child. If it properly saves, it will just render the JSON of the child that was just created and attached with a created success status code. If it fails to save, then it will respond with a JSON of all the validation errors and a unprocessably_entity error status code. You might have also noticed this new thing called `location`, this is a param in the header so that the client will be able to know where this newly created child is (in this case it's the child show page).
  
9. Update Action (responds to PATCH) updates the information of a child given its ID. The @child variable will be set from the `set_child` method and then be populated with the child parameters. Again it will do something similar to create where it checks if the child is valid and return the proper JSON response. 
  
10. Delete Action (responds to DELETE) deletes the child given its ID which is set from the `set_child` method. 
  
11. Lastly **don't forget to add the proper routes to the routes.rb file. `resources :children` should take care of all the routes for your children controller.**

  ```ruby
    class ChildrenController < ApplicationController
      # Controller Code
    
      before_action :set_child, only: [:show, :update, :destroy]
    
      # GET /children
      def index
        @children = Child.all
    
        render json: @children
      end
    
      # GET /children/1
      def show
        render json: @child
      end
    
      # POST /children
      def create
        @child = Child.new(child_params)
    
        if @child.save
          render json: @child, status: :created, location: @child
        else
          render json: @child.errors, status: :unprocessable_entity
        end
      end
    
      # PATCH/PUT /children/1
      def update
        if @child.update(child_params)
          render json: @child
        else
          render json: @child.errors, status: :unprocessable_entity
        end
      end
    
      # DELETE /children/1
      def destroy
        @child.destroy
      end
    
      private
        # Use callbacks to share common setup or constraints between actions.
        def set_child
          @child = Child.find(params[:id])
        end
    
        # Only allow a trusted parameter "white list" through.
        def child_params
          params.permit(:first_name, :last_name, :active)
        end
    end
  ```
    
12. Now we want to test that our API actually works. Whenever we mention the word **endpoint**, it is just another way to say action of your controller since each action is an endpoint of your API that you can hit with a GET or POST request.
    
13. Go to `http://localhost:3000/children` and an empty array should appear. This triggers the index action with the GET request and display no children, since none have been created yet.
    
14. Now we should test how creating a new child. Since we can't easily send POST requests in the browser (not as easy as GET) we will be needing CURL. CURL is a command that you can run in your terminal to hit certain endpoints with GET, POST, etc. requests.
    - Check that ```curl -X GET -H "Accept: application/json" "http://localhost:3000/children"``` from your terminal will return an empty list just like it did in the browser.
    - Now run ```curl -X POST --data "first_name=Test&last_name=Child&active=true" -H "Accept: application/json" "http://localhost:3000/children"``` from your terminal which should return a success status.
    - Check that it has been created by either CURLing the index action or going to the url on chrome.
    - Feel free to test out all the other endpoints if you have time!

15. Now that you already created the Children Controller, **you will need to follow the similar structure and create the controller for the Tasks and Chores Controllers**.  You can add some sample data to the system with the help of the [files found here](https://github.com/profh/ChoreTrackerAPI_populator). **Be sure to read the README for instructions on how to do this quickly.**

  The controllers for Tasks and Chores and pretty identical to Children, except for fields. All of the methods are the same in structure.


# <span class="mega-icon mega-icon-issue-opened"></span>Stop

Show a TA that you have completed the first part (including constructing the API for all three models). Make sure the TA initials your sheet.

* * *



## Part 2 - Documenting the API using Swagger Docs/UI

Now that you have created the API you will need to document it. Documentation is **crucial** for RESTful API's since there are no views tied to the application, which means there is no way for users to know what endpoints exist and what capabilities the API has. Good thing that there is an easy to way autogenerate some nice Documentation using Swagger for your API.

1. There are 2 main portions of swagger documentation. There is the Swagger Doc and Swagger UI. Swagger Doc is a representation that is autogenerated to describe your API and each endpoint (which is in JSON format) and Swagger UI is the HTML/CSS/JS that is autogenerated from the Swagger Doc.

1. First we need to set up swagger docs for the RESTful API application. **Include the swagger docs gem in your Gemfile** `gem 'swagger-docs'` **and then run** `bundle install`
  
  This gem will automatically generate the right JSON files to help document your API if provided the right things. The documentation for the gem is here: [https://github.com/richhollis/swagger-docs](https://github.com/richhollis/swagger-docs)

3. Next you will need to create an initializer file for the gem (in `/config/initializers`) and call it `swagger_docs.rb`. This will tell the gem some basic information about your application and how to generate the JSON for your API. The following code should go in this `config/initializers/swagger_docs.rb` file that you just created. You won't need to fully understand what this whole thing does, but its good to know that all of the autogenerated Swagger Doc files are put in the `public/apidocs` folder. 

  ```ruby
  # config/initializers/swagger_docs.rb

  class Swagger::Docs::Config
    def self.transform_path(path, api_version)
      # Make a distinction between the APIs and API documentation paths.
      "apidocs/#{path}"
    end
  end

  Swagger::Docs::Config.base_api_controller = ActionController::API 

  Swagger::Docs::Config.register_apis({
    "1.0" => {
      # the extension used for the API
      :api_extension_type => :json,
      # the output location where your .json files are written to
      :api_file_path => "public/apidocs",
      # the URL base path to your API (make sure to change this if you are not using localhost:3000)
      :base_path => "http://localhost:3000",
      # if you want to delete all .json files at each generation
      :clean_directory => false,
      # add custom attributes to api-docs
      :attributes => {
        :info => {
          "title" => "Chore Tracker API",
          "description" => "Uses swagger ui and docs to document the ChoreTracker API"
        }
      }
    }
  })
  ```

4. Now that you have set up your swagger docs, you are ready to actually document your API. Again we are only going through documenting your Children Controller and you will need to do the rest without guidance. First, go to your `children_controller.rb` file since all the documentation that you need to add should be in that file. This is because you are only documenting each endpoint and each endpoint is only defined in the controller itself. Step by step add in documentation **within the ChildrenController class** and above the controller code (above the before_action) by following the instructions below:

5. Tell the swagger-docs gem that the children controller is an API and give it a name:

  ```
  swagger_controller :children, "Children Management"
  ```

6. Document the index action by saying what it does:

  ```
  swagger_api :index do
    summary "Fetches all Children"
    notes "This lists all the children"
  end
  ```

7. Document the show action. This is a bit more complicated as it requires some parameters, namely the child id in the url path. Params are defined by type (path, form, header), the name of the parameter, the type of the parameter, whether or not it is required, and the description. You can probably also see that it has 2 different responses, and this is describing what type of error response statuses can be returned from this endpoint. In this case, it will return not_found if the child id is invalid.

  ```
  swagger_api :show do
    summary "Shows one Child"
    param :path, :id, :integer, :required, "Child ID"
    notes "This lists details of one child"
    response :not_found
  end
  ```

8. Next we need to document the create action. Just like with the show action, we need params for the `create` but this time we will also need form params to pass through describing the fields of the child including the first_name, last_name, and active. This time we won't need the not_found response, but rather the not_acceptable response if there is anything wrong with the actual creation of the child (most likely some validation error).
  ```
  swagger_api :create do
    summary "Creates a new Child"
    param :form, :first_name, :string, :required, "First name"
    param :form, :last_name, :string, :required, "Last name"
    param :form, :active, :boolean, :required, "Active"
    response :not_acceptable
  end
  ```

9. The update action is like the combination of the show and create where we need the path param of the child id and the form params to describe the child. However, for updates, none of the fields should be required as users should be able to update only the fields that they want.
  ```
  swagger_api :update do
    summary "Updates an existing Child"
    param :path, :id, :integer, :required, "Child Id"
    param :form, :first_name, :string, :optional, "First name"
    param :form, :last_name, :string, :optional, "Last name"
    param :form, :active, :boolean, :optional, "Active"
    response :not_found
    response :not_acceptable
  end
  ```

10. Lastly is the delete action and this is rather simple since it only needs one param.
  ```
  swagger_api :destroy do
    summary "Deletes an existing Child"
    param :path, :id, :integer, :required, "Child Id"
    response :not_found
   end
  ```

11. Now that you have written up the documentation for the rails API, you can generate the Swagger Docs by running the following command. You should verify that it was properly created by checking the ```public/apidocs/``` folder and seeing if it contains 2 files (api-docs.json and children.json). api-docs.json contains the general info about your API and each controller should have one json file for it. (**Note:** You might encounter something like this when you run the following: ```2 process / 3 skipped``` Just because it says skipped doesn't mean something went wrong! You can just keep on going.)

  ```
  $ rails swagger:docs
  ```

12. After you generate the Swagger Docs, your next step is to get swagger ui to display the JSON in a user-friendly manner using Swagger UI! Change your directory to the public folder:

  ```
  $ cd public/
  ```

13. Then include the Swagger UI in the public folder as a git submodule under the folder name api/. You should now have a folder under public/api/ where all of swagger ui files (html, css, javascript) will be.

  ```
  $ git submodule add https://github.com/cmu-is-projects/RailsSwaggerUI api
  ```

14. Start up your server and then go to `http://localhost:3000/api` and you should see something like this:

  ![](https://github.com/495-Labs-Projects/ChoreTrackerAPI/raw/master/public/swagger_screenshot.png "Swagger UI Screenshot")

15. Play around with the swagger docs and try to view, create, edit, and delete different children using the Swagger Docs/UI. This documents and makes interactions with your API endpoints much more easier and you won't need to use curl to hit an endpoint.

16. Now that you created this for the children_controller, create documentation for both the **tasks_controller** and **chores_controller**, by adding in similar documentation code in the file itself. Remember to run ```rails swagger:docs``` and restart the server every time you make a change to your documentation. For additional swagger docs param types, please refer to this link: https://swagger.io/docs/specification/data-models/data-types/#numbers (Note: Create a couple of Children, Tasks, and Chores to help test out things in the next part.)

# <span class="mega-icon mega-icon-issue-opened"></span> Stop
Show a TA that you have properly created the documentation for the ChoreTrackerAPI!

* * *



## Part 3 - Custom Serialization

1. Once you have created the barebone API for ChoreTracker and documenting it, there are a lot more things you can do to improve it and make it more usable. One main thing is serialization, which is how Rails converts a Child/Task/Chore model object to JSON. With serializers, you can truly customize how you want these objects to show up in your API. One good example of this is to display all the chores that are tied to a child when viewing the show action of a child. While we could use active_models_serializers, for this lab we are going to use fast_jsonapi, which is much faster than active_models_serializers. First of all, add the gem to your Gemfile: `gem 'fast_jsonapi'` and run `bundle install`.

2. Now you can actually generate some boilerplate code for your serializer, by running `rails g serializer <model_name>` so for example, `rails g serializer child` will create a new file called `child_serializer.rb` in the serializers folder in app. **Generate serializer files for each controller** (child, chore, and task).

3. By default for the child serializer you should just see the following. This means that when serializing a child object to JSON it will only display the attributes listed after "attributes". 
    
  ```ruby
  class ChildSerializer
    include FastJsonapi::ObjectSerializer
    attributes 
  end
  ```

4. Let's start off by adding what we want to the ChildSerializer. In this case we want to display the id, name of the child, whether or not it is active, and the list of chores that it has. To do this, after the `:id`, also add `:name` and `:active`. The reason that name works even though the Child Model doesn't have a name attribute (only first_name and last_name) is that we had already defined a method call name in the Child Model that combines the first and last names. Next we need to get all the chores that is related to this child. To do so, we need to add a relationship, just like with the model, by writing `has_many :chores`. Your ChildSerializer should look something like the following. Now, look at the output on Swagger Docs and try to view all children.
    
  ```ruby
  class ChildSerializer
    include FastJsonapi::ObjectSerializer
    attributes :id, :name, :active
    has_many :chores
  end
  ``` 

5. You have probably noticed that Swagger Docs is showing much more information than what we specified in the serializer. For example, when we try to get the information of one child, we see additional attributes such as created_on. The reason for this is that we are not actually calling the serializers on our objects. Go to the children_controller and change the show action so that it looks like following. Make this similar change in all other children_controller actions (index, create, update) that render JSON objects. Check Swagger Docs to verify that it worked.

  ```ruby
  def show
    render json: ChildSerializer.new(@child)
  end 
  ```

6. Let's go onto fixing the TaskSerializer. For this follow the same idea, but we only need to display the **id**, **name**, **points** that its worth, and whether or not it is **active**.

7. After that, you should go on to adding serialization to the ChoreSerializer, which should include the **id**, **child_id**, **task_id**, and **due_on**.

  We also want to add **Completed** to our serializer. We want this to show as "completed" or "pending" (not just true or false). For implementing `:completed` to be not a geeky True/False in the API, we can add the following method to our serializer that takes advantage of our model method:

  ```ruby
  attribute :completed do |object|
    object.status
  end
  ```

8. Verify that your serializers worked properly by using the SwaggerDocs. Make sure that you restart your server before doing so.

9. At this point, you have just very standard serialization for each of these models. Let's make ChildSerializer more interesting! It would probably be useful to include the total number of points that the child has earned (good thing we wrote this function already in the model). Include that as an attribute of the ChildSerializer. 

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

12. Now go to your chore_serializer and instead of displaying task_id, have it display `:task` by writing a custom serialization method called task. In this method, all you need to return is the preview version of the serialized task which includes the `:id` and `name` of the task. Call over a TA if you are having trouble with this concept! Now test if it worked by going to the /children endpoint! It should display the task id and name instead of just the task id.

  ```ruby
  attribute :task do |object|
    ChoreTaskSerializer.new(object.task)
  end
  ```


# <span class="mega-icon mega-icon-issue-opened"></span> Stop
Show a TA that you have properly serialized JSON objects in the ChoreTrackerAPI!
* * *



## Part 4 - CORS

CORS stands for Cross Origin Resource Sharing. Most web applications have CORS disabled, which means that the web app prevents JavaScript from making requests that are outside of the domain. For example, let's say that the web application is hosted on ```cmuis.net```, if CORS is disabled then Javascript code located on another domain (```testdomain.com```) can't make a request to cmuis.net. This is meant to protect malicious Javascript from making requests to your web application.

However, for the purposes of an API, we want CORS to be enabled since we would like code from other domains to access our API. The only reason that the swagger docs works in hitting the API's endpoints is that the swagger docs is located in the same domain. To demonstrate that CORS isn't enabled right now, please create another HTML file and copy paste the code below:

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

After creating this new simple HTML page (let's call it ```cors_test.html```), just open it up. After it tries to make an AJAX request to the ```/children``` endpoint, it should alert out "Error" and if you look in the Console (Right click -> Inspect Element -> Console) there should be an error message saying "No 'Access-Control-Allow-Origin' header is present". This basically means that the API located at ```localhost:3000/children``` doesn't allow for CORS access.

1. In order to fix this for your Chore Tracker API, you will need this new gem called rack-cors (Read more about it here: https://github.com/cyu/rack-cors). Add `gem 'rack-cors'` to your Gemfile and bundle install.

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
Show a TA that your API now allows for Cross Origin Requests!
* * *

