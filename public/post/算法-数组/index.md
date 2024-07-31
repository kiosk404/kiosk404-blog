# 算法 - (数组、链表)

数组、链表是编程语言中最常见的数据结构，也是最基础的数据结构。以下会以几道 LeetCode 巩固自己的基础


<!-- more -->
# 结构对比
数组和链表各有各的优势，比如数组的随机插入和删除都是O(n)的，可谓是很低效了。但是数组的查找是是O(1)，直接指定下标即可找到对应的元素。而链表必须遍历，也就是想要查找你一个元素的时间复杂度是 O(n)。所以数组和链表各有各的优势，互相补充。

| 数据结构      |  操作  		 |  时间复杂度  |
| ------ | ----------- |  ------ |
| 数组         | prepend 	    |   O(1)     |
|             | append     	 |   O(1)      |
|             | lookup       |   O(1)      |
|             | insert       |   O(n)      |
|             | delete       |   O(n)      |
|             | --       |      --   |
| 链表         | prepend 	    |   O(1)    |
|             | append     	 |   O(1)      |
|             | lookup       |   O(n)     |
|             | insert       |   O(1)     |
|             | delete       |   O(1)     |


# 实现

## 数组
``` go
a := []int{1, 1, 2, 2, 2, 3, 4, 5, 5, 5, 6}

var b []int
b = append(c, 11)
b = append(b, a...)


// make( []Type, size, cap )
// 其中 Type 是指切片的元素类型，
// size 指的是为这个类型分配多少个元素，cap 为预分配的元素数量，
// 这个值设定后不影响 size，只是能提前分配空间，降低多次分配空间造成的性能问题。
c := make([]int, 2, 10)  
```

<br/>

## 链表

``` go
// 定义单链表
type ListNode struct {     
	Val 	int
	Next 	*ListNode
}

func (ln *ListNode) add(v int) {
	if ln == nil {
		ln = &ListNode{Val: v,Next: nil}
	}
	for ln.Next != nil {
		ln = ln.Next
	}
	ln.Next = &ListNode{Val: v,Next: nil}
}

func (ln *ListNode) traverse() {
	for ln != nil {
		fmt.Printf("%d ",ln.Val)
		ln = ln.Next
	}
}

```


# Leetcode
## 数组
### No.1 两数之和 

