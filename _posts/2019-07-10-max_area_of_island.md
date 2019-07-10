---
layout:     post
title:      岛屿最大面积问题(python&go分别实现)
date:       2019-07-10
author:     yandenghong
header-img: img/post-moon.png
catalog: true
tags:
    - 算法
    - Python
    - Golang
---
## 题目
给定一个包含了一些 0 和 1的非空二维数组 grid , 一个 岛屿 是由四个方向 (水平或垂直) 的 1 (代表土地) 构成的组合。你可以假设二维矩阵的四个边缘都被水包围着。

找到给定的二维数组中最大的岛屿面积。(如果没有岛屿，则返回面积为0。)

示例 1:

`
[[0,0,1,0,0,0,0,1,0,0,0,0,0],
 [0,0,0,0,0,0,0,1,1,1,0,0,0],
 [0,1,1,0,1,0,0,0,0,0,0,0,0],
 [0,1,0,0,1,1,0,0,1,0,1,0,0],
 [0,1,0,0,1,1,0,0,1,1,1,0,0],
 [0,0,0,0,0,0,0,0,0,0,1,0,0],
 [0,0,0,0,0,0,0,1,1,1,0,0,0],
 [0,0,0,0,0,0,0,1,1,0,0,0,0]]
`

> 对于上面这个给定矩阵应返回 6。注意答案不应该是11，因为岛屿只能包含水平或垂直的四个方向的‘1’。

## Python
```python
def max_area_of_island(grid):
    """
    :type grid: List[List[int]]
    :rtype: int
    """
    if not grid:
        return 0

    l, h = len(grid), len(grid[0])

    def dfs(i, j):
        if 0 <= i < l and 0 <= j < h and grid[i][j]:
            grid[i][j] = 0
            return 1 + dfs(i - 1, j) + dfs(i + 1, j) + dfs(i, j - 1) + dfs(i, j + 1)
        return 0

    result = [dfs(i, j) for i in range(l) for j in range(h) if grid[i][j]]
    return max(result) if result else 0
```

## Golang
```golang
package main

func maxAreaOfIsland(grid [][]int) int {
	maxArea := 0
    for i := range grid {
        for j := range grid[i] {
            area := dfs(i, j, grid)
            if area > maxArea {
                maxArea = area
            }
        }
    }
	return maxArea

}

func dfs(x int, y int, grid [][]int) int {
	if grid[x][y] == 0 {
        return 0
    }
    area := 1
    grid[x][y] = 0
    if x > 0 {
        area += dfs(x-1, y, grid)
    }
    if x < len(grid)-1 {
        area += dfs(x+1, y, grid)
    }
    if y > 0 {
        area += dfs(x, y-1, grid)
    }
    if y < len(grid[0])-1 {
        area += dfs(x, y+1, grid)
    }
    return area
}

func main() {
    a := make([][]int, 0)
	a = append(a, []int{0,0,1,0,0,0,0,1,0,0,0,0,0})
	a = append(a, []int{0,0,0,0,0,0,0,1,1,1,0,0,0})
	a = append(a, []int{0,1,1,0,1,0,0,0,0,0,0,0,0})
	a = append(a, []int{0,1,0,0,1,1,0,0,1,0,1,0,0})
	a = append(a, []int{0,1,0,0,1,1,0,0,1,1,1,0,0})
	a = append(a, []int{0,0,0,0,0,0,0,0,0,0,1,0,0})
	a = append(a, []int{0,0,0,0,0,0,0,1,1,1,0,0,0})
	a = append(a, []int{0,0,0,0,0,0,0,1,1,0,0,0,0})

	maxAreaOfIsland(a)
}
```
