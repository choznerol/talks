# My many years confusion on Unicode & UTF-8

---

## Format
- We will start with **questions**
- Not structured

---

## How to participate
- Feel free to interrupt and raise discussiona at anytime!
    - especially if you have little prior knowledge on Unicode (like me from last week)

---

## Q1: Unicode v.s. UTF-8?

- charset v.s. encoding

---

## Unicode is a charset

- Unicode Character Set (UCS) is a **charset**, a mapping between **characters** and **code points**
    - Other charsets: ASCII

| Grapheme (character) | R | 讚 | 👍 | ... |
| --- | --- | --- | --- | --- |
| Unicode Code Points | [U+0052](https://www.compart.com/en/unicode/U+0052) | [U+8B9A](https://www.compart.com/en/unicode/U+8B9A) | [U+1F44D](https://www.compart.com/en/unicode/U+1F44D) | ... |

---

![unicode.org](https://i.imgur.com/A4gPbgX.png)

---

## UTF-8 is a encoding

- UTF = Unicode Transformation Format
- UTF-8 is a **encoding**, the algorithm for converting **Code Points** to **Bytes**
    - Other encoding: UTF-32, BIG5, GB2312 ...

---

## Let's see UTF-32 first

<small>

| Grapheme (character) | R | 讚 | 👍 |
| --- | --- | --- | --- |
| Unicode Code Points | [U+0052](https://www.compart.com/en/unicode/U+0052) | [U+8B9A](https://www.compart.com/en/unicode/U+8B9A) | [U+1F44D](https://www.compart.com/en/unicode/U+1F44D) ||
| UTF-32 Encoding |	`0x00000052` | `0x00008B9A` | `0x0001F44D` |

</small>
    
- What are some problem?

---

## UTF-8 is a (better) encoding

<small>

| Grapheme (character) | R | 讚 | 👍 |
| --- | --- | --- | --- |
| Unicode Code Points | [U+0052](https://www.compart.com/en/unicode/U+0052) | [U+8B9A](https://www.compart.com/en/unicode/U+8B9A) | [U+1F44D](https://www.compart.com/en/unicode/U+1F44D) |
| UTF-32 Encoding |	`0x00000052` | `0x00008B9A` | `0x0001F44D` |
| UTF-8 encoding | `0x52` <br> `[82]` | `0xE8 0xAE 0x9A` <br> `[232, 174, 154]` | `	0xF0 0x9F 0x91 0x8D` <br> `[240, 159, 145, 141]` |

</small>

---

## UTF-8 is a (better) encoding

- Variable length encoding
    - "R" takes up 1 byte v.s. "讚" takes up 3 bytes
    - Save space
    - Can't to jump character of arbitrary index
- Same ordering as unicode

---

![UTF-8](https://upload.wikimedia.org/wikipedia/commons/3/38/UTF-8_Encoding_Scheme.png)

---

### Ruby demo

```txt
Ruby讚👍🏽
```

```ruby!
content = File.open('my-utf8-file.txt').read
# => "Ruby讚👍🏽\n"





# Why Ruby know how?






Encoding.default_external
# => #<Encoding:UTF-8>





content.length  # Count by characters!





# => 8





content.split('')
# => ["R", "u", "b", "y", "讚", "👍", "🏽", "\n"]

content.bytes
# => [82, 117, 98, 121, 232, 174, 154, 240, 159, 145, 141,
#    240, 159, 143, 189, 10]





content.split('').map{|char| [char, char.bytes]}.to_h
# => {"R"=>[82],
#     "u"=>[117],
#     "b"=>[98],
#     "y"=>[121], 
#    "讚"=>[232, 174, 154],
#    "👍"=>[240, 159, 145, 141],
#    "🏽"=>[240, 159, 143, 189],
#    "\n"=>[10]}





content.split('').map{|char| [char, char.bytes, char.bytes.map{|byte| byte.to_s(2)}]}
# => 
# [["R", [82], ["1010010"]],
#  ["u", [117], ["1110101"]],
#  ["b", [98], ["1100010"]],
#  ["y", [121], ["1111001"]],
#  ["讚", [232, 174, 154], 
#       ["11101000", "10101110", "10011010"]],
#  ["👍", [240, 159, 145, 141], 
#       ["11110000", "10011111", "10010001", "10001101"]],                               
#  ["🏽", [240, 159, 143, 189],
#      ["11110000", "10011111", "10001111", "10111101"]],                               
#  ["\n", [10], ["1010"]]] 


# Look for the pattern:
# - `0xxxxxxx`
# - `1110xxxx 10xxxxxx 10xxxxxx`
# - `11110xxx 10xxxxxx 10xxxxxx`
```

---

## Q: What does "UTF-8 is backward compatible w/ ASCII" actually means?

~~A valid JavaScript program is a valid TypeScript program.~~
A valid ASCII byte string is a valid UTF-8 byte string.

```ruby!
"Ruby".encode("ASCII").bytes
# => [82, 117, 98, 121]

"Ruby".encode("UTF-32").bytes
# => [0, 0, 254, 255, 0, 0, 0, 82, 0, 0, 0, 117, 0, 0, 0, 98, 0, 0, 0, 121]

"Ruby".encode("UTF-8").bytes
# => [82, 117, 98, 121]

"Ruby".encode("UTF-16").bytes
# => [254, 255, 0, 82, 0, 117, 0, 98, 0, 121]
```

---


## Q: Why the hard-coded `4E00-9FFF` for chinese?

- Let's check out [Unicode Planes & BMP](https://zh.wikipedia.org/zh-tw/Unicode%E5%AD%97%E7%AC%A6%E5%B9%B3%E9%9D%A2%E6%98%A0%E5%B0%84#%E5%9F%BA%E6%9C%AC%E5%A4%9A%E6%96%87%E7%A7%8D%E5%B9%B3%E9%9D%A2)
    - Planes are just visualization for code points grouping

![unicode BMP](https://upload.wikimedia.org/wikipedia/commons/thumb/8/8e/Roadmap_to_Unicode_BMP.svg/1100px-Roadmap_to_Unicode_BMP.svg.png =30%x)


- Caveats
    - it’s actually CJVK
    - missing some newer CJK

---


## Q: Why Mojibake / 亂碼 / Buchstabensalat?

A encoded byte string may not contain its encoding. We must guess which encoding to use.

> ![Harry Porter](https://unicodebook.readthedocs.io/_images/Letter_to_Russia_with_krakozyabry.jpg =80%x)
> From [Programming with Unicode - 3.11 Mojibake](https://unicodebook.readthedocs.io/definitions.html#mojibake-1)

---


## Q: "Unicode" is a "encoding" options in Windows?

![Window save as unicode encoding](https://i.imgur.com/8kB8v7D.png =40%x)

- UTF-8 means `UTF-8`
- Unicode big endian means `UTF-16BE`
- Unicode means `UTF-16LE` 😱


<!--

## Q: Why is 新同文堂 awesome?


## Q: BOM and endianness

Endieness

- UTF-16 quicklook
- BOM (byte order mark)
- `0xFF 0xFE ...`: little endian
- `0xFE 0xFF`: big endian
- Gulliver's Travels
- BOM in `UTF-8`!?
- Why we don’t like UTF-16?
- Conclusion
    - Caveats


## Q: Why `UTF-8` has no endieness problem?

There is no ambiguiaty for how to interpret UTF-8 in terms of endieness.

ref: https://superuser.com/a/1648865/954056


## Q: [UTF-8] Why so many `10xxxxxx`?
Can quickly pick up in the middle.


## Q: What is `surrogate`?
-->

---

# Reference

<small>

- [YouTube - Unicode, in friendly terms: ASCII, UTF-8, code points, character encodings, and more](https://www.youtube.com/watch?v=ut74oHojxqo)
    - Only 10 mins
    - Visual demo
- [內核恐慌 Podcast](https://pan.icu/) & [字谈字暢  Podcast](https://www.thetype.com/typechat/)
    - [EP026：Kerning Panic·字谈字串（二）](https://www.thetype.com/typechat/ep-026/)
- [Programming with Unicode](https://unicodebook.readthedocs.io)
    - A comprehensive but succinct little book
- [pjchender - 瞭解網頁中看不懂的編碼：Unicode 在 JavaScript 中的使用](https://pjchender.dev/webdev/guide-unicode/)
- Wikipedia
    - [Unicode](https://zh.wikipedia.org/zh-tw/Unicode)
    - [UTF-8](https://zh.wikipedia.org/zh-tw/UTF-8)
    - [Unicode Plane](https://zh.wikipedia.org/zh-tw/Unicode%E5%AD%97%E7%AC%A6%E5%B9%B3%E9%9D%A2%E6%98%A0%E5%B0%84#%E5%9F%BA%E6%9C%AC%E5%A4%9A%E6%96%87%E7%A7%8D%E5%B9%B3%E9%9D%A2)
- [Unicode、UTF-8、UTF-16，終於懂了](https://www.readfog.com/a/1638084002220969984)
    - A great quick refresher in the future

</small>


