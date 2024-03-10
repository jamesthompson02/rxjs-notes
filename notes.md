# RxJS Notes

## Vid 1 - RxJS in Angular: Terms, Tips and Patterns

### Observables

A lot of RxJS is designed to deal with observables. Observables are just a collection of items/data that are emitted over time. Once they have been emitted, there is no memory of them - they cannot be retained and stored to use later.

Things in an application that can be converted into an observable are:

1. Key presses of a user (when typing in an input)
2. Mouse movement
3. Mouse clicks
4. HTTP responses

RxJS is a useful technology for reacting to these data emissions.

Observables have 3 main types of notifications: next (when next item/datum is emitted), error (when an error occurs) and complete (when it’s finished emitting). A code sample below shows how the next function is used:

```
    export class WidgetEventService {
    private _widgetEvents$ = new Subject<WidgetEvent>();

    public widgetEvents$ = this._widgetEvents$.asObservable();

    public dispatch(event : WidgetEvent) {
    this.widgetEvents$.next(event);
    }
    }

```

Angular has some bultin features which rely on Observables. For example, a FormControl emits values via a valueChanges observable and the http Client emits the response as an observable that can be subscribed to.

Observables by themselves have no impact on an application. To have an impact, they need to be subscribed to or they need to be rendered in the HTML template of a component using the async pipe (e.g. `“photos$ | async”`).

Note that if you subscribe to an observable in Angular, you should include an ngOnDestroy lifecycle method which unsubscribes from the observable in the event that the component is destroyed. This prevents memory issues from occurring.

### Pipes

You can pipe observables so that each emitted item can be transformed/manipulated to your requirements. There are dozens of RxJS operators used inside pipes to perform data manipulation which will be discussed later on.

### Thinking about RxJS

A lot of how to use RxJS in your application can be summarised by asking 3 questions:

1. What do you have?
2. What do you want?
3. When do you want it?

### Classic Pattern for retrieving data: service (procedural) + component

A classic pattern (in the context of an HTTP response) for supplying data to a component is to follow 4 steps:

1. Create a service which contains a function which fetches the data and returns that data.
2. Inject that service into the relevant component.
3. Declare a property (above the constructor in the component) which is intended to have the data assigned to it.
4. Call and subscribe to the function in the service so that the value of the data can be assigned to the property that you created in step 3. Ideally, you would write this code in the ngOnInit lifecycle method of the component (makes unit testing easier).

Example Implementation:

**Service**:

```
    @Injectable({ providedIn: 'root' })
    export class ProductService {

        private productsUrl = 'api/products';

        constructor(private http: HttpClient) { }

        getProducts(): Observable<Product[]> {
            return this.http.get<Product[]›(this.productsUrl)
            .pipe(
            tap(data => console.log(JSON.stringify(data))), catchError (this.handleError)
            );
        }

    }
```

**Component**:

```
export class ProductListComponent implements OnInit, OnDestroy {

    products: Product[];
    sub: Subscription;

    constructor (private productService: ProductService) { }

    ngOnInit (): void {
        this.sub = this.productService.getProducts ().subscribe(
        products => this.products = products
        );
    }
    ngOnDestroy(): void {
        this.sub.unsubscribe();
    }

}

```

### Rendering Observables in the HTML of a component

When rendering an observable in the HTML template, you will want to use the async pipe.

An example of how to do this is as follows:

```

<div *ngIf=“products$ | async as products”>
@for(product of products; track index){
  <button>
     {{product.name}}
  </button>
}
</div>

```

### Retrieve on Action Pattern

Sometimes you will want to render data that depends specifically on a user action. For example, on a webpage you may have a dropdown menu with different categories of products that the user can select from. Another example would be when the user is looking at some sort of table with pagination and clicks to move to a new page in the table. Depending on the action of the user, this will display different data on the page.

When you have a scenario like this, where you may have to pass specific data relating to the user’s action, you should use either a Subject or a BehaviorSubject.

### Subject and BehaviorSubject

Subjects/BehaviorSubjects are special types of observables that we create ourselves. Example implementations of both are as follows:

```

private categorySubject = new Subject<number>();
categorySelectedAction$ = this.categorySubject.asObservable();

selectedCategoryChanged(categoryId: number): void {
this.categorySubject.next(categoryId);
}

private categorySubject = new BehaviorSubject<number>(1);
categorySelectedAction$ = this.categorySubject.asObservable();

```

From the first 4 lines of the example above, it should be clear how a user action would lead to an observable being emitted. The data related to the user action would be passed into the function (selectedCategoryChanged) and then emitted using the next method. That Subject value then gets converted into an Observable.

There are some differences between Subjects and BehaviorSubjects. First, you can emit a default value at the beginning of the BehaviorSubject’s existence (the number 1 in the example above) - you cannot do this with a Subject. Second, if you subscribe to a BehaviorSubject you can always get the last emitted value of the BehaviorSubject - you also cannot do this with a Subject.

### Higher Order Mapping

To continue from the previous example, imagine you had a piece of code that looks like this:

```
private categorySubject = new Subject<number>();
categorySelectedAction$ = this.categorySubject.asObservable();

products$ = this.categorySelectedAction$.pipe(
map(catId => this.http.get‹Product[]>(${this.productsUrl}?cat=${catId}'))
);

```

The problem is that inside the map operator on the penultimate line you have an inner observable. In a situation where you have an inner observable:use a higher-order mapping operator to subscribe to the inner observable as well as flatten the result.

Higher-order mapping operators have a number of characteristics:

1. They automatically subscribe to inner observables
2. They flatten the resulting observable. This means that rather returning Observable<Observable<T>> they return Observable<T>.
3. They automatically unsubscribe from the inner observable when finished.

3 higher-order mapping operators are as follows:

1. Switchmap. Stops the current operation and performs the new operation.
2. ConcatMap. Performs each operation one at a time, in order.
3. Mergemap. Performs each operation concurrently.

Switchmap is an ideal higher-order mapping operator to use when doing things based on user actions (e.g. selecting dropdown menu options or typing into an input). Given the potential for the user to keep changing their mind, every time the user does something new the current operation would get cancelled and a new operation gets performed instead.

### Shape on Action Pattern

Tip: to work with multiple streams of data, use a combination operator.

2 common combination operators are:

1. combineLatest. Emits a combined value when any of the Observables emit. Won’t emit until all Observables have emitted at least once. Combine latest typically takes an array of observables - you would then do some array de-structuring later on to access the observable values later on.
2. merge. Emits the one value when any of the observables emit. Note though this operator should only be used for streams of data that are alike (e.g. combining data related to current customers with data about past customers).

An example on how to use combineLatest is as follows:

```
selectedProduct$ = combineLatest([
this.products$, this.productSelectedAction$
]).pipe(
map(([products, selectedProductId]) =>
products.find(product => product.id === selectedProductId)
));
```

### Retrieve Related Data Pattern (One and Many)

**Many (not the best implementation)**:

```
productSuppliers$ = this.selectedProduct$
.pipe(
    switchMap(product => from(product.supplierIds)
    .pipe(
        mergeMap(supplierId =› this.http.get<Supplier>('${this.sUrI}/${supplierId}')),
        toArray())
    )
);
```

Note that mergeMap is used because all supplierIds would be wanted in this case, not just the most recent one. Consequently, a mergeMap gets used rather than, for example, a switchmap.

**Many (recommended implementation)**:

```
productSuppliers$ = this.selectedProduct$
.pipe(
    switchMap(product =>
        forkJoin(product.supplierIds.map(supplierId => this.http.get‹Supplier>(${this.sUr1}/${supplierId}')))
));

```

## Vid 2 - I only ever use _these_ RxJS operators to code reactively

### map (lowercase ‘m’ important)

The map operator applies a transformation of some sort. An example is as follows:

```
numbers$ = from([1,2,3,4,5]).pipe(
map((number) => number \* 10)
)
```

### filter (lowercase ‘f’ important)

The filter operator only allows data to pass through that satisfy a condition. An example is as follows:

```
numbers$ = from([1,2,3,4,5]).pipe(
map((number) => number \* 10),
filter((number) => number > 30)
)
```

### tap (lowercase ’t’ important)

Tap operators do not actually do anything to the stream of data/values that get emitted. It can be used for debugging purposes or to perform some side effects. For example, a tap operator could be used to console.log something or it could be used to assign the data value to a property in an Angular component. An example is as follows:

```
canActivate() {
    return this.authService.user$.pipe(
        map ((user) => (user ? false : true)),
        tap((canActivate) => {
            if (!canActivate) {
            this.navCtrl.navigateForward('/home');
            }
        })
    );
}
```

### switchMap

The switchMap operator may be a wise choice to use when faced with the prospect of dealing with/combining multiple streams of data. The switchMap operator allows you to switch from one stream to another. Switchmap is useful to use when your arrow function involves another observable (after the arrow) - the switchMap operator will also flatten the result. Another key point to understand about switchMap is its cancelling behaviour - should a new value be emitted whilst data is still being fetched for the previous emission, it will cancel the current operation and start a new operation based on the new value that’s been emitted. Here is an example of how it works:

```
feedback$ = this.route.paramMap.pipe(
    switchMap( (params) =>
    this.clientsStore.feedbacks$.pipe(
        map( (feedbacks) =>
        feedbacks
        ? feedbacks.find((feedback) => feedback.id === params.get(‘id’))
        : null
        )
    )
    )
)
```

### concatMap

The concatMap operator has similarities with the switchMap operator. One major difference, however, is that it does not have the same cancelling behaviour as switchMap. What this means is that if a new value gets emitted and the current operation is still being executed, it will wait until the current operation is executed before beginning a new operation based on the new value that’s been emitted. If a lot of values got emitted very quickly, this could create a very lengthy queue of operations to perform.

### mergeMap

The mergeMap operator has a lot in common with the concatMap operator. One major difference between mergeMap and concatMap, however, is that mergeMap should be used when the order that operations get executed in does not matter. To make this clear, imagine that 3 values got emitted in short succession. Imagine that the operation for the second value got executed a lot more quickly than the operation for the first value. If that has no impact on what you’re trying to do, then use mergeMap.

### combineLatest

The combineLatest operator does not need to be piped. It is used to combine multiple observables together in the form of an array. After they are combined, your next step will be to do some array de-structuring to get access to the multiple observables.

### startWith (used a lot in forms)

Allows you to emit a default value.

### distinctUntilChanged (used a lot in forms)

This ensures that every data emission is different from the previous.

### debounceTime (used a lot in forms)

This allows you to specify a minimum amount of time between emissions. debounceTime(1000) means you have to wait at least a second before the next value gets emitted.

### catchError

This operator is used to handle errors in your stream of data. One useful operator to combine catchError with is retryWhen. The retryWhen operator allows you to restart the stream when an action has been performed. Here is an example:

```
public getClients() {
    const clientsCollection = collection(this.firestore, 'clients');
    return collectionData(clientsCollection, { idField: 'id' }).pipe(
        // Emit null before erroring to clear client data in store, then rethrow error
        catchError((err) => concat(of(null), throwError(err))),
        // Restart stream when user logs back in
        retryWhen((errors) =>
        errors.pipe(
            switchMap(() =>
            this.authService.getLoggedIn().pipe(filter((user) => !!user))
        )
    ) as Observable<Client []>;
}
```
