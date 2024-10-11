---
title: "iOS UI Architecture - MVC to MVVM and VIPER"
date: 2024-10-11
---
Tags: UIKit, MVC, MVVM, VIPER

For a simple view or feature, MVC is the out of the box solution. This architecture uses UIKit View Controller as the Controller component to control, modify, and update both UI and data models. However, it is not a good option for situations like when you want to make the view reusable across multiple experiences. The more features you add to a view, the more bloated its View Controller gets (new components/states/actions), which is a maintainability issue.

Let's check other popular architecture and see how they would solve this issue.

### MVVM - Model View ViewModel
#### What is MVVM
Design pattern which function around 3 distinct components: Models, Views, View Models.
- Models have all the required data.
- View models convert the data in the model into values that can be displayed on a view.
- Views control the visual elements and controls on the screen
- ViewModel provide a bidirectional data channel between the view and the rest of the app.
  - It takes in user interactions from the view and delegates it to the app
  - It listens to notifications from the app and update the models and the view

#### Advantages of MVVM over MVC:
- Reduces the responsibilities of VC by keeping it isolated to views and separate out the business logic.
- We can achieve binding between the view and the view model using various different patterns (Functional Reactive Programming, KVO, custom delegates.. )
  - Reduces a lot of boilerplate code, helps reduce the work being done by the view controller (which is causing it to a massive layer), provides more asynchronous reactive updates.
- Testing with MVVM architecture is much easier than MVC

