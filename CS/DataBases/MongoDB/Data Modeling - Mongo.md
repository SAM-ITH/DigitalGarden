- this note is written reference to a mongoDB data modeling lecture 
##### questions to ask from your self before modeling the database 
- what does my application do?
- what data will i store ?
- how will users access this data?
- what data will be the most valuable to me?

`Data that is accessed together should be stored together` - principals of mongoDB 
### Type of data relationships 
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
### Modeling data relationships 
###### Embedding
taking the related data and insert it into our document. 
###### referencing 
refer to documents in another collection in our document. in below SS take look at the filming locations 
![[Mongo - referencing.png]]

### Embedding data in documents. 
- embedding is mainly using when we have one to many or many to many relationships in the data that being stored. 
- examples of data embedding in the document. in first example name has been embedded as first name and the last name.
```python
	{name:
	{firstName: "Sarah", lastname: "Davis"} 
	}
``` 

![[Embeded - mongoDB.png]]

##### issues that can be arrises when using embedded data models. 
- embedding data into single document can create a large document with over the time. this can be lead to use of excessive memory and add latency to quarries because large documents should read into memory in full. 
- continuously adding data without limit creates unbounded documents this can be exceed the maximum data size of the BSON file of 16mb.
### Referencing data in documents
- referencing mean saving the ID field of one document in another document as a link between two. 
- Using references is called **linking** or **Data normalization**.
- below is an example of referencing used in a document 
	![[referencing example - mongo.png]]
### pro's and con's of embedding vs. referencing
![[Embd vs referencing.png]]




