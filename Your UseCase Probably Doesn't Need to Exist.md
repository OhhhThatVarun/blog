# 14 July 2026

# Your UseCase Probably Doesn't Need to Exist

I have worked with Android for quite some time, read endless blog posts about clean architecture, and looked at many GitHub repositories. After all this time, I have come to the conclusion that most developers don't understand what UseCases are or how and when to use them.

## Problem

Let's take an example on how it usually looks

```kotlin
class GetSomeData(private val someDataRepo: SomeDataRepository) {
	suspend fun invoke(dataID: Int): Flow<List<SomeData>> {
	  return someDataRepo.getSomeDataByID(dataID)
  } 
} 
```
or

```kotlin
class IsUserPremiumUseCase(private val userRepository: UserRepository) {
  fun invoke(userId: Int): Flow<Boolean> {
	  return userRepository.isPremiumUser(userId)
  }
}
```

Both of the above examples look harmless, are easy to follow, and make sense at a glance. But the devil is in the details.

Anyone looking at this might argue it follows the SRP (Single Responsibility Principle), claiming it's easy to maintain, encourages code reuse, and simplifies testing. But the real questions one should be asking: *Which "responsibility" does it fulfill? What is there to test? What is there to maintain? What is there to reuse?*

The answer to all the questions is absolutely nothing. It is just visual noise in the source code. It carries no real responsibility, contains no logic to maintain, it removes no duplication, it doesn't decouple your UI or other parts of the application and offers no test surface. *It's a repository method wearing a costume.* This UseCase and the repository function share the same responsibility: returning data. That is clear implicit duplication.

One heuristic is to ask what would have to happen for a UseCase to change. Apply it to the examples above: both only change when the underlying API changes. That's the tell, there should always be another reason for a UseCase to change, ideally a business requirement.

There is one more type of UseCases, i.e., abstract UseCases

```kotlin
// RefreshTokenUseCase.kt
interface RefreshTokenUseCase {
    fun invoke()
}

//RefreshTokenUseCaseImpl.kt
class RefreshTokenUseCaseImpl(private val tokenRepository: TokenRepository, private val api: Api): RefreshTokenUseCase {
	override fun invoke(): Flow<Unit> {
	  return api.refreshToken(tokenRepository.getToken())
  }
}
```

This, I consider an anti-pattern. Why? UseCases represent real facts about your business, they are supposed to be a rigid unit of work, because here you actually perform business rules, and there should never be an abstraction for it. The UseCase already is the abstraction boundary and the domain logic sits behind it, callers depend on it, DI lets you swap what's behind it. Putting another interface in front of the thing that's already the seam is abstracting the abstraction. UseCases contain your real business logic; if you need to change your business logic, then you should go ahead and change it in the code or change its dependencies.

If you ever need an abstraction for them, just stop there and ask yourself why. And you will find yourself with no real reason. Needing an abstraction for your UseCase is actually a sign of a bad architecture, because UseCases already exist in your domain/application layer. You have a bigger problem, and this is just a symptom of it.

One counterargument developers often present is testing, that UseCase interfaces actually help with testing and provide a seam. This is the biggest red flag. *Why would you mock your real business rules? Isn't that the exact thing you're supposed to be testing?* UseCases already depend upon abstract types, if you need to alter the behaviour of your UseCase for a test, then you should be altering the behaviour of its dependencies.

Mocking the UseCase and asserting `verify(useCase).invoke()` only proves you called a function. Injecting a fake TokenRepository and asserting the real RefreshTokenUseCaseImpl actually refreshed the token proves the rule works.

> Note: Keep in mind while modeling UseCases, the UseCase should only change when the rules it is currently encapsulating change.

There is one more type of UseCases, i.e., Building Block UseCases

```kotlin
class CheckoutUseCase(
    private val canUserCheckoutUseCase: CanUserCheckoutUseCase,
    private val placeOrderUseCase: PlaceOrderUseCase
) {
    suspend fun invoke() {
        val eligibility = canUserCheckoutUseCase.invoke()
        if (eligibility != CheckoutEligibility.Eligible) {
            throw CheckoutException()
        }

        placeOrderUseCase.invoke()
    }
}
```

I also consider this an anti-pattern. Why? Because it blurs architectural boundaries. A UseCase represents what the application can do, not a reusable implementation detail. It is an application boundary, not a building block.

If two UseCases need the same business rule, extract that rule into a Policy, Service, etc and let both UseCases depend on that instead.

> Rule of thumb: Share the business rule, not the UseCase.

## Lifecycle

UseCases should never ever have any lifecycle or lifecycle-dependent dependencies; the only time a UseCase should exist is at the moment we need it, not before and not after. Why? Because rules have no lifecycle, they simply exist, inside and outside of software. I have often seen teams making UseCases Singletons, which makes them aware of the application lifecycle and breaks their semantics. It is named **USE**case and should be modeled as a use-and-throw object. UseCases are usually very light to construct. Also, all the dependencies of a UseCase are usually application-wide (e.g. Database or Api client), which are already Singletons. Then keeping a UseCase Singleton in memory for business logic serves no purpose.

> Note: The return type of UseCase also matters a lot becuase it also tells the half story. For example a UseCase returning `Flow<T>` has the same problem in a different shape. It's no longer a one-shot or single execution of a rule, it's a subscription that stays alive for as long as something collects it, which is lifecycle-shaped behavior even without a Singleton. If you look at `ApplyDiscountUseCase` below, notice it returns a single BigDecimal, not a stream, that is by design and not an accident.

