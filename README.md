## JODY - JsOn for DYnamic sites

Многие web приложения отлично работают с традиционным способом формирования статичных страниц. Суть этого простого процесса заключается в следующем:

1. пользователь нажимает на ссылку
2. запрос уходит на сервер
3. сервер обрабатывает запрос и формирует страницу
4. страница отправляется клиенту
5. браузер перерисовывает страницу

По этой схеме работает большинство приложений в сети.

Однако нам все чаще приходится сталкиваться с приложениями в которых контент одной страницы достаточно богат. На странице могут быть фотографии, видео, интерактивные элементы.

В таких приложениях каждая загрузка страницы становится очень дорогим удовольствием и мы все чаще ищем способ сократить количество перезагрузок страницы и подгружать на страницу только новые элементы

Такими элементами могуть небольшие фрагменты страницы, модальные окна, сообщения об ошибках, уведомления, блоки дополнительного контента.

Ниже мы рассмотрим ситуацию для системы, которая использует в основном шаблонизацию на сервере т.е. использует JADE, HAMl или SLIM для формирования вьюшек.

**Пример распространенной задачи**

На сайте с большим кол-вом "тяжелого" контента планируется реализовать следующее:

1. Вход пользователя в систему должен быть выполнен без перезагрузки страницы
2. В случае успеха или ошибки должно появится уведомление
3. В случае успеха в боковой панели должен появится блок с корзиной пользователя
4. В случае успеха форма входа должна быть заменена на блок с информацией о пользователе

### Вариант 1 - "классический"

Мы рассмотрим вариант решения подобной задачи для Rails приложения, хотя конкретный стек технологий не играет здесь особой роли

Программсит мог бы определить форму входа, как форму посылающую AJAX запрос ```remote: true```

В качестве ответа мы (вероятно) захотели бы получить с сервера управляющий JS код. Для этого в качестве типа ответа пришедшего с сервера мы бы указали ```data: { type: :script } ```

Вот так может выглядеть форма входа в Rails приложении

```ruby
= form_for(User.new, url: new_session_path, remote: true, data: { type: :script }, html: { id: :login_form } ) do |f|
  = f.text_field :email
  = f.password_field :password
```

Вот так может выглядеть метод контроллера Session

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
$("#sidebar").append("<%= render partial: "user_accaunt_block" %>");
App.sidebar_initialize();

$("#login_block").empty().append("<%= render partial: "user_info" %>");
App.user_info_initialize();

AppNotifier.alert("<%= t ".login_success" %>");
```

Такой вариант решения задачи довольно распространен и используется во многих проектах.

Однако многие могут заметить, что мы начали писать код стороны клиента на стороне сервера.

Подобный подход грозит нам тем, что скоро наш JavaScript + JQuery код займет в шаблонах на сервере довольно много места, хотя он должен быть сосредоточен только одном месте - **app/assets/javascript**.

Кроме того, есть вероятность, что шаблон может вам придти к вам на поддержку и в таком виде:

```erb
$("#sidebar").append("<%= render partial: "user_accaunt_block" %>"); AppNotifier.alert("<%= t ".login_success" %>");
App.sidebar_initialize(); $("#login_block").empty().append("<%= render partial: "user_info" %>");
App.user_info_initialize();
```

Этот код я оставлю Без комментариев.

### Вариант 2 - JODY

Для решения подобных задач мы предлагаем использовать технику, которая в своей основе использует ряд соглашений и 2 следующих шага:

1. Фрагменты вида формируются на стороне сервера и передаются на соторону клиента в виде структурированных JSON данных
2. JS обработчик (посредник, медиатор), который контролирует поступающие с сервера JSON данные берет на себя большинство рутинных операций с контентом

Данную технику мы назвали JODY

**JODY позволяет нам**

1. НЕ писать JS код на стороне сервера и НЕ засорять им вьюшки
2. НЕ контролировать бОльшую часть рутинных операций с пришедшим от сервера контентом (замена, добавление, инициализация обработчиков)

**JODY позволяет нам**

1. Обеспечить концентрацию JS кода в каталоге **app/assets/javascript**
2. Оградить backend разработчиков от написания JS кода
3. Сократить время на реализацию многих задач, связанных с динамической загрузкой контента на страницу
4. Повысить надежность системы
5. Сократить количество JS кода и облегчить поддержку

**Основными элементами в технике JODY являются**

1. JS посредник между серверной и клиентской стороной
2. JBuilder - для формирования JSON структур

### JODY Mediator

Говоря о посреднике между серверной частью и клиентом я говорю о файле где выполняется определение глобальной реакции на AJAX ответы:

Это может выглядеть примерно так:

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

Идея заключается в том, что пропуская через себя структурированные ответы и используя ряд соглашений, глобальный обреботчик AJAX ответов может избавить разработчиков от написания огромного количества рутинного кода.

Представьте сколько времени можно съэкономить если писать JS код только для того, что бы **показать уведомление**, **вставить полученный HTML в DOM**, **выполнить функции инициализации HTML**.

Уникальную логику обработки AJAX запросов вы все так же сможете реализовать с помощью добавления обработчиков на конкретные элементы

я говорю о чем-то таком:

```coffeescript
$('#login_form').on 'ajax:success', (data, status, xhr) ->
  if data.resourse.id is 10000
    alert "Wow! You are user № 10.000!"
```


### JBuilder for JODY

Все что вам осталось - это начать использовать JBuilder что бы аккуратно формировать ответы сервера с заданной стурктурой.

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
  
  json.invoke do |exec|
    exec.sidebar_initialize
    exec.user_info_initialize
  end

else

  json.flash do
    json.error t ".login_failure"
  end

end
```

Глядя на этот простой пример вы можете отметить, что количество кода вьюшки увеличилось по сравнению с исходным, но, оценивая полученный результат не забудьте отметить и следующие моменты

1. backend разработчик полностью отделен от особенностей реализации forntend части, и вероятно большинство задач не потребует от него дополнительных навыков
2. Благодаря соглашениям вы сократите кол-во JS кода на клиентской стороне
3. Вы очистили свои вьюшки от JS вставок и существенно улучшили поддерживаемость своей системы

#### Благодарности

[@milushov](https://github.com/milushov) - за то, что своим кодом дал мне повод думать, что одна из моих идей может успешно работать
