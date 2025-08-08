---
theme: default
title: Differential Fuzzing on ⚡
info: |
  ## Differential Fuzzing on ⚡
class: text-center
drawings:
  persist: false
mdc: true
---

# Differential Fuzzing on ⚡
Finding Implementation Bugs Before Users Do

<!--
Hello, my name is Erick Cestari. 

I will talk about differential fuzzing on the Lightning Network - 
a technique to find bugs between different Lightning implementations.
-->

---

<img src="./alice_pay_bob.png" style="width: 800px; height: 500px; object-fit: contain; margin: 0 auto; display: block;" />

<!--
So imagine this scenario: Bob is trying to sell coffee to Alice. 
Bob is using NLightning to generate the invoice, and Alice is using Rust-Lightning to make the payment.
Bob generates and sends the invoice to Alice.
-->

---

<img src="./alice_fail_decode.png" style="width: 800px; height: 500px; object-fit: contain; margin: 0 auto; display: block;" />

<!--
Alice tries to pay the invoice using Rust-Lightning, but it fails to decode the invoice. 
She gets an error and can't complete the payment.
-->

---

<img src="./carol_pay_bob.png" style="width: 800px; height: 500px; object-fit: contain; margin: 0 auto; display: block;" />

<!--
Frustrated, Alice asks her friend Carol for help. 
Carol tries to pay the same invoice using her LND node, and this time it succeeds! 
Same invoice, different implementation, different result.
-->

---

# Implementation Differences

Different Lightning implementations can interpret the same data differently

Real-world impact:

* Poor user experience

Traditional approach:

* Wait for users to report bugs
* Manual testing between implementations
* Reactive fixes

<!--
Different Lightning implementations can interpret the same data differently. Real-world impact: Poor user experience where success depends on which implementation you're using. Traditional approach: Wait for users to report bugs, manual testing between implementations, reactive fixes.
-->
---
class: flex items-center justify-center text-center
---

# How do we systematically find these discrepancies before they cause problems?

<!--
How do we systematically find these discrepancies before they cause problems?
-->

---
class: flex items-center justify-center text-center
---

# Solution: Differential Fuzzing

<!--
So the solution is by doing differential fuzzing. The theme of this presentation :)
-->

---

# Who am I?

