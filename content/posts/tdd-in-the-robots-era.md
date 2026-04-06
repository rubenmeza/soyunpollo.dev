---
title: "TDD in the Robots Era"
date: 2026-04-06T12:00:00-06:00
tags: ["tdd", "testing", "ai", "automation", "software-engineering"]
categories: ["software-engineering"]
ShowToc: false
draft: false
---

Robots are writing more code every day. AI agents, code generators, automation tools — they all produce code faster than any human can. But faster does not mean correct.

When a robot writes code for you, how do you know it works? How do you know it handles edge cases? How do you know it doesn't break something that was working before?

This is where TDD becomes important again.

## The Red/Green Cycle as a Verification Tool

The classic TDD cycle is simple: write a failing test (red), make it pass (green), then refactor. This cycle was always useful, but now it has a new purpose — **it gives humans a way to verify what robots produce**.

Think about it this way: if you write the tests first, you define the rules. You decide what the code should do, what inputs it should handle, and what outputs are correct. Then you let the robot write the implementation. If the tests pass, you have confidence. If they don't, you know exactly where the problem is.

The human writes the specification. The robot writes the implementation. The tests are the contract between both.

## A Simple Example

Let's say you need a function that calculates a discount. You start by writing the tests:

```go
func TestApplyDiscount(t *testing.T) {
    tests := []struct {
        name     string
        price    float64
        percent  float64
        expected float64
    }{
        {"10 percent off", 100.0, 10, 90.0},
        {"zero discount", 50.0, 0, 50.0},
        {"full discount", 80.0, 100, 0.0},
        {"negative discount rejected", 100.0, -5, 100.0},
        {"over 100 percent rejected", 100.0, 150, 100.0},
    }

    for _, tc := range tests {
        t.Run(tc.name, func(t *testing.T) {
            got := ApplyDiscount(tc.price, tc.percent)
            if got != tc.expected {
                t.Errorf("ApplyDiscount(%.2f, %.2f) = %.2f, want %.2f",
                    tc.price, tc.percent, got, tc.expected)
            }
        })
    }
}
```

Now you give this to the robot with a prompt like:

> *Write the `ApplyDiscount` function in Go. It receives a price and a discount percentage. If the percentage is negative or greater than 100, return the original price. Run the tests to make sure they all pass.*

The robot generates the code. You run the tests. Green? Good. Red? You already know what failed and why.

You didn't need to read every line the robot wrote. The tests told you if the output is correct.

## Why This Matters Now

When you write code yourself, you have context. You know what you changed and why. When a robot writes code, you lose that context. Tests bring it back. They are your safety net — for regressions, for edge cases, for security rules.

TDD is not new. But in a world where robots write more and more of our code, the red/green cycle is one of the best tools we have to stay in control.

Write the tests. Let the robots do the rest.
