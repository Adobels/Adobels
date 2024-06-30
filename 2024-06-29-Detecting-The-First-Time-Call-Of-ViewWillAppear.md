# Detecting the First-Time Call of viewWillAppear

In the realm of iOS development, it is a common practice to execute UI first-time configurations within methods such as viewWillAppear, viewDidAppear, updateViewConstraints, viewWillLayoutSubviews, and viewDidLayoutSubviews. A widely known approach to determine if a specific method is executing for the first time involves creating a private Boolean variable, often named isYourFunctionNameExecutingForFirstTime.

```swift
class Toto: UIViewController {
    
    private var isViewWillAppearExecutingForFirstTime: Bool = true

    override func viewWillAppear(_ animated: Bool) {
        super.viewWillAppear(animated)
        if isViewWillAppearExecutingForFirstTime {
            // code to execute only on first execution of viewWillAppear
            isViewWillAppearExecutingForFirstTime = false
        } else {
            // Code to execute on any execution other than the first of viewWillAppear
        }
    }
}
```

Pros of This Approach:
- **Simplicity:** The implementation is easy to write and understand
- **Ideal for Proof-of-Concept Projects:**

Cons of This Approach:
- **Redundancy:** Each method requiring this logic must have its own local variable.
- **Risk of Misuse:** These variables can be inadvertently used in other functions, either by mistake or by design.
- **Not Ideal for Pro Multi-Dev Projects:** Integrating this solution for each new method requires considerable administrative work, such as creating issues, reviewing pull requests, and merging. This is especially challenging in projects with multiple developers and strict version control practices.

### Pro & Cons in Example:
```swift
class FirstViewController: UIViewController {
    
    // Redundancy:
    private var isViewWillAppearExecutingForFirstTime: Bool = true
    private var isViewDidAppearExecutingForFirstTime: Bool = true
    
    override func viewDidLoad() {
        super.viewDidLoad()
        if isViewWillAppearExecutingForFirstTime {
            // Risk of Misuse: this variable should not be used here
        } else {
            // Code to execute on any execution other than the first
        }
    }

    override func viewWillAppear(_ animated: Bool) {
        super.viewWillAppear(animated)
        if isViewWillAppearExecutingForFirstTime {
            // Code to execute only on first execution of viewWillAppear
            // The line below must be called as the last one.
            isViewWillAppearExecutingForFirstTime = false
        } else {
            // Code to execute on any execution other than the first
        }
    }
    
    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)
        if isViewDidAppearExecutingForFirstTime {
            // Code to execute only on first execution of viewDidAppear
            isViewDidAppearExecutingForFirstTime = false
        } else {
            // Code to execute on any execution other than the first
        }
    }
}
```
---

## Improving the solution to be more scalable and reusable
### Reusing the first-time call detection logic in AppViewController

In some projects, you may find an AppViewController class that encapsulates reusable logic shared by its instances. Let's move the first-time call detection logic into this class to make it widely available.
```swift
class AppViewController: UIViewController {
    
    var isViewDidAppearExecutingForFirstTime: Bool = true
    
    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)
        isViewDidAppearExecutingForFirstTime = false
    }
}

class Example_1_ViewController: AppViewController {

// In this implementation, calling super on the first line
// sets isViewDidAppearExecutingForFirstTime to false,
// causing the check if isViewDidAppearExecutingForFirstTime 
// to be skipped and the code inside it not to execute.
    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)
        if isViewDidAppearExecutingForFirstTime {
            let vc = MyViewController()
            vc.modalPresentationStyle = .fullScreen
            vc.view.backgroundColor = .yellow
            present(vc, animated: true)
        } else {
            // Code to execute on any execution other than the first
        }
    }
}

class Example_2_ViewController: AppViewController {

// Calling super at the last line of the function solves the problem.
// However, this is not a reliable solution because some functions
// require super to be called at the first line. Additionally,
// there are cases where the documentation states that calling
// super is optional due to an empty implementation of that function in the superclass.
    override func viewDidAppear(_ animated: Bool) {
        if isViewDidAppearExecutingForFirstTime {
            let vc = MyViewController()
            vc.modalPresentationStyle = .fullScreen
            vc.view.backgroundColor = .yellow
            present(vc, animated: true)
        } else {
            // Code to execute on any execution other than the first
        }
        super.viewDidAppear(animated)
    }
}
```

