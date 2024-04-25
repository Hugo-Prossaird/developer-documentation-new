Plugin upgrade from 4 to 5 step by step
=======================================

Here is a list of steps that most of the Plugins may need to take to upgrade from Mautic 4 to Mautic 5. You should be able to get through each step, make a commit, move to the next one and once you are at finished you have upgraded your Plugin.

1. Continuous Integration
-------------------------

If you don't have CI configured, this is the time to do it. This is an optional step but it makes sense to do it at the beginning rather than later. Here's how to get it done: https://devdocs.mautic.org/en/5.x/plugins/continuous-integration.html

In your PR add also PHP 8.1 and 8.2. And upgrade the Mautic version from 4.4 to 5.1. One more thing is that Mautic 5 have ``local.php`` in ``config/local.php`` instead of ``app/config/local.php`` so update that as well.

2. Autowiring
-------------

Mautic 5 comes with autowiring of PHP services which means the developer experience is way better and the code size gets reduced.

There is a great doc already written on this topic so get that setup and come back: https://devdocs.mautic.org/en/5.x/plugins/autowiring.html

To quickly check that all services are wired and configured correctly you may find this command faster than refreshing the browser window:

``rm -rf var/cache && bin/console``

.. note:: Ideally, you should be able to delete the whole ``services`` section from your ``config.php`` file. But do that as a cherry on top once you are sure everything is working. See step 5 that could get you stuck good. There may be others like it.

3. config.php - controllers
---------------------------

config.php should be way lighter now when all services are gone after autowiring is configured. There is one more thing to check. The controllers are now defined with a different syntax. Here is an example:

.. code:: diff

   - 'controller' => 'CronfigBundle:Cronfig:index',
   + 'controller' => 'MauticPlugin\CronfigBundle\Controller\CronfigController::indexAction'

Symfony 5 is way more explicit. That's a good thing even if it's longer! You don't have to guess what the syntax is. It's basically just standard FQCN (Fully Qualified Class Name) with the full method name behind the 2 colons. You don't even need to call the controller method `*Action` anymore. Your choice.

4. Rendering Views
------------------

As the PHP templating engine was removed from Symfony 5, Mautic had to switch to Twig. And your Plugin must follow. Here is a helpful resource on how to migrate the `*.html.php` files to `*.html.twig` files:

https://github.com/mautic/mautic/blob/5.x/UPGRADE-PHP-TO-TWIG-TEMPLATES.md

In the controllers, you'll also have to update the view paths like this:

.. code:: diff

   - $this->renderView('MauticCoreBundle:Notification:flash_messages.html.php');
   + $this->renderView('@MauticCore/Notification/flash_messages.html.twig');

Running this command is faster than refreshing all the views in the browser. It validates the Twig syntax and can guide you through the process:

``bin/console lint:twig plugins/MyBundle``

.. note:: Update MyBundle with your bundle name.

5. The Integration Class
------------------------

If you went ahead and deleted all services from ``config.php`` with a smile on your face, you may find yourself in a pickle if you are using Mautic's Integration classes and interfaces. The inner workings of the IntegrationsBundle expects that your integration has a service key in a specific format. This should be improved for Mautic 6, but for now add an alias to ``services.php``:

.. code:: php
   $services->alias('mautic.integration.[MY_INTEGRAION]', \MauticPlugin\[MY_INTEGRAION]Bundle\Integration\[MY_INTEGRAION]Integration::class);

.. note:: Replace `[MY_INTEGRAION]` with your Plugin name.

6. Compiler Passes
------------------

If you Plugin uses a compiler pass, you may have to double-check that it works correctly. In many cases you may have to change the service alias with FQCN like so:

.. code:: diff

   - ->setDecoratedService('mautic.form.type.email', 'mautic.form.type.email.inner');
   + ->setDecoratedService(EmailType::class, 'mautic.form.type.email.inner')

7. Getting container in tests
-----------------------------

This one is a quick find and replace:

.. code:: diff

   - $handlerStack = self::$container->get('mautic.http.client.mock_handler');
   + $handlerStack = static::getContainer()->get(MockHandler::class);

Notice you can also use FQCN instead of string service keys which is more convenient.

8. Automated Refactoring
------------------------

Your Plugin should be working on Mautic 5 by now. Wouldn't it be great to shorten the code a little more? Mautic 5 uses PHP 8.0+ so can take advantage of the syntax. And Rector can upgrade the code for you.

Run ``bin/rector process plugins/MyBundle`` and review the changes.

.. note:: Update MyBundle with your bundle name.

9. Automated Code Style
-----------------------

Another great way how to improve your Plugin code base quality is to run the CS Fixer: `bin/php-cs-fixer fix plugins/MyBundle`.

.. note:: Update MyBundle with your bundle name.

10. Static Analysis
-------------------

PHPSTAN is another amazing tool that detects bugs for you. It's better to run it on the whole codebase including core Mautic so it's aware of all classes.

Run ``composer phpstan``

If your plugin has way too many PHPSTAN errors than you can handle right now, consider using [PHPSTAN baseline](https://phpstan.org/user-guide/baseline). It allows you to store your tech debt to a single file and it forces you to write better code from now on. And you can reduce the baseline by small chunks every month to get to 0.

Conclusion
----------

This list of steps is compiled by Mautic Plugin developers for the Mautic Plugin developers. If you find that some common problem isn't addressed here, please add it.
