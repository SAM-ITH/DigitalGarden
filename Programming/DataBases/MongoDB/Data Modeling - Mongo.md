- this note is written reference to a mongoDB data modeling lecture 
##### questions to ask from your self before modeling the database 
- what does my application do?
- what data will i store ?
- how will users access this data?
- what data will be the most valuable to me?

`Data that is accessed together should be stored together` - principals of mongoDB 
## Type of data relationships 
#### Relationship types
- One-to-one 
	- In mongoDB this can be done in a single document.  in below example title and director is one to one relationship and that has been model in a single document. 
```python
		"id" : ObjectID("23764872364823")
		"title" : "Batman : dark knight"
		"director" : "Christoper nolan"
```
- One-to-many
	- in below example it show many case members in a single movie that model in to single document
```python
		"id" : ObjectID("23764872364823")
		"title" : "Batman : dark knight"
		"cast" : [
		{
		"actor": "Christian Bale", "Character" : "Batman"
		"actor" : "Heath Ledger", "Character" : "Joker"
		}
		]
```
- Many-to-many
#### ways to model the relationships 
###### Embedding
taking the related data and insert it into our document. 
###### referencing 
refer to documents in another collection in our document. in below SS take look at the filming locations 
![[Mongo - referencing.png]]
