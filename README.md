## Guide to app architecture

As Android apps grow in size, it's important to define an architecture that allows the app to scale, increases the app's robustness, and makes the app easier to test.

### Separation of concerns

### Drive UI from data models

### Single source of truth

>In an offline-first application, the source of truth for application data is typically a database. 
In some other cases, the source of truth can be a ViewModel or even the UI.

In other words: if the UI is what the user sees, the UI state is what the app says they should see.
the UI is the visual representation of the UI state. Any changes to the UI state are immediately reflected in the UI.

The classes that are responsible for the production of UI state and contain the necessary logic for that task are called state holders. 

As a result, you should expose the UI state in an observable data holder like LiveData or StateFlow.

```
class NewsViewModel(...) : ViewModel() {

    private val _uiState = MutableStateFlow(NewsUiState())
    val uiState: StateFlow<NewsUiState> = _uiState.asStateFlow()

    ...

}
```

or

```
class NewsViewModel(
    private val repository: NewsRepository,
    ...
) : ViewModel() {

    private val _uiState = MutableStateFlow(NewsUiState())
    val uiState: StateFlow<NewsUiState> = _uiState.asStateFlow()

    private var fetchJob: Job? = null

    fun fetchArticles(category: String) {
        fetchJob?.cancel()
        fetchJob = viewModelScope.launch {
            try {
                val newsItems = repository.newsItemsForCategory(category)
                _uiState.update {
                    it.copy(newsItems = newsItems)
                }
            } catch (ioe: IOException) {
                // Handle the error and notify the UI when appropriate.
                _uiState.update {
                    val messages = getMessagesFromThrowable(ioe)
                    it.copy(userMessages = messages)
                 }
            }
        }
    }
}
```

or

```
class NewsActivity : AppCompatActivity() {

    private val viewModel: NewsViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        ...

        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                // Bind the visibility of the progressBar to the state
                // of isFetchingArticles.
                viewModel.uiState
                    .map { it.isFetchingArticles }
                    .distinctUntilChanged()
                    .collect { progressBar.isVisible = it }
            }
        }
    }
}
```

or

```
data class LoginUiState(
    val isLoading: Boolean = false,
    val errorMessage: String? = null,
    val isUserLoggedIn: Boolean = false
)

class LoginViewModel : ViewModel() {
    private val _uiState = MutableStateFlow(LoginUiState())
    val uiState: StateFlow<LoginUiState> = _uiState.asStateFlow()
    /* ... */
}

class LoginActivity : AppCompatActivity() {
    private val viewModel: LoginViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        /* ... */

        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect { uiState ->
                    if (uiState.isUserLoggedIn) {
                        // Navigate to the Home screen.
                    }
                    ...
                }
            }
        }
    }
}
```

or

```
class LatestNewsViewModel(/* ... */) : ViewModel() {

    private val _uiState = MutableStateFlow(LatestNewsUiState(isLoading = true))
    val uiState: StateFlow<LatestNewsUiState> = _uiState

    fun refreshNews() {
        viewModelScope.launch {
            // If there isn't internet connection, show a new message on the screen.
            if (!internetConnection()) {
                _uiState.update { currentUiState ->
                    currentUiState.copy(userMessage = "No Internet connection")
                }
                return@launch
            }

            // Do something else.
        }
    }

    fun userMessageShown() {
        _uiState.update { currentUiState ->
            currentUiState.copy(userMessage = null)
        }
    }
}
```

or

```
@HiltViewModel
class AuthorViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle,
    private val authorsRepository: AuthorsRepository,
    newsRepository: NewsRepository
) : ViewModel() {

    val uiState: StateFlow<AuthorScreenUiState> = …

    // Business logic
    fun followAuthor(followed: Boolean) {
      …
    }
}
```

or

```
class MyViewModel(formatDateUseCase: FormatDateUseCase) : ViewModel() {
    init {
        val today = Calendar.getInstance()
        val todaysDate = formatDateUseCase(today)
        /* ... */
    }
}
```

## Domain layer

>The domain layer is an optional layer that sits between the UI layer and the data layer.
The domain layer is responsible for encapsulating complex business logic, or simple business logic that is reused by multiple ViewModels. 
This layer is optional because not all apps will have these requirements. 
You should only use it when needed-for example, to handle complexity or favor reusable.

