![](http://s17.postimg.org/tp1d3l08b/workflow.jpg "Workflow")

# Workflow

[![Build Status](http://img.shields.io/travis/cerbero90/Workflow.svg?style=flat-square)](https://travis-ci.org/cerbero90/Workflow)
[![Latest Stable Version](http://img.shields.io/packagist/v/cerbero/Workflow.svg?style=flat-square&label=stable)](https://packagist.org/packages/cerbero/workflow)
[![License](http://img.shields.io/packagist/l/cerbero/Workflow.svg?style=flat-square)](https://packagist.org/packages/cerbero/workflow)
[![Code Climate](https://img.shields.io/codeclimate/github/cerbero90/Workflow.svg?style=flat-square)](https://codeclimate.com/github/cerbero90/Workflow)
[![Scrutinizer](https://img.shields.io/scrutinizer/g/cerbero90/Workflow.svg?style=flat-square)](https://scrutinizer-ci.com/g/cerbero90/Workflow/)
[![Gratipay](https://img.shields.io/gratipay/cerbero.svg?style=flat-square)](https://gratipay.com/cerbero/)

Let's assume we want to develop a registration process, the main thing is storing user data, but we also want to validate input and send a welcome email.

Now imagine this process as concentric circles: the storing of user data is the inner circle, whereas validation, email sending and any other logic are the increasingly outer circles.

These concentric circles, or pipes, can be called all at once in a pipeline and have several advantages including running specific code before or after a given command and adding or removing any logic without even touching the controllers.

This package is intended to automate the creation of such pipelines by using simple Artisan commands and owns other facilities like auto-validation, management of pipelines in a single file and visualization of their graphical representation in console.

> **Note**: if you are using Laravel 4, please refer to [this version](https://github.com/cerbero90/Workflow/tree/2.1.0) that leverages the decorators design pattern.

## Installation

Run this command in your application root:
```
composer require cerbero/workflow
```

and add this string to the `providers` array in `config/app.php`:
```php
'Cerbero\Workflow\WorkflowServiceProvider',
```

## Configuration

By default all the generated workflows are placed in `app/Workflows`. You can change it by running:
```
php artisan vendor:publish
```

and then edit the file `config/workflow.php`.

## Create a workflow

Taking the introductive example, let's create the registration workflow by running:

```
php artisan workflow:create RegisterUser --attach="notifier"
```

The following files are automatically created under the `app` directory:

File (click to see an example)                   | Description
------------------------------------------------ | -----------
[Commands/RegisterUserCommand.php][command]      | incapsulates the current task
[Http/Requests/RegisterUserRequest.php][request] | contains the validation rules and permissions
[Workflows/RegisterUser/Notifier.php][notifier]  | the attached pipe to send the welcome email
[Workflows/workflows.yml][workflows]             | contains all the created pipelines with their own pipes

[command]: https://github.com/cerbero90/workflow-demo/blob/master/app/Commands/RegisterUserCommand.php
[request]: https://github.com/cerbero90/workflow-demo/blob/master/app/Http/Requests/RegisterUserRequest.php
[notifier]: https://github.com/cerbero90/workflow-demo/blob/master/app/Workflows/RegisterUser/Notifier.php
[workflows]: https://github.com/cerbero90/workflow-demo/blob/master/app/Workflows/workflows.yml

While both [commands](http://laravel.com/docs/5.0/bus) and [requests](http://laravel.com/docs/5.0/validation#form-request-validation) are well documented, let's have a look at the newly created `Notifier` class:

```php
<?php namespace App\Workflows\RegisterUser;

use Cerbero\Workflow\Pipes\AbstractPipe;

class Notifier extends AbstractPipe {

	/**
	 * Run before the command is handled.
	 *
	 * @param	App\Commands\Command	$command
	 * @return	mixed
	 */
	public function before($command)
	{
		//
	}

	/**
	 * Run after the handled command.
	 *
	 * @param	mixed	$handled
	 * @param	App\Commands\Command	$command
	 * @return	mixed
	 */
	public function after($handled, $command)
	{
		//
	}

}
```

This class has two methods: `before()` is run before handling the `RegisterUserCommand`, which is passed as argument, whereas `after()` is run after handling the command and also has the value returned by the handled command as parameter e.g. the User model.

You can inject whatever you need in both methods, everything is resolved by the service container automatically. Furthermore if you don't need either `before()` or `after()`, you can safely delete the unused method.

All workflows are stored within the `workflows.yml` file that makes it easier to read and update pipelines and their pipes even if, as you will see, there are also Artisan commands to automate these tasks.

As you may have noticed, you only created a Notifier and no class to validate data. Every workflow validates the input automatically by reading the validation rules and permissions from the generated requests e.g. `RegisterUserRequest`.

But sometimes you may not have input to validate, in these cases you can avoid the creation of the Request class by using the `--unguard` flag:

```
php artisan workflow:create TakeItEasy --unguard
```

Here is a recap of the `workflow:create` options:

Option    | Shortcut | Description
--------- | -------- | -----------
--attach  | -a       | The pipes to attach to the workflow
--unguard | -u       | Do not make this workflow validate data

## Read a workflow

To better understand the workflow of a given pipeline, you can run the command:

```
php artisan workflow:read RegisterUser
```
that will output something similar:

```
           ║
╔══════════╬══════════╗
║ Notifier ║ before() ║
║ ╔════════╩════════╗ ║
║ ║  RegisterUser   ║ ║
║ ╚════════╦════════╝ ║
║ Notifier ║ after()  ║
╚══════════╬══════════╝
           ∇
```
The arrow in the middle of the drawing shows the itinerary of the code within the pipeline. Please note how the `Notifier` pipe wraps the `RegisterUser` command and how its methods are run before and after the command respectively.

## Usage

Now that you have created the registration workflow and seen how it works in theory, it's time to see it in action.

There are many ways to run the workflow. To let all controllers have a `$workflow` property after being resolved, edit the `app/Http/Controllers/Controller.php` class like so:

```php
use Cerbero\Workflow\RunsWorkflow;
use Cerbero\Workflow\WorkflowRunner;

abstract class Controller extends BaseController implements WorkflowRunner {

	use RunsWorkflow;

}

```
and now you can run your workflow by calling it through the `$workflow` property in every controller:

```php
public function store()
{
	$this->workflow->registerUser();
}
```
Otherwise if you prefer to run workflow only in some controllers (or even non-controllers), you can type-hint the `Cerbero\Workflow\Workflow` class:

```php
use Cerbero\Workflow\Workflow;

class RegisterController extends Controller {

	public function __construct(Workflow $workflow)
	{
		$this->workflow = $workflow;
	}

	public function store()
	{
		$this->workflow->registerUser();
	}

}
```
Finally, you can use the `Workflow` facade that has been automatically registered during the installation, just for you:

```php
public function store()
{
	\Workflow::registerUser();
}
```

## Update a workflow

One of the biggest advantages of using pipelines is that you can easily add and remove functionalities keeping the rest of your code intact.

The Artisan command `workflow:update` can attach and detach pipes of existing pipelines:

```
php artisan workflow:update RegisterUser --attach="uploader logger" --detach="notifier"
```

By default the detached pipes are not deleted, if you want to, you can use the `--force` flag:

```
php artisan workflow:update RegisterUser --detach="notifier" --force
```

Here is a recap of the `workflow:update` options:

Option    | Shortcut | Description
--------- | -------- | -----------
--attach  | -a       | The pipes to attach to the workflow
--detach  | -d       | The pipes to detach from the workflow
--force   | -f       | Delete the files of detached pipes

## Delete a workflow

To delete an entire pipeline, you can run:

```
php artisan workflow:delete RegisterUser
```

Like the update command, the detached pipes are not deleted by default, again you can use the `--force` flag:

```
php artisan workflow:delete RegisterUser --force
```
