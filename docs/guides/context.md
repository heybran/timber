---
title: "Context"
menu:
  main:
    parent: "guides"
---

The context is one of the most important concepts to understand in Timber. Think of the context as the set of variables that are passed to your Twig template.

In the following example, `$data` is an associative array of values. Each of the values will be available in the Twig template with the key as a variable name.

**single.php**

```php
<?php

$data = array(
    'message' => 'This can be any variable you want',
    'author'  => 'Tom',
);

Timber::render( 'single.twig', $data );
```

**single.twig**

```twig
<h3>Message by {{ author }}</h3>
<p>{{ message }}</p>
```

Of course you don’t have to figure out all the variables you need for yourself. Timber will provide you with a set of useful variables when you call `Timber::context()`. 

**single.php**

```php
<?php

$context = Timber::context();

Timber::render( 'single.twig', $context );
```

Follow this guide to get an overview of what’s in the context, or use `var_dump( $context );` in PHP or `{{ dump() }}` in Twig to display the contents in your browser.

## Global context

The global context is the context that is always set when you load it through `Timber::context()`.

### Global context variables

Among others, the following variables will be available:

- **site** – The `site` variable is a [Timber\Site](/docs/reference/timber-site/) object which will make it easier for you to retrieve site info. If you’re used to using `blog_info( 'sitename' )` in PHP, you can use `{{ site.name }}` in Twig instead.
- **request** - The `request` variable is a `Timber\Request` object, which will make it easier for you to access `$_GET` and `$_POST` variables in your context. Please be aware that you should always be very careful about using `$_GET` and `$_POST` variables in your templates directly. Read more about this in the Escaping Guide.
- **theme** - The `theme` variable is [Timber\Theme](/docs/reference/timber-theme/) object and contains info about your theme.
- **user** - The `user` variable will be a [`Timber\User`](/docs/reference/timber-user/) object if a user/visitor is currently logged in and otherwise it will just be `false`.

