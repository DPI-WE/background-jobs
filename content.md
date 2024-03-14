# Background Jobs 🦸

## Overview
In modern web applications, particularly those relying on significant data processing or scripts that need to run independently of user interactions, background jobs are a vital component. Ruby on Rails, through [Active Job](https://edgeguides.rubyonrails.org/active_job_basics.html), provides a powerful framework for queueing and executing these jobs. This lesson will cover the basics of setting up and using background jobs in a Rails application, utilizing Active Job along with popular queueing backends and dashboards for job management.

## Why Background Jobs?
Imagine a scenario in your application where you need to send out a newsletter to thousands of users, process large data files, or perform time-consuming data analysis. Running these processes synchronously during a web request can lead to poor user experiences, including slow page loads or timeouts. Background jobs allow you to offload these heavy tasks to run asynchronously, improving application responsiveness and scalability.

## Getting Started with Active Job
Active Job is a framework for declaring jobs and making them run on a variety of queueing backends. It provides a standard interface for interacting with different job runners such as Sidekiq, Resque, Delayed Job, Solid Queue and more.

To understand Active Job basics and its integration in a Rails application, it's recommended to start with the [Active Job Basics guide](https://edgeguides.rubyonrails.org/active_job_basics.html).

## Key Concepts:
- **Job**: A class that performs a specific task.
- **Queue**: A list where jobs are saved to be performed.
- **Worker**: A process that picks up jobs from the queue and executes them.

<!-- TODO: aside on `perfom` vs `perform_later` and `deliver_now` vs `deliver_later` -->

## Example: Processing Photos Asynchronously
Imagine we have a social media platform where users can upload photos. Once a photo is uploaded, we want to perform several operations such as generating thumbnails, applying filters, and maybe even running some image recognition to tag the content automatically. These tasks are perfect for background jobs since they can be time-consuming and we don't want to block the main thread, ensuring a smooth user experience.

For this example, we'll use a Photo model and a background job to process the uploaded images.

### Model Setup
First, let's define a class method in the Photo model that encapsulates our image processing tasks:

```ruby
# app/models/photo.rb
class Photo < ApplicationRecord
  # Assuming this model has an image attribute

  def self.process_image(photo_id)
    photo = Photo.find(photo_id)
    
    # Imagine these are methods that handle image processing
    photo.generate_thumbnails
    photo.apply_filters
    photo.run_image_recognition
    
    photo.update(processed: true) # Mark the photo as processed
  end
  
  # Placeholder for the actual image processing methods
  def generate_thumbnails
    # TODO
  end
  def apply_filters
    # TODO
  end
  def run_image_recognition
    # TODO
  end
end
```

### Background Job
Now, we create a background job that calls this class method. This job will be enqueued when a photo is uploaded.

```ruby
# app/jobs/photo_processing_job.rb
class PhotoProcessingJob < ApplicationJob
  queue_as :default

  def perform(photo_id)
    Photo.process_image(photo_id)
  end
end
```

### Enqueuing

#### Option A: From a Controller
When a photo is uploaded through a form, you typically handle the file upload in a controller action. After saving the new photo record, you can enqueue the job directly from the controller.

```ruby
class PhotosController < ApplicationController
  def create
    @photo = Photo.new(photo_params)
    if @photo.save
      # Enqueue the job after the photo is successfully saved
      PhotoProcessingJob.perform_later(@photo.id)
      flash[:success] = "Photo uploaded successfully!"
      redirect_to photos_path
    else
      render :new
    end
  end
end
```

#### Option B: From a Model Callback
Enqueuing from a Model Callback
Alternatively, you can automate this process by using a model callback within the Photo model. This way, every time a photo is saved to the database, the background job is automatically enqueued without needing explicit controller logic.

```ruby
class Photo < ApplicationRecord
  after_commit :process_photo, on: :create

  private

  def process_photo
    PhotoProcessingJob.perform_later(self.id)
  end
end
```

<aside>
## `after_commit` vs. `after_save`

- **after_commit**: This callback is triggered after a record has been created, updated, or destroyed and the database transaction is successfully committed. This is usually the safest place to enqueue a job because it ensures that the job isn't enqueued unless the database changes are truly *saved*, avoiding potential issues where the job runs before the transaction is completed.

- **after_save**: This callback is called after a record is saved. It can be useful but comes with the caveat that if the database operation is wrapped in a larger transaction that fails, the record might not actually be persisted even though the job has been enqueued.
</aside>

Each approach has its appropriate use case. The controller method gives you more control and is more explicit, which can be useful for operations that should only happen under certain conditions. The model callback method is more automatic and ensures the job is always enqueued whenever a photo is created, making it a cleaner solution for operations that should always follow the creation of a record.

## Choosing a Queueing Backend for Monitoring and Managing Jobs
While Active Job abstracts away many of the specifics of interacting with a queueing system, choosing the right backend is critical based on your application's requirements. Factors to consider include performance, persistence, scalability, dependencies (like Redis for in-memory queues), Database based vs In-memory based and monitoring tools.

- [Sidekiq](https://github.com/sidekiq/sidekiq)
- [Resque](https://github.com/resque/resque)
- [Delayed Job](https://github.com/collectiveidea/delayed_job)
- [Good Job](https://github.com/bensheldon/good_job)
- [Solid Queue](https://github.com/basecamp/solid_queue) (w/ [Mission Control Jobs](https://github.com/basecamp/mission_control-jobs)) <-- RECOMMENDED

<!-- TODO: compare db based vs in-memory based -->

## Integrating Background Jobs in Your Rails Application
To integrate background jobs into your Rails application effectively, consider the following best practices:

- **Idempotency**: Ensure your jobs can safely run multiple times. This is crucial for handling retries gracefully.
- **Error Handling**: Implement comprehensive error handling and retry mechanisms. Most queueing backends offer retry capabilities.
- **Job Data**: Be mindful of the data you pass to your jobs. Large objects can increase memory usage and slow down your job system. Consider passing database IDs instead of Ruby objects.

## Conclusion
Background jobs are a powerful way to improve the performance and scalability of your Rails applications. By understanding and implementing Active Job with an appropriate queueing backend and monitoring tools, you can handle long-running tasks efficiently, providing a better experience for your users. Happy coding!

## Further Resources
- [Active Job Basics](https://edgeguides.rubyonrails.org/active_job_basics.html)
- [Whenever](https://github.com/javan/whenever)
- [Crontab.guru](https://crontab.guru/)
