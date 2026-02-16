# Regular Expressions in JavaScript

## Overview

Regular expressions (regex) are powerful patterns for matching and manipulating text. They're frequently used in interviews for string validation, parsing, and transformation tasks.

---

## Basic Syntax

### Creating Regular Expressions

```javascript
// Literal notation (preferred for static patterns)
const regex1 = /pattern/flags;

// Constructor notation (for dynamic patterns)
const regex2 = new RegExp('pattern', 'flags');

// Dynamic pattern
const searchTerm = 'hello';
const dynamicRegex = new RegExp(searchTerm, 'i');

// Escaping special characters in dynamic patterns
function escapeRegex(string) {
  return string.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
}

const userInput = 'price: $10.00';
const escapedRegex = new RegExp(escapeRegex(userInput));
```

### Flags

```javascript
/pattern/g   // Global - find all matches, not just first
/pattern/i   // Case-insensitive
/pattern/m   // Multiline - ^ and $ match line start/end
/pattern/s   // Dotall - . matches newlines too
/pattern/u   // Unicode - treat pattern as Unicode
/pattern/y   // Sticky - match at exact position

// Combining flags
/pattern/gi  // Global and case-insensitive

// Check flags on regex
const regex = /test/gi;
regex.flags;      // "gi"
regex.global;     // true
regex.ignoreCase; // true
```

---

## Character Classes

```javascript
// Literal characters
/hello/        // Matches "hello"

// Any character (except newline)
/h.llo/        // Matches "hello", "hallo", "h9llo"

// Character set - match any one character in brackets
/[aeiou]/      // Matches any vowel
/[a-z]/        // Matches any lowercase letter
/[A-Z]/        // Matches any uppercase letter
/[0-9]/        // Matches any digit
/[a-zA-Z0-9]/  // Matches any alphanumeric

// Negated character set
/[^aeiou]/     // Matches any character that's NOT a vowel
/[^0-9]/       // Matches any non-digit

// Shorthand character classes
/\d/           // Digit [0-9]
/\D/           // Non-digit [^0-9]
/\w/           // Word character [a-zA-Z0-9_]
/\W/           // Non-word character [^a-zA-Z0-9_]
/\s/           // Whitespace (space, tab, newline)
/\S/           // Non-whitespace

// Examples
/\d{3}-\d{4}/        // Matches "123-4567"
/\w+@\w+\.\w+/       // Simple email pattern
```

---

## Quantifiers

```javascript
// Exact count
/a{3}/         // Matches "aaa"
/\d{4}/        // Matches exactly 4 digits

// Range
/a{2,4}/       // Matches "aa", "aaa", or "aaaa"
/\d{2,}/       // Matches 2 or more digits

// Common quantifiers
/a*/           // Zero or more (greedy)
/a+/           // One or more (greedy)
/a?/           // Zero or one (optional)

// Greedy vs Lazy
/a+/           // Greedy - matches as many as possible
/a+?/          // Lazy - matches as few as possible

// Example: Greedy vs Lazy
const html = '<div>content</div>';
html.match(/<.+>/);   // ["<div>content</div>"] - greedy
html.match(/<.+?>/);  // ["<div>"] - lazy

// Practical example
const str = 'aaaaaa';
str.match(/a{2,4}/);  // ["aaaa"] - greedy, takes max
str.match(/a{2,4}?/); // ["aa"] - lazy, takes min
```

---

## Anchors and Boundaries

```javascript
// Position anchors
/^hello/       // Starts with "hello"
/world$/       // Ends with "world"
/^exact$/      // Exactly matches "exact"

// Word boundary
/\bword\b/     // Matches "word" as whole word
/\bcat/        // Matches "cat" at start of word
/cat\b/        // Matches "cat" at end of word

// Examples
'cat catalog'.match(/\bcat\b/g);  // ["cat"]
'cat catalog'.match(/cat/g);       // ["cat", "cat"]

// Non-word boundary
/\Bcat/        // "cat" NOT at word boundary
'catalog'.match(/\Bcat/);  // null
'bobcat'.match(/\Bcat/);   // ["cat"]

// Multiline mode
const text = `line1
line2
line3`;

text.match(/^line/gm);  // ["line", "line", "line"]
text.match(/^line/g);   // ["line"] - only first line
```

---

## Groups and Capturing

### Capturing Groups

