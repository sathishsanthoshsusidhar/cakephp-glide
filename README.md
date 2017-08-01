# CakePHP Glide

[![Build Status](https://img.shields.io/travis/ADmad/cakephp-glide/master.svg?style=flat-square)](https://travis-ci.org/ADmad/cakephp-glide)
[![Coverage Status](https://img.shields.io/codecov/c/github/ADmad/cakephp-glide.svg?style=flat-square)](https://codecov.io/github/ADmad/cakephp-glide)
[![Total Downloads](https://img.shields.io/packagist/dt/ADmad/cakephp-glide.svg?style=flat-square)](https://packagist.org/packages/ADmad/cakephp-glide)
[![License](https://img.shields.io/badge/license-MIT-blue.svg?style=flat-square)](LICENSE.txt)

CakePHP 3.x plugin to help using [Glide](http://glide.thephpleague.com/) image manipulation library.

The plugin consists of a middlware, view helper and a Glide response class.

## Requirements

* CakePHP 3.5+ (For 3.4 and below use 2.x, for 3.2 and below use 1.x)

## Installation

Install the plugin through composer:

```
composer require admad/cakephp-glide
```

Load the plugin in `config/bootstrap.php`:

```
Plugin::load('ADmad/Glide');
```

## Usage

### Middleware

In your `config/routes.php` setup the `GlideMiddleware` for required scope which
intercepts requests and serves images generated by Glide. For e.g.:

```php
Router::scope('/images', function ($routes) {
    $routes->registerMiddleware('glide', new \ADmad\Glide\Middleware\GlideMiddleware([
        // Either an instance of League\Glide\Server or config array to be used to
        // create server instance.
        // http://glide.thephpleague.com/1.0/config/setup/
        'server' => [
            // Path or League\Flysystem adapter instance to read images from.
            // http://glide.thephpleague.com/1.0/config/source-and-cache/
            'source' => WWW_ROOT . 'uploads',

            // Path or League\Flysystem adapter instance to write cached images to.
            'cache' => WWW_ROOT . 'cache',

            // URL part to be omitted from source path. Defaults to "/images/"
            // http://glide.thephpleague.com/1.0/config/source-and-cache/#set-a-base-url
            'base_url' => '/images/',

            // Response class for serving images. If unset (default) an instance of
            // \ADmad\Glide\Responses\PsrResponseFactory() will be used.
            // http://glide.thephpleague.com/1.0/config/responses/
            'response' => null,
        ],

        // http://glide.thephpleague.com/1.0/config/security/
        'security' => [
            // Boolean indicating whether secure URLs should be used to prevent URL
            // parameter manipulation. Default false.
            'secureUrls' => false,

            // Signing key used to generate / validate URLs if `secureUrls` is `true`.
            // If unset value of Cake\Utility\Security::salt() will be used.
            'signKey' => null,
        ],

        // Cache duration. Default '+1 days'.
        'cacheTime' => '+1 days',

        // Any response headers you may want to set. Default null.
        'headers' => [
            'X-Custom' => 'some-value',
        ]
    ]));

    $routes->applyMiddleware('glide');

    $routes->connect('/*');
});
```

For the example config shown above, for URL like `domain.com/images/user/profile.jpg`
the source image should be under `webroot/uploads/user/profile.jpg`.

__Note__: Make sure the image URL does not directly map to image under webroot.
Otherwise as per CakePHP's default URL rewriting rules the image will be served by
webserver itself and request won't reach your CakePHP app.

#### Exceptions

If you have enabled secure URLs and a valid token is not present in query string
the middleware will throw a `Cake\Network\Exception\BadReqestException` exception.

If a response image could not be generated by the Glide library then by default
the middleware will throw a `Cake\Network\Exception\NotFoundException`.
A `Glide.failure` event will also be triggered in this case. You can setup a
listener for this event and return a response instance from it. In this case
that response will be sent to client and `NotFoundException` will not be thrown.

```php
\Cake\Event\EventManager::instance()->on(
    \ADmad\Glide\Middleware\GlideMiddleware::FAILURE_EVENT,
    function ($event) {
        return (new Response())
            ->withFile('/path/to/default-image.jpg');
    }
);
```

### Helper

The provided `GlideHelper` helps creating URLs and image tags for generating
images. You can load the helper in your `AppView::initialize()` as shown in
example below:

```php
public function initialize()
{
    // All option values should match the corresponding options for `GlideFilter`.
    $this->loadHelper('ADmad/Glide.Glide', [
        // Base URL.
        'baseUrl' => '/images/',
        // Whether to generate secure URLs.
        'secureUrls' => false,
        // Signing key to use when generating secure URLs.
        'signKey' => null,
    ]);
}

```

Here are the available methods of `GlideHelper`:

```php
    /**
     * Creates a formatted IMG element.
     *
     * @param string $path Image path.
     * @param array $params Image manipulation parameters.
     * @param array $options Array of HTML attributes and options.
     *   See `$options` argument of `Cake\View\HtmlHelper::image()`.
     * @return string Complete <img> tag.
     */
    GlideHelper::image($path, array $params = [], array $options = [])

    /**
     * URL with query string based on resizing params.
     *
     * @param string $path Image path.
     * @param array $params Image manipulation parameters.
     * @return string Image URL.
     */
    GlideHelper::url($path, array $params = [])
```

The main benefit of using this helper is depending on the value of `secureUrls`
config, the generated URLs will contain a token which will be verified by the
dispatch filter. The prevents modification of query string params.
