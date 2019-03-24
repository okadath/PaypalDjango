```bash
virtualenv venv --python=python3.6
pip  install -r requirements.txt 
pip install django-paypal
```
en codenvy use python3.5 con pytz-2018.9 y Django==2.1.0
 que tiene errores de seguridad! 
 correr en codenvy sin errores de ip 
 ```bash
 python manage.py runserver 0.0.0.0:8000
 ```

editar el `settings.py`, IPN==Instant Payment Notification

```python
INSTALLED_APPS = [
    ...
    'ecommerce_app',
    'paypal.standard.ipn',    
]
...

PAYPAL_RECEIVER_EMAIL = 'youremail@mail.com'
 
PAYPAL_TEST = True

```
```
python manage.py migrate
```
y modificar el `urls.py`:
```python
urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('cart.urls')),
    path('paypal/', include('paypal.standard.ipn.urls')),
]
```
la mayoria del codigo esta en `views.py` y `cart.py`

agregamos al `views.py`:
```python
def process_payment(request):
    order_id = request.session.get('order_id')
    order = get_object_or_404(Order, id=order_id)
    host = request.get_host()
 
    paypal_dict = {
        'business': settings.PAYPAL_RECEIVER_EMAIL,
        'amount': '%.2f' % order.total_cost().quantize(
            Decimal('.01')),
        'item_name': 'Order {}'.format(order.id),
        'invoice': str(order.id),
        'currency_code': 'USD',
        'notify_url': 'http://{}{}'.format(host,
                                           reverse('paypal-ipn')),
        'return_url': 'http://{}{}'.format(host,
                                           reverse('payment_done')),
        'cancel_return': 'http://{}{}'.format(host,
                                              reverse('payment_cancelled')),
    }
 
    form = PayPalPaymentsForm(initial=paypal_dict)
    return render(request, 'ecommerce_app/process_payment.html', {'order': order, 'form': form})
```

para llamar a la API de paypal

Luego para permitir el manejo a traves de su pasarela por si algo sale mal agregamos al archivo 

```python
from django.views.decorators.csrf import csrf_exempt 
@csrf_exempt
def payment_done(request):
    return render(request, 'ecommerce_app/payment_done.html')

@csrf_exempt
def payment_canceled(request):
    return render(request, 'ecommerce_app/payment_cancelled.html')
```
el checkout del proyecto estaba asi:
```python
def checkout(request):
    if request.method == 'POST':
        form = CheckoutForm(request.POST)
        if form.is_valid():
            cleaned_data = form.cleaned_data
            o = Order(
                name = cleaned_data.get('name'),
                email = cleaned_data.get('email'),
                postal_code = cleaned_data.get('postal_code'),
                address = cleaned_data.get('address'),
            )
            o.save()

            all_items = cart.get_all_cart_items(request)
            for cart_item in all_items:
                li = LineItem(
                    product_id = cart_item.product_id,
                    price = cart_item.price,
                    quantity = cart_item.quantity,
                    order_id = o.id
                )

                li.save()

            cart.clear(request)

            request.session['order_id'] = o.id

            messages.add_message(request, messages.INFO, 'Order Placed!')
            return redirect('checkout')


    else:
        form = CheckoutForm()
        return render(request, 'ecommerce_app/checkout.html', {'form': form})
```

lo editaremos asi al final de la funcion checkout:

```python
		cart.clear(request)
		request.session['order_id'] = o.id
		return redirect('process_payment') 
else:
	form = CheckoutForm()
	return render(request, 'ecommerce_app/checkout.html', locals())

```
despues creamos sus vistas en `ecommerce_app/templates/blog`:

+ `process_payment.html`:

```django	
{% extends 'ecommerce_app/base.html' %}
 
{% block title %}Make Payment{% endblock %}
 
{% block content %}
    <h4>Pay with PayPal</h4>
    {{ form.render }}
{% endblock %}
```

+ `payment_done.py`:

