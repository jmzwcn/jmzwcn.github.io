---
tags: go
layout: post
title: '回应网友算法问题-go语言如何求取二维数组最长路径？'
category: Linux
---
在studygolang上看到网友提出的算法问题，觉得挺好，也算是一起学习，现给出我的一个实现。

<!--more-->

data.txt请直接参看原文吧（[http://studygolang.com/topics/1867](http://studygolang.com/topics/1867)）

`代码`：

```go
package main

import (
    "bufio"
    "fmt"
    "os"
    "strconv"
    "strings"
)

var tempArray, longestLink []int

func main() {
    inputFile, err := os.Open("data.txt")
    if err != nil {
        panic(err)
    }

    defer inputFile.Close()

    buf := bufio.NewReader(inputFile)
    i, j := 0, 0
    var data [][]int
    for {
        line, err := buf.ReadString('\n')
        line = strings.TrimSpace(line)
        a := strings.Fields(line)
        if len(a) > 0 {
            var row []int
            for j = 0; j < len(a); j++ {
                v, err := strconv.Atoi(a[j])
                if err != nil {
                    panic(err)
                }
                row = append(row, v)

            }
            data = append(data, row)
            i = i + 1
        }

        if err != nil {
            break
        }

    }
    fmt.Println(data)
    max, x, y := getMaxAndLocation(data)
    beginLink([]int{max}, x, y, j, i, data)
    fmt.Println(longestLink)
}

func getMaxAndLocation(values [][]int) (int, int, int) {
    max, x, y := 0, 0, 0
    for i, h := range values {
        for j, cell := range h {
            if cell > max {
                max = cell
                x = i
                y = j
            }
        }

    }
    return max, x, y
}

func beginLink(currentArray []int, x, y, r, c int, data [][]int) {
    compareAndPickLongest(currentArray)
    if (y-1) >= 0 && data[x][y-1] < data[x][y] {
        tempArray = append(currentArray, data[x][y-1])
        beginLink(tempArray, x, y-1, r, c, data)
    }

    if (y+1) < r && data[x][y+1] < data[x][y] {
        tempArray = append(currentArray, data[x][y+1])
        beginLink(tempArray, x, y+1, r, c, data)
    }

    if (x-1) >= 0 && data[x-1][y] < data[x][y] {
        tempArray = append(currentArray, data[x-1][y])
        beginLink(tempArray, x-1, y, r, c, data)
    }

    if (x+1) < c && data[x+1][y] < data[x][y] {
        tempArray = append(currentArray, data[x+1][y])
        beginLink(tempArray, x+1, y, r, c, data)
    }
}

func compareAndPickLongest(given []int) {
    if len(given) > len(longestLink) {
        longestLink = given
    }
}
```



---
