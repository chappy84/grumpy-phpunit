# Test Doubles
A key part of doing any sort of unit testing is the ability to create fake
versions of dependencies that are required to test code. They are referred
to by many names, names that might often confuse people who are new to
writing unit tests.

In the pure testing world, there are three types of test doubles:

* dummy objects
* test stubs
* test spies
* test mocks
* test fakes

PHPUnit is not a purist in what it does. Dummy objects, stubs and mocks
are available out of the box, and you can kind of, sort of, create spies
using some of the methods provided by the mocking API.

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

The purpose of using test doubles is so that we can create versions of our
dependencies to test specific scenarios. Sometimes we just need something
that can stand in for a dependency and we are not worrying about faking any
functionality. For this we can just create a dummy object.

Let's say we are creating a scenario where we are testing some code that
accepts two objects via constructor injection, but we are only testing
code that exercises one of those dependencies.

Now, in order to test anything we need to pass in a Foo and Bar object.
Also, because we are using type hinting in the constructor we need to make
sure that we actually pass in objects of the correct type.

{: lang="php" }
    <?php
    public function testThatBarMergesCorrectly()
    {
        $foo = $this->getMockBuilder('Foo')->getMock();

        // Other code that mocks our Bar dependency...
        // Not revealed because we cover it later...

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

A totally contrived example to be sure, but why did we do the test this way?
In our code under test we are specifically using type hinting, so the code
is expecting to be given a dependency of a specific type.

We don't need to completely mock out *$foo* because our test doesn't require
a *Foo* object to do anything. Remember, the goal when writing tests is to
also minimize the amount of testing code you need to write. Don't mock
things if they don't need to be mocked!

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

Okay, so there isn't anything really exciting about using these tools to
create an empty object. The next step is to create test stubs.

A test stub is a dummy object that you create and then tell it that when
specific methods of that object are called, we expect to get a specific
response back.

Next, we tell the mock what method we want it to run. In this case we are
trying to stub out the functionality of *getStatus()* to replace whatever it
really wants to do.

A word of warning: PHPUnit cannot mock protected or private class methods.
To do that you need to use PHP's Reflection API to create a copy of the
object you wish to test and set those methods to be publicly visible.

I realize that from a code architecture point of view protected and
private methods have their place. As a tester, they are a pain. 

To end the stub method, we use *will()* to tell the stubbed method what we
want it to return. 

In my experience, understanding how to stub methods in the dependencies
of the code you are trying to test is the number one skill that good testers
learn.

You will learn to use test stubs as a "code smell", since it will reveal
that you might have too many dependencies in your code, or that you have an
object that is trying to do too much. Inception-level test stubs inside
test stubs inside test stubs is an indication that you need to do some
rethinking of your architecture. 

### Expectations during execution
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

There are a number of values that we can use for *expects()*:

* *$this->at()* can be used to set expected return values based on how many times the method is running, starting at index 0
* *$this->any()* won't care how many times you run it, and is the choice of lazy
programmers everywhere
* *$this->never()* expects the method to never run
* *$this->once()* expects, well, I think you can figure that one out
* *$this->atLeastOnce()* is an interesting one, a good alternative to any()

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

You can also write tests where you can mock an object that will return
different results based on specific inputs, using the *with()* method.
This is useful if you are testing some code that uses a dependency in a
loop and wish to verify that a certain series of values are returned in
a known order.

### Mocking method calls with multiple parameters
If you need to mock a method that accepts multiple parameters, you can
specify that inside the with() method

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

To test it, we unleash the power of reflection.

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
            $message = PHPUnit_Framework_Assert::readAttribute(
                $testFoo, 'message'
            );

            $this->assertEquals(
                $expectedMessage,
                $message,
                'Did not get expected message'
            );
        }
    }

## Test Spies
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

    // In your test class...
    public function betaProcessCalledExpectedTime()
    {
        $beta = $this->getMockBuilder('Beta')->getMock();
        $beta->expects(3)->method('process');
        $deltas = array(-18.023, -14.123, 3.141);

        $alpha = new Alpha($beta);
        $alpha->cromulate($deltas);
    }

Often you will want to test that a certain method in your code has been run
a specific number of times. You can do that using the *expects()* method as
part of your mock object. 

In this code, we want to make sure that, given a known number of 'deltas',
*Beta::process()* gets called the correct number of times. In the above test example,
the test case would fail if *Beta::process()* is not run 3 times. 

## More Object Testing Tricks

### Testing Traits
If you are using PHP 5.4 and want to compose objects using traits,
you have two options. You can test objects that use traits, and then
test the functionality that the traits add.

If you are using PHPUnit 3.6 or newer, you can use a helper
that will give you a test double that uses the trait:

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
