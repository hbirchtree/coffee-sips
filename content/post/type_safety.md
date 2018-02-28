---
title: "Templates for secure integer casting"
date: 2018-02-24T04:02:25+01:00
draft: false
tags:
  - "cpp"
  - "stl-templates"
---

There are several properties desirable when casting integers between different sizes and signs. Let's look at an example:

We have the suspicious number `1234567890`, which we have no knowledge about in our system. This might represent a filesize (although, you really should keep file sizes in `size_t`). In a typical scenario, assigning this to a `uint16_t` type might look harmless on the surface, assuming that "the files won't be that big anyway".
Accident strikes.
The number in the `uint16_t` variable is now `722`.
The number we get out of the `uint16_t` is hard to interpret, it might look correct.
The worse case is between `signed` and `unsigned` types. Imagine `-1` being cast into a `uint32_t`. The number quickly becomes `2^31 - 1`, which looks quite suspicious, but it's not practical to check for.

A possible solution is this:

 - Is the number larger than its new containing type? Clamp it to the numeric limit of the new type.
 - Is the number outside of the representable range? Clamp it either to the numerical minimum limit or maximum limit.

This would mean that `-1` becomes `0` when casting `signed` types to `unsigned`, and for the reverse case, a maximum value is chosen. The results are more consistent, and typical errors with reading bad memory (as a result of returning or getting `-1` as a size) are eliminated. Although a lot of these problems may be mitigated by sticking to `unsigned` types all over, but some native APIs (such as POSIX `fstat`) return `signed` types.

The following template does a compile-time check for the types being cast, but runtime-checking of the value. The runtime portion can be shortened immensely by using `constexpr` semantics, but the codebase is aiming for C++11 compatibility. Still, a good compiler at -O2 or above will crush these tests into very few instructions.

{{< highlight cpp >}}
#define IS_INT(Type) std::is_integral<Type>::value

template<typename D,
         typename T,
         typename std::enable_if<IS_INT(D) && IS_INT(T)>::type* = nullptr,
         typename std::enable_if<
    (std::numeric_limits<T>::max() > std::numeric_limits<D>::max()) ||
    (std::numeric_limits<T>::min() < std::numeric_limits<D>::min())
             >::type* = nullptr>
/*!
 * \brief If there is a risk of going out of the
 *  range of type D, return the closest value.
 * \param from
 * \return
 */
static inline D C_FCAST(T from)
{
    static constexpr D max_val = std::numeric_limits<D>::max();
    static constexpr D min_val = std::numeric_limits<D>::min();

    static constexpr T T_max_val = std::numeric_limits<T>::max();
//    static constexpr T T_min_val = std::numeric_limits<T>::min();

    static constexpr bool T_is_signed = std::is_signed<T>::value;
    static constexpr bool D_is_signed = std::is_signed<D>::value;
    static constexpr bool same_sign = T_is_signed == D_is_signed;

    const ::uint64_t T_max_val_u64 = static_cast<::uint64_t>(T_max_val);
    const ::uint64_t D_max_val_u64 = static_cast<::uint64_t>(max_val);

    if(same_sign)
    {
        /* This part is quite simple, because comparisons
         *  with same sign is well-defined */
        if(from < min_val)
            return min_val;
        else if(from > max_val)
            return max_val;
        else
            return from;
    }else{
        /* Comparisons with different sign is literally iffy */
        if(from < 0)
            /* If `from' is signed and below 0,
             *  we know that `D' must be unsigned.
             *  We can only return the smallest value for `D'. */
            return min_val;
        else if(T_max_val_u64 > D_max_val_u64 &&
                static_cast<uint64_t>(from) > D_max_val_u64)
            /* Check that, if T has a larger range than D,
             *  that from is still within range.
             * If not, return max_val.
             *  At this point, we know that from is larger
             *  than or equal to 0, and it is safe to cast
             *  to the largest unsigned type. */
            return max_val;
        else
            return from;
    }
}
{{< / highlight >}}

The template is only intended for the cases where a conversion **might** fail. It will not intervene with upcasting `uint16_t` to `uint32_t` or `int16_t` to `int32_t`. One bad assumption is that `uint64_t` is the largest type in the case where a comparison has to be made.

[Here](https://godbolt.org/g/U1YVUq) is a quick test case running on Godbolt. On most architectures except ARM 32-bit, the function decomposes into less than 10 instructions.

Sadly, this piece of code does not work on MSVC. While it does work on all other compilers on Godbolt, MSVC refuses to compile it.

