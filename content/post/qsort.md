---
title: "快速排序理解"
date: 2021-02-25T16:42:00+08:00
categories: ["算法"]
---

今天看到了一个快速排序比较简洁的写法，代码如下：

```cpp

void quickSort(vector<int>& nums, int left, int right) {

        if (left < right) {
            int pi = partition(nums, left, right);

            quickSort(nums, left, pi - 1);
            quickSort(nums, pi + 1, right);
        }
        
    }

int partition(vector<int>& nums, int left, int right) {
    int pivot = left;
    for (int i = left; i < right; ++i) {
        if (nums[i] < nums[right]) 
            swap(nums[pivot++], nums[i]);
    }
    swap(nums[pivot], nums[right]);
    return pivot;
}

```

主要在其`partition()`函数部分，一开始首先初始化了支点pivot的值为left,这个地方容易误解，`pivot`并不是选取left为关键点，其实际选的关键点为`nums[right]`,也就是数组最右边的值，可以看到后面也是一直在和`nums[right]`比较，`pivot`其代表的是经过partition之后关键点在数组中所在的位置，以pivot作为分界，将数组分为两部分，其左边为小于`nums[right]`的数（为方便称其为小区），其右边为大于`nums[right]`的数（为方便称其为大区）。例如输入为`[5, 3, 9, 2, 7, 8]`, 最右边值为`8`, 经过第一次`partition()`之后，数组为`[5, 3, 2, 7, 8, 9]`,返回的`pivot`为4，也就是`8`在数组中的下标。  

知道上面这一点后，这个函数就容易理解了。函数会每次选取数组的最右边的值作为关键点，在下标为 [left - right-1] 的区间内内遍历和关键点比较，如果其值小于关键点，就交换`nums[pivot]`和`nums[i]`，也就是将`nums[i]`放到小区内，然后`pivot++`。此番遍历之后`pivot`所代表的下标位置就是关键点在数组中应该在的位置，函数最后`swap(nums[pivot], nums[right])`将其交换到其位置。也就是说每一次经过`partition()`之后，都有一个数确定被放到了正确的位置，这个数就是关键点`nums[right]`。  

`quicksort()`函数比较好理解，使用分治的思想递归的排序数组中每个部分，直到输入数组中只有一个元素，所有数字就排序好了，left < right这个边界条件也需要注意，之所以不取 left == right是因为在关键点返回的`pivot`为数组第一个或最后一个的时候，会出现left > right的情况。如输入`[5,2,3,1]`，关键点为1，经过第一次`partition()`返回的pivot为0，那么下次的左区`quicksort()`输入参数left=0，right=-1。若输入`[2,1,3,5]`，经过第一次`partition()`返回的pivot为3，那么下次的右区`quicksort()`输入参数left=4，right=3。