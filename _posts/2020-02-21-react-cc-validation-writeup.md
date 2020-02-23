---
title: Analyzing React Credit Card Validation 
description: A detailed write-up covering the building of react-cc-validation.
permalink: "/blog/analyzing-react-credit-card-validation"
---

# Motivation

I've always known about credit card checksums and thought it would be an interesting activity to work on. I didn't want to simply calculate it by running Python from the command line, though. I wanted there to be an interface and I figured it would be fun to mess around in React, anyway.

Plus, there's just something nice about seeing the logos light up. I've seen that concept elsewhere when entering details for online purchases, so I wanted to make it happen in this project.

#### Demo

View the demo on [GitHub](https://github.com/riosdcs/react-cc-validation/blob/master/demo/demo.gif) or on in [projects]({{ site.root }}/projects/react-cc-validation.html).

#### Source code

The source code for this project is on [GitHub](https://github.com/riosdcs/react-cc-validation).

# Structure

## `App`

`App` returns a `header`, a `div` containing `<CreditCardForm />`, and a `footer`.

## `CreditCardForm`

This is where everything of interest happens, of course. 

### Componentization methodology

Since `button` is simple and only occurs once, it was left as is. `input` is a controlled component, but was not broken out into its own component. It certainly could have been, though, and may be a likely candidate in the future.

However, since I need to output multiple logos that will have varying state that may be unique from each other based on user input, they were broken out into their own component. A clearer line of reasoning about each `Logo` seemed to be best.

### Execution flow

When input is received, a number of things happen:

- `handleChange` sets `cardNumber` to the input's value
- `componentDidUpdate` is triggered after the state is changed and the component is re-rendered
- multiple conditions are checked inside `componentDidUpdate` comparing previous state to current state to determine which operation(s) to apply
- those operations then call various functions, such as `determineType`, `verifyNumber`, and `getValidMessage`
- functions then may call additional functions based on conditions, such as `purgeInactive`

Let's explore what those functions do and why they're necessary.

#### `determineType`

We loop through `prefixes`, which is a `Map` containing the supported issuers (Visa, Mastercard, Discover, and American Express) as keys and each issuer identification number (IIN), also known as prefixes, corresponding to that issuer.

> Visa and American Express have few prefixes while Mastercard and Discover have hundreds.

If the start of the input matches any of the prefixes in the list, we set the state of the component's `type` to the matching issuer, call `purgeInactive`, and return from the function.

#### `verifyNumber`

We utilize the [Luhn algorithm](https://en.wikipedia.org/wiki/Luhn_algorithm) used by most credit card issuers to determine the _potential_ validity of a given credit card number. The reason we say that it is only the potential validity is because the algorithm with not detect certain errors, such as the transposition of _09_ to _90_ (per Wikipedia).

> More robust algorithms for this purpose do exist: see the [Verhoeff](https://en.wikipedia.org/wiki/Verhoeff_algorithm) and [Damm](https://en.wikipedia.org/wiki/Damm_algorithm) algorithms.

We begin by setting `sum = 0`, making a copy of the component's `cardNumber`, setting the checksum digit to the last digit of the copy, and setting `parity` to the length of the input modulo 2.

The algorithm is as follows:

1. Double every other digit, starting from the second-to-last digit, and if
    - the result is greater than 9: add the individual digits _or_ subtract 9 from the result
    - otherwise, do nothing
2. Sum all the digits.
3. If `sum mod 10 === 0`, the number is potentially valid.

For the doubling operation, we check that `i % 2 === parity`. That is, we make sure our loop variable `i` is at an index that corresponds to starting from the second-to-last digit and doubling every other digit. Whether we start from the second-to-last or the first digit programmatically, the result is the same. As such, we loop through it forward as it's easier to reason about.

If the `sum`, making sure to include its `checkDigit` divides evenly into 10, we return `true`. Otherwise, we return `false`.

#### `getValidMessage`

We return a message indicating validity (positive or negative) based on the component's value for `valid`. This is meant to fire only if `cardNumber.length === maxLength`.

#### `purgeInactive`

To ensure that no other logos remain active when input is changed, we call `purgeInactive`. This is only called when another logo becomes active by having the input's start match a prefix. It solved a bug where once another logo have become active, it would not becoming inactive if we selected the entire input and immediately replaced it with a number that matched another issuer's prefix.

Immediately replacing input in this manner led to several issues, one of which remains and is detailed below.

#### reset

A reset `button` allows us to quickly clear the input and validity message, returning us to a clean state.

# Lessons Learned

## Pitfalls of `forEach`

We can’t use `return`, `break`, or `continue` inside `forEach`. Since it uses a callback function on each iteration, returning from any given iteration doesn’t affect the other calls to that callback function on future iterations. Similarly, we can’t `break` or `continue` as we're not actually in a loop — rather, we're inside the callback function itself.

## Changing a class based on a dynamic value

By default, a `Logo` is grayed out, or inactive, using `opacity: 0.5`. This makes sense as a card number of length `0` should never correspond to any issuer. If we enter a number whose start matches a known prefix, the corresponding issuer's `Logo` is no longer inactive, using `opacity: 1`. If the input no longer corresponds to that issuer, the `Logo` reverts to its inactive state. This occurs in real time.

To do this, we need to change the opacity of the corresponding `Logo` only. We used `activeVisa`, `activeMastercard`, `activeDiscover`, and `activeAmex` to store the active state of each `Logo`. This is far from ideal and should be iterated on further in the future.

However, we needed a way to compute the `X` portion of `activeX` while the application was running. The answer lies in computed values:

```javascript
['active' + this.state.type]: true
```

Now, we were able to set active states to `true` or `false` in real time based on the input.

## Declaring and initializing multiple variables at once

While trying to use `parseInt`, `NaN` was being output by `console.log` tests. After placing additional `console.log` tests and following the flow of execution, the culprit was found:

```javascript
let sum, temp = 0;
```

Well, then. Don’t forget to initialize your variables **properly**!

## Updating state

This is far trickier than it initially seems. State updates may happen [asynchronously](https://reactjs.org/docs/state-and-lifecycle.html#state-updates-may-be-asynchronous).

You can use two methods to ensure state is truly updated before operating on it:

- callback function in `setState`
- use `componentDidUpdate`

If using a callback function to perform some operation after `setState` finishes and the component is re-rended, using `componentDidUpdate` is [recommended by React](https://reactjs.org/docs/react-component.html#setstate).

#### Tricky state lag

American Express posed an issue several times throughout development. Since American Express credit card numbers are composed of 15 digits instead of 16 digits, `maxLength` must be updated to reflect this when the corresponding prefix is entered. This goes both ways: that is, `maxLength` must change from 16 to 15 when an American Express number is entered and must go from 15 to 16 when an American Express number is entered previous to a different number corresponding to one of the other issuers.

> When the input corresponds to no issuer, `maxLength` defaults to 16.

The issue creates a definitive error when the length of the input is at the current `maxLength` and the input is then instantly replaced by input that is of the length of the other permissible length. So, input with length 15 when `maxLength` is currently 16 or input with length 16 when `maxLength` is currently 15.

To reproduce this, we must select the entire input at once and paste the new input. The state, updating asynchronously, does not update in time to mitigate the error. As such, a number of things occur:

- `verifyNumber` is not called
- `getValidMessage` text from a previous call remains rendered
- `maxLength` updates, but not immediately
- if `maxLength` is 15 and the length of the new input is 16, the input is truncated

If the input is partially deleted or `reset` is clicked, order is restored. The cause appears to be, as indicated above, that state is being operated on that shouldn't be. The exact manner of fixing this issue is still being investigated.

[See a demonstration of this issue](https://github.com/riosdcs/react-cc-validation/blob/master/demo/selection_error.gif)

As an aside, `componentDidUpdate` feels too busy. Best practices should be looked into for this.

## Room for improvement

- manage the state lag (above)
- input validation
- insert spaces into the input as they would be on an actual credit card of the corresponding issuer
- flow