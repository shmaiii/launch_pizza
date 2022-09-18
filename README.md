# Launch-Pizza API

The functionality will be splitted into 2 ends - one end for the customers to send, track and see receipt of their orders, the other end for the pizza workers to monitor to see current orders and update status of each order. 

## Tools: 
Here the backend is built using Python's Django framework, which has a built-in database of sqlite.

## Models: 
The backend needs to be able to create objects and retrieve these data from its associated database. The models are in a file called model.py includes:
- Pizza: an object representing a pizza with fields like: name, description(ingredients), size, price, etc
- Status: status of the order with values as choices from a list of [ordered, processing, delivering, complete]
- Order: represents an order made by the customer and seen by the pizza store with fields like: order_time (default set to now), pizza (a ForeignKey towards the pizza model), status (a ForeignKey towards status model), user (a ForeignKey pointing to a user)

## Customer side:

### Ordering:
From the client side of the customer, they will be able to add pizza to their carts. Hitting the button "Order" will create an event thay triggers a fetch API call:

```

fetch(`/send_order/`, {
    method: 'POST',
    headers: {
        'X-CSRFToken': getCoookie('csrftoken),
    }
    body: JSON.stringify({
        pizza: pizza,
        user: get_current_user(),  // assuming there is a function that retrieves the user in the session
    })
})

```

This sends a message to the backend. The backend will receive the info and handle it like:

```
def send_order(request):
    if request.method == 'POST':
        data = json.loads(request.body)
        pizza = data.get("pizza", "")
        user = data.get("user", "")

        order = Order(pizza=pizza, status="ordered", user=user)
        order.save()
        
        return JsonResponse({
            "message": "Order successfully"
        }, status = 201)
    
    return JsonResponse({
        "message": "Some errors occur"
    }, status = 400)

```

### Tracking:

The customer can go to a tracking page. This page makes a GET request to retrieve all orders that belong to the current customer that has a status of anything but "complete"

```
fetch(`/get_tracking_orders/${user.id}`)
.then (reponse => response.json())
.then (orders => setOrders(orders));
```

The backend side can return some JSonResponse like:

```
def get_tracking_orders(request, user_id):
    user = User.objects.get(pk=user_id)
    orders = Order.objects.filter(user=user).exclude(status="complete")

    return JsonResponse([order.serialize() for order in orders], safe=False)
```

### Seeing past receipts up to one year:

The customer can go to a past orders page which makes a GET request to retrieve all orders that have a status of complete

```
fetch('get_past_orders/${user.id}`)
.then (response => response.json())
.then (orders => setOrders(orders));

```

The django backend can return all of the order within the current year

```
def get_past_orders(request, user_id):
    user = User.objects.get(pk=user_id)
    orders = Order.objects.filter(user=user, order_time__year = datetime.datetime.now().year)

    return JsonResponse([order.serialize() for order in orders], safe=False)
```

## Pizza store side:

The pizza store should be able to view all current orders with a GET request, similar to the ones above. In addition to that, pizza workers should be able to update the status of an order for the customer to track. This can be done through a PUT request:

```
fetch(`update_status/${order_id}`, {
    method: "PUT",
    body: JSON.stringify({
        status: new_status,
    })
})

```