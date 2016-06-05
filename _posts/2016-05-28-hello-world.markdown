---
layout: post
title:  "[leetcode]283. Move Zeroes"
date:   2016-05-28 14:53:48 +0800
categories: leetcode
---
[leetcode 283. Move Zeroes][leetcode_283_Move_Zeroes]

>Given an array nums, write a function to move all 0's to the end of it while maintaining the relative order of the non-zero elements.
>For example, given nums = [0, 1, 0, 3, 12], after calling your function, nums should be [1, 3, 12, 0, 0].

## Solution #1
Iterate the `nums` and save all the non-zero elements in a list.
Then, set the element of list into `nums` in order.
Last, set the values of all the remaining elements in `nums` to `0`.
{% highlight java %}
public class Solution {
    public void moveZeroes(int[] nums) {
        ArrayList<Integer> nonZeroNumList = new ArrayList<>();
        for (int i = 0; i < nums.length; i++) {
            if (nums[i] != 0) {
                nonZeroNumList.add(nums[i]);
            }
        }
        for (int i = 0; i < nonZeroNumList.size(); i++) {
            nums[i] = nonZeroNumList.get(i);
        }
        for (int i = nonZeroNumList.size(); i < nums.length; i++) {
            nums[i] = 0;
        }
    }
}
{% endhighlight %}

## Solution #2
Use 2 indexes `i` and `j` to iterate the `nums`. `i` is the current looping index of the `nums`. And `j` is the current index of non-zero element to be relocate into the `nums`.
{% highlight java %}
public class Solution {
    public void moveZeroes(int[] nums) {
        for (int i = 0, j = 0; i < nums.length; i++) {
            if (nums[i] != 0) {
                if (i != j) {
                    nums[j] = nums[i];
                    nums[i] = 0;
                }
                j++;
            }
        }
    }
}
{% endhighlight %}

[leetcode_283_Move_Zeroes]: https://leetcode.com/problems/move-zeroes/