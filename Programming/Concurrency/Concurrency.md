> All following notes related to concurrency are written for swift language. but all the concepts are mostly universal among the other languages. 

### What is concurrency ? 
- concurrency is the execution of multiple instructions sequences at the same time .

### What is parallelism ? 
- parallelism is using multiple resource to execute the multiple instruction sequences at the same time. 
- for an example: there is two queues for the single hot dog stand. in parallelism we can use another new resource of hot dog stand to feed the second queue. 

## Concurrency vs parallelism
![[concurerncyVSparallelism.png]]

- in concurrency multiple tasks are appeared to be happen at the same time. but at the core only single instance working at a one time. 
- in parallelism truly tasks are happens in the same time. so they need to multiple resources to do it. as show in the above image. 

### How to achieve concurrency 

- [[Time Slicing and Context Switching]]

#### Achieving Multi Threading in iOS

- [[Manual Thread Creation]]






