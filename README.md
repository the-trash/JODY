## JODY - JsOn for DYnamic sites

Many web applications work perfectly well rendering static pages the traditional way. Its principles are quite simple:


1. User clicks a link
2. Request is sent to server
3. Server processes the request and builds a page
4. Page is sent to the client
5. Browser redraw the page

Most of the web apps work like that.

Still applications with rich-content web-pages get more and more common. On one page there are photos and carousels, videos, interactive and embedded elements.

In this kind of apps processing of the page is rather resource consuming and we look for the ideas to reduce frequent reloads. One of the ways is trying to add new elements to the existing page.

These elements include small fragments of the page, modal windows and pop-ups, different notifications and additional parts of the content.

Let's analyze behavior of the system which uses server-side templating with Jade, HAML or SLIM for building views.

**An example of common task**

On the rich-content website we'll try to implement the following task:

1. User login should be executed without reload of the page
2. In case of success or error a corresponding notification pop-ups
3. In case of success a shopping-cart widget shows up on a sidebar
4. In case of success login form is replaced by the user info widget


### Variant A - "classic"

We'll analyze a technique of solving such a task in Rails application, although exact technology stack is not relevant.

Developer can define login form as a form which sends AJAX request ```remote: true```

We suppose to get JS script which should be executed on client side. We achieve that by specifying ```data: { type: :script } ``` as a response type.

This is how login form in Rails app looks like

```ruby
= form_for(User.new, url: new_session_path, remote: true, data: { type: :script }, html: { id: :login_form } ) do |f|
  = f.text_field :email
  = f.password_field :password
```

Here is the method of Session controller

**session_controller.rb**
```ruby
def create
  if @user = User.find_by_email_and_password(params[:email], params[:password])
    sign_in @user
    render template: :sign_in_success, layout: false
  else
    render template: :sign_in_error, layout: false
  end
end
```

and this is one of the templates:

**sign_in_success.js.erb**
```erb
$("#sidebar").append("<%= render partial: "shopping_cart_block" %>");
App.sidebar_initialize();

$("#login_block").empty().append("<%= render partial: "user_info" %>");
App.user_info_initialize();

AppNotifier.alert("<%= t ".login_success" %>");
```

This technique is commonly used in a lot of projects.

However we understand that we'd started writing client-side code on the server side.


Such approach will lead us to Javascript & JQuery code taking up a lot of space in server templates while it has to be concentrated in **app/assets/javascript**.

Unfortunately, in real world you can to get to supporting following code

```erb
$("#sidebar").append("<%= render partial: "shopping_cart_block" %>"); AppNotifier.alert("<%= t ".login_success" %>");
App.sidebar_initialize(); $("#login_block").empty().append("<%= render partial: "user_info" %>");
App.user_info_initialize();
```

I leave it uncommented. Yeah.

### Variant "B" - JODY

We recommend the following technique for implementing solutions for tasks like that. It is based on several conventions and includes 2 parts:

1. Partials are built on the server-side and transferred to the client-side as structured JSON
2. Global JS-handler (Mediator) which controls incoming JSON and executes routine operations with content

We call it JODY

**JODY includes**

1. JS-mediator for client to server communication/interaction
2. Builder of JSON (we use JBuilder)
3. Conventions for JSON building on server side

**JODY helps**

1. Remove server-side JS code, which pollutes view templates
2. Reduce code which provide control over majority of  operations with content received from server: replacing it, adding new parts, invoking initializing methods

**JODY gives**

1. Concentration of JS in **app/assets/javascript** folder
2. Salvation from writing JS for back-end developers
3. Reduced time on tasks regarding dynamic page content
4. High system reliability
5. Reduced JS code and easier support

### JODY Mediator

We define mediator of client to server interaction as a file where we have few handlers for global reaction to AJAX responses:

And it looks like that:

**JODY.js.coffee**
```coffeescript
$(document).ajaxError (event, request, settings) ->
  if typeof (data = request.responseJSON) is "object"
    flash  = data.flash
    errors = data.errors
    url    = data.redirect_to 

    TheNotification.show_flash flash
    TheNotification.show_errors errors

    Redirect.to url

$(document).ajaxSuccess (event, xhr, params, data) ->
  flash  = data.flash
  errors = data.errors
  url    = data.redirect_to 
  
  html_partials = data.html_partials
  initializers  = data.initializers

  TheNotification.show_flash flash
  TheNotification.show_errors errors
  
  #...
  
  HtmlRenderer.render  html_partials
  InvokeFunctions.exec initializers
  
  Redirect.to url
```

