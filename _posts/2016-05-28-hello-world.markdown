---
layout: post
title:  "Hello GitHub io"
date:   2016-05-28 14:53:48 +0800
categories: jekyll update
---
This is my first post of GitHub io.

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