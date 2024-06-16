---
tags:
  - salesforce
  - lwc
  - lightning-web-component
  - javascript
  - es6
---

Today I'm announcing to the public `Bolt` for LWC. A library that I've been developping since almost 6 months now, as I worked on a deeply front-end logic centered *Salesforce Experience Cloud* App. The library has inherited from certain characteristics that are intricated with the state of the project I worked on was at that time (more on that later).

## Isn't LWC enough to make front-end intensive reactive apps in Salesforce ?

Well, to a certain extent yes. But in reality, when your app starts to grow in size, when you have multiple components with repeating logic, when you feel exhausted of rewriting the same statement over and over just to access one `SObject` and so on.. you actually feel *quit* limited by the standard capabilities of LWC. 

Ok, then what other choice do we have ? I have 2 in mind : 
* Trying to integrate *poorly* a framework like React, Angular or Vue in an Experience website
* Build the app from scratch hosted on Heroku with a synced db to Salesforce. Benefit is, you get to *choose* the tech stack

Our client wasn't willing to pay for heroku, so second option was out of the window and let's face it, trying to integrate React or whatever framework in such a **claustrophobic** environment that is Salesforce, is like shooting you in the foot before you even began to write a single line of code.

So, that's how, bit by bit, as I worked on this real-world application and with the complexity surrounding it, I came up with small utility functions here and there, that, piled up, ended up making a whole library, namely, Bolt.

## Eliminate boilerplate and get straight to the point