```javascript
// Basic capturing group
const dateRegex = /(\d{4})-(\d{2})-(\d{2})/;
const match = '2024-01-15'.match(dateRegex);

match[0];  // "2024-01-15" (full match)
match[1];  // "2024" (first group)
match[2];  // "01" (second group)
match[3];  // "15" (third group)

// Named capturing groups
const namedRegex = /(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/;
const namedMatch = '2024-01-15'.match(namedRegex);

namedMatch.groups.year;   // "2024"
namedMatch.groups.month;  // "01"
namedMatch.groups.day;    // "15"

// Destructuring named groups
const { year, month, day } = namedMatch.groups;
```

### Non-Capturing Groups

```javascript
// Non-capturing group - grouping without capturing
/(?:https?):\/\//  // Groups "http" or "https" but doesn't capture

// Compare:
'http://example.com'.match(/(https?):\/\//);
// ["http://", "http"]

'http://example.com'.match(/(?:https?):\/\//);
// ["http://"]
```

### Backreferences

```javascript
// Reference captured group within pattern
/(\w+)\s+\1/       // Matches repeated word: "hello hello"

// Named backreference
/(?<word>\w+)\s+\k<word>/  // Same with named group

// Examples
'hello hello'.match(/(\w+)\s+\1/);  // ["hello hello", "hello"]
'hello world'.match(/(\w+)\s+\1/);  // null

// Finding duplicate words
const text = 'the the quick brown fox';
text.match(/\b(\w+)\s+\1\b/g);  // ["the the"]
```

---

## Lookahead and Lookbehind

### Lookahead

```javascript
// Positive lookahead - match if followed by pattern
/foo(?=bar)/      // "foo" only if followed by "bar"

'foobar'.match(/foo(?=bar)/);  // ["foo"]
'foobaz'.match(/foo(?=bar)/);  // null

// Negative lookahead - match if NOT followed by pattern
/foo(?!bar)/      // "foo" only if NOT followed by "bar"

'foobaz'.match(/foo(?!bar)/);  // ["foo"]
'foobar'.match(/foo(?!bar)/);  // null

// Practical example: Password validation
// At least one digit, one lowercase, one uppercase
const password = /^(?=.*\d)(?=.*[a-z])(?=.*[A-Z]).{8,}$/;

password.test('Password1');  // true
password.test('password1');  // false (no uppercase)
password.test('PASSWORD1');  // false (no lowercase)
```

### Lookbehind

```javascript
// Positive lookbehind - match if preceded by pattern
/(?<=\$)\d+/      // Digits only if preceded by "$"

'$100'.match(/(?<=\$)\d+/);   // ["100"]
'€100'.match(/(?<=\$)\d+/);   // null

// Negative lookbehind - match if NOT preceded by pattern
/(?<!\$)\d+/      // Digits only if NOT preceded by "$"

'€100'.match(/(?<!\$)\d+/);   // ["100"]
'$100'.match(/(?<!\$)\d+/);   // ["00"] (matches after $1)

// Practical example: Match price without currency
const prices = 'Apple $5, Orange €3, Banana $2';
prices.match(/(?<=\$)\d+/g);  // ["5", "2"]
```

---

## String Methods with Regex

### test() - Returns boolean

```javascript
const regex = /hello/i;

regex.test('Hello World');  // true
regex.test('Hi there');     // false

// Gotcha: stateful with global flag
const globalRegex = /a/g;
globalRegex.test('aaa');  // true (index 0)
globalRegex.test('aaa');  // true (index 1)
globalRegex.test('aaa');  // true (index 2)
globalRegex.test('aaa');  // false (index reset)

// Solution: reset lastIndex or don't use global for test
globalRegex.lastIndex = 0;
```

### match() - Returns matches

```javascript
// Without global flag - returns first match with groups
const str = 'cat bat rat';
str.match(/[cbr]at/);
// ["cat", index: 0, input: "cat bat rat", groups: undefined]

// With global flag - returns all matches (no groups)
str.match(/[cbr]at/g);
// ["cat", "bat", "rat"]

// No match returns null
'hello'.match(/xyz/);  // null

// Safe access
const matches = str.match(/pattern/g) || [];
```

### matchAll() - Iterator of all matches with groups

```javascript
const str = 'test1 test2 test3';
const regex = /test(\d)/g;

// Returns iterator
const matches = str.matchAll(regex);

for (const match of matches) {
  console.log(match[0], match[1]);
}
// "test1" "1"
// "test2" "2"
// "test3" "3"

// Convert to array
const allMatches = [...str.matchAll(regex)];
```

### replace() and replaceAll()

