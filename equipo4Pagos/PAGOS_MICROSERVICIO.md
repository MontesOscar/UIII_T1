# Microservicio de Pagos — Equipo 4 (Payment Service)

## Qué hace este microservicio

Simula una pasarela de cobro. Recibe un pedido y datos de tarjeta, consulta el total
al servicio de pedidos (Equipo 3), procesa el cobro (simulado) y si es exitoso
notifica a pedidos para cambiar su estado a "Pagado".

**Solo consume al Equipo 3.** El `user_id` lo recibe directamente del cliente en el body.

---

## Dependencias con otros equipos

| Acción | Método | Endpoint (Equipo 3) | Para qué |
|--------|--------|----------------------|---------|
| Consultar pedido | GET | `/api/orders/<id>/` | Obtener el total a cobrar y verificar estado |
| Actualizar estado | PATCH | `/api/orders/<id>/status/` | Cambiar estado a "Pagado" |

---

## Paso 1 — Crear el proyecto Django

```bash
# Dentro de la carpeta UIII_T1/
mkdir equipo4Pagos
cd equipo4Pagos

python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

pip install django djangorestframework requests python-decouple drf-spectacular
pip freeze > requirements.txt

django-admin startproject pagos_service .
python manage.py startapp pagos
```

Estructura resultante:
```
equipo4Pagos/
├── manage.py
├── requirements.txt
├── pagos_service/
│   ├── settings.py
│   ├── urls.py
│   └── ...
└── pagos/
    ├── models.py
    ├── serializers.py
    ├── views.py
    └── urls.py
```

---

## Paso 2 — Configurar `settings.py`

```python
# pagos_service/settings.py

INSTALLED_APPS = [
    ...
    'rest_framework',
    'drf_spectacular',
    'pagos',
]

REST_FRAMEWORK = {
    'DEFAULT_SCHEMA_CLASS': 'drf_spectacular.openapi.AutoSchema',
}

SPECTACULAR_SETTINGS = {
    'TITLE': 'Payment Service API',
    'DESCRIPTION': 'Microservicio de pagos — Equipo 4',
    'VERSION': '1.0.0',
}

# URL base del servicio de pedidos (Equipo 3)
# Mientras no esté disponible, apunta al mock local
ORDERS_SERVICE_URL = 'http://127.0.0.1:8003'   # cambiar a la IP real en red local
```

> Puedes usar `python-decouple` y un archivo `.env` para no hardcodear la URL:
> ```
> ORDERS_SERVICE_URL=http://192.168.1.X:8003
> ```

---

## Paso 3 — Crear el modelo `Pago`

```python
# pagos/models.py

from django.db import models

class Pago(models.Model):
    ESTADO_CHOICES = [
        ('exitoso', 'Exitoso'),
        ('fallido', 'Fallido'),
    ]

    order_id        = models.IntegerField()
    user_id         = models.IntegerField()
    total_cobrado   = models.DecimalField(max_digits=10, decimal_places=2)
    card_number     = models.CharField(max_length=4)   # solo últimos 4 dígitos
    expiration_date = models.CharField(max_length=7)   # MM/YY
    estado          = models.CharField(max_length=10, choices=ESTADO_CHOICES)
    fecha           = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return f"Pago #{self.id} — Pedido {self.order_id} — {self.estado}"
```

```bash
python manage.py makemigrations
python manage.py migrate
```

---

## Paso 4 — Crear el serializer

```python
# pagos/serializers.py

from rest_framework import serializers
from .models import Pago

class ProcesarPagoSerializer(serializers.Serializer):
    """Datos que recibe el endpoint para procesar un pago."""
    order_id        = serializers.IntegerField()
    user_id         = serializers.IntegerField()
    card_number     = serializers.CharField(max_length=19)
    expiration_date = serializers.CharField(max_length=7)   # MM/YY
    cvv             = serializers.CharField(max_length=4, write_only=True)


class PagoResponseSerializer(serializers.ModelSerializer):
    """Datos que se devuelven tras procesar el pago."""
    class Meta:
        model  = Pago
        fields = ['id', 'order_id', 'user_id', 'total_cobrado',
                  'card_number', 'estado', 'fecha']
```

---

## Paso 5 — Crear el cliente HTTP para Equipo 3

Crea un archivo separado para aislar las llamadas externas:

