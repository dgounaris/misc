# How to write testable and extensible validators by applying Open-Closed principle

Writing validators has consistently been a very common business-driven tasks for software engineers. While the initial construction of such rules might appear trivial, the maintenance burden increases the more these rules change and overlap.

A lot of libraries exist to help with this problem by replacing boilerplate code with domain-specific language or annotations. In this post, we will go over a different approach on reducing this complexity with the use of Dependency Injection and modularized behavior.

## The problem

On of the main problems with most complex validations is that they introduce too much coupling. This is usually done in an effort to keep their logic concise and free of duplication. On the other hand, it makes the logic too complex to effectively test, and advanced concepts such as nested business rules are harder to apply. For example, imagine a validator that checks if a string field capturing a username is null or bigger than 20 characters:

```java
class UserService {
    private UsernameValidator usernameValidator;

    // other code
    usernameValidator.validate(username)
    // other code
}

class UsernameValidator {
    private final String USERNAME_IS_NULL_ERROR = "Username can not be empty.";
    private final String USERNAME_IS_TOO_BIG_ERROR = "Username can not be more than 20 characters.";

    public List<String> validate(String username) {
        List<String> validationErrors = new ArrayList<>();
        if (username == null) {
            validationErrors.add(USERNAME_IS_NULL_ERROR);
        }
        else if (username.length() > 20) {
            validationErrors.add(USERNAME_IS_TOO_BIG_ERROR);
        }
        return validationErrors;
    }
}
```

Let's assume that we then want to also filter the usernames so that we don't allow offensive names. For simplicity, this will be done using a static set of known offensive keywords.

The above code would be amended as such:

```java
class UsernameValidator {
    private final String USERNAME_IS_NULL_ERROR = "Username can not be empty.";
    private final String USERNAME_IS_TOO_BIG_ERROR = "Username can not be more than 20 characters.";
    private final String USERNAME_IS_OFFENSIVE_ERROR = "Username can not contain offensive wording.";

    private final Set<String> offensiveWords = new HashSet<>();

    public List<String> validate(String username) {
        List<String> validationErrors = new ArrayList<>();
        if (username == null) {
            validationErrors.add(USERNAME_IS_NULL_ERROR);
        }
        else if (username.length() > 20) {
            validationErrors.add(USERNAME_IS_TOO_BIG_ERROR);
        }

        if (offensiveWords.stream().anyMatch(offensiveWord -> username.contains(offensiveWord))) {
            validationErrors.add(USERNAME_IS_OFFENSIVE_ERROR);
        }

        return validationErrors;
    }
}
```

Despite this example being fairly simplified, the complexity of the code is already significantly impacted. There are already 4 different inputs to test with different expected output. On top of that, there is even a `NullPointerException` error lurking in this seemingly innocent code.

## The solution

To solve this increasing complexity, we will abide by the common pattern of keeping our classes "open for extension, closed for modification". This means that the architecture should be extensible, but not by modifying existing robust and battle-tested code.

The first step we will have to take is to abstract the validation implementation from its caller. This can be done with the implementation of an interface and the use of some Dependency Injection framework to feed the correct validator implementation.

Because we will be leveraging Dependency Injection, the caller will only need to depend on the interface, without exact knowledge of the underlying implementations.

For example, the above class definition would be changed to:

```java
interface UsernameValidator {
    public List<String> validate(String username);
}

class UsernameValidatorImpl implements UsernameValidator {
    // validation code
}

class UserService {
    private List<UsernameValidator> usernameValidators; // referencing the interface, instances are auto-resolved through Dependency Injection

    // other code
    usernameValidators.stream().map(validator ->
        validator.validate(username)
    ).flatMap(it -> it.stream()).toList();
    // other code
}
```

Now that our implementation is agnostic of the actual validation classes, our next aim is to reduce the overall complexity of the validator. This will be done by splitting the validations in distinct classes, with a single use-case per validation, like so:

```java
interface UsernameValidator {
    public String validate(String username);
}

// other validators skipped for brevity

class UsernameIsNotTooBigValidatorImpl implements UsernameValidator {
    private final String USERNAME_IS_TOO_BIG_ERROR = "Username can not be more than 20 characters.";

    public String validate(String username) {
        if (username != null &&
            username.length() > 20
        ) {
            return USERNAME_IS_TOO_BIG_ERROR
        }
        return null;
    }
}

class UserService {
    private List<UsernameValidator> usernameValidators; // referencing the interface, instances are auto-resolved through Dependency Injection

    // other code
    usernameValidators.stream().map(validator ->
        validator.validate(username)
    ).toList();
    // other code
}
```