```javascript
// Basic replace (first occurrence)
'hello world'.replace('world', 'there');
// "hello there"

// Replace all with regex
'hello world world'.replace(/world/g, 'there');
// "hello there there"

// replaceAll (ES2021)
'hello world world'.replaceAll('world', 'there');
// "hello there there"

// Using captured groups
'John Smith'.replace(/(\w+) (\w+)/, '$2, $1');
// "Smith, John"

// Named groups in replacement
'2024-01-15'.replace(
  /(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/,
  '$<month>/$<day>/$<year>'
);
// "01/15/2024"

// Function as replacement
'hello world'.replace(/\w+/g, (match) => match.toUpperCase());
// "HELLO WORLD"

// Function with groups
'John Smith'.replace(/(\w+) (\w+)/, (match, first, last) => {
  return `${last.toUpperCase()}, ${first}`;
});
// "SMITH, John"
```

### split()

```javascript
// Split on pattern
'a,b;c|d'.split(/[,;|]/);
// ["a", "b", "c", "d"]

// Split with captured delimiter
'a1b2c3d'.split(/(\d)/);
// ["a", "1", "b", "2", "c", "3", "d"]

// Limit results
'a,b,c,d'.split(/,/, 2);
// ["a", "b"]
```

### search()

```javascript
// Returns index of first match (or -1)
'hello world'.search(/world/);  // 6
'hello world'.search(/xyz/);    // -1

// Unlike indexOf, search uses regex
'Hello World'.search(/world/i); // 6
```

---

## Common Patterns

### Email Validation

```javascript
// Simple email pattern
const simpleEmail = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;

// More comprehensive (but still not perfect)
const email = /^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/;

email.test('user@example.com');     // true
email.test('user.name@domain.co');  // true
email.test('invalid@');             // false
```

### URL Validation

```javascript
// Basic URL pattern
const url = /^(https?:\/\/)?([\da-z.-]+)\.([a-z.]{2,6})([\/\w .-]*)*\/?$/;

// Using URL constructor is often better
function isValidUrl(string) {
  try {
    new URL(string);
    return true;
  } catch {
    return false;
  }
}
```

### Phone Number

```javascript
// US phone formats
const phone = /^(\+1)?[-.\s]?\(?[0-9]{3}\)?[-.\s]?[0-9]{3}[-.\s]?[0-9]{4}$/;

phone.test('123-456-7890');     // true
phone.test('(123) 456-7890');   // true
phone.test('+1 123 456 7890');  // true

// Extract parts
const phoneRegex = /^\(?(\d{3})\)?[-.\s]?(\d{3})[-.\s]?(\d{4})$/;
'(123) 456-7890'.match(phoneRegex);
// ["(123) 456-7890", "123", "456", "7890"]
```

### Password Strength

```javascript
// At least 8 chars, 1 uppercase, 1 lowercase, 1 digit, 1 special
const strongPassword = /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]{8,}$/;

// Individual checks
const hasLower = /[a-z]/;
const hasUpper = /[A-Z]/;
const hasDigit = /\d/;
const hasSpecial = /[@$!%*?&]/;
const minLength = /.{8,}/;

function checkPassword(password) {
  return {
    hasLower: hasLower.test(password),
    hasUpper: hasUpper.test(password),
    hasDigit: hasDigit.test(password),
    hasSpecial: hasSpecial.test(password),
    minLength: minLength.test(password),
    isStrong: strongPassword.test(password)
  };
}
```

### HTML Tags

```javascript
// Match HTML tags
const htmlTag = /<\/?[\w\s="/.':;#-\/]+>/gi;

// Match specific tag
const divTag = /<div[^>]*>(.*?)<\/div>/gi;

// Extract tag content
const html = '<div class="test">Hello</div>';
html.match(/<div[^>]*>(.*?)<\/div>/)[1];  // "Hello"

// Strip HTML tags
function stripTags(html) {
  return html.replace(/<[^>]*>/g, '');
}

stripTags('<p>Hello <b>World</b></p>');  // "Hello World"
```

### Whitespace Handling

```javascript
// Trim whitespace
const trim = /^\s+|\s+$/g;
'  hello  '.replace(trim, '');  // "hello"

// Collapse multiple spaces
const collapse = /\s+/g;
'hello    world'.replace(collapse, ' ');  // "hello world"

// Remove all whitespace
'h e l l o'.replace(/\s/g, '');  // "hello"
```

### Numbers

```javascript
// Integer
const integer = /^-?\d+$/;

// Decimal
const decimal = /^-?\d*\.?\d+$/;

// With thousand separators
const formatted = /^-?\d{1,3}(,\d{3})*(\.\d+)?$/;
formatted.test('1,234,567.89');  // true

// Extract numbers from string
'abc123def456'.match(/\d+/g);  // ["123", "456"]

// Parse formatted number
function parseNumber(str) {
  return parseFloat(str.replace(/,/g, ''));
}
```

