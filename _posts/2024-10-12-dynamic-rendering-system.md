---
title: "Dynamic Rendering"
date: 2024-10-12
---

Dynamic Rendering architecture is invented based on these motivations
- Server-driven rendering
- Avoid subclassing view controllers. Some hierachies have ummanageable depth
- Lack of clear separation of concerns: Many concerns, even if themselves have a good separation of concerns, are put into a view controller, which blurs out those individual architectural boundaries.
- Code duplication: this is a consequence of some of the points above, but is a big enough problem to be listed separately

## DR architecture
The starting point of DR is the `DynamicPageViewController` class. It needs an object that conform to `ContentProvider` which will supply it with views.
```Swift
let vc = DynamicPageViewController(contentProvider: ColorsContentProvider())
```
The key takeaways here are
- We use the same view controller to render different kinds of content by giving it different content providers
- We don't need one view controller per screen

### Content Provider
The functionality to render views are now hosted in `ContentProvider` instead a view controller.
A view controller needs a data source, and the content provider can provide it. A content provider implements data source methods. The data source must conforms to `PageRendererCallbacksHandler`.

```Swift
/*
callout(PageRendererCallbacksHandler):
   is literally defined as:
 
   `UICollectionViewDataSource & UICollectionViewDelegateFlowLayout`
*/

class SimpleContentProvider: NSObject, ContentProvider, PageRendererCallbacksHandler {
  func callbacksHandler(for collectionView: UICollectionView) -> PageRendererCallbacksHandler {
    collectionView.register(UICollectionViewCell.self, forCellWithReuseIdentifier: "foo")
    return self
  }

  func collectionView(_ collectionView: UICollectionView, numberOfItemsInSection section: Int) -> Int {
    return 5
  }

  func collectionView(_ collectionView: UICollectionView, cellForItemAt indexPath: IndexPath) -> UICollectionViewCell {
    let cell = collectionView.dequeueReusableCell(withReuseIdentifier: "foo", for: indexPath)
    cell.backgroundColor = UIColor.green
    return cell
  }
}

let vc = DynamicPageViewController(contentProvider: SimpleContentProvider())
```

### Section
ContentProvider is a good first step to bring data source stuff out of a VC. It however does not allow us to combine different data sources. E.g. we can't put cities and colors on the same page easily. Section solves this issue.

`Section` is a model representation for a group of items. You can think of it as a model object for a section in a collection view or table view, or a view model in MVVM terminology. A section must provide the following information at minimum:
- `count` (number of items withint the section)
- `id`
- `header` (optional)
- `footer` (optional)

This allows for a more declarative way of handling layout and can be serialized to an over-the-wire format (JSON) thus making way for server-driven rendering.

```Swift
struct CitiesSection: Section {
  var id: String = "cities"
  var cities: [String]
  var count: Int {
    return cities.count
  }

  init(cities: [String]) {
    self.cities = cities
  }
}
```

Every `Section` needs a `SectionRenderer`