This make the overall classes more testable by reducing the possible alternate flows to a single branching logic. Additionally, note the change of the function signature from `List<String>` to `String`. This is done in order to force each validator to return a single result to a single validation logic.

## Extending our logic

The refactored implementation above clearly has some benefits, but also some drawbacks exist:
- All validations are executed in arbitrary order.
= There is no way to have a "showstopper" validation, which if failing no other validation is executed.
- Parts of similar logic for different validations would be duplicated. For example, more null checks have to be explicitly declared.

Solving these problems is very easy with our new, refactored implementation and would have been very difficult to navigate in the previous implementation.

**Execute validators in defined order**

To execute the validations in a predefined order, we can alter the interface and the service call to:
```java
interface UsernameValidator {
    public int getPriority();
    public String validate(String username);
}

// other code skipped for brevity

class UserService {
    private List<UsernameValidator> usernameValidators; // referencing the interface, instances are auto-resolved through Dependency Injection

    // other code
    usernameValidators.stream()
    .sorted((v1, v2) -> v1.getPriority() > v2.getPriority())
    .map(validator ->
        validator.validate(username)
    ).toList();
    // other code
}
```

Note that our implementation is agnostic to the validation rules and each validation class is isolated from others. This makes it possible to reorder the validations simply by changing the priority value, without any risk of logic alteration that would result otherwise from coupling.

**Create "showstopper" validations**

Similarly to how we applied our validation ordering above, validations that block subsequent ones can be defined as part of the interface too:

```java
interface UsernameValidator {
    public boolean stopOnError();
    public int getPriority();
    public String validate(String username);
}

// other code skipped for brevity

class UserService {
    private List<UsernameValidator> usernameValidators; // referencing the interface, instances are auto-resolved through Dependency Injection

    // other code
    List<UsernameValidator> sortedUsernameValidators = usernameValidators.stream().sorted((v1, v2) -> v1.getPriority() > v2.getPriority()).toList();
    List<String> validationErrors = new ArrayList<String>();
    for (UsernameValidator validator : sortedUsernameValidators) {
        String validatorResult = validator.validate(username);
        if (validatorResult != null) {
            validationErrors.add(validatorResult)
            if (validator.stopOnError()) {
                break;
            }
        }
    }
    // other code
}
```

Note that our service is responsible for orchestrating the validator execution. Thus, each validator is not polluted with business rules such as whether it should run after another has failed. This logic is abstracted and generalized in the service instead, making everything more error-proof and testable.

**Nesting validation rules to reduce duplication**

Avoiding duplication is a crucial part of complex validation logic. Not only does it result in cleaner code and less effort, but also provides benefits when validations involve communication. For example, if a validator relies heavily on calls to an external service API or database, we would prefer to allow our validators to reuse common results from these APIs.

To solve this problem, a nested logic that follows the same pattern as our overall architecture can be applied. Each validator can depend on additional validators which implement a different interface. Alternatively, if some validators rely on multiple results that are captured from many sources, an in-memory "cache" can be used.

Depending on the particular scenario, different structures can be applied. All of them share the premise that validators are isolated and their behavioral characteristics (priority, dependencies) are captured by clear, well-defined, easy to understand interface rules.

We hint a way to implement the described solution below:

```java
interface UsernameValidator {
    public boolean stopOnError();
    public int getPriority();
    public List<String> validate(String username);
}

interface NotNullUsernameValidator {
    public boolean stopOnError();
    public int getPriority();
    public String validate(String username);
}

class UsernameValidatorImpl implements UsernameValidator {
    private final String USERNAME_IS_NULL_ERROR = "Username can not be null.";
    private List<NotNullUsernameValidator> notNullUsernameValidators;

    public List<String> validate(String username) {
        List<String> validationResult = new ArrayList<String>();
        if (username == null) {
            validationResult.add(USERNAME_IS_NULL_ERROR)
            return validationResult;
        } else {
            return notNullUsernameValidators.stream()
            .map(validator -> validator.validate(username))
            .toList();
        }
    }
}

// other code skipped for brevity
```

## Conclusion

Open-Closed principle is a powerful design decision that can help with many problem spaces. This is particularly true in scenarios where modularity and extensibility is critical to avoid mistakes.

By applying best-practices, we showed how our validation logic can remain testable, robust and infinitely extensible, without use of any additional libraries or tooling.