For a full list of variables, go have a look at the reference for [`Timber::context()`](/docs/reference/timber-timber/#context).

### Hook into the global context

In your theme, you probably have elements that you use on every page, like a navigation menu, or a postal address or phone number. You don’t want to add these variables to the context every time you call `Timber::render()` or `Timber::compile()`. And you don’t have to! You can use the `timber/context` filter to add your own data that will always be available.

Here’s an example for how you could **add a navigation menu** to your context, so that it becomes available in every template you use:

**functions.php**

```php
// Example: Add a menu to the global context.
add_filter( 'timber/context', function( $context ) {
    $context['menu'] = new Timber\Menu( 'primary-menu' );

    return $context;
} );
```

For menus to work, you will first need to [register them](https://codex.wordpress.org/Navigation_Menus).

## Context cache

The global context will be cached. That’s why you need to define your `timber/context` filter before using `Timber::context()` for the first time. Otherwise, the cache will be set before you could add your own data. Having a cached global context can be useful if you need the context in other places. For example if you compile the template for a shortcode:

```php
/**
 * Shortcode for address inside a WYSIWG field
 */
add_shortcode( 'company_address', function() {
    return Timber::compile(
        'shortcode/company-address.twig',
        Timber::context_global()
    );
} );
```

Inside `shortcode/company-address.twig`, you will have access to all the global variables that you can use in your normal template files as well. Whenever you only need the global context, you can use the `Timber::context_global()` function. You can call that function multiple times without losing performance.

## Template contexts

The context can change based on what template is displayed. Timber will always set `post` or `posts` in the context, if it can. But sometimes this is not what you want. Sometimes you need to change some arguments when fetching posts, or write your own posts query. In that case, it’s important to tell Timber about your change. Otherwise it will perform a separate query in the back, which might affect the performance of your page load. Since version 2.0 of Timber, you have more control over this process.

Timber won’t cache template contexts.

### Singular templates

The `post` variable will be available in singular templates ([is_singular()](https://developer.wordpress.org/reference/functions/is_singular/)), like posts or pages. It will contain a `Timber\Post` object of the currently displayed post.

This means that the most basic singular template might look like this:

**single.php**

```php
<?php

Timber::render( 'single.twig', Timber::context() );
```

If you want to use a different post than the default post, you can do that with the `post` parameter. In the following example, we’re passing a post ID directly.

**single.php**

```php
<?php

$context = Timber::context( array(
    'post' => 85,
) );

Timber::render( 'single.twig', $context );
```

Of course you can also pass in a `Timber\Post` object instead of just an ID.

**single.php**

```php
<?php
$context = Timber::context( array(
    'post' => new Timber\Post( 85 ),
) );

Timber::render( 'single.twig', $context );
```

This is also the way to go if you want to use [your own post class](/docs/guides/extending-timber/). In that case, you could pass a post object directly.

**single.php**

```php
<?php
$context = Timber::context( array(
    'post' => new Extended_Post(),
) );

Timber::render( 'single.twig', $context );
```

### Archive templates

The `posts` variable will be available in archive templates ([is_archive()](https://developer.wordpress.org/reference/functions/is_archive/)), like your posts index page, category or tag archives, date based or author archives. It will contain the posts that are fetched with `Timber\PostQuery` with the arguments that WordPress uses for the default query of the currently displayed page.

#### Change arguments for default query

You can change arguments for the default query that WordPress will use to fetch posts by passing a `posts` argument array to the `context()` function. For example, if you’d want to change the default query to *only show pages that have no parents*, you could pass in a `post_parent` argument:

**archive.php**

```php
$context = Timber::context( array(
    'posts' => array(
        'post_parent' => 0,
    ),
) );

Timber::render( 'archive.twig', $context );
```

Your `$context` will then contain a `posts` entry with a posts collection fetched by the default query. This is practically the same as using [`pre_get_posts` filter](https://developer.wordpress.org/reference/hooks/pre_get_posts/) in a default WordPress project, but it might be a little more convenient.

#### Overwrite default query

You may not want to use the default query that WordPress uses. Sometimes you want to write your own query from scratch. To **overwrite the default query** that WordPress uses, there’s a parameter `cancel_default_query` that you can use in combination with the arguments array that you pass to `context()`:

**archive.php**

```php
// Cancel default query for a small performance improvement
$context = Timber::context( array(
    'cancel_default_query' => true,
) );

// Write your own query
$context['posts'] = new Timber\PostQuery( array(
    'post_type'      => 'book',
    'posts_per_page' => -1,
    'post_status'    => 'publish',
) );

Timber::render( 'archive.twig', $context );
```

For a more compact way of writing this, you can also pass in the arguments directly through the `posts` parameter. If `cancel_default_query` is set to `true`, the arguments that you pass in `posts` will be used to perform a query that will then be present in `$context['posts']`.

**archive.php**

```php
Timber::render( 'archive.twig', Timber::context( array(
    'cancel_default_query' => true,
    'posts'                => array(
        'post_type'      => 'book',
        'posts_per_page' => -1,
        'post_status'    => 'publish',
    ),
) ) );
```

### Disable template contexts

Automatic template contexts might not be what you want. In that case, you can disable them.

#### Disable archive context

You can also directly pass `false` to the `posts` argument when you call `context()`:

**archive.php**

```php
$context = Timber::context( array(
    'posts' => false,
) );
```

#### Disable singular context

You can directly pass `false` to the `post` argument when you call `context()`:

**single.php**

```php
$context = Timber::context( array(
    'post'  => false,
) );
```

If you disable `post` in the context, you should pass your new post through `Timber::context_post()`.

**single.php**

```php
<?php
// If you need to set post yourself, use `context_post()` to improve compatibility.

$context         = Timber::context( array( 'post' => false ) );
$context['post'] = Timber::context_post( new Extended_Post() );

Timber::render( 'single.twig', $context );
```

The `Timber::context_post()` function is needed **for better compatibility with third party plugins**. Through this function, Timber will set `$wp->query->in_the_loop` to `true`, will call the `loop_start` action and will call `$wp_query->setup_postdata()`, which in turn will call the `the_post` action.

### Disable template context with a filter

If you want to globally disable template contexts, you can use the `timber/context/args` filter:

**functions.php**

```php
// Disable `post` and/or `posts` in context
add_filter( 'timber/context/args', function( $args ) {
    $args['post'] = false;
    $args['posts'] = false;

    return $args;
} );
```