# Fat Models? Fat Controllers? Services to the rescue!!!

MVC has become the "de facto" standard for building web (and not only web) application. Enforcing the separation of concerns between presentation and business logic, MVC has become a sort of "common land" that makes easy to follow the application flow in a cross language manner.

## Models, Controllers and Business ...

Why are we talking about Fat Controllers and Models?

Mvc leaves to developers the freedom to choose where to place and elaborate the business logic of the application. When we talk about business logic we refer to a set of classe (library) that actually encapsulate the actions/methods that represent the application identity. If for example we are coding an e-commerce app we will need logic about what products I sell, how and where do I configure these products etc... All this logic should be the core app library that exposes an API to be used through an MVC pattern.
When all or part of this logic is coded into controllers and/or models we talk about fat controllers and models. 

Why is this an issue? Having fat controllers and models says that our business logic is sparsed all around and that when the application grows will be more and more difficult to maintain and refactor our codebase.

At the end we are talking about coding a flexible codebase respecting some well known rule like the **single responsibility principle**. Something that is fat in terms of methods and dependencies will for sure brake this basic rule messing things around.    

Before starting with the code let's make something clear about models and controllers.

* a **MODEL** represents the instance of a defined entity that makes part of the application domain. Any method called on the model instance should be oriented to expose some aspect of the instance itself.

* a **CONTROLLER** is an endpoint. It should only be concerned about receiving a call, giving the application business logic a place to execute what needs to be executed and render the view using the provided data.   

### Example. A Rails basic app.

Let's say we are coding an application that has a model called **Task** with some basic attributes:

	 # Task.rb
	 class Task
	 	include Mongoid::Document
		field :name
		field :description
	 end

If we want to persists a task we normally implement the logic into a method in a TasksController:

	 #tasks_controller.rb
	 class TaskController < ApllicationController
	 
	 	def create 
		 	task = Task.new params[:task]
			if task.save
			 ........
		end
	 
	 end 

This is quite basic and works well even if, from a theorical point of view, adding an **ORM** as mongoid in our example extends the model responsibility over what he should know about the world around. But since **ORM** makes our life much more easy and productive we are happy to accept this as a minor side effects.
 
Now let's go a bit further. A new story in the next sprint says that once a Task has been created we must enqueue a background job to notify some other app and even send a notification to every user that joined our account. We think this is easy enough and we add this logic to our controller:

	 #tasks_controller.rb
	 class TaskController < ApplicationController
	 
	 	def create 
		 	@task = Task.new params[:task]
			if @task.save
				_enqueue_job
				_notify_taks_to_users
			 ........
		end
	 
	 	def _enqueue_job
		   .....	
		end
		
		def _notify_taks_to_users
			.....
		end

	 end 
	 
This 2 new methods should already "smell" cause we are adding business logic to the controller that should not be aware of what happens when a Task is created. Our controller is getting **FAT**. But, at the end, this was easy and we closed the story very fast!! Is friday and a cold beer is waiting for us.

The next sprint complicates a bit our life. We are asked to add a module to the admin area of our application where the users with SUPERADMIN role should be able to manage tasks as well in any application account scope.

We add an /admin/tasks_controller to our application cause we cannot reuse the other controller cause they are base on different acl rules, etc....
 

	 #/admin/tasks_controller.rb
	 class TaskController < ApllicationController
	 
	 	def create 
		 	@task = Task.new params[:task]
			if @task.save
				_enqueue_job
				_notify_taks_to_users
			 ........
		end
	 
	 	def _enqueue_job
		   .....	
		end
		
		def _notify_taks_to_users
			.....
		end

	 end 
	 
 
Here we have an issue. We are duplicating the same business logic in 2 different points. Even if we are lazy coders we see that this is not a safe way. We could create a module to be included but more than a solutions looks more like hiding the problem.
 
Looking at the code we see that the 2 extra methods we added belongs to a particular phase of our application. Any time a task is created we must enqueue a job and send out a notification. So maybe this duties belongs to the Task model itself. Great! We think we are done. We reset the 2 controllers to the basic state:
 
	 #tasks_controller.rb
	 #/admin/tasks_controller.rb_
	 class TaskController < ApllicationController
 
	 	def create 
		 	task = Task.new params[:task]
			if task.save
			 ........
		end
 
	 end 
 
 and we rely on the Task model callback to accomplish our post creation duties:
 
     # Task.rb
     class Task
 	include Mongoid::Document
	field :name
	field :description
	
	after_create :_enqueue_job, :_notify_taks_to_users
	
 	def _enqueue_job
	   .....	
	end
	
	def _notify_taks_to_users
		.....
	end
	
     end
 
