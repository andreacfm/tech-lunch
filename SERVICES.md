# Fat Models? Fat Controllers? Services to the rescue!!!

MVC has become the "de facto" standard for building web (and not) application. Enforcing the separation of concerns between presentation and business logic, MVC has become a sort of "shared land" that makes easy to follow the application flow in a cross language manner.

## Models, Controllers and Business ...

Mvc leaves the developer the freedom to choose where to place and elaborate the business logic of the application.


FAT Controllers and Models comes from the decisions you make about the business logic implementation.


Before starting with the code let's make something clear about models and controllers.

* a **MODEL** represents the instance of a defined entity that makes part of the application domain. Any method called on the model instance should be something related to the instance itself.

* a **CONTROLLER** is an endpoint. It should only be concerned about receiving a call, giving the application business logic a place to execute what needs to be executed and render the view accordingly to the call.   

### A Rails basic example

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

This is quite basic a works well even if from a theorical point of view we should note that even adding an ORM to a model extends the model over what should be his own duty that is representing an entity state. 
 
But while **ORM** makes our life much more and productive we are happy to accept this as the minor side effects.
 
 Now let's go a bit further. A new story in the next sprint says that once a Task has been created we must enqueue a background job to notify some other app and even send a notification to the every user that joins the platforms. We think this is easy enough and we add this logic to out controller:

	 #tasks_controller.rb
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
	 
This 2 new methods should already alarm us cause we are adding business logic to the controller that should not be aware of this. Our controller is getting FAT. But this was easy and we closed the story very easily and anyone is happy !!! We think we can live with this. Is friday and a cold beer is waiting for us.

The next sprint complicates a bit our life. We are asked to add a module to the admin area of our application where the system SUPER admin may be able to manage tasks as well and he must be able to create a task in any application account.

We add an /admin/tasks_controller to our application cause we cannot reuse the other controller. They are base on different acl rules, etc....
 

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
	 
 
 After we write this we see that we have an issue. We are duplicating the same business logic in 2 differents points. Even if we are lazy we see that this is a bad way and that we are making steps to a not mainainable code.
 
 Looking at the code we see that the 2 estra methods we added belongs to a particular phase of our application. Any time a task is created we must enqueue a job and send out a notification. So maybe this duties belongs to the Task model itself!!! Great we are done. We reset the 2 controllers to its own basic state:
 
	 #tasks_controller.rb
	 #/admin/tasks_controller.rb_
	 class TaskController < ApllicationController
 
	 	def create 
		 	task = Task.new params[:task]
			if task.save
			 ........
		end
 
	 end 
 
 and we rely on a Task model callback to accomplish our post creation duties:
 
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
 
 We already note a downside. Our code works but now anytime a Task instance is persisted the 2 callbacks gets fired. Even if we create the instance from a spec factory the callbacks are fired .... We do not like this cause adds unwanted noise to any Task creation but we think we will survive. The controllers are skinny, the model is getting FAT ... but we decide we are still going well.
 
 And then comes the pain. We are asked to send a notification ot the user that created the task (actually the current_user) but not if the user is the SUPERADMIN.
  
 How we can do that? The business logic is now completely managed by the Model that does not know nothing about teh concept of Task creator. We could add this info as a field to the Model but is not something thet need to be persisted.
 
 Btw there is always a solution.....
 
	 # Task.rb
	 class Task
	 	....
		attr_accessor :creator
		after_create :_enqueue_job, :_notify_taks_to_users
		.....

		def _notify_taks_to_users
			send a copy to @creator unless @creator.superadmin?
		end
	
	 end

 	def create 
	 	task = Task.new params[:task]
		task.creator = current_user
		if task.save
		 ........
	end
	  
 
 .... we can do this. 
 Why is this bad?
 * We are adding not required external dependencies to a Model. From now any time a Task istance is created the creator accessor must be provided. Over that the Task model also needs to be aware of the User role ....
 * We complicate our test suite factory creation
 * ....
 
### Services to the rescue

We could easily solve this problem encapsulating the business logic that stays behind the creation of a Task into his own class.
Reset out model to his own basic impementation. Then we add a service class:

	  #/app/services/create_task_service.rb
	  class CreateTaskService 
	  	attr_accessor :current_user, :task_params
	  	
		def initialize(args={})
		  @current_user = args.fetch(:current_user, params) do 
		  	raise ArgumentException
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
			send a copy to current_user unless current_user.superadmin?
		end
		
	  end

 What about our 2 controllers?
 
	 #tasks_controller.rb
	 #/admin/tasks_controller.rb_
	 class TaskController < ApllicationController

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
 
 
