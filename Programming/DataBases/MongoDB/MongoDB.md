- mongoDB database structured data as documents. (similar to JSON objects)
- Documents correspond to objects in code. 
- Hierarchy of the MongoDB 
	- DataBase
	- Collection 
	- Document 
### Document Model 
- mongoDB documents are displayed in JSON format but they store in BSON. 
	- compare to JSON BSON are supported additional data types. 
###### Object IDs 
- A data type that can be used to create a unique id for the required id filed. 
- All the documents need to have a object id. 

- MongoDB support polymorphic data. 

[[Data Modeling - Mongo]]
[[Connecting to a MongoDB Database]]

### MongoDB CRUD Operations 

- Replaced a single document by using `db.collection.replaceOne()`
- Updated a field value by using the `$set` update operator in `db.collection.updateOne()`.
- Added a value to an array by using the `$push` update operator in `db.collection.updateOne()`.
- Added a new field value to a document by using the `upsert` option in `db.collection.updateOne()`.
- Found and modified a document by using `db.collection.findAndModify()`.
- Updated multiple documents by using `db.collection.updateMany()`.
- Deleted a document by using `db.collection.deleteOne()`.
-  insertOne()
	- `db.collection.insertOne()`
### MongoDB CRUD Operations: Modifying Query Results

In this unit, you learned how to modify query results with MongoDB. Specifically, you learned how to:

- Return query results in a specified order by using `cursor.sort()`.
    
- Constrained the number of results returned by using `cursor.limit()`.
    
- Specified fields to return by adding a projection document parameter in calls to `db.collection.find()`.
    
- Counted the number of documents that match a query by using `db.collection.countDocuments()`.











