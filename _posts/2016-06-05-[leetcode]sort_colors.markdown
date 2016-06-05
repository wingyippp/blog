---
layout: post
title:  "[leetcode]75. Sort Colors"
date:   2016-06-05 14:26:00 +0800
categories: leetcode
---

[leetcode 75. Sort Colors][leetcode_sort_colors]

>Given an array with n objects colored red, white or blue, sort them so that objects of the same color are adjacent, with the colors in the order red, white and blue.
>Here, we will use the integers 0, 1, and 2 to represent the color red, white, and blue respectively.
>Note:
>You are not suppose to use the library's sort function for this problem.

### Solution#1

Loop the nums to count the number of red, white and blue. Then reset the value of the nums according to the count of 3 colors. Very straight forward.
{% highlight java %}
public class Solution {
    private static final int colorRed = 0;
    private static final int colorWhite = 1;
    private static final int colorBlue = 2;
    public void sortColors(int[] nums) {
        int redCount = 0;
        int whiteCount = 0;
        int blueCount = 0;
        for (int i = 0; i < nums.length; i++) {
            switch (nums[i]) {
                case colorRed:
                    redCount++;
                    break;
                case colorWhite:
                    whiteCount++;
                    break;
                case colorBlue:
                    blueCount++;
                default:
                    break;
            }
        }
        for (int i = 0; i < redCount; i++) {
            nums[i] = colorRed;
        }
        for (int i = 0; i < whiteCount; i++) {
            nums[i + redCount] = colorWhite;
        }
        for (int i = 0; i < blueCount; i++) {
            nums[i + redCount + whiteCount] = colorBlue;
        }
    }
}
{% endhighlight %}

### Solution#2

Loop the nums only once. Keep track of the starting index of color WHITE and BLUE.
Unfortunately, this solution takes more time then the Solution #1 to finish the test cases, because so many if else in the loop.
{% highlight java %}
public class Solution {
    private static final int RED = 0;
    private static final int WHITE = 1;
    private static final int BLUE = 2;
    
    public void sortColors(int[] nums) {
        int startWhiteIndex = -1;
        int startBlueIndex = -1;
        for (int i = 0; i < nums.length; i++) {
            switch (nums[i]) {
                case RED:
                    if (startWhiteIndex != -1) {
                        nums[startWhiteIndex++] = RED;
                        if (startBlueIndex != -1) {
                            nums[startBlueIndex++] = WHITE;
                            nums[i] = BLUE;
                        } else {
                            nums[i] = WHITE;
                        }
                    } else if (startBlueIndex != -1) {
                        nums[startBlueIndex++] = RED;
                        nums[i] = BLUE;
                    }
                    break;
                case WHITE:
                    if (startWhiteIndex == -1) {
                        if (startBlueIndex != -1) {
                            startWhiteIndex = startBlueIndex;
                            nums[startBlueIndex++] = WHITE;
                            nums[i] = BLUE;
                        } else {
                            startWhiteIndex = i;
                        }
                    } else {
                        if (startBlueIndex != -1) {
                            nums[startBlueIndex++] = WHITE;
                            nums[i] = BLUE;
                        }
                    }
                    break;
                case BLUE:
                    if (startBlueIndex == -1) {
                        startBlueIndex = i;
                    }
                    break;
            }
        }
    }
}
{% endhighlight %}

### Solution#3

Optimized the Solution #2 inspired by [others][others].
The code is much more graceful now.
{% highlight java %}
public class Solution {
    private static final int RED = 0;
    private static final int WHITE = 1;
    private static final int BLUE = 2;
    
    public void sortColors(int[] nums) {
        int indexRed = -1;
        int indexWhite = -1;
        int indexBlue = -1;
        for (int i = 0; i < nums.length; i++) {
            if (nums[i] == RED) {
                nums[++indexBlue] = BLUE;
                nums[++indexWhite] = WHITE;
                nums[++indexRed] = RED;
            } else if (nums[i] == WHITE) {
                nums[++indexBlue] = BLUE;
                nums[++indexWhite] = WHITE;
            } else {
                nums[++indexBlue] = BLUE;
            }
        }
    }
}
{% endhighlight %}


[leetcode_sort_colors]:https://leetcode.com/problems/sort-colors/
[others]:http://www.cnblogs.com/ganganloveu/p/3703746.html