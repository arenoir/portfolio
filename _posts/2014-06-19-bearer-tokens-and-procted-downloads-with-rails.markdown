---
layout: post
title:  "Using bearer tokens to authenticate protected images"
date:   2014-06-19
---

I am developing a long running javascript application using ember.js.  Best practice authenticating ember applications is not defined but I am moving toward token auth. Authorizing requests to the backend rest api by passing a `auth_token` in the request header.

How do you authorize images? You can't add headers to img tags.  I could add the `auth_token` to the url in a query string but that would not be ideal. Not only would that fill my server logs with auth tokens, it would make it very easy for a user to accidentally send their token to someone. For example a user thinks they can send the img url to someone in an email.

My solution was to create essentially another token and call it a `download_ticket`.  This ticket is the `auth_token` but encrypted.  So ember makes a request for all `protected_images` rails serializes the `protected_image` attributes and creates a `download_ticket` for each. Ember then makes the request for the image file including the `download_ticket` in the query params. Rails decrypts the `download_token` and verifies the `auth_token` is still valid and streams the file. 


Here is the relevant rails code.  


```ruby
#app/models/download_ticket.rb
class DownloadTicket
  attr_reader :encryptor

  def initialize(id)
    @asset_id  = attachment_id
    @encryptor = Rails.application.message_verifier("asset-#{id}")
  end

  def ticketize(token)
    encryptor.generate(token)
  end

  def tokenize(ticket)
    encryptor.verify(ticket)
  end

end



#app/controllers/protected_images_controller.rb
class ProtectedImagesController < ApplicationController
  # sets current_user and auth_token throws exception when not authorized.
  before_filter :authenticate_user_from_token!
  
  def index
    render json: collection, auth_token: auth_token
  end
  
  private
  
  def collection
    apply_scopes(ProtectedImage).where(user_id: current_user.id)
  end
  
end



#app/controllers/download_controller.rb
class DownloadController < ApplicationController
  before_filter :authenticate!
  
  def show
    send_file( asset.path, type: asset.content_type, inline: !params[:download] )
  end
  
  private
  
  def authenticate!
    ticket = params[:download_ticket]

    raise unless ticket

    token = DownloadTicket.new(params[:id]).tokenize(ticket)

    # sets current_user and auth_token throws exception when not authorized.
    authenticate_user_from_token(token)
  end
  
  def asset
    ProtectedImage.where(user_id: current_user.id).find(params[:id])
  end
  
end



#app/serializer/protected_image_serializer.rb
class ProtectedImageSerializer < ActiveModel::Serializer
  
  attributes(
    :id,
    :user_id,
    :title,
    :filename,
    :description,
    :download_ticket
  )

  def download_ticket
    DownloadTicket.new(object.id).ticketize(@options[:auth_token])
  end

end

```