The idea is that global handler of AJAX responses works with structured responses and relieves developers of writing redundant code.

Can you imagine how much time can you save if you write JS only to **show notification**, **insert obtained HTML into DOM**, **execute HTML initialization**?

You can still use unique logic for handling AJAX requests implementing with hendlers to specific elements.

Something like that for instance:

```coffeescript
$('#login_form').on 'ajax:success', (data, status, xhr) ->
  if data.resourse.id is 10000
    alert "Wow! You are user № 10.000!"
```

### JBuilder for JODY

Let's get back to the previous task and solve it using the power of JODY.

Here is the login form, which should to get JSON from server. Notice we've changed the expected server response type to ```type: :json```


```ruby
= form_for(User.new, url: new_user_session_path, remote: true, data: { type: :json }) do |f|
  = f.text_field :email
  = f.password_field :password
```

And take a look on the Session controller:

```ruby
def create
  if @user = User.find_by_email_and_password(params[:email], params[:password])
    sign_in @user
  end
end
```

and your jbuilder template will look like that:

**create.json.jbuilder**
```ruby
if current_user

  json.flash do
    json.info t '.login_success'
  end

  json.html_partials do |html|
    html.replace do |div|
      div.login_block render(partial: "user_info.html")
    end
    
    html.append do |div|
      div.sidebar render(partial: "shopping_cart_block.html")
    end
  end
  
  json.initializers do |run|
    run.sidebar_initialize
    run.user_info_initialize
  end

else

  json.errors @user.errors
  
  json.flash do
    json.error t ".login_failure"
  end

end
```

You can notice more code in the view in comparison to the source but don't forget the following 

1. Backend developer do not interfere in the front-end and no additional skills are required of him
2. You reduce JS on the client side thanks to the conventions
3. You've purified your views from JS and improved your system's sustainability

### Yet another example

Ниже я приведу пример кода, который формирует на сервере данные для модального окна и инициализирует элементы шаблона, когда они будут отрисованы на странице:

```ruby
json.modal do |modal|
  modal.title "Post: #{ @post.title }"
  modal.content render(partial: "posts/form.html")
  modal.initialze true
end

json.window do |window|
  window.title "Edit post: #{ @post.title }"
  window.url   edit_post_url(@post)
end
```

This JSON response can be the intermediary for instructions JODY that would do following things:

1. create a modal window to edit the post
2. Initialize the post editing form after HTML inserting into DOM
3. Change a title of browser's window
4. Set the current browser URL

### So what's so new?

Yes, the idea of returning partials to the client side with JSON is not novel and you can refer to discussions like  [this](http://stackoverflow.com/questions/13713250/render-to-string-partial-format-error-in-controller) on stackowerflow

We just suggest using this technique on the advanced level to improve and organize sustainable systems with conventions and few lines of code.

### Am I limited to Rails if I want to use JODY?

Well of course not! There can be more of challenge to setup sending of AJAX requests to server but it's not that big of a deal.

There are following basic requirements if you want to use JODY:

1. System mostly uses server side rendering
2. Server responses with structured JSON data which accords to your corporate conventions


All you have to do is pick out the most routine operations on client side and simplify them with JODY handler.

### Are there thorough specifications for JODY?

Writing a JODY handler is an easy task. Agreeing on conventions is a process requiring participation of a lot of people. You also need to review and breakdown several successful practices implemented in major projects.

We're not ready to give the complete recipe to the community at the moment and offer you to come to your own conventions. We're sure it's gonna be a fun task for a team.

### About autor

Ilya Zykin - rails developer and team lead at [CreateDigital LLC](http://createdigital.me). Ilya and his team of talented developers look for interesting practices and solutions which improve quality of the products and simplify the development process. Ilya’s area of interest includes e-publishing systems and social services.

CreateDigital LLC specializes in development of complex high load web services with a real value, which benefit the proprietors and bring joy to the users.

### Discussion about JODY

Please, let me know about your opinion or practice

[Discussion](https://github.com/the-teacher/JODY/issues/1)

#### Acknowledgements

[@milushov](https://github.com/milushov) - Thanks to Roman, whose code helped finally formulating the idea

[@bel9ev](https://github.com/bel9ev) - for help with En version and awesome project management
