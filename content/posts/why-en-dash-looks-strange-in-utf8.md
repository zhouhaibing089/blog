---
title: "why en-dash looks strange in utf8"
date: 2024-01-04T22:31:00-08:00
draft: false
aliases:
- why-en-dash-looks-strange-in-utf8
---

I have been using hyphen and dash exchangeably for a long time, and I thought
they are the same. Until I realized there are:

* en-dash (–)
* em-dash (—)

One is longer than the other, and both of them are longer than hyphen (-)
(probably depends on which font you are using).

I went through all this because there was an issue I investigated recently in
kubernetes.

## The story

> Note: To simplify the context, the example below is faked with only the
> details that matter.

A pod can be retrieved in JSON, but can't be retrieved in YAML because there
are control characters detected, but how?

```console
$ kubectl get pods nginx-pod -o yaml
error: error converting JSON to YAML: yaml: control characters are not allowed
$ kubectl get pods nginx-pod -o json
{
    "apiVersion": "v1",
    "kind": "Pod",
    "metadata": {
        "annotations": {
            "range": "0â9"
        },
    ...
```

In order to see the invisible characters, we can leverage other tools (for
instance, json viewer in Firefox):

```json
{
    "metadata": {
        "annotations": {
            "range": "0â\u0080\u00939"
        }
    }
}
```

We can see there are two invisible (the visible `â` is [U+00E2][3]) unicode
characters in between:

* [U+0080][1]
* [U+0093][2]

The first one is "Padding Character" while the second one is "Set Transmit
State". Both of them are control characters.

After talking with customer, the value was supposed to be `0–9` (0, en-dash, 9).
So how could an en-dash `–` turn into `\u00e2\u0080\u0093`.

## ChatGPT

In a pre-ChatGPT world, we would search for possibilities, but that is pretty
hard because we don't have an idea to start with. Encoding issue maybe, but what
should I put in a search box? I decided to give it a try on ChatGPT (free
version) and here is my conversation.

---

> What is the encoding of 0â9?

It seems there's some confusion or issue with the representation of characters
in your provided text "0â9." The correct representation of "0–9" (0 to 9 with
an en dash) in UTF-8 encoding would be:

```
30 E2 80 93 39
```

Here:

* `30` is the hexadecimal representation of the ASCII character '0'.
* `E2 80 93` represents the UTF-8 encoding of the en dash character '–'.
* `39` is the hexadecimal representation of the ASCII character '9'.

If you are seeing unexpected characters like "â," it indicates a decoding or
display issue. Ensure that you are correctly interpreting the text as UTF-8
during decoding, and also check the encoding settings of your display or
processing environment to avoid misinterpretation.

> What might be the issue?

The issue with the characters "â" appearing in the text "0â9" suggests a
problem with character encoding or decoding. Here are some common reasons for
such issues:

1. Double Encoding or Incorrect Decoding: The text may have been encoded twice
with different character encodings, or it might be decoded using an incorrect
encoding. For example, if the original encoding is UTF-8, but it is being
decoded as ISO-8859-1, you may see unexpected characters.

1. Incorrect Declaration: The encoding declaration might be incorrect or
missing. It's essential to know the correct encoding to interpret the characters
properly.

1. Data Corruption: If the data has undergone corruption during transmission or
storage, it may result in incorrect character interpretation.

To address this issue:

* Check Encoding Declarations: Ensure that the encoding declaration or
specification matches the actual encoding of the text. In this case, it should
be UTF-8.

* Use the Correct Decoding: When decoding the text, use the appropriate decoding
method for the specified encoding. In this case, use UTF-8 decoding.

* Inspect Data Sources: Check the source of the data to see if there are any
issues with how it's being generated or stored.

* Update Software: Ensure that the software you are using to read or process the
data is up-to-date and supports the expected character encoding.

If you have control over the data source, make sure the data is consistently
encoded in UTF-8. If the issue persists, further investigation into the specific
details of your system and data flow may be necessary.

> What is your guess?

Given the presence of the characters "â" in the text "0â9," it strongly
suggests a misinterpretation of character encoding, likely involving UTF-8.
Here's a common scenario:

1. The original text "0–9" is encoded in UTF-8, where the en dash character (–)
is represented by the bytes `0xE2 0x80 0x93`.
1. During decoding or display, the interpreting system or software might be
expecting a different encoding (e.g., ISO-8859-1) or incorrectly interpreting
the UTF-8 encoding, leading to the display of "â" instead of the en dash.