After running our test suite ee already note a downside. Our code works but now anytime a Task instance is persisted the 2 callbacks gets fired. Even if we create the instance from a spec factory the callbacks are fired .... We do not like this because it adds unwanted noise to any Task creation but we think we will survive. The controllers are skinny, the model is getting FAT ... but we decide we are still going well.
 
And then comes the pain! We are asked to send a different notification to the user that created the task (actually the current_user) but not if the user is the SUPERADMIN.
  
How we can do that? The business logic is now completely managed by the Model that does not know nothing about the concept of Task creator. We could add this info as a field to the Model but is not something that is not required to be persisted.
 
Btw there is always a solution.....
 
	 # Task.rb
	 class Task
	 	....
		attr_accessor :creator
		after_create :_enqueue_job, :_notify_taks_to_users, :_notify_creator
		.....

		def _notify_taks_to_users
			notify anyone
		end
	
		def _notify_creator
			notify @creator unless @creator.superadmin?
		end
	
	 end

	# in both the tasks controllers
 	def create 
	 	task = Task.new params[:task]
		task.creator = current_user
		if task.save
		 ........
	end
	  
 
We can do this so why is this bad?

* we are adding a not required external dependencies to a Model. From now any time a Task instance is persisted the creator accessor must be provided. Over that the Task model also needs to be aware of the User role ....
* we complicate our test suite factory. We must provide a creators of differents roles.
* we open a way to add more and more dependencies and we will end with a model that plays more and more roles.
 
### Services to the rescue

We could easily solve the problem encapsulating the business logic that stays behind the creation of a Task into his own class. A class that makes part of the application library and is responsible just for creating tasks.
We reset the model to the basic implementation. Then we add a service class:

	  #/app/services/create_task_service.rb
	  class CreateTaskService 
	  	attr_accessor :current_user, :task_params
	  	
		def initialize(args={})
		  @current_user = args.fetch(:current_user, params) do 
		  	raise ArgumentException 'Current user is missing!'
		  end	
		  @task_params = args.fetch(:params, {})
		end
		
		def run! 
		  task = Task.new task_params
		  _after_create(task) if @task.save!
		  task
		end
		
		def _after_create(task)
		  _enqueue_job(task)
		  _notify_taks_to_users(task)
		end
		
	 	def _enqueue_job(task)
		   .....	
		end
	
		def _notify_taks_to_users(task)
			notify anyone
			notify current_user unless current_user.superadmin?
		end
		
	  end

 What about our 2 controllers?
 
	 #tasks_controller.rb
	 #/admin/tasks_controller.rb_
	 class TaskController < ApplicationController

	 	def create 
		    begin
				@task = CreateServiceTask.new(
					current_user: current_user, params: params[:task] 
				).run!
			
			rescue => e
			  #handle 
			end
		end

	 end 
	 
What are the benefits here?

* CreateTaskService is a brick of our app core library. It just does one thing, creates a task, exposing a clear API. It requires a user, the task params and returns a task instance. Being devoted to this particular task is completely acceptable that it has dependencies like a Background job engine or a MailService. 
* Testing a plain ruby class is easy and fast. 
* Testing the business logic in completely isolation from the framework. 
* Approaching new feature is easier because we can really focus on the logic coding and then use it inside of the framework. A similar approach to coding a rest api.
* We slowly build the core application that exposes all the methods we need. When we add new features we will look into this API as when we look for endpoints in a REST api. More services will means more endpoint to rely on.
* We do not mess things up in our spec factories. A task_factory will give us back a Task instance following our attributes configuration. No callbacks. No tricks.

Does implementing a Service layer slow development down?

In my experience no. Once the pattern is clear and respected by any team member it can also increase the team productivity. Is much more easy to code on expectations when you know that your teammate is developing a class that will require some params and will give you back what you expect. You do not need to wait the real code as you do with rest API.
Over that, when the application grows, you will have a set of self documented class that can help anybody to better understand the application logic and find the right method to use. 

#### Notes:

* using the name **services** with a **run** method is just a convention. At the end we are talking about plain classes. The only important thing to keep in mind is that a class is done to perform a task and that you must be able to rely on the methods signature and returns.

Happy coding!!! 
  
 
 
