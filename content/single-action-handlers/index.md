---
title: PSR Standards - Single Action Handlers in PHP Frameworks
description: Single Action Handlers in PHP Frameworks
slug: psr-standards-single-action-handlers
date: 2025-04-14 00:00:00+0000
image: cover.jpg
categories:
    - PHP
tags:
    - PHP
    - PSR
    - Laravel
    - Symfony
    - Mezzio
---

## Introduction: What is the Single Action Handler Pattern?

In modern PHP web development, the **Single Action Handler** (often implemented as an **Invokable Controller** or specifically as a **PSR-15 Request Handler**) represents a shift towards more focused, decoupled, and testable code, especially for APIs and specific web actions.

This pattern moves away from traditional Model-View-Controller (MVC) structures where a single Controller class might handle numerous related routes via multiple methods (e.g., `UserController` with `index()`, `show()`, `store()`, etc.).

Instead, a **Single Action Handler is a class dedicated exclusively to processing the logic for one specific route or endpoint.** It achieves this typically through:

1.  Implementing PHP's magic `__invoke()` method, allowing the class instance to be treated as the action itself.
2.  Implementing the `Psr\Http\Server\RequestHandlerInterface`, which defines a `handle(ServerRequestInterface $request): ResponseInterface` method – the standard defined by PSR-15.

This README explores this pattern, its benefits, the crucial role of PSR standards, and how it can be implemented effectively (and with varying degrees of native support) in popular PHP frameworks like Mezzio, Symfony, and Laravel.

**Example Contrast:**

* **Traditional Multi-Action Controller:**
    ```php
    // UserController.php
    class UserController { /* Handles multiple user-related routes */ }
    // routes.php: Route::get('/users/{id}', [UserController::class, 'show']);
    ```

* **Single Action Handler Pattern (Conceptual):**
    ```php
    // ShowUserHandler.php (PSR-15) or ShowUserAction.php (__invoke)
    class ShowUserHandler implements RequestHandlerInterface { /* Logic only for showing a user */ }
    // routes.php: Route::get('/users/{id}', ShowUserHandler::class); // Route directly to the handler
    ```

## Why Use This Pattern? The Rationale

Adopting the single action handler pattern brings significant advantages:

1.  **Single Responsibility Principle (SRP):** Each class does exactly one thing. This drastically improves clarity, making code easier to understand, modify safely, and debug.
2.  **Enhanced Testability:** Unit testing becomes much simpler. Smaller classes have fewer dependencies to mock, and the scope of each test is clearly defined.
3.  **Improved Organization:** Prevents "fat controllers." Code can be neatly organized by feature or domain slice, leading to a more maintainable structure, especially in large applications.
4.  **Increased Readability:** The class name itself often describes the action (e.g., `ProcessPaymentHandler`, `GetUserApiEndpoint`). The single `handle()` or `__invoke()` method contains all relevant logic.
5.  **Precise Dependency Management:** Only dependencies needed for that *specific action* are injected, leading to cleaner constructors and more efficient resource usage.
6.  **Reduced Cognitive Load:** Developers can focus entirely on the task of a single endpoint without the mental overhead of unrelated actions in the same file.

## The Importance of PSR Standards

Understanding PSR (PHP Standard Recommendations) is crucial for appreciating the full benefits of modern PHP development and patterns like PSR-15 Request Handlers. PSRs are specifications published by the PHP Framework Interop Group (PHP-FIG), comprised of members from various major PHP projects. Their goal is to promote **interoperability** and **standardization** across the PHP ecosystem.

**Key PSRs for Web Development:**

* **PSR-7 (HTTP Message Interfaces):** Defines standard interfaces for HTTP request (`RequestInterface`, `ServerRequestInterface`) and response (`ResponseInterface`) objects, along with related objects like URIs and streams. A key feature is **immutability**, which prevents unexpected side effects when messages are passed through multiple layers (like middleware).
* **PSR-15 (HTTP Server Request Handlers & Middleware):** Builds upon PSR-7.
    * `RequestHandlerInterface`: Defines a standard way to process a PSR-7 request and return a PSR-7 response (our Single Action Handler!).
    * `MiddlewareInterface`: Defines a standard interface for middleware components that process requests *before* or *after* a handler (or other middleware).

**Why Adhering to PSR Standards (like PSR-7/15) is a Good Idea:**

1.  **Interoperability:** This is the primary driver. Code written against PSR interfaces (handlers, middleware, HTTP clients, factories) can often be used across different frameworks and libraries that also adhere to those standards. You can mix and match components from different vendors.
2.  **Reduced Vendor Lock-In:** By relying on community standards rather than framework-specific abstractions for core functionalities like HTTP handling, your application becomes less tied to a single framework's ecosystem, making future migrations or integrations potentially easier.
3.  **Reusability:** Logic encapsulated in PSR-15 handlers or middleware can be more easily reused across different projects, even if those projects use different underlying frameworks (provided they support PSR-15).
4.  **Consistency & Predictability:** Standard interfaces mean developers encounter familiar patterns across different projects and libraries. This reduces the learning curve and makes codebases easier to understand and contribute to.
5.  **Modern Best Practices:** PSR-7's immutability and PSR-15's clear definition of middleware and handlers encourage decoupled, layered application design, which is widely considered a best practice for building robust web applications.
6.  **Future-Proofing:** Basing core application logic on community-agreed standards makes it less vulnerable to breaking changes within a specific framework's internal abstractions.
7.  **Easier Package Development:** If you're creating reusable packages (e.g., authentication middleware, API validation logic), targeting PSR interfaces makes them instantly usable in a much wider range of applications.

