---
theme: gaia
paginate: true
style: |
  @import url('https://cdnjs.cloudflare.com/ajax/libs/font-awesome/5.14.0/css/all.min.css');
  @import url('https://cdn.jsdelivr.net/npm/hack-font@3/build/web/hack-subset.css');
  @import url('https://fonts.googleapis.com/css2?family=Rubik:ital,wght@0,400;0,700;1,400;1,700&display=swap');
  code { font-family: Hack; }
  section { font-family: Rubik, sans-serif; letter-spacing: 0; }
  section.lead.invert { text-shadow: 0 0 10px black, 0 0 20px black; }
  pre code { background-color: #042029; }
  .hljs-string { color: #8ae234; }
  .hljs-number, .hljs-literal { color: #729fcf; }
  .hljs-params { color: #e9b96e; font-style: italic; }
  .hljs-built_in { color: #fce94f; font-weight: bold; }
  .hljs-keyword { color: #fcaf3e; font-weight: bold; }
  .hljs-attr { color: #e9b96e; }
  .hljs-variable { color: red; font-weight: bold; }
  /* .hljs-comment, .hljs-regexp, .hljs-symbol */
---

<!--
_class: invert lead
_footer: <span style="color:yellow;">**Status: DRAFT**</span>
-->

# ⌚ **Temporal**

**Philip Chimento**
Igalia, in partnership with Bloomberg  
TC39 August/September 2021

<!--
-->

---

# Temporal update

- IETF string format standardization update
- Asking for consensus on several minor normative changes
  - Changes originally intended by the champions but incorrectly expressed in the spec text ("bugs")
  - Changes suggested by implementors ("adjustments")

---

<!-- _class: invert lead -->

# IETF string format standardization

---

TODO

---

<!-- _class: invert lead -->

# Bugs

---

<!-- _footer: ✅ spec text ✅ tests -->

### Totally wrong Duration property bag (PR [#1659](https://github.com/tc39/proposal-temporal/pull/1659))

```js
Temporal.Duration.from({ years: 5 });
// Intended: creates a Temporal.Duration of 5 years
// Actual, according to current spec text: throws RangeError
```

- Duration property bags unintentionally had to have all 10 properties!

---

<!-- _footer: ✅ spec text ✅ tests -->

### Duration string serialization bugs

```js
Temporal.Duration.from({ minutes: 5, seconds: 30 }).toString()
// Intended: 'PT5M30S'
// Actual, according to current spec text: 'PT5M30S30S'

new Temporal.Duration().toString({ fractionalSecondDigits: 2 })
// Intended: 'PT0.00S'
// Actual, according to current spec text: 'PT0S'
```

- Obviously not the intention
- Pull requests:
  - PR [#1592](https://github.com/tc39/proposal-temporal/pull/1592)
  - PR [#1725](https://github.com/tc39/proposal-temporal/pull/1725)

---

<!-- _footer: ✅ spec text ✅ tests -->

### Adjust Duration string parsing (PR [#1683](https://github.com/tc39/proposal-temporal/pull/1683))

- Valid ISO strings were inadvertently rejected by the grammar used by `Temporal.Duration.from()`
- Examples:
  - `"PT1H1S"` - minutes absent in between two other time units
  - `"P1Y1D"` - months/weeks absent in between two other calendar units

---

<!-- _footer: ✅ spec text ✅ tests -->

### Non-integers in `Duration.with` (PR [#1735](https://github.com/tc39/proposal-temporal/pull/1735))

```js
Temporal.Duration.from({ hours: 6.7 })  // throws, as intended
Temporal.Duration.from({ hours: 6 }).with({ hours: 6.7 })
// Intended: throws
// Actual, according to current spec text: a Duration of 6 hours
```

- When making non-integer Duration properties throw, in order to avoid surprises for users, we forgot about `with()`
- **TODO:** What the constructor does is on the agenda for the next champions meeting

---

<!-- _footer: ✅ spec text ✅ tests -->

### Observable order of Duration checks (PR [#1601](https://github.com/tc39/proposal-temporal/pulls/1601))

```js
called = 0;
observer = { valueOf() { called++; return Infinity; }};
d = new Temporal.Duration(observer, observer); // => throws RangeError

// Intended:
called === 1
// Actual, according to the current spec text:
called === 2
```
- Other Temporal types have the intended behaviour

---

<!-- _footer: ✅ spec text ❌ tests -->

### Unintended observable property access (PR [#1616](https://github.com/tc39/proposal-temporal/pulls/1616), [#1673](https://github.com/tc39/proposal-temporal/pull/1673))

```js
d = Temporal.Duration.from('P12M');
Object.defineProperty(d, 'months', { get() { return Infinity; }});
d.total({ unit: 'days', relativeTo: Temporal.Now.plainDateTimeISO() });
// Throws according to current spec text. Not intended
```

An accidental <span style="color: darkred;">Get(_duration_, **"months"**)</span> instead of <span style="color: darkred;">_duration_.[[Months]]</span>

---

<!-- _footer: ✅ spec text ✅ tests -->

### Accidental duplicate call in ZDT.with (PR [#1688](https://github.com/tc39/proposal-temporal/pull/1688))

```js
class T extends Temporal.TimeZone {
  constructor() { super('America/Vancouver'); }
  getPossibleInstantsFor(plainDateTime) {
    console.log(`${plainDateTime}`);
    return super.getPossibleInstantsFor(plainDateTime);
  }
}
const datetime = new Temporal.ZonedDateTime(1615708800_000_000_000n, new T());
datetime.with({ hour: 2 }, { offset: 'prefer', disambiguation: 'earlier' });
// According to current spec text: logs 2021-03-14T02:00:00, 2021-03-14T02:00:00, 2021-03-14T01:00:00
// Intended: logs 2021-03-14T02:00:00, 2021-03-14T01:00:00
//   (no duplicate call)
```

- Algorithm was unnecessarily fetching the same information twice, potentially from user code.

---

<!-- _footer: ✅ spec text ✅ tests -->

### Align difference options in PDT & ZDT (PR [#1736](https://github.com/tc39/proposal-temporal/pull/1736))

```js
class C extends Temporal.Calendar {
    constructor() { super('iso8601'); }
    dateUntil(one, two, options = {}) {
        if (options.bad) throw new Error('this is bad');
        return super.dateUntil(one, two, options);
    }
}
const options = { largestUnit: 'year', bad: true };
const start = Temporal.PlainDateTime.from('2000-01-01').withCalendar(new C());

const pdt = Temporal.Now.plainDateTime(new C());
pdt.since(start, options);  // throws; unchanged
const zdt = Temporal.Now.zonedDateTime(new C());
zdt.since(start.toZonedDateTime(Temporal.Now.timeZone()), options);
// Intended: options object passed into dateUntil; should throw
// Actual, according to current spec text: doesn't throw
```

---

### Align difference options in PDT & ZDT (cont'd)

- PlainDateTime difference passes the options object into `calendar.dateUntil()` for the benefit of custom calendar authors
- Intention was for ZonedDateTime to be consistent with PlainDateTime

---

<!-- _footer: ✅ spec text ✅ tests -->

### Wrong error type (PRs [#1646](https://github.com/tc39/proposal-temporal/pulls/1646), [#1720](https://github.com/tc39/proposal-temporal/pull/1720))

```js
instant = Temporal.Now.instant();
instant.round();  // missing a unit to round to

duration = Temporal.Duration.from({ seconds: 45 });
duration.total();  // missing a unit to get the total of

// Intended: throws TypeError
// Actual, according to current spec text: throws RangeError
```

- `TypeError` is more appropriate here than `RangeError`

---

<!-- _footer: ✅ spec text ✅ tests -->

### ±∞ in property bags (PR [#1638](https://github.com/tc39/proposal-temporal/pull/1638))

```js
date = Temporal.PlainDate.from({ year: 2021, month: 8, day: Infinity });
// Intended: throws RangeError
// Actual, according to current spec text: 2021-08-31

date = date.with({ month: -Infinity });
// Intended: throws RangeError
// Actual, according to current spec text: 2021-01-31
```
- Also, corresponding changes to all other APIs where a property bag is automatically converted into a Temporal object
- Surprising results above due to `Infinity` being subject to `{ overflow: 'constrain' }`

---

<!-- _footer: ❌ spec text ❌ tests -->

### Fix wrong value passed to user code (PR [#1667](https://github.com/tc39/proposal-temporal/pull/1667))

```js
class C extends Temporal.Calendar {
    constructor() { super('iso8601'); }
    dateAdd(date, duration, options) {
        console.log(JSON.stringify(options));
        return super.dateAdd(date, duration, options);
    }
}
plain = Temporal.PlainDateTime.from('2021-03-14T02:30').withCalendar(new C());
plain.toZonedDateTime('America/Vancouver');
// Intended: logs {"overflow":"constrain"}
// Actual, according to current spec text: logs "constrain"
```
- Passes string instead of options object to user code
- Also missing an argument

---

<!-- _footer: ❌ spec text ❌ tests (issue 1686) -->

### Object passed twice to user code (PR #TODO)

```js
class C extends Temporal.Calendar {
    constructor() { super('iso8601'); }
    dateAdd(date, duration, options) {
        const result = super.dateAdd(date, duration, options);
        options.overflow = 'bad value';
        return result;
    }
}
month = Temporal.Now.plainDate(new C()).toPlainYearMonth();
month.add({ months: 1 });
// Intended: same result as if the options object hadn't been messed with
// Actual, according to current spec text: throws RangeError
```

- We audited the spec text for instances of this, but missed two

---

### Object passed twice to user code (cont'd)

```js
class C extends Temporal.Calendar {
    constructor() { super('iso8601'); }
    dateFromFields(fields, options) {
        const result = super.dateFromFields(fields, options);
        options.overflow = 'bad value';
        return result;
    }
}
plain = Temporal.Now.plainDateTime(new C());
plain.with({ hour: 13 });
// Intended: same result as if the options object hadn't been messed with
// Actual, according to current spec text: throws RangeError
```

- **TODO**: PlainYearMonth.add and PlainYearMonth.subtract

---

<!-- _footer: ✅ spec text ✅ tests -->

### Return type of `Calendar.mergeFields()` (PR [#1719](https://github.com/tc39/proposal-temporal/pull/1719))

```js
class C extends Temporal.Calendar {
    constructor() { super('iso8601'); }
    mergeFields(fields, additionalFields) {
        return "I'm not supposed to return this";
    }
}
plain = Temporal.Now.plainDate(new C());
plain.with({ day: 1 });
// Intended: throws TypeError
// In current spec text, this fails an assertion
```

- Anywhere the `mergeFields()` of a calendar is called in the spec text, it is required to return an Object

---

<!-- _footer: ✅ spec text ✅ tests -->

### Watch out for modulo definition (PR [#1709](https://github.com/tc39/proposal-temporal/pull/1709))

- Modulo in Ecma-262 is defined differently than `%` in JS
- Defines a 'remainder' operation for mathematical values

---

<!-- _footer: ✅ spec text ✅ tests -->

### Mark options parameters as optional (PR [#1640](https://github.com/tc39/proposal-temporal/pull/1640))

- Affects `length` property of some functions

---

<!-- _footer: ✅ spec text ❌ tests -->

### Missing brand checks (PR [#1693](https://github.com/tc39/proposal-temporal/pull/1693))

TODO (ask Ms2ger about [#1692](https://github.com/tc39/proposal-temporal/issues/1692))

---

<!-- _footer: ✅ spec text ❌ tests -->

### Incorrect assertion in CalendarDaysInMonth (PR [#1716](https://github.com/tc39/proposal-temporal/pull/1716))

TODO

---

<!-- _footer: ✅ spec text ✅ tests -->

### Incorrect assertion in `Duration.compare()` (PR [#1726](https://github.com/tc39/proposal-temporal/pull/1726))

TODO

---

<!-- _footer: ✅ spec text ✅ tests -->

### Undefined variable (PR [#1687](https://github.com/tc39/proposal-temporal/pull/1687))

- Fix spec algorithm that was nonsensical due to a missing variable definition

---

<!-- _footer: ✅ spec text ✅ tests -->

### Typos that were normative 😱

- Fix algorithms that don't work as described in the current spec text due to typos
- List of pull requests:
  - [#1718](https://github.com/tc39/proposal-temporal/pull/1718)
  - [#1723](https://github.com/tc39/proposal-temporal/pull/1723)
  - [#1728](https://github.com/tc39/proposal-temporal/pull/1728) ❌

---

<!-- _class: invert lead -->

# Adjustments

---

<!-- _footer: ❌ spec text ❌ tests -->

### Mathematical values in Duration ([#1604](https://github.com/tc39/proposal-temporal/issues/1604))

- General principle, internal slots should store MVs
- Values of `Temporal.Duration` are unbounded
- Advantage: consistency, avoidance of subtle bugs
- Disadvantages: potentially disruptive for implementors, potentially less performant
- Thanks to André Bargull for noting this is a normative change

---

<!-- _footer: ❌ spec text ❌ tests -->

### No sub-minute time zone offsets ([#1544](https://github.com/tc39/proposal-temporal/issues/1544))

- IETF standardization of the time zone and calendar string annotations blocked on this (FIXME: change)
- 2021-03: Temporal must remain behind a feature flag until these annotations are standardized

---

### No sub-minute time zone offsets (cont'd)

- Change ZonedDateTime.toString and Instant.toString to output time zone offsets only with minutes precision
- Change ZonedDateTime.from to accept HH:MM precision for non-minute-offset time zones, even with `{ offset: 'reject' }`.
- No change to `offset` property of ZonedDateTime, or in property bags, or to TimeZone.getOffsetStringFor.

---

```js
timeZone = Temporal.TimeZone.from('Africa/Monrovia');
zdt = Temporal.PlainDate.from('1972-01-01').toZonedDateTime(timeZone);

// Before:
zdt.toString() === '1972-01-01T00:00:00-00:44:30[Africa/Monrovia]'
zdt.toInstant().toString({ timeZone }) === '1972-01-01T00:00:00-00:44:30'
zdt.offset === '-00:44:30'
timeZone.getOffsetStringFor(zdt.toInstant()) === '-00:44:30'
Temporal.ZonedDateTime.from('1972-01-01T00:00:00-00:44:30[Africa/Monrovia]').equals(zdt)
Temporal.ZonedDateTime.from('1972-01-01T00:00:00-00:45[Africa/Monrovia]') // throws

// After:
zdt.toString() === '1972-01-01T00:00:00-00:45[Africa/Monrovia]'
zdt.toInstant().toString({ timeZone }) === '1972-01-01T00:00:00-00:45'
zdt.offset === '-00:44:30'
timeZone.getOffsetStringFor(zdt.toInstant()) === '-00:44:30'
Temporal.ZonedDateTime.from('1972-01-01T00:00:00-00:44:30[Africa/Monrovia]').equals(zdt)
Temporal.ZonedDateTime.from('1972-01-01T00:00:00-00:45[Africa/Monrovia]').equals(zdt)
```

---

<!-- _footer: ❌ spec text ❌ tests -->

### Guard against garbage in `Calendar.fields` (PR TODO)

- `Temporal.Calendar.prototype.fields()` expects an iterable as argument
  - Will now throw if the iterable yields any duplicate values
  - Will now throw if the iterable yields any values that are not `'year'`, `'month'`, `'monthCode'`, `'day'`, `'hour'`, `'minute'`, `'second'`, `'millisecond'`, `'microsecond'`, `'nanosecond'`

---

### Guard against garbage (cont'd)

```js
Temporal.Calendar.from('iso8601').fields({
    *[Symbol.iterator]() {
        while(true)
            yield 'garbage';
    }
});
// Before: infinite loop
// After: throws RangeError
```

---

<!-- _footer: ✅ spec text ✅ tests -->

### Align addition in PlainDate & Calendar (PR [#1710](https://github.com/tc39/proposal-temporal/pull/1710))

- The treatment of Duration input differed between the common case of adding with PlainDate and the lower-level case of adding with Calendar directly
  - Occurs when adding e.g. 24 hours, 86400 seconds
  - Per [an earlier decision](https://github.com/tc39/proposal-temporal/issues/324), smaller units are balanced up to larger units, but calendars as separate objects didn't exist at that time
- Advantage: Less surprise for programmers
- Disadvantage: Chance for userland calendars to be inconsistent

---

```js
calendar = new Temporal.Calendar('iso8601');
date = new Temporal.PlainDate(2000, 1, 1, calendar);

date.add({ hours: 24 }) // 2000-01-02
calendar.dateAdd(date, { hours: 24 })
// Before: throws TypeError due to no applicable properties in property bag
// After: 2000-01-02

date.add('PT24H') // 2000-01-02
calendar.dateAdd(date, 'PT24H')
// Before: 2000-01-01
// After: 2000-01-02

hours24 = new Temporal.Duration(0, 0, 0, 0, 24);
date.add(hours24) // 2000-01-02
calendar.dateAdd(date, hours24)
// Before: 2000-01-01
// After: 2000-01-02
```

---

<!-- _footer: ❌ spec text ❌ tests -->

### Consistent default options (PR #TODO)

- When Temporal invokes a calendar operation with the default options:
  - Sometimes passed `undefined` as the options argument
  - Sometimes passed `Object.create(null)` as the options argument
- This should be consistent, because it is observable in userland calendars

---

<!-- _footer: ✅ spec text ✅ tests -->

### Consistent order of operations in toPlainDate (PR [#1734](https://github.com/tc39/proposal-temporal/pull/1734))

```js
class C extends Temporal.Calendar {
  constructor() { super('iso8601'); }
  fields(f) {
    console.log('boo!');
    return super.fields(f);
  }
}
const yearMonth = new Temporal.PlainYearMonth(2021, 8, new C());
yearMonth.toPlainDate('bad input');  // throws TypeError; unchanged
const monthDay = new Temporal.PlainMonthDay(8, 31, new C());
monthDay.toPlainDate('bad input');
// Before: logs boo!, then throws TypeError
// After: throws TypeError
```
- Fixes an inconsistency in the order of user-visible operations
- Allows implementors to combine an operation

---

<!-- _footer: ❌ spec text ❌ tests -->

### TODO: -0 in Duration.negated

On the agenda for next champions meeting
