================
django-robokassa
================

`Читать по-русски <README.rst>`_

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

In `initial` all parameters are optional. For detailed information regarding these paramters, it would be best to look in the Robokassa `documentation <http://robokassa.ru/ru/Doc/Ru/Interface.aspx#222>`_. It is possible to send the values of "custom parameters" in `initial`, described in ROBOKASSA_EXTRA_PARAMS ('shp', again, does not need to be appended to these parameters).

Corresponding template::

    {% extends 'base.html' %}

    {% block content %}
        <form action="{{ form.target }}" method="POST">
            <p>{{ form.as_p }}</p>
            <p><input type="submit" value="Purchase"></p>
        </form>
    {% endblock %}

The form is displayed as a set of hidden input tags.

The form has the attribute `target`, containing the URL to which the form will be sent. In test mode, this will be the test URL.

Please note: you do not need to include {% csrf_token %} on the form (and furthermore, adding it would be unsafe), as the form leads to an external site: the Robokassa website.

Instead of forwarding forms, it is possible to create a GET request. The form has a `get_redirect_url` method, which returns the proper address with all parameters. Redirecting to this address is the same as sendign the form with the GET method.

django-robokassa does not include models for "Purchase" ("Order" for example), as this model will change from one site to another. Processing status changes of purchases must be implemented in the signal handlers.

Receiving Payment Results
------------------------------
There are several methods in Robokassa for determining the result of a payment:

1. Upon page arrival, Success and Fail guarantee that the payment went through or did not go through, respectively.

2. Upon receiving a successful or failed payment, Robokassa sends a POST or GET request to the Result URL.

3. One can request the status of a payment via an XML service.

At this time, methods 1 and 2, and any combination of the two, are supported in django-robokassa (an additional check: that upon arrival to SuccessURL, a notification to ResultURL was already there when using the option ROBOKASSA_STRICT_CHECK = True).

For the sake of security, it is always better to use strict checks (with confirmation via Result URL). The mechanism:

1. After payment, robokassa.ru sends a "background" request to ResultURL.

2. Inside the view, associated with ResultURL, a check of the MD5 signature occurs within the request via ROBOKASSA_PASSWORD2 (this is the second password, which is not transmitted over the network and is known only to the sender and recipient). ROBOKASSA_PASSWORD2 is necessary to confirm that the request was sent precisely to robokassa.ru.

3. If the request is correct, then the view sends the signal ``robokassa.signals.result_received``. To make changes in the site (for example, to add resources according to the arrival of requests or to change the status of an order), you must add the appropriate handler of that signal.

4. If everything is okay, then the view bound to the Result URL renders the robokassa.ru response of type ``OK<operation_id>``, where ``<operation_id>`` is a unique id for the current operation. This response is necessary to ensure that robokassa.ru received confirmation that all  necessary actions were performed.

5. If robokassa.ru receives this response, then the user is redirected to the Success URL. On this page, it is usually best to display a message indicating that the payment was sent successfully. If the view bound to the Result URL does not correspond as expected, then the user is redirected not to the Success URL, but rather to the Fail URL; there, it would be best to show a message about the error.

Signals
-------

Processing status changes of payments should be implemented in the signal handlers.

* ``robokassa.signals.result_received`` - This signal is sent when a notification is received from Robokassa. When this signal is received, it means that the payment was successful. In the `sender` property, an instance of the SuccessNotification model is passed, which has the attributes InvId and OutSum.

* ``robokassa.signals.success_page_visited`` - This signal is sent when the user arrives at the successful payment page. This signal should be used instead of `result_received`, if you are not using strict checks (ROBOKASSA_STRICT_CHECK=False)

* ``robokassa.signals.fail_page_visited`` - This signal is sent when the user arrives at the payment error page. Receiving this signal indicates that the payment was not carried out. In the handler, you should unlock the goods in stock, and so on.

All signals receive the parameters InvId (order number), OutSum (payment amount), and extra (dictionary with additional parameters, described in ROBOKASSA_EXTRA_PARAMS).

Example::

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

To configure Result URL, Success URL, and Fail URL, you can include the robokassa.urls module::

    urlpatterns = patterns('',
        #...
        url(r'^robokassa/', include('robokassa.urls')),
        #...
    )

The addresses that you must specify in the robokassa panel will, in this case, look like the following

* Result URL: ``http://yoursite.ru/robokassa/result/``
* Success URL: ``http://yoursite.ru/robokassa/success/``
* Fail URL: ``http://yoursite.ru/robokassa/fail/``


Templates
-------

* ``robokassa/success.html`` - displayed when a payment is successful. In the context there is a `form` variable of type ``SuccessRedirectForm``, IndId and OutSum with order parameters, and also all additional parameters described in ROBOKASSA_EXTRA_PARAMS.

* ``robokassa/fail.html`` - displayed when a payment fails. In the context there is a `form` variable of type ``FailRedirectForm``, InvId and OutSum with order parameters, and also all additional parameters described in ROBOKASSA_EXTRA_PARAMS.

* ``robokassa/error.html`` - displayed when there is an error for the request to the "success" or "failure" page (for example, when there is a checksum error). In the context there is a `form` variable of the class ``FailRedirectForm`` or ``SuccessRedirectForm``.

Development
===========

Development is underway on github: https://github.com/kmike/django-robokassa

Submit feature requests, ideas, bug reports, etc. to the tracker: https://github.com/kmike/django-robokassa/issues

License - MIT.

Testing
-------

To run tests, install `tox <http://tox.testrun.org/>`_, clone the repository and execute the command

::

    $ tox

from the repository root.
