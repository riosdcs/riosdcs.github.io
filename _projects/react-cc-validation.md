---
name: React Credit Card Validation
tech: React, CSS, HTML
description: React component for determining the potential validity of a credit card number.
date: "2020-02-21"
---
---

This is a React component that helps one determine the potential validity of a credit card number.

When the user enters a number, it checks that number to see if it matches any major credit card issuer's issuer identification number (IIN). The IIN is a range of numbers that a given number must begin with to be identified as belonging to a given issuer.

Once the user has entered a complete credit card number, the [Luhn algorithm](https://en.wikipedia.org/wiki/Luhn_algorithm) is run to determine the potential validity of that number.

The reset button does exactly what one might imagine: resets the state so the input field is empty and, as a result, all issuer logos are no longer active.

## Analysis

You can find a [write-up]({{ site.root }}/blog/2020-02-21-react-cc-validation.html) about this on the blog.

## Source code

You can find the source code on [GitHub](https://github.com/riosdcs/react-cc-validation).

## Demo

![react-cc-validation demo]({{ site.root }}/assets/images/demo.gif)