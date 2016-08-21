---
layout: post
title:  "[leetcode]374. Guess Number Higher or Lower"
date:   2016-08-03 14:26:00 +0800
categories: leetcode
---

[leetcode 374. Guess Number Higher or Lower][leetcode_guess_number_higher_or_lower]

>We are playing the Guess Game. The game is as follows:
>I pick a number from 1 to n. You have to guess which number I picked.
>Every time you guess wrong, I'll tell you whether the number is higher or lower.

>-1 : My number is lower

> 1 : My number is higher

> 0 : Congrats! You got it!

### Binary Search Recursively

It's quite straight forward to use the binary search recursively to implement the `guessNumber` method.
However, we will get a StackOverFlow exception when submitting this piece of code.
{% highlight java %}
public class Solution extends GuessGame {
    public int guessNumber(int n) {
        return guessNumberInternal(1, n);
    }
    
    private int guessNumberInternal(int left, int right) {
        int middle = (left + right) / 2;
        int result = guess(middle);
        switch (result) {
            case 1:
                return guessNumberInternal(middle + 1, right);
            case -1:
                return guessNumberInternal(left, middle);
            default:
                return middle;
        }
    }
}
{% endhighlight %}

### Binary Search with for loop

Use for loop to implement the binary search. Unfortunately, it turn out to be another error `Time Limit Exceeded`.
{% highlight java %}
public class Solution extends GuessGame {
    public int guessNumber(int n) {
        int left = 1;
        int right = n;
        int guessNumber = 1;
        while (left <= right) {
            int middle = (left + right) / 2;
            int result = guess(middle);
            boolean find = false;
            switch (result) {
                case 1:
                    left = middle + 1;
                    break;
                case -1:
                    right = middle - 1;
                    break;
                default:
                    guessNumber = middle;
                    find = true;
                    break;
            }
            if (find) {
                break;
            }
        }
        return guessNumber;
    }
}
{% endhighlight %}

### Integer overflow

It took me quite a few days to figure it out that was because of the overflow of the Integer when calcuating the `middle` valuable.
So change it to `int middle = left + (right - left) / 2`.
Finally, it's accepted!
{% highlight java %}
public class Solution extends GuessGame {
    public int guessNumber(int n) {
        int left = 1;
        int right = n;
        int guessNumber = 1;
        while (left <= right) {
            int middle = left + (right - left) / 2;
            int result = guess(middle);
            boolean find = false;
            switch (result) {
                case 1:
                    left = middle + 1;
                    break;
                case -1:
                    right = middle - 1;
                    break;
                default:
                    guessNumber = middle;
                    find = true;
                    break;
            }
            if (find) {
                break;
            }
        }
        return guessNumber;
    }
}
{% endhighlight %}

[leetcode_guess_number_higher_or_lower]: https://leetcode.com/problems/guess-number-higher-or-lower/