The main issue I had when I learned LWC (and that I still have), was that to use the salesforce *preferred way* of retrieving data in the context of an LWC, that is using [Lightning Data Service](https://developer.salesforce.com/docs/atlas.en-us.lightning.meta/lightning/data_service.htm), you have to write so many boilerplate code that, if you have to write one component every month, that's ok, but when you're dealing with a codebase of 20+ LWC, you're starting to reconsider life decisions. I wrote about this exact [topic a while ago](https://www.brtheo.dev/blog/retrieve-fields-in-lwc-:-be-smart).

So, one of the starting point of Bolt was this very issue : **reducing boilerplate to its bare minium while abstracting best practices**.

To achieve that, Bolt is heavily relying on the concept of Mixins.

## Using latest APIs

Bolt is relying on the latest LWC available APIs and therefore requires your org to be on [LightningWebSecurity](https://developer.salesforce.com/docs/platform/lwc/guide/security-lwsec-intro.html) to be fully functionnal.

Words are good, but let's see it in action. 

Let met give you the context :
1. We're building a search form to query data on an external api (that somehow we'll reach through Apex)
2. Once the data back, we want to let the user alterate them the way to want to create a salesforce record based on those received values. 
3. Finally, an external system should be aware of the creation of this new record in Salesforce and should calculate some amounts for it.
   1. So another webservice call has to be made upon saving the record
   2. The client should be listening for the response that will be sent over a `PlatformEvent`.
   

### Search Form

```js
import {track, LightningElement} from 'lwc';
import {
  mix,
  useFormValidation,
  useReactiveBinding,
  useState
} from 'c/bolt';
import calloutInApex from '@salesforce/apex/controller.calloutMethod';
export default searchForm extends mix(
  [useFormValidation],
  [useReactiveBinding],
  [useState, {IDLE: true, SEARCHING: false, RECEIVED: false, ERROR: false}],
  LightningElement
) {
  @track formData = {
    contractNumber: '',
    clientName: '',
    mileage: ''
  };
  @track apiResponse;
  async handleCalloutExternalAPI() {
    if(this.isFormValid) {
      this.SEARCHING = true;
      this.apiResponse = await calloutInApex(this.formData)
        .catch(err => this.ERROR = true);
        // some kind of latency
      if(this.apiResponse)
        this.RECEIVED = true;
    }
  }
}
```

```html
<template>
  <template lwc:if={IDLE}>
    <lightning-input
      data-checkable
      data-bind="formData.contractNumber"
      value={formData.contractNumber}
      label="Contract number"
      onchange={bind}
    ></lightning-input>
    <lightning-input
      data-checkable
      data-bind="formData.clientName"
      value={formData.clientName}
      label="Client name"
      onchange={bind}
    ></lightning-input>
    <lightning-input
      data-checkable
      data-bind="formData.mileage"
      value={formData.mileage}
      label="Mileage"
      onchange={bind}
    ></lightning-input>
    <lightning-button 
      label="Search" 
      onclick={handleCalloutExternalAPI}
    ><lightning-button/>
  </template>
  <template lwc:elseif={SEARCHING}>
    <c-bolt-skeleton rows="5"></c-bolt-skeleton>
  </template>
  <template lwc:elseif={RECEIVED}>
    <c-record-creation-form 
      lwc:spread={apiResponse}
    ></c-record-creation-form>
  </template>
  <template lwc:else>
    <!--ERROR CASE-->
  </template>
</template>
```

In this, rather long but good introduction, of Bolt we can see the use of 
* 3 mixins 
  * [`useReactiveBinding`](https://brtheo.github.io/bolt/mixins/usereactivebinding/)
  * [`useFormValidation`](https://brtheo.github.io/bolt/mixins/useformvalidation/)
  * [`useState`](https://brtheo.github.io/bolt/mixins/usestate/)
* Utility component [`<c-bolt-skeleton />`](https://brtheo.github.io/bolt/components/boltskeleton/)

**useReactiveBinding** lets you bind together one property of your class to one input, just by adding `[data-bind]` to the input and by referencing the `bind()` method to the `onchange` handler of the input.

**useFormValidation** provides you with a single getter `isFormValid` that returns a boolean and will report validity on the form's input if any errors are found.

**useState** lets you declare all the different states you lwc can be in, letting you adapt the view accordingly

**<c-bolt-skeleton />** is just an infinite loading bar component that can be used in place of a <lightning-spinner />. It is fully customisable through css custom properties.

### Record Creation Form

```js
import {
  useForm,
  useDML,
  mix,
  BoltElement
} from 'c/bolt';
import caseFields from './caseFields.js';
import contactFields from './contactFields.js';
import secondCalloutInApex from '@salesforce/apex/controller.secondCalloutMethod';
export default class recordCreationForm extends mix(
  [useForm, [caseFields, contactFields], 'insert'],
  [useDML],
  BoltElement
) {
  async handleSaveAndCallout() {
    if(this.refs.form.validity) {
      const [case, contact] = await this.saveRecords([this.Case, this.Contact])
        .catch(err => errorHandlingSomehow(err));
      if(case.Id && contact.Id) {
        const request = buildRequestSomehow(case, contact);
        const response = await secondCalloutInApex(request);
      }
    }  
  }
  get records() { return [this.$Case, this.$Contact] }
 }
```
```html
<template>
  <c-bolt-form record={$Case} mode='insert' lwc:ref="form">
  </c-bolt-form>
  <lightning-button label="Save the record" onclick={handleSaveAndCallout}>
  </lightning-button>
</template>
```
Here we can see the use of 
* 2 mixins 
  * [`useForm`](https://brtheo.github.io/bolt/mixins/useform/)
  * [`useDML`](https://brtheo.github.io/bolt/mixins/usedml/)
* The [`BoltElement`](https://brtheo.github.io/bolt/components/boltelement/) base class
* Utility component [`<c-bolt-form />`](https://brtheo.github.io/bolt/components/boltform/)

**useForm** accepts 2 parameters that are of type `Field[][]` and `string`, where `Field` is what you get when you import a field like this `import ContactName from "@salesforce/schema/Contact.Name";`. 

Because here, we're importing a bunch of fields from both the Case and Contact object, our class will be augmented with several new reactive properties :
* `this.Case`
* `this.Contact`
* `this.$Case`
* `this.$Contact`

`Case` and `Contact` props are simple JS objects whose keys will be the apiName of the imported fields and as the value their respective values. In this example, because the form is in `insert` mode, all fields will be assigned an empty default value.

The `$` prefixed props are the ones holding records information (dataType, label, picklist values ...) as well as a reference to the previous properties containing the data.

**useDML** simply provides utility methods to save or delete a single or a bunch of records.

**BoltElement** is the glue that holds this complex logic together.

**<c-bolt-form />** is a helper component that relies on Bolt's [`<c-bolt-input />`](https://brtheo.github.io/bolt/components/boltinput/) component to display in a grid all of the fields passed as attribute. It ensures data binding of any changes you made to an input.


### Listening for a PlatformEvent in an Experience Cloud website
Now let's add a third mixin to this component that will gives us the ability to *listen* for an update caused by the reception of a PlatformEvent.

> Assuming in the request that precedes the PlatformEvent you send the Id of the record that should be updated upon receiving the PE

```js
import {
  useForm,
  useDML,
  usePoller,
  mix,
  BoltElement
} from 'c/bolt';
import caseFields from './caseFields.js';
import contactFields from './contactFields.js';
import secondCalloutInApex from '@salesforce/apex/controller.secondCalloutMethod';
import getUpdatedRecord from '@salesforce/apex/controller.getUpdatedRecord';
export default class recordCreationForm extends mix(
  [useForm, [caseFields, contactFields], 'insert'],
  [useDML],
  [usePoller, {
    resolveCondition: results => results.status__c === 'WELL_RECEIVED',
    wiredMethod: 'updatedRecord',
    maxIteration: 60,
    interval: 1000
  }],
  BoltElement
) {
  @track caseId;
  connectedCallback() {
    super.connectedCallback();
    this.template.addEventListener('polling-end', (detail:{status, response}) => {
      if(status === 'OK') // doSomething
      if(status === 'POLLING_LIMIT_EXCEEDED') // doSomething
    })
  }
  @wire(getUpdatedRecord, {recordId: '$caseId'})
  updatedRecord;
  async handleSaveAndCallout() {
    if(this.refs.form.validity) {
      const [case, contact] = await this.saveRecords([this.Case, this.Contact])
        .catch(err => errorHandlingSomehow(err));
      if(case.Id && contact.Id) {
        this.caseId = case.Id;
        this.initPoller();
        const request = buildRequestSomehow(case, contact);
        const response = await secondCalloutInApex(request);
      }
    }  
  }
  get records() { return [this.$Case, this.$Contact] }
 }
```

Here we listen for the 'polling-end' event than can be fired with two values possible for the `status` parameter
* **OK** it means a record whose `id` is equals `this.caseId` with its field `status__c` has been found with a value of `WELL_RECEIVED` as defined in the `resolveCondition` of the [`usePoller`](https://brtheo.github.io/bolt/mixins/usepoller/).
* **POLLING_LIMIT_EXCEEDED** it means no record was found in the time span that was given.

## Future
For the V1, [Bolt](https://brtheo.github.io) provides :
* 13 mixins
* 4 components
* 1 base class
* 2 class factories
* 6 utility functions

A great addition I'm planning on working on, is type-safe mixins to improve a bit the DX. Because right now whenever you'll use a method or property injected by a mixin, VSCode for example, won't recognize it. So stay tuned for that !

Thanks.