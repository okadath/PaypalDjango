## Integrando Paypal con Django
```bash
git clone git clone https://github.com/overiq/simple_ecommerce.git
virtualenv venv --python=python3.6
pip  install -r requirements.txt 
pip install django-paypal
```
En codenvy use python3.5 con pytz-2018.9 y Django==2.1.0(que tiene errores de seguridad!)

Correr en codenvy sin errores de ip 
 ```bash
 python manage.py runserver 0.0.0.0:8000
 ```

### Local
Editar el `settings.py`, IPN==Instant Payment Notification

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
Y migrar, ya se puede probar

```
python manage.py migrate
```
![list items](https://raw.githubusercontent.com/okadath/PaypalDjango/master/items.png)

Modificar el `urls.py`:
```python
urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('cart.urls')),
    path('paypal/', include('paypal.standard.ipn.urls')),
]
```
La mayoria del codigo esta en `views.py` y `cart.py`

Agregamos al `views.py`:
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
Estos atributos son requeridos por la API de Paypal:
![variables](https://raw.githubusercontent.com/okadath/PaypalDjango/master/var.png)

Luego para permitir el manejo a traves de su pasarela por si algo sale mal agregamos al archivo  `views.py`, ademas de que le agregaremos el decorador `csrf_exempt` por que usaremos POST de otra pagina web:

```python
from django.views.decorators.csrf import csrf_exempt 
@csrf_exempt
def payment_done(request):
    return render(request, 'ecommerce_app/payment_done.html')

@csrf_exempt
def payment_canceled(request):
    return render(request, 'ecommerce_app/payment_cancelled.html')
```
El checkout del proyecto estaba asi en `views.py`:
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
Despues creamos sus vistas en `ecommerce_app/templates/blog`:

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
![return](https://raw.githubusercontent.com/okadath/PaypalDjango/master/return.png)

Agregar al `ecommerce_app/urls.py`:
```python
	path('process-payment/', views.process_payment, name='process_payment'),
	path('payment-done/', views.payment_done, name='payment_done'),
	path('payment-cancelled/', views.payment_canceled, name='payment_cancelled'),
```
![pay item](https://raw.githubusercontent.com/okadath/PaypalDjango/master/buy.png)

## Remoto
Crear en developers.paypal.com 2 cuentas (casi siempre vienen por default) para manejar pagos
![paypal developers](https://raw.githubusercontent.com/okadath/PaypalDjango/master/p_sand.png)

 comprador: vintaw.01-buyer@gmail.com

Despues de esto ya funciona, solo hace falta setear la informacion para que el IPN acceda al server desde internet,sin esto no se actualiza la informacion acerca de si ya se llevo a cabo la transaccion o no
![no updated](https://raw.githubusercontent.com/okadath/PaypalDjango/master/ipnno.png)

En el tutorial usan NGROK para crear una IP fija en local, yo levantare una instancia temporal de AWS o Codenvy para que sea mas realista

![codenvy](https://raw.githubusercontent.com/okadath/PaypalDjango/master/codenvy.png)

Ya subido en codenvy con sus configuraciones es posible que la API de paypal registre si las operaciones se llevaron a cabo o no por medio del IPN por lo cual ya podemos verificar nosostros si las transacciones se llevaron a cabo o no en la tabla de orders
![updated ipn via web](https://raw.githubusercontent.com/okadath/PaypalDjango/master/ipn.png)

AL CREARLAS POR DEFAULT LAS PONE EN FALSE Y YA PAGADAS CAMBIA A TRUE ese es el workflow de las e-shops pero aun no lo hace

![article no paid](https://raw.githubusercontent.com/okadath/PaypalDjango/master/listnopay.png)

Para ello django tiene dos señales 
+ valid_ipn_received
+ invalid_ipn_received

Asi que debemos de crear un archivo que las maneje en `ecommerce_app/signals.py`

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

Cuando se manda una señal de `valid_ipn_received` signal se llama a la funcion `payment_notification()` con elobjeto PayPalIPN como enviador
`ipn = sender` lo asigna
verificamos en el if si se completo la transaccion 
si es asi comparamos el costo a pagar con el de la tansaccion(para evitar pagar 1$ por algo de 100$ via `order.total_cost() == ipn.mc_gross`) 
si es asi ponemos `order.paid=True`


No hemos registrado el manejador de la señal en `ecommerce_app/apps.py`:
```python
from django.apps import AppConfig
class EcommerceAppConfig(AppConfig):
    name = 'ecommerce_app'
    def ready(self):
        # import signal handlers
        import ecommerce_app.signals
```

Para que django acceda a PaymentConfig agregar en `simple_ecommerce/django_project/ecommerce_app/__init__.py`:

```python
default_app_config = 'ecommerce_app.apps.EcommerceAppConfig'
```
y ya procesa transacciones
![paypal paid](https://raw.githubusercontent.com/okadath/PaypalDjango/master/payart.png)

![article paid](https://raw.githubusercontent.com/okadath/PaypalDjango/master/listpay.png)

### Passing Custom Parameter ( Pass-through variables)
Son datos temporales en el flujo de la aplicacion que paypal no almacena

+ custom
+ item_number
+ invoice

Actualizamos el `paypal_dict` en la funcion `process_payment()` en `ecommerce_app/views.py` 
para pasarle `custom`

```python
...
'currency_code': 'USD',
'custom': 'a custom value',
'notify_url': 'http://{}{}'.format(host, reverse('paypal-ipn')),
...
```
![passing variables](https://raw.githubusercontent.com/okadath/PaypalDjango/master/pass_var.png)
Si necesitasemos pasar muchos valores los pasamos en un array con un delimitador
```bash
>>> l = ["one", "two", "three"] # custom parameters to pass to PayPal
>>> packed_data = str.join("|", l)
>>> packed_data
'one|two|three'
>>> unpacked_data = packed_data.split("|")
>>> unpacked_data
['one', 'two', 'three']
```

TODO:
+ ojo!! elimine el quantize por errores al parsear el dato, no se si asi corra bien con centavos, aun lo tengo que probar!

![error decimales](https://raw.githubusercontent.com/okadath/PaypalDjango/master/dec.png)