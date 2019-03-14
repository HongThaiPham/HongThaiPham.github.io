---
layout: post
title: 'Angular SEO with schema and JSON-LD'
date: 2019-03-14 13:40:00 +0700
categories: [angular]
tags: [angular, seo]
image: place_holder.jpeg
---

Author: Cory Rylan

Nov 5, 2017 

There are many important details to get right to get good SEO search results. [Schema structured data](https://developers.google.com/search/docs/guides/intro-structured-data) is a way to provide additional metadata that describes our content on our pages. This metadata can then be used by search engines like Google to provide rich SEO snippets to users. Here is an example code snippet using the JSON-LD format in our HTML.

The schema above describes an Organization. The schema is letting us be explicit about our content on our pages making it easier for search engines to understand whats on the page. There are a few formats available for schema, but Google currently recommends JSON-LD format as seen above. This format is typically more accessible to generate over other formats programmatically.

## Angular

Using the JSON-LD format causes some issues in Angular. Angular removes any script tags from templates and for a good reason. We shouldn’t be loading script tags but use es2015 modules in our Angular apps. But for use cases of inert tags like JSON-LD, this causes a problem. To use this format we have to set up a couple more steps in our Angular components.

## JSON-LD Component

To use JSON-LD, we have to use the Angular *DOMSanitizer* service to bypass Angular to allow script tags in our template. By default in Angular adding arbitrary HTML will be sanitized for any malicious code and script tags.

```typescript
import { Component, OnInit } from '@angular/core';
import { DomSanitizer, SafeHtml } from '@angular/platform-browser';

@Component({
  selector: 'app-root',
  template: '<div [innerHTML]="jsonLD"></div>'
})
export class JsonLdComponent implements OnChanges {
  jsonLD: SafeHtml;
  constructor(private sanitizer: DomSanitizer) { }

  ngOnInit(changes: SimpleChanges) {
    const json = {
      "@context": "http://schema.org",
      "@type": "Organization",
      "url": "https://google.com",
      "name": "Google",
      "contactPoint": {
        "@type": "ContactPoint",
        "telephone": "+1-000-000-0000",
        "contactType": "Customer service"
      }
    };

    this.jsonLD = this.getSafeHTML(json);
  }

  getSafeHTML(value: {}) {
    // If value convert to JSON and escape / to prevent script tag in JSON
    const json = value ? JSON.stringify(value, null, 2).replace(/\//g, '\\/') : '';
    const html = `${json}`;
    return this.sanitizer.bypassSecurityTrustHtml(html);
  }
}

```

In our component, we pass our JSON to our *getSafeHTML* method which builds our JSON-LD script tag and our JSON embedded in it. We then call the *bypassSecurityTrustHtml* method to inject our script tag into our template.

Using *bypassSecurityTrustHtml* works, and we get our JSON-LD in our templates, but we wouldn’t want to repeat this over and over in our application. Since we know how it works now, we can use a small component library I built that simplifies and encapsulates this into a single component.

## ngx-json-ld

The https://ngxlite.com package is a super small library that provides a single component that can efficiently bind JSON-LD schema into our Angular Components and supports Angular Universal server-side rendering. To use it install via npm.

```
npm install @ngx-lite/json-ld --save
```

Once installed we can add it to our *AppModule*.

```typescript
import { BrowserModule } from '@angular/platform-browser';
import { NgModule } from '@angular/core';

import { AppComponent } from './app.component';

// Import library module
import { NgxJsonLdModule } from '@ngx-lite/json-ld';

@NgModule({
  declarations: [
    AppComponent
  ],
  imports: [
    BrowserModule,

    // Register module
    NgxJsonLdModule.forRoot()
  ],
  providers: [],
  bootstrap: [AppComponent]
})
export class AppModule { }
```

Once the module is registered, we can start using it in our app.

```typescript
@Component({
  selector: 'app',
  template: `<ngx-json-ld [json]="schema"></ngx-json-ld>`
})
class AppComponent {
  schema = {
    '@context': 'http://schema.org',
    '@type': 'WebSite',
    'name': 'angular.io',
    'url': 'https://angular.io'
  };
}
```

Our given output would be the following:

```typescript
<ngx-json-ld>
  <script type="application/ld+json">
    {
      "@context": "http://schema.org",
      "@type": "WebSite",
      "name": "angular.io",
      "url": "https://angular.io"
    }
  </script>
</ngx-json-ld>
```

Check out the documentation on ngx-json-ld in the demo link below!

[Support](https://www.patreon.com/bePatron?u=10431713)
[Code demo](https://ngxlite.com/docs/json-ld)