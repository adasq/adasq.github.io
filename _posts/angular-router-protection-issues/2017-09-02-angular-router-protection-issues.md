---
title: Angular Router Protection Issues
date: 2017-09-02 16:00:00 +07:00
modified: 2017-09-02 16:00:00 +07:00
tags: [javascript, angular]
description: Angular router is not perfect, yet. At least latest stable version, `4.3.6` which is a scope of this article.
image: "/assets/img/angular-router-protection-issues/background.jpeg"
---


Angular router is not perfect, yet. At least latest stable version, `4.3.6` which is a scope of this article. You will notice it while struggling to prototype more sophisticated routing architecture. Nested structure full of `resolve` and `canActivation` guards is a must sometimes, especially when your application grows. In this article, I will try shed light on some the difficulties I had to face lately while working with Angular Router.

<figure>
<img src="{{ page.image }}" alt="fence">
<figcaption>Fig 1. Fence</figcaption>
</figure>

But for now, let’s start with some basic stuff. Imagine we have an information page with two car brands: `tesla` and `arrinera`. Routing might look like:

```
/cars
/cars/tesla
/cars/arrinera
```

All right, let’s define routing for it:

```js
{ 
    path: 'cars',
    component: CarsComponent,
    children: [
        { path: ':cid', component: CarComponent }
    ] 
}
```

Quiet simple, huh? Now, we need some information about cars, let’s create a `CarsResolver`, as a data provider:

```js
@Injectable()
class CarsResolver implements Resolve<any> {
    public resolve() {
        return Observable.of([{name: 'tesla'}, {name: 'arrinera'}]);
    }
}
```

And update route definition:

```js
{ 
    path: 'cars',
    component: CarsComponent,
    resolve: { cars: CarsResolver },
    children: [
        { path: ':cid', component: CarComponent }
    ]
}
```

Imagine now, that we want to protect our `:cid` route to have only two values: `tesla` and `arrinera`, as provided by `CarsResolver`. Our primary objective:

```
/cars/tesla -> ok!
/cars/arrinera -> ok!
/cars/ford -> not allowed!
```

_Simple_, you might think,_ Let’s use CanActivate_. Ok, let’s give it a try. It should check whether provided by user car brand (`:cid`) is available in resolved by parent route `cars` list. If so, allow component to be initialized, prohibit otherwise.

```js
{ 
    path: 'cars',
    component: CarsComponent,
    resolve: { cars: CarsResolve },
    children: [
        { 
            path: ':cid',
            canActivate: [ CarGuard ],
            component: CarComponent 
        }
    ]
}
```

Definition of `CarGuard`:

```js
@Injectable()
class CarGuard implements CanActivate<boolean> {
    public resolve(route: ActivatedRouteSnapshot) {
        const resolvedCars = route.parent.data.cars;
        const carId = route.params.cid;
        const car = resolvedCars.find(car => car.name === carId);
        
        return Observable.of(!!car);
    }
}
```

Quite simple: access `cars` data resolved by `CarsResolver` on parent route. Find in list car brand object, by provided by a user in URL `cid`, and resolve boolean value, representing car availability.

