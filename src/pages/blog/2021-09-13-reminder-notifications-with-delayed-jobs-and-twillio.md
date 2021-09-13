---
templateKey: blog-post
title: Reminder notifications with delayed jobs and Twillio
date: 2021-09-13T12:58:44.694Z
description: >-
  When working with delayed jobs you might find yourself in a situation where
  you need to find a specific job in order to update it.

  This guide will walk you through an example of just that, where I use polymorphic associations to add record type and ID to the delayed jobs.
featuredpost: false
featuredimage: /img/blake-connally-435076-unsplash.jpg
tags:
  - rails
---
Delayed jobs is a well-known way to integrate background processes in rails.
The idea is to add some tasks or method calls to a queue.  Then pass a datetime to the run_at method, so the background process can be executed at a selected time.

The plain and simple integration of delayed job is pretty straight forward.
It does get a little more complicated when you need to add custom attributes to the different jobs.  It could be if you wanted to execute the job at a certain time, depending on a timestamp in a record in the database.
An example could be a reminder.  You want to send a reminder to the user 2 hours before the event starts.  You are able to do this by adding the run_at time on creation of the job.  But what are we going to do if the event is update and the timestamp is changed?  We need to be able to find the job in the jobqueue and update it accordingly.  One way to do this could be adding polymorphic association to the delayed jobs.


### Best explained with an example;

