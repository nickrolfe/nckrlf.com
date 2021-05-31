---
layout: post
title:  "How do C and C++ unit test frameworks find all the test functions in a program?"
date:   2018-02-02 20:30:00 +0000
---
I recently started a new programming job, and it's been a fascinating and exhausting experience diving into a huge, old codebase written in C. Some of the original authors are long-gone from the company, but I still feel like I'm getting to know them and their personalities by the code they left behind.

One of the components I inherited is a custom C unit-test framework. Sure, there are many off-the-shelf C frameworks for unit-testing, but ours supports one surprising feature: automatic registration/discovery of tests. You can define a test function anywhere, in any C file, and it *will* be found and run by the test runner.

I thought it was really exciting, so I took the time to figure out how it works, and I even managed to simplify it quite a bit. I'd like to share with you what I learned.

## Let's write a minimal test framework

We want our test functions to take zero arguments and return a boolean value indicating whether the test passed:

```c
typedef bool (*TestFunctionPtr)(void);
```

The test runner will need the name of each test and a pointer to the function to call:

```c
typedef struct TestInfo {
  const char     *name;
  TestFunctionPtr function;
} TestInfo;
typedef const TestInfo* TestInfoPtr;
```

Now let's create a couple of example tests:

```c
bool test_add_one(void) {
  // test the add_one() function here
  return true;
}

bool test_multiply(void) {
  // test the multiply() function here
  return true;
}

const TestInfo test_info_add_one = {
  "add_one",
  test_add_one
};
const TestInfo test_info_multiply = {
  "multiply",
  test_multiply
};
```

You'll notice there's quite a lot of boilerplate there. That's okay --- we could avoid all that code duplication with some macros. I won't show you the macros here, though, because C macros are as ugly as sin.

So far, our test runner has no way of finding `test_info_add_one` and `test_info_multiply`, and our tests won't run. We *could* create an array for our test runner to iterate over:

```c
TestInfoPtr all_tests[] = {
  &test_info_add_one,
  &test_info_multiply
};
```

But this is manual registration, and it's error prone, and it's exactly what we'd like to avoid! It would be really easy to write a test and then forget to update the `all_tests` array. I mean, I'm sure *you'd* never forget, but wouldn't it be nice to get rid of that tiny bit of mental overhead?

## How GoogleTest does it

A note from their [documentation][GoogleTestPrimer]:

> The testing framework should liberate test writers from housekeeping chores and let them focus on the test content. Google C++ Testing Framework automatically keeps track of all tests defined, and doesn't require the user to enumerate them in order to run them.

Sounds great to me! How do they do it? Why, it's perfectly simple:

```cpp
// Helper macro for defining tests.
#define GTEST_TEST_(test_case_name, test_name, parent_class, parent_id)\
class GTEST_TEST_CLASS_NAME_(test_case_name, test_name) : public parent_class {\
 public:\
  GTEST_TEST_CLASS_NAME_(test_case_name, test_name)() {}\
 private:\
  virtual void TestBody();\
  static ::testing::TestInfo* const test_info_ GTEST_ATTRIBUTE_UNUSED_;\
  GTEST_DISALLOW_COPY_AND_ASSIGN_(\
      GTEST_TEST_CLASS_NAME_(test_case_name, test_name));\
};\
\
::testing::TestInfo* const GTEST_TEST_CLASS_NAME_(test_case_name, test_name)\
  ::test_info_ =\
    ::testing::internal::MakeAndRegisterTestInfo(\
        #test_case_name, #test_name, NULL, NULL, \
        ::testing::internal::CodeLocation(__FILE__, __LINE__), \
        (parent_id), \
        parent_class::SetUpTestCase, \
        parent_class::TearDownTestCase, \
        new ::testing::internal::TestFactoryImpl<\
            GTEST_TEST_CLASS_NAME_(test_case_name, test_name)>);\
void GTEST_TEST_CLASS_NAME_(test_case_name, test_name)::TestBody()
```

I told you C macros were ugly. Also, this is actually C++, which is going to be an important distinction very shortly.

Most of the details here are not important. The interesting part is that it defines a global `TestInfo` variable, and initialises it by calling `MakeAndRegisterTestInfo`. That means the `MakeAndRegisterTestInfo` function will be called when the program starts, and it can add the test information to Google Test's equivalent of our `all_tests` array.

