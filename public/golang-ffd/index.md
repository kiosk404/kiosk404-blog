# 算法笔记-首次适应递减算法（FFD）


FFD 算法，即首次适应递减算法（First Fit Decreasing），是一种用于解决装箱问题（Bin Packing Problem）的近似算法。装箱问题的目标是将一系列具有不同大小的物品，尽可能高效地装入有限容量的箱子中，使得使用的箱子数量最少。

<!--more-->

在计算机科学和运筹学领域，装箱问题（Bin Packing Problem）是一个经典且具有实际应用价值的问题。该问题旨在将不同大小的物品合理地装入有限容量的箱子中，使得使用的箱子数量最少。

装箱问题在计算机科学和运筹学中有着广泛的应用场景，例如：

- 云资源分配中的虚拟机部署
- CDN流量调度规划
- 物流运输中的货物装载优化
- 内存管理中的内存块分配

</br>

## FFD 算法基础

FFD 算法的核心思想分为两个主要步骤：

1. **排序阶段**：首先将所有待装箱的物品按照大小从大到小进行排序。这样做的目的是优先处理较大的物品，因为较大的物品在装箱时更难找到合适的空间，先处理它们可以减少后续小物品放置时的空间浪费。
2. **装箱阶段**：依次考虑排序后的每个物品，将其放入第一个能够容纳它的箱子中。如果当前没有箱子能够容纳该物品，则开启一个新的箱子来放置它。

### 算法复杂度分析

- **时间复杂度**：排序阶段通常使用高效的排序算法（如快速排序），时间复杂度为 \(O(n log n)\)，其中 n 是物品的数量。装箱阶段需要遍历每个物品并尝试将其放入合适的箱子中，时间复杂度为 \(O(n^2)\)。因此，FFD 算法的总体时间复杂度为 \(O(n^2)\)。
- **空间复杂度**：主要的空间开销用于存储排序后的物品列表和箱子信息，空间复杂度为 \(O(n)\)。

### go 语言实现

``` go
package main

import (
    "fmt"
    "sort"
)

// Bin 表示一个箱子，包含箱子的容量和已放置的物品列表
type Bin struct {
    capacity int
    items    []int
}

// NewBin 创建一个新的箱子，初始容量为指定值
func NewBin(capacity int) *Bin {
    return &Bin{
        capacity: capacity,
        items:    []int{},
    }
}

// Load 获取当前总负载（已使用容量）
func (b *Bin) Load() int {
	total := 0
	for _, item := range b.items {
		total += item
	}
	return total
}

// AddItem 将一个物品添加到箱子中
func (b *Bin) AddItem(item int) bool {
    if b.capacity >= item {
        b.items = append(b.items, item)
        b.capacity -= item
        return true
    }
    return false
}

// FirstFitDecreasing 实现 FFD 算法
func FirstFitDecreasing(items []int, binCapacity int) []*Bin {
    // 对物品按照大小从大到小排序
    sort.Slice(items, func(i, j int) bool {
        return items[i] > items[j]
    })

    bins := []*Bin{NewBin(binCapacity)}
    for _, item := range items {
        addNewOne := false
        for _, bin := range bins {
            if bin.AddItem(item) {
                addNewOne = true
                break
            }
        }
        if addNewOne {
            newBin := NewBin(binCapacity)
            newBin.AddItem(item)
            bins = append(bins, newBin)
        }
    }
    return bins
}
```

以下是调用的示例：

```go
func main() {
    items := []int{5, 4, 7, 2, 3}
    binCapacity := 10
    bins := FirstFitDecreasing(items, binCapacity)
    for i, bin := range bins {
        fmt.Printf("Bin %d: Items %v, Remaining Capacity %d\n", i+1, bin.items, bin.capacity)
    }
}

/*
输出结果：
Bin 1: Items [7 3], Remaining Capacity 0
Bin 2: Items [5 4], Remaining Capacity 1
Bin 3: Items [2], Remaining Capacity 8
*/
```

## FFD 的变种改进

### 装箱更平均

**原理**：变种需要 **在保证合理箱子数量的前提下尽可能均衡各箱子的负载**。

正常的 FFD 一定是追求箱子最少的，也就是没有去追求平均。

**需求**：现在假设箱子总数是一定的， 需求是需要判断所有的物品能否放到这些箱子中，如果能放下，我需要物品尽可能的平均。

**实现**：