In this example I am using an event model and the [Twilio](https://www.twilio.com/ "Twilio") service for text messages.

First off we need to add the delayed_jobs gem to our rails app.
I'm using active record orm in this example, so we ill go with that - [delayed_job_active_record](https://github.com/collectiveidea/delayed_job_active_record "delayed_job_active_record")

Gemfile
```ruby
gem 'delayed_job_active_record', '~> 4.1', '>= 4.1.4' 
````


and run the generator: 

```ruby
rails generate delayed_job:active_record
````

This will give you the following migration:

````ruby
class CreateDelayedJobs < ActiveRecord::Migration[5.2]
 def self.up
   create_table :delayed_jobs, force: true do |table|
     table.integer :priority, default: 0, null: false # Allows some jobs to jump to the front of the queue
     table.integer :attempts, default: 0, null: false # Provides for retries, but still fail eventually.
     table.text :handler,                 null: false # YAML-encoded string of the object that will do work
     table.text :last_error                           # reason for last failure (See Note below)
     table.datetime :run_at                           # When to run. Could be Time.zone.now for immediately, or sometime in the future.
     table.datetime :locked_at                        # Set when a client is working on this object
     table.datetime :failed_at                        # Set when all retries have failed (actually, by default, the record is deleted instead)
     table.string :locked_by                          # Who is working on this object (if locked)
     table.string :queue                              # The name of the queue this job is in
     table.timestamps null: true
   end
 
   add_index :delayed_jobs, [:priority, :run_at], name: "delayed_jobs_priority"
 end
 
 def self.down
   drop_table :delayed_jobs
 end
end
 
````

For now we will not add any extra columns to this migration, because I want to show the simple solution first, without the polymorphic association.

Lets start out with the simple implementation;

The generator created the migration above, so remember to do the 
````ruby
rails db:migrate
````

So now we need to make the method that will create the job.  Let's call it reminder.
In the events model, we can add this callback.

````ruby
after_save :reminder
````

and then the reminder method.

````ruby
 def reminder
   # Send a text message reminder 2 hours before event start
   @client = Twilio::REST::Client.new(Rails.application.credentials.twilio_account_sid, Rails.application.credentials.twilio_auth_token)
          
   @client.messages.create({
     from: "Name",
     to: "phone number",
     body: "Message to the user"
     })
 end
````

What happens now is that after a new event is created, the reminder method will run.
The reminder method contains dummy code, but the fields should be populated with relevant data..

Now to the important part, right now this text will be sent as soon as the event is created.
delayed_jobs gives us this method that we can add after the method:

````ruby
handle_asynchronously :reminder, :run_at => Proc.new { |i| i.when_to_run }
````

This handles the reminder method asynchronously and a proc is created to determine the value of the :run_at parameter.
I created a when_to_run method which is passed to the proc, where a value is returned based on the events starts_at value.

````ruby
def when_to_run
   minutes_before_appointment = 120.minutes
   self.event.starts_at - minutes_before_appointment
 end
````

And that is actually it.
The reminder method is now run 2 hours before the starts_at for the event and a text message is sent.

The issue we face now is if the events starts_at datetime is changed, then we have no way of finding the job and update the :run_at value.
To solve this issue, we will have to add a few more columns to the delayed_jobs table.

### Extended implementation
Now that we understands the basic implementation we can consider if we need the option to update the delayed_jobs or not.
If so, we will have to add extra columns to the delayed_jobs table.

I suggest using polymorphic association, in order to provide the option to filter the different types of delayed_jobs.  We will also be  able to find and update specific jobs.

So lets start out with the migration:

````ruby
rails generate migration add_delayable_to_delayed_jobs delayable:references{polymorphic}
````

A new migration is created

````ruby
class AddDelayableAssociationToDelayedJobs < ActiveRecord::Migration[5.2]
 def change
   add_reference :delayed_jobs, :delayable, polymorphic: true
 end
end
`````

run the migration
````ruby
rails db:migrate
`````

We now have the option to populate the 2 fields that references the type and id of a record.
Since the delayed_jobs.rb is a headless model, we will have to add a delayed_job.rb file to our initializers.

````
touch config/initializers/delayed_job.rb
````

While we are at it, create a new folder called jobs under the app dir.
````
mkdir app/jobs
````
This is where we will store the different jobs.

Open up the delayed_job.rb file and add the following:

````ruby
class Delayed::Job < ActiveRecord::Base
 belongs_to :delayable, :polymorphic => true
end
````


Lets start by creating a reminder job in the jobs folder.
````
touch app/jobs/reminder_job.rb
````

Inside this file we have a few have to define a few methods:

````ruby
ReminderJob = Struct.new(:event) do
 
 def enqueue(job)
   job.delayable = event
   job.save!
 end
 
 def perform
   event.send_reminder
 end
 
 def queue_name
   'reminder_queue'
 end
end
 
`````

We are creating a new Struct, where we take event as an argument.
The event has a starts_at datetime that we will use to determine when to send the reminder.
The enqueue method queues the job and the perform method executes the job.
This will get clearer in a moment.
I also added a queue_name for scalability, it is good practice to have more than 1 queue if you plan on extending this feature with other delayed jobs.

Let's revisit our reminder method.
Open up the model file and add a method called new_reminder and also rename the callback to new_reminder.

````ruby
after_save :new_reminder
 
 def new_reminder
   Delayed::Job.enqueue(ReminderJob.new(self), { run_at: self.when_to_run })
 end
 
 def send_reminder
   # Send a text message reminder 2 hours before event start
   @client = Twilio::REST::Client.new(Rails.application.credentials.twilio_account_sid, Rails.application.credentials.twilio_auth_token)
          
   @client.messages.create({
     from: "Name",
     to: "phone number",
     body: "Message to the user"
     })
 end
 
 def when_to_run
   minutes_before_appointment = 120.minutes
   self.starts_at - minutes_before_appointment
 end
`````

What is happening now is that upon create, we use the after_save callback to run the new_reminder method.
This method will queue up a delayed_job where it creates a new ReminderJob.  The run_at is again populated with the value from the when_to_run method.

The new reminder job queues the job with the association to the event and then runs the perform method at the selected time, which comes from the run_at method.

And that's it.

A few things to keep in mind.
If an event is updated, you will have to fetch the job and update it with a new run_at value.
You can do this, since each job will contain the id and type as references.

An option would be to add a after_update callback, where you check if the starts_at is changed.  If it is, then update the run_at value for the delayed job and save it.

````ruby
def update_reminder
  if.self.starts_at_changed?
    job = Delayed::Job.find_by(delayable_id: event.id)
    job.run_at = @event.starts_at - 2.hours
    job.save
  end
end
`````