To address this:

* Double-check the encoding declaration or specification to ensure it is set to
UTF-8.
* Use UTF-8 decoding when interpreting the text.

If you have control over the source or decoding process, make sure the correct
encoding is used to interpret the characters. If the issue persists, inspecting
the data source, encoding settings, and the software being used for any
discrepancies or misconfigurations would be recommended.

---

## ISO-8859-1

While ChatGPT didn't say exactly what is the reason, but it did mention few
times that `ISO-8859-1` might have been used when decoding the original text.

> Note: I had a much better experience when using internal GPT which is also
> based on gpt-3.5. That conversation told me right away that the text I
> provided has a mixed encoding which includes both utf-8 and iso-8859-1.

I wrote a small Go program to verify this:

```go
package main

import (
    "fmt"
    "os"
    "unicode/utf8"

    "golang.org/x/text/encoding/charmap"
)

func main() {
    utf8Bytes := make([]byte, utf8.RuneLen('–'))
    written := utf8.EncodeRune(utf8Bytes, '–')
    fmt.Printf("– has %d bytes in utf8\n", written)
    for _, b := range utf8Bytes {
        fmt.Printf("%b ", b)
    }
    fmt.Println()

    isoBytes := make([]byte, 10)
    written, read, err := charmap.ISO8859_1.NewDecoder().Transform(
        isoBytes, utf8Bytes, true)
    if err != nil {
        fmt.Printf("failed to transform into iso8859-1: %s\n", err)
        os.Exit(1)
    }
    fmt.Printf("transformed in iso8859-1 has %d bytes (%d bytes read)\n",
        written, read)
    for i, b := range isoBytes {
        if i == written {
            break
        }
        fmt.Printf("%b ", b)
    }
    fmt.Println()
}
```

This program basically executes the following steps:

1. Reads `–` (en-dash) as UTF-8.
1. Prints out its binary form.
1. Converts to ISO8859-1.
1. Prints out its binary form.

The output is:

```
– has 3 bytes in utf8
11100010 10000000 10010011 
transformed in iso8859-1 has 6 bytes (3 bytes read)
11000011 10100010 11000010 10000000 11000010 10010011
```

## UTF-8

Now if we read the last 6 bytes as UTF-8, we have to figure out how to
interprete such bytes. In another word, we need to understand [UTF-8][4] a
little bit. Fortunately that is not hard (well, you could also just ask ChatGPT
what they are):

* UTF-8 is a multi-byte encoding.
* Any byte that starts with `10` is a continuation byte.

Essentially we have three unicodes:

* `11000011 10100010` (the second byte starts with `10` so it is a continuation
byte, but the one after is not as that starts with `11`)
* `11000010 10000000` (same)
* `11000010 10010011` (same)

Then according to the its [wiki page][4]:

|First code point|Last code point|Byte 1|Byte 2|Byte 3|Byte 4|
|-|-|-|-|-|-|
|U+0000|U+007F|0xxxxxxx||||
|U+0080|U+07FF|110xxxxx|10xxxxxx|||
|U+0800|U+FFFF|1110xxxx|10xxxxxx|10xxxxxx||
|U+10000|U+10FFFF|11110xxx|10xxxxxx|10xxxxxx|10xxxxxx|

So we can turn those binaries into the exact code point:

* `11000011 10100010` => `00011 100010` => `000 1110 0010` => `00E2`
* `11000010 10000000` => `00010 000000` => `000 1000 0000` => `0080`
* `11000010 10010011` => `00010 010011` => `000 1001 0011` => `0093`

Remember what we observed at the beginning?

```
U+00E2 U+0080 U+0093
```

That matches exactly!

And as it turns out finally, the client service which receives payload from
Github Webhook, has a different encoding `-Dfile.encoding=ISO-8859-1` so that's
how an en-dash turned into a 6 bytes as 3 unicodes.

## Summary

I stumbled on this problem for a while before I turned into ChatGPT and it
really helped on providing hints and potential explanations. I also take this
chance and now have a better understanding of UTF-8 encoding, especially after
doing all this math.

[1]: https://www.compart.com/en/unicode/U+0080
[2]: https://www.compart.com/en/unicode/U+0093
[3]: https://www.compart.com/en/unicode/U+00E2
[4]: https://en.wikipedia.org/wiki/UTF-8
