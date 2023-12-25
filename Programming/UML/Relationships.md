Relationships describe how entities interact or collaborate within the problem domain. 

1. Dependency:
    
    - Dependency represents a relationship where one entity depends on another entity for some functionality or information.
    - It indicates that a change in one entity may affect the other entity.
    - For example, a Customer entity may depend on a PaymentGateway entity to process payments.
	![[dependency.png]]

1. Aggregation:
    
    - Aggregation represents a "whole-part" relationship between entities, where one entity is composed of or contains other entities.
    - It implies that the aggregated entity has a lifecycle independent of the whole entity.
    - For instance, in a Car Rental domain, a Rental entity may aggregate multiple RentalItems (such as cars and accessories).

1. Composition:
    
    - Composition is a stronger form of aggregation, indicating a strong lifecycle dependency between entities.
    - It implies that the composed entity cannot exist independently of the whole entity.
    - For example, in a Music Library domain, a Playlist entity may compose multiple Song entities. If the Playlist is deleted, the Songs associated with it would also be deleted.
4. Inheritance:
    
    - Inheritance represents an "is-a" relationship between entities, where one entity inherits attributes and behaviors from a more general entity.
    - It allows entities to inherit and extend the properties and functionalities of a parent entity.
    - For instance, in a Banking domain, a CheckingAccount entity may inherit from a more general Account entity, sharing common attributes and behaviors.
   ![[inheritance.png]]



