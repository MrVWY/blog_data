---
title: "Disjoint_set"
date: "2021-02-02 23:56:57"
categories:
  - LeetCode
tags:
  - LeetCode
toc: true
math: true
type: about
---

### template

```go
type unionFind struct {
    parent, rank []int
}

func newunionFind(n int) *unionFind {
    parent := make([]int, n)
    rank := make([]int, n)
    for i := range parent {
        parent[i] = i
        rank[i] = 1
    }
    return &unionFind{parent, rank}
}

func (uf *unionFind) find(x int) int {
    if uf.parent[x] != x {
        uf.parent[x] = uf.find(uf.parent[x])
    }
    return uf.parent[x]
}

func (uf *unionFind) union(x, y int) bool {
    fx, fy := uf.find(x), uf.find(y)
    if fx == fy {
        return false
    }
    if uf.rank[fx] < uf.rank[fy] {
        fx, fy = fy, fx
    }
    uf.parent[fy] = fx
    uf.rank[fx] += uf.rank[fy]
    return true
}
```

### Reference

1. [维基百科](https://zh.wikipedia.org/wiki/%E5%B9%B6%E6%9F%A5%E9%9B%86)
2. [OI Wiki - 数据结构 - 并查集](https://oi-wiki.org/ds/dsu/#_6)