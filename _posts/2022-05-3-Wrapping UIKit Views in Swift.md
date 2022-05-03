layout: post
title: "Wrapping UIKit views in SwiftUI"
date: 2022-05-03 hh:mm:ss -0000
categories: Swift

# Wrapping UIKit views for SwiftUI

SwiftUI includes the ability to wrap existing UIKit views such as WKWebView into SwiftUI by using UIViewRepresentable.  As most apps were built using UIKit before SwiftUI came around, this backwards compatibility can be great for filling in gaps in behaviour while Apple works to add all of the necessary view kinds to SwiftUI.  One such view type is the WKWebView, SwiftUI does not currently have a way of embedding a WebKit view.  Wrapping the UIKit solution is an option that works well.

# UIViewRepresentable

```swift
protocol UIViewRepresentable : View where Self.Body == Never
```

It is a protocol that implements a basic SwiftUI View allowing it to be placed within other SwiftUI views and declared in the same way.

### Self.Body == Never?

In most SwiftUI views the body property will return a View.  In the case of a Button with a Image used as a label, the button returns the View from the label (the Image).  Eventually the View bodies have to stop otherwise it would go on calling body forever.  Primitive views such as Text, Image, Spacer cannot have subviews in the way a Button can, so these views are where the body recursion will end.  These primitive views all have bodies of type Never.  In the same way that an Image cannot have subviews, UIKit views can also not contain SwiftUI Views.  Therefore they also have a body of type Never.

# Using UIViewRepresentable

As show below we can simply place UIViewRepresentable views alongside other SwiftUI Views, even including them in VStacks.  This brings the SwiftUI ease of declarative UI to existing UIKit views.

```swift
struct SwiftWebView: UIViewRepresentable {
    var url: URL
    
    func updateUIView(_ webView: WKWebView, context: Context) {
        // Not current used, can react to SwiftUI state changes
    }
    
    func makeUIView(context: Context) -> WKWebView {
        let view = WKWebView()
        
        let request = URLRequest(url: url)
        view.load(request)
        
        return view
    }
}
```

```swift
struct MyView: View {
	var body: some View {
		VStack {
			Text("Hello World!")
			MyColorWellView()
			Text("Goodbye World!")
		}
	}
}
```

As you can see this WKWebView is fully interactable and embedded between the two SwiftUI Text Views.  However we only had to add one line of code to our parent view in order to achieve this.  While getting all the benefits of an older WKWebView.

![Image.png](https://res.craft.do/user/full/16a7b6a2-c41e-fe56-150b-e24b32001329/doc/566AC233-7F5E-4425-A537-D77FDEE44727/D34675C2-7E55-461C-8072-C9ED133E4A18_2/jzi73zfALFy5rjBSK6LWynKgi5tPQTH1XEOiru8zb2Ez/Image.png)

![Image.png](https://res.craft.do/user/full/16a7b6a2-c41e-fe56-150b-e24b32001329/doc/566AC233-7F5E-4425-A537-D77FDEE44727/2B9C0AA3-F06F-4000-9A1E-EDDC276A8A0D_2/BBqrVhxHP0Mx4CMkxDQoJHZlRBTHxzhHFGuh7DYbMUEz/Image.png)

# Expanding abilities with Coordinators

That's great we can now see the web page inside our SwiftUI ContentView.  But what about interfacing with the activity of the WebView.  For this we can use a Coordinator, this acts as a delegate for the UIKit view and so can handle events from the UIKit View.

Inside the coordinator we let it know what parent it belongs to, that way it can access and pass data to and from the parent SwiftUI View when UIKit events take place.

```swift
class SwiftWebCoordinator: NSObject, WKNavigationDelegate {
    let parent: SwiftWebView
        
    init(_ parent: SwiftWebView) { 
        self.parent = parent
    }
        
    func webView(_ webView: WKWebView, didFinish navigation: WKNavigation!) {
        parent.didFinish(webView: webView)
    }
}
```

We make sure to assign the navigationDelegate of the View to the coordinator, so it can receive events.

```swift
struct SwiftWebView: UIViewRepresentable {
    var url: URL
    
    @Binding var testBinding: String
    
    func updateUIView(_ webView: WKWebView, context: Context) {
        
    }
    
    func makeCoordinator() -> SwiftWebCoordinator {
        SwiftWebCoordinator(self)
    }
    
    func makeUIView(context: Context) -> WKWebView {
        let view = WKWebView()
        view.navigationDelegate = context.coordinator
        
        let request = URLRequest(url: url)
        view.load(request)
        
        return view
    }
    
    func didFinish(webView: WKWebView) {
        testBinding = webView.title ?? "Unknown"
    }
}
```

In this case we have added a @Binding to the SwiftWebView, this will allow parent views to pass in a state variable and have them be modified by the SwiftWebView.  This binding is updated when the UiKit Web View finishes loading a page, calling didFinish on the parent SwiftWebView.  In this case we have a loading text string that we pass as a binding to the SwiftWebView.

```swift
struct ContentView: View {
    @State var loadingText = "Loading"
    
    var body: some View {
        VStack {
            Text(loadingText)
            SwiftWebView(url: URL(string: "http://www.apple.com")!, testBinding: $loadingText)
        }
    }
}
```

![Image.png](https://res.craft.do/user/full/16a7b6a2-c41e-fe56-150b-e24b32001329/doc/566AC233-7F5E-4425-A537-D77FDEE44727/1D5509E7-64A6-4F68-98F5-5D7F0BF72602_2/z9xDTEglnKw7SlORxyNeffaJVyZCoJziuxOhNxLPvN0z/Image.png)

![Image.png](https://res.craft.do/user/full/16a7b6a2-c41e-fe56-150b-e24b32001329/doc/566AC233-7F5E-4425-A537-D77FDEE44727/FCB89098-140E-46E2-AB98-EE123B2C8217_2/PrV49dGxMjQy13zVE5jijVXwfYT14123CympUWkfekYz/Image.png)

The SwiftUI Text at the top of the screen uses the LoadingText string as input.  When the SwiftWebView finishes loading the page, the coordinator passes the current webView to the SwiftWebView and updates the binding to display the title of the loaded web page.  This then triggers a SwiftUI refresh on the @State var loadingText and updates the Text view at the top of the screen.

In this example, a coordinator implementing WKNavigationDelegate can also respond to events such as a failed load, a download link or can even ask permission to move to the next page in the website.

