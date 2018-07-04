# Clean Architecute, Usecases, RxJava/rxKotlin and how to combine all of it into something awesome!

### Introduction

A lot of the projects that we develop for Android make use of the architecture defined in this awesome article by
[Fernando Cejas](https://fernandocejas.com/2015/07/18/architecting-android-the-evolution/)

He recently also wrote an updated arcticle demostrating using some of the newer components such as Kotlin Coroutines instead of RxJava/RxKotlin 
[here](https://fernandocejas.com/2018/05/07/architecting-android-reloaded/)

This article will discuss the `UseCase` section and changes we introduced to it to improve our interaction with it using RxKotlin/RxJava

### Basics of the UseCase

Some of the things we would like to accomplish with our `UseCase` class:
- Need to encapsulate and orchestrate our Business Logic 
- Single, specific interactions with data (Single responsibility Principle)
- Simplify asynchronous processes by worrying about the threading details for us (observers and subscribers)
- Allow us the ability to chain/combine multiple of these interactors/usecases where necessary (this is also sometimes known as a `Workflow`)
- Clearly indicate the intented outcome of the UseCase - will there be a multiple outcomes (Flowable/Observable), a single outcome (Single) or we simply want to know if the action was completed or not (Completable)
- Additional nice to have - Ability to validate that our UseCases are acting on the correct threads (especially when we start chaining them)

This means in the end we create 3 different types of `UseCases`:
- SingleUseCase 
- CompletableUseCase
- Flowable/ObservableUseCase  

### The Abstract UseCase
```kotlin
abstract class UseCase {
    private lateinit var schedulerTransformer: SchedulerTransformer
    private lateinit var debugTransformer: Optional<DebugTransformer>

    @Inject
    fun setSchedulerTransformer(schedulerTransformer: SchedulerTransformer) {
        this.schedulerTransformer = schedulerTransformer
    }

    ...
}
```
@[2-3] (Main purpose of this class is to allow us to attach any transformers on our observable that we are interested in. In most cases this wll be the threads to observe/schedule on and possibly a transformer for debugging the observables)

### The Single UseCase
This is an example of an use case without parameters:

```kotlin
/**
 *
 * @param <T> The return value type
</T> */
abstract class SingleUseCase<out T> : UseCase() {

    protected abstract fun build(): Single<T>

    protected fun make(wrap: Boolean): Single<T> {
        val single = wrapDebug(build())
        return if (wrap) wrap(single) else single
    }

    fun get(): Single<T> {
        return make(true)
    }

    fun chain(): Single<T> {
        return make(false)
    }

    protected fun wrap(observable: Single<T>): Single<T> {
        return observable.compose(schedulerTransformer().applySingleSchedulers())
    }

    private fun wrapDebug(observable: Single<T>): Single<T> {
        return if (debugTransformer().isPresent() && isDebugLogsEnabled()) {
            observable.compose(debugTransformer().get().applySingleDebugger(getClass().getSimpleName()))
        } else observable
    }
}
```
@[3] (Build is what we would override to define our business logic)
@[10] (get ensures we add the transformers to the call that we are interested in)
@[14] (chain allows us to ignore the transformers and only apply them when the UseCase that uses this finally calls get())

### The Observable/Flowable UseCase

Once you see the structure of the UseCases they will all feel very familiar. The only difference is mapping it to the correct observable type.
Here is an example of an UseCase with parameters:

```kotlin
/**
 *
 * @param <T> The return value type
 * @param <P> The paremeter being passed in
</P></T> */
abstract class FlowableParameterisedUseCase<out T, in P> : UseCase() {

    protected abstract fun build(params: P): Flowable<T>

    private fun make(params: Params<P>?, wrap: Boolean): Flowable<T> {
        if (params == null) {
            throw IllegalArgumentException("Params must be defined")
        }

        val single = wrapDebug(build(params.param()))
        return if (wrap) wrap(single) else single
    }

    operator fun get(params: P): Flowable<T> {
        return make(Params.create(params), true)
    }

    fun chain(params: P): Flowable<T> {
        return make(Params.create(params), false)
    }

    private fun wrap(observable: Flowable<T>): Flowable<T> {
        return observable.compose(schedulerTransformer().applyFlowableSchedulers())
    }

    private fun wrapDebug(observable: Flowable<T>): Flowable<T> {
        return if (debugTransformer().isPresent && isDebugLogsEnabled) {
            observable.compose(debugTransformer().get().applyFlowableDebugger(javaClass.simpleName))
        } else observable
    }
}
```

### Example implementation

Imagine the following scenario:
- We have a Packaging company
- In order for us to send a package we need to do 2 things
  - We need to get the shipping information of the company we are sending to via a companyId
  - We need to update our records with that address and mark the package as ReadyToSend

Here is an example of how such a UseCase could look:

```kotlin
class PrepareShipPackageToCompanyUseCase @Inject internal constructor(private val getCompanyInfoUseCase: GetCompanyInfoUseCase, 
                                                               private val updatePackageUseCase: UpdatePackageUseCase)
    : SingleParameterisedUseCase<SubmitPackageResponse, SubmitPackageRequest>() {

    override fun build(request: SubmitPackageRequest): Single<SubmitPackageResponse> {
        return getCompanyInfo(request.companyId).flatMap { updatePackage(request, it)  }
    }

    private fun updatePackage(request: SubmitPackageRequest, companyInfo : CompanyInfo): Single<SubmitPackageResponse> {
        val request = request.copy { shippingAddress = companyInfo.address }
        return updatePackageUseCase.chain(request)
    }

    private fun getCompanyInfo(companyId : Int): Single<CompanyInfo> =
            getCompanyInfoUseCase.chain()
}
```

### Summary

Of course in the real world there is a lot of ways this can be used and adapted for your needs. Thanks for reading. Hope you enjoyed it.