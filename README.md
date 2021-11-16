# benefits of this

- You can standardize the concurrency and threading models and patterns and behaviors in your project.

# Code

```swift
//
//  ViewController.swift
//  SwiftMVVM_EventBus
//
//  Created by paige on 2021/11/16.
//

import UIKit

// MARK: - BUS
final class Bus {
    static let shared = Bus()
    
    private init() {}
    
    enum EventType {
        case userFetch
    }
    
    struct Subscription {
        let type: EventType
        let queue: DispatchQueue
        let block: (Event<Any>) -> Void
    }
    
    private var subscriptions = [Subscription]()
    
    // Subscriptions
    func subscribe(_ event: EventType, block: @escaping((Event<Any>)) -> Void) {
        let new = Subscription(type: event, queue: .global(), block: block)
        subscriptions.append(new)
    }
    
    func subscribeOnMain(_ event: EventType, block: @escaping((Event<Any>)) -> Void) {
        let new = Subscription(type: event, queue: .main, block: block)
        subscriptions.append(new)
    }
    
    // Publications
    func publish(type: EventType, event: Event<Any>) {
        let subscribers = subscriptions.filter({$0.type == type})
        subscribers.forEach { subscriber in
            subscriber.queue.async {
                subscriber.block(event)
            }
        }
    }
    
}

// MARK: - EVENT
class Event<T> {
    let identifer: String
    let result: Result<T, Error>?
    
    init(identifier: String, result: Result<T, Error>?) {
        self.identifer = identifier
        self.result = result
    }
}

// Sub-class of Events
class UserFetchEvent: Event<[User]> {
    let created = Date()
}

// MARK: - MODELS
struct User: Codable {
    let name: String
}

// MARK: - VIEWMODEL
struct UserListViewModel {
    
    public var users = [User]()
    
    func fetchData() {
        guard let url = URL(string: "https://jsonplaceholder.typicode.com/users") else {
            return
        }
        let task = URLSession.shared.dataTask(with: url) { data, _, _ in
            guard let data = data else { return }
            let event: UserFetchEvent
            do {
                let users = try JSONDecoder().decode([User].self, from: data)
                event = UserFetchEvent(identifier: UUID().uuidString, result: .success(users))
            }
            catch {
                print(error)
                event = UserFetchEvent(identifier: UUID().uuidString, result: .failure(error))
            }
            Bus.shared.publish(type: .userFetch, event: event)
        }
        task.resume()
    }
}

// MARK: - VIEW
class ViewController: UIViewController, UITableViewDelegate, UITableViewDataSource {
        
    private var viewModel = UserListViewModel()
    
    private let tableView: UITableView = {
       let table = UITableView()
        table.register(UITableViewCell.self, forCellReuseIdentifier: "cell")
        return table
    }()

    override func viewDidLoad() {
        super.viewDidLoad()
        view.addSubview(tableView)
        tableView.frame = view.bounds
        tableView.delegate = self
        tableView.dataSource = self
        Bus.shared.subscribeOnMain(.userFetch) { [weak self] event in
            switch event.result {
            case .success(let users):
                self?.viewModel.users = users
                self?.tableView.reloadData()
                break
            case .failure(let error):
                print(error)
            case .none:
                print("")
            }
        }
        
        viewModel.fetchData()
    }
    
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return viewModel.users.count
    }
    
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: "cell", for: indexPath)
        cell.textLabel?.text = viewModel.users[indexPath.row].name
        return cell
    }
    
}
```
