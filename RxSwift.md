## RxSwift相關
- 新版Rx有這個回收方法
```
_ =
sequence
    .take(until: self.rx.deallocated)
    .subscribe {
        print($0)
    }
```

- 檢查MemoryLeak, 升級到6.2
```
在Pods -> RxSwift-> building setting -> Other Swift Flags -> Debug新增 -D TRACE_RESOURCES
可使用RxSwift.Resources.total
```
- URLSession extensions don't return result on MainScheduler by default.
- 新的decode: .decode(type: [User].self, decoder: JSONDecoder())
- observable:withUnretained(self), 但無法優雅處理onError & onComplete，因為還是要加[weak self]:
```
observable
        .withUnretained(self)
        .subscribe { (owner, string) in
            owner.doSomething(string)
        } onError: { [weak self] (error) in
            guard let self = self else { return }
            self.handleError(error)
        } onCompleted: { [weak self] () in
            guard let self = self else { return }
            owner.handleDone()
        }
        .disposed(by: bag)
```
此時可改用Subscribe(with:):
```
observable
        .subscribe(with: self) { (owner, string) in
            owner.doSomething(string)
        } onError: { (owner, error) in
            owner.handleError(error)
        } onCompleted: { (owner) in
            owner.handleDone()
        }
        .disposed(by: bag)
```
drive不適用, 可改用.subscribe(with: <#T##Object#>, onNext:...(observable & drive皆適用)
```
/*You can also use withUnretained with other RxSwift operators to avoid the weak self retain cycle issue:*/
func setupLogin(_ submit: Observable<(String?, String?)>) -> Observable<User> {
    return submit
        .withLatestFrom(credentials)
        .withUnretained(self)
        .flatMap { (model, creds) -> Observable<String> in
            model.authorize(username: creds.0, password: creds.1)
        }
        .withUnretained(self)
        .flatMap { (model, token) -> Observable<User> in
            model.user(authorization: token)
        }
}
```
- Infallible(不會出錯，只有.next(element)跟.completed), 跟Driver&Signal差別: Driver跟Signal只使用在MainScheduler，而且share元素(share()); Infallible為單純不會出錯observable
- bind & drive 可多重binding
```
viewModel.string.bind(to: input1, input2, input3)
viewModel.string.drive(input1, input2, input3)
viewModel.number.emit(input4, input5)
```
- distinctUntilChange 可用Key Paths
```
//origin:
myStream.distinctUntilChanged { $0.searchTerm == $1.searchTerm }

//new:
myStream.distinctUntilChanged(at: \.searchTerm)
```
- new ReplayRelay
```
ReplayRelay<Int>.create(bufferSize: 3)
```
- DisposeBag新的使用方式，用Wrapper的方式將subscriber包起來
```
var disposeBag = DisposeBag {
     observable1.bind(to: input1)
    observable2.drive(input2)
    observable3.subscribe(onNext: { val in 
        print("Got \(val)")
    })
}

// Also works for insertions
disposeBag.insert {
    observable4.subscribe()
    observable5.bind(to: input5)
}
```
- `Observable:Single`改回傳`Result<Element, Swift.Error>`
- Rx 6.5 可與async/await合用, [Reference](https://github.com/ReactiveX/RxSwift/blob/main/Documentation/SwiftConcurrency.md)
- [What's new in RxSwift 6 ?](https://dev.to/freak4pc/what-s-new-in-rxswift-6-2nog)
- [Using withUnretained in RxSwift 6.0](https://betterprogramming.pub/using-withunretained-in-rxswift-6-0-8e3e221b37ee)