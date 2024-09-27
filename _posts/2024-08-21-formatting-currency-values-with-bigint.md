---
layout: post
title: Formatting Currency Values with BigInt
date: 2024-09-26 15:15 +0200
categories:
- aurelianix
tags:
- javascript
- TIL
---
I am building a small double-entry accounting app that serves my specific needs. One key thing is multi-currency support done right. I know that "done right" can mean whatever you want it to mean, but for my use case it means inventory tracking. All multi-currency operations should be associated with a cost, and a collection of multiple postings on an account create an inventory with values tracked at cost in my primary currency.

I live in Spain. Here, the decimal separator is comma `,` and the thousands separator can be a period `.` or a space. There are two main problems for me:

1. All programming languages I know default to printing floating point or decimal values with a dot as decimal separator.
2. I actually use integers and track each currency's smallest unit because I'm very afraid of floating points

JavaScript has the Intl API to format things according to the user's locale. The nice thing about it is that it has pre-built currency support, so you can do something like this with it:

```javascript
let formatter = new Intl.NumberFormat('es-ES', { style: 'currency', currency: 'USD' });
console.log(formatter).format(109324)
// prints: 109.324,00 US$
```

Issue 1 is solved super easily - check. Intl.NumberFormat also supports Bigint:


```javascript
let formatter = new Intl.NumberFormat('es-ES', { style: 'currency', currency: 'USD' });
console.log(formatter).format(109324n)
// prints: 109.324,00 US$
```

However, I do not want to print `109.324,00 US$`, I want this to print `1.093,24 US$` because my integer value tracks US cents, not US dollars.

Things I can do to get where I want:

### Convert BigInt to Number

```typescript
function print(value: bigint, decimalDigits: number) {
    const howManyCentsToADollar = 10**decimalDigits // 10 to the power of digits, e.g. dollar has 2 decimal digits => 10^2 => 100
    const dollarValue = Number(value) / howManyCentsToADollar
    const formatter = new Intl.NumberFormat('es-ES', { style: 'currency', currency: 'USD' });
    console.log(formatter.format(dollarValue))
}
```

This works for the most part. My value is still tracked in bigint, so I avoid errors with floating point calculations. 64 bit floating point values are fine for displaying data with up to 15 digits. In USD, 15 digits of accuracy means I can display values up to `999999999999999`, right?

```javascript
// 15 digits
print(999999999999999n) // 9.999.999.999.999,99 US$
// 16 digits
print(1999999999999999n) // 100.000.000.000.000,00 US$
```
It's more than Jeff Bezos is worth, so it should be just enough for me.

Buuuut... not everything is in USD. And every decimal digit reduces the amount of dollars you can safely track by 10! If I move to Lebanon, suddenly those 999999999999999 (the Lebanese Pound does not have any fractional unit) are "only" 11bn USD. A quick google search tells me around 150-200 people are worth more than that, and _I could be one of them_ (some day). Now's definitely the moment to future proof my function.

### Search google

