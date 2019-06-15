---
layout: post
title:  "Data driven dataSource"
subhead: "Empower your dataSources with some flexibility"
date:   2019-06-15 10:35:32 +0100
categories: ["cocoa"]
---

Creating a tableView is a common task for any iOS developer and this also calls for creating a dataSource.
Which implies any improvement to the process of creating and maintaining thoses dataSource can be a huge amount of time saved.

# Creating a DataSource class or conforming the viewController to the dataSource protocol

Both of theses method are valid and have their pros and cons. Let's try and compare!
The quickest way is inevitably conforming your viewController to the dataSource protocol : you just have to implement the methods you need and your ready to go. But that his the only advantage of using this method. Creating a dataSource class as the benefit of having a reusable dataSource code, which means when two tableView look enough like each other then their dataSources can be two instances of the same class and do not need to be written twice. It also enable to declutter your viewController and avoid the massive viewController syndrom. It also enables to have multiple dataSources for multiple tableViews in your viewController, which granted this is not a frequent case.

To me the choice is pretty much obvious to which solution should be promoted.

# Reusing DataSources

Now that we have decided creating a class is better option and mostly because we are going to be able to reuse it, we have to try and make it as reusable as possible.
At some point in our carreer we have all seen or written a dataSoure that look like this

~~~swift
     private var data: Data   

    // Mark - UITableViewdatasource
    
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return 1
            + (data.shouldDisplayBCell ? 1 : 0)
    }
    
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        switch indexPath.row {
        case 0:
            let cell: ATableViewCell = tableView.dequeueCell(at: indexPath)
            cell.configure(with: data.aCell)
            /* configure cell A according to our need */
            return cell
        case 1:
            let cell: BTableViewCell = tableView.dequeueCell(at: indexPath)
            cell.configure(with: "b cell title")
            /* configure cell B according to our need */
            return cell
        default:
            assertionFailure("This case should not happen")
            return UITableViewCell()
        }
    }
~~~
This is ~~fine~~ bearable, but there is now way to reuse a dataSource like this for different tableViews, moreover this is not maintainable at all.
In the above exemple our `Data` looks something like this :
~~~swift
struct Data {
    let shouldDisplayBCell: Bool
    let aCell: ACellModel
}
~~~
Which is a problem because what we did not anticipate is that our `BTableViewCell` in a second screen is the same but just need a different title.
This is not going to be a problem for long because we juste have to replace our boolean value by a `String?` representing the title of our button, and voilà.
That way our DataSource class fits both use cases and wider customisation of the tableView through its `Data` input
~~~swift
struct Data {
    let aCell: ACellModel
    let bCell: String?
}
~~~
Unfortunately after a while this is not good enough anymore because we now need to display multiple ATableViewCells, so lets tweak our `Data` struct once again to be able to handle any case we might encounter.
~~~swift
struct Data {
    let aCell: [ACellModel]
    let bCell: String?
}
~~~
For the sake of it let's push this even further : let's suppose we need multiple `ATableViewCell` followed by the same number of `BTableViewCell` but also in a different tableView multiple `BTableViewCell` followed by one `ATableViewCell`

I would be easy to replace `bCell` by an array of `String` so we can have multiple `ATableViewCell` and multiple `BtableViewCell`, but it's not good enough because we cannot cover both or first and second use case. With this model there is no way for our data source class to figure out which cells come first.

# Rich enum to the rescue :

Let's create a CellViewModel which can be either a **ATableViewCellViewModel** or a **BTableViewCellViewModel** :
~~~swift
enum MyCellModel {
    case a(ATableViewCellModel)
    case b(BTableViewCellModel) // BTableViewCellViewModel aka String?
}
~~~

So now our ViewModel has become :
~~~swift
struct Data {
    let cells: [MyCellModel]
}
~~~

🤔 written like this I cannot resist the urge to make it a generic struct
~~~swift
struct TableViewModel<CellModel> {
    let cells: [CellModel]
}
~~~

How did all this changes have affected my `DataSource` class

