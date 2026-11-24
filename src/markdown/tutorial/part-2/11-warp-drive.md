<!--lint disable no-undefined-references-->

```run:server:start hidden=true cwd=super-rentals expect="Local:   http://localhost:4200/"
npm start
```

In this chapter, we will work on removing some code duplication in our route handlers, by switching to using WarpDrive to manage our data. The end result looks exactly the same as before:

![The Super Rentals app by the end of the chapter](/images/tutorial/part-2/warp-drive/homepage@2x.png)

During this refactor, you will learn about:

* WarpDrive schemas
* Sharing logic with plain functions
* Unit testing plain functions
* Request builders
* Loading records in routes
* The WarpDrive store and RequestManager

## What is WarpDrive?

Now that we've added some features, it's time to do some clean up again!

A while back, we added the `rental` route. If memory serves us well, we didn't do anything too fancy when we added that new route; we just copy-pasted a lot of the same logic from the `index` route.

```run:file:show lang=js cwd=super-rentals filename=app/routes/index.js
```

```run:file:show lang=js cwd=super-rentals filename=app/routes/rental.js
```

This duplication incurred a bit of *[technical debt][TODO: link to technical debt]* for us, making our code base harder to maintain in the long run. For example, if we wanted to change something about how our data-fetching logic worked, we'd have to change it in *both* the `index` and `rental` routes. If we changed things in one place, but forgot about the other spot, we could end up with really subtle bugs in our app! Yikes!

Chances are, as we keep working on this app, we will need to add more routes that fetch data from the server. Since all of our server's API endpoints follow the [JSON:API](https://jsonapi.org/) format, we'd have to keep copying this boilerplate for every single new route we add to the app!

