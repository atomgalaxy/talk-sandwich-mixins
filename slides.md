---
title: Sandwich Mixins
slideNumber: true
controls: true
theme: black

---

\newcommand{\@@}{\cdot}
\newcommand{\true}{\text{true}}
\newcommand{\false}{\text{false}}

\newcommand{\iff}{\Leftrightarrow}
\newcommand{\imp}{\Rightarrow}
\newcommand{\Z}{\mathbb{Z}}
\newcommand{\N}{\mathbb{N}}
\newcommand{\Q}{\mathbb{Q}}
\newcommand{\R}{\mathbb{R}}
\newcommand{\C}{\mathbb{C}}


# Sandwich Mixins {bg=#fff}

Gašper Ažman

August 3rd, 2019

<img src='assets/cpplondon-logo-light.svg' style='height: 3ex'>


# This is a mixin

```cpp
template <typename Base>
struct my_mixin : Base /* additional bases */ {
    // stuff you want in your interface
};
```


# Example

```cpp
template <typename Base>
struct stdout_logger : Base {
    template <typename... Ts>
    void log(Ts&&... xs) const { (std::cout << ... << xs); }
};
```


# Using it - naiively

```cpp
struct my_class : stdout_logger<my_class> {
    void foo() { log("whee"); }
};
```


# Using it from another mixin

```cpp
template <typename Base>
struct frobnicator : Base {
    void frobnicate() const {
        this->log("frobnicated.");
    }
};
```

# But this only works one way!

## Works
```cpp
struct concrete_1 : frobincator<stdout_logger<concrete_1>> {};
```

## Does not work
```cpp
// log() not a member of Base!
struct concrete_2 : stdout_logger<frobnicator<concrete_2>> {};
```


# Sanwich Mixins

Top slice:

```cpp
template <typename MostDerived>
struct top { /* not a mixin */
    using self_type = MostDerived;
    decltype(auto) self() &       { return static_cast<self_type&>(*this);        }
    decltype(auto) self() &&      { return static_cast<self_type&&>(*this);       }
    decltype(auto) self() const&  { return static_cast<self_type const&>(*this);  }
    decltype(auto) self() const&& { return static_cast<self_type const&&>(*this); }
};
```

# Changing `frobnicator`

From:
```cpp
template <typename Base>
struct frobnicator : Base {
    void frobnicate() const {       this->log("frobnicated."); }
};
```

To:
```cpp
template <typename Base>
struct frobnicator : Base {
    using Base::self;
    void frobnicate() const { this->self().log("frobnicated."); }
};
```

# Bottom slice

Works!
```cpp
struct concrete : 
    stdout_logger<
        frobnicator<
            top<concrete>
        >
    >
{
};
```

# Initial lesson

## Sandwiching

Classes composed of mixins need a bottom slice and a top slice.

## The topping interface

Toppings (mixins) use `self()` to get at the functionality of the whole class.


# Packaging

Because writing `top<concrete>` is annoying, and all those `<>` are weird.

We want this:

```cpp
struct concrete : mixin<concrete, frobnicator, stdout_logger> {
};
```

We'll get there, but I saved it for last. It's metaprogramming.

# Initialization

Initializing them is a bit more difficult. Let's add another implementation of
logger.

```cpp
template <typename Base>
struct ostream_logger : Base {
    ostream_logger(std::ostream& out_)
        , _out(&out_) {}

    template <typename... Ts>
    void log(Ts&&... xs) const { ((*_out) << ... << xs); }
  private:
    std::ostream* _out;
};
```

# Ideally, we'd initialize like this:

```cpp
struct concrete2 : mixin<concrete2, ostream_logger, frobnicator> {
    concrete2(std::ostream& out_) : ostream_logger(out_) {}
};
```

It does not work. Works with multiple inheritance, though.

# Diversion: Why chain inheritance?

In other words, contrast with:
```cpp
struct concrete : frobnicator<concrete>, stdout_logger<concrete>
```

Where `frobnicator` and `stdout_logger` *upcast* using CRTP to get at each other:
```cpp
template <typename CRTP>
struct frobnicator {
    void frobnicate() const { static_cast<CRTP const&>(*this).log("frobnicate"); }
};
```

# Problems:
## Casts everywhere:
```cpp
    void frobnicate() const { static_cast<CRTP const&>(*this).log("frobnicate"); }
```

## Sometimes, bigger size.
Every subobject must have its own address, even if empty, if it has a vtable,
in multiple inheritance. Single inheritance chains get away with a single
vtable pointer.

# Initialization, part 2:

Let's say we'd rather see this:
```cpp
struct concrete2 : mixin<concrete2, ostream_logger, frobnicator> {
    concrete2(std::ostream& out_) : mixin(out_) {}
};
```

We need to make our mixins forward constructor arguments:

# Changes to Logger

```cpp
#define FWD(name) std::forward<decltype(name)>(name)
template <typename Base>
struct ostream_logger : Base {
    ostream_logger(std::ostream& out_, auto&&... rest)
        : Base(FWD(rest)...)
        , _out(&out_) {}

    template <typename... Ts>
    void log(Ts&&... xs) const { ((*_out) << ... << xs); }
  private:
    std::ostream* _out;
};
```

# Changes to `frobnicator`

```cpp
template <typename Base>
struct frobnicator : Base {
    frobnicator(auto&&... rest) : Base(FWD(rest)...) {}

    void frobnicate() const { this->self().log("frobnicate."); }
};
```

# Let's add a third class, `echoer`

```cpp
template <typename Base>
struct echoer : Base {
    echoer(std::string prefix_, auto&&... rest) 
        : Base(FwD(rest)...)
        ,_prefix(std::move(prefix_)) { }
    std::string echo(std::string arg) { return _prefix + arg; }
    private:
    std::string _prefix;
};
```

# So, ideally, this would work:

```cpp
struct concrete2 : mixin<concrete2, ostream_logger, frobnicator, echoer> {
    concrete2(std::ostream& out_) : mixin(/*for ostream_logger*/out_,
                                          /*for echoer*/"my prefix") {}
};
```

And, it does! With some help from our friend `mixin`.


# The Butter: `mixin`

```
template <typename Concrete, template <class> class... Mixins>
struct mixin 
    : mixin_impl<Concrete, Mixins...> // fancy magic #1
{
    // fancy magic #2
    mixin(auto&&... rest) : mixin_impl<Concrete, Mixins...>(FWD(rest)...) {}

    // rule of 7
    mixin() = default;
    mixin(mixin const&) = default;
    mixin(mixin&&) = default;
    mixin& operator=(mixin const&) = default;
    mixin& operator=(mixin&&) = default;
    ~mixin() = default;
};
```
`mixin` allows us to just spell `mixin` instead of the magic weird name that
the nested composition of classes generated.


# `mixin_impl`

Just chain-inherits from `top<Concrete>` and the rest of the mixins.
```cpp
template <typename Concrete, template <class> class... Mixins>
using mixin_impl = typename chain_inherit<top<Concrete>, Mixins...>::type;
```

# `chain_inherit`
This is basically just a right fold.
```cpp
template <typename Concrete, template <class> class H, template <class> class... Tail>
struct chain_inherit {
    using result = typename chain_inherit<Concrete, Tail...>::type;
    using type = H<result>;
};
// base-case
template <typename Concrete, template <class> class H>
struct chain_inherit<Concrete, H> {
    using type = H<Concrete>;
};

```

# END

That's it! Questions?