Moving the first-time call detection code to AppViewController revealed that handling cases triggered by calling the superclass in overridden methods is challenging. Calling super at the first line of an overridden method leads to the first-time call block of code not executing at all. While calling super at the last line of an overridden method works, this position is less commonly used compared to calling super at the first line. Additionally, there are cases where calling super in an overridden method is optional due to an empty implementation in the superclass.

To improve the first-time call detection code, it must be independent of the super call's position. We can achieve this by using a computed property with a side effect. Each time this property is accessed, a counter increments. The property then compares the counter to the value 1, returns the result of this comparison, and ensures that the logic works regardless of where the super call is placed
```swift
class AppViewController: UIViewController {
    
    private var isViewDidAppearExecutingForFirstTimeCounter = 0

    var isViewDidAppearExecutingForFirstTime: Bool {
        isViewDidAppearExecutingForFirstTimeCounter += 1
        return isViewDidAppearExecutingForFirstTimeCounter == 1
    }
}
```
Now the first-time call detection code is independent of the execution of super and its position. Both Example 1 and Example 2 now work well.
### Scaling the first-time call detection logic to other view controllers
The first-time call detection logic was implemented and reused in UIViewController. However, for UITableViewController and UICollectionViewController, the solution must be copied and repeated.
### Unit Testing the first-time call detection logic
In professional multi-developer projects, that logic must be unit tested. The problem here is that the unit test logic will be repeated between tests of AppViewController, AppTableViewController, AppCollectionViewController, etc. To avoid repeating the unit test code, the first-time detection logic must be moved to a reusable type FunctionCallTracker.
```swift
class FunctionCallTracker {

    private var isLineOfCodeExecutingForFirstTimeCounter = 0

    var isLineOfCodeExecutingForFirstTime: Bool {
        isLineOfCodeExecutingForFirstTimeCounter += 1
        return isLineOfCodeExecutingForFirstTimeCounter == 1
    }
}

class AppViewController: UIViewController {
    
    private let viewDidAppearTracker = FunctionCallTracker()
    
    var isViewDidAppearExecutingForFirstTime: Bool {
        viewDidAppearTracker.isLineOfCodeExecutingForFirstTime
    }
}

class FirstViewController: AppViewController {

    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)
        if isViewDidAppearExecutingForFirstTime {
            // First-time call logic
        } else {
            // Code to execute on any execution other than the first
        }
    }
}
```
Now, the logic can be tested once and resused in many palces. However, integration testing of FunctionCallTracker within AppViewController and other view controllers must still be repeated.
### Adding First-Time Call Detection Logic to New Functions
Suppose we already have the viewWillAppear method supported with first-time call logic, and now we want to add support for the viewIsAppearing method. Here's how we can add support for the new function:
The issue now is that if we want to support this method in other view controllers, such as AppTableViewController and AppCollectionViewController, the same logic must be implemented in each of these classes, along with the necessary unit tests.
```swift
class AppViewController: UIViewController {
    
    private let viewWillAppearTracker = FunctionCallTracker()
    private let viewIsAppearingTracker = FunctionCallTracker()
    
    var isViewWillAppearExecutingForFirstTime: Bool {
        viewWillAppearTracker.isLineOfCodeExecutingForFirstTime
    }
    var isViewIsAppearingExecutingForFirstTime: Bool {
        viewIsAppearingTracker.isLineOfCodeExecutingForFirstTime
    }
}
```
Despite these improvements, the solution still has constraints. Integration tests are necessary, and you must add an instance property for FunctionCallTracker and a computed property for its isLineOfCodeExecutingForFirstTime variable each time you want to implement first-time call logic in new functions. While this approach is better, it is not yet optimal.

---
## The Solution
Thanks to macros #fileID, #line, and #column available in Swift, it is possible to create a universal, scalable, and unit-testable solution.

