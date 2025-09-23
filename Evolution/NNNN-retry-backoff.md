# Retry & Backoff

* Proposal: [NNNN](NNNN-retry-backoff.md)
* Authors: [Philipp Gabriel](https://github.com/ph1ps)
* Review Manager: TBD
* Status: **Implemented**

## Introduction

This proposal introduces a `retry` function and a suite of backoff strategies to Swift Async Algorithms, enabling robust retry of failed asynchronous operations with customizable delays and error-driven retry decisions.

Swift forums thread: [Discussion thread topic for that proposal](https://forums.swift.org/)

## Motivation

Retry logic with backoff is a common requirement in asynchronous programming, especially for operations subject to transient failures such as network requests. Today, developers must reimplement retry loops manually, leading to fragmented and error-prone solutions across the ecosystem.  

Providing a standard `retry` function and reusable backoff strategies in Swift Async Algorithms ensures consistent, safe, and well-tested patterns for handling transient failures.

## Proposed solution

This proposal introduces a retry function that executes an async operation up to a specified number of attempts, with customizable delays and error-based retry decisions between attempts.

```swift
public func retry<Result, ErrorType, ClockType>(
  maxAttempts: Int,
  tolerance: ClockType.Instant.Duration? = nil,
  clock: ClockType = ContinuousClock(),
  isolation: isolated (any Actor)? = #isolation,
  operation: () async throws(ErrorType) -> Result,
  strategy: (ErrorType) -> RetryAction<ClockType.Instant.Duration> = { _ in .backoff(.zero) }
) async throws -> Result where ClockType: Clock, ErrorType: Error

public enum RetryAction<Duration: DurationProtocol> {
  case backoff(Duration)
  case stop
}
```

Additionally, this proposal includes a family of backoff strategies that can be used to generate delays between retry attempts. The core strategies provide different patterns for calculating delays: constant intervals, linear growth, exponential growth, and decorrelated jitter.

```swift
public enum Backoff {
  public static func constant<Duration: DurationProtocol>(_ constant: Duration) -> some BackoffStrategy<Duration>
  public static func constant(_ constant: Duration) -> some BackoffStrategy<Duration>
  public static func linear<Duration: DurationProtocol>(increment: Duration, initial: Duration) -> some BackoffStrategy<Duration>
  public static func linear(increment: Duration, initial: Duration) -> some BackoffStrategy<Duration>
  public static func exponential<Duration: DurationProtocol>(factor: Int, initial: Duration) -> some BackoffStrategy<Duration>
  public static func exponential(factor: Int, initial: Duration) -> some BackoffStrategy<Duration>
  public static func decorrelatedJitter<RNG: RandomNumberGenerator>(factor: Int, base: Duration, using generator: RNG = SystemRandomNumberGenerator()) -> some BackoffStrategy<Duration>
}
```

These strategies can be modified to enforce minimum or maximum delays, or to add jitter for preventing the thundering herd problem.

```swift
extension BackoffStrategy {
  public func minimum(_ minimum: Duration) -> some BackoffStrategy<Duration>
  public func maximum(_ maximum: Duration) -> some BackoffStrategy<Duration>
  public func fullJitter<RNG: RandomNumberGenerator>(using generator: RNG = SystemRandomNumberGenerator()) -> some BackoffStrategy<Duration>
  public func equalJitter<RNG: RandomNumberGenerator>(using generator: RNG = SystemRandomNumberGenerator()) -> some BackoffStrategy<Duration>
}
```

## Detailed design

### Retry

The retry algorithm follows this sequence:
1. Execute the operation
2. If successful, return the result
3. If failed and this was not the final attempt:
- Call the `strategy` closure with the error
  - If strategy returns `.stop`, rethrow the error immediately
  - If strategy returns `.backoff`, suspend for the given duration
    - Return to step 1
4. If failed on the final attempt, rethrow the error without consulting the strategy

Given this sequence, there is a total of four termination conditions:
1. **Success**: The operation completes without throwing an error
2. **Maximum attempts exhausted**: The operation has been attempted `maxAttempts` times
3. **Strategy decision to stop**: The strategy closure returns `.stop`
3. **Clock throws**: The given clock throws, which will be rethrown

#### Cancellation

`retry` itself does not introduce any specific cancellation handling. If asynchronous code opts into cooperative cancellation by throwing an error, it has to make sure it handles this case in the retry strategy, by returning `.stop`, as this is a non-retryable error, usually. 
If you forget to do this, retrying will not be stopped, except when the given clock does cancel cooperatively by throwing (which at the time of writing both `ContinuousClock` and `SuspendingClock` do).

### Backoff

All strategies conform to:
```swift
public protocol BackoffStrategy<Duration> {
  associatedtype Duration: DurationProtocol
  mutating func nextDuration() -> Duration
}
```
Each call to nextDuration() returns the delay for the next retry attempt. Strategies are stateful - they may track the number of invocations or the previously returned duration to calculate the next delay.

#### Constant
Formula: $`f(n) = constant`$
#### Linear
Formula: $`f(n) = initial + increment * n`$
#### Exponential
Formula: $`f(n) = initial * factor ^ n`$
#### Decorrelated Jitter
Formula: $`f(n) = random(base, f(n - 1) * factor)`$, $`f(0) = base`$
#### Minimum
Formula: $`f(n) = max(minimum, g(n))`$, `g(n)` is the base strategy.
#### Maximum
Formula: $`f(n) = min(maximum, g(n))`$, `g(n)` is the base strategy.
#### Full Jitter
Formula: $`f(n) = random(0, g(n))`$, `g(n)` is the base strategy.
#### Equal Jitter
Formula: $`f(n) = random(g(n) / 2, g(n))`$, `g(n)` is the base strategy.

## Effect on API resilience

This proposal introduces purely additive API with no impact on existing functionality or API resilience.

## Alternatives considered

Describe alternative approaches to addressing the same problem, and
why you chose this approach instead.

## Acknowledgments

Thanks to [Philippe Hausler](https://github.com/phausler), [Franz Busch](https://github.com/FranzBusch) and [Honza Dvorsky](https://github.com/czechboy0) for their thoughtful feedback and suggestions that helped refine the API design and improve its clarity and usability.
