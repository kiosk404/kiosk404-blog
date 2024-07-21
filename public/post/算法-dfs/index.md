# 算法 - (深度优先搜索)


Depth-First Search，也就是DFS算法，一般可以用来遍历或者搜索树或图。基本思想是假设走一条只有一个出口，但是路上可能会有很多分叉口的路，从头开始，每次遇到一个分叉口就随机选择一条路，当该条路走不通时，就回到上个分叉口重新选择。直到遍历到正确的路线。

<!--more-->

## Depth-First Search 

其基本定式一般使用递归的方式解决该问题。

<br/>

### No.51 N皇后

- 链接：[https://leetcode.cn/problems/n-queens](https://leetcode.cn/problems/n-queens/)
- 思路：回溯的解决方法

```go
var solutions [][]string

func solveNQueens(n int) [][]string {
	solutions = [][]string{}
	col := make(map[int]bool, n)  // 标记不能放在该列
	s1 := make(map[int]bool, n)   // 标记不能放在对角1
	s2 := make(map[int]bool, n)   // 标记不能放在对角2

	// queues 用来记录每一层的Q的位置
	queues := make([]int, n)
	for i := 0; i <= n; i ++ {
		queues[i] = -1
	}
	// 回溯
	backtrack(queues, n, 0, col, s1, s2)

	return solutions
}

func backtrack(queues []int, n int, row int, col, s1, s2 map[int]bool) {
	// 遍历到第N行了开始打印
	if row == n {
		board := generateBoard(queues, n)
		solutions = append(solutions, board)
	}

	// 循环列
	for i := 0; i < n ; i++ {
		if col[i] {     // 第i列不行
 			continue
		}
		if s1[row+i] {  // 对角1不行
			continue
		}
		if s2[row-i] {  // 对角2不行
			continue
		}

		col[i] = true    // 这个i可以，并将字典进行标记
		s1[row+i] = true
		s2[row-i] = true
		queues[row] = i
		backtrack(queues, n, row + 1, col, s1, s2)  // 回溯
		queues[row] = -1
		delete(col, i)
		delete(s1, row+1)
		delete(s2, row-1)
	}
}

func generateBoard(queens []int, n int) []string {
	var board []string
	for i := 0; i < n; i++ {
		row := make([]byte, n)
		for j := 0; j < n; j++ {
			row[j] = '.'
		}
		row[queens[i]] = 'Q'
		board = append(board, string(row))
	}
	return board
}
```





<br/>

岛屿问题属于图的搜索问题，一般使用 DFS，BFS或者 UF 来解决。

判断搜索的起始点，从某一点开始尝试所有可能，对每个点向四周搜索，上下左右。

### No.200 岛屿数量

- 链接：[https://leetcode.cn/problems/number-of-islands/](https://leetcode.cn/problems/number-of-islands/)
- 思路：遍历到一个点，从这个点开始进行 DFS。遍历网格，每遇到一块陆地('1')，就从它开始进行 DFS，同时将它标记为**“已访问”**。然后，依次进入到它的上、下、左、右邻居，同样进行 DFS，每次 DFS 也同样将当前节点进行标记。

```go
func numIslands(grid [][]byte) int {
	var count int
    if len(grid) == 0{
        return count
    }
    rows := len(grid)
    cols := len(grid[0])

    for i := 0; i < rows; i++{
        for j := 0; j < cols; j++{
            if grid[i][j] == '1'{
                dfs(grid, i, j, rows, cols)
                count++
            }
        }
    }

    return count
}

func dfs(grid [][]byte, i, j ,rows, cols int) {
	if i < 0 || j < 0 || i >= rows || j >= cols {
        return
    }
    if grid[i][j] == '0' {
        return
    }

    grid[i][j] = '0'
    dfs(grid, i-1, j, rows, cols)
    dfs(grid, i+1, j, rows, cols)
    dfs(grid, i, j-1, rows, cols)
    dfs(grid, i, j+1, rows, cols)

    return
}
```



<br/>

### No.1905 统计子岛屿

- 链接：[https://leetcode.cn/problems/count-sub-islands/](https://leetcode.cn/problems/count-sub-islands/)
- 思路：还是 DFS，不过这里如果 grid2 符合条件再 count ++ 

```go
func countSubIslands(grid1 [][]int, grid2 [][]int) int {
    
    if len(grid2) == 0{
        return 0
    }
    rows := len(grid2)
    cols := len(grid2[0])

    count := 0
    var add bool 
    var dfs func(i,j int)
    dfs = func(i, j int){
        if i < 0 || i >= rows || j < 0 || j >= cols { return }
        if grid2[i][j] == 0 { return }
        
        grid2[i][j] = 0  
        if grid1[i][j] == 0{
            add = false
        }
        dfs(i+1, j)
        dfs(i-1, j)
        dfs(i, j+1)
        dfs(i, j-1)
    }

    for i :=0; i < rows; i++{
        for j :=0; j < cols; j++{
            if grid2[i][j] == 1{
                add = true
                dfs(i,j)
                if add {
                    count++
                }
            }
        }
    }
    return count
}

```

<br/>

### No. 695 岛屿的最大面积

链接：[https://leetcode.cn/problems/max-area-of-island/](https://leetcode.cn/problems/max-area-of-island/)

思路： 这个题和岛屿数量一样，几乎没有什么难度，每个dfs 执行一遍就size++，dfs全部执行完就是岛屿的面积大小

```go
var size int 

func maxAreaOfIsland(grid [][]int) int {
	if len(grid) == 0 {
		return 0
	}

	row := len(grid)
	col := len(grid[0])

	maxSize := 0
	for i := 0; i < row; i++ {
		for j := 0; j < col; j++ {
			if grid[i][j] == 1 {
				dfsPro(grid, i, j, row, col)
				if maxSize < size {
					maxSize = size
				}
				size = 0
			}
		}
	}
	return maxSize
}

func dfsPro(grid [][]int, i, j ,row, col int) {
	if i < 0 || i >= row || j < 0 || j >= col {
		return
	}
	
	if grid[i][j] == 0 {
		return
	}

    size ++ 
	grid[i][j] = 0

	dfsPro(grid, i + 1, j, row, col)
	dfsPro(grid, i - 1, j, row, col)
	dfsPro(grid, i, j - 1, row, col)
	dfsPro(grid, i, j + 1, row, col)
}
```




