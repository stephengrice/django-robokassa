================
django-robokassa
================

django-robokassa is an application for integrating the ROBOKASSA payment system with Django.

Before use, familiarize yourself with the
ROBOKASSA documentation (http://docs.robokassa.ru/en/). The application implements the communication protocol described in this document.

Installation
=========

::

    $ pip install django-robokassa

Then, add 'robokassa' to INSTALLED_APPS and run the following ::

    $ python manage.py syncdb

or, if using South, ::

    $ python manage.py migrate

For it to work, django >= 1.3.1 is required.
Use django-robokassa version 0.9.3 if your project uses django 1.2.x or django 1.1.x.

Settings
=========

You need to specify the following settings in settings.py:

* ROBOKASSA_LOGIN - Login
* ROBOKASSA_PASSWORD1 - Password #1

Optional parameters:

* ROBOKASSA_PASSWORD2 - Password #2. You can omit this, if
  django-robokassa is used only for displaying a payment form.
  If django-robokassa is used for accepting payments, then this parameter is required.

* ROBOKASSA_USE_POST - whether the POST method is used when receiving a response from
  ROBOKASSA. True by default. It is assumed that for the Result URL, Success URL, and
  Fail URL, the same method is selected.

* ROBOKASSA_STRICT_CHECK - whether to use strict verification (which requires
  preliminary notification to the ResultURL). True by default.

* ROBOKASSA_TEST_MODE - whether test mode is enabled. False by default.
  (i.e. combat mode is enabled).

* ROBOKASSA_EXTRA_PARAMS - list of additional parameter names
  that will be sent with requests. "Shp" does not need to be appended to them.

* ROBOKASSA_TEST_FORM_TARGET - The ROBOKASSA url for test mode.
  This setting is for times when there is no domain available on the internet (for example, when using localhost) and instead of a ROBOKASSA server, you must use your own.


Usage
=============

Form for Receiving Payments
-------------------------

In order to simplify the design of the HTML form for sending users to Robokassa, django-robokassa has a form called RobokassaForm. This form is necessary to simplfy the information output to the templates, computing the checksum and forming the parameters of the GET requests.

Example::

    # views.py

    from django.shortcuts import get_object_or_404, render
    from django.contrib.auth.decorators import login_required

    from robokassa.forms import RobokassaForm

    @login_required
    def pay_with_robokassa(request, order_id):
        order = get_object_or_404(Order, pk=order_id)

        form = RobokassaForm(initial={
                   'OutSum': order.total,
                   'InvId': order.id,
                   'Desc': order.name,
                   'Email': request.user.email,
                   # 'IncCurrLabel': '',
                   # 'Culture': 'ru'
               })

        return render(request, 'pay_with_robokassa.html', {'form': form})

В initial все параметры необязательны. Детальную справку по параметрам
лучше посмотреть в `документации <http://robokassa.ru/ru/Doc/Ru/Interface.aspx#222>`_
к Robokassa. Можно передавать в initial значения "пользовательских параметров",
описанных в ROBOKASSA_EXTRA_PARAMS ('shp' к ним приписывать опять не нужно).

Соответствующий шаблон::

    {% extends 'base.html' %}

    {% block content %}
        <form action="{{ form.target }}" method="POST">
            <p>{{ form.as_p }}</p>
            <p><input type="submit" value="Купить"></p>
        </form>
    {% endblock %}

Форма выведется в виде набора скрытых input-тегов.

У формы есть атрибут target, содержащий URL, по которому форму следует
отправлять. В тестовом режиме это будет тестовый URL, в боевом - боевой.

Обратите внимание, {% csrf_token %} в форме не нужен (и более того, добавлять
его к форме небезопасно), т.к. форма ведет на внешний сайт - сайт робокассы.

Вместо отправки формы можно сформировать GET-запрос. У формы есть
метод get_redirect_url, который возвращает нужный адрес со всеми параметрами.
Редирект на этот адрес равносилен отправке формы методом GET.

django-robokassa не включает в себя модели "Покупка" (``Order`` в примере),
т.к. эта модель будет отличаться от сайта к сайту. Обработку смены статусов
покупок следует осуществлять в обработчиках сигналов.


Получение результатов платежей
------------------------------
В Robokassa есть несколько методов определения результата платежа:

1. При переходе на страницы Success и Fail гарантируется, что платеж
   соответственно прошел и не прошел

2. При успешном или неудачном платеже Robokassa отправляет POST или GET запрос
   на Result URL.

3. Можно запрашивать статус платежа через XML-сервис.

В django-robokassa на данный момент поддерживаются методы 1 и 2 и их совмещение
(дополнительная проверка, что при переходе на Success URL уже было уведомление
на Result URL при использовании опции ROBOKASSA_STRICT_CHECK = True).

В целях безопасности лучше всегда использовать строгую проверку
(с подтверждением через Result URL). Ее механизм:

1. После оплаты robokassa.ru отправляет "фоновый" запрос на ResultURL.

2. Внутри view, связанного с ResultURL, происходит проверка содержащейся в
   запросе md5-подписи через ROBOKASSA_PASSWORD2 (это второй пароль, который не
   передается по сети и известен только отправителю и получателю).
   ROBOKASSA_PASSWORD2 нужен для подтверждения того, что запрос был послан
   именно с robokassa.ru.

3. Если запрос правильный, то view шлет сигнал
   ``robokassa.signals.result_received``. Чтоб производить
   манипуляции внутри сайта (например, начислять средства согласно
   пришедшему запросу или менять статус заказа), нужно добавить
   соответствующий обработчик этого сигнала.

4. Если все в порядке, то view, связанный с Result URL,
   отдает robokassa.ru ответ вида ``OK<operation_id>``,
   где ``<operation_id>`` - уникальный id текущей операции.
   Этот ответ  необходим для того, чтобы robokassa.ru получила
   подтверждение того, что все необходимые действия произведены.

5. Если robokassa.ru получает этот ответ, то пользователь перенаправляется
   на Success URL. На этой страничке обычно лучше вывести сообщение
   об успешном прохождении платежа/оплаты. Если ответ view, связанной
   с Result URL, не соответвтует ожидаемому, то пользователь перенаправляется
   не на Success URL, а на Fail URL; там ему хорошо бы показать
   сообщение о произошедшей ошибке.


Сигналы
-------

Обработку смены статусов покупок следует осуществлять в обработчиках сигналов.

* ``robokassa.signals.result_received`` - шлется при получении уведомления от
  Robokassa. Получение этого сигнала означает, что оплата была успешной.
  В качестве sender передается экземпляр модели SuccessNotification, у
  которой есть атрибуты InvId и OutSum.

* ``robokassa.signals.success_page_visited`` - шлется при переходе пользователя
  на страницу успешной оплаты. Этот сигнал следует использовать вместо
  result_received, если не используется строгая проверка
  (ROBOKASSA_STRICT_CHECK=False)

* ``robokassa.signals.fail_page_visited`` - шлется при переходе пользователя
  на страницу ошибки оплаты. Получение этого сигнала означает, что оплата
  не была произведена. В обработчике следует осуществлять разблокирвку товара
  на складе и т.д.

Все сигналы получают параметры InvId (номер заказа), OutSum (сумма оплаты) и
extra (словарь с дополнительными параметрами, описанными в
ROBOKASSA_EXTRA_PARAMS).

Пример::

    from robokassa.signals import result_received
    from my_app.models import Order

    def payment_received(sender, **kwargs):
        order = Order.objects.get(id=kwargs['InvId'])
        order.status = 'paid'
        order.paid_sum = kwargs['OutSum']
        order.extra_param = kwargs['extra']['my_param']
        order.save()

    result_received.connect(payment_received)



urls.py
-------

Для настройки Result URL, Success URL и Fail URL можно подключить
модуль robokassa.urls::

    urlpatterns = patterns('',
        #...
        url(r'^robokassa/', include('robokassa.urls')),
        #...
    )

Адреса, которые нужно указывать в панели robokassa, в этом случае будут иметь вид

* Result URL: ``http://yoursite.ru/robokassa/result/``
* Success URL: ``http://yoursite.ru/robokassa/success/``
* Fail URL: ``http://yoursite.ru/robokassa/fail/``


Шаблоны
-------

* ``robokassa/success.html`` - показывается в случае успешной оплаты. В
  контексте есть переменная form типа ``SuccessRedirectForm``, InvId
  и OutSum с параметрами заказа, а также все дополнительные параметры, описанные
  в ROBOKASSA_EXTRA_PARAMS.

* ``robokassa/fail.html`` - показывается в случае неуспешной оплаты. В
  контексте есть переменная form типа ``FailRedirectForm``, InvId
  и OutSum с параметрами заказа, а также все дополнительные параметры, описанные
  в ROBOKASSA_EXTRA_PARAMS.

* ``robokassa/error.html`` - показывается при ошибочном запросе к странице
  "успех" или "неудача" (например, при ошибке в контрольной сумме). В контексте
  есть переменная form класса ``FailRedirectForm`` или ``SuccessRedirectForm``.

Разработка
==========

Разработка ведется на github: https://github.com/kmike/django-robokassa

Пожелания, идеи, баг-репорты и тд. пишите в трекер: https://github.com/kmike/django-robokassa/issues

Лицензия - MIT.

Тестирование
------------

Для запуска тестов установите `tox <http://tox.testrun.org/>`_, склонируйте репозиторий
и выполните команду

::

    $ tox

из корня репозитория.
