Validating Domains with a Regular Expression
============================================

When working with URLs, using built-in libraries such as Python's `urlparse` or JavaScript's `URL` constructor often pose a security challenge: these libraries can interpret and accept non-standard hostnames/domain names which may break or exploit bugs elsewhere:

```python
>>> urlparse("https://googl<script>alert(1);#script>e.com").netloc
'googl<script>alert(1);'
```

```js
> new URL("https://google.com'onerror=alert(5)").hostname
"google.com'onerror=alert(5)"
```

To ensure you only accept well-formated *domain* names, use the following regex to validate extracted hostnames:

```
^(?!.{254,})((?!-)[A-Za-z0-9-]{1,63}(?<!-)\.)+[A-Za-z]{2,63}$
```

If you require punycode support, you can use the following regex:

```
^(?!.{254,})(?:(?!-)(?:xn--[A-Za-z0-9-]{1,59}|(?!xn--)[A-Za-z0-9-]{1,63})(?<!-)\.)+(?:xn--[A-Za-z0-9-]{1,59}|[A-Za-z]{2,63})$
```

**Note**: The above regular expressions should only be tested against extracted hostnames. Do not remove the anchors to attempt to extract hostnames from unformatted text, as it will lead to [ReDoS vulnerabilities](https://owasp.org/www-community/attacks/Regular_expression_Denial_of_Service_-_ReDoS). If you need to parse unformatted text for hostnames, split the text by spaces, and attempt to parse URLs (or something like that), before applying the regex.

**Note**: If you want to include *hostnames* (or other similar identifiers) which do not have a `.` separator (such as `hostname` or `my-local-hostname`), change `+(?:xn--[A-Za-z0-9-]{1,59}|[A-Za-z]{2,63}$` to `?(?:xn--[A-Za-z0-9-]{0,59}|[A-Za-z]{0,63}$` (or `+[A-Za-z]{2,63}$` to `?[A-Za-z]{0,63}$`).

**Note**: If you want to consider IP addresses as valid domains or hostnames, use the expressions detailed in [IP Address Validation](/IP-Address-Validation.md).

JavaScript's `URL` constructor automatically converts unicode to punycode. Python requires manual encoding:

```
>>> urlparse("https://01-БЕЗОПАСНОСТЬ.рф").netloc
'01-БЕЗОПАСНОСТЬ.рф'
>>> urlparse("https://01-БЕЗОПАСНОСТЬ.рф").netloc.encode('idna').decode() # Decodes to to punycode
'xn--01--8cdeyo3chcizco4m.xn--p1ai'
>>> urlparse("https://google.com").netloc.encode('idna').decode() # No need to decode, but it doesn't hurt
'google.com'
```

The above punycode-supported regular expression has been tested:

**Pass**:

```
stackoverflow.com
foodemo.net
bar.ba.test.co.uk
www.demo.com
foo.co.uk
regexr.com
g.com
xn--d1ai6ai.xn--p1ai
xn--stackoverflow.com
lol.international
xn--niato-otabd.xn--niato-otabd
stackoverflow.xn--com
stackoverflow.co.uk
google.com.au
a.net
0-0o.com
0-oz.co.uk
0-tension.com.br
0-wh-ao14-0.com-com.net
a-1234567890-1234567890-1234567890-1234567890-1234567890-1234-z.eu.us
www.google.com
abcdefghijklmnopqrstuvwxyz.ABCDEFGHIJKLMNOPQRSTUVWXYZ
google.com
mkyong123.com
mkyong-info.com
sub.mkyong.com
sub.mkyong-info.com
mkyong.com.au
g.co
mkyong.t.t.co
mkyong.com
```

**Fail**:

```
127.0.0.1
1.2.3.4
xn-fsqu00a.xn-0zwm56d
0123456789 +-.,!@#$%^&*();\\/|<>\"\'
12345 -98.7 3.141 .6180 9,000 +42
555.123.4567
+1-(800)-555-2468
g-.com
com.g
-g.com
-0-0o.com
0-0o_.com
-a.dot
a-1234567890-1234567890-1234567890-1234567890-1234567890-12345-z.eu.us
foo.co.uk.foo.co.uk.foo.co.uk.foo.co.uk.foo.co.uk.foo.co.uk.foo.co.uk.foo.co.uk.foo.co.uk.foo.co.uk.foo.co.uk.foo.co.uk.foo.co.uk.foo.co.uk.foo.co.uk.foo.co.uk.foo.co.uk.foo.co.uk.foo.co.uk.foo.co.uk.foo.co.uk.foo.co.uk.foo.co.uk.foo.co.uk.foo.co.uk.foo.co.uk.foo.co.uk.foo.co.uk.foo.co.uk
mkyong.t.t.c
mkyong,com
mkyong
mkyong.123
.com
-mkyong.com
mkyong-.com
sub.-mkyong.com
sub.mkyong-.com
```