What Google have effectively done here is leave a note to the C++ compiler, asking really politely if it wouldn't mind registering the test for us, just before it drops us off in `main`. Sweet! That was easy.

## Getting help from the linker

Unfortunately, we can't use the same trick in C. The language just doesn't allow us to call functions to intialise variables.

To solve this, we have to use some non-standard, compiler-specific attributes, and ask the linker to give us a hand. First, a quick reminder about what the compiler does when we define a constant:

```c
const int MY_CONSTANT = 9000;
```

```terminal
$ gcc -S file.c
```

`MY_CONSTANT` will get placed in a section called `.rodata` (read-only data):

```
    .section .rodata
    .align 4
    .type   MY_CONSTANT, @object
    .size   MY_CONSTANT, 4
MY_CONSTANT:
    .long   9000
```

We're not limited to putting it in `.rodata`, though. If we're willing to write a bit of compiler-specific code, we can create our own section:

```c
// Not standard C! Needs gcc on Linux.
const int MY_CONSTANT __attribute__((section ("my_section"))) = 9000;
```

```
    .section    my_section,"a",@progbits
    .align 4
    .type   MY_CONSTANT, @object
    .size   MY_CONSTANT, 4
MY_CONSTANT:
    .long   9000
```

Nice! What's even cooler is that, since the section name `my_section` is also a valid C identifier, we can write C code assuming the following symbols exist, and the gcc linker will make sure they are defined for us!

```c
extern int __start_my_section;
extern int __stop_my_section;
```

Now we can take the address of these symbols: `&__start_my_section` will be the address of the first byte of `my_section`, and `&__stop_my_section` points to one byte after the end of `my_section`.

This is wonderful news for two reasons:

1. We can put constants in `my_section` from any C file.
2. We can write a loop that iterates over everything in `my_section`, without any knowledge of what might end up in it.

Let's write that for loop:

```c
#include <stdio.h>

const int a __attribute__((section ("my_section"))) = 100;
const int b __attribute__((section ("my_section"))) = 200;
const int c __attribute__((section ("my_section"))) = 300;
const int d __attribute__((section ("my_section"))) = 400;

extern int __start_my_section;
extern int __stop_my_section;

int main() {
  for (int *cur_val = &__start_my_section;
            cur_val < &__stop_my_section;
            cur_val++) {
    printf("%d ", *cur_val);
  }
  printf("\n");
}
```

If we compile and run this, it prints:

```
400 300 200 100
```

Wow --- nothing in the code directly referenced `a`, `b`, `c`, or `d`, but we still managed to find all of them and print their values!

They didn't come out in the order I expected, but that's okay --- I think it's perfectly acceptable for a unit test runner to run the tests in any order. I don't really know for sure, but I suspect the C standard doesn't have an opinion on the ordering of those constants in memory, and it's up to the compiler and linker to do whatever they want here.

Also, a warning: in some situations the compiler might decide to be helpful and optimise the variables away completely, since nothing seems to be using them. In that case, you can tell the compiler that you really know best with the `used` [attribute][GCCAttributes].

## Putting it all together

Now we can take what we learned about custom sections and apply it to our `TestInfo` structures. I've decided to place them in a section called `test_section`.

```c
const TestInfo test_info_add_one __attribute__((section ("test_section"))) = {
  "add_one",
  test_add_one
};
const TestInfo test_info_multiply __attribute__((section ("test_section"))) = {
  "multiply",
  test_multiply
};
```

And we can adapt our `for` loop above into a test runner:

```c
extern TestInfo __start_test_section;
extern TestInfo __stop_test_section;

void run_all_tests(void) {
  int num_passes   = 0;
  int num_failures = 0;

  for (TestInfo *cur_test = &__start_test_section;
                 cur_test < &__stop_test_section;
                 cur_test++) {
    printf("Calling test '%s'... ", cur_test->name);
    bool test_passed = cur_test->function();
    if (test_passed) {
      num_passes++;
      printf("passed\n");
    } else {
      num_failures++;
      printf("FAILED :(\n");
    }
  }
  printf("%d failures, %d passes\n", num_failures, num_passes);
}
```

Let's compile and see what happens:

```terminal
$ gcc -o tests tests.c
$ ./tests
Calling test 'add_one'... passed
Calling test 'multiply'... passed
0 failures, 2 passes
```

