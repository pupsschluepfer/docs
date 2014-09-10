How-to build a RESTful API with Symfony 2 on a LAMP stack
=========================================================

This tutorial will lead you, step-by-step, through the process of building a simple RESTful API using the [REST Edition for Symfony](https://github.com/gimler/symfony-rest-edition) by Gordon Franke.

No explanations, just instructions: a straightforward approach to get things running!

**estimated time**: 30 - 40 min

**goal**: getting a [CRUD](http://en.wikipedia.org/wiki/Create,_read,_update_and_delete)-like API, using [JSON](http://json.org/), for managing an example entity

- [Preconditions](#preconditions)
- [Getting started](#getting-started)
  - [Installation via Composer](#installation-via-composer)
  - [Configure filehandling](#configure-filehandling)
  - [First impression](#first-impression)
- [Cleanup Acme](#cleanup-acme)
- [Creating a bundle](#creating-a-bundle)
- [The Example entity](#the-example-entity)
  - [Persisting in database](#persisting-in-database)
  - [Fixtures](#fixtures)
- [Controller](#controller)
- [Routing](#routing)
- [Final](#final)
- [Next steps](#next-steps)

Preconditions
-------------

installed and running:

- Linux
- Webserver (doc root: /var/www/)
- MySQL (host: "localhost", user: "root", password: "root") 
- PHP (PHP 5 >= 5.3.0)

Getting started
---------------

### Installation via Composer

```sh
$ cd /tmp
$ curl -sS https://getcomposer.org/installer | php
$ php composer.phar create-project gimler/symfony-rest-edition --stability=dev /var/www/example

Do you want to remove the existing VCS (.git, .svn..) history? [Y,n]? y [ENTER]
```

### Configure filehandling

```sh
$ cd /var/www/example
$ chmod -R 777 app/cache/ app/logs/
```

Uncomment the following line
```php
// app/console
umask(0000);
```
and this one:
```php
// web/app_dev.php 
umask(0000);
```

### First impression

Switch to your browser and call `http://localhost/example/web/config.php`

This site should appear: 

![](https://github.com/MarioLiebel/docs/blob/master/Symfony2/pics/Config.jpg)

If there're (major) problems, fix these (in `php.ini` file) and **restart the webserver**.

Calling for the [NelmioApiDocBundle](https://github.com/nelmio/NelmioApiDocBundle) via `http://localhost/example/web/app_dev.php/api/doc/` results in this view: 

![](https://github.com/MarioLiebel/docs/blob/master/Symfony2/pics/NelmioAPIDocInitial.jpg)

Fiddle around and get a feeling for the API doc (especially for the Sandbox feature)! 

cleanup Acme
------------

1. remove (source) files

    ```
    rm note.json
    rm -rf src/Acme/
    ```

2. remove this line from Kernel

    ```php
    // app/AppKernel.php
    $bundles[] = new Acme\DemoBundle\AcmeDemoBundle();
    ```

3. remove the routing directive

    ```yml
    # app/config/routing_dev.yml
    _demo_note:
        resource: "@AcmeDemoBundle/Controller/NoteController.php"
        type:     rest
    ```

Calling `http://localhost/example/web/app_dev.php/api/doc/` again shows the empty API doc site, without any dynamic generated content.

(Good moment for an initial commit into a [RCS](http://en.wikipedia.org/wiki/Revision_Control_System).)

Creating a bundle
-----------------

```sh
app/console generate:bundle

Bundle namespace: Foo/ExampleRestBundle [ENTER]

Bundle name [FooExampleRestBundle]: [ENTER]

Target directory [/var/www/example/src]: [ENTER]

Configuration format (yml, xml, php, or annotation): yml [ENTER]

Do you want to generate the whole directory structure [no]? [ENTER]

Do you confirm generation [yes]? [ENTER]

Confirm automatic update of your Kernel [yes]? [ENTER]

Confirm automatic update of the Routing [yes]? [ENTER]
```

The Example entity
------------------

```sh
app/console doctrine:generate:entity

The Entity shortcut name: FooExampleRestBundle:Example [ENTER]

Configuration format (yml, xml, php, or annotation) [annotation]: [ENTER]

New field name (press <return> to stop adding fields): exampleString [ENTER]
    Field type [string]: [ENTER] 
    Field length [255]:  [ENTER]

New field name (press <return> to stop adding fields): exampleInteger [ENTER]
    Field type [string]: integer [ENTER]

New field name (press <return> to stop adding fields): exampleDateTime [ENTER]
    Field type [string]: datetime [ENTER]

New field name (press <return> to stop adding fields): [ENTER]

Do you want to generate an empty repository class [no]? [ENTER]
```

#### Persisting in database

1. configure the parameters for MySQL instance

    ```yml
    # app/config/parameters.yml
    parameters:
      database_driver:   pdo_mysql
      database_host:     127.0.0.1
      database_port:     ~
      database_name:     symfony
      database_user:     root
      database_password: root
    ```
2. create database

    ```sh
    app/console doctrine:database:create
    ```
    
3. create schema for Example entity

    ```sh
    app/console doctrine:schema:create
    ```

#### Fixtures

1. install fixtures bundle by adding dependency in `composer.json` (don't forget adding the comma in the line above)

    ```javascript
    // composer.json
    "require": {
            ...,
            "doctrine/doctrine-fixtures-bundle": "2.2.*"
    }
    ```
    
2. update via Composer

    ```sh
    php /tmp/composer.phar update doctrine/doctrine-fixtures-bundle
    ```

3. adding the fixtures bundle in AppKernel

    ```php
    // app/AppKernel.php
    $bundles = array(
        // ...
        new Doctrine\Bundle\FixturesBundle\DoctrineFixturesBundle(),
        // ...
    );
    ```
    
4. creating the directory structure for fixtures

    ```sh
    mkdir -p src/Foo/ExampleRestBundle/DataFixtures/ORM
    ```

5. writing an Example fixture

    ```php
    <?php
    // src/Foo/ExampleRestBundle/DataFixtures/ORM/LoadExampleData.php
    namespace Foo\ExampleRestBundle\DataFixtures\ORM;

    use Doctrine\Common\DataFixtures\FixtureInterface;
    use Doctrine\Common\Persistence\ObjectManager;
    use Foo\ExampleRestBundle\Entity\Example;

    /**
     * LoadExampleData 
     * 
     * @package ExampleRestBundle
     * @author Mario Liebel
     */
    class LoadExampleData implements FixtureInterface
    {
        /**
        * {@inheritDoc}
        */
        public function load(ObjectManager $manager)
        {
            $example = new Example();
            $example->setExampleString('string');
            $example->setExampleInteger(42);
            $example->setExampleDateTime(new \DateTime('2012-01-28T17:00:00+02:00'));

            $manager->persist($example);
            $manager->flush();
        }
    }
    ```
    
6. load the fixtures

    ```sh
    app/console doctrine:fixtures:load

    Careful, database will be purged. Do you want to continue Y/N ? y [ENTER]
    ```

###Controller

```php
<?php
// src/Foo/ExampleRestBundle/Controller/ExampleController.php
namespace Foo\ExampleRestBundle\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\Controller,
    Symfony\Component\HttpFoundation\Response,
    FOS\RestBundle\Controller\Annotations as Rest,
    Nelmio\ApiDocBundle\Annotation\ApiDoc;

/**
 * ExampleController 
 * 
 * @package ExampleRestBundle
 * @author Mario Liebel
 */
class ExampleController extends Controller
{
    /**
     * Get all Examples.
     *
     * @ApiDoc(
     *   resource = true,
     *   statusCodes = {
     *     200 = "OK",
     *     404 = "Not Found"
     *   }
     * )
     *
     * @Rest\View
     *
     * @throws NotFoundHttpException
     */
    public function indexAction() {
        $em = $this->getDoctrine()->getManager();

        $examples = $em->getRepository('FooExampleRestBundle:Example')
                       ->findAll();

        if (!$examples) {
            throw $this->createNotFoundException();
        }
        
        return $examples;
    }

    /**
     * Create Example by JSON.
     *
     * @ApiDoc(
     *   resource = true,
     *   statusCodes = {
     *     201 = "Created",
     *     400 = "Bad Request"
     *   }
     * )
     *
     * @Rest\View
     */
    public function createAction()
    {
        $serializer = $this->container->get('serializer');

        $example = $serializer->deserialize($this->get('request')->getContent(), 
                                            'Foo\ExampleRestBundle\Entity\Example', 
                                            'json');

        $em = $this->getDoctrine()->getManager();
        $em->persist($example);
        $em->flush();
    }

    /**
     * Get Example by id.
     *
     * @ApiDoc(
     *   resource = true,
     *   statusCodes = {
     *     200 = "OK",
     *     404 = "Not Found"
     *   }
     * )
     *
     * @Rest\View
     *
     * @throws NotFoundHttpException
     */
    public function retrieveAction($id)
    {
        $em = $this->getDoctrine()->getManager();

        $example = $em->getRepository('FooExampleRestBundle:Example')
                      ->find($id);

        if (!$example) {
            throw $this->createNotFoundException();
        }

        return $example;
    }

    /**
     * Update Example by id and JSON.
     *
     * @ApiDoc(
     *   resource = true,
     *   statusCodes = {
     *     204 = "No Content",
     *     404 = "Not Found"
     *   }
     * )
     *
     * @Rest\View
     *
     * @throws NotFoundHttpException
     */
    public function updateAction($id)
    {
        $em = $this->getDoctrine()->getManager();

        $example = $em->getRepository('FooExampleRestBundle:Example')
                      ->find($id);

        if (!$example) {
            return $this->createNotFoundException();
        }

        $serializer = $this->container->get('serializer');

        $example = $serializer->deserialize($this->get('request')->getContent(), 
                                            'Foo\ExampleRestBundle\Entity\Example', 
                                            'json');

        $em->merge($example);
        $em->flush();        

        return $example;
    }

    /**
     * Delete Example by id.
     *
     * @ApiDoc(
     *   resource = true,
     *   statusCodes = {
     *     200 = "OK",
     *     204 = "No Content",
     *   }
     * )
     *
     * @Rest\View
     */
    public function deleteAction($id)
    {
        $em = $this->getDoctrine()->getManager();

        $responseCode = 204;

        $example = $em->getRepository('FooExampleRestBundle:Example')
                      ->find($id);

        if ($example) {

            $em->remove($example);
            $em->flush();

            $responseCode = 200;
        }

        return new Response(null, $responseCode);
    }
}
```

## Routing

```yml
# src/Foo/ExampleRestBundle/Resources/config/routing.yml

foo_example_examples:
    pattern: /examples
    defaults: { _controller: FooExampleRestBundle:Example:index }
    methods: [GET]

foo_example_example_create:
    pattern: /example
    defaults: { _controller: FooExampleRestBundle:Example:create }
    methods: [POST]

foo_example_example_retrieve:
    pattern: /example/{id}
    defaults: { _controller: FooExampleRestBundle:Example:retrieve }
    methods: [GET]

foo_example_example_update:
    pattern: /example/{id}
    defaults: { _controller: FooExampleRestBundle:Example:update }
    methods: [PUT]

foo_example_example_delete:
    pattern: /example/{id}
    defaults: { _controller: FooExampleRestBundle:Example:delete }
    methods: [DELETE]
```

Final
-----

So, that's it: you're done. 

Refresh your API doc site; resulting in:
![](https://github.com/MarioLiebel/docs/blob/master/Symfony2/pics/NelmioAPIDocFinal.jpg)

Important while playing around in the Sandbox:
- set your Headers correct: `Accept: application/json`
- set your Content-Type correct for PUT/POST: `Content-Type: application/json`

Next steps
----------

- include [HATEOAS](https://github.com/willdurand/Hateoas)
- use [Collections+JSON](http://amundsen.com/media-types/collection/format/) for indexing actions
- learn more about [JMS](https://github.com/schmittjoh/JMSSerializerBundle)
- extend the [API documentation](https://github.com/nelmio/NelmioApiDocBundle) with more informations
- writing (functional) tests for the API using the [LiipFunctionalTestBundle](https://github.com/liip/LiipFunctionalTestBundle)
- dynamic [Content Negotiation](http://en.wikipedia.org/wiki/Content_negotiation) (eg. for XML handling)
- Exception handling, Refactoring etc. 
