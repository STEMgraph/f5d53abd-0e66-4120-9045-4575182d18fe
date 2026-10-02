<!---
{
  "id": "f5d53abd-0e66-4120-9045-4575182d18fe",
  "teaches": "GNU Make: Debug and Release Configurations",
  "depends_on": [
    "7507db49-63c1-4b1f-865c-a722eb2f164e"
  ],
  "author": "Stephan Bökelmann",
  "first_used": "2026-10-02",
  "keywords": [
    "make",
    "debug",
    "release",
    "target-specific variables",
    "ifeq",
    "BUILD",
    "CFLAGS",
    "configuration"
  ]
}
--->

# GNU Make: Debug and Release Configurations

> In this exercise you will give your `Makefile` two configurations, `debug` and `release`, selected by a variable or by target, each with its own flags and its own build directory, so that `make debug` and `make release` never overwrite each other's objects. No `sudo` is required.

## Introduction

A debug build wants `-g -O0` and maybe `-fsanitize=address`; a release build wants `-O2 -DNDEBUG`. Switching flags in a single build directory is a trap: `make` only looks at timestamps, not at flags, so after `make CFLAGS=-g` a following `make CFLAGS=-O2` happily reports "up to date" and ships a debug binary.

`make` offers two mechanisms for configurations:

- **Conditionals** such as `ifeq ($(BUILD_TYPE),debug)`, evaluated when the `Makefile` is read.
- **Target-specific variables**: `debug: CFLAGS += -g -O0` sets `CFLAGS` only while `debug` and everything it depends on is being built.

Both are combined with a per-configuration build directory from the previous exercise, so that objects of different configurations never share a path.

### 1.1) Further Readings and Other Sources

* [GNU Make Manual: Conditional Parts of Makefiles](https://www.gnu.org/software/make/manual/html_node/Conditionals.html)
* [GNU Make Manual: Target-specific Variable Values](https://www.gnu.org/software/make/manual/html_node/Target_002dspecific.html)
* [GNU Make Manual: Overriding Variables (`override`)](https://www.gnu.org/software/make/manual/html_node/Override-Directive.html)

## Tasks

### Task 1: Demonstrate the Trap

In `~/make-intro` with the out-of-tree `Makefile`:

```bash
make clean
make CFLAGS="-g -O0 -MMD -MP"
make CFLAGS="-O2 -MMD -MP"
file build/main_program | tr ',' '\n' | grep -i debug
```

The second `make` did nothing, the binary still contains debug info.

### Task 2: A Configuration Variable With a Conditional

```make
CC      = gcc
BUILD_TYPE ?= release
BUILD   = build/$(BUILD_TYPE)
SRCDIR  = src

CFLAGS  = -Wall -Wextra -MMD -MP
ifeq ($(BUILD_TYPE),debug)
  CFLAGS += -g -O0 -DDEBUG
else ifeq ($(BUILD_TYPE),release)
  CFLAGS += -O2 -DNDEBUG
else
  $(error unknown BUILD_TYPE '$(BUILD_TYPE)', use debug or release)
endif
```

Keep the rest of the `Makefile` from the previous exercise (`SRCS`, `OBJS`, the pattern rule with `$(dir $@)`, `clean`, `-include`). Then:

```bash
make clean
make
make BUILD_TYPE=debug
make BUILD_TYPE=profile
ls build
```

Note that `ifeq` lines start in column one, not with a tab.

### Task 3: Make the Program Notice

```c
// at the top of src/main.c
#include <stdio.h>
#ifdef DEBUG
#define LOG(msg) fprintf(stderr, "[debug] %s\n", msg)
#else
#define LOG(msg)
#endif
```

Add `LOG("start");` as the first statement in `main`, then:

```bash
make && ./build/release/main_program
make BUILD_TYPE=debug && ./build/debug/main_program
```

### Task 4: Phony Targets as Shortcuts

Typing `BUILD_TYPE=debug` is tedious. Add two targets that re-invoke `make` with the variable set:

```make
debug:
	$(MAKE) BUILD_TYPE=debug

release:
	$(MAKE) BUILD_TYPE=release

.PHONY: debug release
```

```bash
make debug
make release
```

`$(MAKE)` instead of `make` passes flags such as `-j` and `-n` through to the sub-invocation. Check with `make -n debug`.

### Task 5: Target-Specific Variables Instead of Recursion

A variant without a second `make` process:

```make
debug:   BUILD_TYPE = debug
debug:   CFLAGS += -g -O0 -DDEBUG
release: CFLAGS += -O2 -DNDEBUG
debug release: $(BUILD)/main_program
```

Try it. It does **not** work as expected: `$(BUILD)` was already expanded when the prerequisite list was read, before the target-specific `BUILD_TYPE` took effect. Run `make -n debug` and look at the path. This is the classic limitation of target-specific variables: they apply to recipes, not to prerequisite lists. Go back to the recursive variant from Task 4, or keep the conditional from Task 2.

### Task 6: Sanitizers in Debug Only

```make
ifeq ($(BUILD_TYPE),debug)
  CFLAGS  += -g -O0 -DDEBUG -fsanitize=address
  LDFLAGS += -fsanitize=address
endif
```

Add `$(LDFLAGS)` to the link line. Introduce a deliberate out-of-bounds write in `src/util.c`, build both configurations and run both binaries. Which one reports the bug, and where?

### Task 7: Show the Configuration

```make
info:
	@echo BUILD_TYPE = $(BUILD_TYPE)
	@echo BUILD      = $(BUILD)
	@echo CFLAGS     = $(CFLAGS)
.PHONY: info
```

```bash
make info
make info BUILD_TYPE=debug
BUILD_TYPE=debug make info
```

Why do the last two print the same thing, and what would change if `BUILD_TYPE` were assigned with `=` instead of `?=`?

## Questions

- Why does `make` not notice that `CFLAGS` changed between two runs? What would it have to store to notice?
- `ifeq` versus a target-specific variable: which one can influence the prerequisite list, and why?
- What is the difference between `make debug` recursing with `$(MAKE)` and a user typing `make BUILD_TYPE=debug` directly?
- What does `$(error ...)` do, and when is it evaluated?
- `-DNDEBUG` disables `assert()`. Name one reason to keep assertions in release builds and one reason to remove them.
- How does CMake express the same idea? Look up `CMAKE_BUILD_TYPE`.

## Advice

The rule that keeps you out of trouble: **one configuration, one build directory**. Flags are not part of what `make` tracks, so they must be part of the path. With `BUILD = build/$(BUILD_TYPE)` you get that for free, and `make clean` still removes everything.

Task 5 is worth remembering as a limitation rather than a bug. Target-specific variables are useful for small things such as adding `-g` to a single object, but they cannot change *what* gets built, only *how*.