A quick search on how to format bigint currency values gives me [this valuable Stack Overflow post](https://stackoverflow.com/questions/68848954/how-to-format-a-big-number-represented-by-string-in-javascript-with-currency-a).

I'll save you a click and post the solution here:

```javascript
const locale = 'de-DE'
const currency = 'EUR'
const amount = "321321321321321321.357" // parseFloat(c) gives 321321321321321340

// in the comments I also give an example for '321321321321321321.998' because of rounding issue
const [mainString, decimalString] = amount.split(".") // ['321321321321321321', '.357' | '998']

const decimalFormat = new Intl.NumberFormat(locale, { minimumFractionDigits: 2, maximumFractionDigits: 2 })
const decimalFullString = `0.${decimalString}` // '0.357' | '0.998'
const decimalFullNumber = Number.parseFloat(decimalFullString) // 0.357 | 0.998
const decimalFullFinal = decimalFormat.format(decimalFullNumber) // '0,36' | '1,00'
const decimalFinal = decimalFullFinal.slice(1) // ',36' | ',00'

const mainFormat = new Intl.NumberFormat(locale, { minimumFractionDigits: 0 })
let mainBigInt = BigInt(mainString) // 321321321321321321n
if (decimalFullFinal[0] === "1") mainBigInt += BigInt(1) // 321321321321321321n | 321321321321321322n
const mainFinal = mainFormat.format(mainBigInt) // '321.321.321.321.321.321' | '321.321.321.321.321.322'

const amountFinal = `${mainFinal}${decimalFinal}` // '321.321.321.321.321.321,36' | '321.321.321.321.321.322,00'

const currencyFormat = new Intl.NumberFormat(locale, { style: "currency", currency, maximumFractionDigits: 0 })
const template = currencyFormat.format(0) // '€0'
const result = template.replace("0", amountFinal) // '€321.321.321.321.321.321,36' | '€321.321.321.321.321.322,00
```

Just assume you got the string from a bigint value of `321321321321321321357n`. The idea is to format the main part and the fractional part separately, then to join them again. There are several gotchas to get to the end result:

- Some countries put the currency symbol after the value (e.g. "100 €"), but some before (looking at you, United States, Mexico, Australia, Argentina, Chile, Colombia, New Zealand, Hong Kong, Pacific Island nations and English-speaking Canada). Hence, the template variable uses the value 0 that ought to be replaced. Then
- We need to format the fractional part as "0.[fractional_part]" to get the correct decimal separator and prevent using a thousands separator for long decimal values.
- the fractional part must have less than 15 digits because it's parsed into a number rather than a bigint - this is necessary because of the bullet point above
- The solution rounds 3 decimal digits to 2 decimal digits because no one asked for it. It has to make sure that when truncating digits, rounding to a full value is correct. Think of `0.999` rounding to `1.00` when rounding to two digits. So suddenly, `321.999` turns into `322.00` and you need to add that value to the main part.

In general, it works, but I don't trust it one bit. It feels a bit brittle because I have zero knowledge about number representation around the world. [Wikipedia](https://en.wikipedia.org/wiki/Decimal_separator) helped me feel a bit safer that it's mostly a question of which character to use rather than "you have no idea", but come on? `decimalFullFinal.slice(1)` will break as soon as a country invents a number formatting that is always padded with 5 leading zeroes.

### Read documentation

Finally, after spending countless months on Stack Overflow, I've decided to read the Intl.NumberFormat documentation. To save everyone else the horror of having to read documentation rather than just reinventing the wheel yourself, here's the important bit:

> A Number, BigInt, or string, to format. Strings are parsed in the same way as in number conversion, except that format() will use the exact value that the string represents, avoiding loss of precision during implicitly conversion to a number.

Aha! We can just pass a string. Let's try

```javascript
let formatter = new Intl.NumberFormat('es-ES', { style: 'currency', currency: 'USD' });
formatter.format(321321321321321321.357) // 321.321.321.321.321.340,00 US$
formatter.format("321321321321321321.357") // 321.321.321.321.321.321,36 US$
```

Ok, fine. You got me. It even got that rounding thingy correctly because it knows that USD has only two decimal digits. But if it didn't know, you could specify it yourself:

```javascript
new Intl.NumberFormat('es-ES', {  minimumFractionDigits: 2, maximumFractionDigits:2 }).format(321.356)
```

### Custom currencies

Ok, Intl is super nice, but what if I want to buy Apple stock and track it as a currency? Intl.NumberFormat only takes known currencies, not some invented symbol.

Now we need string replacement again:


```typescript
function print(value: bigint, locale: string, currencyCode: string, currencyDigits: number) {
  const valueAsString = value.toString();
  // just a workaround around typescript definition for intl.numberformat - sometimes I hate literal types
  const valueAsCommaString: any =
    valueAsString.slice(0, -currencyDigits) + "." + valueAsString.slice(-currencyDigits);
  try {
    const formatter = new Intl.NumberFormat(locale, {
      currency: currencyCode,
      style: "currency",
    });
    console.log(formatter.format(valueAsCommaString));
  } catch {
    // we get here if our currency was invalid

    // this can be e.g. "USD 0.00" or "0,00 USD"
    // we hard-code USD here because we know it has two ditis
    const currencyTemplate = new Intl.NumberFormat(locale, {
      currency: "USD",
      style: "currency",
      currencyDisplay: "code",
    }).format(0);

    // this is only the formatted value, so "0.00" or "0,00",
    // omitting the "USD" code
    const valueTemplate = new Intl.NumberFormat(locale, {
      minimumFractionDigits: 2,
      maximumFractionDigits: 2,
    }).format(0);

    // this is our actual value formatted
    const formattedValue = new Intl.NumberFormat(locale, {
      minimumFractionDigits: currencyDigits,
      maximumFractionDigits: currencyDigits,
    }).format(valueAsCommaString);

    console.log(currencyTemplate.replace(valueTemplate, formattedValue).replace("USD", currencyCode));
  }
}

print(1000n, 'es-ES', 'EUR', 2)
// 10,00 €
print(1000n, 'es-ES', 'USD', 2)
// 10,00 US$
print(54322345n, 'es-ES', 'AAPL', 4)
// 5432,2345 AAPL
print(123456789123456789n, 'en-US', 'USD', 2)
// $1,234,567,891,234,567.89
print(123456789123456789n, 'en-US', 'AAPL', 4)
// AAPL 12,345,678,912,345.6789
```

Huh, looks good enough I guess. I've got a question for United States, Mexico, Australia, Argentina, Chile, Colombia, New Zealand, Hong Kong, Pacific Island nations and English-speaking Canada: Would you display your stock inventory as `AAPL 10.2341` or `10.2341 AAPL`? To me, putting the currency sign before the value looks so strange that the former just feels wrong, but maybe I'm culturally biased.

PS: I don't like the final solution, but it's the best I could come up with to include custom currency codes for stock symbols.