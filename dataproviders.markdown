# Data Providers

## Why You Should Use Data Providers
One of your main goals should always be to write the bare minimum amount of
code in order to solve a particular problem you are facing. This is no different
when it comes to tests, which are really nothing more than code.

One of the earliest lessons I learned when I first started writing what I felt
were comprehensive test suites was to always be on the lookout for duplication
in your tests. Here's an example of a situation where this can happen.

Most programmers are familiar with the [FizzBuzz](http://en.wikipedia.org/wiki/FizzBuzz)
problem, if only because it is commonly presented as a problem to be solved
as part of an interview. In my opinion it is a good problem to present
because it touches on a lot of really elementary basics of programming.

When you write tests for FizzBuzz, what you want to do is pass it a set
of values and verify that they are FizzBuzzed correctly. This could 
result in you having multiple tests that are the same except for the
values you are testing with. Data providers give you a way to simplify
that process.

A data provider is a way to create multiple sets of testing data that
can be passed in as parameters to your test method. You create a method
that is available to the class your tests are in that returns an array of values
that match the parameters that you are passing into your test.

I know it sounds more complicated than it really is. Let's look at an example.

## Look At All Those Tests
If you didn't know about data providers, what might your FizzBuzz tests look like?

{: lang="php" }
    <?php
    class FizzBuzzTest extends PHPUnit_Framework_Testcase
    {
        public function setup()
        {
            $this->fb = new FizzBuzz();
        }

        public function testGetFizz()
        {
            $expected = 'Fizz';
            $input = 3;
            $response = $this->fb->check($input);
            $this->assertEquals($expected, $response);
        }

        public function testGetBuzz()
        {
            $expected = 'Buzz';
            $input = 5;
            $response = $this->fb->check($input);
            $this->assertEquals($expected, $response);
        }

        public function testGetFizzBuzz()
        {
            $expected = 'FizzBuzz';
            $input = 15;
            $response = $this->fb->check($input);
            $this->assertEquals($expected, $response);
        }

        function testPassThru()
        {
            $expected = '1';
            $input = 1;
            $response = $this->fb->check($input);
            $this->assertEquals($expected, $response);
        }
    }

I'm sure you can see the pattern:

* multiple input values
* tests that are extremely similar in setup and execution
* same assertion being used over and over

## Creating Data Providers
A data provider is another method inside your test class that returns an
array of results, with each result set being an array itself. Through
some magic internal work, PHPUnit converts the returned result set into parameters
which your test method signature needs to accept.

{: lang="php" }
    <?php
    public function fizzBuzzProvider()
    {
        return array(
            array(1, '1'),
            array(3, 'Fizz'),
            array(5, 'Buzz'),
            array(15, 'FizzBuzz')
        );
    }

The function name for the provider doesn't matter, but use some common
sense when naming them as you might be stumped when a test fails and
tells you about a data provider called 'ex1ch2', or something else equally meaningless.

To use the data provider, we have to add an annotation to the docblock
preceding our test so that PHPUnit knows to use it. Give it the name of
the data provider method.

{: lang="php" }
    <?php
    /**
     * Test for our FizzBuzz object
     *
     * @dataProvider fizzBuzzProvider
     */
    public function testFizzBuzz($input, $expected)
    {
        $response = $this->fb->check($input);
        $this->assertEquals($expected, $response);
    }

Now we have just one test (less code to maintain) and can add scenarios to our
heart's content via the data provider (even better). We have also learned the
skill of applying some critical analysis to the testing code we are writing
to make sure we are only writing the tests that we actually need.

When using a data provider, PHPUnit will run the test method each time
for every set of data being passed in by the provider. If the test fails
it will indicate which index in the associative array was being used
for that test run.

## More Complex Examples
Don't feel like you can only have really simple data providers. All you need
to do is return an array of arrays, with each result set matching the
parameters that your testing method is expecting. Here's a more complex example:

{: lang="php" }
    <?php
    public function complexProvider()
    {
        // Read in some data from a CSV file
        $fp = fopen("./fixtures/data.csv");
        $response = array();

        while ($data = fgetcsv($fp, 1000, ",")) {
            $response[] = array($data[0], $data[1], $data[2]);
        }

        fclose($fp);

        return $response;
    }

So don't think you need to limit yourself in what your data providers
are allowed to do. The goal is to create useful data sets for testing
purposes.

## Data Provider Tricks
Since data providers return associative arrays, you can assign them a more
descriptive key to help with debugging. For example, we could refactor
the data provider for our FizzBuzz test:

{: lang="php" }
    <?php
    return array(
        'one'      => array(1, '1'),
        'fizz'     => array(3, 'Fizz'),
        'buzz'     => array(5, 'Buzz'),
        'fizzbuzz' => array(15, 'FizzBuzz')
    );

Also, data providers don't have to be methods inside the same class. You can use
methods in other classes, you just have to make sure to define them as
`public`. You can use namespaces as well. Here are two examples:

* `@dataProvider Foo::dataProvider`
* `@dataProvider Grumpy\Helpers\Foo::dataProvider`

This allows you to create helper classes that are just data providers and cut down on
the amount of duplicated code you have in your tests themselves.
