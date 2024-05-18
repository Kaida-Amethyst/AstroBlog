---
title: C++11上的简单format函数
description: C++20引入了std::format，但是如果我们不得不在低版本的C++上工作，而且又无法使用外部库的话，那不如自己来写一个吧。
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Cpp/4.jpg
category: 编程实践
published: 2023-06-12
tags: [C++]
---

`std::format`是c++20下的功能，但是我们可能希望在C++11下就可以使用这项功能。如果不使用外部库的话，我们可以自己来写一个。

## Description

1. 只需要支持ascii字符，无需考虑宽字符。
2. 对于每一个{}，无需考虑arg-id的问题，即第一个{}就对应第一个参数，第二个{}对应第二个参数。
3. 对于每一个{}，内部的参数类型有限，只支持类似于`{:f}`，`{:.2f}`，`{:10.5f}`，`{:8x}`，`{:04}`，`{:08x}`这一类的参数。即只考虑数字类型的参数，参数类型只有`f`，`d`，`x`， `s`或者无参数，但需要支持宽度和`0`填充。
4. 函数只可以使用C++11特性。不使用类似`sizeof...`之类的C++17特性。

---

## Examples

```cpp
fmt::format("Hello, {}!", "World") -> "Hello, World!"  
fmt::format("Hello integer, {}, {}, {}", 1, 2, 3) -> "Hello, 1, 2, 3"  
fmt::format("Hello float: {:.2f}", 3.1415926) -> "Hello float: 3.14"  
fmt::format("Hello string, integer, and float: {}, {:02x}, {:.2f}", "Hello", 1, 3.1415926) -> 
    "Hello string, integer, and float: Hello, 01, 3.14"  

std::string world = "World";  
fmt::format("Hello c++ string, {}!", world) -> "Hello, World!"
```

---

## Prototype

```cpp
namespace fmt {
    template<typename... Args>
        std::string format(const std::string& fmt, const Args&... args);
};
```

---

## Thinking

基本的思路是，扫描`fmt`，对于，每一个`{}`，如果里面没有参数，就把对应的输出参数直接调用输出流输出，如果有format参数，就解析format参数，然后再根据format格式进行输出。

另外，因为是在C++11下实现，没有`sizeof...`运算符，也没有`...`操作符，就只能递归调用模板函数。

---

## Implementation

```cpp
namespace fmt {

    template<typename T>
    std::string parse_format_arg(const std::string& fmt_arg, T& t) {
        bool fill_zero = false;
        int width = -1, precision = -1;
        char type = 'u'; // u means undefined, just print it
        if (fmt_arg.empty()) {
            return "";
        }
        size_t pos = 0;

        // skip ":"
        if (fmt_arg[0] == ':') {
            pos ++;
        }
  
        // parse the fill character, notice that in here, we only allow '0' as fill character
        if (fmt_arg[pos] == '0') {
            fill_zero = true;
            pos ++;
        }

        // parse the width
        if (fmt_arg[pos] >= '1' && fmt_arg[pos] <= '9') {
            width = fmt_arg[pos] - '0';
            pos ++;
            while (fmt_arg[pos] >= '0' && fmt_arg[pos] <= '9') {
                width = width * 10 + (fmt_arg[pos] - '0');
                pos ++;
            }
        }

        // parse the precision
        if (fmt_arg[pos] == '.') {
            pos ++;
            if (fmt_arg[pos] >= '0' && fmt_arg[pos] <= '9') {
                precision = fmt_arg[pos] - '0';
                pos ++;
                while (fmt_arg[pos] >= '0' && fmt_arg[pos] <= '9') {
                    precision = precision * 10 + (fmt_arg[pos] - '0');
                    pos ++;
                }
            }
        }

        // parse the type, notice the argument may not have type
        switch (fmt_arg[pos]) {
            case 'd':
            case 'f':
            case 'x':
            case 's':
                type = fmt_arg[pos];
                pos ++;
            case 'u':
                break;
            default:
                std::cerr << "{ERROR: FORMAT ARGUMENT TYPE NOT SUPPORTED!}";
                exit(1);
        }

        // Number check
        switch (type) {
            case 'd':
            case 'f':
            case 'x':
                if (!std::is_integral<T>::value && !std::is_floating_point<T>::value) {
                    std::cerr << "{ERROR: FORMAT ARGUMENT TYPE MISMATCH!}";
                    exit(1);
                }
                break;
        }

        std::ostringstream oss;
        if (width != -1) {
            oss << std::setw(width);
        }
        if (precision != -1) {
            oss << std::setprecision(precision);
        }
        if (fill_zero && type != 's') {
            oss << std::setfill('0');
        }

        // change t according to the format argument
        switch (type) {
            case 'd': oss << std::dec;   break;
            case 'f': oss << std::fixed; break;
            case 'x': oss << std::hex;   break;
        }
        oss << t;

        return oss.str();
    }

    // Return when sizeof...(Args) == 0
    template<typename...Args>
    void format_core(std::stringstream& ss, const std::string& fmt, size_t pos) {
        if (pos < fmt.size()) {
            ss << fmt.substr(pos);
        }
        return ;
    }

    // format call this function recursively
    template<typename T, typename...Args>
    void format_core(std::stringstream& ss, const std::string& fmt, size_t pos, const T& t, const Args&... args) {
        size_t next_pos = fmt.find('{', pos);
        if (next_pos == std::string::npos) {
            ss << fmt.substr(pos);
            return;
        }
        ss << fmt.substr(pos, next_pos - pos);
        size_t end_pos = fmt.find('}', next_pos);
        if (end_pos == std::string::npos) {
            std::cerr << "{ERROR: NO MATCHING '}' IN FMT STRING!}";
            exit(1);
        }
        std::string fmt_arg = fmt.substr(next_pos + 1, end_pos - next_pos - 1);
        fmt_arg.erase(0, fmt_arg.find_first_not_of(' '));
        if (fmt_arg.empty()) {
            ss << t;
        } else {
            ss << parse_format_arg(fmt_arg, t);
        }
        format_core(ss, fmt, end_pos + 1, args...);
    }

    template<typename... Args>
    std::string format(const std::string& fmt, const Args&... args) {
        std::stringstream ss;
        format_core(ss, fmt, 0, args...);
        return ss.str();
    }

    template<typename... Args>
    void print(const std::string& fmt, const Args&... args) {
        std::cout << format(fmt, args...);
    }

    template<typename... Args>
    void println(const std::string& fmt, const Args&... args) {
        std::cout << format(fmt, args...) << std::endl;
    }
}
```
