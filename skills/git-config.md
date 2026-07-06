---
name: git config
description: Standardize git config
license: MIT
author: Raymond Andreassen
version: "1.0.0"
source:
---

# UiT git guidance
(Akkurat begynt)

## Code quality
- Use Git integration in Visual Studio, with small commits and pull requests.

## .git/config example
- Cleanup branches when remotes are deleted
  ``` 
  [fetch]
  	prune = true
  ```