---

## Interview Questions

### Q1: Remove duplicate words

```javascript
function removeDuplicateWords(str) {
  return str.replace(/\b(\w+)\s+\1\b/gi, '$1');
}

removeDuplicateWords('the the quick brown fox');
// "the quick brown fox"

removeDuplicateWords('hello hello world world');
// "hello world"
```

### Q2: Convert camelCase to kebab-case

```javascript
function camelToKebab(str) {
  return str.replace(/([a-z])([A-Z])/g, '$1-$2').toLowerCase();
}

camelToKebab('backgroundColor');  // "background-color"
camelToKebab('getHTTPResponse');  // "get-http-response"

// Handle consecutive capitals
function camelToKebabAdvanced(str) {
  return str
    .replace(/([a-z])([A-Z])/g, '$1-$2')
    .replace(/([A-Z]+)([A-Z][a-z])/g, '$1-$2')
    .toLowerCase();
}
```

### Q3: Extract query parameters

```javascript
function parseQueryString(url) {
  const params = {};
  const regex = /[?&]([^=]+)=([^&]*)/g;
  let match;

  while ((match = regex.exec(url)) !== null) {
    params[decodeURIComponent(match[1])] = decodeURIComponent(match[2]);
  }

  return params;
}

parseQueryString('https://example.com?name=John&age=30');
// { name: "John", age: "30" }

// Modern alternative using URL API
function parseQueryStringModern(url) {
  const params = new URL(url).searchParams;
  return Object.fromEntries(params);
}
```

### Q4: Validate IPv4 address

```javascript
function isValidIPv4(ip) {
  const regex = /^(\d{1,3})\.(\d{1,3})\.(\d{1,3})\.(\d{1,3})$/;
  const match = ip.match(regex);

  if (!match) return false;

  return match.slice(1).every(octet => {
    const num = parseInt(octet, 10);
    return num >= 0 && num <= 255;
  });
}

isValidIPv4('192.168.1.1');    // true
isValidIPv4('256.1.1.1');      // false
isValidIPv4('192.168.1');      // false
```

### Q5: Mask sensitive data

```javascript
function maskEmail(email) {
  return email.replace(/^(.{2})(.*)(@.*)$/, (_, start, middle, domain) => {
    return start + '*'.repeat(middle.length) + domain;
  });
}

maskEmail('john.doe@example.com');  // "jo******@example.com"

function maskCreditCard(number) {
  return number.replace(/\d(?=\d{4})/g, '*');
}

maskCreditCard('4111111111111111');  // "************1111"
```

### Q6: Template string replacement

```javascript
function template(str, data) {
  return str.replace(/\{\{(\w+)\}\}/g, (match, key) => {
    return data[key] !== undefined ? data[key] : match;
  });
}

template('Hello {{name}}, you are {{age}} years old', {
  name: 'John',
  age: 30
});
// "Hello John, you are 30 years old"
```

---

## Performance Tips

```javascript
// 1. Compile regex once, reuse
// BAD - creates new regex each call
function validate(str) {
  return /pattern/.test(str);
}

// GOOD - compile once
const pattern = /pattern/;
function validate(str) {
  return pattern.test(str);
}

// 2. Be specific to avoid backtracking
// BAD - catastrophic backtracking possible
/(.+)+$/

// GOOD - be specific
/[\w\s]+$/

// 3. Use non-capturing groups when you don't need capture
// Less memory when not capturing
/(?:https?):\/\//  // vs /(https?):\/\//

// 4. Anchor when possible
/^prefix/  // Faster than /prefix/ if checking start

// 5. Use test() for boolean checks (faster than match)
/pattern/.test(str)  // vs str.match(/pattern/)
```

---

## Key Takeaways

1. **Character classes**: `\d`, `\w`, `\s` and their negations
2. **Quantifiers**: `*`, `+`, `?`, `{n,m}` - know greedy vs lazy
3. **Anchors**: `^`, `$`, `\b` for position matching
4. **Groups**: Capturing `()`, non-capturing `(?:)`, named `(?<name>)`
5. **Lookahead/behind**: `(?=)`, `(?!)`, `(?<=)`, `(?<!)`
6. **Methods**: `test()`, `match()`, `matchAll()`, `replace()`, `split()`
7. **Flags**: `g`, `i`, `m`, `s`, `u` - know when to use each
8. **Escape special chars**: `.*+?^${}()|[\]\\`
