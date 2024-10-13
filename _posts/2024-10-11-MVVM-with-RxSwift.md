---
title: "MVVM with RxSwift"
date: 2024-10-11
---
TODO: Add more view model(s). One view controller can hold a strong reference to several view models and also bind to them if wanted.
TODO: Add gestures type: tapped and swipe

An MVVM design where bindings between ViewModel and View Controller is done with RxSwift. App/System notifications are wrapped in RxSwift Observables.

### View Controller
```Swift
import RxSwift

extension UIViewController {
  // a class representing a view's life-cycle events
  // each state is both an observable and an observer (PublishSubject)
  public class ViewLifeCycle {
    let loaded = PublishSubject<Void>()
    let willAppear = PublishSubject<Void>()
    let willDisappear = PublishSubject<Void>()
  }
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

  override func loadView() {
    view = MyView()
  }

  override func viewDidLoad() {
    super.viewDidLoad()

    viewLifeCycle.loaded.onCompleted()
  }

  override func viewWillAppear(_ animated: Bool) {
    super.viewWillAppear(animated)
    viewLifeCycle.willAppear.onNext(())
  }

  override func viewWillDisappear(_ animated: Bool) {
    super.viewWillDisappear(animated)

    viewLifeCycle.willDisappear.onNext(())
  }

  private func bind(to viewModel: MyViewModel) {
    disposeBag = DisposeBag()
    let input = MyViewModel.Input(viewLifeCycle: viewLifeCycle)

    viewModel.getOutput(from: input)
      .subscribe({ [weak self] event in // use weak to prevent potential retaining cycle when using disposeBag below
        guard let self = self else { return }
        guard let state = event.element else { return }

        self.render(state) // TODO: implement this below
      })
      .disposed(by: disposeBag)
  }

  private func render(_ state: MyViewModel.State) {
  
  }

}

class MyView: UIView {}

```

### View Model
A `ViewModel` contains a state object which is the data source for a view.
It also contains methods to set up bindings between a View Controller's lifecycle events (wrapped inside `Input`) and Observable(s) that emitting system/app events.
It means that for each View Controller life-cycle event such as `viewDidLoad`, we can start observing a collection of events of our choice and update our view model or view accordingly.

```Swift
import RxSwift

class MyViewModel {
  // The state of the view; TODO: extend this?
  struct State {}

  // The inptut from the view
  struct Input {
    let viewLifeCycle: UIViewController.ViewLifeCycle
  }

  // `Output` drives the view to re-render itself
  typealias Output = Observable<State>

  private var disposeBag = DisposeBag()
  // Use notification + RxSwift to receive system/app events wrapped in observables
  private let notificationCenter: NotificationCenter
  private var state: State {
    return State()
  }

  init(notificationCenter: NotificationCenter = NotificationCenter.default) {
     self.notificationCenter = notificationCenter
  }

  func getOutput(from input: Input) -> Output {
    disposeBag = DisposeBag()

    // Get an observable that provides system/app notifications when the view did load
    let viewDidLoadNotificationObservable = input.viewLifeCycle.loaded
      .asObservable()
      .flatMap(self.viewDidLoadNotificationObservables)

    return Observable
      .merge([
        viewDidLoadNotifcationObservable
      ])
      .flatMapLatest { (_) -> Output in
        return .just(State()) // Every time a notification is received, the brand new state object is created and return to the view/view controller
    }

  }

  // Notifications we want to listent to after viewDidLoad() has occurred
  private func viewDidLoadNotificationObservables() -> Observable<Notification> {
    return Observable.merge([
      notificationCenter.rx.notification(.castingStatusChanged),
      notificationCenter.rx.notification(UIApplication.userDidTakeScreenshotNotification)
    ])
  }

  // Notifications we only want to listen to after viewWillAppear()
  private func viewWillAppearNotificationObservables() -> Observable<Notification> {
    return Observable.merge([
      notificationCenter.rx.notification(UIApplication.didEnterBackgroundNotification),
      notificationCenter.rx.notification(UIApplication.didBecomeActiveNotification)
    ])
  }
  
}
```

