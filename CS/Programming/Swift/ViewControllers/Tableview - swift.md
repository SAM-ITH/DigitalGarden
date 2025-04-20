
- adding table view to the storyboard. 

## setup table view DataSource and Delegate 

- table view has two weak objects 
	- DataSource - object is responsible for providing cells and their data for a table view
	- Delegate - It handles selection, deletion, header/footer, and other related actions.

``` swift
    override func viewDidLoad() {
        super.viewDidLoad()
        
        tableView.delegate = self
        tableView.dataSource = self
    }
```	

- next we need to confirm our delgate and dataSource with our view Controller. This is because the delegate and dataSource objects allow assigning a class that must conform to their protocols.

``` 
class ViewController: UIViewController, UITableViewDelegate, UITableViewDataSource

```
- adding required method stubs 

> A method stub or simply stub in software development is a piece of code used to stand in for some other programming functionality.

- there are two method stubs that added by Xcode.
	- first number of row in section 
	- cell for row at 

1. First number of row in section return as Int

responsible for drawing the number of cells returned by this method.

``` swift
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        //TODO: Return items count
        return 0
    }

```

2. other method is cell for row at 

returns an object of UITableViewCell type. This is where we can create a cell and pass data to it to configure its UI.

``` swift
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        //TODO: Return cell
        return UITableViewCell()
    }
```

- we use count of the array as the return object of the first method stub.

``` swift
return array.count
```