First, let's imagine how this new logic would be used in code.
```swift
class ViewController: AppViewController {

    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)
        if codeTracker.isFirstTimeCall(fileID: #fileID, line: #line, column: #column) {
            // first time logic
        } else {
            // Code to execute on any execution other than the first
        }
    }
}
```
Now let's adapt AppViewController and FunctionCallTracker with this API.
```swift
class AppViewController: UIViewController {
    let codeTracker = FunctionCallTracker()
}
```
First, let's eliminate the need for computed properties. The idea is that the mentioned macros can identify a specific place in the code with increasingly fine precision: from file, to line, to column.
```swift
class FunctionCallTracker {
    func isFirstTimeCall(fileID: String, line: String, column: String) -> Bool
}
```
Since the API now exists but lacks logic, let's develop it. Each line of code can be uniquely identified by three elements: fileID, line, and column. To determine if a line of code has been executed, we can store these identifiers. If an identifier is already in the store, it indicates that the line of code has been executed. We can use a set of strings for the store, as order and repetition are not important.
```swift
public class CodeCallTracker {

    private var store: Set<String> = []

    public init() { }

    public func isFirstTimeCall(_ fileId: String, _ line: Int, _ column: Int) -> Bool {
        let funcIdentifier = fileID + String(line) + String(column)
        if store.contains(funcIdentifier) {
            return false
        } else {
            store.insert(funcIdentifier)
            return true
        }
    }
}
```
Now let's simplify the calling side of the API by adding default values for the function arguments.
```swift
class FunctionCallTracker {

    private var store: Set<String> = []

    public init() { }

    func isFirstTimeCall(fileID: String = #fileID, line: Int = #line, column: Int = #column) -> Bool {
        let funcIdentifier = fileID + String(line) + String(column)
        if store.contains(funcIdentifier) {
            return false
        } else {
            store.insert(funcIdentifier)
            return true
        }
    }
}
```
which leads to a simpler call site:
```swift
class ViewController: AppViewController {

    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)
        if codeTracker.isFirstTimeCall() {
            // first time logic
        } else {
            // Code to execute on any execution other than the first
        }
    }
}
```
Now it's not a problem to add the codeTracker.isFirstTimeCall check in other functions.
```swift
class ViewController: AppViewController {

    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)
        if codeTracker.isFirstTimeCall() {
            // first time logic
        } else {
            // Code to execute on any execution other than the first
        }
    }
    
    override func viewIsAppearing(_ animated: Bool) {
        super.viewIsAppearing(animated)
        if codeTracker.isFirstTimeCall() {
            // first time logic
        } else {
            // Code to execute on any execution other than the first
        }
    }
}
```
The API is not limited to methods; it can be used anywhere.
```swift
class ViewController: AppViewController {

    var myComputedProperty: Bool {
        if codeTracker.isFirstTimeCall() {
            return true
        } else {
            return false
        }
    }
    
    var myComputedPropertyWithTwoChecksInTheSameColumn: Bool {
        var randomValue = [true, false].randomElement()!
        if codeTracker.isFirstTimeCall(), randomValue, codeTracker.isFirstTimeCall() {
            return true
        } else {
            return false
        }
    }
}
```
Because of its many usage options, let's rename FunctionCallTracker to CodeCallTracker.
```swift
class CodeCallTracker {

    private var store: Set<String> = []

    public init() { }

    func isFirstTimeCall(fileID: String = #fileID, line: Int = #line, column: Int = #column) -> Bool {
        let funcIdentifier = fileID + String(line) + String(column)
        if store.contains(funcIdentifier) {
            return false
        } else {
            store.insert(funcIdentifier)
            return true
        }
    }
}
```
## CodeCallTracker Framework
For more information, the CodeCallTracker framework, featuring the first-time call detection logic explained in the examples above, is available as a unit-tested Swift Package framework on GitHub at https://github.com/Adobels/CodeCallTracker.

This framework can be seamlessly integrated into proof-of-concept projects, single-developer projects, and professional multi-developer projects, ensuring reliable execution of code blocks that should run only once, as demonstrated in the UIViewController, UITableViewController, and UICollectionViewController examples discussed earlier.
