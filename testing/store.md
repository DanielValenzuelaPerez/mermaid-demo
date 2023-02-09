# Store
## Cart & Cart Items
### Creación de carrito de compras automáticamente
El carrito de compras se crea automáticamente al añadir algún artículo. Es decir, si no existe carrito de compras, se crea uno y se añade el artículo; si ya hay un carrito de compras, entonces se agrega el artículo al carrito.

```mermaid
flowchart LR
    cart_exists{cart exists?}
    create_cart[[create cart]]
    CART_ITEM-->|add item|cart_exists
    cart_exists-->|yes, add item|CART
    cart_exists-->|no|create_cart
    create_cart-->|add item|CART
```

#### Endpoint para añadir artículos:
```
[POST]
/store/cart_item
```

#### Payload para añadir artículos:
```
{
    presentation: int,
    model: int,
    quantity: int
}
```

### Un solo carrito de compras activo por usuario
Sólo puede existir un carrito de compras activo por usuario. Esto se controla por medio de los estados en los que pueda estar un carrito.

    1. DRAFT
    2. RESERVED
    3. COMPLETED

*Nota. Los artículos del carrito comparten el mismo estado que el carrito de compras al que pertenecen.*

```mermaid
erDiagram
    USER||--|{CART : has
    CART||--|{CART_ITEM : contains
    CART {
        int status
        int user_id
    }
    CART_ITEM {
        int status
        int cart_id
        int available_id
        int quantity
    }
```

**DRAFT** es el estado en el que empiezan los carritos de compras. En este estado se pueden agregar y editar artículos. También es posible regresar a este estado si la orden de pago falla.

Cuando un carrito de compras pasa a **RESERVED**, todos los artículos se congelan y ya no es posible agregar más artículos. Esto pasa cuando se llama el endpoint* para crear la orden de pago.

##### * Revisar la sección **Orden de Pago**
El carrito de compras pasa a **COMPLETED** cuando la orden de pago es pagada.

### Cómo funciona agregar artículos

El endpoint para agregar artículos requiere 3 valores en el payload: presentación, modelo y cantidad.

Al momento de recibir una petición, el sistema busca y agrupa todos los availables que tengan la misma presentación y modelo, y que pertenezcan a almacenes que vendan en línea; similar a la manera en la que el sistema le muestra al cliente el total de productos entre todos los almacenes, en lugar de mostrar productos repetidos. La idea es que el cliente pueda agregar artículos al carrito de compras sin tener que revisar disponibilidad por almacén.

```mermaid
graph TD
    subgraph AVAILABLES["AVAILABLES"]
        A[id=1<br>presentation_id=121<br>model_id=34<br>quantity=5<br>warehouse_id=A<br>sells_online=True]
        B[id=2<br>presentation_id=121<br>model_id=34<br>quantity=8<br>warehouse_id=B<br>sells_online=True]
        C[id=3<br>presentation_id=121<br>model_id=34<br>quantity=10<br>warehouse_id=C<br>sells_online=False]
    end

    subgraph STORE_VIEW["STORE VIEW"]
        D[presentation_id=121<br>model_id=34<br>quantity=13]
    end

    A-->|join availables|D
    B-->|join availables|D
```

Por ejemplo, supongamos que hay dos almácenes que venden en línea (A y B), donde ambos tienen un producto X con la misma presentación y modelo.
- Almacén A tiene 5 productos X
- Almacén B tiene 8 productos X

*Nota. Producto X corresponde a dos Availables que comparten el mismo modelo y presentación, pero pertenecen a diferentes almacenes.*

En la tienda en línea, se mostrarán en total 13 productos X disponibles. El usuario puede seleccionar 10 y agregarlos a su carrito de compras. El sistema agregará entonces 8 unidades del almacén B y 2 del almacén A en el carrito de compras del usuario.

Carrito de compras del cliente:
- Item 1: { Product: X, Warehouse B, Quantity: 8 }
- Item 2: { Product: X, Warehouse A, Quantity: 2 }