```kotlin
class ApplyDiscountUseCase(private val userRepository: UserRepository) {
    suspend fun invoke(cartTotal: BigDecimal): BigDecimal {
        val user = userRepository.getCurrentUser()
        val discount = when {
            user.isFirstPurchase -> cartTotal * 0.10
            user.loyaltyTier == LoyaltyTier.GOLD -> cartTotal * 0.15
            else -> BigDecimal.ZERO
        }
        return (cartTotal - discount).coerceAtLeast(BigDecimal.ZERO)
    }
}
```

Let's consider the above UseCase. What good will having a Singleton of `ApplyDiscountUseCase` do? Nothing, as it is very easy to re-construct.

## What is a good UseCase

I actively encourage people to not use a UseCase. Why? Because it's often very difficult to discover an actual business rule that deserves one UseCase. Once we start treating UseCases lightly then each function which should be within a service boundary or in a Repository ends up being a UseCase. It also starts a silent war within the codebase between boundaries. For example, where should `SubmitOrder` go? `OrderService` or `SubmitOrderUseCase`? Which also makes it a little difficult for junior developers to understand. Which results in an unpredictable codebase, because no one can guarantee if `OrderService` lives is inside a `Service` or `UseCase` without looking at the codebase.

> Note: Discovering an actual UseCase is difficult because you're not looking for an operation, you are looking for a business rule. Most functions in an application simply move data between layers, and those don't become UseCases just because they have a verb in their name.

A side effect of this is that it makes it harder to find where real domain logic lives, since every use case looks equally important whether it's genuine orchestration or a rename.

It is very simple to identify an ideal UseCase because it represents or performs a verb, it encapsulates one business rule or orchestrates a business workflow, and performs it well. Business rules could be anything, validation, mapping, or transformation. It doesn't mean it always has to cross some boundaries to be a valid UseCase.

```kotlin
class CalculateShippingCostUseCase {
    fun invoke(cartTotal: BigDecimal, isPrimeMember: Boolean): BigDecimal {
        if (isPrimeMember) return BigDecimal.ZERO
        return if (cartTotal >= BigDecimal(50)) BigDecimal.ZERO else BigDecimal(5.99)
    }
}
```
The above example is a valid UseCase. Why? Because it changes only when the rule changes, it doesn't have any lifecycle, and it is verb-shaped and has a single rule. Anyone who looks at its API will immediately understand and won't have to guess anything.

Let's take one more example

```kotlin
class CanUserCheckoutUseCase(
    private val cartRepository: CartRepository,
    private val inventoryRepository: InventoryRepository,
    private val userRepository: UserRepository
) {
    suspend fun invoke(): CheckoutEligibility {
        val cart = cartRepository.getCurrentCart()
        val user = userRepository.getCurrentUser()

        if (cart.items.isEmpty()) return CheckoutEligibility.EmptyCart
        if (user.isBanned) return CheckoutEligibility.AccountRestricted

        val outOfStock = cart.items.filter {
            inventoryRepository.getStock(it.productId) < it.quantity
        }
        if (outOfStock.isNotEmpty()) return CheckoutEligibility.ItemsUnavailable(outOfStock)

        return CheckoutEligibility.Eligible
    }
}
```

> Note: Whether this rule lives inside a Domain Service, or Application UseCase depends on your architecture. The important point isn't the layer—it is that the class represents a business rule rather than forwarding data.

Again, the above example encapsulates a single rule, i.e., can a user checkout. It orchestrates from three different repositories. It has a real test surface, we can test it against real-case scenarios. We can test it very easily; we just need to pass scenario-based repository instances, for example, `EmptyCartRepository`, `OutOfStockInventoryRepository`, `UserBannedRepository`.

I opened this by saying most developers don't understand what UseCases are for. By now the reason should be obvious: nobody taught them the one test that actually matters, *would this class change for any reason other than a business rule?* Every anti-pattern here fails that test. Every legitimate UseCase passes it.

| Question to ask | If true | Verdict |
|---|---|---|
| Does it just forward a single call to a repository/API/prefs with no logic? | Yes | Anti-pattern — a repository method wearing a costume |
| Would it only change when the underlying API or storage mechanism changes? | Yes | Anti-pattern — it's tracking implementation, not a business rule |
| Is there an interface sitting in front of the UseCase's own implementation? | Yes | Anti-pattern — abstracting the abstraction; the UseCase is already the seam |
| To test it, do you mock the UseCase itself rather than its dependencies? | Yes | Anti-pattern — you're asserting a function was called, not that the rule works |
| Is it held as a Singleton or given any lifecycle awareness? | Yes | Anti-pattern — risks scope mismatch with shorter-lived dependencies, for no benefit |
| Was it minted mainly to give one verb (get/set, is/set) its own class, splitting a cohesive concern in two? | Yes | Anti-pattern — forces duplicated state/keys across classes that should stay together |
| Would it only change because a business rule changed? | Yes | Legitimate — this is what a UseCase is for |

Next time you need to write `class DoTheThingUseCase`, run it through that table before you run it through a code review. If you can't name the rule it encapsulates, you haven't found a UseCase, you've found a class looking for a job 🤣.