在算法中实现 `BinHeap` 最小堆的核心目的是为了**高效地维护和选择当前负载最小的可用箱子**，从而确保物品能够尽可能均匀地分配到所有箱子中。以下是详细的解释：

#### 1. **问题本质**

当需要将多个物品分配到固定数量的箱子时，想要实现「平均分配」，关键在于**每次分配时选择当前负载最小的箱子**。这样做可以：

- 避免某些箱子过早被填满
- 动态平衡各箱子的负载
- 最大化利用所有箱子的剩余容量

#### 2. **传统方法的缺陷**

如果使用普通数组存储箱子，每次分配物品时需要遍历所有箱子（时间复杂度为 `O(k)`，`k` 是箱子数量）来找到负载最小的箱子。当箱子数量较大时，这种方法的效率会显著降低。

#### 3. **最小堆的优势**

为了快速找到容量最小的箱子这里使用了最小堆，最小堆（Priority Queue）的特性是能始终在 `O(1)` 时间内获取当前负载最小的箱子，并在 `O(log k)` 时间内完成堆结构的调整。这种数据结构完美契合了「动态选择负载最小箱子」的需求。

```go
type BinHeap []*Bin

func (h BinHeap) Len() int { return len(h) }
func (h BinHeap) Less(i, j int) bool {
	return h[i].Load() < h[j].Load() // 按当前负载排序
}
func (h BinHeap) Swap(i, j int) { h[i], h[j] = h[j], h[i] }

func (h *BinHeap) Push(x interface{}) {
	*h = append(*h, x.(*Bin))
}

func (h *BinHeap) Pop() interface{} {
	old := *h
	n := len(old)
	x := old[n-1]
	*h = old[0 : n-1]
	return x
}
```

下面是利用最小堆进行平衡FFD的查找算法。

```go

func CanBalanceFirstFitDecreasing(items []int, binCapacity int, binCount int) (bool, []*Bin) {
	// 预处理检查
	totalVolume := 0
	for _, item := range items {
		if item > binCapacity { // 存在无法放入单个箱子的物品
			return false, nil
		}
		totalVolume += item
	}
	if totalVolume > binCapacity*binCount { // 总体积超过总容量
		return false, nil
	}

	// 初始化箱子
	bins := make([]*Bin, binCount)
	for i := 0; i < binCount; i++ {
		bins[i] = &Bin{capacity: binCapacity}
	}

	// 降序排序物品
	sort.Slice(items, func(i, j int) bool {
		return items[i] > items[j]
	})

	// 使用最小堆管理箱子负载
	h := make(BinHeap, binCount)
	copy(h, bins)
	heap.Init(&h)

	// 分配物品
	for _, item := range items {
		// 临时存储弹出的箱子
		var temp []*Bin
		found := false

		// 尝试找到可以放入的负载最小箱子
		for h.Len() > 0 {
			bin := heap.Pop(&h).(*Bin)
			if bin.AddItem(item) {
				temp = append(temp, bin)
				found = true
				break
			}
			temp = append(temp, bin)
		}

		// 把临时存储的箱子重新放回堆
		for _, b := range temp {
			heap.Push(&h, b)
		}

		if !found {
			return false, nil // 无法放入任何箱子
		}
	}

	return true, bins
}
```

执行效果：

```go
func main() {
	// 改进的 FFD 算法
	testCases := []struct {
		items       []int
		binCapacity int
		binCount    int
	}{
		{[]int{2, 3, 7}, 10, 5},          // 可分配
		{[]int{9, 7, 2, 5, 3}, 10, 4},    // 可分配
		{[]int{8, 8, 8, 8}, 10, 3},       // 不可分配（总体积超限）
		{[]int{5, 5, 5, 5, 5, 5}, 10, 3}, // 完美分配
	}

	for _, tc := range testCases {
		possible, bins := CanBalanceFirstFitDecreasing(tc.items, tc.binCapacity, tc.binCount)
		fmt.Printf("物品：%v\n箱子容量：%d 数量：%d\n",
			tc.items, tc.binCapacity, tc.binCount)
		if possible {
			fmt.Println("可以分配，分配结果：")
			for i, bin := range bins {
				fmt.Printf("  箱子%d: 负载=%d (剩余%d) 物品=%v\n",
					i+1, bin.Load(), bin.capacity, bin.items)
			}
		} else {
			fmt.Println("无法完成分配")
		}
		fmt.Println("-------------------")
	}
}

// 执行的结果
物品：[7 3 2]
箱子容量：10 数量：5
可以分配，分配结果：
  箱子1: 负载=7 (剩余3) 物品=[7]
  箱子2: 负载=2 (剩余8) 物品=[2]
  箱子3: 负载=0 (剩余10) 物品=[]
  箱子4: 负载=0 (剩余10) 物品=[]
  箱子5: 负载=3 (剩余7) 物品=[3]
-------------------
物品：[9 7 5 3 2]
箱子容量：10 数量：4
可以分配，分配结果：
  箱子1: 负载=9 (剩余1) 物品=[9]
  箱子2: 负载=5 (剩余5) 物品=[5]
  箱子3: 负载=5 (剩余5) 物品=[3 2]
  箱子4: 负载=7 (剩余3) 物品=[7]
-------------------
物品：[8 8 8 8]
箱子容量：10 数量：3
无法完成分配
-------------------
物品：[5 5 5 5 5 5]
箱子容量：10 数量：3
可以分配，分配结果：
  箱子1: 负载=10 (剩余0) 物品=[5 5]
  箱子2: 负载=10 (剩余0) 物品=[5 5]
  箱子3: 负载=10 (剩余0) 物品=[5 5]
-------------------
```