```python
# pagos/services.py

import requests
from django.conf import settings

ORDERS_URL = settings.ORDERS_SERVICE_URL

def obtener_pedido(order_id):
    """
    GET /api/orders/<id>/
    Retorna el dict del pedido o None si falla.
    """
    try:
        response = requests.get(
            f'{ORDERS_URL}/api/orders/{order_id}/',
            timeout=5
        )
        response.raise_for_status()
        return response.json()
    except requests.exceptions.ConnectionError:
        return None
    except requests.exceptions.HTTPError:
        return None
    except requests.exceptions.Timeout:
        return None


def marcar_pedido_pagado(order_id):
    """
    PATCH /api/orders/<id>/status/
    Cambia el estado del pedido a 'Pagado'.
    """
    try:
        response = requests.patch(
            f'{ORDERS_URL}/api/orders/{order_id}/status/',
            json={'status': 'Pagado'},
            timeout=5
        )
        response.raise_for_status()
        return True
    except requests.exceptions.RequestException:
        return False
```

---

## Paso 6 — Crear la vista principal

```python
# pagos/views.py

from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from drf_spectacular.utils import extend_schema

from .serializers import ProcesarPagoSerializer, PagoResponseSerializer
from .models import Pago
from . import services


def _simular_cobro(card_number, cvv, expiration_date, total):
    """
    Simulación de cobro. En producción real aquí iría la integración
    con Stripe, Conekta, etc.
    Regla simple: si el número de tarjeta termina en '0000' → falla.
    """
    if card_number.replace(' ', '').endswith('0000'):
        return False
    return True


@extend_schema(
    request=ProcesarPagoSerializer,
    responses={200: PagoResponseSerializer, 402: None, 503: None}
)
class ProcesarPagoView(APIView):

    def post(self, request):
        serializer = ProcesarPagoSerializer(data=request.data)
        if not serializer.is_valid():
            return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

        data     = serializer.validated_data
        order_id = data['order_id']
        user_id  = data['user_id']

        # 1. Consultar pedido a Equipo 3
        pedido = services.obtener_pedido(order_id)
        if pedido is None:
            return Response(
                {'error': 'El servicio de pedidos no responde. Intente más tarde.'},
                status=status.HTTP_503_SERVICE_UNAVAILABLE
            )

        # 2. Verificar que el pedido pertenece al usuario
        if pedido.get('user_id') != user_id:
            return Response(
                {'error': 'El pedido no pertenece al usuario indicado.'},
                status=status.HTTP_403_FORBIDDEN
            )

        # 3. Verificar que el pedido está pendiente
        if pedido.get('status') != 'Pendiente':
            return Response(
                {'error': f"El pedido tiene estado '{pedido.get('status')}'. Solo se pueden pagar pedidos 'Pendiente'."},
                status=status.HTTP_400_BAD_REQUEST
            )

        total = pedido.get('total')

        # 4. Simular cobro con los datos de tarjeta
        cobro_exitoso = _simular_cobro(
            card_number     = data['card_number'],
            cvv             = data['cvv'],
            expiration_date = data['expiration_date'],
            total           = total
        )

        # 5. Guardar registro del pago (exitoso o fallido)
        pago = Pago.objects.create(
            order_id        = order_id,
            user_id         = user_id,
            total_cobrado   = total,
            card_number     = data['card_number'][-4:],   # solo últimos 4 dígitos
            expiration_date = data['expiration_date'],
            estado          = 'exitoso' if cobro_exitoso else 'fallido'
        )

        if not cobro_exitoso:
            return Response(
                {'error': 'El cobro fue rechazado. Verifique los datos de su tarjeta.'},
                status=status.HTTP_402_PAYMENT_REQUIRED
            )

        # 6. Notificar a Equipo 3 que el pedido fue pagado
        notificado = services.marcar_pedido_pagado(order_id)
        if not notificado:
            # El pago se procesó pero no se pudo actualizar el pedido
            # Se guarda el pago y se devuelve advertencia
            return Response({
                'advertencia': 'Pago procesado pero no se pudo actualizar el estado del pedido.',
                'pago': PagoResponseSerializer(pago).data
            }, status=status.HTTP_207_MULTI_STATUS)

        return Response(
            PagoResponseSerializer(pago).data,
            status=status.HTTP_200_OK
        )
```

---

## Paso 7 — Configurar las URLs

```python
# pagos/urls.py

from django.urls import path
from .views import ProcesarPagoView

urlpatterns = [
    path('process/', ProcesarPagoView.as_view(), name='procesar-pago'),
]
```

