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
4. In case of success login form is replaced by the user info widget (???)


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

и примерно вот так будет выглядеть один из шаблонов:

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

And there is a possibility that you'd find yourself supporting a code like that one:

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

На стороне сервера JODY обеспечивается JBuilder'ом. Он используется для того, что бы аккуратно формировать ответы сервера с заданной структурой.

Вернемся начальной задаче и посмотрим, как бы она могла быть решена с использованием JODY

Вот так может выглядеть форма входа для JODY. Здесь мы изменили тип ожидаемого от сервера ответа ```type: :json```

```ruby
= form_for(User.new, url: new_user_session_path, remote: true, data: { type: :json }) do |f|
  = f.text_field :email
  = f.password_field :password
```

Вот так может выглядеть метод контроллера Session

```ruby
def create
  if @user = User.find_by_email_and_password(params[:email], params[:password])
    sign_in @user
  end
end
```

и примерно вот так будет выглядеть jbuilder шаблон:

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
      div.sidebar render(partial: "user_accaunt_block.html")
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

Глядя на этот простой пример вы можете отметить, что количество кода вьюшки увеличилось по сравнению с исходным, но, оценивая полученный результат не забудьте отметить и следующие моменты

1. backend разработчик полностью отделен от особенностей реализации forntend части, и вероятно большинство задач не потребует от него дополнительных навыков
2. Благодаря соглашениям вы сократите кол-во JS кода на клиентской стороне
3. Вы очистили свои вьюшки от JS вставок и существенно улучшили поддерживаемость своей системы

### Еще один пример

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

Подобный JSON ответ может быть инструкцией JODY посреднику для того, что бы сформировать

1. сформировать модальное окно для редактирования поста
2. Проинициализировать форму редактирования поста после построения HTML
3. Изменить заголовок окна браузера
4. Установить текущи URL браузера. 

### Но ведь идея не нова?

Да, идея возвращения фрагментов вида на сторону клиента с помощью JSON структур не нова и вы можете встретить вопросы на эту тему на stackowerflow, например, [тут](http://stackoverflow.com/questions/13713250/render-to-string-partial-format-error-in-controller)

Мы лишь предлагаем использовать эту возможность на чуть более продвинутом уровне, что бы с помощью соглашений и небольшого количества кода существенно улучшить поддерживаемые системы и внести в них долю порядка.

### Могу ли я использовать технику JODY не для Rails?

Да, конечно! Вероятно вам придется приложить чуть больше сил, что бы обеспечить отправку AJAX запросов на сервер, но трудностей у вас возникнуть не должно.

Главные ожидания от систем которые хотят использовать JODY следующие

1. Шаблонизация производится в основном на серверной стороне
2. Сервер возвращает ответы в виде JSON данных со структурой подчиненной вашим корпоративным соглашениям

Все что вам останется - выделить наиболее рутинные операции на клиентской части приложения и автоматизировать их выполнение с помощью JODY посредника 

### Есть ли у JODY более точная спецификация?

Написание JODY посредника - задача довольно простая. Формирование соглашений - процесс требующий вовлечения большого количества людей и обзора хотя бы нескольких удачных практик на крупных проектах и выделение из них рационального зерна.

На данный момент мы не готовы дать сообществу готовый рецепт и предлагаем вам при необходимости придти к своим соглашениям. Наверняка это будет увлекательным процессом для всей команды.

#### Об авторе

Илья Зыкин -  rails разработчик и teamlead в компании [CreateDigital.me](http://createdigital.me). Илья в команде с талантливыми разработчиками компании CreateDigital старается найти интересные практики и решения для улучшения качеств создаваемых продуктов и упрощения процесса разработки. Основное увлечение Ильи - системы электронных публикаций и социально-ориентированные сервисы.

Компания СreateDigital специализируется на создании сложных, комплексных, массовых и высоконагруженных веб-сервисов, которые несут в себе реальную ценность, приносят дивиденды своим владельцам и доставляют пользу и радость людям.

#### Благодарности

[@milushov](https://github.com/milushov) - Спасибо Роману, фрагменты кода которого дали повод довести описанную идею до логического завершения
