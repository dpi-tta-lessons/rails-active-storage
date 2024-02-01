# Image Uploads with Ruby on Rails, Active Storage, and Amazon AWS S3

This guide focuses on setting up image uploads in a Ruby on Rails application using Active Storage and Amazon AWS S3. Before diving into the setup process, it's important to understand the concept of BLOBs (binary large objects) and their storage requirements.

## Introduction to BLOBs

BLOBs, or Binary Large Objects, refer to large data items such as images, audio files, or any type of multimedia files that are typically stored in binary format. Unlike traditional text-based data, which is easily stored and managed in standard database fields, BLOBs require special handling due to their size and binary nature.

### Why Store BLOBs Differently:

1. **Size and Scalability**: BLOBs can be very large, often exceeding the size limits of standard database fields. Storing them directly in a database can rapidly inflate the size of the database and degrade performance.
   
2. **Efficiency**: Databases are optimized for operations on structured, textual data. Binary data storage and retrieval operations can be less efficient when handled directly by the database.
   
3. **Cost and Complexity**: Large BLOBs can contribute to increased costs for database storage and backup. Separating BLOB storage from the database can reduce costs and simplify data management.

4. **Serving Media**: Serving large files directly from a database can be inefficient. Using dedicated services like AWS S3 for storing and serving BLOBs can provide better performance and scalability.

With these considerations in mind, let's explore how to implement a robust and scalable solution for handling BLOBs in Ruby on Rails using Active Storage and Amazon AWS S3.

## Part 1: Setting Up Active Storage with Amazon S3

Install Active Storage

```bash
rails active_storage:install
rails db:migrate
```

<aside>
Active Storage uses 3 tables.

`active_storage_attachments`: Connects blobs to your models
`active_storage_blobs`: Stores data about uploaded files in the cloud, such as filename and content type.
`active_storage_variant_records`: tracks different variants (if enabled)
</aside>


In this we'll use AWS, however, you could use any cloud storage service you like. 

Let's add the [AWS SDK Gem](https://rubygems.org/gems/aws-sdk-s3/)
Add this line to your Gemfile:

```ruby
gem 'aws-sdk-s3', require: false
```

Then, run `bundle install`.

Setting Up an AWS S3 Bucket

1. [Sign up for an AWS account](https://aws.amazon.com/).

2. Create an S3 Bucket
In the AWS Management Console, go to the [S3 service](https://s3.console.aws.amazon.com/s3) and create a new bucket.

You can keep the default recommended settings. The 2 settings you'll need to set are name (eg `my-app-name-development`) and region (eg `us-east-2`).

I recommend naming the bucket the same as your app with the env at the end. This way you can easily identify production, development, and test environments. Later on we'll be able to set the bucket name dynamically in our code like this `your-app-name-<%= Rails.env %>`.

3. Configure IAM for Security

IAM is a security tool to configure who has access to your resources. In the [IAM dashboard](https://us-east-1.console.aws.amazon.com/iam/), [create a new user](https://us-east-1.console.aws.amazon.com/iam/home#/users/create) with username (eg `my-app-name-development`) and programmatic access. Attach the “AmazonS3FullAccess” policy or a custom policy. 

We'll need to create a access key ID and secret access key for this IAM user so your application can access the bucket as this user. On the users page click "create access key" and take a note of the "access key" and "secret access key". You'll need to do this step again if you lose the "secret access key".

<aside>
It is considered an AWS best practice to grant permissions to a "user group" and then add users to the group. This is especially helpful when managing permissions for many users at larger organizations.
</aside>

<aside>
If your application needs to access the S3 bucket from a different domain, you'll need to set up CORS.

In the S3 dashboard, click on your bucket.

Go to the “Permissions” tab.

Scroll down to “Cross-origin resource sharing (CORS).”

Click “Edit” and add a CORS policy. Example for a general setup:

```json
[
  {
    "AllowedHeaders": ["*"],
    "AllowedMethods": ["GET", "POST", "PUT", "DELETE", "HEAD"],
    "AllowedOrigins": ["*"],
    "ExposeHeaders": []
  }
]
```
Click “Save changes.”
</aside>

5. Integrating S3 with Rails
Add the S3 credentials to your Rails credentials file:

```bash
EDITOR="code --wait" rails credentials:edit
```

```yaml
aws:
  access_key_id: [YOUR_ACCESS_KEY_ID]
  secret_access_key: [YOUR_SECRET_ACCESS_KEY]
  region: [YOUR_BUCKET_REGION]
```

Configure S3 Service
In config/storage.yml, set up the AWS S3 service to use your credentials:

```yaml
amazon:
  service: S3
  access_key_id: <%= Rails.application.credentials.dig(:aws, :access_key_id) %>
  secret_access_key: <%= Rails.application.credentials.dig(:aws, :secret_access_key) %>
  region: <%= Rails.application.credentials.dig(:aws, :region) %>
  bucket: your-app-name-<%= Rails.env %>
```

I recommend naming the bucket the same as your app with the env at the end. This way you can easily identify production, development, and test environments.

In `config/environments/production.rb`, set Active Storage to use S3 in production:

```ruby
config.active_storage.service = :amazon
```

Attach File to a Model
In your model (e.g., `app/models/user.rb`):

```ruby
class User < ApplicationRecord
  has_one_attached :avatar
end
```

Update Form for File Upload
In your form view:

```erb
<%= form_with model: @user, local: true do |form| %>
  <%= form.file_field :avatar %>
  <%= form.submit %>
<% end %>
```

<!-- TODO: show how this is rendered in html -->

Handle File Upload in Controller
In your controller, permit the file parameter:

```ruby
def user_params
  params.require(:user).permit(:name, :avatar)
end
```

Displaying the Upload
In your view file:

```erb
<%= image_tag @user.avatar if @user.avatar.attached? %>
```

Conclusion
This guide covered setting up image uploads in Ruby on Rails using Active Storage and Amazon AWS S3. Remember to keep your AWS credentials secure and manage permissions carefully for security.

Additional Resources
[Active Storage Overview](https://guides.rubyonrails.org/active_storage_overview.html)
[AWS SDK for Ruby](https://aws.amazon.com/sdk-for-ruby/)
[AWS S3 Documentation](https://docs.aws.amazon.com/s3/)