```python
# pagos_service/urls.py

from django.contrib import admin
from django.urls import path, include
from drf_spectacular.views import SpectacularAPIView, SpectacularSwaggerView

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/payments/', include('pagos.urls')),

    # Swagger / documentación
    path('api/schema/', SpectacularAPIView.as_view(), name='schema'),
    path('api/docs/', SpectacularSwaggerView.as_view(url_name='schema'), name='swagger-ui'),
]
```

---

## Paso 8 — Mock de Equipo 3 (para desarrollo aislado)

Mientras Equipo 3 no esté disponible, crea un mock local en `pagos/mock_orders.py`:

```python
# pagos/mock_orders.py
# Borrar este archivo cuando Equipo 3 esté disponible

MOCK_ORDERS = {
    1: {'id': 1, 'user_id': 1, 'total': '250.00', 'status': 'Pendiente'},
    2: {'id': 2, 'user_id': 2, 'total': '89.99',  'status': 'Pendiente'},
}

def mock_obtener_pedido(order_id):
    return MOCK_ORDERS.get(order_id)

def mock_marcar_pagado(order_id):
    if order_id in MOCK_ORDERS:
        MOCK_ORDERS[order_id]['status'] = 'Pagado'
        return True
    return False
```

Y en `services.py`, activa el mock con una variable de entorno:

```python
# pagos/services.py (versión con mock)

import os, requests
from django.conf import settings

USE_MOCK = os.getenv('USE_MOCK_ORDERS', 'false').lower() == 'true'

def obtener_pedido(order_id):
    if USE_MOCK:
        from .mock_orders import mock_obtener_pedido
        return mock_obtener_pedido(order_id)
    # ... código real con requests ...

def marcar_pedido_pagado(order_id):
    if USE_MOCK:
        from .mock_orders import mock_marcar_pagado
        return mock_marcar_pagado(order_id)
    # ... código real con requests ...
```

Ejecutar con mock activo:
```bash
USE_MOCK_ORDERS=true python manage.py runserver 8004
```

---

## Paso 9 — Probar con curl o Postman

### Caso exitoso
```bash
curl -X POST http://127.0.0.1:8004/api/payments/process/ \
  -H "Content-Type: application/json" \
  -d '{
    "order_id": 1,
    "user_id": 1,
    "card_number": "4111 1111 1111 1234",
    "expiration_date": "12/27",
    "cvv": "123"
  }'
```

Respuesta esperada (`200 OK`):
```json
{
  "id": 1,
  "order_id": 1,
  "user_id": 1,
  "total_cobrado": "250.00",
  "card_number": "1234",
  "estado": "exitoso",
  "fecha": "2026-04-09T10:00:00Z"
}
```

### Caso fallido (tarjeta rechazada — termina en 0000)
```bash
curl -X POST http://127.0.0.1:8004/api/payments/process/ \
  -H "Content-Type: application/json" \
  -d '{
    "order_id": 2,
    "user_id": 2,
    "card_number": "4111 1111 1111 0000",
    "expiration_date": "12/27",
    "cvv": "999"
  }'
```

Respuesta esperada (`402 Payment Required`):
```json
{
  "error": "El cobro fue rechazado. Verifique los datos de su tarjeta."
}
```

---

## Paso 10 — Conectar con Equipo 3 en red local

Cuando Equipo 3 esté disponible en la red Wi-Fi del salón:

1. Pedir su IP local (ej. `192.168.1.50`) y puerto (ej. `8003`)
2. Actualizar en `settings.py`:
   ```python
   ORDERS_SERVICE_URL = 'http://192.168.1.50:8003'
   ```
3. Desactivar el mock:
   ```bash
   python manage.py runserver 0.0.0.0:8004
   # El 0.0.0.0 hace que tu API también sea accesible desde la red local
   ```

---

## Resumen de endpoint

| Método | Ruta | Descripción |
|--------|------|-------------|
| POST | `/api/payments/process/` | Procesar un pago |
| GET | `/api/docs/` | Swagger UI (documentación) |

### Body esperado en `/api/payments/process/`
```json
{
  "order_id": 1,
  "user_id": 1,
  "card_number": "4111 1111 1111 1234",
  "expiration_date": "12/27",
  "cvv": "123"
}
```

### Códigos de respuesta
| Código | Significado |
|--------|-------------|
| 200 | Pago exitoso |
| 400 | Datos inválidos o pedido no en estado "Pendiente" |
| 402 | Tarjeta rechazada |
| 403 | El pedido no pertenece al usuario |
| 503 | Servicio de pedidos no disponible |