```
class MyUseCase(
    private val defaultDispatcher: CoroutineDispatcher = Dispatchers.Default
) {

    suspend operator fun invoke(...) = withContext(defaultDispatcher) {
        // Long-running blocking operations happen on a background thread.
    }
}
```

or

```
/**
 * This use case fetches the latest news and the associated author.
 */
class GetLatestNewsWithAuthorsUseCase(
  private val newsRepository: NewsRepository,
  private val authorsRepository: AuthorsRepository,
  private val defaultDispatcher: CoroutineDispatcher = Dispatchers.Default
) {
    suspend operator fun invoke(): List<ArticleWithAuthor> =
        withContext(defaultDispatcher) {
            val news = newsRepository.fetchLatestNews()
            val result: MutableList<ArticleWithAuthor> = mutableListOf()
            // This is not parallelized, the use case is linearly slow.
            for (article in news) {
                // The repository exposes suspend functions
                val author = authorsRepository.getAuthor(article.authorId)
                result.add(ArticleWithAuthor(article, author))
            }
            result
        }
}
```

## Data Layer

For simplicity, NewsRepository uses a mutable variable to cache the latest news. 
To protect reads and writes from different threads, a `Mutex` is used. 
To learn more about shared mutable state and concurrency, see the Kotlin documentation.

```
class NewsRepository(
  private val newsRemoteDataSource: NewsRemoteDataSource
) {
    // Mutex to make writes to cached values thread-safe.
    private val latestNewsMutex = Mutex()

    // Cache of the latest news got from the network.
    private var latestNews: List<ArticleHeadline> = emptyList()

    suspend fun getLatestNews(refresh: Boolean = false): List<ArticleHeadline> {
        if (refresh || latestNews.isEmpty()) {
            val networkResult = newsRemoteDataSource.fetchLatestNews()
            // Thread-safe write to latestNews
            latestNewsMutex.withLock {
                this.latestNews = networkResult
            }
        }

        return latestNewsMutex.withLock { this.latestNews }
    }
}
```

## Recommendations for Android architecture

The following snippet outlines how to collect the UI state in a lifecycle-aware manner:

```
class MyFragment : Fragment() {

    private val viewModel: MyViewModel by viewModel()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect {
                    // Process item
                }
            }
        }
    }
}
```

## ViewModel best practice

ViewModels should expose data to the UI through a single property called uiState. 
If the UI shows multiple, unrelated pieces of data, the VM can expose multiple UI state properties.

1, **suspend functions to perform actions using `viewModelScope`.**

2, Use the `ViewModel` class, not `AndroidViewModel`. 
The Application class shouldn't be used in the ViewModel. Instead, move the dependency to the UI or the data layer.

3, You should make uiState a StateFlow.

4, You should create the uiState using the `stateIn` operator with the `WhileSubscribed(5000)` policy (example) if the data comes as a stream of data from other layers of the hierarchy.

5, For simpler cases with no streams of data coming from the data layer, it's acceptable to use a MutableStateFlow exposed as an immutable StateFlow (example).

6, You can choose to have the ${Screen}UiState as a data class that can contain data, errors and loading signals.
This class could also be a sealed class if the different states are exclusive.

The following snippet outlines how to expose UI state from a ViewModel:

```
@HiltViewModel
class BookmarksViewModel @Inject constructor(
    newsRepository: NewsRepository
) : ViewModel() {

    val feedState: StateFlow<NewsFeedUiState> =
        newsRepository
            .getNewsResourcesStream()
            .mapToFeedState(savedNewsResourcesState)
            .stateIn(
                scope = viewModelScope,
                started = SharingStarted.WhileSubscribed(5_000),
                initialValue = NewsFeedUiState.Loading
            )

    // ...
}
```

## Lifecycle

Do not override lifecycle methods such as `onResume` in Activities or Fragments. Use `LifecycleObserver` instead. 
If the app needs to perform work when the lifecycle reaches a certain `Lifecycle.State`, use the `repeatOnLifecycle` API.

The following snippet outlines how to perform operations given a certain Lifecycle state:

```
class MyFragment: Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        viewLifecycleOwner.lifecycle.addObserver(object : DefaultLifecycleObserver {
            override fun onResume(owner: LifecycleOwner) {
                // ...
            }
            override fun onPause(owner: LifecycleOwner) {
                // ...
            }
        }
    }
}
```

