### 条件装箱

**需求**：

在原有装箱算法基础上，新增以下需求：

1. **强制亲和性**：特定物品必须放入指定箱子
2. **偏好亲和性**：优先但非强制放入指定箱子
3. **组合约束**：多个物品需要放入同一箱子
4. **冲突约束**：某些物品不能共处一箱

其实这样的需求就和 K8S 的调度算法有异曲同工之妙，类似的，还有 CDN 对线上接入流量的分配算法等，其本质都是基于 FFD



#### 1. 问题的本质

当需要多个条件都满足的复杂情况下，这是一个多条件择优的问题，想要实现该算法，必须按照优先级原则一一实现过滤方法。

算法本身是没有什么复杂度。

#### 2. 传统方法的缺陷

选择箱子的策略单一，但实际上

**实现**:

```go
// EnhancedItem 增强型物品定义
type EnhancedItem struct {
	ID        int
	Volume    int
	Affinity  map[int]int       // 亲和
	Conflicts map[int]bool      // 冲突
}

func (i EnhancedItem) String() string {
	return fmt.Sprintf("物品%d(体积%d)", i.ID, i.Volume)
}

type EnhancedBin struct {
	capacity int
	items    []EnhancedItem
	itemSet  map[int]bool
	tags     map[string]interface{}
}

func NewEnhancedBin(capacity int, tags map[string]interface{}) *EnhancedBin {
	return &EnhancedBin{
		capacity: capacity,
		tags:     tags,
		itemSet:  make(map[int]bool),
	}
}

// Load 计算当前负载
func (b *EnhancedBin) Load() int {
	total := 0
	for _, item := range b.items {
		total += item.Volume
	}
	return total
}

// SafeAdd 安全添加物品方法
func (b *EnhancedBin) SafeAdd(item EnhancedItem) bool {
	if item.Volume > b.capacity {
		return false
	}

	for conflictID := range item.Conflicts {
		if b.itemSet[conflictID] {
			return false
		}
	}

	if b.itemSet[item.ID] {
		return false // 关键修复：防止重复添加
	}

	b.items = append(b.items, item)
	b.capacity -= item.Volume
	b.itemSet[item.ID] = true
	return true
}

// EnhancedBinHeap 最小堆实现
type EnhancedBinHeap []*EnhancedBin

func (h EnhancedBinHeap) Len() int           { return len(h) }
func (h EnhancedBinHeap) Less(i, j int) bool { return h[i].Load() < h[j].Load() }
func (h EnhancedBinHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }

func (h *EnhancedBinHeap) Push(x interface{}) {
	*h = append(*h, x.(*EnhancedBin))
}

func (h *EnhancedBinHeap) Pop() interface{} {
	old := *h
	n := len(old)
	x := old[n-1]
	*h = old[0 : n-1]
	return x
}
```

**带亲和性的装箱算法**