**Relevance to Handlers:** Using PSR-15 `RequestHandlerInterface` directly ties your core request-handling logic to these community standards, unlocking the benefits of interoperability, reusability, and consistency. Frameworks that embrace these standards natively often provide a smoother path to achieving these advantages.

## Implementation Across Frameworks

Frameworks vary in their native support for PSR-15 handlers.

* **PSR-15 Native Frameworks (e.g., Mezzio):** Use PSR-15 handlers as the fundamental way to process requests.
* **Full-Stack Frameworks (e.g., Symfony, Laravel):** Primarily use their own abstractions but provide mechanisms (bridges, extension points) to integrate PSR-15 handlers, requiring varying levels of effort.

### 1. Middleware Frameworks (PSR-15 Native - Mezzio Example)

Frameworks like Mezzio (formerly Zend Expressive) are built *from the ground up* around PSR-7 and PSR-15. Using single-action `RequestHandlerInterface` implementations is the standard, idiomatic way.

* **Core Concept:** Requests flow through a PSR-15 middleware pipeline, ending at a route-specific `RequestHandlerInterface`.
* **Example Implementation:** (See previous README example - it's the native approach)
* **Pros (Mezzio/PSR-15 Native):**
    * **Pure PSR Adherence:** Natively uses PSR-7/15, maximizing interoperability and benefits of the standards. No bridging needed for core HTTP handling.
    * **Minimalism & Performance:** Very lean core, potentially high performance.
    * **Maximum Flexibility & Decoupling:** Full control over components; promotes decoupled design.
* **Cons (Mezzio/PSR-15 Native):**
    * **More Initial Setup:** Requires assembling the application stack (router, container, ORM, etc.).
    * **Smaller Framework-Specific Ecosystem:** Fewer Mezzio-specific bundles compared to Symfony/Laravel (though any standard PHP/PSR package works).

### 2. Symfony: Achieving PSR-15 Compliance

Symfony is highly flexible and *can* work cleanly with PSR-15 handlers, though its core uses `HttpFoundation`.

* **Option 1: Adapter Pattern (The Basic Approach)**
    * **Concept:** Create a standard Symfony controller (`__invoke`) that acts as a bridge. It receives the `HttpFoundation\Request`, converts it to PSR-7 `ServerRequestInterface` (using `symfony/psr-http-message-bridge`), calls your actual `RequestHandlerInterface`, converts the PSR-7 `ResponseInterface` back to `HttpFoundation\Response`, and returns it.
    * **Pros:** Explicit, relatively easy to understand for a single handler.
    * **Cons:** **Significant boilerplate** - requires one adapter class per PSR-15 handler. Feels cumbersome.

* **Option 2: Centralized Listener (The Cleaner Approach)**
    * **Concept:** Leverage Symfony's Kernel Events. Create an Event Listener for the `kernel.controller` event. This listener checks if the controller resolved by the router implements `RequestHandlerInterface`. If it does, the listener takes over: it uses the PSR-7 bridge to convert the request, executes the handler's `handle` method, converts the response back, and sets it directly on the event (`$event->setResponse()`), bypassing Symfony's standard controller execution.
    * **Detailed Steps:**
        1.  Install bridge: `composer require symfony/psr-http-message-bridge nyholm/psr7` (Nyholm is a popular PSR-7 implementation).
        2.  Create your PSR-15 handler class (implementing `RequestHandlerInterface`). Ensure it's registered as a service.
        3.  Create an Event Listener class implementing `EventSubscriberInterface` or listening to `KernelEvents::CONTROLLER`.
        4.  Inject `HttpMessageFactoryInterface` and `HttpFoundationFactoryInterface` (from the bridge) into the listener.
        5.  In the listener method:
            * Get controller from `$event->getController()`. Check if it implements `RequestHandlerInterface`.
            * If yes: Convert `$event->getRequest()` -> PSR-7 Request.
            * Call `$controller->handle($psr7Request)`.
            * Convert PSR-7 Response -> Symfony Response.
            * Call `$event->setResponse($symfonyResponse)`.
        6.  Register the listener with appropriate priority.
        7.  Route directly to the service ID or FQCN of your PSR-15 handler class.
    * **Pros:** **Eliminates adapter boilerplate.** Centralizes bridging logic. Allows clean routing directly to PSR-15 handlers. Promotes PSR standard usage cleanly within Symfony.
    * **Cons:** Requires deeper understanding of Symfony's Kernel Events. The listener becomes a critical piece of infrastructure that needs careful testing.

* **Overall Symfony & PSR:** Symfony offers excellent tools and flexibility (`psr-http-message-bridge`, Kernel Events) to achieve clean PSR-15 integration via the listener approach. While not PSR-native at its core HTTP layer, its robust component system and extensibility make it a strong choice for developers wanting a full-stack framework that respects and facilitates PSR standard adherence.

### 3. Laravel: Achieving PSR-15 Compliance

Laravel prioritizes developer experience and convention. While it uses HttpFoundation internally and doesn't natively execute PSR-15 handlers, integration is possible.

* **Option 1: Adapter Pattern (The Basic Approach)**
    * **Concept:** Similar to Symfony's adapter - create an invokable Laravel controller (`__invoke`) that uses the PSR-7 bridge (included via dependencies) to convert the `Illuminate\Http\Request`, call the PSR-15 handler, and convert the PSR-7 response back to a Laravel-compatible response. Requires PSR-7 implementation like Nyholm and manual factory setup.
    * **Pros:** Explicit.
    * **Cons:** **Significant boilerplate** per handler. Feels unnatural in the Laravel ecosystem.

* **Option 2: Centralized Middleware (The Cleaner Approach)**
    * **Concept:** Create a custom Laravel Middleware. This middleware checks if the controller class resolved by the router for the current route implements `RequestHandlerInterface`. If it does, the middleware takes over: it instantiates the handler (via container), performs the Request/Response bridging using PSR-7 factories, executes the handler's `handle` method, and returns the converted response directly, bypassing Laravel's standard controller dispatch (`$next($request)` is skipped).
    * **Detailed Steps:**
        1.  Install PSR-7 implementation: `composer require nyholm/psr7`. Ensure bridge is available (usually is).
        2.  Create your PSR-15 handler class.
        3.  Create a Middleware class (e.g., `HandlePsr15Requests`).
        4.  Inject or create PSR-7 bridge factories within the middleware.
        5.  In the `handle($request, Closure $next)` method:
            * Inspect `$request->route()->getController()` or `$request->route()->getActionName()` to find the target handler class.
            * Use Reflection or the container to check if it implements `RequestHandlerInterface`.
            * If yes: Instantiate handler via `app()->make()`. Convert `Illuminate\Http\Request` -> PSR-7 Request. Call `handle()`. Convert PSR-7 Response -> Laravel Response. `return $laravelResponse;`
            * If no: `return $next($request);`
        6.  Register the middleware globally or in a specific route group (e.g., `api`) in `app/Http/Kernel.php`.
        7.  Route directly to the FQCN of your PSR-15 handler class.
    * **Pros:** **Eliminates adapter boilerplate.** Centralizes bridging logic. Allows routing directly to PSR-15 handlers.
    * **Cons:** Middleware becomes complex and critical. Requires careful handling of route information and potential pipeline order issues. Setting up PSR-7 factories might need a Service Provider. Works slightly "against the grain" of Laravel's typical controller flow.

* **Overall Laravel & PSR:** Laravel *can* be made to work with PSR-15 handlers via middleware, significantly cleaning up the adapter approach. However, compared to Symfony's kernel events or Mezzio's native support, it feels less integrated. Laravel's strength lies in its opinionated, rapid development workflow using its native components (Eloquent, Blade, Facades). Pushing for strict PSR-15 handler compliance requires bypassing some of that core flow.

## General Pros and Cons of the Single Action Handler Pattern (Summary)

**Pros:** SRP, Testability, Organization, Readability, Precise Dependencies.
**Cons:** Potentially more files, requires conscious architectural choice.

## When to Use This Pattern

* API endpoints.
* Single, distinct web actions (form processing, etc.).
* When controller actions become complex.
* To enforce SRP and improve testability.
* As the default in PSR-15 native frameworks.

## Conclusion & Framework Choice Considerations

The Single Action Handler pattern, especially when implemented using the PSR-15 `RequestHandlerInterface`, represents a powerful approach for building modern, maintainable, and testable PHP applications. Adhering to PSR standards like PSR-7 and PSR-15 brings significant benefits in interoperability, reusability, and future-proofing your codebase.

When choosing a framework with PSR adherence in mind:

* **For Native PSR-15/7 Experience:** If maximizing interoperability and working directly with PSR standards at the core HTTP layer is your highest priority, a **middleware-centric framework like Mezzio** is designed specifically for this. It offers the cleanest, most direct path to leveraging PSR-7/15.
* **For Full-Stack Power with PSR Flexibility:** If you require a comprehensive framework with a vast ecosystem, but still want the ability to cleanly integrate and prioritize PSR-15 handlers, **Symfony** provides excellent flexibility. Its robust DI container, component system, and kernel events (allowing the centralized listener approach) make it a strong contender for building large applications that adhere well to PSR standards with reasonable effort.
* **For Rapid Development & DX Prioritizing Framework Conventions:** If top development speed, an integrated ecosystem, and convention over configuration are paramount, **Laravel** excels. While achieving strict PSR-15 handler compliance requires more deliberate effort (via the centralized middleware approach, working slightly against the native flow), it's possible. However, the main draw of Laravel often lies in embracing *its* way of doing things.
