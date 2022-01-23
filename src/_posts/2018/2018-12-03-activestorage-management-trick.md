---
layout:   post
title:    An Easier Way to Accommodate Deletion of ActiveStorage Attachments
date:       2018-12-03 22:45:00
categories: ruby, rails, activestorage
---

## A Little Background

As I continue to phase out [Paperclip](https://github.com/thoughtbot/paperclip) in favor of [ActiveStorage](https://edgeguides.rubyonrails.org/active_storage_overview.html), I've wanted to keep the methods I used to manage these assets as succinct
and reusable as possible.

ActiveStorage gives us a dead simple way to save and update assets, but the ability to delete assets independently of the parent record, particularly if you're using `has_many_attached`, has been left to each individual app to figure out.

I have stumbled upon a couple of different takes on how to potentially do this, but I've not been satisfied with their approaches. Many do not take into account authorization and permissions, which if you're not careful, allows any user to delete any attachment they choose just by randomly hitting URLs in your application. With that in mind I set out to try and roll my own.

## The Approach

Since all attachments are saved as the same object/models types, [Attachment](https://api.rubyonrails.org/classes/ActiveStorage/Attachment.html) and [Blob](https://api.rubyonrails.org/classes/ActiveStorage/Blob.html), we can use a common controller to interact and modify with them, regardless of the parent record that it belongs to.

In my case I chose to stick closely to the naming conventions already given to us, so I created a new controller named `ActiveStorage::AttachmentsController`, and since for now I'm only worried about deleting attachments, we only need one method, `delete`.

I use [Devise](https://github.com/plataformatec/devise) for authentication, so we need to make sure that the user is logged in before we let them do anything, which we do with `:authenticate_user!`.

I also want to make sure that the user has the permissions to modify these attachments, so I check their permissions to see if they can modify the parent record using the `authorize_attachment_parent!` method. I use [CanCanCan](https://github.com/CanCanCommunity/cancancan) in this case, but you can always swap out your auth call to fit whatever method you use. This piece is *__critical__* to ensure that your users aren't deleting anything they shouldn't.

The `purge_later` call will take care of the actual file deletion in your background queue, and will delete the corresponding `Blob` record.

<div class="overflow-x-scroll">
  {% highlight ruby %}
    # app/controllers/active_storage/attachments_controller.rb
    class ActiveStorage::AttachmentsController < ApplicationController
      before_action :authenticate_user!
      before_action :set_attachment, :authorize_attachment_parent!

      def destroy
        @attachment.purge_later
      end

      private

        def set_attachment
          @attachment = ActiveStorage::Attachment.find(params[:id])
          @record = @attachment.record
        end

        def authorize_attachment_parent!
          authorize! :manage, @record
        end
    end
  {% endhighlight %}
</div>

Also be sure to wire this new controller up in your routes:

<div class="overflow-x-scroll">
  {% highlight ruby %}
    # config/routes.rb
    scope :active_storage, module: :active_storage, as: :active_storage do
      resources :attachments, only: [:destroy]
    end
  {% endhighlight %}
</div>


It's also important to update the user interface to remove any reference to the attachment, and in this case I used Rails UJS. The javascript necessary to remove the element from my UI is simple and straightforward:

<div class="overflow-x-scroll">
  {% highlight javascript %}
    // app/views/active_storage/attachments/destroy.js.erb
    document.getElementById("<%= dom_id @attachment %>").remove()
  {% endhighlight %}
</div>

The only assumptions made here are that you had the representation of your attachment wrapped in a div with ID of `attachment_#{id}`, which you can easily render using Rails handy `dom_id` method.

You can now use the following markup next all of your attachments to allow for easy deletion:

<div class="overflow-x-scroll">
  {% highlight ruby %}
    link_to "Delete", active_storage_attachment_path(attachment),
             method: :delete, remote: :true,
             data: { confirm: "Are you sure you wanna this?" }
  {% endhighlight %}
 </div>


## Wrapping it Up
This approach is relatively simple, and relies heavily on the tools and features that Rails provides to us.

In my case I'm also leveraging a common partial to render thumbnails, metadata, and links for attachments, making it dead simple to add attachments to any model I choose to in the future.

As always things evolve over time, and while I'd like ot think I'm perfect I'm sure there are ways to improve this code, so if you have any suggestions or improvements [I'd love to hear them](https://twitter.com/kylekeesling).