```go
// AffinityPacking 带亲和性的装箱算法
func AffinityPacking(items []EnhancedItem, bins []*EnhancedBin, binCapacity int, allowNewBins bool) ([]*EnhancedBin, error) {
	sort.Slice(items, func(i, j int) bool {
		return items[i].Volume > items[j].Volume
	})

	for _, item := range items {
		if placed := tryForceAffinity(item, bins); placed {
			continue
		}

		if placed := tryPreferredAffinity(item, bins); placed {
			continue
		}

		candidates := filterConflictBins(item, bins)
		if target := selectOptimalBin(item, candidates); target != nil {
			target.SafeAdd(item)
			continue
		}

		if allowNewBins {
			newBin := NewEnhancedBin(binCapacity, nil)
			newBin.SafeAdd(item)
			bins = append(bins, newBin)
			continue
		}

		return nil, fmt.Errorf("无法分配 %v", item)
	}

	return bins, nil
}
```

辅助函数

```go
// 强制分配
func tryForceAffinity(item EnhancedItem, bins []*EnhancedBin) bool {
	for binID, priority := range item.Affinity {
		if priority >= 0 {
			continue
		}

		if binID < len(bins) && bins[binID].SafeAdd(item) {
			return true
		}
	}
	return false
}

// 亲和性分配
func tryPreferredAffinity(item EnhancedItem, bins []*EnhancedBin) bool {
	type candidate struct {
		bin      *EnhancedBin
		priority int
	}

	var candidates []candidate
	for binID, p := range item.Affinity {
		if p > 0 && binID < len(bins) {
			candidates = append(candidates, candidate{
				bin:      bins[binID],
				priority: p,
			})
		}
	}

	// 按优先级降序排序
	sort.Slice(candidates, func(i, j int) bool {
		return candidates[i].priority > candidates[j].priority
	})

	for _, c := range candidates {
		if c.bin.SafeAdd(item) {
			return true
		}
	}
	return false
}

// 冲突分配
func filterConflictBins(item EnhancedItem, bins []*EnhancedBin) []*EnhancedBin {
	var valid []*EnhancedBin
	for _, bin := range bins {
		hasConflict := false
		for conflictID := range item.Conflicts {
			if bin.itemSet[conflictID] {
				hasConflict = true
				break
			}
		}
		if !hasConflict {
			valid = append(valid, bin)
		}
	}
	return valid
}

// 选择一个箱子
func selectOptimalBin(item EnhancedItem, candidates []*EnhancedBin) *EnhancedBin {
	if len(candidates) == 0 {
		return nil
	}

	h := make(EnhancedBinHeap, len(candidates))
	copy(h, candidates)
	heap.Init(&h)

	var temp []*EnhancedBin
	defer func() {
		for _, b := range temp {
			heap.Push(&h, b)
		}
	}()

	for h.Len() > 0 {
		bin := heap.Pop(&h).(*EnhancedBin)
		if bin.SafeAdd(item) {
			return bin
		}
		temp = append(temp, bin)
	}

	return nil
}
```

代码执行

```go
func main() {
	// 测试用例
	bins := []*EnhancedBin{
		NewEnhancedBin(10, map[string]interface{}{"type": "special"}),
		NewEnhancedBin(10, nil),
		NewEnhancedBin(10, nil),
	}

	items := []EnhancedItem{
		{
			ID:       1,
			Volume:   8,
			Affinity: map[int]int{0: -1}, // 必须放入0号箱
		},
		{
			ID:       2,
			Volume:   5,
			Affinity: map[int]int{1: 5}, // 优先放入1号箱
		},
		{
			ID:        3,
			Volume:    4,
			Conflicts: map[int]bool{2: true}, // 与物品2冲突
		},
		{
			ID:       4,
			Volume:   2,
			Affinity: nil,              // 无所谓
		},
	}

	result, err := AffinityPacking(items, bins, 10, false)
	if err != nil {
		panic(err)
	}

	fmt.Println("装箱结果：")
	for i, bin := range result {
		fmt.Printf("箱子%d [标签:%v]:\n", i, bin.tags)
		fmt.Printf("  负载: %d/%d\n", bin.Load(), bin.Load()+bin.capacity)
		fmt.Printf("  物品: %v\n", lo.Map(bin.items, func(item EnhancedItem, index int) string {
			return fmt.Sprintf("{%s}", item.String())
		}))
		fmt.Println("-------------------")
	}
}


// 执行结果
装箱结果：
箱子0 [标签:map[type:special]]:
  负载: 8/10
  物品: [{物品1(体积8)}]
-------------------
箱子1 [标签:map[]]:
  负载: 5/10
  物品: [{物品2(体积5)}]
-------------------
箱子2 [标签:map[]]:
  负载: 6/10
  物品: [{物品3(体积4)} {物品4(体积2)}]
-------------------
```


