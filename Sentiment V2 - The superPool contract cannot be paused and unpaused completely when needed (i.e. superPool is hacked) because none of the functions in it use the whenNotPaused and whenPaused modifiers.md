# The superPool contract cannot be paused and unpaused completely when needed (i.e. superPool is hacked) because none of the functions in it use the whenNotPaused and whenPaused modifiers

## Summary

A summary with the following structure: **{root cause} will cause [a/an] {impact} for {affected party} as {actor} will {attack path}**

The `superPool` contract cannot be `paused` and `unpaused` completely when needed (i.e. `superPool` is hacked) because none of the functions in it use the `whenNotPaused` and `whenPaused` modifiers

## **Root Cause**


[`superPool`](https://github.com/sherlock-audit/2024-08-sentiment-v2-Emanueldlvg/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/SuperPool.sol#L25) contract inherits from `Pausable` , but none of the functions in it use the `whenNotPaused` and `whenPaused` modifiers


## **Impact**


The `superPool` contract cannot be `paused` and `unpaused` completely


## **Mitigation**

Consider adding `whenNotPaused` and `whenPaused` modifiers to critical functions (i.e `deposit`, `mint`, `withdraw`, `redeem`, and `reallocate`)
