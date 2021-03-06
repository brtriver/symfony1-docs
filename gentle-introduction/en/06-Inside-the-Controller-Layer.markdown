Chapter 6 - Inside The Controller Layer
=======================================

In symfony, the controller layer, which contains the code linking the business logic and the presentation, is split into several components that you use for different purposes:

  * The front controller is the unique entry point to the application. It loads the configuration and determines the action to execute.
  * Actions contain the applicative logic. They check the integrity of the request and prepare the data needed by the presentation layer.
  * The request, response, and session objects give access to the request parameters, the response headers, and the persistent user data. They are used very often in the controller layer.
  * Filters are portions of code executed for every request, before or after the action. For example, the security and validation filters are commonly used in web applications. You can extend the framework by creating your own filters.

This chapter describes all these components, but don't be intimidated by their number. For a basic page, you will probably need to write only a few lines in the action class, and that's all. The other controller components will be of use only in specific situations.

The Front Controller
--------------------

All web requests are handled by a single front controller, which is the unique entry point to the whole application in a given environment.

When the front controller receives a request, it uses the routing system to match an action name and a module name with the URL typed (or clicked) by the user. For instance, the following request URL calls the `index.php` script (that's the front controller) and will be understood as a call to the action `myAction` of the module `mymodule`:

    http://localhost/index.php/mymodule/myAction

If you are not interested in symfony's internals, that's all that you need to know about the front controller. It is an indispensable component of the symfony MVC architecture, but you will seldom need to change it. So you can jump to the next section unless you really want to know about the guts of the front controller.

### The Front Controller's Job in Detail

The front controller does the dispatching of the request, but that means a little more than just determining the action to execute. In fact, it executes the code that is common to all actions, including the following:

  1. Load the project configuration class and the symfony libraries.
  2. Create an application configuration and a symfony context.
  3. Load and initiate the core framework classes.
  4. Load the configuration.
  5. Decode the request URL to determine the action to execute and the request parameters.
  6. If the action does not exist, redirect to the 404 error action.
  7. Activate filters (for instance, if the request needs authentication).
  8. Execute the filters, first pass.
  9. Execute the action and render the view.
  10. Execute the filters, second pass.
  11. Output the response.

### The Default Front Controller

The default front controller, called `index.php` and located in the `web/` directory of the project, is a simple PHP file, as shown in Listing 6-1.

Listing 6-1 - The Default Production Front Controller

    [php]
    <?php
    require_once(dirname(__FILE__).'/../config/ProjectConfiguration.class.php');

    $configuration = ProjectConfiguration::getApplicationConfiguration('frontend', 'prod', false);
    sfContext::createInstance($configuration)->dispatch();

The front controller creates an application configuration instance, which takes care of steps 2 through 4. The call to the `dispatch()` method of the `sfController` object (which is the core controller object of the symfony MVC architecture) dispatches the request, taking care of steps 5 through 7. The last steps are handled by the filter chain, as explained later in this chapter.

### Calling Another Front Controller to Switch the Environment

One front controller exists per environment. As a matter of fact, it is the very existence of a front controller that defines an environment. The environment is defined by the second argument you pass to the `ProjectConfiguration::getApplicationConfiguration()` method call.

To change the environment in which you're browsing your application, just choose another front controller. The default front controllers available when you create a new application with the `generate:app` task are `index.php` for the production environment and `frontend_dev.php` for the development environment (provided that your application is called `frontend`). The default `mod_rewrite` configuration will use `index.php` when the URL doesn't contain a front controller script name. So both of these URLs display the same page (`mymodule/index`) in the production environment:

    http://localhost/index.php/mymodule/index
    http://localhost/mymodule/index

and this URL displays that same page in the development environment:

    http://localhost/frontend_dev.php/mymodule/index

Creating a new environment is as easy as creating a new front controller. For instance, you may need a staging environment to allow your customers to test the application before going to production. To create this staging environment, just copy `web/frontend_dev.php` into `web/frontend_staging.php`, and change the value of the second argument of the `ProjectConfiguration::getApplicationConfiguration()` call to `staging`. Now, in all the configuration files, you can add a new `staging:` section to set specific values for this environment, as shown in Listing 6-2.

Listing 6-2 - Sample `app.yml` with Specific Settings for the Staging Environment

    staging:
      mail:
        webmaster:    dummy@mysite.com
        contact:      dummy@mysite.com
    all:
      mail:
        webmaster:    webmaster@mysite.com
        contact:      contact@mysite.com

If you want to see how the application reacts in this new environment, call the related front controller:

    http://localhost/frontend_staging.php/mymodule/index

Actions
-------

The actions are the heart of an application, because they contain all the application's logic. They call the model and define variables for the view. When you make a web request in a symfony application, the URL defines an action and the request parameters.

### The Action Class

Actions are methods named `executeActionName` of a class named `moduleNameActions` inheriting from the `sfActions` class, and grouped by modules. The action class of a module is stored in an `actions.class.php` file, in the module's `actions/` directory.

Listing 6-3 shows an example of an `actions.class.php` file with only an `index` action for the whole `mymodule` module.

Listing 6-3 - Sample Action Class, in `apps/frontend/modules/mymodule/actions/actions.class.php`

    [php]
    class mymoduleActions extends sfActions
    {
      public function executeIndex($request)
      {
        // ...
      }
    }

>**CAUTION**
>Even if method names are not case-sensitive in PHP, they are in symfony. So don't forget that the action methods must start with a lowercase `execute`, followed by the exact action name with the first letter capitalized.

In order to request an action, you need to call the front controller script with the module name and action name as parameters. By default, this is done by appending the couple `module_name`/`action_name` to the script. This means that the action defined in Listing 6-4 can be called by this URL:

    http://localhost/index.php/mymodule/index

Adding more actions just means adding more `execute` methods to the `sfActions` object, as shown in Listing 6-4.

Listing 6-4 - Action Class with Two Actions, in `frontend/modules/mymodule/actions/actions.class.php`

    [php]
    class mymoduleActions extends sfActions
    {
      public function executeIndex($request)
      {
        // ...
      }

      public function executeList($request)
      {
        // ...
      }
    }

If the size of an action class grows too much, you probably need to do some refactoring and move some code to the model layer. Actions should often be kept short (not more than a few lines), and all the business logic should usually be in the model.

Still, the number of actions in a module can be important enough to lead you to split it in two modules.

>**SIDEBAR**
>Symfony coding standards
>
>In the code examples given in this book, you probably noticed that the opening and closing curly braces (`{` and `}`) occupy one line each. This standard makes the code easier to read.
>
>Among the other coding standards of the framework, indentation is always done by two blank spaces; tabs are not used. This is because tabs have a different space value according to the text editor you use, and because code with mixed tab and blank indentation is impossible to read.
>
>Core and generated symfony PHP files do not end with the usual `?>` closing tag. This is because it is not really needed, and because it can create problems in the output if you ever have blanks after this tag.
>
>And if you really pay attention, you will see that a line never ends with a blank space in symfony. The reason, this time, is more prosaic: lines ending with blanks look ugly in Fabien's text editor.

### Alternative Action Class Syntax

An alternative action syntax is available to dispatch the actions in separate files, one file per action. In this case, each action class extends `sfAction` (instead of `sfActions`) and is named `actionNameAction`. The actual action method is simply named `execute`. The file name is the same as the class name. This means that the equivalent of Listing 6-4 can be written with the two files shown in Listings 6-5 and 6-6.

Listing 6-5 - Single Action File, in `frontend/modules/mymodule/actions/indexAction.class.php`

    [php]
    class indexAction extends sfAction
    {
      public function execute($request)
      {
        // ...
      }
    }

Listing 6-6 - Single Action File, in `frontend/modules/mymodule/actions/listAction.class.php`

    [php]
    class listAction extends sfAction
    {
      public function execute($request)
      {
        // ...
      }
    }

### Retrieving Information in the Action

The action class offers a way to access controller-related information and the core symfony objects. Listing 6-7 demonstrates how to use them.

Listing 6-7 - `sfActions` Common Methods

    [php]
    class mymoduleActions extends sfActions
    {
      public function executeIndex(sfWebRequest $request)
      {
        // Retrieving request parameters
        $password    = $request->getParameter('password');

        // Retrieving controller information
        $moduleName  = $this->getModuleName();
        $actionName  = $this->getActionName();

        // Retrieving framework core objects
        $userSession = $this->getUser();
        $response    = $this->getResponse();
        $controller  = $this->getController();
        $context     = $this->getContext();

        // Setting action variables to pass information to the template
        $this->setVar('foo', 'bar');
        $this->foo = 'bar';            // Shorter version
      }
    }

>**SIDEBAR**
>The context singleton
>
>You already saw, in the front controller, a call to `sfContext::createInstance()`. In an action, the `getContext()` method returns the same singleton. It is a very useful object that stores a reference to all the symfony core objects related to a given request, and offers an accessor for each of them:
>
>`sfController`: The controller object (`->getController()`)
>
>`sfRequest`: The request object (`->getRequest()`)
>
>`sfResponse`: The response object (`->getResponse()`)
>
>`sfUser`: The user session object (`->getUser()`)
>
>`sfRouting`: The routing object (`->getRouting()`)
>
>`sfMailer`: The mailer object (`->getMailer()`)
>
>`sfI18N`: The internationalization object (`->getI18N()`)
>
>`sfLogger`: The logger object (`->getLogger()`)
>
>`sfDatabaseConnection`: The database connection (`->getDatabaseConnection()`)
>
>All these core objects are availables through the `sfContext::getInstance()` singleton from any part of the code. However, it's a really bad practice because this will create some hard dependencies making your code really hard to test, reuse and maintain. You will learn in this book how to avoid the usage of `sfContext::getInstance()`. 

### Action Termination

Various behaviors are possible at the conclusion of an action's execution. The value returned by the action method determines how the view will be rendered. Constants of the `sfView` class are used to specify which template is to be used to display the result of the action.

If there is a default view to call (this is the most common case), the action should end as follows:

    [php]
    return sfView::SUCCESS;

Symfony will then look for a template called `actionNameSuccess.php`. This is defined as the default action behavior, so if you omit the `return` statement in an action method, symfony will also look for an `actionNameSuccess.php` template. Empty actions will also trigger that behavior. See Listing 6-8 for examples of successful action termination.

Listing 6-8 - Actions That Will Call the `indexSuccess.php` and `listSuccess.php` Templates

    [php]
    public function executeIndex()
    {
      return sfView::SUCCESS;
    }

    public function executeList()
    {
    }

If there is an error view to call, the action should end like this:

    [php]
    return sfView::ERROR;

Symfony will then look for a template called `actionNameError.php`.

To call a custom view, use this ending:

    [php]
    return 'MyResult';

Symfony will then look for a template called `actionNameMyResult.php`.

If there is no view to call--for instance, in the case of an action executed in a batch process--the action should end as follows:

    [php]
    return sfView::NONE;

No template will be executed in that case. It means that you can bypass completely the view layer and set the response HTML code directly from an action. As shown in Listing 6-9, symfony provides a specific `renderText()` method for this case. This can be useful when you need extreme responsiveness of the action, such as for Ajax interactions, which will be discussed in Chapter 11.

Listing 6-9 - Bypassing the View by Echoing the Response and Returning `sfView::NONE`

    [php]
    public function executeIndex()
    {
      $this->getResponse()->setContent("<html><body>Hello, World!</body></html>");

      return sfView::NONE;
    }

    // Is equivalent to
    public function executeIndex()
    {
      return $this->renderText("<html><body>Hello, World!</body></html>");
    }

In some cases, you need to send an empty response but with some headers defined in it (especially the `X-JSON` header). Define the headers via the `sfResponse` object, discussed in the next chapter, and return the `sfView::HEADER_ONLY` constant, as shown in Listing 6-10.

Listing 6-10 - Escaping View Rendering and Sending Only Headers

    [php]
    public function executeRefresh()
    {
      $output = '<"title","My basic letter"],["name","Mr Brown">';
      $this->getResponse()->setHttpHeader("X-JSON", '('.$output.')');

      return sfView::HEADER_ONLY;
    }

If the action must be rendered by a specific template, ignore the `return` statement and use the `setTemplate()` method instead.

    [php]
    public function executeIndex()
    {
      $this->setTemplate('myCustomTemplate');
    }
    
With this code, symfony will look for a `myCustomTemplateSuccess.php` file, instead of `indexSuccess.php`.

### Skipping to Another Action

In some cases, the action execution ends by requesting a new action execution. For instance, an action handling a form submission in a POST request usually redirects to another action after updating the database.

The action class provides two methods to execute another action:

  * If the action forwards the call to another action:

        [php]
        $this->forward('otherModule', 'index');

  * If the action results in a web redirection:

        [php]
        $this->redirect('otherModule/index');
        $this->redirect('http://www.google.com/');


>**NOTE**
>The code located after a forward or a redirect in an action is never executed. You can consider that these calls are equivalent to a `return` statement. They throw an `sfStopException` to stop the execution of the action; this exception is later caught by symfony and simply ignored.

The choice between a redirect or a forward is sometimes tricky. To choose the best solution, keep in mind that a forward is internal to the application and transparent to the user. As far as the user is concerned, the displayed URL is the same as the one requested. In contrast, a redirect is a message to the user's browser, involving a new request from it and a change in the final resulting URL.

If the action is called from a submitted form with `method="post"`, you should **always** do a redirect. The main advantage is that if the user refreshes the resulting page, the form will not be submitted again; in addition, the back button works as expected by displaying the form and not an alert asking the user if he wants to resubmit a POST request.

There is a special kind of forward that is used very commonly. The `forward404()` method forwards to a "page not found" action. This method is often called when a parameter necessary to the action execution is not present in the request (thus detecting a wrongly typed URL). Listing 6-11 shows an example of a `show` action expecting an `id` parameter.

Listing 6-11 - Use of the `forward404()` Method

    [php]
    public function executeShow(sfWebRequest $request)
    {
      // Doctrine
      $article = Doctrine::getTable('Article')->find($request->getParameter('id'));
      
      // Propel
      $article = ArticlePeer::retrieveByPK($request->getParameter('id'));
      
      if (!$article)
      {
        $this->forward404();
      }
    }

>**TIP**
>If you are looking for the error 404 action and template, you will find them in the `$sf_symfony_ lib_dir/controller/default/` directory. You can customize this page by adding a new `default` module to your application, overriding the one located in the framework, and by defining an `error404` action and an error404Success template inside. Alternatively, you can set the `error_404_module` and `error_404_action` constants in the `settings.yml` file to use an existing action.

Experience shows that, most of the time, an action makes a redirect or a forward after testing something, such as in Listing 6-12. That's why the `sfActions` class has a few more methods, named `forwardIf()`, `forwardUnless()`, `forward404If()`, `forward404Unless()`, `redirectIf()`, and `redirectUnless()`. These methods simply take an additional parameter representing a condition that triggers the execution if tested true (for the `xxxIf()` methods) or false (for the `xxxUnless()` methods), as illustrated in Listing 6-12.

Listing 6-12 - Use of the `forward404If()` Method

    [php]
    // This action is equivalent to the one shown in Listing 6-11
    public function executeShow(sfWebRequest $request)
    {
      $article = Doctrine::getTable('Article')->find($request->getParameter('id'));
      $this->forward404If(!$article);
    }

    // So is this one
    public function executeShow(sfWebRequest $request)
    {
      $article = Doctrine::getTable('Article')->find($request->getParameter('id'));
      $this->forward404Unless($article);
    }

Using these methods will not only keep your code short, but it will also make it more readable.

>**TIP**
>When the action calls `forward404()` or its fellow methods, symfony throws an `sfError404Exception` that manages the 404 response. This means that if you need to display a 404 message from somewhere where you don't want to access the controller, you can just throw a similar exception.

### Repeating Code for Several Actions of a Module

The convention to name actions `executeActionName()` (in the case of an `sfActions` class) or `execute()` (in the case of an `sfAction` class) guarantees that symfony will find the action method. It gives you the ability to add other methods of your own that will not be considered as actions, as long as they don't start with `execute`.

There is another useful convention for when you need to repeat several statements in each action before the actual action execution. You can then extract them into the `preExecute()` method of your action class. You can probably guess how to repeat statements after every action is executed: wrap them in a `postExecute()` method. The syntax of these methods is shown in Listing 6-13.

Listing 6-13 - Using `preExecute()`, `postExecute()`, and Custom Methods in an Action Class

    [php]
    class mymoduleActions extends sfActions
    {
      public function preExecute()
      {
        // The code inserted here is executed at the beginning of each action call
        ...
      }

      public function executeIndex($request)
      {
        ...
      }

      public function executeList($request)
      {
        ...
        $this->myCustomMethod();  // Methods of the action class are accessible
      }

      public function postExecute()
      {
        // The code inserted here is executed at the end of each action call
        ...
      }

      protected function myCustomMethod()
      {
        // You can also add your own methods, as long as they don't start with "execute"
        // In that case, it's better to declare them as protected or private
        ...
      }
    }

>**TIP**
>As the pre/post execute methods are called for **each** actions of the current module, be sure you really need the execute this code for **all** your actions, to avoid unwanted side-effects.

Accessing the Request
---------------------

The first argument passed to any action method is the request object, called `sfWebRequest` in symfony. You're already familiar with the `getParameter('myparam')` method, used to retrieve the value of a request parameter by its name. Table 6-1 lists the most useful `sfWebRequest` methods.

Table 6-1 - Methods of the `sfWebRequest` Object

Name                             | Function                               |  Sample Output
-------------------------------- | -------------------------------------- | -----------------------------------------------------------------------
**Request Information**          |                                        |
`isMethod($method)`              | Is it a post or a get?                 | true or false
`getMethod()`                    | Request method name                    | `'POST'`
`getHttpHeader('Server')`        | Value of a given HTTP header           | `'Apache/2.0.59 (Unix) DAV/2 PHP/5.1.6'`
`getCookie('foo')`               | Value of a named cookie                | `'bar'`
`isXmlHttpRequest()`*            | Is it an Ajax request?                 | `true`
`isSecure()`                     | Is it an SSL request?                  | `true`
**Request Parameters**           |                                        |
`hasParameter('foo')`            | Is a parameter present in the request? | `true`
`getParameter('foo')`            | Value of a named parameter             | `'bar'`
`getParameterHolder()->getAll()` | Array of all request parameters        |
**URI-Related Information**      |                                        |
`getUri()`                       | Full URI                               | `'http://localhost/frontend_dev.php/mymodule/myaction'`
`getPathInfo()`                  | Path info                              | `'/mymodule/myaction'`
`getReferer()`**                 | Referrer                               | `'http://localhost/frontend_dev.php/'`
`getHost()`                      | Host name                              | `'localhost'`
`getScriptName()`                | Front controller path and name         | `'frontend_dev.php'`
**Client Browser Information**   |                                        |
`getLanguages()`                 | Array of accepted languages            | `Array( ` ` [0] => fr ` ` [1] => fr_FR ` ` [2] => en_US ` ` [3] => en )`
`getCharsets()`                  | Array of accepted charsets             | `Array( ` ` [0] => ISO-8859-1 ` ` [1] => UTF-8 ` ` [2] => * )`
getAcceptableContentTypes()      | Array of accepted content types        | `Array( [0] => text/xml [1] => text/html`

`*` *Works with prototype, Prototype, Mootools, and jQuery*

`**` *Sometimes blocked by proxies*

You don't have to worry about whether your server supports the `$_SERVER` or the `$_ENV` variables, or about default values or server-compatibility issues--the `sfWebRequest` methods do it all for you. Besides, their names are so evident that you will no longer need to browse the PHP documentation to find out how to get information from the request.

User Session
------------

Symfony automatically manages user sessions and is able to keep persistent data between requests for users. It uses the built-in PHP session-handling mechanisms and enhances them to make them more configurable and easier to use.

### Accessing the User Session

The session object for the current user is accessed in the action with the `getUser()` method and is an instance of the `sfUser` class. This class contains a parameter holder that allows you to store any user attribute in it. This data will be available to other requests until the end of the user session, as shown in Listing 6-14. User attributes can store any type of data (strings, arrays, and associative arrays). They can be set for every individual user, even if that user is not identified.

Listing 6-14 - The `sfUser` Object Can Hold Custom User Attributes Existing Across Requests

    [php]
    class mymoduleActions extends sfActions
    {
      public function executeFirstPage($request)
      {
        $nickname = $request->getParameter('nickname');

        // Store data in the user session
        $this->getUser()->setAttribute('nickname', $nickname);
      }

      public function executeSecondPage()
      {
        // Retrieve data from the user session with a default value
        $nickname = $this->getUser()->getAttribute('nickname', 'Anonymous Coward');
      }
    }

>**CAUTION**
>You can store objects in the user session, but it is strongly discouraged. This is because the session object is serialized between requests. When the session is deserialized, the class of the stored objects must already be loaded, and that's not always the case. In addition, there can be "stalled" objects if you store Propel or Doctrine objects.

Like many getters in symfony, the `getAttribute()` method accepts a second argument, specifying the default value to be used when the attribute is not defined. To check whether an attribute has been defined for a user, use the `hasAttribute()` method. The attributes are stored in a parameter holder that can be accessed by the `getAttributeHolder()` method. It allows for easy cleanup of the user attributes with the usual parameter holder methods, as shown in Listing 6-15.

Listing 6-15 - Removing Data from the User Session

    [php]
    class mymoduleActions extends sfActions
    {
      public function executeRemoveNickname()
      {
        $this->getUser()->getAttributeHolder()->remove('nickname');
      }

      public function executeCleanup()
      {
        $this->getUser()->getAttributeHolder()->clear();
      }
    }

The user session attributes are also available in the templates by default via the `$sf_user` variable, which stores the current `sfUser` object, as shown in Listing 6-16.

Listing 6-16 - Templates Also Have Access to the User Session Attributes

    [php]
    <p>
      Hello, <?php echo $sf_user->getAttribute('nickname') ?>
    </p>

### Flash Attributes

A recurrent problem with user attributes is the cleaning of the user session once the attribute is not needed anymore. For instance, you may want to display a confirmation after updating data via a form. As the form-handling action makes a redirect, the only way to pass information from this action to the action it redirects to is to store the information in the user session. But once the confirmation message is displayed, you need to clear the attribute; otherwise, it will remain in the session until it expires.

The flash attribute is an ephemeral attribute that you can define and forget, knowing that it will disappear after the very next request and leave the user session clean for the future. In your action, define the flash attribute like this:

    [php]
    $this->getUser()->setFlash('notice', $value);

The template will be rendered and delivered to the user, who will then make a new request to another action. In this second action, just get the value of the flash attribute like this:

    [php]
    $value = $this->getUser()->getFlash('notice');

Then forget about it. After delivering this second page, the `notice` flash attribute will be flushed. And even if you don't require it during this second action, the flash will disappear from the session anyway.

If you need to access a flash attribute from a template, use the `$sf_user` object:

    [php]
    <?php if ($sf_user->hasFlash('notice')): ?>
      <?php echo $sf_user->getFlash('notice') ?>
    <?php endif; ?>

or just:

    [php]
    <?php echo $sf_user->getFlash('notice') ?>

Flash attributes are a clean way of passing information to the very next request.

### Session Management

Symfony's session-handling feature completely masks the client and server storage of the session IDs to the developer. However, if you want to modify the default behaviors of the session-management mechanisms, it is still possible. This is mostly for advanced users.

On the client side, sessions are handled by cookies. The symfony session cookie is called `symfony`, but you can change its name by editing the `factories.yml` configuration file, as shown in Listing 6-17.

Listing 6-17 - Changing the Session Cookie Name, in `apps/frontend/config/factories.yml`

    all:
      storage:
        class: sfSessionStorage
        param:
          session_name: my_cookie_name

>**TIP**
>The session is started (with the PHP function `session_start()`) only if the `auto_start` parameter is set to true in `factories.yml` (which is the case by default). If you want to start the user session manually, disable this setting of the storage factory.

Symfony's session handling is based on PHP sessions. This means that if you want the client-side management of sessions to be handled by URL parameters instead of cookies, you just need to change the `use_trans_sid` setting in your php.ini. Be aware that this is not recommended.

    session.use_trans_sid = 1

On the server side, symfony stores user sessions in files by default. You can store them in your database by changing the value of the `class` parameter in `factories.yml`, as shown in Listing 6-18.

Listing 6-18 - Changing the Server Session Storage, in `apps/frontend/config/factories.yml`

    all:
      storage:
        class: sfMySQLSessionStorage
        param:
          db_table:    session              # Name of the table storing the sessions
          database:    propel               # Name of the database connection to use
          # Optional parameters
          db_id_col:   sess_id              # Name of the column storing the session id
          db_data_col: sess_data            # Name of the column storing the session data
          db_time_col: sess_time            # Name of the column storing the session timestamp

The `database` setting defines the database connection to be used. Symfony will then use `databases.yml` (see Chapter 8) to determine the connection settings (host, database name, user, and password) for this connection.

The available session storage classes are `sfCacheSessionStorage`, `sfMySQLSessionStorage`, `sfMySQLiSessionStorage`, `sfPostgreSQLSessionStorage`, and `sfPDOSessionStorage`; the latter is preferred. To disable session storage completely, you can use the `sfNoStorage` class.

Session expiration occurs automatically after 30 minutes. This default setting can be modified for each environment in the same `factories.yml` configuration file, but this time in the `user` factory, as shown in Listing 6-19.

Listing 6-19 - Changing Session Lifetime, in `apps/frontend/config/factories.yml`

    all:
      user:
        class:       myUser
        param:
          timeout:   1800           # Session lifetime in seconds

To learn more about factories, refer to Chapter 19.

Action Security
---------------

The ability to execute an action can be restricted to users with certain privileges. The tools provided by symfony for this purpose allow the creation of secure applications, where users need to be authenticated before accessing some features or parts of the application. Securing an application requires two steps: declaring the security requirements for each action and logging in users with privileges so that they can access these secure actions.

### Access Restriction

Before being executed, every action passes by a special filter that checks if the current user has the privileges to access the requested action. In symfony, privileges are composed of two parts:

  * Secure actions require users to be authenticated.
  * Credentials are named security privileges that allow organizing security by group.

Restricting access to an action is simply made by creating and editing a YAML configuration file called `security.yml` in the module `config/` directory. In this file, you can specify the security requirements that users must fulfill for each action or for `all` actions. Listing 6-20 shows a sample `security.yml`.

Listing 6-20 - Setting Access Restrictions, in `apps/frontend/modules/mymodule/config/security.yml`

    read:
      is_secure:   false       # All users can request the read action

    update:
      is_secure:   true        # The update action is only for authenticated users

    delete:
      is_secure:   true        # Only for authenticated users
      credentials: admin       # With the admin credential

    all:
      is_secure:  false        # false is the default value anyway

Actions are not secure by default, so when there is no `security.yml` or no mention of an action in it, actions are accessible by everyone. If there is a `security.yml`, symfony looks for the name of the requested action and, if it exists, checks the fulfillment of the security requirements. What happens when a user tries to access a restricted action depends on his credentials:

  * If the user is authenticated and has the proper credentials, the action is executed.
  * If the user is not identified, he will be redirected to the default login action.
  * If the user is identified but doesn't have the proper credentials, he will be redirected to the default secure action, shown in Figure 6-1.

The default login and secure pages are pretty simple, and you will probably want to customize them. You can configure which actions are to be called in case of insufficient privileges in the application `settings.yml` by changing the value of the properties shown in Listing 6-21.

Figure 6-1 - The default secure action page

![The default secure action page](http://www.symfony-project.org/images/book/1_4/F0601.jpg "The default secure action page")

Listing 6-21 - Default Security Actions Are Defined in `apps/frontend/config/settings.yml`

    all:
      .actions:
        login_module:  default
        login_action:  login

        secure_module: default
        secure_action: secure

### Granting Access

To get access to restricted actions, users need to be authenticated and/or to have certain credentials. You can extend a user's privileges by calling methods of the `sfUser` object. The authenticated status of the user is set by the `setAuthenticated()` method and can be checked with `isAuthenticated()`. Listing 6-22 shows a simple example of user authentication.

Listing 6-22 - Setting the Authenticated Status of a User

    [php]
    class myAccountActions extends sfActions
    {
      public function executeLogin($request)
      {
        if ($request->getParameter('login') === 'foobar')
        {
          $this->getUser()->setAuthenticated(true);
        }
      }

      public function executeLogout()
      {
        $this->getUser()->setAuthenticated(false);
      }
    }

Credentials are a bit more complex to deal with, since you can check, add, remove, and clear credentials. Listing 6-23 describes the credential methods of the `sfUser` class.

Listing 6-23 - Dealing with User Credentials in an Action

    [php]
    class myAccountActions extends sfActions
    {
      public function executeDoThingsWithCredentials()
      {
        $user = $this->getUser();

        // Add one or more credentials
        $user->addCredential('foo');
        $user->addCredentials('foo', 'bar');

        // Check if the user has a credential
        echo $user->hasCredential('foo');                      =>   true

        // Check if the user has both credentials
        echo $user->hasCredential(array('foo', 'bar'));        =>   true

        // Check if the user has one of the credentials
        echo $user->hasCredential(array('foo', 'bar'), false); =>   true

        // Remove a credential
        $user->removeCredential('foo');
        echo $user->hasCredential('foo');                      =>   false

        // Remove all credentials (useful in the logout process)
        $user->clearCredentials();
        echo $user->hasCredential('bar');                      =>   false
      }
    }

If a user has the `foo` credential, that user will be able to access the actions for which the `security.yml` requires that credential. Credentials can also be used to display only authorized content in a template, as shown in Listing 6-24.

Listing 6-24 - Dealing with User Credentials in a Template

    [php]
    <ul>
      <li><?php echo link_to('section1', 'content/section1') ?></li>
      <li><?php echo link_to('section2', 'content/section2') ?></li>
      <?php if ($sf_user->hasCredential('section3')): ?>
        <li><?php echo link_to('section3', 'content/section3') ?></li>
      <?php endif; ?>
    </ul>

As for the authenticated status, credentials are often given to users during the login process. This is why the `sfUser` object is often extended to add login and logout methods, in order to set the security status of users in a central place.

>**TIP**
>Among the symfony plug-ins, the [`sfGuardPlugin`](http://www.symfony-project.org/plugins/sfGuardPlugin) and [`sfDoctrineGuardPlugin`](http://www.symfony-project.org/plugins/sfDoctrineGuardPlugin) extend the session class to make login and logout easy. Refer to Chapter 17 for more information.

### Complex Credentials

The YAML syntax used in the `security.yml` file allows you to restrict access to users having a combination of credentials, using either AND-type or OR-type associations. With such a combination, you can build a complex workflow and user privilege management system--for instance, a content management system (CMS) back-office accessible only to users with the admin credential, where articles can be edited only by users with the `editor` credential and published only by the ones with the `publisher` credential. Listing 6-25 shows this example.

Listing 6-25 - Credentials Combination Syntax

    editArticle:
      credentials: [ admin, editor ]              # admin AND editor

    publishArticle:
      credentials: [ admin, publisher ]           # admin AND publisher

    userManagement:
      credentials: [[ admin, superuser ]]         # admin OR superuser

Each time you add a new level of square brackets, the logic swaps between AND and OR. So you can create very complex credential combinations, such as this:

    credentials: [[root, [supplier, [owner, quasiowner]], accounts]]
                 # root OR (supplier AND (owner OR quasiowner)) OR accounts

Filters
-------

The security process can be understood as a filter by which all requests must pass before executing the action. According to some tests executed in the filter, the processing of the request is modified--for instance, by changing the action executed (`default`/`secure` instead of the requested action in the case of the security filter). Symfony extends this idea to filter classes. You can specify any number of filter classes to be executed before the action execution or before the response rendering, and do this for every request. You can see filters as a way to package some code, similar to `preExecute()` and `postExecute()`, but at a higher level (for a whole application instead of for a whole module).

### The Filter Chain

Symfony actually sees the processing of a request as a chain of filters. When a request is received by the framework, the first filter (which is always the `sfRenderingFilter`) is executed. At some point, it calls the next filter in the chain, then the next, and so on. When the last filter (which is always `sfExecutionFilter`) is executed, the previous filter can finish, and so on back to the rendering filter. Figure 6-3 illustrates this idea with a sequence diagram, using an artificially small filter chain (the real one contains more filters).

Figure 6-3 - Sample filter chain

![Sample filter chain](http://www.symfony-project.org/images/book/1_4/F0603.png "Sample filter chain")

This process justifies the structure of the filter classes. They all extend the `sfFilter` class, and contain one `execute()` method, expecting a `$filterChain` object as parameter. Somewhere in this method, the filter passes to the next filter in the chain by calling `$filterChain->execute()`. See Listing 6-26 for an example. So basically, filters are divided into two parts:

  * The code before the call to `$filterChain->execute()` executes before the action execution.
  * The code after the call to `$filterChain->execute()` executes after the action execution and before the rendering.

Listing 6-26 - Filter Class Struture

    [php]
    class myFilter extends sfFilter
    {
      public function execute ($filterChain)
      {
        // Code to execute before the action execution
        ...

        // Execute next filter in the chain
        $filterChain->execute();

        // Code to execute after the action execution, before the rendering
        ...
      }
    }

The default filter chain is defined in an application configuration file called `filters.yml`, and is shown in Listing 6-27. This file lists the filters that are to be executed for every request.

Listing 6-27 - Default Filter Chain, in `frontend/config/filters.yml`

    rendering: ~
    security:  ~

    # Generally, you will want to insert your own filters here

    cache:     ~
    execution: ~

These declarations have no parameter (the tilde character, `~`, means "null" in YAML), because they inherit the parameters defined in the symfony core. In the core, symfony defines `class` and `param` settings for each of these filters. For instance, Listing 6-28 shows the default parameters for the `rendering` filter.

Listing 6-28 - Default Parameters of the rendering Filter, in `sfConfig::get('sf_symfony_lib_dir')/config/config/filters.yml`

    rendering:
      class: sfRenderingFilter   # Filter class
      param:                     # Filter parameters
        type: rendering

By leaving the empty value (`~`) in the application `filters.yml`, you tell symfony to apply the filter with the default settings defined in the core.

You can customize the filter chain in various ways:

  * Disable some filters from the chain by adding an `enabled: false` parameter. For instance, to disable the `security` filter, write:

        security:
          enabled: false

  * Do not remove an entry from the `filters.yml` to disable a filter; symfony would throw an exception in this case.
  * Add your own declarations somewhere in the chain (usually after the `security` filter) to add a custom filter (as discussed in the next section). Be aware that the `rendering` filter must be the first entry, and the `execution` filter must be the last entry of the filter chain.
  * Override the default class and parameters of the default filters (notably to modify the security system and use your own security filter).

### Building Your Own Filter

It is pretty simple to build a filter. Create a class definition similar to the one shown in Listing 6-26, and place it in one of the project's `lib/` folders to take advantage of the autoloading feature.

As an action can forward or redirect to another action and consequently relaunch the full chain of filters, you might want to restrict the execution of your own filters to the first action call of the request. The `isFirstCall()` method of the `sfFilter` class returns a Boolean for this purpose. This call only makes sense before the action execution.

These concepts are clearer with an example. Listing 6-29 shows a filter used to auto-log users with a specific `MyWebSite` cookie, which is supposedly created by the login action. It is a rudimentary but working way to implement the "remember me" feature offered in login forms.

Listing 6-29 - Sample Filter Class File, Saved in `apps/frontend/lib/rememberFilter.class.php`

    [php]
    class rememberFilter extends sfFilter
    {
      public function execute($filterChain)
      {
        // Execute this filter only once
        if ($this->isFirstCall())
        {
          // Filters don't have direct access to the request and user objects.
          // You will need to use the context object to get them
          $request = $this->getContext()->getRequest();
          $user    = $this->getContext()->getUser();

          if ($request->getCookie('MyWebSite'))
          {
            // sign in
            $user->setAuthenticated(true);
          }
        }

        // Execute next filter
        $filterChain->execute();
      }
    }

In some cases, instead of continuing the filter chain execution, you will need to forward to a specific action at the end of a filter. `sfFilter` doesn't have a `forward()` method, but `sfController` does, so you can simply do that by calling the following:

    [php]
    return $this->getContext()->getController()->forward('mymodule', 'myAction');

>**NOTE**
>The `sfFilter` class has an `initialize()` method, executed when the filter object is created. You can override it in your custom filter if you need to deal with filter parameters (defined in `filters.yml`, as described next) in your own way.

### Filter Activation and Parameters

Creating a filter file is not enough to activate it. You need to add your filter to the filter chain, and for that, you must declare the filter class in the `filters.yml`, located in the application or in the module `config/` directory, as shown in Listing 6-30.

Listing 6-30 - Sample Filter Activation File, Saved in `apps/frontend/config/filters.yml`

    rendering: ~
    security:  ~

    remember:                 # Filters need a unique name
      class: rememberFilter
      param:
        cookie_name: MyWebSite
        condition:   %APP_ENABLE_REMEMBER_ME%

    cache:     ~
    execution: ~

When activated, the filter is executed for each request. The filter configuration file can contain one or more parameter definitions under the `param` key. The filter class has the ability to get the value of these parameters with the `getParameter()` method. Listing 6-31 demonstrates how to get a filter parameter value.

Listing 6-31 - Getting the Parameter Value, in `apps/frontend/lib/rememberFilter.class.php`

    [php]
    class rememberFilter extends sfFilter
    {
      public function execute($filterChain)
      {
        // ...

        if ($request->getCookie($this->getParameter('cookie_name')))
        {
          // ...
        }

        // ...
      }
    }

The `condition` parameter is tested by the filter chain to see if the filter must be executed. So your filter declarations can rely on an application configuration, just like the one in Listing 6-30. The remember filter will be executed only if your application `app.yml` shows this:

    all:
      enable_remember_me: true

### Sample Filters

The filter feature is useful to repeat code for every action. For instance, if you use a distant analytics system, you probably need to put a code snippet calling a distant tracker script in every page. You could put this code in the global layout, but then it would be active for all of the application. Alternatively, you could place it in a filter, such as the one shown in Listing 6-32, and activate it on a per-module basis.

Listing 6-32 - Google Analytics Filter

    [php]
    class sfGoogleAnalyticsFilter extends sfFilter
    {
      public function execute($filterChain)
      {
        // Nothing to do before the action
        $filterChain->execute();

        // Decorate the response with the tracker code
        $googleCode = '
    <script src="http://www.google-analytics.com/urchin.js"  type="text/javascript">
    </script>
    <script type="text/javascript">
      _uacct="UA-'.$this->getParameter('google_id').'";urchinTracker();
    </script>';
        $response = $this->getContext()->getResponse();
        $response->setContent(str_ireplace('</body>', $googleCode.'</body>',$response->getContent()));
       }
    }

Be aware that this filter is not perfect, as it should not add the tracker on responses that are not HTML.

Another example would be a filter that switches the request to SSL if it is not already, to secure the communication, as shown in Listing 6-33.

Listing 6-33 - Secure Communication Filter

    [php]
    class sfSecureFilter extends sfFilter
    {
      public function execute($filterChain)
      {
        $context = $this->getContext();
        $request = $context->getRequest();

        if (!$request->isSecure())
        {
          $secure_url = str_replace('http', 'https', $request->getUri());

          return $context->getController()->redirect($secure_url);
          // We don't continue the filter chain
        }
        else
        {
          // The request is already secure, so we can continue
          $filterChain->execute();
        }
      }
    }

Filters are used extensively in plug-ins, as they allow you to extend the features of an application globally. Refer to Chapter 17 to learn more about plug-ins.

Module Configuration
--------------------

A few module behaviors rely on configuration. To modify them, you must create a `module.yml` file in the module's `config/` directory and define settings on a per-environment basis (or under the `all:` header for all environments). Listing 6-34 shows an example of a `module.yml` file for the `mymodule` module.

Listing 6-34 - Module Configuration, in `apps/frontend/modules/mymodule/config/module.yml`

    all:                  # For all environments
      enabled:            true
      is_internal:        false
      view_class:         sfPHP
      partial_view_class: sf

The `enabled` parameter allows you to disable all actions of a module. All actions are redirected to the `module_disabled_module`/`module_disabled_action` action (as defined in `settings.yml`).

The `is_internal` parameter allows you to restrict the execution of all actions of a module to internal calls. For example, this is useful for mail actions that you must be able to call from another action, to send an e-mail message, but not from the outside.

The `view_class` parameter defines the view class. It must inherit from `sfView`. Overriding this value allows you to use other view systems, with other templating engines, such as Smarty.

The `partial_view_class` parameter defines the view class used for partials of this module. It must inherit from `sfPartialView`.

Summary
-------

In symfony, the controller layer is split into two parts: the front controller, which is the unique entry point to the application for a given environment, and the actions, which contain the page logic. An action has the ability to determine how its view will be executed, by returning one of the `sfView` constants. Inside an action, you can manipulate the different elements of the context, including the request object (`sfRequest`) and the current user session object (`sfUser`).

Combining the power of the session object, the action object, and the security configuration, symfony provides a complete security system, with access restriction and credentials. And if the `preExecute()` and `postExecute()` methods are made for reusability of code inside a module, the filters authorize the same reusability for all the applications by making controller code executed for every request.
