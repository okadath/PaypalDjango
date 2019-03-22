```bash
virtualenv venv --python=python3.6
pip  install -r requirements.txt 
pip install django-paypal
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











