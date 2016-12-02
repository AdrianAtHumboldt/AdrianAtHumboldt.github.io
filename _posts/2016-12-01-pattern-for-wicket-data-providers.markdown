---
published: false
title: A Pattern for Wicket Data Providers
layout: post
tags: [java]
---
Apache Wicket provides the [AjaxFallbackDefaultDataTable](https://ci.apache.org/projects/wicket/apidocs/8.x/org/apache/wicket/extensions/ajax/markup/html/repeater/data/table/AjaxFallbackDefaultDataTable.html) for displaying a paged view of a large data set, and this component requires an implementation of [ISortableDataProvider](https://ci.apache.org/projects/wicket/apidocs/8.x/org/apache/wicket/extensions/markup/html/repeater/data/table/ISortableDataProvider.html) to populate the rows. This data provider has several responsibilities:

* Keep track of the current sort order.
* Store any search parameters in a `Serializable` form.
* Fetch a count of rows that match the search parameters
* Provide an iterator over a subset of rows that match the search parameter, ordered by the current sort.

If you are satisfied with sorting on a single column at any time, the best approach is to extend the abstract class [SortableDataProvider](https://ci.apache.org/projects/wicket/apidocs/8.x/org/apache/wicket/extensions/markup/html/repeater/util/SortableDataProvider.html), which handles all the work involved in tracking the single sort property. 

As the data provider must be `Serializable`, it cannot reference the database directly,  but must find a database connection via some type of service locator. I've used [Wicket-Spring](https://ci.apache.org/projects/wicket/guide/8.x/single.html#_integrating_wicket_with_spring) to inject a repository object that provides interfaces that map closely to the `ISortableDataProvider` interface, and to leave all search and sorting logic to the repository. A single instance of the repository object can be shared between many data providers, so the search and sorting parameters have to be passed in with every request:

{% highlight java %}
@Component
public class ClientSearchBackend  {

    public long count(@Nonnull ClientSearchParams filter) {
        // Carry out database query
        ....
    }

    public List<ClientData> list(long first, long count, 
                                 @Nonnull ClientSearchParams filter, 
                                 @Nonnull String sortparam, boolean ascending) {
        // Carry out database query
        ....
    }
}
{% endhighlight %}

The data provider is then only responsible for passing parameters to the repository. Note that this particular data provider returns a `Serializable` summary object which can be stored directly in a model.  More complex return types might benefit from wrapping the data in a `LoadableDetachableModel`.

{% highlight java %}
public class ClientSearchProvider extends SortableDataProvider<ClientData, String> {

    private final ClientSearchParams filter = new ClientSearchParams ();

    @SpringBean
    private ClientSearchBackend backend;

    public ClientSearchProvider () {
        // As this is not a Component, it must trigger @SpringBean injection manually
        Injector.get().inject(this);
        setSort("accountNumber", SortOrder.ASCENDING);
    }

    public ClientSearchParams getFilter() {
        return filter;
    }

    @Override
    public Iterator<? extends ClientData> iterator(long first, long count) {
        // getSort() can be null if table unsorted. Default to 
        Optional<SortParam<String>> sort = Optional.ofNullable(getSort());
        return backend.list(first, count, filter,
                sort.map(SortParam::getProperty).orElse("accountNumber"),
                sort.map(SortParam::isAscending).orElse(true))
                .iterator();
    }

    @Override
    public long size() {
        return backend.count(filter);
    }

    @Override
    public IModel<ClientData> model(ClientData clientData) {
        return new Model<>(clientData);
    }
}
{% endhighlight %}

While the data provider could access a database connection directly via `@SpringBean`, this would make unit test of pages containing the data provider extremely difficult. Instead, unit tests can simply mock `ClientSearchBackend ` and return fake data for the `count` and `list` calls.

For completeness, the Kotlin version of the data provider looks like this. Mostly similar, but handling the possibility of a null `sort` property is less verbose:


{% highlight kotlin %}
class ClientExtendedDataProvider : SortableDataProvider<ClientData, String>() {

    @SpringBean
    private val backend: ClientExtendedBackend? = null

    val filter = ClientExtendedFilter()

    init {
        Injector.get().inject(this)
        setSort("accountNumber", SortOrder.ASCENDING)
    }

    override fun iterator(first: Long, count: Long): Iterator<ClientData > {
        return backend!!.list(first, count, filter,
                sort?.property ?: "accountNumber",
                sort?.isAscending() ?: true)
                .iterator()
    }

    override fun size(): Long {
        return backend!!.count(filter)
    }

    override fun model(clientData: ClientData ): IModel<ClientData > {
        return Model(clientData)
    }
}
{% endhighlight %}