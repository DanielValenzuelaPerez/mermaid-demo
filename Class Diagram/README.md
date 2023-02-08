# Class Diagram
```mermaid
classDiagram

    class Order {
        +OrderedStatus status
    }

    class OrderStatus {
        <<enumeration>>
        FAILED
        PENDING
        PAID
    }

    class PaymentProcessor {
        <<interface>>
        -String apiKey
        #connect(String url, JSON header)
        +processPayment(Order order) OrderStatus
    }

    class Customer {
        +String name
    }

    Customer <|-- PrivateCustomer
    Customer <|-- BusinessCustomer
    PaymentProcessor <|-- StripePaymentProcessor
    PaymentProcessor <|-- PaypalPaymentProcessor
    Order o-- Customer
    Car *-- Engine
```