- 链接：[https://leetcode-cn.com/problems/two-sum/](https://leetcode-cn.com/problems/two-sum/)
- 思路：将每一个数需要的另一半数存入一个 hash 表中

``` go
func twoSum(nums []int, target int) []int {
	tmp := make(map[int]int, len(nums)/2)
	for i := 0;i < len(nums);i++ {
		if index, ok := tmp[target-nums[i]]; ok {
			return []int{index,i}
		}
		tmp[nums[i]] = i
	}
	return []int{}
}
```
<br/>

### No.3 无重复字符的最长子串

- 链接：[https://leetcode.cn/problems/longest-substring-without-repeating-characters/description/](https://leetcode.cn/problems/longest-substring-without-repeating-characters/description/)
- 本来想的是将所有可能的字串遍历一遍不就得了，如下面代码，没想到超出运行时间了，有一例测试用例是800 长的字符串，所以代码超过运行时间了。



>  首先需要明确， 无重复字符串是什么意思。 awwke , 最长无重复子串是 wke ，因为 ww 重复了！！

```go
func lengthOfLongestSubstring(s string) int {
	if len(s) == 0 {
		return 0
	}

	checkFunc := func(sub string) bool {
		m := make(map[byte]struct{})
		for i := 0 ; i < len(sub); i++ {
			if _, ok := m[sub[i]]; ok {
				return false
			} else {
				m[sub[i]] = struct{}{}
			}
		}
		return true
	}

	maxLength := 1
	for i := 0; i < len(s); i++ {
		for j := i; j < len(s); j ++ {
			subStr := s[i:j+1]
			if checkFunc(subStr) {
				if len(subStr) > maxLength {
					maxLength = len(subStr)
				}
			}
		}
	}
	return maxLength
}
```

看了一下官方题解，原来是要搞个滑动窗口。

1. 首先，用一个map表示滑动窗口。
2. 不断移动右指针，当出现某个字符不为0时，证明子串里有重复了（map 里面已经存过了）。
3. 重复的肯定是左指针指向的值，所以删除左指针所指的值，保持滑动窗口向前滑动。
4. 直到计算出最大子串的值

```go
func lengthOfLongestSubstring(s string) int {
	right := 0
	// 用于存当前窗口中都有哪些字符，当map的value不为1时，表示需要滑动了
	storeMap := make(map[byte]int)
	max := 0

	for left := 0; left < len(s); left++ {
        // 左指针向前移动一下，消除窗口中的重复值
		if left != 0 {
			delete(storeMap, s[left-1])
		}
		for ; right < len(s); right++ {
			// 为0表示是新元素
			if storeMap[s[right]] == 0 {
				storeMap[s[right]]++
			} else {
				break
			}
		}
		// 窗口中存在重复字符串了(或者遍历完了数组), 且肯定出现的重复字符是最早加入的（left所指的值）
		if max < right-left {
			max = right - left
		}
	}

	return max
}
```



 <br/>

### No.5 最长回文子串

- 链接：[https://leetcode.cn/problems/longest-palindromic-substring/](https://leetcode.cn/problems/longest-palindromic-substring/)
- 思路：这道题就是一个长串里找回文子串，使用中心扩展法。

回文有两种形式， "aba" 和 "aabb" 都是回文的模式。我们对长串的每一个字段都进行一次中心扩展。

```go
func longestPalindrome(s string) string {
	if len(s) <= 1 {
		return s
	}
	longestStr := ""

	for i := 0; i < len(s); i++ {
		str1 := helper(s, i, i+1)  // 考虑 abba 形式
		str2 := helper(s, i, i)    // 考虑 aba 形式
		if len(longestStr) < len(str1) {
			longestStr = str1
		}
		if len(longestStr) < len(str2) {
			longestStr = str2
		}
	}

	return longestStr
}

// 查找最长子串
func helper(s string, left, right int) string {
	var str string
	for ; left >= 0 && right <= len(s)-1 && s[left] == s[right]; left, right = left-1, right+1 {
		str = s[left : right+1]
	}
	return str
}
```



<br/>



### No.11 盛最多水的容器

- 链接：[https://leetcode.cn/problems/container-with-most-water/](https://leetcode.cn/problems/container-with-most-water/)
- 思路：吃了上面第3题的亏，肯定不能再给出暴力解法了。需要有一个滑动窗口的概念。

1. 数组从两边开始向中间凑。选出一个最大的面积。

```go
func maxArea(height []int) int {
	maxV := 0
	left, right := 0, len(height)-1
	
	for left < right {
		if height[left] > height[right] {
			maxV = max((right-left) * height[right], maxV)
			// 左边更高，右边左移
			right -- 
		} else {
			maxV = max((right-left) * height[left], maxV)
			// 右边更高，左边右移动
			left ++ 
		}
	}
	
	return maxV
}

func max(a,b int) int {
    if a > b {
        return a 
    }
    return b
}
```





<br/>

### No.15 三数之和

- 链接：[https://leetcode.cn/problems/3sum/](https://leetcode.cn/problems/3sum/)
- 思路：先将数组排序，再2层循环。如[-1,0,1,2,-1,-4 ] 先排序，排序之后 [-4, -1, -1, 0, 1, 2] 数组，再进行一次双重循环，第一次取 -4 ，再对 -4 之后的数组进行一次循环，-4 + 数组头元素 + 数组尾元素 > 0 就证明，数组尾元素过大， < 0 就证明数组头元素太小。逐渐的收拢第二个数组。

```go
func threeSum(nums []int) [][]int {
	sort.Ints(nums)

	validNums := make(map[[3]int]struct{})

	for i := 0; i < len(nums)-2; i++ {
		minNum := nums[i]
		midNumIndex := i + 1
		maxNumIndex := len(nums) - 1
		// 中间索引 小于 最大索引
		for midNumIndex < maxNumIndex {
			midNum := nums[midNumIndex]
			maxNum := nums[maxNumIndex]
			// 大于0了，证明最大的数太大了
			if minNum+midNum+maxNum > 0 {
				maxNumIndex--
				continue
			}
			// 小于0了，证明中间数太小了
			if minNum+midNum+maxNum < 0 {
				midNumIndex++
				continue
			}
			// 只能是等于 0 了, 中间数加点，最大数减点，看看是否还能拼出0
			if minNum+midNum+maxNum == 0 {
				validNums[[3]int{minNum, midNum, maxNum}] = struct{}{}
				midNumIndex++
				maxNumIndex--
			}
		}
	}

	var res [][]int
	for k, _ := range validNums {
		res = append(res, []int{k[0], k[1], k[2]})
	}

	return res
}
```

<br/>

### No.26 删除排序数组中的重复项

- 链接: [https://leetcode-cn.com/problems/remove-duplicates-from-sorted-array/](https://leetcode-cn.com/problems/remove-duplicates-from-sorted-array/)
- 思路1：**双指针法** [题解动画](https://leetcode-cn.com/problems/remove-duplicates-from-sorted-array/solution/26-shan-chu-pai-xu-shu-zu-zhong-de-zhong-fu-xia-89/)

``` go
func removeDuplicates(nums []int) int {
	if len(nums) == 0 {
		return 0
	}

	slow := 1
	for fast := 1; fast < len(nums); fast++ {
		if nums[fast] != nums[fast-1] {
			nums[slow] = nums[fast]
			slow++
		}
	}

	return slow
}
```
- 思路2： **旋转数组法** 旋转数组应该是我比较喜欢的一种方式，比较简单，那就是数组翻转2次，可以将首位数转到末尾曲，重复项判断刚好使用这种方式，<font style='color:red'>对于移动数组类的题是比较通吃的一个方法</font>问题就是时间复杂度比较大

  

  <img src='https://img1.kiosk007.top/static/images/leetcode/list_1.jpeg' style="height: 300px; zoom: 150%;"/>

``` go
func reversal(nums []int) {
	length := len(nums)
	for i := 0; length > 1 && i < length/2; i ++ {
		nums[i],nums[length-1-i] = nums[length-1-i],nums[i]
	}
}

func removeDuplicates(nums []int) int {
	length := len(nums)

	for i := 0; i < length; i ++ {
		if i + 1 < length && nums[i] == nums[i+1] {
			reversal(nums[i+1:length])
			reversal(nums[i+1:length-1])
			length = length - 1
			i = i - 1
		}
	}
	return length
}
```
<br/>

### No.27 移除元素

- 链接：[https://leetcode.cn/problems/remove-element/description/](https://leetcode.cn/problems/remove-element/description/)
- 思路：考虑2层循环，第二层循环直到找到不需要的移除的元素

```go
func removeElement(nums []int, val int) int {
	index := 0
	for i := 0; i < len(nums); i++ {
		for j := i; j < len(nums); j++ {
			if nums[j] != val {
				nums[index] = nums[j]
				index++
				i = j        // 将第一层循环的下标移动至不需要移除的地方
				break
			}
		}
	}
	return index
}
```



<br/>

### No.29 顺时针打印矩阵

- 链接：[https://leetcode.cn/problems/shun-shi-zhen-da-yin-ju-zhen-lcof/description/](https://leetcode.cn/problems/shun-shi-zhen-da-yin-ju-zhen-lcof/description/)
- 思路：创造边缘条件，进行顺时针打印。

<img src="https://img1.kiosk007.top/static/images/blog/jianzhi_29_fig1.png" alt="jianzhi_29_fig1" style="zoom:43%;" />



```go
func spiralOrder(matrix [][]int) []int {
	if len(matrix) == 0 || len(matrix[0]) == 0 {
		return []int{}
	}

	var (
		rows, columns = len(matrix), len(matrix[0])
		order = make([]int, rows * columns)
		index = 0
		left, right, top, bottom = 0, columns -1 , 0 , rows -1
	)

	for left <= right && top <= bottom {
		// 打印最上层(包含top行最后一个)
		for col := left; col <= right; col ++ {
			order[index] = matrix[top][col]
			index++
		}
		// 打印最右边(包含最右列最后一个)
		for row := top + 1; row <= bottom; row ++ {
			order[index] = matrix[row][right]
			index++
		}
		//
		if left < right && top < bottom {
			// 打印最下边(不包含bottom行最后一个和第一个)
			for col := right - 1; col > left; col -- {
				order[index] = matrix[bottom][col]
				index++
			}
			// 打印最左边(不包含left列最后一个)
			for row := bottom; row > top; row -- {
				order[index] = matrix[row][left]
				index++
			}
		}
	}
	left ++
	right --
	top ++
	bottom --

	return order
}
```



<br/>

### No.209 长度最小的子数组

- 链接：[https://leetcode.cn/problems/minimum-size-subarray-sum/description/](https://leetcode.cn/problems/minimum-size-subarray-sum/description/)
- 思路：其题意是在数组中找到满足 **连续** 、**子数组和大于target**和 **数量最少** 三个条件的子数组，由连续就能快速联想并对应到滑动窗口的概念。

```go
func minSubArrayLen(target int, nums []int) int {
	ri := 0  // 右指针
	minLength := math.MaxInt  

	if len(nums) == 0 {
		return 0
	}

	curSize := 0

	// 移动左指针
	for i := 0 ; i < len(nums); i ++ {
		if i != 0 { 
			curSize -= nums[i-1]
		}
		
		// 移动右指针。当前Size >= target就退出
		for ri < len(nums) && nums[ri] + curSize < target {
			curSize += nums[ri]
			ri ++
		}
        // 是我们想要的退出时机，而不是ri遍历完了
		if ri < len(nums) && curSize + nums[ri] >= target {
			if ri + 1 - i < minLength {
				minLength = ri + 1 -i
			}
		}
	}
	if minLength == math.MaxInt {
		return 0
	}

	return minLength
}
```

第二种写法，也是标准的滑动窗口。

```go
func minSubArrayLen(target int, nums []int) int {
	var count int = math.MaxInt64
	var targetList []int
	var right int

	for left := 0; left < len(nums); left++ {

		// 持续移动窗口的左边，直到窗口中的数组之和不满足和大于target
		for len(targetList) > 0 {
			// 找到最小
			if sum(targetList) >= target && len(targetList) <= count {
				count = len(targetList)
			} else {
				break
			}
			// 逐渐左移下标
			targetList = targetList[1:]
		}

		// 移动滑动窗口的右边界
		for ; right < len(nums); right++ {
			targetList = append(targetList, nums[right])
			if sum(targetList) < target {
				continue
			} else {
				// 能到这里表示滑动窗口里的值是满足要求的，右指针再往右一格
				if len(targetList) <= count {
					count = len(targetList)
				}
				right++
				break
			}
		}
	}

	if count == math.MaxInt64 {
		return 0
	}

	return count
}

func sum(list []int) int {
	all := 0
	for _, v := range list {
		all += v
	}
	return all
}

```

<br/>

### No.415 字符串相加

- 链接：[https://leetcode.cn/problems/add-strings/](https://leetcode.cn/problems/add-strings/)
- 采用末尾向前递进的方式

```go
func addStrings(num1 string, num2 string) string {
    add := 0
    ans := ""
    for i, j := len(num1) - 1, len(num2) - 1; i >= 0 || j >= 0 || add != 0; i, j = i - 1, j - 1 {
        var x, y int
        if i >= 0 {
            x = int(num1[i] - '0')
        }
        if j >= 0 {
            y = int(num2[j] - '0')
        }
        result := x + y + add
        ans = strconv.Itoa(result%10) + ans
        add = result / 10
    }
    return ans
}
```



<br/>

### No.459 重复的子字符串

- 链接：[https://leetcode.cn/problems/repeated-substring-pattern/submissions/](https://leetcode.cn/problems/repeated-substring-pattern/submissions/)
- 思路：先第一重循环，找出子串。子串的头已经有了，就是字符串的第一个元素，所以移动下标找末尾元素即可。再到第二个循环中从子串长度开始循环，遍历完整个字符串。

``` go
func repeatedSubstringPattern(s string) bool {
	length := len(s)
	for i := 1; i * 2 <= length; i++ {  // i为子串长度
		if length % i == 0 {    // 总长度 % 子串长度 必须为0
			var flag bool = true
			for j := i ; j < length ; j ++ {  // 循环一遍，不能为false
				if s[j] != s[j-i] {  // 子串在父串中可循环
					flag = false
					break
				}
			}
			if flag { return flag }
		}
	}
	return false
}

```



解法二：如果将字符串叠加，再移除头部和尾部的字符串，如果是重复的，一定可以在合并字符串中找到原本的自己。



```go
func repeatedSubstringPattern(s string) bool {
    return strings.Index((s + s)[1:len(s)*2-1], s) != -1
}
```



<br/>

### No. 674 最长连续递增序列

- 链接：[https://leetcode.cn/problems/longest-continuous-increasing-subsequence/](https://leetcode.cn/problems/longest-continuous-increasing-subsequence/)
- 思路：暴力查找法，其实就O(N)，一次遍历

```go
func findLengthOfLCIS(nums []int) int {
	if len(nums) == 0 {
		return 0
	}
	prev := nums[0]
	maxLength := 0

	for i := 0; i < len(nums); i++ {
		curMax := 1
		for j := i + 1; j < len(nums); j ++ {
			if nums[j] > prev {
				curMax ++
				prev = nums[j]
			} else {
				i = j -1 
				break
			}
		}
		if curMax > maxLength {
			maxLength = curMax
		}
	}
	return maxLength
}
```





<br/>

## 链表

### No.2 两数相加

- 链接：https://leetcode.cn/problems/add-two-numbers/description/
- 思路：

```go
func addTwoNumbers(l1, l2 *ListNode) (head *ListNode) {
    var tail *ListNode
    carry := 0
    for l1 != nil || l2 != nil {
        n1, n2 := 0, 0
        if l1 != nil {
            n1 = l1.Val
            l1 = l1.Next
        }
        if l2 != nil {
            n2 = l2.Val
            l2 = l2.Next
        }
        sum := n1 + n2 + carry
        sum, carry = sum%10, sum/10
        if head == nil {
            head = &ListNode{Val: sum}
            tail = head
        } else {
            tail.Next = &ListNode{Val: sum}
            tail = tail.Next
        }
    }
    if carry > 0 {
        tail.Next = &ListNode{Val: carry}
    }
    return
}
```





<br/>

### No.21 合并两个链表

- 链接：[https://leetcode-cn.com/problems/merge-two-sorted-lists/](https://leetcode-cn.com/problems/merge-two-sorted-lists/)
- 思路1: [迭代法](https://leetcode-cn.com/problems/merge-two-sorted-lists/solution/he-bing-liang-ge-you-xu-lian-biao-by-leetcode-solu/)
``` go
func mergeTwoLists(l1 *ListNode, l2 *ListNode) *ListNode {
	tmp := &ListNode{}

	prev := tmp
	for l1 != nil && l2 != nil {
		if l1.Val < l2.Val{
			prev.Next = l1
			prev = prev.Next

			l1 = l1.Next
		}else {
			prev.Next = l2
			prev = prev.Next

			l2 = l2.Next
		}
	}

	if l2 != nil {
		prev.Next = l2
	}

	if l1 != nil {
		prev.Next = l1
	}

	return tmp.Next
}
```
<br/>



### No.22 链表的倒数第k个节点

- 链接：[https://leetcode.cn/problems/lian-biao-zhong-dao-shu-di-kge-jie-dian-lcof/](https://leetcode.cn/problems/lian-biao-zhong-dao-shu-di-kge-jie-dian-lcof/)
- 思路1：将节点全部保存到列表里，再返回索引，当然这个方法费空间。思路2：快慢指针。即 fast、slow 两个指针，fast 先移动k步，再slow和fast一起移动，当fast遍历结束时，slow就是倒数第k步。

```go
// 非常low的写法
func getKthFromEnd(head *ListNode, k int) *ListNode {
	var rest []*ListNode
	for node := head; node != nil; node = node.Next {
		rest = append(rest, node)
	}
	if len(rest) >= k {
		return rest[len(rest) - k]
	}
	return nil
}

// 快慢指针
func getKthFromEnd(head *ListNode, k int) *ListNode {
	fast, slow := head, head 
	for fast != nil && k > 0 {
		fast = fast.Next
		k--
	}
	for fast != nil {
		fast = fast.Next
		slow = slow.Next
	}
	return slow 
}
```





<br/>

### No.25 K个一组翻转链表

- 链接：[https://leetcode.cn/problems/reverse-nodes-in-k-group/description/](https://leetcode.cn/problems/reverse-nodes-in-k-group/description/)
- 思路：其实很简单，只要会链表翻转就会K个一组翻转。写一个翻转函数，将头部K个翻转，K个后面的再拼回来。循环此过程

```go
func reverseKGroup(head *ListNode, k int) *ListNode {
	if head == nil {
		return nil
	}

	var ok bool = true
	var sub *ListNode
	var root *ListNode = &ListNode{}
	var tmp *ListNode = root

	for ok {
		sub, ok = helper(head, k)
		tmp.Next = sub
		head = sub
		if ok {
			tmpK := k
			for tmpK > 0 {
				tmp = tmp.Next
				head = head.Next
				tmpK --
			}
		}
	}

	return root.Next
}

func helper(head *ListNode, k int) (*ListNode, bool) {
	if head == nil {
		return nil, false
	}
	root := head
	length := 0
	for head != nil {
		head = head.Next
		length++
	}

	if k > length {
		return root, false
	}

	head = root
	var prev *ListNode = nil
	cur := prev

	for head != nil && k > 0 {
		next := head.Next

		head.Next = prev
		cur = head
		prev = cur

		head = next
		k--
	}
	next := head
	ret := cur

	for cur.Next != nil {
		cur = cur.Next
	}
	cur.Next = next

	return ret, true
}

```





<br/>

### No.83 删除排序链表重复项

- 链接：[https://leetcode-cn.com/problems/remove-duplicates-from-sorted-list/](https://leetcode-cn.com/problems/remove-duplicates-from-sorted-list/)
- 思路1：题解感觉和我差不太多，我的思路是先把重复项移到尾端，然后cur 指针指向重复项的尾端就好了。但不知道为啥 8ms 打败 7.3% 。理论就是 O(N) 啊。
``` go
func deleteDuplicates(head *ListNode) *ListNode {
	tmp := &ListNode{}
	prev := tmp
	for head != nil {
		for head != nil && head.Next != nil && head.Val == head.Next.Val {
			head = head.Next
		}

		next := head.Next
		prev.Next = head
		prev = prev.Next
		head = next
	}

	return tmp.Next
}
```
<br/>

### No.92 翻转局部链表

- 链接：[https://leetcode.cn/problems/reverse-linked-list-ii/description/](https://leetcode.cn/problems/reverse-linked-list-ii/description/)
- 思路：取前半部分，再写一个链表翻转的函数，最后将后半部分接上。这里可以把链表翻转抽象出来。

```go
func reverseBetween(head *ListNode, left int, right int) *ListNode {
    if head == nil {
		return nil
	}
	length := right - left
	tmp := &ListNode{}
	tmp.Next = head
	cur := tmp

    // 将指针移动到翻转的前半部分
	for head != nil && left - 1 > 0 {
		next := head.Next

		cur.Next = head
		cur = cur.Next

		head = next
		left --
	}
    
	next := head
	cur.Next = reverseList(next, length+1)

	return tmp.Next
}

// 生成翻转和翻转的后半部分 
func reverseList(head *ListNode, length int) *ListNode {
if head == nil || length <= 0 {
		return head
	}

	var prev *ListNode = nil
	var cur *ListNode = prev

	for head != nil && length > 0 {
		next := head.Next

		head.Next = prev
		cur = head
		prev = cur

		length --
		head = next
	}
	next := head

	tmp := cur
	for tmp.Next != nil {
		tmp = tmp.Next
	}
	tmp.Next = next

	return cur
}
```





<br/>

### No.141 环形链表

- 链接: [https://leetcode-cn.com/problems/linked-list-cycle/](https://leetcode-cn.com/problems/linked-list-cycle/)
- 思路1：这道题思路非常简单，就是快慢指针，慢指针每次前进一步，快指针每次前进两步。如果存在环快指针一定会追上慢指针，**问题就是边界条件太多，需要仔细判断。**
- 思路2：hash表存已有的数据做对比，这个最简单，不演示啦~

``` go
func hasCycle(head *ListNode) bool {
    if head == nil {
		return false
	}
	slow := head
	fast := head.Next
	for fast != nil && fast.Next != nil {
		slow = slow.Next
		fast = fast.Next.Next
		if slow == fast {
			return true
		}
	}

	return false
}
```
<br/>

### No.148 排序链表

- 链接：[https://leetcode.cn/problems/sort-list/](https://leetcode.cn/problems/sort-list/)
- 思路：这道题采用归并排序的方法，将链表拆成足够小的部分，然后排序，就相当于是有序链表的合并，而拆到足够细时，因为链表就一个值，所以当然是有序的

```go
func sortList(head *ListNode) *ListNode {
	return sort(head, nil)
}

func sort(head *ListNode, tail *ListNode) *ListNode {
	if head == nil {
		return nil
	}

    // 拆到足够细了，就是链表的一个节点
	if head.Next == tail {
		head.Next = nil
		return head
	}

	fast := head
	slow := head

	for fast.Next != tail {
		fast = fast.Next
		slow = slow.Next
		if fast.Next != tail {
			fast = fast.Next
		}
	}
	return merge(sort(head, slow), sort(slow, tail))
}

// 有序链表的合并
func merge(p, q *ListNode) *ListNode {
	if p == nil {
		return q
	}
	if q == nil {
		return p
	}

	tmp := &ListNode{}
	cur := tmp
	for p != nil && q != nil {
		if p.Val > q.Val {
			cur.Next = &ListNode{Val: q.Val, Next: nil}
            q = q.Next
		} else {
			cur.Next = &ListNode{Val: p.Val, Next: nil}
			p = p.Next
		}
		cur = cur.Next
	}

	if p != nil {
		cur.Next = p
	} else {
		cur.Next = q
	}

	return tmp.Next
}

```







<br/>

### No.160 相交链表

- 链接：[https://leetcode.cn/problems/intersection-of-two-linked-lists/](https://leetcode.cn/problems/intersection-of-two-linked-lists/)
- 思路：可以直接将所有节点存到 hash 表中。

```go
func getIntersectionNode(headA, headB *ListNode) *ListNode {
	tmp := make(map[*ListNode]struct{})
	for headA != nil {
		tmp[headA] = struct{}{}
		headA = headA.Next
	}
	for headB != nil {
		if _, ok := tmp[headB]; ok {
			return headB
		}
		headB = headB.Next
	}
	return nil 
}
```

<br/>

### No.203 移除链表元素

- 链接：[https://leetcode.cn/problems/remove-linked-list-elements/](https://leetcode.cn/problems/remove-linked-list-elements/)
- 思路：很简单呐，就是双循环，里面那层循环遇到相等的就往后移动

```go
func removeElements(head *ListNode, val int) *ListNode {
	if head == nil {
		return nil 
	}
	
	tmp := &ListNode{}
	prev := tmp
	
	for head != nil {
		for head != nil && head.Val == val {
			head = head.Next
		}
		prev.Next = head
		prev = prev.Next
		if head != nil {
			head = head.Next
		}
	}
	
	return tmp.Next
}
```

<br/>

### No.206 反转链表

- 链接：[https://leetcode-cn.com/problems/reverse-linked-list/](https://leetcode-cn.com/problems/reverse-linked-list/)
- 思路1：[双指针](https://leetcode-cn.com/problems/reverse-linked-list/solution/206-fan-zhuan-lian-biao-shuang-zhi-zhen-fa-di-gui-/)  本质还是遍历 head， 上面的`next := head.Next`和下面的`head = next` 就是为了遍历， 中间的三行是 当前head的节点的下一个指向之前，cur即为当前head节点，cur 成为历史 prev

``` go
func reverseList(head *ListNode) *ListNode {
	if head == nil {
		return nil
	}

	var prev *ListNode = nil
	cur := prev

	for head != nil {
		next := head.Next  // 存下一个

		head.Next = prev // haed 的Next指向 prev
		cur = head       // cur 就是 head
		prev = cur		 // cur 成为 prev

		head = next   // head 前进
	}

	return cur
}
```

<br/>

### No.234 回文链表

- 链接：[https://leetcode.cn/problems/palindrome-linked-list/](https://leetcode.cn/problems/palindrome-linked-list/)
- 思路：这道题很奇怪，就是遍历链表，赋值到数组中，再循环判断。

```go
func isPalindrome(head *ListNode) bool {
var vals []int
	for ; head != nil; head = head.Next {
		vals = append(vals, head.Val)
	}
	n := len(vals)
	for i, v := range vals[:n/2] {
		if v != vals[n-1-i] {
			return false
		}
	}
	return true
}
```

<br/>

# 实现一个 LRU 缓存

``` go
package lru

import (
	"container/list"
	"errors"
)

const (
	MemoryOverFlow				int = 0
	MemorySizeError				int = 1
	NotFoundObject				int = 11
)


var lruErrorName = map[int]string{
	MemoryOverFlow:      "MemoryOverFlow",
	MemorySizeError: 	 "MemorySizeError",
	NotFoundObject: 	 "NotFoundObject",
}


// 记录存储数据的大小
type Value interface {
	Len() int64
}

// 存储的对象
type entry struct {
	key 	string
	value 	Value
}

type Cache struct {
	maxBytes int64        // 最大使用内存
	nBytes 	 int64        // 当前已使用内存
	ll *list.List         // 链表存储淘汰关系
	cache  map[string]*list.Element   //节点放到字典中，加速查找
	OnEvicted  func(key string, value Value)   //某条记录被删除时候的回调函数
}

func CreateLRUCache(maxByte int64,evicted func(string,Value)) *Cache {
	return &Cache{
		maxBytes:  maxByte,
		nBytes:    0,
		ll:        new(list.List),
		cache:     make(map[string]*list.Element),
		OnEvicted: evicted,
	}
}


func (c *Cache) Get(key string) (value Value,err error) {
	if ele, ok := c.cache[key]; ok {
		c.ll.MoveToFront(ele)
		return ele.Value.(*entry).value, nil
	}else {
		return nil, errors.New(lruErrorName[NotFoundObject])
	}
}

func (c *Cache) Add(key string, value Value) error {
	// 判断是否可以加入(太大会把所有缓存冲掉)
	if c.isOutOfMaxMemory(value.Len()) {
		return errors.New(lruErrorName[MemoryOverFlow])   // 内存不足以添加该缓存
	}

	// 判断缓存中已有该键, 更新
	if v, ok := c.cache[key]; ok {
		oldValue := v.Value.(*entry)
		// 可以加入,将该键移动到队头
		c.ll.MoveToFront(v)
		c.cache[oldValue.key] = &list.Element{Value: &entry{key,value}}
		c.nBytes = c.nBytes - oldValue.value.Len() + value.Len()
		return nil
	}
	// 没有该键,第一次添加
	ele := c.ll.PushFront(&entry{key, value})
	c.cache[key] = ele
	c.nBytes = c.nBytes + value.Len()

	// 若内存不够，需要循环删除掉最老的
	for c.maxBytes != 0 && c.maxBytes < c.nBytes {
		if err := c.RemoveOldest(); err != nil {
			return err
		}
	}

	return nil
}

func (c *Cache) RemoveOldest() error {
	oldest := c.ll.Back()
	if oldest != nil {
		oldestEntry := oldest.Value.(*entry)
		c.ll.Remove(oldest) // 删除链表节点
		c.nBytes = c.nBytes - oldestEntry.value.Len() // 删除字节长度
		delete(c.cache, oldestEntry.key)  // 删除字典
		if c.nBytes < 0 {
			return errors.New(lruErrorName[MemorySizeError])
		}
		if c.OnEvicted != nil {
			c.OnEvicted(oldestEntry.key, oldestEntry.value)
		}
	}
	return nil
}

func (c *Cache) Len() int64 {
	return int64(c.ll.Len())
}

func (c *Cache) isOutOfMaxMemory(size int64) bool {
	return size > c.maxBytes

```


**测试**
``` go
type GeeByte struct {
	Value []byte
}

func (g GeeByte) Len() int64 {
	return int64(len(g.Value))
}


func TestLruCache(t *testing.T) {
	lruObject := lru.CreateLRUCache(20, func(key string, value lru.Value) {
		fmt.Printf("%s 被删除了, 释放了 %d 字节的空间\n", key, value.Len())
	})

	_ = lruObject.Add("key1", GeeByte{[]byte("Hello, Golang")})
	_ = lruObject.Add("key2", GeeByte{[]byte("Hello, ByteDance")})

	if value, err := lruObject.Get("key2"); err != nil {
		log.Fatalln(err)
	}else {
		fmt.Printf("%s",value)
	}
}

```

io
