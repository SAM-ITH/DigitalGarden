- domain model is a conceptul representation of specific subject area or domain. 
- Elements of a domain model 
	- Entities - fundamental concepts or objects in the domain 
	- Attributes - properties or characteristics of entities 
	- [[Relationships]] - how entities are connected with each other 
	- [[Associations]] - represent the connections between entities in the domain
	- Behaviours - actions or operations that entities can perform with in the domain 

### example 

domain model of a online book store 

Entities:

1. Customer: Represents a customer who can browse and purchase books.
    
    - Attributes: Name, Email, Address
    - Behaviors: Register, Login, PlaceOrder, MakePayment
2. Book: Represents a book available for sale.
    
    - Attributes: Title, Author, ISBN, Price
    - Behaviors: GetDetails, AddToCart
3. Order: Represents a customer's order.
    
    - Attributes: OrderNumber, Date, TotalAmount
    - Behaviors: AddItem, RemoveItem, CalculateTotalAmount

Relationships and Associations:

1. Customer-Order Relationship: A customer can place multiple orders.
    
    - Association: One Customer can have multiple Orders.
    - Cardinality: Customer (1) to Order (0 to many).
2. Order-Book Relationship: An order can contain multiple books, and a book can be part of multiple orders.
    
    - Association: One Order can have multiple Books, and one Book can be in multiple Orders.
    - Cardinality: Order (1) to Book (0 to many).