## Selectors & React.PureComponent friendly immutable update

Reduce updates of your `redux` state means:

* Reduce recalculations of your `reselect`/`re-reselect` selectors
* Reduce re-rendering of your `react` components

### Note

`reupdate` has no dependencies to `redux`, `reselect`, `react`, etc. so you can use it with other frameworks.

## Motivation

Typically, the state changes less frequently than it is read.  

To avoid extra re-evaluations (and re-rendering) with `redux`, `reselect`, `react` and others we should return same reference to the (maybe nested) state when it was updated but actually not changed (some isDeepEqual(state, nextState) gives true). 

## Rule 

If your update of `src` value do not change it (in sense of deep equality) then `result === src` must give `true`:   

`isDeepEqual(src, value) |=> result === src`

This rule also must work for nested not changed values as is:

`isDeepEqual(src.a.b.c, value.a.b.c) |=> result.a.b.c === src.a.b.c`  

### Difference from `object-path-immutable` and `immutability-helper`

`object-path-immutable` and `immutability-helper` usually expect that you know what is the difference from `src` and `value` and some times returns reference for `value` despite it is deep equal to `src`. As a result we have extra recalculations of selectors and/or re-rendering of components.

In such cases `reupdate` returns reference to `src`, so it prevents extra recalculations and re-rendering. Profit! 

## Problem

Using native JS syntax we always create new references:  
```javascript
const myReducer = (state, action) => {  
  /* Imagine current state is: */
  state = {wip: true, data: [1, 2, 3]};
  
  /* Set new state: 
   * Create new object 
   * |=> reselect will recompute, react will re-render 
   */
  return {wip: true, data: [1, 2, 3]};
  
  /* Set new state (wip only changed): 
   * Create new object 
   * |=> reselect will recompute, react will re-render 
   */
  return {wip: false, data: [1, 2, 3]};    
  
  /* Extend existing state: 
   * Create new object 
   * |=> reselect will recompute, react will re-render 
   */
  return {...state, wip: true, data: [1, 2, 3]};
  
  /* Extend existing state (wip only changed): 
   * Create new object 
   * |=> reselect will recompute, react will re-render 
   */
  return {...state, wip: false, data: [1, 2, 3]};
};
``` 

## Solution

Use `set` and `extend` functions in your reducers:

```javascript
import {set, extend} from 'reupdate';

const myReducer = (state, action) => {  
  /* Imagine current state is: */
  state = {wip: true, data: [1, 2, 3]};
  
  /* Set new state: 
   * Return same reference to the state 
   * because nothing was actually changed 
   */
  return set(state, {wip: true, data: [1, 2, 3]}); // state   
  
  /* Set new state (wip only changed): 
   * Return new object with .data references to original state.data 
   * because it was not actually changed 
   */
  return set(state, {wip: false, data: [1, 2, 3]}); // {wip: false, data: state.data}  
  
  /* Extend existing state: 
   * Return same reference to the state 
   * because nothing was actually changed 
   */
  return extend(state, {wip: true, data: [1, 2, 3]}); // state
  
  /* Extend existing state (wip only changed): 
   * Return new object with .data references to original state.data 
   * because it was not actually changed 
   */
  return extend(state, {wip: false, data: [1, 2, 3]}); // {wip: false, data: state.data}
};
``` 

## Performance of React.Component.shouldComponentUpdate()

