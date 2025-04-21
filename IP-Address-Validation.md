Validating IP Addresses with Regular Expressions
============================================

Validating IP addresses using regex is generally easy for IPv4, and generally difficult for IPv6. Regex is not really the correct solution here, but it is sometimes the preferred method.

# IPv4

The following two expressions conform to the restrictions of standard IPv4 addresses.

With PCRE:

```
^((25[0-5]|(2[0-4]|1\d|[1-9]|)\d)\.?\b){4}$
```

With Extended Regular Expressions ([EXE](https://en.wikibooks.org/wiki/Regular_Expressions/POSIX-Extended_Regular_Expressions):

```
^((25[0-5]|2[0-4][0-9]|1[0-9][0-9]|[1-9]?[0-9])\.){3}(25[0-5]|2[0-4][0-9]|1[0-9][0-9]|[1-9]?[0-9])$
```

The EXE version can be helpful when using grep (I don't like to use the `-P` flag). For example, my bashrc contains:

```
alias ipgrep='grep -E "\b((25[0-5]|2[0-4][0-9]|1[0-9][0-9]|[1-9]?[0-9])\.){3}(25[0-5]|2[0-4][0-9]|1[0-9][0-9]|[1-9]?[0-9])\b"'
```

The `\b` values on each side ensures that `1.1.1.1a` and `1.1.1.1234` do not partially match, while `1.1.1.1.some` partially matches the `1.1.1.1`.

Both expressions (PCRE and EXE) have been tested:

# Pass

```
127.0.0.1
192.168.1.1
192.168.1.255
255.255.255.255
0.0.0.0
```

# Fail

```
1.1.1.01
30.168.1.255.1
127.1
192.168.1.256
-1.2.3.4
1.1.1.1.
3...3
1.1.1.01
1.1.1.01
```

# IPv6

TODO. Tested against https://regex101.com/r/5gqiQm/1
