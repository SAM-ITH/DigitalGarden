- in swift exceptions are represented by instance of the `Error` protocol or its subtype. 
- we are using `do-catch` block to handle the exceptions. `Try` keyword is used before the call because it might throw an error. 
```swift 
do {

            let user = try await realmApp.login(credentials: .anonymous)

            username = user.id

        } catch {

            print("Failed to login to Realm : \(error.localizedDescription)")

        }
```