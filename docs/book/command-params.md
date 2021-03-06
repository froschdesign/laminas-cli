# Command Params

The Symfony Console component allows commands to define
[input arguments and options](https://symfony.com/doc/current/console/input.html).

The laminas-cli package adds a third possibility, **input parameters**, which
allow you to define command options that will become interactive prompts when
omitted during command invocation.

## Using input parameters

Internally, input parameters behave like standard options with the following
modifications:

- Input parameters are optional by default.
- In interactive mode, when retrieving a parameter value that was not provided
  as an option, the user will be asked to provide the value.
- In non-interactive mode, if the user does not supply the option during
  invocation, and the parameter is not marked as required, its
  default value will be used; otherwise, an exception will be thrown,
- **If the parameter has validations or normalizations supplied, these will be
  applied regardless of whether the option is provided during invocation or when
  the user is prompted to provide it.**
- If the user provides a value via a prompt, that value will be stored as an
  input option so it can be passed to the [next command in the
  chain](commands-chain.md) (if one is defined).

## Input parameter types

Input parameters are classes that implement
`Laminas\Cli\Input\InputParamInterface`:

```php
namespace Laminas\Cli\Input;

use Symfony\Component\Console\Input\InputOption;
use Symfony\Component\Console\Question\Question;

interface InputParamInterface
{
    /**
     * Default value to use if none provided.
     *
     * @return null|mixed
     */
    public function getDefault();

    public function getDescription(): string;

    public function getName(): string;

    /**
     * Return the Symfony\Component\Console\Input\InputOption::VALUE_* type.
     */
    public function getOptionMode(): int;

    public function getShortcut(): ?string;

    public function getQuestion(): Question;

    public function isRequired(): bool;

    /**
     * @param mixed $defaultValue
     */
    public function setDefault($defaultValue): self;

    public function setDescription(string $description): self;

    public function setShortcut(string $shortcut): self;

    public function setRequiredFlag(bool $required): self;
}
```

We provide the abstract class `Laminas\Cli\Input\AbstractInputParam` to
implement the majority of the methods in the interface; the only items you need
to fill in are:

- `getQuestion()`: Each parameter type will define its own `Question` to return.
- `protected $optionMode`, which is set to `Symfony\Component\Console\Input\InputOption::VALUE_REQUIRED`
  by default.

The constructor has one required parameter, a `string $name`; you will need to
call `parent::__construct($name)` if you override the constructor (e.g., to
supply other required arguments).

Many input types can likely allow multiple values. For example, if you have a
`--path` option where you want to allow the user to supply a set of paths on
which to perform an operation, or a `--class` operation to allow the user to
specify multiple clases to generate. In those scenarios, we provide a trait,
`Laminas\Cli\Input\AllowMultipleTrait`, which defines a single method:

```php
public function setAllowMultipleFlag(bool $flag): self
```

This method will update the `$optionMode` mask to either include or remove the
`InputOption::VALUE_IS_ARRAY` bit. Compose it in your custom param
implementations if you want to allow users the ability to specify more than one
value for a parameter.

Another trait, `Laminas\Cli\Input\StandardQuestionTrait`, provides the method
`createQuestion()`, which returns a `Symfony\Component\Console\Question\Question`
instance with a prompt in the form of:

```text
<question>{description}</question> [<comment>{default}</comment>]:
>
```

This allows an implementation to use the standard format question, and then add
things such as [normalizers](https://symfony.com/doc/current/components/console/helpers/questionhelper.html#normalizing-the-answer),
[validators](https://symfony.com/doc/current/components/console/helpers/questionhelper.html#validating-the-answer),
or [autocompletion](https://symfony.com/doc/current/components/console/helpers/questionhelper.html#autocompletion).

Additionally, when the option mode includes `InputOption::VALUE_IS_ARRAY` (per
the `setAllowMultipleFlag()` method as described previously), the application
will prompt for multiple values using the same question and default value (if a
default is available) until the user presses `Return` without entering anything.

You can compose this trait in your own input param implementations, and call it
from your `getQuestion()` method if that question format will work for you.

## Standard input parameter types

We ship several standard input parameter types for use in your applications. All
parameters require the parameter name as the initial argument, and additional
arguments as specified below.

All parameter types EXCEPT the `BoolParam` compose the `AllowMultipleTrait`,
exposing the `setAllowMultipleFlag()` flag.

Additionally, when writing validators, you do not need to check if the value is
not required and an empty value submitted;
`ParamAwareInputInterface::getParam()` does this for you.

### BoolParam

`Laminas\Cli\Input\BoolParam` allows specifying a parameter with a boolean
value. It emits a `Symfony\Component\Console\Question\ConfirmationQuestion`.

```php
namespace Laminas\Cli\Input;

final class BoolParam extends AbstractInputParam
{
    public function __construct(string $name);
}
```

### ChoiceParam

`Laminas\Cli\Input\ChoiceParam` allows specifying a set of choices from which
the user may select a value, emitting a `Symfony\Component\Console\Question\ChoiceQuestion`.

```php
namespace Laminas\Cli\Input;

final class ChoiceParam extends AbstractInputParam
{
    use StandardQuestionTrait;

    public function __construct(string $name, array $haystack);
}
```

### IntParam

`Laminas\Cli\Input\IntParam` allows specifying that a value must be an integer,
and optionally be more than a minimum value and/or less than a maximum value.

```php
namespace Laminas\Cli\Input;

final class IntParam extends AbstractInputParam
{
    use StandardQuestionTrait;

    public function __construct(string $name);

    public function setMin(?int $min): self;

    public function setMax(?int $max): self;
}
```

### PathParam

`Laminas\Cli\Input\PathParam` allows specifying that a value must be a path on
the filesystem, requiring that you specifiy which, and optionally requiring that
it exist.

```php
namespace Laminas\Cli\Input;

final class PathParam extends AbstractInputParam
{
    use StandardQuestionTrait;

    public const TYPE_DIR  = 'dir';
    public const TYPE_FILE = 'file';

    /**
     * @param string $pathType One of the TYPE_* constants, indicating whether
     *     the path expected should be a directory or a file.
     */
    public function __construct(string $name, string $pathType);

    public function setPathMustExist(bool $flag): self;
}
```

### StringParam

`Laminas\Cli\Input\StringParam` allows specifying that a value must be a string,
and optionally match a PCRE regex.

```php
namespace Laminas\Cli\Input;

final class StringParam extends AbstractInputParam
{
    use StandardQuestionTrait;

    public function __construct(string $name);

    /**
     * @param string $pattern A valid PCRE regex pattern.
     */
    public function setPattern(string $pattern): self;
}
```

## Adding input parameters to a command

In order to define input parameters in your command, your command will need to
extend `Laminas\Cli\Command\AbstractParamAwareCommand`. That class class extends
`Symfony\Component\Console\Command\Command` and provides one additional method,
`addParam()`, for adding an input parameter. The method accepts a single
parameter, a `Laminas\Cli\Input\InputParamInterface` instance.

As an example, the following command adds a "name" parameter that expects a
string:

```php
use Laminas\Cli\Command\AbstractParamAwareCommand;
use Laminas\Cli\Input\ParamAwareInputInterface;
use Laminas\Cli\Input\StringParam;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

final class HelloCommand extends AbstractParamAwareCommand
{
    /** @var string */
    public static $defaultName = 'example:hello';

    protected function configure() : void
    {
        $this->setName(self::$defaultName);
        $this->addParam(
            (new StringParam('name'))
            ->setDescription('Your name')
            ->setShortcut('n')
        );
    }

    /**
     * @param ParamAwareInputInterface $input
     */
    protected function execute(InputInterface $input, OutputInterface $output) : int
    {
        $name = $input->getParam('name');
        $output->writeln('Hello ' . $name . '!');

        return 0;
    }
}
```

> ### ParamAwareInputInterface
>
> Internally, we decorate the input instance from the
> application with a class implementing our own `ParamAwareInputInterface`. This
> interface extends each of `Symfony\Component\Console\Input\InputInterface` and
> `StreamableInputInterface`, and adds the method `getParam(string $name)` for
> retrieving the input parameter. To ensure IDE completion and static analysis
> work as expected, you should typehint the `$input` argument as a
> `ParamAwareInputInterface` via a parameter annotation. (You cannot do so via
> type hint, as it would break compatibility with the `AbstractCommand`
> signature.)

Adding parameters is similar to adding arguments or options, with one key
difference: you provide an instance, instead of a series of values. The
`InputParamInterface` and the various concrete implementations define builder
methods that return `$this`, allowing you to create the instances in-line in a
declarative manner.

Accessing the parameter value is exactly like accessing an option or argument.
However, unlike those items, parameters will prompt for values if they are
missing.

Let's see what happens when we call the command without the parameter:

```console
$ vendor/bin/laminas example:hello
Your name:
 > Michal

Hello Michal!
```

If we provide the param value as an option during invocation:

```console
$ vendor/bin/laminas example:hello --name=Michal

Hello Michal!
```

As you can see, there is no prompt!

> ### AbstractParamAwareCommand::run
>
> The `AbstractParamAwareCommand` class overrides the symfony/console
> `Command::run()` method internally in order to decorate the input in a
> `ParamAwareInputInterface` implementation. If you decide you need to override
> the method yourself you MUST add one of the following lines within your
> implementation:
>
> ```php
> // If you will be calling the parent run() method:
> parent::run($input, $output);
>
> // If you will not be calling the parent run() method:
> $input = $this->decorateInputToBeParamAware($input, $output);
> ```

## Creating custom parameter types

If none of the [standard input parameter types](#standard-input-parameter-types)
satisfy your use case, you can define a custom parameter type.

As an example:

```php
use Laminas\Cli\Input\AbstractInputParam;
use Symfony\Component\Console\Input\InputOption;
use Symfony\Component\Console\Question\Question;

class CustomParam extends AbstractInputParam
{
    public function getQuestion(): Question
    {
        $customQuestion = new Question('Please provide value for custom parameter:');
        // some question modifications, like:
        $customQuestion->setValidator(static function (string $value) { /* ... */ });
        $customQuestion->setAutocompleterCallback(static function (string $answer) { /* ... */ });
        $customQuestion->setNormalizer(static function (string $answer) { /* ... */ });
        // etc...
        return $question;
    }
}
```

In your command, you would then add it:

```php
$this->addParam(
  (new CustomParam('custom'))
      ->setDescription('Custom parameter')
      ->setRequiredFlag(true)
);
```

Alternately, you could define it as an anonymous class implementation, inline
where you add the parameter:

```php
$this->addParam(
  (new class('custom') extends \Laminas\Cli\Input\AbstractInputParam
      {
          public function getQuestion(): Question
          {
              $customQuestion = new Question('Please provide value for custom parameter:');
              // some question modifications, like:
              $customQuestion->setValidator(static function (string $value) { /* ... */ });
              $customQuestion->setAutocompleterCallback(static function (string $answer) { /* ... */ });
              $customQuestion->setNormalizer(static function (string $answer) { /* ... */ });
              // etc...
              return $question;
          }
      }
  )
      ->setDescription('Custom parameter')
      ->setRequiredFlag(true)
);
```
