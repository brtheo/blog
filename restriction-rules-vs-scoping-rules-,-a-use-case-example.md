---
tags:
  -  salesforce
  -  scoping rules
  -  restriction rules
  -  lwc
  -  soql
---

![who sees what](https://images.pexels.com/photos/3811807/pexels-photo-3811807.jpeg?auto=compress&cs=tinysrgb)

In the context of a salesforce application, data visibility and user permissions might be the trickiest one to deal with.

And when the source of truth regarding visibility comes from an external system, implementing it in salesforce can quickly turn out to be tricky. 

That's what we had to deal with at my job a while ago and you see how one wrong choice can lead to weird bugs and serious headaches.

## Our data model and visibility system

We have two apps running in our salesforce org :
- a partner community experience
- a crm salesforce app

They all connect to salesforce using their own ping federate company login.

### Partner community

First tricky thing is a rule dictated by our client own architecture : **one physical partner user might have multiple technical user**.
Therefore, a record owned by say, *Salesman 1-1* should be visible by *Salesman 1-2*, but of course not by *Salesman 2-1*.

If that makes sense ?

Alright then, here comes more of the fun : 

Each unique salesman belongs to one dealership. Each dealership has a salesmanager **that should has visiblity rights on each records owned by salesman of its dealership**.

### CRM

Finally, on the national level, dealerships are grouped by zones and managed by zone managers. Each zone managers **should have visibility on records owned by any salesman/salesmanager whose dealership are in its zone**.

## Implementation

Should we go standard ? Meh, of course not...

So, a bit of apex sharing to handle the sharing of partner-owned records with CRM users and we were good.

A big piece of the cake is left to be eaten though... ensuring correct visibility calculation for the multi-accounts salesman.

## The mistake

I'll quote the salesforce architect that thought of this implementation in the first place 
> Let's be honest, if those requirements were given prior to Winter ‘22, it would've be impossible to implement that.

Indeed, [restriction rules](https://help.salesforce.com/s/articleView?id=sf.security_restriction_rule.htm&type=5) and [scoping rules](https://help.salesforce.com/s/articleView?id=sf.security_scoping_rule.htm&type=5) both became generally available in salesforce release Winter ‘22. 

### Restriction rules
> Restriction rules let you enhance your security by allowing certain users to access only specified records. They prevent users from accessing records that can contain sensitive data or information that isn’t essential to their work. Restriction rules filter the records that a user has access to so that they can access only the records that match the criteria you specify.

![Restriction rules schema](https://resources.help.salesforce.com/images/30c218878df027f0e4aab7f2f176d98a.png)

### Scoping rules
> Scoping rules let you control the records that your users see based on criteria that you select. You can set up scoping rules for different users in your Salesforce org so that they can focus on the records that matter to them. Users can switch the set of records they’re seeing as needed.

![Restriction rules schema](https://resources.help.salesforce.com/images/12dd46780d27f59ae02d1bcd1f31c93e.png)

Before chosing which one will help with what we want to achieve, let's redefine our goal.

![visibility model](https://raw.githubusercontent.com/brtheo/blog/dev/visibilitymodel.png?token=GHSAT0AAAAAACRYFTY57USOG7JZXKCRBLQOZTEMEMQ)

So, 
1. Salesman XY, as a physical user, has 2 technical users that belongs to dealership X and Y. As it's the same user, he needs to keep track of his records regardless of the current technical user he's connected with. 
2. His co workers in the same dealership shouldn't see his records. 
3. Only the manager of the currently logged in multiple-accounts user should see his records.


Let's say for the UI we have a an LWC using a `<lightning-datatable` to display data coming from an apex method that returns SOQL results like so : 
```java
public static with sharing class SomeController {
  @AuraEnabled(cacheable=true)
  public static SomeObject__c[] getAllData() {
    return [
      SELECT ...
      FROM SomeObject__c
      WHERE ...
    ];
  }
}
```

Without any sharing, and by relying solely on the hierarchy system, in one dealership everyone would see each other's records. 

So, the chosed solution at the time, was to set each salesmanager as *partner super user* and to set a restriction rule applied to `SomeObject__c` on user based on a custom field representing a specific company profile name and whose check condition would be `if current_user.multiUserId__c == current_record.owner.multiUserId__c`.

So each coworkers will be prevented from seeing records whose owner `multiUserId__c` field won't be equal to their own.

And that worked... 




Unless ...

## Unless we want to do more advance CRUD
It just sounds dumb, I know, but it's the sad truth. 

For whatever reason, while active for the salesman users, this restriction rules, will prevent them from doing any other CRUD operations besides *insert*. And this will reflect back to the CRM users that were granted visibility on those records over apex sharing as well.

![this is fine](https://m.media-amazon.com/images/I/71zv--AZ4VL._AC_UF894,1000_QL80_.jpg)

The fix was quit simple actually, we just needed to think about it.

We first replaced the restriction rules with a scoping rules, evaluating basically the same condition and then had to update our SOQL query with the following statement `USING SCOPE scopingRule`
```java
public static with sharing class SomeController {
  @AuraEnabled(cacheable=true)
  public static SomeObject__c[] getAllData() {
    return [
      SELECT ...
      FROM SomeObject__c
      USING SCOPE scopingRule
      WHERE ...
    ];
  }
}
```

And tadaaa...