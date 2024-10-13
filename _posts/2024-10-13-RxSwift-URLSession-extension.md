---
title: "RxSwift+URLSession extension"
date: 2024-10-13
---

### Initial version

```Swift
import RxSwift

extension URLSession {
  static func jsonObservable(
    for request: URLRequest,
    delegate: URLSessionDelegate) -> Observable<Any>? {

    let session = URLSession(configuration: URLSessionConfiguration.default, delegate: delegate, delegateQueue: nil)
    let maxAttemptCount = 3
    return session.rx.json(request: request, retryExponentially(maxAttemptCount: maxAttempCount)
  }
}

```

### Generic version
```Swift
protocol DataTaskObservable {
    associatedtype ResponseData
    
    // TODO: RocalService logic should be migrated out of that class so
    // we the extra dependencies here of operationName and service.
    func observable(for request: URLRequest,
                    delegate: RocalErrorHandler & URLSessionDelegate,
                    service: RocalService,
                    operationName: String) -> Observable<ResponseData>
}
 
struct RocalTaskObservable<R: RocalModel>: DataTaskObservable {
    typealias ResponseData = R
    
    func observable(for request: URLRequest,
                    delegate: RocalErrorHandler & URLSessionDelegate,
                    service: RocalService,
                    operationName: String) -> Observable<R> {
        return URLSession.dataTaskObservable(for: request,
                                             delegate: delegate,
                                             service: service,
                                             operationName: operationName)
    }
}

extension URLSession {
    static func dataTaskObservable<R>(for request: URLRequest,
                     delegate: RocalErrorHandler & URLSessionDelegate,
                     service: RocalService,
                     operationName: String) -> Observable<R> where R: RocalModel {
        let serviceClass = type(of: service)
        let session = URLSession(configuration: URLSessionConfiguration.default,
                                 delegate: delegate,
                                 delegateQueue: nil)
        let strategy = ExponentialBackoffRetryStrategy { (error) -> Bool in
            guard let error = error as? DataTaskPublishingError else { return false }
            
            if case let .responseError(responseError) = error {
                let castedError = responseError as NSError
                
                return castedError.shouldWaitBetweenRetries()
            } else {
                return false
            }
        }
        
        return session.rx.response(request: request)
            .map({ (response, data) -> (response: URLResponse, data: Data) in
                if let error = response.am_retryableServiceError() {
                    throw DataTaskPublishingError.responseError(error)
                }
                
                return (response, data)
            })
            .retryExponentially(strategy: strategy)
            .map({ (response, data) -> R in
                guard let urlResponse = response as? HTTPURLResponse else {
                    throw DataTaskPublishingError.invalidResponse
                }
                guard let descriptor = serviceClass.descriptor(for: operationName) else {
                    throw DataTaskPublishingError.responseParsing
                }
                
                var rocalResponse: RocalResponse?
                
                do {
                    rocalResponse = try RocalResponse(descriptor: descriptor,
                                                      data: data,
                                                      urlResponse: urlResponse,
                                                      requestSigner: nil)
                } catch {
                    delegate.handleRocalError(error, service: serviceClass.serviceName())
                }
                
                if let responseObject = rocalResponse?.object as? R {
                    return responseObject
                }
                
                throw DataTaskPublishingError.responseParsing
            })
            .do(onDispose: {
                session.invalidateAndCancel()
            })
    }
}

```