We did it! Now we can start writing real tests, and, as long as we remember to create a `TestInfo` with the right section attribute --- which we'll totally automate with an ugly C macro, right? --- our runner will automatically find and run them for us.

## Bonus points: clang on macOS

I mentioned that the custom-section attributes were specific to gcc on Linux. On macOS with clang we have to make a slight adjustment to the section attribute:

```c
const TestInfo test_info_add_one __attribute__((section ("__DATA,test_section"))) = {
...
```

Unfortunately, the clang linker doesn't seem to provide the `__start` and `__stop` symbols for us automatically, but, with a little sprinkling of inline assember, we can persuade it to do what we want:

```c
extern TestInfo __start_test_section __asm("section$start$__DATA$test_section");
extern TestInfo __stop_test_section __asm("section$end$__DATA$test_section");
```

Otherwise, everything is the same as on Linux.

## Double bonus points: MSVC on Windows

Windows is the last OS still standing that *isn't* a Unix clone, so it sometimes gets a bit neglected by the developer community. That's a shame, because it has by far the most usable debugger on any platform, not to mention billions of users. Let's show it a little love.

Here's how we tell MSVC to put our `TestInfo` structures in a custom `tests` section (we're limited to 8 characters, so we can't call it `test_section`):

```c
#pragma section("tests", read)
__declspec(allocate("tests")) const TestInfo test_info_add_one = {
...
```
Like with clang on macOS, the MSVC linker won't automatically create `__start` and `__stop` symbols for us, but there's a funny little workaround. The PE/COFF object file format says the `$` character is interpreted specially in section names: the linker discards it and any characters following it. This means `tests`, `tests$a`, and `tests$x` would all be linked into the same `tests` section. However, the characters after the dollar sign *do* contribute to the (lexical) ordering, meaning that everything in the `tests$a` section is guaranteed to be linked before everything in the `tests$x` section.

Now we can come up with a plan:

* Create a dummy 'start' variable --- call it `__start_test_section`, for consistency --- and place it in `tests$a`.
* Place all our real `TestInfo` structures in `tests$b`.
* Create a dummy 'stop' variable --- `__stop_test_section` --- and place it in `tests$c`.

The dummy variables could be any type, but making them `TypeInfo` means we don't have to cast them or worry about any alignment issues; the address of the first *real* `TestInfo` will be `sizeof(TestInfo)` bytes after `__start_test_section`.

Here's what our plan looks like in action:

```c
#pragma section("tests$a", read) // for the start symbol
#pragma section("tests$b", read) // for the real TestInfo structures
#pragma section("tests$c", read) // for the stop symbol

__declspec(allocate("tests$b"))
const TestInfo test_info_add_one = {
  "add_one",
  test_add_one
};

__declspec(allocate("tests$b"))
const TestInfo test_info_multiply = {
  "multiply",
  test_multiply
};

__declspec(allocate("tests$a"))
TestInfo __start_test_section;
__declspec(allocate("tests$c"))
TestInfo __stop_test_section;
```

Since we need to skip over the `__start_test_section` structure, we need to make a small change to our test runner:

```c
for (TestInfo *cur_test = &__start_test_section + 1; // + 1 to skip the dummy var
               cur_test < &__stop_test_section;
               cur_test++) {
...
```

## Summary

How cool is that? We now have automatic test discovery working on all three major desktop platforms.

Here's what I learned:

* C++ makes life easy, since you can call functions in variable initialisers. You can initialise each `TestInfo` by calling a function that adds the corresponding test to a list.
* In C, it's only possible to do automatic test discovery --- as far as I know --- with platform-specific extensions.
* Compilers let us control which section our `TestInfo` structures get placed in with some special attributes.
* Linkers will find and group together all variables that were placed in a particular section.
* Linkers can also provide us with symbols to mark the start and end of that section, so we know exactly where it put all the `TestInfo` structures.
* Platform-specific and other gory implementation details can be hidden away with some carefully-crafted macros. Have a look at an example implementation including macros [on GitHub](https://gist.github.com/nickrolfe/ffc9b1c02381b9dc17c975b98db42172) to see what I mean. There's some really hairy code in there --- but the test functions themselves are squeaky clean.

Thanks for reading!

[GCCAttributes]: https://gcc.gnu.org/onlinedocs/gcc/Common-Variable-Attributes.html
[GoogleTestPrimer]: https://github.com/google/googletest/blob/master/googletest/docs/Primer.md