~~~swift
     private var data: Data<MyCellModel> 

    // Mark - UITableViewdatasource
    
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return data.cells.count
    }
    
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        switch data.cells[indexPath.row] {
        case let .a(model):
            let cell: ATableViewCell = tableView.dequeueCell(at: indexPath)
            cell.configure(with: model)
            /* configure cell A according to our need */
            return cell
        case let .b(model):
            let cell: BTableViewCell = tableView.dequeueCell(at: indexPath)
            cell.configure(with: model)
            /* configure cell B according to our need */
            return cell
        }
    }
~~~
Ok this is great in the process we even got rid of the awkward assertionFailure. But still this as many short commings :
what if I need multiple sections, a header, a footer?

# Creating a generic TableViewModel

Let's do this by starting with creating a generic ViewModel for our sections: 
~~~swift
struct TableSectionViewModel<HeaderFooterViewModel, CellViewModel> {
    
    let header: HeaderFooterViewModel?
    let cells: [CellViewModel]
    let footer: HeaderFooterViewModel?
}
~~~
now that we have sections lets create the complete TableViewModel
~~~swift
struct TableViewModel<HeaderFooterViewModel, CellViewModel> {
    typealias Section = TableSectionViewModel<HeaderFooterViewModel, CellViewModel>

    let sections: [Section]
}
~~~
It is that simple. Maybe now we can add some syntactic sugar to gain readability.
~~~swift
struct TableSectionViewModel<HeaderFooterViewModel, CellViewModel> {

    init(header: HeaderFooterViewModel? = nil,
         cells: [CellViewModel],
         footer: HeaderFooterViewModel? = nil) {
        self.header = header
        self.cells = cells
        self.footer = footer
    }

    init() {
        self.init(cells: [])
    }

    var header: HeaderFooterViewModel?
    var cells: [CellViewModel]
    var footer: HeaderFooterViewModel?
}

struct TableViewModel<HeaderFooterViewModel, CellViewModel> {
    typealias Section = TableSectionViewModel<HeaderFooterViewModel, CellViewModel>

    var sections: [Section]

    var headers: LazyMapCollection<[Section], HeaderFooterViewModel?> { return sections.lazy.map { $0.header } }
    var footers: LazyMapCollection<[Section], HeaderFooterViewModel?> { return sections.lazy.map { $0.footer } }

    init(sections: [Section]) {
        self.sections = sections
    }

    init(section: Section) {
        self.init(sections: [section])
    }

    init(cells: [CellViewModel]) {
        let section = Section(cells: cells)
        self.init(section: section)
    }

    init() {
        self.init(sections: [])
    }

    subscript(indexPath: IndexPath) -> CellViewModel {
        return sections[indexPath.section][indexPath.row]
    }
}
~~~

Okay so now we have what seems to be a versatile generic TableViewModel lets see what will become our DataSource.
For starter lets create a typealias for `TableViewModel<CellViewModel>` that way we can manipulate this type and style have a readable code.
~~~swift
typealias MytableViewModel = TableViewModel<Never, CellViewModel>
/* We do not have any header or footer, so we use Never to enforce it */

class DataSource: NSObject, 
    UITableViewDataSource {

    private var viewModel = MytableViewModel()

    func configure(with viewModel: MyViewModel) {
        self.viewModel = viewModel		
    }

    // MARK: - UICollectionViewDataSource

    func numberOfSections(in collectionView: UICollectionView) -> Int {
        return viewModel.sections.count
    }

    func collectionView(_ collectionView: UICollectionView, numberOfItemsInSection section: Int) -> Int {
        return viewModel.sections[section].cells.count
    }

    func collectionView(_ collectionView: UICollectionView,
                        cellForItemAt indexPath: IndexPath) -> UICollectionViewCell {
        switch viewModel[indexPath] {
        case let .a(cellViewModel):
            let cell: ATableViewCell = tableView.dequeueCell(at: indexPath)
            cell.configure(with: cellViewModel)
            return cell
        case let .b(cellViewModel):
            let cell: BTableViewCell = tableView.dequeueCell(at: indexPath)
            cell.delegate = self
            cell.configure(with: cellViewModel)
            return cell
    }
}
~~~
This is neat now we can display any number of cells both of type A or B in any order. And if we want to add another type of cell, we just have to add another case to the CellViewModel. 
We could try and refactor this so all of our DataSource classes do not have to reimplement those `numberOfSections(in:)` `collectionView(_:numberOfItemsInSection:)` functions, but let's leave it to that for today.