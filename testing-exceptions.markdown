# Testing Exceptions

{: lang="php" }
    <?php
    class Foo
    {
        protected $api;

        public function __construct($api)
        {
            $this->api = $api;
        }

        public function findAll()
        {
            try {
                $this->api->connect();
                $response = $this->api->getAll();
            } catch (Exception $e) {
                throw new ApiException($e->getMessage())
            }

            return $response;
        }
    }

If you're into writing what I refer to as "modern PHP", you are definitely
going to want to be using exceptions to trap all your non-fatal errors.
Code that has exceptions also needs to be tested. Never fear, PHPUnit can
show you the way.

## Testing Using Annotations 
PHPUnit can use annotations to indicate what exceptions and messages it
is expecting to encounter when testing code. Let's create a test for our
code sample above.

{: lang="php" }
    <?php
    /**
     * Test that makes sure we are correctly triggering an
     * exception when we cannot connect to our remote API
     *
     * @expectedException ApiException
     * @expectedExceptionMessage Cannot connect
     */
    public function testThrowsCorrectException()
    {
        $api = $this->getMockBuilder('Api')
            ->disableOriginalConstructor()
            ->getMock();
        $api->expects($this->any())
            ->method('connect')
            ->will($this->throwException(new Exception('Cannot connect')));
        $foo = new Foo($api);
        $foo->connect();
    }

PHPUnit is able to verify the exception and corresponding message through the use
of two annotations.

`@expectedException` is set to be the exception you are expecting to be thrown
while `@expectedExceptionMessage` should be set to the actual message you
are expecting to be generated by the exception.

There is a third annotation you can use, `@expectedExceptionCode` if you like
to have your exceptions throw messages and an associated code value.

## Testing Using `setExpectedException`
You don't have to use annotations if you don't want to. PHPUnit provides a
helper method called `setExpectedException()`.

Here's the test re-written using it:

{: lang="php" }
    public function testThrowsCorrectException()
    {
        $api = $this->getMockBuilder('Api') 
            ->disableOriginalConstructor()
            ->getMock();
        $api->expects($this->any())
            ->method('connect')
            ->will($this->throwException(new Exception('Cannot connect')));
        $foo = new Foo($api);
        $this->setExpectedException('Exception');
        $foo->connect();
    }

The reason to use `setExpectedException` over annotations is that by placing
the call to it right before the code you expect to generate the exception, you
make it easier to debug the test if it fails. When you use the annotation,
PHPUnit is simply expecting the test to throw an exception, not caring at
what point in the test it happened.

`setExpectedException()` also accepts a second optional parameter that can
be used to indicate the message you are expecting the exception to generate.

{: lang="php" }
    public function testThrowsCorrectException()
    {
        // Code is the same up to this point...
        $this->setExpectedException('Exception', 'Cannot connect');
        $foo->connect();
    }
    
This feature is handy (both with this method and when using annotations)
if you have to test code that throws the same exception but with potentially
different messages given certain execution paths.

## Testing Using `try-catch`
As a third option, you can always trap them using a `try-catch` block in the
test itself.

{: lang="php"}
    /**
     * Test that makes sure we are correctly triggering an
     * exception when we cannot connect to our remote API
     */
    <?php
    public function testThrowsCorrectException()
    {
        try {
            $api = $this->getMockBuilder('Api')
                ->disableOriginalConstructor()
                ->getMock();
            $api->expects($this->any())
                ->method('connect')
                ->will($this->throwException(new Exception('Cannot connect')));
            $foo = new Foo($api);
            $foo->connect();
        } catch (ApiException $e) {
            return;
        }

        $this->fail('Did not throw expected ApiException');
    }

If the code under test throws an exception, it will be caught by the `catch`
block and the test will pass. Otherwise `fail()` will cause the test to
have been considered to not have passed.
