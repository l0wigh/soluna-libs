# Format Library

This library is used as a replacement for the concat method to writeln. Some idea come from Chez Scheme format primitive.

## Functions

- `(format do-print raw-string &rest values)`

## Usage

``` common-lisp
(format true "Hello from %v\n" "Soluna")
(defvar formated (format false "Username: %v, User id: %v" "L0Wigh" 1145))
```
