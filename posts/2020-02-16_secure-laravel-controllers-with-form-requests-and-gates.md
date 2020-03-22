# Secure Laravel controller with form requests and gates
In this post i will explain how i secure my Laravel controller with [Form-Requests](https://laravel.com/docs/master/validation#form-request-validation) and [Gates](https://laravel.com/docs/master/authorization#gates).

## TL;DR

With this approach, my authorization logic is
1. Easily testable with unit tests
2. All in one place, giving a nice overview of who can do what
3. Using modular authorization logic, which can be reused anywhere (e.g. in blade templates)

[Example Repository](https://github.com/KayDomrose/larave-request-gate-example)

## Doing the old way 
I used to place my authorization logic first in my controller methods and throw exceptions there.  
That did work, but there are three major problems with that:

1. Hard to test  
Laravel controllers are really heavy and do a lot of stuff (request, business logic, response).  
That's why unit tests are getting heavy as well, with all those different parameters and mocks.
But especially in case of authorization there must be a lot of testing to capture every possibility, otherwise your is not secure.  

2. Not very readable  
Authorization checks happen everywhere in the code, so its not very easy to see at once which permissions exist and how they work. This is much more true in large apps, where permissions are more complicated than in most basic tutorials.

3. Not reusable  
When authorization logic (im gonna use this phrase as lot, im sorry) is fixed to my controller, i have no change of reusing it, for example in blade templates.

After some years and testing different methods i found a good solution, that solves all of this problems, although it might look like a lot of work at first glance.

## Custom form requests and gates
The authorization check gets passed from my controller via [Form-Request](https://laravel.com/docs/master/validation#form-request-validation) to a [Gate](https://laravel.com/docs/master/authorization#gates).

Each step can easily be checked via unit test to ensure that my controller methods are secured.  
I also have to check only one place (my gate definition) to see all possible permissions (which can also be used somewhere else like blade templates).

## How it works by example
In this example i want to get all information for a given user.  
The corresponding URL might look like this: `/user/{user}`.  

### Step 1: Controller
```php
// /app/Http/Controllers/UserController.php

// ...

class UserController extends Controller 
{
    public function read(UserReadRequest $request, User $user)
    {
        return $user;
    }
}
```

The first parameter in my `read()` method is my custom form request (more later).
It will be instantiated automatically via [Dependency Injection](https://laravel.com/docs/master/controllers#dependency-injection-and-controllers) and authorization as well as validation will be check.  

Using [Route-Model-Binding](https://laravel.com/docs/master/routing#route-model-binding), i also get the user i want to work with or a 404 error when the user can't be found, without any additional query.

Now i can be sure that the request is authorized, validated and `$user` exists when my method is called. Without any additional checks i can go right down to business.

#### Controller test
An easy test can ensure that my controller method indeed uses the request i want to.
That's very important, as another or no request could bypass the authorization check and thus expose sensible information. 

```bash
php artisan make:test --unit Controllers/UserControllerTest
```

```php
// /tests/Unit/Controllers/UserControllerTest.php

// ...

class UserControllerTest extends TestCase
{
    use HttpTestAssertions;

    public function test_controller_calls_correct_request()
    {
        $this->assertActionUsesFormRequest(
            UserController::class,
            'read',
            UserReadRequest::class
        );
    }
}

```

I use [HttpTestAssertions](https://github.com/jasonmccreary/laravel-test-assertions) to check if a particular controller and method use my form request.  

When this test is green i can be sure that the method indeed uses the correct request.

### Step 2: Form Request
I usually name my requests following this schema: `<ControllerName><MethodenName>Request`. For one, its easy to search in my codebase for exactly the request i want without comparing paths or something, but it's also instantly clear where a requests belongs to.

```bash
php artisan make:request User/UserReadRequest
```

```php
// /app/Http/Requests/User/UserReadRequest.php

// ...

class UserReadRequest extends FormRequest
{
    public function authorize()
    {
        return Gate::forUser($this->user())->allows('user.read', $this->user);
    }
    
    // ...
}
```
At this point i call a gate, passing the current user, an action and the model i want to work with.  
Using `Gate::forUser()` i can ensure that the gate checks with the user that actually did the request.
`$this->user()` returns `User` or `null`.  
When not using `forUser()`, Laravel tries to use the currently authenticated user. That does work most of the time, but i think it's better to explicitly set the user instead of relying on Laravel. As i said, securing your application should be one of your highest priorities.

`allows()` requires the first parameter to be a string, representing the action. I use `<model>.<action>`, so in this case its `user.read`.  
When Laravel cannot find a gate definition for this action, `allows()` will return `false` instead of throwing an exception or something, so be aware of typos.  
The second parameter can be a model, when the authorization depends on that (like: can this user read this particular model).  
Because i use route model binding, i can access the model/user i want to read with `$this->user` in my request (generally: `'/{model}'` => `$this->model`).

Important difference:  
`$this->user()` = Currently authenticated user doing the request  
`$this->user` = The user (model) i want information about

The return value of `authorize()` will be cast to boolean. That decides whether to call the controller or to throw a 403 error  (`AuthenticationException`).  
You could do your authorization logic here, but in gates the logic is encapsulated, more easy to test and reusable.  

#### Request test
Like the controller, i need to test this step as well. That will ensure that, when this request is used, it will call a gate with the correct currently authenticated user, the correct action and the model.  
So lets create a test!

```bash
php artisan make:test --unit Requests/User/UserReadRequestTest
```

```php
// /test/Unit/Requests/User/UserReadRequestTest.php

// ...
// use PHPUnit\Framework\TestCase;
use Tests\TestCase;

class UserReadRequestTest extends TestCase
{
    public function test_authorize_calls_gate()
    {
        $request = new UserReadRequest();
        $request->setUserResolver(function () {
            return factory(User::class)->make([
                'name' => 'currentUserName'
            ]);
        });
        $request->user = factory(User::class)->make([
            'name' => 'targetUserName',
        ]);

        $gate = Mockery::mock(GateContract::class);
        $gate
            ->shouldReceive('allows')
            ->with(
                'user.read',
                Mockery::on(function ($user) {
                    return $user->name === 'targetUserName';
                })
            )
            ->once();

        Gate::shouldReceive('forUser')
            ->with(Mockery::on(function ($user) {
                return $user->name === 'currentUserName';
            }))
            ->andReturn($gate)
            ->once();

        $request->authorize();
    }
}
```
As you can see, testing this few lines of request code is a lot of work. Mocking up only the request needs 8 lines on its own.  
That's one of the reasons to let a gate do the logic, as you need a test for every possible scenario (and testing gates is much more easy than this, as you will see). 

I use [factories](https://laravel.com/docs/master/database-testing#writing-factories) to generate two test users and attach them to the request. With unique names for that, i can see in my test which user appears at which spot.  
Important: Creating tests with `artisan`, Laravel will now use `PHPUnit\Framework\TestCase`.  When working with factories, this causes an error and you need to use `Tests\TestCase` instead ([#30879](https://github.com/laravel/framework/issues/30879#issuecomment-567456608)).

Next i create a mock based on `GateContract` to check the `allows()` method.  
First parameter i expect to be `'user.read'`, the second one an object with a `name` property and the name i gave my test user. Like i said, this shows me that the correct user will be passed to the gate.

Because i use `Facade` for `Gate`, i can make predictions for `forUser()`, also checking `name` instead of only asking for a `User` instance.

When everything is set up i call `$request->authorize()`. The mocks i created for `forUser()` and `allows()` would fail when the methods are not called or called with other parameters than described.

### Step 3: Gate
Are you still with me? Great!  
It's almost over, i promise.

Now i can finally think about who can read my user.

A lot of stuff, right?

Right.   
But now i have a secure, standardized and reusable way to bundle all my authorization logic in one spot. Anyone new to the project only has to read my gate definitions to know who can do what.

As the project grows and authorisation gets more complex, i start using multiple gate files, one for each concept.  Those have to be registered when Laravel starts.

```php
// /app/Providers/AuthServiceProvider.php

public function boot()
{
    // ...

    UserGate::register();

}
```

Now we can finally define the gate for our `User`.

```php
// /app/Gates/UserGate.php

class UserGate
{
    public static function register()
    {
        Gate::define('user.read', function(User $currentUser, User $targetUser):bool {
            return $currentUser->getAttribute('id') === $targetUser->getAttribute('id');
        });
    }
}
```

More on how to define gates can be found in the [official documentation about gates](https://laravel.com/docs/master/authorization#writing-gates).

I define the action `'user.read'`. I get the current authenticated user via `forUser` and also the model i want to read via `allows('user.read', $user)`.
In this case, i simply check whether the `id`s match (so every user can read just themself).

#### Gate-Test
You guessed it, tests again.  
But instead of mocking the hell out of something again, we can now write tests that will check each possible scenario and how the authorization will react.

```bash
php artisan make:test --unit Gates/UserGateTest
```

```php
// /app/test/Unit/Gates/UserGateTest.php

//use PHPUnit\Framework\TestCase;
use Tests\TestCase;

class UserGateTest extends TestCase
{
    use RefreshDatabase;

    public function test_allow_reading_user_when_same_id()
    {
        $currentUser = factory(User::class)->create();

        $result = Gate::forUser($currentUser)->allows('user.read', $currentUser);

        $this->assertTrue($result);
    }

    public function test_dont_allow_reading_user_when_not_same_id()
    {
        $currentUser = factory(User::class)->create();
        $targetUser = factory(User::class)->create();

        $result = Gate::forUser($currentUser)->allows('user.read', $targetUser);

        $this->assertFalse($result);
    }

    public function test_dont_allow_for_guests()
    {
        $currentUser = null;
        $targetUser = factory(User::class)->create();

        $result = Gate::forUser($currentUser)->allows('user.read', $currentUser);

        $this->assertTrue($result);
    }

}
```

As you can see, tests for gates are much more easy to set up than everything before. That makes writing tests also much more easy and lowers the risk of errors during test setup.  
After all: the more you write, the more you can write wrong.

I made temporary models in the request test with (`->make()`), which are not stored in database and speeding up tests.
But as i check the model `id` in my gate, i have to have models with an `id`, which are only set when storing to a database using  `->create()` (you could also overwrite `id` with `make()` via factory, but i want to show the difference).  
To make sure the database and all tables are present, `use RefreshDatabase`.

Also, use `Tests\TestCase` again instead of `PHPUnit\Framework\TestCase`.

## Summary
That's it!

Now i a have a dedicated place where all my authorization code is stored, as clean and readable as possible, and can be reused anywhere i need to.  
I also have some unit tests, confirming that a gate is connected to a controller method via request and everything gets passed down correct.

I hope you found a thing or two you didn't know and is helpful for your next project.

As this is my first post (and also in english), i will be especially thankful for your feedback.

Thank you very much for reading. 
