---
author: "Christos Panagiotakopoulos"
title: "How to make a simple chat app in C (Part 1)"
date: "2020-01-23"
description: "A detailed tutorial on how to make your on chat app using sockets in C."
tags: ["c", "sockets", "tutorial"]
categories: ["programming"]
draft: true
---

## Introduction



```c
#include <stdio.h>

int main(int argc, char** argv) {
    
    return 0;
}
```

### Arguments parsing

As we described before, our chat application needs to behave either as a server or as a client. This 

```c

int main(int argc, char** argv) {
    char* type = argv[1];

    return 0;
}

```