```mermaid
graph TD
    endpoint["/store/cart_item"]-->|presentation=121<br>model=34<br>quantity=10|AVAILABLES

    subgraph AVAILABLES["FILTERED AVAILABLES"]
        A[warehouse_id=A<br>available_id=1<br>quantity=5<br>sells_online=True]
        B[warehouse_id=B<br>available_id=2<br>quantity=8<br>sells_online=True]
    end


    subgraph CART_ITEMS["CART ITEMS"]
        I_1["ITEM 1"<br>available_id=2<br>quantity=8]
        I_2["ITEM 2"<br>available_id=1<br>quantity=2]
    end

    A-->I_2
    B-->I_1

    subgraph CARTS
        C_1["CART 1"<br>status=COMPLETED]
        C_2["CART 2"<br>status=COMPLETED]
        C_3["CART 3"<br>status=DRAFT]
    end

    I_1 --> C_3
    I_2 --> C_3
```
*Nota. En este ejemplo, la llamada al endpoint para agregar un artículo al carrito de compras resultó agregando dos artículos por falta de disponibilidad en un almacén.*

*Notas adicionales. Almacén C no aparece porque no vende en linea. Las órdenes de Cart 1 y 2 ya fueron completadas.*

## Order

### Orden de pago

Las órdenes en la tienda en línea se crean a partir de los artículos asociados al carrito de compras.

```mermaid
erDiagram
    USER||--|{CART : has
    CART||--|{CART_ITEM : contains
    CART||--||ORDER : has
    CART {
        int status
        int user_id
    }
    CART_ITEM {
        int status
        int cart_id
        int available_id
        int quantity
        int price
    }
    ORDER {
        int status
        int cart_id
        int total
    }
```

#### Para crear una orden se hace una llamada al endpoint:
```
[POST]
/store/order
```
*Nota. No es necesario mandar payload; se crea a partir del usuario quien haga la llamada.*

#### Los estados de una orden de la tienda en línea son:

    1. DRAFT
    2. PROCESSING
    3. PAID
    4. CONFIRMED
    5. SENT
    6. DELIVERED
    7. CANCELED
    8. TIME OUT
    9. FAILED

Una orden comienza en estado **DRAFT**, donde es solamente para visualizar el total de la orden. En este estado la orden aún puede ser modificada.

La orden pasa a **PROCESSING** al momento de actualizar el estado de la orden en el endpoint de la orden.

#### Endpoint para actualizar la orden a PROCESSING:

```
[PUT]
/store/order/:id
```

#### Payload para actualizar la orden a PROCESSING:

```
{
    "status": 2
}
```

*Nota. Al intentar actualizar a PROCESSING el sistema puede responder con algunos errores; ver sección Posibles errores al querer crear una orden de pago.*

#### Respuesta exitosa de actualizar orden a PROCESSING:
```
{
    "payment_request_id": "27ed09f1-b491-4692-865b-f9731f9c3208",
    "object_type": "payment_link", 
    "status": "CHECKOUT_CREATED", 
    "last_status_message": "Payment Request created succesfully", 
    "created_at": "2023-02-01T18:27:30Z", 
    "payment_request_url": "https://completa-tu-pago2.payclip.com/27ed09f1-b491-4692-865b-f9731f9c3208", 
    "modified_at": "2023-02-01T18:27:30Z", 
    "expired_at": "2023-02-04T18:27:30Z"
}
```

Al cambiar a **PROCESSING**, se genera una enlace de pago (payment_request_url) para pagar en CLIP.  En este momento, el carrito de compras y los artículo se congelan para que no puedan modificarse los artículos mientras el cliente realiza el pago.

El cliente tendrá que realizar su pago entrando al enlace de pago generado por CLIP.

Cuando se pague, CLIP mandará un Webhook al sistema indicando que la orden fue pagada exitosamente. Al recibir esta llamada, el sistema pasará el carrito de compras y los artículos a COMPLETED. También pasará la orden a **PAID**.

```mermaid
sequenceDiagram
    actor CLIENT
    CLIENT->>SYSTEM: [POST] /store/order
    Note right of SYSTEM: query cart items<br>and create order
    SYSTEM-->>CLIENT: return order
    CLIENT->>SYSTEM: [PUT] /store/order/:id
    Note right of CLIENT: status=2
    Note right of SYSTEM: validate items availability,<br>user data, and<br>freeze cart & items
    SYSTEM->>CLIP: request payment order
    CLIP-->>SYSTEM: return payment order
    SYSTEM-->>CLIENT: return payment order
    CLIENT->>CLIP: Pay order
    CLIP-->>SYSTEM: Order paid successfully
    Note right of SYSTEM: update order, cart & items
```

### Posibles errores al querer crear una orden de pago

Para crear una orden de pago, el usuario necesita tener guardado su número de teléfono y una dirección física.

**Error: Por favor agregue una dirección física a su perfil.**
```
[POST]
/common/address_user
```

**Error: Por favor agregue un correo electrónico y un número de telefono a su perfil.**
```
[PUT]
/common/customer/{id}
```
