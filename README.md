# Day-71
### **Bucket List project part 4**

***Downloading data from Wikipedia***

Wikipedia's API allows us to send in GPS coordinates and it will give us back a list of places that are nearby. JSON data will be sent back precisely, so we need to do a little work to define Codable structs capable of storing it all.

1. The main result contains the result of our query in a key called "query".
2. Inside the query is a "pages" dictionary, with page IDs as the key and the Wikipedia pages themselves as values.
3. Each page has a lot of information, including its coordinates, title, terms and more.

This all can be represented in structures like this:

```
struct Result: Codable {
    let query: Query
}

struct Query: Codable {
    let pages: [Int: Page]
}

struct Page: Codable {
    let pageid: Int
    let title: String
    let terms: [String: [String]]?
}
```

We also need something to show while we fetch the data, and this can be done using enums. And change the state when something happens.

```
// Add in EditView
enum LoadingState {
    case loading, loaded, failed
}

// In our form
Section("Nearby…") {
    switch loadingState {
    case .loaded:
        ForEach(pages, id: \.pageid) { page in
            Text(page.title)
                .font(.headline)
            + Text(": ") +
            Text("Page description here")
                .italic()
        }
    case .loading:
        Text("Loading…")
    case .failed:
        Text("Please try again later.")
    }
}
```

We use '+' in this case to add text view together and allowing to format each part differently.

Lastly, we need to get data from Wikipedia. As this func is async, we can call it using .task() modifier + await keyword.

```
func fetchNearbyPlaces() async {
    let urlString = "https://en.wikipedia.org/w/api.php?ggscoord=\(location.coordinate.latitude)%7C\(location.coordinate.longitude)&action=query&prop=coordinates%7Cpageimages%7Cpageterms&colimit=50&piprop=thumbnail&pithumbsize=500&pilimit=50&wbptterms=description&generator=geosearch&ggsradius=10000&ggslimit=50&format=json"

    guard let url = URL(string: urlString) else {
        print("Bad URL: \(urlString)")
        return
    }

    do {
        let (data, _) = try await URLSession.shared.data(from: url)

        // we got some data back!
        let items = try JSONDecoder().decode(Result.self, from: data)

        // success – convert the array values to our pages array
        pages = items.query.pages.values.sorted { $0.title < $1.title }
        loadingState = .loaded
    } catch {
        // if we're still here it means the request failed somehow
        loadingState = .failed
    }
}
```

