- kalkulator is a cool looking calculator create by using swift and UIkit storyboards. 

#### Objective
- main objective of this project is to learn UIkit and Swift. this going to be submit for the app store as my first iOS app. 

#### Plan to launch 
- [x] Create the technical doc by watching a tutorial.
	- [ ] Focus on creating the class diagram and structure of the code
- [ ] Creating the UI using storyboards
- [ ] Attaching the UI components to view controller 
- [ ] Coding the logic
- [ ] Polishing the UI 
- [ ] Testing 
- [ ] Add sounds to the button press and animation
- [ ] Get the apple developer ID 
- [ ] Submit app for review

### Notes on structure of the code 
- we label all the button with tag so all buttons are going to call the same function so value will be shown according to that 

```swift 
if resultLabel.text == "0" 
{
resultLabel.text = "\(tag)"
}
else if let text = resultLabel.text {
	  resultLabel.text = "\(text)\(tag)"
}
``` 
- we have to add a tag to every button 
```swift 
button1.tag = 1 
```
- in the tutorial its adding target for the button i think in my case that won't be necessary ( min 23.41)
- we are dealing with the operation as same way that we identified the number, by using tags 

#### Tutorial 
<iframe src="https://www.youtube.com/embed/QsLHrEUCvVQ"></iframe>
