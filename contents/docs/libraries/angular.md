---
title: Angular
icon: >-
  https://res.cloudinary.com/dmukukwp6/image/upload/posthog.com/contents/docs/integrate/frameworks/angular.svg
---

PostHog makes it easy to get data about traffic and usage of your [Angular](https://angular.dev/) app. Integrating PostHog into your site enables analytics about user behavior, custom events capture, session recordings, feature flags, and more.

This guide walks you through integrating PostHog into your Angular app using the [JavaScript Web SDK](/docs/libraries/js).

## Installation

import AngularInstall from "../integrate/_snippets/install-angular.mdx"

<AngularInstall />

> **Note:** If you're using Typescript, you might have some trouble getting your types to compile because we depend on `rrweb` but don't ship all of their types. To accommodate that, you'll need to add `@rrweb/types@2.0.0-alpha.17` and `rrweb-snapshot@2.0.0-alpha.17` as a dependency if you want your Angular compiler to typecheck correctly.
>
> Given the nature of this library, you might need to completely clear your `.npm` cache to get this to work as expected. Make sure your clear your CI's cache as well.
>
> In the rare case the versions above get out-of-date, you can check our [JavaScript SDK's `package.json`](https://github.com/PostHog/posthog-js/blob/main/package.json) to understand what's the exact version you need to depend on.

## Capture custom events

To [capture custom events](/docs/product-analytics/capture-events), import `posthog` and call `posthog.capture()`. Below is an example of how to do this in a component:

```typescript file=app.component.ts
import { Component } from '@angular/core';
import posthog from 'posthog-js'

@Component({
 // existing component code
})

export class AppComponent {
  handleClick() {
    posthog.capture(
      'home_button_clicked', 
    )
  }
}
```

## Tracking pageviews

PostHog only captures pageview events when a [page load](https://developer.mozilla.org/en-US/docs/Web/API/Window/load_event) is fired. Since Angular creates a single-page app, this only happens once, and the Angular router handles subsequent page changes.

If we want to capture every route change, we must write code to capture pageviews that integrates with the router.

To start, we need to add `capture_pageview: false` to the PostHog initialization to avoid double capturing the first pageview.

```ts file=main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { appConfig } from './app/app.config';
import { AppComponent } from './app/app.component';
import posthog from 'posthog-js'

posthog.init(
  '<ph_project_api_key>',
  {
    api_host:'<ph_client_api_host>',
    capture_pageview: false
  }
)

bootstrapApplication(AppComponent, appConfig)
  .catch((err) => console.error(err));
```

Next, depending on your Angular version, add the following code to track pageviews:

### Tracking pageviews in Angular v17 and above

Import the following in `app.component.ts`:

1. `posthog`
2. `Router`, `Event`, and `NavigationEnd` from `@angular/router`
3. `Observable` from `rxjs` and `filter` from `rxjs/operators`

With these, we set up a subscription to the router events that captures a `$pageview` event on every `NavigationEnd` event:

```ts file=app.component.ts
import { Component } from '@angular/core';
import { RouterOutlet, Router, Event, NavigationEnd } from '@angular/router';
import { filter } from 'rxjs/operators';
import { Observable } from 'rxjs';
import posthog from 'posthog-js';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [RouterOutlet],
  templateUrl: './app.component.html',
  styleUrl: './app.component.css'
})
export class AppComponent {
  title = 'angular-spa';

  navigationEnd: Observable<NavigationEnd>;

  constructor(public router: Router) {
    this.navigationEnd = router.events.pipe(
      filter((event: Event) => event instanceof NavigationEnd)
    ) as Observable<NavigationEnd>;
  }

  ngOnInit() {
    this.navigationEnd.subscribe((event: NavigationEnd) => {
      posthog.capture('$pageview');
    });
  }
}
```

### Tracking pageviews in Angular v16 and below

To track pageviews in Angular v16 and below, import `posthog` into `app-routing.module.ts`, subscribe to router events and then capture `$pageview` events on `NavigationEnd` events:

```js
// other imports...
import { RouterModule, Routes, Router, NavigationEnd } from '@angular/router';
import posthog from 'posthog-js';

//... routes, @NgModule

export class AppRoutingModule {
  constructor(private router: Router) {
    this.router.events.subscribe(event => {
      if (event instanceof NavigationEnd) {
        posthog.capture('$pageview');
      }
    });
  }
 }
```

Now, every time a user moves between pages, PostHog captures a `$pageview` event, not just on the first page load.

## Capturing pageleaves

Setting `capture_pageview: false` also disables automatic pageleave capture. To re-enable this, add `capture_pageleave: true` to your PostHog initialization.

```ts file=main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { appConfig } from './app/app.config';
import { AppComponent } from './app/app.component';
import posthog from 'posthog-js'

posthog.init(
  '<ph_project_api_key>',
  {
    api_host:'<ph_client_api_host>',
    capture_pageview: false,
    capture_pageleave: true
  }
)

bootstrapApplication(AppComponent, appConfig)
  .catch((err) => console.error(err));
```

## Session replay

Session replay uses change detection to record the DOM. This can clash with Angular's change detection.

The recorder tool attempts to detect when an Angular zone is present and avoid the clash but might not always succeed.

If you see performance impact from recording in an Angular project, ensure that you use [`ngZone.runOutsideAngular`](https://angular.io/api/core/NgZone#runoutsideangular). 

```ts file=posthog.service.ts
import { Injectable } from '@angular/core';
import posthog from 'posthog-js'

@Injectable({ providedIn: 'root' })
export class PostHogSessionRecordingService {
  constructor(private ngZone: NgZone) {}
initPostHog() {
    this.ngZone.runOutsideAngular(() => {
      posthog.init(
        /* your config */
      )
    })
  }
}
```

## Idiomatic service for Angular v17 and above

For larger projects, you might prefer to set up PostHog as a singleton service. To do this, start by creating and injecting a `PosthogService` instance:

```ts file=main.ts
import { Component } from "@angular/core";
import { RouterOutlet } from "@angular/router";
import { PosthogService } from "./services/posthog.service";

@Component({
  selector: "app-root",
  styleUrls: ["./app.component.scss"],
  template: `
    <router-outlet />`,
  imports: [RouterOutlet],
})
export class AppComponent {
  title = "angular-app";

  constructor(posthogService: PosthogService) {}
}
```

The service itself looks like this:
```ts file=posthog.service.ts
import { DestroyRef, Injectable, NgZone } from "@angular/core";
import posthog from "posthog-js";
import { environment } from "../../environments/environment";
import { NavigationEnd, Router } from "@angular/router";
import { filter } from "rxjs";
import { takeUntilDestroyed } from "@angular/core/rxjs-interop";

@Injectable({ providedIn: "root" })
export class PosthogService {
  constructor(
    private ngZone: NgZone,
    private router: Router,
    private destroyRef: DestroyRef,
  ) {
    this.initPostHog();
  }
  private initPostHog() {
    this.ngZone.runOutsideAngular(() => {
      posthog.init(environment.posthogKey, {
        api_host: environment.posthogHost,
        debug: false,
        cross_subdomain_cookie: true,
        person_profiles: "always",
        capture_pageview: false,
        capture_pageleave: true,
        loaded: (posthogInstance) => {
          this.router.events
            .pipe(
              filter((event) => event instanceof NavigationEnd),
              takeUntilDestroyed(this.destroyRef),
            )
            .subscribe((event: NavigationEnd) => {
              posthogInstance.capture("$pageview");
            });
        },
      });
    });
  }
}
```

This service contains PostHog initialization, and upon success, enables pageviews.


## Next steps

For any technical questions for how to integrate specific PostHog features into Angular (such as feature flags, A/B testing, surveys, etc.), have a look at our [JavaScript Web SDK docs](/docs/libraries/js/features).

Alternatively, the following tutorials can help you get started:

- [How to set up Angular analytics, feature flags, and more](/tutorials/angular-analytics)
- [How to set up A/B tests in Angular](/tutorials/angular-ab-tests)
- [How to set up surveys in Angular](/tutorials/angular-surveys)