As it is written in the documentation of the [shouldComponentUpdate](https://reactjs.org/docs/react-component.html#shouldcomponentupdate) method:
```
We do not recommend doing deep equality checks 
or using JSON.stringify() in shouldComponentUpdate(). 
It is very inefficient and will harm performance.
```

```jsx harmony
import React, { Component } from 'react';
import { connect } from 'react-redux';
// import deepEqual from 'lodash/isEqual';

class AddressComponent extends Component {
  shouldComponentUpdate(nextProps) {
    /** Without reupdate we should use deep equal */
    // return !deepEqual(this.props.address, nextProps.address);
    
    /** With reupdate we can just compare references.
    * or just use React.PureComponent class 
    * with shallow-equal implementation of shouldComponentUpdate.
    */
    return this.props.address !== nextProps.address;
  }
  render() {
    const {country, city, street, postalCode} = this.props.address;
    return (
      <div>
        country
        <br/>
        city
        <br/>
        street
        <br/>
        postalCode
      </div>
    );
  }
}

AddressComponent = connect(
  state => ({
    address: state.user.bestFriend.address
  })
)(AddressComponent);
```

See also [React.PureComponent](https://reactjs.org/docs/react-api.html#reactpurecomponent)

## API

* [set(value, newValue)](#setvalue-newvalue)
* [setAt(value, path, newValue)](#setatvalue-path-newvalue)
* [updateAt(value, path, updater)](#updateatvalue-path-updater)
* [deleteAt(value, path)](#deleteatvalue-path)
* [extend(object, extensionObject)](#extendobject-extensionobject)
* [extendAt(value, pathToObject, extensionObject)](#extendatvalue-path-extensionobject)

### set(value, newValue)

Returns `value` if `newValue` has nothing new (is deep equal to `value`).

Returns new object/array with: 
  * Same references to not changed properties/items 
  * New references to changed properties/items

Example:
```javascript
import set from 'reupdate/set'

const src = {
  name: 'Alex',
  info: {
    greeting: 'Hello',
    description: 'I am developer'
  },
  address: {
    country: 'Russia',
    city: 'Moscow'
  },
  friends: [
    { name: 'Vasya', age: 25 }, 
    { name: 'Fedor', age: 33 }
  ]
};
const replacement = {
  name: 'Alex',
  info: {
    greeting: 'Hello',
    description: 'I am developer'
  },
  address: {
    country: 'Russia',
    city: 'Moscow'
  },
  friends: [
    { name: 'Vasya', age: 3 }, // The ONLY actual change!  
    { name: 'Fedor', age: 33 }
  ]
};

const res = set(src, replacement);

expect(res).toEqual(replacement); // Correct result

expect(res === src).toBe(false);
expect(res.info === src.info).toBe(true); // Same reference!
expect(res.address === src.address).toBe(true); // Same reference!
expect(res.friends === src.friends).toBe(false);
expect(res.friends[0] === src.friends[0]).toBe(false);
expect(res.friends[1] === src.friends[1]).toBe(true); // Same reference!
```  

### setAt(value, path, newValue)

`set` nested part of value

### updateAt(value, path, updater)

Example: 
```js
updateAt(state, 'a.b[1].c', c => c + 1)
```

### deleteAt(value, path)

Equal to `setAt(value, path, undefined)`
        
### extend(object, extensionObject)

Returns `object` if `extensionObject` has nothing new (has deep equal only properties).

Returns new object with: 
  * Same references to not changed properties 
  * New references to changed properties
  
```javascript
import extend from 'reupdate/extend'

const src = {
  name: 'Alex',
  info: {
    greeting: 'Hello',
    description: 'I am developer'
  },
  address: {
    country: 'Russia',
    city: 'Moscow'
  },
  friends: [
    { name: 'Vasya', age: 25 }, 
    { name: 'Fedor', age: 33 }
  ]
};
const extension = {
  friends: [
    { name: 'Vasya', age: 3 }, // The ONLY actual change! 
    { name: 'Fedor', age: 33 }
  ]
};

const res = extend(src, extension);

expect(res).toEqual({...src, ...extension}); // Correct result

expect(res === src).toBe(false);
expect(res.info === src.info).toBe(true); // Same reference!
expect(res.address === src.address).toBe(true); // Same reference! 
expect(res.friends === src.friends).toBe(false);
expect(res.friends[0] === src.friends[0]).toBe(false);
expect(res.friends[1] === src.friends[1]).toBe(true); // Same reference!
```  

### extendAt(value, pathToObject, extensionObject)

`extend` nested object of value