Saving, running and bang! It does not work. `route.parent.data.cars` is not defined. Why? Because `CarsResolve` has not started yet. Under the hood, Angular calls `CanActivate` guards before data resolution (`resolve`) process steps in, so we have no access to resolved data. `canActivate` approach is more like: _Are we allowed to navigate route? If so, run resolvers, then initialize component_. To make things worse, even when we have two `canActivate` guards (one defined in child route, another in parent route), both of them are being called at the same time. It’s [well-known issue](https://github.com/angular/angular/issues/15670), which is already fixed in [5.0.0-beta.1](https://github.com/angular/angular/blob/master/CHANGELOG.md#500-beta1-2017-07-27), but leave it for now. The truth is, that in this specific circumstance, we can’t take advantages of Angular Guards capabilities.

What we need here is some hybrid of `Resolve` and `canActivate` guard. `Resolve` is allowed to access data resolved by parent route. It’s a huge advantage over `canActivate`. Guards, on the other hand, can prevent a component from being initialized, but how is it achieved? Let’s look through some framework innards…

When any user defined `canActivate` resolves to `false` ([look here](https://github.com/angular/angular/blob/4.3.6/packages/router/src/apply_redirects.ts#L309-L320)), under the hood, `canLoadFails` function is being called. This function, calls another shared function, `navigationCancelingError` which [produces an Observable error](https://github.com/angular/angular/blob/4.3.6/packages/router/src/shared.ts#L99-L105), which emits interesting instance of `Error` object, with `ngNavigationCancelingError` property set to `true`. Such well-shaped Error forces `Angular`, to not initialize component. This is what we need to protect our route! We can use this knowledge, to define our custom resolve, which will act like a `canActivate` and `resolve` at once. Let’s name it `CarResolve`.

Replace non-working `canActivate` with our new `CarResolve`:


```js
{ 
    path: 'cars',
    component: CarsComponent,
    resolve: { cars: CarsResolve },
    children: [
        { 
            path: ':cid',
            resolve: { car: CarResolve },
            component: CarComponent 
        }
    ]
}
```

And define `CarResolve` as:

```js
@Injectable()
class CarResolve implements Resolve<any> {
    public resolve(route: ActivatedRouteSnapshot) {
        
        const resolvedCars = route.parent.data.cars;
        const cid = route.params.cid;
        const car = resolvedCars.find(car => car.name === cid);
        if (car) {
            return Observable.of(car);
        } else {
            return Observable.throw({ 
                ngNavigationCancelingError: true 
            });
        }
    }
}
```

How it works? As it’s a `resolve`, we have access to resolved by parent route `cars` list. On the other hand, while throwing object with `ngNavigationCancelingError` property set, we act like `canActivate` guard. This forces Angular to reject component initialization. Eventually, `NavigationCancel` route event is being produced and sent. It will be helpful later, but for now, let’s test our routing transitions from source to target:

```
/cars -> /cars/tesla          // allowed!
/cars/tesla -> /cars/arrinera // allowed!
/cars/arrinera -> /cars/ford  // not allowed!
```

It works like a charm for state transitions! As described above, we have already source state activated (`/cars`, `/cars/tesla`, `/cars/arrinera`), and then we are trying to navigate a target route. It might succeed or fail. But what happens internally when it fails? `resetUrlToCurrentUrlTree` function is [being called](https://github.com/angular/angular/blob/4.3.6/packages/router/src/router.ts#L760-L763). What it does is trivial URL replacement for source route URL. So, when have already state activated (let’s say: `/cars`) and we are trying to navigate the invalid route, `/cars/ford`, after canceling transition, this function reverts URL to latest activated route, which is `/cars` in this case. From user perspective, nothing has changed, we stay on source route.

But what happens, when we have no source state activated? Is it possible? Yes, it is. We can achieve it by opening new browser tab, and pasting `http://localhost:4200/cars/ford` in an address bar. Angular router is trying to activate `/cars/ford` state, and it fails… Unfortunately, as we have no source state activated, a blank page is displayed, at least the root `<route-outlet>` element is not filled at all. Not sure whether it’s intended behavior, what’s more, this problem occurs also for `canActivate` guards.

How can we bypass this issue? Just by watching source URL, after route activation rejection. Let’s apply subscription for `NavigationCancel` event like this:

```js
this.router.events
    .filter(event => event instanceof NavigationCancel)
    .subscribe(event => {
         const { url } = this.router.currentRouterState.snapshot;
         if (url === '') { 
             this.router.navigate(['/cars']);
         }
    });
```

So, when we receive `NavigationCancel` event, and `ActivatedRouteSnapshot`'s `url` property points to an empty string it means, that a user opened url (which points to invalid application state) in new browser tab. We can redirect the user to specific route, preventing the blank page from being displayed. And yes, we can put this redirection inside `CarResolve` definition, but I think, that `resolve` should only resolve data (or throw an error) without additional tricky redirection logic.

[Click here](http://plnkr.co/edit/yilxe6XsHF88IrnoAq6A?p=preview) to open plunker with an example application embodying described issues. Review code and necessarily click _Launch the preview in a separate window_ icon in order to play with routing capabilities in new browser tab. Observe console output.
