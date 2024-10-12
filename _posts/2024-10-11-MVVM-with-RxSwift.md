---
title: "MVVM with RxSwift"
date: 2024-10-11
---

View Controller

```Swift
import RxSwift

extension UIViewController {
  // a class representing states of a view's life cycle
  // each state is both an observable and an observer (PublishSubject)
  public class ViewLifeCycle {
    let loaded = PublishSubject<Void>()
    let willAppear = PublishSubject<Void>()
    let willDisappear = PublishSubject<Void>()
}

class MyViewController: UIViewController {
  private var disposeBag = DisposeBag()
  private let viewModel: MyViewModel
  private let viewLifeCycle = ViewLifeCycle()

  init(viewModel: MyViewModel = MyViewModel()) {
    self.viewModel = viewModel
    super.init(nibName: nil, bundle: nil)

    // set up the reactive binding to view model
    bind(to: viewModel)
  }

  private func bind(to viewModel: MyViewModel) {
    disposeBag = DisposeBag()

    
  }
}

}


```

ViewModel