```django
{% extends 'ecommerce_app/base.html' %}
 
{% block content %}
 
    <p>Payment done.</p>
 
{% endblock %}
```

+ `payment_cancelled.html`:

```django	
{% extends 'ecommerce_app/base.html' %}
 
{% block content %}
 
    <p>Payment cancelled.</p>
 
{% endblock %}
```

agregar al `ecommerce_app/urls.py`:
```python
	path('process-payment/', views.process_payment, name='process_payment'),
	path('payment-done/', views.payment_done, name='payment_done'),
	path('payment-cancelled/', views.payment_canceled, name='payment_cancelled'),
```


crear en developers.paypal.com 2 cuentas (casi siempre vienen por default) para manejar pagos

 comprador: vintaw.01-buyer@gmail.com

despues de esto ya funciona, solo hace falta setear la informacion para que el IPN acceda al server desde internet, en el tutorial usan ngrok para crear una IP fija, yo levantare una instancia temporal de AWS para que sea mas realista


ya subido en codenvy con sus configuraciones es posible que la API de paypal registre si als operaciones se llevaron
a cabo o no por medio del IPN por lo cual ya podemos verificar nosostros si las transacciones se llevaron a cabo o no en la tBLA DE ORDERS

AL CREARLAS POR DEFAULT LAS PONE EN FALSE Y YA PAGADAS LAS PONE EN TRUE ese es el workflow de las eshops

para ello django tiene dos se;ales 
+ valid_ipn_received
+ invalid_ipn_received

asi que debemos de crear un archivo que las maneje en `ecommerce_app/signals.py`

```python
from django.shortcuts import get_object_or_404
from .models import Order
from paypal.standard.ipn.signals import valid_ipn_received
from django.dispatch import receiver
 
 
@receiver(valid_ipn_received)
def payment_notification(sender, **kwargs):
    ipn = sender
    if ipn.payment_status == 'Completed':
        # payment was successful
        order = get_object_or_404(Order, id=ipn.invoice)
 
        if order.total_cost() == ipn.mc_gross:
            # mark the order as paid
            order.paid = True
            order.save()
```


When valid_ipn_received signal is sent, payment_notification() function is called 
with the PayPalIPN object as a sender.
In line 1, we assign the PayPalIPN object to the ipn variable.
In line 2, we check whether the payment_status attribute is equal to 'Completed'.
 If it is, we fetch the Order object and compare the total cost of the order 
 (via get_total_cost()) with the transaction amount ( i.e mc_gross). 
 This comparison is necessary, because it keeps someone from trying to pay $1 
 for an order of $100. If the comparison succeeds we mark the product paid by setting 
 order.paid=True.


no hemos registrado el manejador de la se;al en `ecommerce_app/apps.py`:
```python
from django.apps import AppConfig
class EcommerceAppConfig(AppConfig):
    name = 'ecommerce_app'
    def ready(self):
        # import signal handlers
        import ecommerce_app.signals
```

To inform Django about the existence of PaymentConfig add the following line in
` __init__.py` file:

`simple_ecommerce/django_project/ecommerce_app/__init__.py`:

```python
default_app_config = 'ecommerce_app.apps.EcommerceAppConfig'
```
y ya procesa transacciones

Passing Custom Parameter ( Pass-through variables)
son datos temporales en el flujo de la aplicacion que paypal no almacena

+ custom
+ item_number
+ invoice

actualizamos el `paypal_dict` en la funcion `process_payment()` en `ecommerce_app/views.py` 
para pasarle `custom`

```python
...
'currency_code': 'USD',
'custom': 'a custom value',
'notify_url': 'http://{}{}'.format(host, reverse('paypal-ipn')),
...
```

si necesitasemos pasar muchos valores los pasamos en un array con un delimitador
```bash
 l = ["one", "two", "three"] # custom parameters to pass to PayPal
>>> packed_data = str.join("|", l)
>>> packed_data
'one|two|three'
>>> unpacked_data = packed_data.split("|")
>>> unpacked_data
['one', 'two', 'three']
```


