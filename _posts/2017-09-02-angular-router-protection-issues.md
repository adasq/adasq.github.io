---
layout: post
title: adasq3/angular-router-protection-issues
date: 2017-09-02 16:00:00
---

Angular router is not perfect, yet. At least latest stable version, `4.3.6` which is a scope of this article. You will notice it while struggling to prototype more sophisticated routing architecture. Nested structure full of `resolve` and `canActivation` guards is a must sometimes, especially when your application grows. In this article, I will try shed light on some the difficulties I had to face lately while working with Angular Router.


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

Simple, you might think, Let’s use CanActivate. Ok, let’s give it a try. It should check whether provided by user car brand (`:cid`) is available in resolved by parent route `cars` list. If so, allow component to be initialized, prohibit otherwise.

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

Saving, running and bang! It does not work. `route.parent.data.cars` is not defined. Why? Because `CarsResolve` has not started yet. Under the hood, Angular calls `CanActivate` guards before data resolution (`resolve`) process steps in, so we have no access to resolved data. `canActivate` approach is more like: Are we allowed to navigate route? If so, run resolvers, then initialize component. To make things worse, even when we have two `canActivate` guards (one defined in child route, another in parent route), both of them are being called at the same time. It’s [well-known issue|https://github.com/angular/angular/issues/15670], which is already fixed in 5.0.0-beta.1, but leave it for now. The truth is, that in this specific circumstance, we can’t take advantages of Angular Guards capabilities.
