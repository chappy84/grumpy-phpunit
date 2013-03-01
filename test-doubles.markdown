# Test Doubles

## Why We Need Them
When you are writing unit tests, you are writing tests to create
scenarios where you control the inputs and want to verify that your
code is returning expected results. 

Code you write often depends on other code: things like
database handles or values from a globally-available registry.
In order to write tests for code using these dependencies, you need to be able
to set those dependencies to specific states in order to 
predict the expected result.

One way to do this is through the use of [dependency injection](http://en.wikipedia.org/wiki/Dependency_injection).
That Wikipedia article is long and technical, but the lesson to be learned
is that you should be creating small modules of code that accept their
dependencies one of three ways:

* passing them in via an object's constructor method and assigning them to class attributes
* by assigning dependencies to class attributes directly or by setters
* by using a dependency injection container that is globally available

Whatever method you choose (I prefer constructor injection), you can create
objects in a known state through the use of test doubles created using
PHPUnit's built-in mocking functionality.

## What Are They
In the pure testing world, there are three types of test doubles:

* dummy objects
* test stubs
* test spies
* test mocks
* test fakes

PHPUnit is not a purist in what it does. Dummy objects, stubs and mocks
are available out of the box, and you can kind of, sort of, create spies
using some of the methods provided by the mocking API. No test fakes either.

## Dummy Objects
{: lang="php" }
    <?php
    class Baz
    {
        public $foo;
        public $bar;

        public function __construct(Foo $foo, Bar $bar)
        {
            $this->foo = $foo;
            $this->bar = $bar;
        }

        public function processFoo()
        {
            return $this->foo->process();
        }

        public function mergeBar()
        {
            if ($this->bar->getStatus() == 'merge-ready') {
                $this->bar->merge();
                return true;
            }

            return false;
        }
    }

Sometimes we just need something
that can stand in for a dependency and we are not worrying about faking any
functionality. For this we use a dummy object.

Looking at the code above, in order to test anything we need to pass in a Foo and Bar object.

{: lang="php" }
    <?php
    public function testThatBarMergesCorrectly()
    {
        $foo = $this->getMockBuilder('Foo')->getMock();

        $bar = $this->getMockBuilder('Bar')
            ->setMethods(array('getStatus', 'merge'))
            ->getMock();
        $bar->expects($this->once())
            ->method('getStatus')
            ->willReturn($this->getValue('merge-ready'));

        // Create our Baz object and then test our functionality
        $baz = new Baz($foo, $bar);
        $expectedResult = true;
        $testResult = $baz->mergeBar();

        $this->assertEquals(
            $expectedResult,
            $testResult,
            'Baz::mergeBar did not correctly merge our Bar object'
        );
    }

Our dummy object in this test is the `Foo` object we created. This test
doesn't care if Foo does anything. Remember, the goal when writing tests
is to also minimize the amount of testing code you need to write. Don't
create test doubles for things if they aren't needed for the test!

## Test Stubs
{: lang="php" }
    <?php
    public function testMergeOfBarDidNotHappen()
    {
        $foo = $this->getMockBuilder('Foo')->getMock();
        $bar = $this->getMockBuilder('Bar')->getMock();
        $bar->expects($this->any())
            ->method('getStatus')
            ->will($this->returnValue('pending'));

        $baz = new Baz($foo, $bar);
        $testResult = $baz->mergeBar();

        $this->assertFalse(
            $testResult,
            'Bar with pending status should not be merged'
        );
    }

A test stub is a mock object that you create (remember, a dummy object
is really a mock object without any functionality) and then alter it
so that when specific methods are called, we get a specific response back.

In the test above, we are creating a test stub of `Bar` and controlling
what a call to `getStatus` will do.

A word of warning: PHPUnit cannot mock protected or private class methods.
To do that you need to use PHP's [Reflection API](http://php.net/manual/en/book.reflection.php) to create a copy of the
object you wish to test and set those methods to be publicly visible.

I realize that from a code architecture point of view protected and
private methods have their place. As a tester, they are a pain. 

To end the stub method, we use `will` to tell the stubbed method what we
want it to return. 

Understanding how to stub methods in the dependencies
of the code you are trying to test is the number one skill that good testers
learn.

You will learn to use test stubs as a "code smell", since it will reveal
that you might have too many dependencies in your code, or that you have an
object that is trying to do too much. Inception-level test stubs inside
test stubs inside test stubs is an indication that you need to do some
rethinking of your architecture. 

### Expectations during execution
Here is an example of a test that sets expectations for what a particular
method should return when called multiple times.

{: lang="php" }
    <?php
    public function testShowingUsingAt()
    {
        $foo = $this->getMockBuilder('Foo')->getMock();
        $foo->expects($this->at(0))
            ->method('bar')
            ->will($this->returnValue(0));
        $foo->expects($this->at(1))
            ->method('bar')
            ->will($this->returnValue(1));
        $foo->expects($this->at(2))
            ->method('bar')
            ->will($this->returnValue(2));
        $this->assertEquals(0, $foo->bar());
        $this->assertEquals(1, $foo->bar());
        $this->assertEquals(2, $foo->bar());
    }

There are a number of values that we can use for `expects()`:

* `$this->at()` can be used to set expected return values based on how many times the method is run. It accepts an integer as a parameter and starts at 0
* `$this->any()` won't care how many times you run it, and is the choice of lazy testers everywhere
* `$this->never()` expects the method to never run
* `$this->once()` expects the method to be called only once during the test. If you are using `$this->with` that we talk about in the next section, PHPUnit will check that the method is called once with those specific parameters being passed in 
* `$this->atLeastOnce()` is an interesting one, a good alternative to any()

When you are creating expectations for multiple calls, be aware that 
whatever response you are setting through the use of `this->with` is
only applicable to that specific expectation.

### Returning Specific Values Based On Input
{: lang="php" }
    <?php
    public testChangingReturnValuesBasedOnInput()
    {
        $foo = $this->getMockBuilder('Foo')->getMock();
        $foo->expects($this->once())
            ->method('bar')
            ->with('1')
            ->will($this->returnValue('I'));
        $foo->expects($this->once())
            ->method('bar')
            ->with('4')
            ->will($this->returnValue('IV'));
        $foo->expects($this->once())
            ->method('bar')
            ->with('10')
            ->will($this->returnValue('X'));
        $expectedResults = array('I', 'IV', 'X');
        $testResults = array();
        $testResults[] = $foo->bar(1);
        $testResults[] = $foo->bar(4);
        $testResults[] = $foo->bar(10);

        $this->assertEquals($expectedResults, $testResults);
    }

Often, the
methods you invoke on dependencies accept arguments, and will return
different values depending on what was passed in. With PHPUnit, you can tell a mock object what to
expect for arguments, and bind a specific return value for that input. To do
this, you use the `with()` method to detail arguments, and the `returnValue()`
method, inside the `will()` method, to detail return values.

The test above shows you how to go about creating the expectation of a
specific result based on a specific set of input parameters.

### Mocking method calls with multiple parameters
If you need to mock a method that accepts multiple parameters, you can
specify that inside the with() method; specify the arguments in the same order
the method accepts them:

{: lang="php"}
    <?php
    // Method is fooSum($startRow, $endRow)
    $foo = $this->getMockBuilder('MathStuff')->getMock();
    $startRow = 1;
    $endRow = 10;
    $foo->expects($this->once())
        ->method('fooSum')
        ->with($startRow, $endRow)
        ->will($this->returnValue(55));

### Testing protected and private methods and attributes 
I will be up front: I am not an advocate of writing tests for
protected and private class methods. In most cases, when you
write tests for public methods, you will end up testing
private and protected methods that it calls.

Of course, if these private and protected methods have
side effects, you will probably need to test them just
to make sure they are tested properly.

In PHP, the only way to do this easily is by using
the Reflection API that is available in PHP 5.

Here is a very contrived example:

{: lang="php"}
    <?php
    class Foo
    {
        protected $message;

        protected function bar($environment)
        {
            if ($environment == 'dev') {
                $this->message = 'CANDY BAR';
            }
 
            $this->message = "PROTECTED BAR";
        }
    }

To create a test double that we can use, we need to follow these steps:

* use the Reflection API to create a reflected copy of the object
* call `setAccessible(true)` on the method in the reflected object you want to test
* call `invoke()` on the reflected object, passing in the name of the method to test and any parameters you wish to pass in

If you need to perform an assertion against the contents of a protected or
private attribute, you can then use `assertAttribute` just like you would
do any other assertion. [Chapter 4](http://www.phpunit.de/manual/current/en/writing-tests-for-phpunit.html#writing-tests-for-phpunit.assertions) 
of the PHPUnit documentation covers all the different types of assertions
you can make.

{: lang="php"}
    <?php
    class FooTest extends PHPUnit_Framework_TestCase
    {
        public function testProtectedBar()
        {
            $testFoo = new Foo();
            $expectedMessage = 'PROTECTED BAR';
            $reflectedFoo = new ReflectionMethod($testFoo, 'bar');
            $reflectedFoo->setAccessible(true);
            $reflectedFoo->invoke($testFoo, 'production');

            $this->assertAttributeEquals(
                'production',
                $reflectedFoo,
                $expectedMessage,
                $message,
                'Did not get expected message'
            );
        }
    }


## Test Spies
The main difference between a test spy and a test stub is that you're not
concerned with testing return values, only that a method you are testing
has been called a certain number of times.

Given the code below:

{: lang="php" }
    <?php
    class Alpha
    {
        protected $beta;

        public function __construct($beta)
        {
            $this->beta = $beta;
        }

        public function cromulate($deltas)
        {
            foreach ($this->deltas as $delta) {
                $this->beta->process($delta);
            }
        }

        // ...
    }

If you wanted to create test spies to make sure a method is executed 3
times, you could do something like this:

{: lang="php" }
    <?php

    // In your test class...
    public function betaProcessCalledExpectedTime()
    {
        $beta = $this->getMockBuilder('Beta')->getMock();
        $beta->expects(3)->method('process');
        $deltas = array(-18.023, -14.123, 3.141);

        $alpha = new Alpha($beta);
        $alpha->cromulate($deltas);
    }


In this code, we want to make sure that, given a known number of 'deltas',
`process()` gets called the correct number of times. In the above test example,
the test case would fail if `process()` is not run 3 times. 

As you saw in earlier tests in this chapter, `expects()` can accept a wide
variety of options, but you can pass it an integer that matches the number of
times you are expecting the method to be called in your test.

## More Object Testing Tricks

### Testing Traits
Because Traits can be defined once, but used many times, you will not want
to necessarily test the functionality defined in traits in every object in
which they are consumed. At the same time, you _do_ want to test the traits
themselves. 

PHPUnit 3.6 and newer offers functionality for mocking traits
via the `getObjectForTrait()` method; this will return an object composing
the trait, so that you can unit test only the trait itself.

Here's a sample trait:

{: lang="php" }
    <?php
    trait Registry
    {
        protected $values = array();

        public function set($key, $value)
        {
            $this->values[$key] = $value;
        }

        public function get($key)
        {
            if (!isset($this->values[$key])) {
                return false;
            }

            return $this->values[$key];
        }
    }

Okay, now we test it

{: lang="php" }
    <?php
    class RegistryTest extends PHPUnit_Framework_TestCase
    {
        public function testReturnsFalseForUnknownKey()
        {
           $registry = $this->getObjectForTrait('Registry');
            $response = $registry->get('foo');
            $this->assertFalse(
                $response,
                "Did not get expected false result for unknown key"
            );
        }
    }