Fortunately, we're not going to do any of that. As it turns out, there's a much better solution here: we can use WarpDrive! [WarpDrive](https://warp-drive.io/guides/) is a library that helps manage data and *[application state][TODO: link to application state]* in Ember applications.


> Zoey says...
>
> WarpDrive used to be called EmberData, and you will still see that name in older guides and addons. WarpDrive also works with other frameworks, not only Ember. You can read more about it in the [WarpDrive Docs](https://warp-drive.io)

There's a lot to learn about WarpDrive, but let's start by uncovering features that help with our immediate problem.

## WarpDrive Schemas

WarpDrive is built around the idea of organizing your app's data into *records*. Each record has a *type*, and a *[schema](https://warp-drive.io/guides/the-manual/schemas/)* that describes the fields of that type. Records represent units of information that our application presents to the user. For example, the rental property data we have been working with would be a good candidate.

Enough talking, why don't we give that a try!

```run:file:create lang=js cwd=super-rentals filename=app/schemas/rental.js
import { withDefaults } from '@warp-drive/core/reactive';

export const RentalSchema = withDefaults({
  type: 'rental',
  fields: [
    { kind: 'field', name: 'title' },
    { kind: 'field', name: 'owner' },
    { kind: 'field', name: 'city' },
    { kind: 'field', name: 'location' },
    { kind: 'field', name: 'category' },
    { kind: 'field', name: 'image' },
    { kind: 'field', name: 'bedrooms' },
    { kind: 'field', name: 'description' },
  ],
});
```

Here, we created a `RentalSchema` for the `rental` type. A schema is plain data: it lists the fields, but it does not contain any code. The `withDefaults` function adds the settings that most schemas need, such as the `id` field. When fetching the listing data from the server, each individual rental property will be represented by a *[record](https://warp-drive.io/guides/the-manual/misc/terminology#resource)* that follows this schema.

We used `field` entries to declare the attributes of a rental property. These attributes correspond directly to the `attributes` data we expect the server to provide in its responses:

```run:file:show lang=json cwd=super-rentals filename=public/api/rentals/grand-old-mansion.json
```

We can access these attributes on a rental record using standard dot notation, such as `rental.title` or `rental.location.lat`. In addition to the fields we declared here, there will always be an *id* field as well, which is used to uniquely identify the record and can be accessed using `rental.id`.

> Zoey says...
>
> The `location` field holds an object. A `field` gives us the whole object, but WarpDrive does not track changes to the values inside it. Our app never changes a location, so this is fine for us. If your app needs nested values that update, use a [`schema-object` field](https://warp-drive.io/guides/the-manual/schemas/complex-fields#schema-objects) instead.

## The Store Service

WarpDrive provides a `store` *[service](../../../services/)*. The store needs to know about our schema before it can create rental records. The default blueprint generated an `app/services/store.js` file for us. Let's replace its contents:

```run:file:create lang=js cwd=super-rentals filename=app/services/store.js
import { useRecommendedStore } from '@warp-drive/core';
import { JSONAPICache } from '@warp-drive/json-api';
import { RentalSchema } from 'super-rentals/schemas/rental';

const Store = useRecommendedStore({
  cache: JSONAPICache,
  schemas: [RentalSchema],
});

export default Store;
```

The `useRecommendedStore` function creates a store class with the recommended defaults. The `cache` option tells the store how to cache the responses from our server, which use the [JSON:API](https://jsonapi.org/) format. The `schemas` option registers our rental schema.

```run:command hidden=true cwd=super-rentals
# @ember/app-blueprint before 7.5 (ember-cli/ember-app-blueprint#338) installs the legacy package.
# Remove this line when the release channel is on 7.5.
if grep -q '"@warp-drive/legacy"' package.json; then pnpm remove @warp-drive/legacy; fi
pnpm test
git add app/schemas/rental.js
git add app/services/store.js
git add package.json
```

## Sharing the Rental Type Logic

Our schema does not have a `type` field. The server does not send one: our routes compute it from the `category` of each rental. That logic is not part of the data, so it does not belong in the schema. Instead, let's move it into a plain function that any part of our app can import:

```run:file:create lang=js cwd=super-rentals filename=app/utils/rental-type.js
const COMMUNITY_CATEGORIES = ['Condo', 'Townhouse', 'Apartment'];

export function rentalType(rental) {
  if (COMMUNITY_CATEGORIES.includes(rental.category)) {
    return 'Community';
  } else {
    return 'Standalone';
  }
}
```

Our components can call this function in their templates. In Ember, a plain function works as a *[helper](../../../components/helper-functions/)*, so `{{rentalType @rental}}` calls `rentalType` with the rental and renders the result. Let's use it in both of our rental components:

```run:file:patch lang=gjs cwd=super-rentals filename=app/components/rental.gjs
@@ -3,2 +3,3 @@ import Map from 'super-rentals/components/map';
 import { LinkTo } from '@ember/routing';
+import { rentalType } from 'super-rentals/utils/rental-type';
 
@@ -20,3 +21,3 @@ import { LinkTo } from '@ember/routing';
       <div class="detail type">
-        <span>Type:</span> {{@rental.type}}
+        <span>Type:</span> {{rentalType @rental}}
       </div>
```

```run:file:patch lang=gjs cwd=super-rentals filename=app/components/rental/detailed.gjs
@@ -4,2 +4,3 @@ import Map from 'super-rentals/components/map';
 import ShareButton from 'super-rentals/components/share-button';
+import { rentalType } from 'super-rentals/utils/rental-type';
 
@@ -31,3 +32,3 @@ import ShareButton from 'super-rentals/components/share-button';
       <div class="detail type">
-        <span>Type:</span> {{@rental.type}} – {{@rental.category}}
+        <span>Type:</span> {{rentalType @rental}} – {{@rental.category}}
       </div>
```

The components now compute the type themselves, so the rental data in our component tests does not need a `type` anymore:

```run:file:patch lang=gjs cwd=super-rentals filename=tests/integration/components/rental-test.gjs
@@ -21,3 +21,2 @@ module('Integration | Component | rental', function (hooks) {
         category: 'Estate',
-        type: 'Standalone',
         bedrooms: 15,
```

```run:file:patch lang=gjs cwd=super-rentals filename=tests/integration/components/rental/detailed-test.gjs
@@ -26,3 +26,2 @@ module('Integration | Component | rental/detailed', function (hooks) {
       category: 'Estate',
-      type: 'Standalone',
       bedrooms: 15,
```

## Unit Testing Plain Functions

So far, we haven't had a good place to write tests for the rental property's `type` logic. Now that it lives in its own function, it is easy to test this behavior. Let's create a test file for it:

```run:file:create lang=js cwd=super-rentals filename=tests/unit/utils/rental-type-test.js
import { rentalType } from 'super-rentals/utils/rental-type';
import { module, test } from 'qunit';

module('Unit | Utility | rental-type', function () {
  test('it computes the type from the category', function (assert) {
    assert.strictEqual(rentalType({ category: 'Estate' }), 'Standalone');
    assert.strictEqual(rentalType({ category: 'Condo' }), 'Community');
    assert.strictEqual(rentalType({ category: 'Townhouse' }), 'Community');
    assert.strictEqual(rentalType({ category: 'Apartment' }), 'Community');
  });
});
```

This test is also known as a *[unit test](../../../testing/unit-testing-basics/)*. Unlike any of the other tests that we've written thus far, this test doesn't actually *render* anything. It calls our function with a few rentals and checks the result. Because `rentalType` is a plain function, the test does not need the store, or even a running app. Any object with a `category` works.

Running the tests in the browser confirms that everything is working as intended:

```run:command hidden=true cwd=super-rentals
pnpm test
git add app/utils/rental-type.js
git add app/components/rental.gjs
git add app/components/rental/detailed.gjs
git add tests/integration/components/rental-test.gjs
git add tests/integration/components/rental/detailed-test.gjs
git add tests/unit/utils/rental-type-test.js
```

```run:screenshot width=1024 height=1024 retina=true filename=pass-1.png alt="All the tests pass!"
visit http://localhost:4200/tests?nocontainer&nolint&deterministic
wait  #qunit-banner.qunit-pass
```

## Request Builders

Next, let's load our rentals with the store. To do that, we need to describe each request: its URL, and anything else that the server needs. In WarpDrive, a *[builder](https://canary.warp-drive.io/guides/the-manual/requests/builders)* is a plain function that returns this description, a *request object*. A builder does not send anything. Let's write one builder for each request that our app makes:

```run:file:create lang=js cwd=super-rentals filename=app/builders/rentals.js
/**
 * Gets all rental properties.
 *
 * - Endpoint: `GET /api/rentals.json`
 */
export function getAllRentals() {
  return { url: '/api/rentals.json' };
}

/**
 * Gets one rental property.
 *
 * - Endpoint: `GET /api/rentals/:id.json`
 *
 * @param {string} id the id of the rental property
 */
export function getRental(id) {
  return { url: `/api/rentals/${id}.json` };
}
```

Each builder has a name that says what it does, and a comment that describes its endpoint. Together, the builders in `app/builders` are the one place that says how our app talks to its server. If an endpoint changes, we change its builder, and every part of the app that uses it gets the change.

> Zoey says...
>
> WarpDrive also has general builders, such as `findRecord` and `query` from `@warp-drive/utilities/json-api`. You can use them inside your own builders, but give every request in your app its own named builder.

## Loading Records in Routes

Alright, now that we have our schema and our builders, it's time to refactor our route handlers to use WarpDrive and remove the duplication!

```run:file:patch lang=js cwd=super-rentals filename=app/routes/index.js
@@ -1,22 +1,11 @@
 import Route from '@ember/routing/route';
-
-const COMMUNITY_CATEGORIES = ['Condo', 'Townhouse', 'Apartment'];
+import { service } from '@ember/service';
+import { getAllRentals } from 'super-rentals/builders/rentals';
 
 export default class IndexRoute extends Route {
-  async model() {
-    let response = await fetch('/api/rentals.json');
-    let { data } = await response.json();
-
-    return data.map((model) => {
-      let { id, attributes } = model;
-      let type;
+  @service store;
 
-      if (COMMUNITY_CATEGORIES.includes(attributes.category)) {
-        type = 'Community';
-      } else {
-        type = 'Standalone';
-      }
-
-      return { id, type, ...attributes };
-    });
+  async model() {
+    const { content } = await this.store.request(getAllRentals());
+    return content.data;
   }
```

```run:file:patch lang=js cwd=super-rentals filename=app/routes/rental.js
@@ -1,20 +1,11 @@
 import Route from '@ember/routing/route';
-
-const COMMUNITY_CATEGORIES = ['Condo', 'Townhouse', 'Apartment'];
+import { service } from '@ember/service';
+import { getRental } from 'super-rentals/builders/rentals';
 
 export default class RentalRoute extends Route {
-  async model(params) {
-    let response = await fetch(`/api/rentals/${params.rental_id}.json`);
-    let { data } = await response.json();
-
-    let { id, attributes } = data;
-    let type;
+  @service store;
 
-    if (COMMUNITY_CATEGORIES.includes(attributes.category)) {
-      type = 'Community';
-    } else {
-      type = 'Standalone';
-    }
-
-    return { id, type, ...attributes };
+  async model(params) {
+    const { content } = await this.store.request(getRental(params.rental_id));
+    return content.data;
   }
```

Wow... that removed a lot of code! This is all possible thanks to the power of conventions!

## The WarpDrive Store

As mentioned above, WarpDrive provides a `store` service, which we can inject into our route using the `@service store;` declaration, making the WarpDrive store available as `this.store`. It provides the `request` method for making fetch requests using `RequestManager`. As its name implies: the `RequestManager` is request centric. Instead of answering questions about specific records or types of records, we ask it about the status of a specific request. To start a request, we call `store.request` with the request object from one of our builders. The server responds with a JSON:API document, and `content.data` holds the rental records that the store made from it, using our schema.

WarpDrive can do many things, and in the default setup it provides caching. WarpDrive's store caches server responses, allowing instant access to previously fetched data. If the data is already cached, you don't need to wait for the server to respond again. If not, the store fetches it for you.

That's a lot of theory, but is this going to work in our app? Let's run the tests and find out!

```run:command hidden=true cwd=super-rentals
pnpm test
git add app/builders/rentals.js
git add app/routes/index.js
git add app/routes/rental.js
```

```run:screenshot width=1024 height=1024 retina=true filename=pass-2.png alt="Once again, all the tests are passing!"
visit http://localhost:4200/tests?nocontainer&nolint&deterministic
wait  #qunit-banner.qunit-pass
```

All the tests pass! The UI works exactly the same as before as well, just with much less code!

```run:screenshot width=1024 retina=true filename=homepage.png alt="The homepage works exactly the same as before, but with much less code!"
visit http://localhost:4200/?deterministic
wait  .rentals li:nth-of-type(3) article.rental
```

```run:screenshot width=1024 retina=true filename=detailed.png alt="The details page works exactly the same as before, but with much less code!"
visit http://localhost:4200/rentals/grand-old-mansion?deterministic
wait  .rental.detailed
```

WarpDrive offers many, many features (like managing the *relationships* between different types of records) and there's a lot more we can learn about it. For example, *handlers* let you change every request in one place, such as adding an authentication header. We are just scratching the surface here. If you want to learn more about WarpDrive, check out the [WarpDrive guides](https://warp-drive.io/guides/)!

```run:server:stop
npm start
```

```run:checkpoint cwd=super-rentals
Chapter 11
```