* Erick Cestari
* Vinteum Grantee (Bitcoin development funding)
* Maintainer of bitcoinfuzz - found 15+ bugs across Lightning implementations (we'll see some of them)
* **Why this matters:** I've seen firsthand how implementation differences create real problems

<!--
But first, Who am I?

My name is Erick Cestari.

I am a Vinteum grantee - that's Bitcoin development funding for those who aren't familiar.

I'm the maintainer of bitcoinfuzz, where I've found over 15 bugs across various Lightning implementations and reported some security disclosures.

Why does this matter for today's talk? I've seen firsthand how these implementation differences create real problems for users and the network. That's what motivated me to develop systematic approaches to find these issues before they impact real users.
-->
---
---
# Bitcoin: Code as Specification
If we built a new Bitcoin implementation today, where would we find the specification?

* Bitcoin Core codebase - the reference implementation
* No formal written specification document
* Consensus rules are implicit in the code
<!--
Bitcoin has an interesting characteristic - there's no formal specification document. 
If you want to build a new Bitcoin implementation today, you essentially have to reverse-engineer Bitcoin Core.
The consensus rules are implicit in the code, which means implementation differences can be catastrophic.
-->
---
---
# Lightning: Specification-First Approach

Lightning Network took a different approach with BOLT specifications

* BOLT = Basis of Lightning Technology
* Formal written specifications for all protocol aspects
* Multiple implementations can follow the same spec
* But... specifications can be ambiguous or incomplete

![Bolts github image](./bolts.png)

<!--
Lightning Network learned from Bitcoin's approach and took a specification-first approach.
BOLT stands for Basis of Lightning Technology - these are formal written specifications that cover all aspects of the Lightning protocol.
This allows multiple implementations to follow the same spec and theoretically be compatible.

But here's the catch - specifications can be ambiguous or incomplete. 
Even with a formal spec, different teams can interpret the same requirements differently.
This is where our differential fuzzing comes in - to find these interpretation differences systematically.
-->
---
---
## Edge Cases
BOLT specifications are comprehensive, but they can't cover every edge case.
When the spec says `r field should contain one or more entries` - 
what happens with zero entries? The spec doesn't explicitly say.

This is where implementations diverge:
- Some reject it (Rust-Lightning, Core Lightning)  
- Others accept it (LND, Eclair)

Differential fuzzing systematically explores these specification gaps.

<!--
So here's the core challenge with specifications - even when they're comprehensive like BOLT, they can't anticipate every possible edge case.

Take this real example from our fuzzing results: The BOLT11 specification says the 'r' field should contain "one or more entries" for routing information. Sounds clear, right?

But what happens when you encounter an 'r' field that exists but contains zero entries? The specification doesn't explicitly address this scenario.

And this is exactly where we see implementations diverge. Rust-Lightning and Core Lightning take a strict interpretation - they reject invoices with empty 'r' fields. Meanwhile, LND and Eclair are more permissive - they accept these invoices.

Neither approach is necessarily wrong - they're just different interpretations of an ambiguous specification.

This is the perfect example of why differential fuzzing is so valuable. Instead of waiting for users to discover these incompatibilities in production, we can systematically generate edge cases like this and find where implementations behave differently. This helps us identify specification gaps before they cause real-world payment failures.
-->
---
---
# So let's start simple, what is fuzzing?

- Fuzzing is a software testing technique.
- Generate thousands of inputs and feed them to a program.

What is the problem with this double function?

```rust
use libfuzzer_sys::fuzz_target;

fn double(x: i32) -> i32 {
    x * 2
}

fuzz_target!(|data: &[u8]| {
    if let Some(x) = consume_i32(data) {
        let _ = double(x);
    }
});
```
---
---

# Let's try now with double function fixed

```rust
use libfuzzer_sys::fuzz_target;

fn double(x: i32) -> Option<i32> {
    x.checked_mul(2)
}

fuzz_target!(|data: &[u8]| {
    if let Some(x) = consume_i32(data) {
        let _ = double(x);
    }
});
```
---
---

# Coverage-Guided Fuzzing

Smarter Fuzzing with Coverage
Instead of random inputs:

- Track which code paths are exercised
- Mutate inputs that discover new paths
- Focus on edge cases that trigger unusual behavior

Result: Find bugs faster with fewer test cases

---
---
## Coverage-Sanitizer

We use coverage-sanitizer to track which code paths are exercised.

It inserts calls to user-defined functions on function-, basic-block-, and edge- levels.

```c
#include <stdio.h>
#include <stdlib.h>

int calculate_grade(int score) {
    if (score >= 90) {
        return 'A';
    } else if (score >= 80) {
        return 'B';
    } else if (score >= 70) {
        return 'C';
    } else if (score >= 60) {
        return 'D';
    } else {
        return 'F';
    }
}
```
---
---
<img src="./no_sanitizer_coverage.png" style="width: 800px; height: 500px; object-fit: contain; margin: 0 auto; display: block;" />
---
---
<img src="./with_sanitizer_coverage.png" style="width: 1100px; height: 500px; object-fit: contain; margin: 0 auto; display: block;" />
---
---

# Differential Fuzzing

- Generate thousands of inputs and feed them simultaneously to **multiple implementations**.

```rust
use libfuzzer_sys::fuzz_target;

fn double(x: i32) -> Option<i32> {
    x.checked_mul(2)
}

fn double2(x: i32) -> Option<i32> {
    // Off-by-one: using >= instead of >
    if x >= i32::MAX / 2 || x <= i32::MIN / 2 {
        None
    } else {
        Some(x * 2)
    }
}

fuzz_target!(|data: &[u8]| {
    if let Some(x) = consume_i32(data) {
        let res = double(x);
        let res2 = double2(x);
        if res != res2 {panic!("x: {}, res: {:?}, res2: {:?}", x, res, res2);}
    }
});
```

---
---

<div style="height: 500px; overflow: hidden;">

<div style="transform: scale(0.7); transform-origin: top center; height: 125%;">

```mermaid
graph TB
    subgraph "Fuzzing Engine"
        A[Seed Corpus]
        B[Input Mutator]
        C[Coverage Tracker]
        D[Interesting Input Queue]
        E[Crash Detector]
    end
    
    A --> B
    B --> F[Mutated BOLT11 Invoice]
    F --> G[LND Parser]
    F --> H[Core Lightning Parser]
    F --> I[Rust-Lightning Parser]
    
    G --> J[Result + Coverage Info]
    H --> K[Result + Coverage Info]
    I --> L[Result + Coverage Info]
    
    J --> M{Compare Results}
    K --> M
    L --> M
    
    M -->|All Same| N{New Coverage?}
    M -->|Different| O[⚠️ Discrepancy Found]
    
    N -->|Yes| P[Add to Corpus]
    N -->|No| Q[Discard Input]
    
    O --> R[Log Bug + Save Reproducer]
    
    P --> D
    Q --> B
    D --> B
    R --> S[Investigate: Bug or Spec Ambiguity?]
    
    subgraph "Coverage Feedback Loop"
        C --> N
        J -.-> C
        K -.-> C
        L -.-> C
    end

```
</div>
</div>

---
---

## Bitcoinfuzz: Bitcoin Differential Fuzzing

**What we're building:**
* Differential Fuzzing framework for Bitcoin protocol implementations and libraries
* Focus: Find discrepancies before they cause issues

**Current targets (for lightning network):**

* modules: LND, Core Lightning, Rust-Lightning
* targets: deserialize_invoice, deserialize_offer

**Status:** 30 bugs found so far.

---
---
# What happens when implementations differ?

* Different parsing of the same BOLT11 invoice
* Different parsing of the same BOLT12 offer
* Different handling of malformed payment requests

The problem: These discrepancies create unpredictable behavior across the network

Example: Your wallet generates a valid invoice, but some nodes reject it while others accept it
---